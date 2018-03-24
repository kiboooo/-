Glide的缓存机制
---
> Glide的简单调用：
> `Glide.with(this).load(url).into(imageView);`

`Glide.with()`方法用于创建一个加载图片的实例；
`with()`方法可以接受` Context，Activity,，Fragment`类型的参数，根据绑定的类型，决定了该实例的生命周期，例如：若绑定的是 `ApplicationContext`，那么只有应用程序被杀掉的时候，图片加载才会停止；( 主要是明白自己的生命周期 )

`load（）`用于指定加载的资源，对于Glide来说 ，它可以支持 
+ 网络图片
+ 本地图片
+ 应用资源
+ 二进制流
+ Uri对象 等。。

`into() `用于指定图片加载入哪一个`ImageView (容器)`；

> 小技巧：
> `Glide` 利用向当前的 `Activity `中添加一个隐藏的 `Fragment;`
> 利用 `Fragment `和 `Activity `生命周期的绑定机制，当`Activity`被销毁了，`Fragment`中可以监听到，故可以通知`Glide`停止加载操作；


#### Glide 的缓存策略：
##### 缓存Key
> 既然需要缓存，就有用于缓存的Key，而Glide中的Key的生成是再 Engine  类的 load( ) 方法中；

```java
//该方法是获取一个Id 字符串 ，这个字符串是加载图片的唯一标识；
//当这个图片是网络图片的话，id = url 地址；
 long startTime = LogTime.getLogTime();

//构建一个EngineKey对象，作为Glide中缓存的Key存在；
//由此可见，决定缓存的因素很多，每当更改其中任何一个属性，都会生成相应的缓存；
//该方法主要是重写了 equals() 和 hashCode() 方法，保证只有所有参数相同的情况下，才会生成同一个EngineKey；
EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,resourceClass, transcodeClass, options);

EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
    if (cached != null) {
      cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Loaded resource from cache", startTime, key);
      }
```

##### 内存缓存

在Glide中，内存缓存是默认实现的，只要被Glide加载过一边的图片，就自动把它加载到内存缓存之中；
我们可以通过  在  `.into( ) `方法设置之前，设置 `.skipMemoryCache(true)`方法，就可以跳过内存缓存的这一步；

> 所有内存缓存，都是使用了LruCache算法，主要实现的功能是：
> 把最近使用的对象的强引用存储在LinkedHashMap中，并把最近最少使用的对象在缓存值达到预设值之前，从内存中移除；

在Glide中不仅使用 LruCache 算法（从LruResourceCache即 linkHashMap 中获取对象） 还结合了一种弱引用的机制（用弱引用声明了一个HashMap，用来缓存正在使用的图片，防止被LruCache 回收）；

> 对于 Glide 来说，它的缓存机制为二级缓存（内存缓存和 硬盘缓存），网络缓存某种层面上并不能算是一种缓存；

##### Glide 在加载图片的时候：

> into() -> 获取 request 对象调用 runRequest-> begin() -> 
设置加载图片的宽高 onSizeReady ->  Engine.load -> load() ->loadFromCache() 一级缓存 LHM -> loadFromActiveResources() 二级缓存 弱引用HM ->  engineJob.start(runnable)会启动EngineRunnable的start()方法（磁盘）-> decode( ) -> decodeFromCache 磁盘获取 
以上操作都没有才会从网络中获取 decodeFromSoure();
+ 先调用` loadFromCache()` 方法 从LinkedHashMap中查找有没有缓存的实例；
+ 再调用`loadFromActiveResources()`从弱引用生命的HashMap 即当前使用图片缓存中查找有没有缓存的实例；
+ 最后，内存缓存中都不存在该图片的实例的时候，就会通过 EngineJob 中的 start（）方法开线程去磁盘中查找，如果没有就从网络中加载；

图片的写入内存缓存的契机在一个图片加载完成的时候，
+ 这个时候通过回调，把当前的图片加入到弱引用的HashMap中缓存；
+ 而在使用图片的时候 维护了一个 信号量 `acquired` 来记录图片被引用的次数，实现Lru算法，当 `acquired == 0` 的时候，该图片从弱引用的HashMap中出来，放入linkedHashMap中；

##### 硬盘缓存
`diskCacheStrategy()`方法基本上就是Glide硬盘缓存功能的一切，它可以接收四种参数：

+ DiskCacheStrategy.NONE： 表示不缓存任何内容。
+ DiskCacheStrategy.SOURCE： 表示只缓存原始图片。
+ DiskCacheStrategy.RESULT： 表示只缓存转换过后的图片（默认选项）。
+ DiskCacheStrategy.ALL ： 表示既缓存原始图片，也缓存转换过后的图片

Glide是使用的自己编写的DiskLruCache工具类, 来进行把图片存入硬盘中；
 
 从硬盘中获取图片的契机是，在内存缓存中获取不到图片实例；然后通过开线程加载图片，调用DecodeJob的`decodeResultFromCache()`方法来获取缓存(一个获取压缩过的图片)，如果获取不到，会再调用`decodeSourceFromCache()`方法获取缓存（获取原始图片），这两个方法的区别其实就是`DiskCacheStrategy.RESULT`和`DiskCacheStrategy.SOURCE`这两个参数的区别；

由于在没有缓存的情况下，会调用`decodeFromSource()方法来读取原始图片` 在该方法中，通过判断是否允许缓存，允许就调用了`getDiskCache()`方法来获取`DiskLruCache`实例，接着调用它的`put()`方法就可以写入硬盘缓存了，注意原始图片的缓存Key是用的`resultKey.getOriginalKey()`。

而对于，压缩过后的缓存，把图片经过压缩转换后，同样调用`getDiskCache()`方法来获取`DiskLruCache`实例，接着调用它的`put()`方法，把缓存key设为	`resultKey`。


### Glide
glide  功能更强大，支持Activity，fragment，Context 。
picasso    支持Context ，不支持gif，加载慢。
fresco  易用性不及Glide，内存占用高，特别是占用系统的对内存。

glide：内存和磁盘缓存。根据ImagVIew空间大小获取对应大小的Bitmap展示，。
fresco：三级缓存，Bitmap缓存，未解码图片缓存，文件缓存。缓存原始图片。

fresco功能强大，占用内存高，需要依赖多，操作复杂。
glide生命周期好管理，bitmap复用和主动回收，加载快，内存开销小。
 
Glide.with(this).load(url).into(imageView);
with是Glide类中的多个重载方法，Activity,Fragment,Context为参数;
with方法调用get方法，
如果参数是Application对象，直接掉通用getApplicationManager()获得一个RequestManager的对象，因为application的生命周期和程序一样，不需要特殊处理
如果是Activity，fgragment的非Application对象，会添加一个隐藏的Fragment来监听Activity的生命周期，如果Activity被销毁，Glide就会及时停止图片加载，但如果非主线程使用Gldie都会被当做apllication处理，。
with返回RequestManager对象，继续调用
Load方法，Load方法在RequestManager源码中又调用了loadGeneric这个方法并返回了一个
DrawableTypeRequest对象，DrawableTypeRequest提供了默认方式和asBitmap和asGif的资源获取方式。还有EngineJob,用卡来开启线程，异步加载图片，从服务器读取时，decodeSream（）会先读取两个字节确定是图片还是gif，然后解码。DrawableTypeRequest中的glide.buildTranscoder()构建ResourseTranscoder对图片进行转码，返回一个bitmap对象，加载的时候在把bitmap转为drawable对象。
然后DrawableTypeRequest再调用
into方法，into方法在DrawTypeRequest的父类GenericQuestBuilder的into方法，调用glide.buildImageViewTarget()方法返回一个最终展示图片用到的Target对象，这个Target对象是由ImageVewTargeFactory类中的buildTarget（）返回的，buildTarget会根据class参数构建不同的，如果之前调用了asBitmap方法，就会构建出BitmapImageViewTarget的对象，否则就是GlideDrawableImageViewTarget对象。into方法最后一行把刚获取的GlideDrawableViewTarget的对象又传到了另一个into（）方法中。
二次的这个into方法	，用buildRequest方法构建了一个Request对象，并执行了这个request， Request是用来发送加载图片的请求，buildrequest调用了buildRequestRecursive,obtainRequest（）方法在调用GenericRequest的obtain()，new GenericRequest() 获取一个request对象。创建好之后二层的into方法后调用了requestTracker.runRequest,这个方法会判断Glide是否处于等待状态，不等待就调用Request的begin方法执行请求，否则就把request加到执行队列，等待状态结束后，按顺序执行。
在图片请求开始之前，先调用占位图方法在ImageVIew的位置。
如果指定了图片的宽高，调用onSizeReady.
如果未指定高度，target.getSize()，然后onSizeReady;
！！！！ onSizeReady调用loadProvider.getModelLoader（）；


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

Glide 在加载图片的时候：
+ 先调用` loadFromCache()` 方法 从LinkedHashMap中查找有没有缓存的实例；
+ 再调用`loadFromActiveResources()`从软引用生命的HashMap 即当前使用图片缓存中查找有没有缓存的实例；
+ 最后，二级缓存中都不存在该图片的实例的时候，才会去开线程去加载图片；

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
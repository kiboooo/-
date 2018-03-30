#  OkHttp与Retrofict

[流行网络请求框架](https://upload-images.jianshu.io/upload_images/944365-f48072d21b613aaf.png)

![enter image description here](https://upload-images.jianshu.io/upload_images/944365-f48072d21b613aaf.png)


## OkHttp:
> Android 提供了两种与HTTP交互的方式：
> HttpURLConnection 和 Apache HTTP Client，但是OkHttp更加高效。
> 但是并不是基于他们来进行实现的；
> 其特点：
> 1.支持Google提出的SPDY网路传输协定，主要用来传送网页内容，它共享了一个Socket来处理同一个服务器的所有请求；
> 2.若SPDY不可用了，则实现了连接池来减少请求的延时；
> 3.拥有缓存机制，利用缓存响应数据来减少重复的网络请求；
> 基本流程框图如下：
![enter image description here](https://upload-images.jianshu.io/upload_images/2540122-e1e23f03d13906e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)


###  OkHttp的基本使用：
> 运用责任链模式，每层都使用相应的拦截器进行逻辑处理；
> 请求流程：
> + OkHttpClient 实现 Call.Factory，负责为 Request 创建 Call；
+ RealCall 为具体的 Call 实现，其 enqueue() 异步接口通过 Dispatcher 利用 ExecutorService 实现，
+ 而最终进行网络请求时和同步 execute() 接口一致，都是通过 getResponseWithInterceptorChain() 函数实现；
+ getResponseWithInterceptorChain() 中利用 Interceptor 链条，分层实现缓存、透明压缩、网络 IO 等功能；

#### 1.创建Okhttp的Client对象；
+ 在Buider中可以对Client进行一些链接的状态设定；
+ 例如：
	+ .connectTimeout(20, TimeUnit.SECONDS)链接超时的时间；
	+  .readTimeout(20, TimeUnit.SECONDS)，读取超时时间；
	+  代理，缓存，拦截器等自定义设置；

#### 2.发起Http请求；
> 每个	call	只能被执行一次！

+ 利用Request.Builder 绑定Url，建立Request (体现了建造者模式的使用)；
	+ 建造者模式：将一个复杂对象的构建与它表示进行分离，使得同样的构建过程可以创建不同的表示；
	+ 优点：良好的封装性；独立，容易扩展；
	+ 缺点：会产生多余的Builder 对象以及 Director 对象，消耗内存；对象创建过程被暴露；
+ 通过Client中的newCall来处理Request请求；
+ 在newCall中回调了实现了Call方法的实现类RealCall，并使得RealCall类中持有Client 和 request的引用；
+ 通过RealCall这个实现类，去调用同步请求（execute）还是异步请求 ( enqueue ) ；

> OkHttp内部维护了三个调用队列
> readyAsyncCalls : 存放入未能放入运行队列的调用请求
> runningSyncCalls：依次同步运行队列内的调用请求
> runningAsyncCalls：依次异步运行队列内的调用请求


+ execute：同步调用，通过client中的dispatcher这个事件分发者去处理，调用dispatcher中的execute方法，把RealCall对象放入runningSyncCalls同步调用的双端队列中；通过getResponseWithInterceptorChain获取请求返回的结果 Response ;


+ enqueue：异步调用，同样是需要使用dispatcher这个事件分发者中调用enqueue的方法，为了维持call唯一性的原则，创建了一个异步调用（AsyncCall）对象传入enqueue；同时若满足runningAsyncCalls异步队列未满（最大并发数未达到 64 个），runningCallsForHost共享主机未达到最大值（5 个）的情况下：除了添加runningAsyncCalls之外，还添加到线程池，并且由线程池（线程池的线程等待时间为60s，大于这个时间线程回退出），通过 RealCall 的内部类 AsyncCall 调用 execute 方法 。若未满足条件，就放入readyAsyncCalls准备队列，等待请求主机少于最大值再依次放入调用数组和线程池中 。
+ 不管是同步还是异步；都要通过getResponseWithInterceptorChain 方法调用责任链的形式，通过相应的拦截器 去获取 response；
	> Okhttp实现的拦截器：
> 1.RetryAndFollowUpInterceptor
>  这拦截器主要是做重试，网络错误，以及请求重定向的一些操作。
> + 通过一系列的try - cach 检测路由和 服务器的连接；
> + 获取返回的 response ；
> + 判断 response 和 followUp 的请求是否同一个连接；若为重定向就销毁旧连接，创建新连接；
>
> 2.BridgeInterceptor
> 这个拦截器，主要把用户的请求转换为网络的请求，封装为正常请求URI, 把服务器返回的响应转换为用户友好的响应的，添加一些头部信息等
> 3.CacheInterceptor
> 缓存拦截器，负责读取缓存直接返回、更新缓存的（缓存逻辑的实现按，网络和内存缓存，需要用户显式声明调用）
> + 获取之前对应的缓存 response 比较其时效性，获取缓存的策略；判断是否适用，不合适就关闭；
> + 根据当前网络状况选择返回缓存 还是 返回网络请求的返回数据，再更新新的缓存；

> 4.ConnectInterceptor
> 连接拦截器，和服务器建立连，主要是处理连接服务器，以及http，https的包装；通过HttpCode 对象对应 HTTP/1.1 和 HTTP/2 版本的实现；
>  + doExtensiveHealthChecks先查询当前的连接是否良好，若断开从连接池中找可用连接；
>  + 通过 newStream 创建 HttpCodec 对象，确定相应的连接协议；
>  + HttpCodec 实际上利用的是 OKio这个接口进行数据流传输，实现方式为 Socket；
>  
> 5.CallServerInterceptor
>  服务拦截器，主要是发送（write、input）、读取（read、output）数据。也是拦截器的最后一个环节，这里就真正拿到了网络的结果了。
>  **在这里，位置决定了功能，最后一个	Interceptor	一定是负责和服务器实际通讯的， 重定向、缓存等一定是在实际通讯之前的。**
>  在Interceptor这个链式分布处理机制上，很好体现了责任链这种设计模式；
>   让每一个Interceptor只处理自己可以处理的任务，把不能处理的部分保留并传递给下一个Interceptor直到被处理完毕；从而把网络请求这个就从Realcall类中抽离出来，只需要专注接受请求成功的结果；（该模式在点击事件TouchEvent中也有体现，通过不同的不同拦截器，拦截消费点击事件）

+ OkHttp内部是使用拦截器来完成请求和相应；而拦截器可以简单理解为“拒绝接受你不想接受的东西”，拦截器是java反射机制的体现，它只对Action起作用，OkHttp用它进行网络调用的Action拦截，再对内容进行修改；

#### 3.返回数据的获取；
+ 通过getResponseWithInterceptorChain()把自定义的拦截器和OkHttp制定的拦截器加入一个interceptors数组；
+ 创建一个拦截器链——RealInterceptorChain，用上述的数组和请求头（Request）作为参数传入；
+ 由于RealInterceptorChain是Interceptor中的一个Chain接口的实现类，通过不断的调用数组中每个Interceptor中的proceed方法，依次调用每种类型的拦截器去处理发出去的请求，每一个拦截器都可以根据相应的逻辑判断是否已经满足获得 response 或者向下传递；
+ 异步请求的操作，是通过线程池来存储请求，使用的	AsyncCall	是	RealCall	的一个内部类，它实现了	Runnable	，所以可 以被提交到	ExecutorService	上执行，而它在执行时会调用AsyncCall中的execute方法中的getResponseWithInterceptorChain()函数，并把结果通 过responseCallback传递给上层使用者；
+ 线程池重用维护：
	+ OkHttp不是在线程池中维护线程的个数，线程是一直在Dispatcher中直接控制。线程池中的请求都是运行中的请求。这也就是说线程的重用不是线程池控制的，通过源码我们发现线程重用的地方是请求结束的地方finished(AsyncCall call) ，而真正的控制是通过其中的promoteCalls方法， 根据maxRequests和maxRequestsPerHost来调整runningAsyncCalls和readyAsyncCalls，使运行中的异步请求不超过两种最大值，并且如果队列有空闲，将就绪状态的请求归类为运行中；
+ 不管同步请求还是异步请求都需要使用Dispatcher中的finished()方法把当前的调用请求从队列中去掉；而异步请求不同的是，需要依靠其中的promoteCalls方法调用把readyAsyncCalls队列中的元素加入异步队列；
+ **Response中的Body数据部分，常常返回的信息特别大，需要使用数据流的方式去访问（提供了string()和bytes() 去一次性读取完毕）**
	+ 需要注意：
	+ 每个body只能被消费一次，多次消费会抛出异常
	+ body使用后必须要关闭，否则会发生资源泄露（string方法中已经实现了close）
	==可以转入Gson解析Json数据==


#### 4.Http的缓存；
> 简单回顾一下LRU（最近最久未使用）置换算法
> 主要思想：顾名思义，主要把最久未被使用的页面淘汰；
> 基础实现是赋予每一个页面一个访问字段，用来记录一个页面自上次被访问以来经历过的时间 t，当需要淘汰页面的时候，就是选择当前 t 值最大的；
> 具体的实现常有寄存器和栈；
> 寄存器：
> 一个n位的移位寄存器代表一个页面，当某个页面被调用，把相应寄存器的代表的Rn-1位(最高位)置1 ；
> 设置一个间隔时间(100ms)，就将整个寄存器右移一位（相当于把值/2）；
> 当需要页面调度的时候，判断哪一个页面的寄存器最小，就淘汰；
> 栈：
> 把每一个调用的页面的页号压入栈中，保证在栈底的元素是最久未使用的，栈顶永远是刚刚被使用的页面；
> 若原本栈中存在该页号，就把该页号直接移动到栈顶；
> **Glide的图片缓存的调度算法也是用LRU，是基于一个LinkedHashMap实现的**

#### CacheInterceptor 实现细节

```java
@Override public Response intercept(Chain chain) throws IOException {
    //默认cache为null,可以配置cache,不为空尝试获取缓存中的response
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;

    long now = System.currentTimeMillis();
    //根据response,time,request创建一个缓存策略，用于判断怎样使用缓存
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;

    if (cache != null) {
      cache.trackResponse(strategy);
    }

    if (cacheCandidate != null && cacheResponse == null) {
      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }

    // If we're forbidden from using the network and the cache is insufficient, fail.
    //如果缓存策略中禁止使用网络，并且缓存又为空，则构建一个Resposne直接返回，注意返回码=504
    if (networkRequest == null && cacheResponse == null) {
      return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }

    // If we don't need the network, we're done.
    //不使用网络，但是又缓存，直接返回缓存
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }

    Response networkResponse = null;
    try {
      //直接走后续过滤器
      networkResponse = chain.proceed(networkRequest);
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      //保证缓存不会因为，I/O这样的阻塞事件发生而影响泄漏
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }

    // If we have a cache response too, then we're doing a conditional get.
    //当缓存响应和网络响应同时存在的时候，选择用哪个
    if (cacheResponse != null) {
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        //如果返回码是304，客户端有缓冲的文档并发出了一个条件性的请求（一般是提供If-Modified-Since头表示客户
        // 只想比指定日期更新的文档）。服务器告诉客户，原来缓冲的文档还可以继续使用。
        //则使用缓存的响应
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        cache.trackConditionalCacheHit();
        cache.update(cacheResponse, response);
        return response;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }
    //使用网络响应
    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();
    //所以默认创建的OkHttpClient是没有缓存的
    if (cache != null) {
      //将响应缓存
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
        // Offer this request to the cache.
        //缓存Resposne的Header信息
        CacheRequest cacheRequest = cache.put(response);
        //缓存body
        return cacheWritingResponse(cacheRequest, response);
      }
      //只能缓存GET....不然移除request
      if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
          cache.remove(networkRequest);
        } catch (IOException ignored) {
          // The cache cannot be written.
        }
      }
    }

    return response;
  }
```
其中的 `InternalCache cache` 在构造函数被赋值，传入的对象就是 OkHttpClient ；所以这就导致了如果你需要OkHttp进行缓存操作的话，就需要在 Client 构造的时候，进行构造：`OkHttpClient client = new OkHttpClient.Builder()
        .cache(cache)
        .build();
`
对于一开始的缓存的获取：
```java
Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;
```

其中的 cache . get（）方法存在于实现了 IntemalCache接口的 Cache 类。
通过 对 requset.Url 进行取相应的 key 值，作为缓存的获取键值；
get（）主要操作：
+ 初始化日志文件和 lruEntries
+ 检查保证key正确后获取缓存中保存的Entry
+ 操作计数器+1
+ 往日记文件中写入这次Read 操作
+ 根据 readunbdantOpCount 判断是否需要清理日志信息；
+ 需要则开启线程清理；
+ 不需要则返回缓存；

那么缓存的具体取用：
+ 通过执行DiskLruCache 的 get方法 拿到  snapshort
+ 从 snapshort 中的 cleanFiles[0] 中保存的头信息，构建相关的信息的Entry；
+ 从 cleanFiles[1] 构建 body 信息，最终构建成缓存中保存的 Response;
+ 返回 缓存中的 Response ；

##### CacheInterceptor 的主要流程：
1.通过Request 尝试到 Cache 中拿缓存（流程由上所述），前提是开启了缓存的设置；
2. 根据 response，time，request 创建一个缓存策略，用于判断如何使用内存；
3. 如果缓存策略中设置禁止使用网络，并且缓存又为空，则构建一个Response 直接返回 304（页面未更新）；
4. 禁止网络，但是有缓存，直接返回缓存；
5. 接着去执行过滤器流程， chain.proceed(networkRequest);
6. 当缓存存在的时候，如果网络返回 Response 为 304，就使用缓存的 Response；
7. 构建网络请求的Response；
8. 当设置了缓存需求，就把新的Response 缓存，先缓存 header ，再缓存 body
9. 返回相应的 Response;

> okHttp的开发者认为，从效率角度考虑，最好不要支持POST请求的缓存，暂时只要支持GET形式的缓存,源码如下。


```java
@Nullable CacheRequest put(Response response) {
    String requestMethod = response.request().method();

    if (HttpMethod.invalidatesCache(response.request().method())) {
      //OKhttp只能缓存GET请求！。。。
      try {
        remove(response.request());
      } catch (IOException ignored) {
        // The cache cannot be written.
      }
      return null;
    }
    if (!requestMethod.equals("GET")) {
      //OKhttp只能缓存GET请求！。。。
      // Don't cache non-GET responses. We're technically allowed to cache
      // HEAD requests and some POST requests, but the complexity of doing
      // so is high and the benefit is low.
      return null;
    }

    if (HttpHeaders.hasVaryAll(response)) {
      return null;
    }

    Entry entry = new Entry(response);
    DiskLruCache.Editor editor = null;
    try {
      editor = cache.edit(key(response.request().url()));
      if (editor == null) {
        return null;
      }
      //缓存了Header信息
      entry.writeTo(editor);
      return new CacheRequestImpl(editor);
    } catch (IOException e) {
      abortQuietly(editor);
      return null;
    }
  }
```

在拦截器CacheInterceptor向下分发Resqust 执行责任链操作之前，我们检查响应是否已经被缓存、缓存是否可用，时效性的保证是网络请求返回头所携带的 Cache-Control 对应的属性值决定 ， 但是返回头如果不存在该信息的设定，你又想实现这个有效的时间控制保活的机制，那么可以添加自定义的拦截器，实现人为的添加 Reponse 中的请求头，再把其返回给下层处理；

如果可用则直接返回缓存的数据，否则就进行后面的流程，并在返回之前，把网络的数据写入缓存；

在执行缓存的流程的过程中，涉及到 journalFile 这个日志文件，就是为了记录缓存一系列的一些操作；

内存管理：
主要涉及 HTTP 协议缓存细节的实现，而具体的缓存逻辑 OkHttp 内置封装了一个 Cache 类，它利用 DiskLruCache，用磁盘上的有限大小空间进行缓存，按照 LRU（最近最久未使用） 算法进行缓存淘汰；

> 我们可以在进行OkHttpClient构造时，设置Cache对象，指定目录和缓存大小；
> 也可以自定义一个实现了InternalCache接口的拦截器，在里面实现自定义的缓存策略；

## Retrofits
> Retrofit是一个基于OkHttp实现的网络请求框架；
> 网络请求的工作本质是由OkHttp完成的，而Retrofit仅负责网络请求接口的封装；
> 通过注解去配置请求：请求的方法，请求的参数，请求头，返回值等；
> 可以搭配多种Converter将获得的解析，提供序列化的形式（支持Gson，Jackson，Protobuf等）；
> 提供对RxJava的支持；
> 在众多网络请求框架中，性能最好，处理最快；
> 扩展性差，很多高度封装的必然后果，解析数据都是使用统一的converter,倘若服务器不能给出统一的API格式，将很难进行

向服务器请求API所有的请求框架都一样：
+ build request (API参数设置)
+ executor （执行请求）
+ parse callback （解析数据，返回给上层）

而对应于Retrofit来说：
+ 通过注解去配置API
+ CallAdapter（相当于execute）
+ Converter（解析数据）

调用流程图：
![enter image description here](http://123.207.145.251:8080/SimpleBox/picture/1514455314611.jpg)

#### 简单使用：
##### 1.创建 接受服务器返回数据 的类，用于解析数据；
##### 2.创建 用于网络请求的接口
> 将 Http请求 抽象成 Java接口：采用 **注解** 描述网络请求参数 和配置网络请求参数 ；
> + 用动态代理 动态 将该接口的注解 “翻译” 成一个Http请求，最后再执行Http请求；
> + 每个方法都需要使用注解标注，否则会报错；

 注解类型图解：
![enter image description here](http://upload-images.jianshu.io/upload_images/944365-ee747d1e331ed5a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
特殊的：@HTTP ：用于替换以上7个注解的作用；
##### 3.创建 Retrofit实例
进行一些请求属性的设置；
##### 4.创建 网络请求接口的实例 并 配置网络请求参数
最关键的是 retrofit.create 方法，运用了动态代理的技术创建出API的实例；简单来说，就是动态生成接口的实现类，并创建其实例（称之为代理）；代理把对接口的调用转发给 InvocationHandler 实例，而在 InvocationHandler 的实现中，除了执行真正的逻辑（例如再次转发给真正的实现类对象），我们还可以进行一些有用的操作，例如统计执行时间、进行初始化和清理、对接口调用进行检查等。

> 使用动态代理的原因是：
> 解耦+方便调用不需要写太多的实例接口类；
> 因为对接口的所有方法的调用都会集中转发到 InvocationHandler#invoke 函数中，我们可以集中进行处理，更方便了

```java
@Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
            //例如调用 equals，toString 就直接调用
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
            //调用的 default 方法，就是在接口中可以包含方法体，java8新特性
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            //重要：
            //主要是把接口方法的调用转为一次HTTP调用
            ServiceMethod serviceMethod = loadServiceMethod(method);
            //一个 ServiceMethod 对象对应于一个 API interface 的一个方法，loadServiceMethod(method) 方法负责加载 ServiceMethod：
            
            //封装了OkHttp.Call来进行同步和异步的请求
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
```


###### 4.1 简单了解动态代理：
> 所谓的代理：
> 在某些情况下，我们不希望或是不能直接访问对象 A，而是通过访问一个中介对象 B，由 B 去访问 A 达成目的，这种方式我们就称为代理。这里对象 A 所属类我们称为委托类，也称为被代理类，对象 B 所属类称为代理类。

实现动态代理需要：
+ (1). 新建委托类；
+ (2). 实现InvocationHandler接口，这是负责连接代理类和委托类的中间类必须实现的接口；
+ (3). 通过Proxy类新建代理类对象。

`Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service }, new InvocationHandler() `
> service.getClassLoader()表示类加载器
> service 表示委托类的接口，生成代理类时需要实现这些接口
> new  InvocationHandler实现类对象，负责连接代理类和委托类的中间类
> 所谓的动态代理，就是双层的静态代理 ；
> 开发者提供了委托类 B，程序动态生成了代理类 A。开发者还需要提供一个实现了InvocationHandler的子类 C，子类 C 连接代理类 A 和委托类 B，它是代理类 A 的委托类，委托类 B 的代理类。用户直接调用代理类 A 的对象，A 将调用转发给委托类 C，委托类 C 再将调用转发给它的委托类 B


##### 5.发送请求（同步/ 异步）
由于Retrofit是基于OkHttp实现的，它的主要作用是对请求接口的API进行一个封装；
所以，无论是同步还是异步请求，都是经过Retrofit封装的OkHttpCall类中的OkHttp的引用去调用execute和enqueue方法；最终都是调用Client实例的newCall方法去进行网络请求；

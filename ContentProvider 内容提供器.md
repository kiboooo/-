## ContentProvider 内容提供器
ContentProvider 作为搬运工的角色，作为数据源和数据传输中的桥梁；跨进程通信

数据源可以是： 数据库，文件，XML，网络等等；

总的来说就是将存放在Android的数据，以安全的方式进行封装，最终提供统一的存储 和 获取数据的接口供其他进程使用，从而实现跨进程通信；

> 跨进程的通信方式：
> 文件，AIDL（基于Binder），Binder，Messager（基于Binder），ContentProvider（基于Binder），Socket；

#### URI（统一资源标识符）
作为 ContentProvider 和其中的数据的唯一标识； 

URL分为系统预置和自定义， 对应系统内置的数据（通信录，日程表等） 和 自定义数据库等

##### 自定义的形式：
例如：
`content://com.carson.provider/User/1`
其中：
主题名： content://  ，URI的前缀；
授权信息（Authority）：com.carson.provider ，Content Provider 的唯一标识符，用于对不同的应用程序来进行区分，通常采用包名来命名；
表名（path）：User ， ContentProvider 指向数据库中的某个表名；
记录： 1  ， 表中的某个记录；

`Uri	uri	=	Uri.parse("content://com.carson.provider/User/1")`
所代表的资源为：
名为	`com.carson.provider`的`ContentProvider `	中表名	为`User`	中的	`id`为1的数；

URI 模式可以存在匹配的通配符：

例如 ： *  和  # 

 （*）：匹配任意长度的任何有效字符的字符串；
 （#）： 匹配任意长度的数字字符的字符串；

#### MIME数据类型：
> 指定某个扩展名的文件用某种应用程序来打开，就是设定默认执行；

通过 `ContentProvider.getType(Uri)`返回需要的MIME类型；

##### MIME类型的组成 = 类型 + 子类型（字符串）；
 > text / html ： 类型 = text；子类型 = html

##### MIME 类型主要有 2 种形式；
+ 单条记录：vnd.android.cursor.item/自定义
+ 多条记录： vnd.android.cursor.dir/自定义 

注：
vnd： 表示父类型和子类型具有非标准的，特定的形式；

#### ContentProvider 类主要以表格的形式存储数据（也支持文件）
表格的逻辑结构，和数据库大体相同；

##### 进程中共享数据的本质是：增删改查；
核心方法也是这四个：
```java
public Uri insert(Uri  uri,	ContentValues values)			
//	外部进程向 ContentProvider	中添加数据
public int delete(Uri uri, String selection, String[] selectionArgs)			//	外部进程	删除	ContentProvider	中的数据

public int update(Uri uri, ContentValues values, String	select ion,	String[]	selectionArgs)		
//	外部进程更新 ContentProvider	中的数据
public Cursor query(Uri uri, String[] projection, String selection,	String[] selectionArgs, String sortOrder)　			
//	外部应用	获取	ContentProvider	中的数
```
这四个方法都是由外部进程进行回调，并运行在ContentProvider 进程的 Binder 线程池中；

当出现多线程并发访问的时候，需要实现线程同步；

> ContentProvider 类不会直接与外部进程交互，而是通过 ContentResolver 类；

#### ContentResolver 
> 作用：
> 1. 通过 URI 操作不同的ContentProvider 中的数据；
> 2. 外部进程需要通过 ContentResolver 类 从而与 ContentProvider 类进行交互；

通过ContentResolver这个封装类去统一管理 ContentProvider ；就不必要去了解每一个使用到的 ContentProvider 的具体实现，就可以直接完成数据交互，降低了操作成本和难度；

为了体现两者的无缝对接，ContentResolver 中提供了与 ContentProvider 相同名字和作用的 4 个方法；

##### 辅助 ContentProvider 的工具类
+ ContentUris
操作对象是Uri ，核心方法：
	+ withAppendedId（）向URI追加一个 ID 
	+  parseId（）获取URI的ID
+ UriMatcher
在 ContentProvider 中注册 URI；
根据 URI 匹配 ContentProvider 中对应的数据表；
+ ContentObserver
观察 URI 引起的 ContentProvider 中数据的变化，并通知外界；

#### ContentProvider 不仅用来解决进程中通信，同时也适用于进程内通信；

#### 优点：
1 . 为应用间的数据交互提供了一个安全的环境；
2. 访问简单和高效；
> 对比：
> 文件：需要不断的读写数据
> Sharedpreferences 需要使用它的API来读写；
> ContentProvider 解耦了底层数据的存储方式，使得无论底层数据采用何种形式，外界对数据的访问都是唯一的，也就是说通过封装类 对数据进行操作；






   
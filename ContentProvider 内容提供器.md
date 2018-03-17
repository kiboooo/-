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
 *：匹配任意长度的任何有效字符的字符串；
 # ： 匹配任意长度的数字字符的字符串；

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

### Android7.0 危险文件访问适配
为了提高私有目录的安全性，防止应用信息的泄漏，从7.0开始，应用私有目录的访问权限被做了限制；

不能直接通过 file :// URI 访问其他引用的私有目录文件或让其他应用访问自己的私有目录；
否则就会出现 FileUriExposedException 异常，出现应用崩溃闪退等问题；

举例：在7.0上，希望下载下来的APK可以直接通过获取文件存放位置，使用 Uri.fromFile( filePath ) 获取相应的URI，借助 intent 调用系统安装程序 ，进行安装；

#### 利用 FileProvider 解决
FileProvider 是 contentProvider 的一个特殊子类，把访问受限制的 file：// 转化为可以授权分享的 content：// URI ;

使用：
#### 注册 FileProvider
需要在 Manifest 文件中添加注册信息；
```java
<provider
 android:name="android.support.v4.content.FileProvider"
  android:authorities="com.atguigu.chinarallway.fileprovider"
            android:grantUriPermissions="true"
            android:exported="false">

     <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/filepaths" />
        </provider>
```
android :  authorities 的属性值是一个由 build.gradle 文件中的 applicationId 值 和自定义名称组成的Uri字符串；

#### 第二步，添加共享目录
在 res / xml 目录下 新建一个 xml 文件，用于存放应用需要共享的目标文件。
```java
<resources
   >
    <paths>
        <external-path path ="." name="downloadVideo" />
        <external-path path="" name = "download"/>
        <!--<external-files-path path="Download/" name="Download" />-->
    </paths>
</resources>
```
`<files-path> `：内部存储空间应用私有目录下的 files/ 目录，等同于 Context.getFilesDir() 所获取的目录路径；

`<cache-path>`：内部存储空间应用私有目录下的 cache/ 目录，等同于 Context.getCacheDir() 所获取的目录路径；

`<external-path>`：外部存储空间根目录，等同于 Environment.getExternalStorageDirectory() 所获取的目录路径；

`<external-files-path>`：外部存储空间应用私有目录下的 files/ 目录，等同于 Context.getExternalFilesDir(null) 所获取的目录路径；

`<external-cache-path>`：外部存储空间应用私有目录下的 cache/ 目录，等同于 Context.getExternalCacheDir()；

##### path 属性 
> 用于指定当前子元素所代表目录下需要共享的子目录名称。
注意：path 属性值不能使用具体的独立文件名，只能是目录名

##### name属性
> 用于给 path 属性所指定的子目录名称取一个别名。后续生成 content:// URI 时，会使用这个别名代替真实目录名。这样做的目的，很显然是为了提高安全性

#### 生成 Content URI
从 Uri.fromFile() 转换到 FileProvider . getUriForFile();
`Uri contentUri = FileProvider.getUriForFile(this,
                BuildConfig.APPLICATION_ID + ".myprovider", myFile);
`
第二个参数是 Manifest 文件中 注册的FileProvider 设置的 authorities 属性；

第三个参数为要共享的文件，必须在path文件中添加的子目录里；

#### 授予 Content URI 访问权限
有两种授权方式：
1， 使用 context 提供的 ` grantUriPermission(package, Uri, mode_flags)` 
package  ：表示授权访问 URI 对象的其他应用包名；
Uri : 授权访问的Uri对象
mode_ flags ： 授权类型；
为 Intent类 提供读写类型的常量：
+ FLAG_GRANT_READ_URI_PERMISSION
+ FLAG_GRANT_WRITE_URI_PERMISSION

授权有效期截至到设备发生重启 或者 手动调用 `revokeUriPermission（）` 撤销授权；

2， 配合 Intent 使用，通过` setData()` 方法向` intent `对象添加` Content URI`。然后使用` setFlags() `或者` addFlags()` 方法设置读写权限;

有效期：其他应用所处的堆栈销毁，一旦授权某个组件，该应用的其他组件拥有相同的访问权限；

#### 提供 Content URI 给其他应用
+ startActivity 或者 setResult 传递；
+ 如果你需要一次性传递多个 URI 对象，可以使用 intent 对象提供的 setClipData() 方法，并且 setFlags() 方法设置的权限适用于所有 Content URIs。







   
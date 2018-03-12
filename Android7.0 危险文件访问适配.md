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


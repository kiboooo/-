Binder机制
---
> 其他IPC方式： 管道，StstemV, Socket ,文件共享 等；
> RPC: 典型的 Client/Server 模式，由客户端对服务器发出若干请求，服务器收到后根据客户端提供的参数进行操作，然后返回执行结果给 客户端；
> 基于 Binder 机制： AIDL ，Messenger, contentProvider，Socket；

常见IPC( 进程间 )通信的优缺点：
##### Bundle 
优点：简单易用，序列化传递数据；
缺点：只能传输 Bundle 支持的数据类型；
使用场景： 四大组件的进程间通信；

##### 文件共享：
优点：简单易用；
缺点：不适合高并发场景，无法做到进程间的即时通讯；
场景： 没有并发的情形，交换简单的数据；

##### AIDL
优点： 功能强大，支持一对多的串行通信，支持实时通信；
缺点：使用稍复杂，需要处理好线程同步；
场景：一对多通信 且有 RPC 需求；
只有允许不同应用的客户端用 IPC 方式调用远程方法，并且想要在服务中处理多线程时，才使用 AIDL；
如果只是需要调用远程的方法，不需要处理并发的IPC  ，就实现一个 Binder 接口；


##### Messenger
优点：功能一般，支持一对多的串行通信，支持实时通信；
缺点：不能很好处理并发，不支持 RPC ，只能传输Bundle 支持的数据类型；
场景： 低并发的一对多即时通讯，无RPC需求；
如果执行IPC只是为了传递数据，不涉及方法的调用，也不需要高并发，就使用 Messenger 来实现接口；

##### ContentProvider 
优点：数据访问功能大，支持一对多并发数据共享；
缺点：守约束的AIDL，主要提供数据源的 CRUD （增删改查）操作；
场景：一对多进程间的数据共享；
解决需要处理一对多的进程间数据的共享；

##### Socket
优点：功能强大，可通过网络传输字节流，支持一对多的并发实时通信；
缺点：繁琐
场景：网络数据交换；
实现一对多的并发实时通信；

> #####使用Binder的原因：
> 性能方面：Binder 数据拷贝只需要一次，而管道，消息队列，Socket 都需要两次，相比之下，共享内存的方式一次内存拷贝都不需要，但是实现方式较为复杂；
> 安全方面：传统的进程通信方式对通信双方的身份并没有做出严格的验证，比如 Socket 通信所使用的 ip 地址是客户端手动填入的，容易进行伪造；而 Binder 协议支持通信双方进行身份的校验；   

### IPC 原理

> ##### 什么是 iotl?
>  作为设备驱动程序中对设备的I/O通道 进行管理的函数，如果驱动程序对 ioctl 的支持，用户就能在用户程序中使用 ioctl 函数进行控制设备的I/O通道；

![enter image description here](https://camo.githubusercontent.com/5e54530f211525de5e76b3ff2b84ea5f8d5845c9/687474703a2f2f75706c6f61642d696d616765732e6a69616e7368752e696f2f75706c6f61645f696d616765732f333938353536332d613337323265653338373739333131342e706e673f696d6167654d6f6772322f6175746f2d6f7269656e742f7374726970253743696d61676556696577322f322f772f31323430)

每个 Android 的进程 ，只能运行在自己进程所拥有的虚拟地址空间。对应一个 4 GB的虚拟地址空间，其中 3G 是内存空间， 1 GB是内核空间，当然内核空间的大小是可以通过参数配置调整的；
用户空间对于不同进程之间是不共享的，但是内核空间是可以被共享的。Client 端与 Servier 端进程采用 ioctl 等方法和内核空间的驱动进行交互；

### Binder 原理
Binder通信采用 C/S 架构，从组件角度来说，包含 Client，Server ，ServiceManager 以及 binder 驱动，其中 ServiceManager 用于管理系统中的各种服务。
架构图如下：
![enter image description here](https://camo.githubusercontent.com/586ad010dd3368a3878821ff4e021fae2edf73ce/687474703a2f2f75706c6f61642d696d616765732e6a69616e7368752e696f2f75706c6f61645f696d616765732f333938353536332d356666326334383136353433633433332e6a70673f696d6167654d6f6772322f6175746f2d6f7269656e742f7374726970253743696d61676556696577322f322f772f31323430)

Client 进程：使用服务的进程；
Server进程：提供服务的进程
ServiceManager进程：ServiceManager 的作用是将字符形式的 Binder 名字转化成 Client 中对该 Binder 的引用；使得 Client 能够通过 Binder 名字获得对 Server 中 Binder 实体的引用；
Binder 驱动：驱动负责进程之间 Binder 通信的建立， Binder 在进程之间的传递，Binder引用计算管理， 数据包在进程之间的传递和交互等一系列底层支持；

#### Binder 运行机制：
+ 注册服务（addService）：Server 进程要先注册 Service 到 ServiceManager。
+ 获取服务（getService）： Client 进程使用某个 Service 前，从 ServiceManager 中获取相应的 Service 。
+ 使用服务：Client 根据得到的 Service 信息建立与 Server 进程通信的通路，然后就可以直接与 Service交互；

图中的Client ，Server，ServiceManager 之间的交互通过 Binder 驱动进行交互的，从而实现的 IPC 的通信方式；

#### 实例：
```java
//获取WindowManager服务引用
WindowManager wm = (WindowManager)getSystemService(getApplication().WINDOW_SERVICE);  
//布局参数layoutParams相关设置略...
View view=LayoutInflater.from(getApplication()).inflate(R.layout.float_layout, null);  
//添加view
wm.addView(view, layoutParams);
```
注册服务：
在Android 开机的过程中，Android 会初始化系统的各种 Service , 并且将这些 Service 向 ServiceManager 注册，由其进行管理；

获取服务：
客户端需要得到具体的 Service 直接向 ServiceManager 申请获取；获取到 Service 引用的代理对象，对数据进行一些处理操作，如上得到的就算 Window Manager的对象引用；

使用服务:
通过 这个引用向具体的服务端发送请求，服务端执行完成后就返回。如上述最后一行，调用 windowManager 的 addView 函数，触发远程调用 运行在 SystemServer 进程中的 WindowManager 的 addView 函数；

实现流程：
![enter image description here](https://camo.githubusercontent.com/bc392c6cc8355677af59d11e7f2be12a7b77bfad/687474703a2f2f75706c6f61642d696d616765732e6a69616e7368752e696f2f75706c6f61645f696d616765732f333938353536332d373237646436333031376432313133622e706e673f696d6167654d6f6772322f6175746f2d6f7269656e742f7374726970253743696d61676556696577322f322f772f31323430)

+ Client 通过获得一个Server 的代理接口，对 Server 进行调用；
+ 代理接口中的定义的方法与 Server 中定义的方法是一一对应的；
+ Client 调用某个代理接口中的方法时，代理接口的方法会将 Client 传递的参数打包成 Parcel（包）对象；
+ 代理接口 将 Parcel 发送给内核中的 binder driver；
+ Server 会读取 binder driver 中的请求数据，如果是发送给自己的，解析包 Parcel 对象中的数据，并处理返回；
+ 整个调用过程是一个同步的过程，在 server 处理的时候，Client 会被 block 住。所以不要在主线程进行调用逻辑；

###  AIDL 的使用
> AIDL 是一种接口定义语言，用于生成可以在 Android 设备上两个进程之间进行进程间通信的代码；
> 例如： 在一个Activity 中调用另一个进程中的对象，可以通过AIDL接口生成可序列化的参数，来进行进程通信

具体使用：
实现需要三部分：
1. 客户端，调用远程服务 
2. 服务端，提供服务
3. AIDL 接口用于传递参数；

#### AIDL 的默认格式：
```java
interface IBookManager {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     * 在AIDL中，演示了可以用作参数和返回值的一些基本类型。
     */
    void basicTypes(int anInt, long aLong, boolean aBoolean, 
    float aFloat,double aDouble, String aString);
}
```
 把需要传递的数据，封装成一个 beam 类，并且继承 序列化接口 Parcelabel ；
 以 book 类为例子：
```java
public class Book implements Parcelable {
    public int bookId;
    public String bookName;

    public Book() {
    }

    public Book(int bookId, String bookName) {
        this.bookId = bookId;
        this.bookName = bookName;
    }

    public int getBookId() {
        return bookId;
    }

    public void setBookId(int bookId) {
        this.bookId = bookId;
    }

    public String getBookName() {
        return bookName;
    }

    public void setBookName(String bookName) {
        this.bookName = bookName;
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(this.bookId);
        dest.writeString(this.bookName);
    }

    protected Book(Parcel in) {
        this.bookId = in.readInt();
        this.bookName = in.readString();
    }

    public static final Parcelable.Creator<Book> CREATOR = new Parcelable.Creator<Book>() {
        @Override
        public Book createFromParcel(Parcel source) {
            return new Book(source);
        }

        @Override
        public Book[] newArray(int size) {
            return new Book[size];
        }
    };
}
```
AIDL 只支持基本的数据类型，其他类型必须使用 import 导入，即使他们在同一个包内；例如上述的 book；
最终 IBookManager.aidl 的实现：
```java
// Declare any non-default types here with import statements
// 在这里声明任何非默认类型的导入语句
import com.lvr.aidldemo.Book;

interface IBookManager {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);

    void addBook(in Book book);

    List<Book> getBookList();

}
```
##### 如果是自定义的Parcelable 对象，必须创建一个和它同名的 AIDL 文件 ，并在其中声明它是 parcelable 类型 ；
```java
Book.aidl

// Book.aidl
package com.lvr.aidldemo;

parcelable Book;
```
然后 Make Project ， SDK会自动为我们生成对应的 Binder 类；
在如下的路径下：
![enter image description here](https://camo.githubusercontent.com/293a200000dca93ac2adcfb2398332e793e43099/687474703a2f2f75706c6f61642d696d616765732e6a69616e7368752e696f2f75706c6f61645f696d616765732f333938353536332d353364366130666465616665666137342e706e673f696d6167654d6f6772322f6175746f2d6f7269656e742f7374726970253743696d61676556696577322f322f772f31323430)

该接口中有个重要的内部类 Stub ，继承了 Binder 类，同时实现了 IBooklManager 接口。
```java
public static abstract class Stub extends android.os.Binder implements com.lvr.aidldemo.IBookManager{}
```

##### 服务端
在服务端首先要创建一个Service 用来监听客户端的监听请求，然后在 Service 中实现 Stub 类，并定义接口中方法的具体实现；
```java
//实现了AIDL的抽象函数
    private IBookManager.Stub mbinder = new IBookManager.Stub() {
        @Override
        public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, String aString) throws RemoteException {
            //什么也不做
        }

        @Override
        public void addBook(Book book) throws RemoteException {
            //添加书本
            if(!mBookList.contains(book)){
                mBookList.add(book);
            }
        }

        @Override
        public List<Book> getBookList() throws RemoteException {
            return mBookList;
        }
    };
```

##### 当客户端连接服务端时候，服务端会调用：
```java
public IBinder onBind(Intent intent) {
        return mbinder;
        // Stub 的实现对象；
    }
```
返回 Stub 实现对象返回客户端，该对象是一个 Binder 对象，进行进程间通信；
通过在 Service 开启另一个进程进行模拟：
```java
 <service
      android:name=".MyService"
      android:process=":remote">
      <intent-filter>
           <category android:name="android.intent.category.DEFAULT"/>
           <action android:name="com.lvr.aidldemo.MyService"/>
      </intent-filter>
</service>
```

#### 客户端：
首先将服务端工程中的 aidl 文件夹内容下的整个拷贝到客户端工程的对应位置下。
那么客户端只需要绑定服务端的 Service ：

```java
	Intent intentService = new Intent();
    intentService.setAction("com.lvr.aidldemo.MyService");
    intentService.setPackage(getPackageName());
    intentService.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
	MyClient.this.bindService(intentService, mServiceConnection, BIND_AUTO_CREATE);
```
将服务端返回的 Binder 对象转换为 AIDL 接口所属的类型，接着就可以调用 AIDL 中的方法；
` mIBookManager.addBook(new Book(18,"新添加的书")); `

#### AIDL 的工作原理：
> 由于 Bindler 机制的运行主要包括： 注册服务，获取服务 和 执行服务；
> AIDL：本质属性都是一个Interface（AIDL文件是IInterface，而Interface是继承自Interface的）

在 执行服务 的时候：
#### 1. Binder 对象的获取
Binder 对象在服务端和客户端是共享的 ，同一个 Binder 对象。在客户端通过 Binder 对象获取实现了 IInterface 接口的对象来调用远程服务，最后通过 Binder 来实现参数传递；

在服务端中获取 binder 对象 并且 保存 IInterface 接口对象：
```java
public class Binder implement IBinder{
        void attachInterface(IInterface plus, String descriptor)
        IInterface queryLocalInterface(String descriptor)
         //从IBinder中继承而来
    
}
```

Binder 具有跨进程传输能力是因为它实现了 IBinder 接口。系统为了每个实现了该接口的对象提供了跨进程传输；其中的 attachInterface  方法将 （plus ， descriptor） 作为 <Value ,key > 对 存入 Binder 对象中的一个 Map <String , IInterface > 对象中；

> ##### queryLocalInterface 获取 IInterface 对象的引用；

在服务端进程，通过实现了 private IBookManager.Stub mbinder = new IBookManager.Stub() { } 抽象类；获取Binder对象 和 IInterface 对象；

在客户端 通过 bindService 获取 Binder 对象；
```java
//获取 binder 对象
MyClient.this.bindService(intentService, mServiceConnection, BIND_AUTO_CREATE);

//获取 IInterface 对象；
private ServiceConnection mServiceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder binder) {
            //通过服务端onBind方法返回的binder对象得到IBookManager的实例，得到实例就可以调用它的方法了
            mIBookManager = IBookManager.Stub.asInterface(binder);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            mIBookManager = null;
        }
    };
```

`asInterface(binder)`  的实现如下：
```java
public static com.lvr.aidldemo.IBookManager asInterface(android.os.IBinder obj)
{
if ((obj==null)) {
return null;
}
android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
//判断是否是同一个进程对象，是就 返回 IBookManager 对象
if (((iin!=null)&&(iin instanceof com.lvr.aidldemo.IBookManager))) {
return ((com.lvr.aidldemo.IBookManager)iin);
}
//不是一个进程的对象 ，就返回 Binder 代理对象 Proxy
return new com.lvr.aidldemo.IBookManager.Stub.Proxy(obj);
}
```
#### 2. 调用服务端的的方法：
通过代理对象获取了 IInterface 对象，就可以调用具体的方法了； 
以 addBook 方法为例子，调用后 客户端线程挂起，等待唤醒：
```java
@Override public void addBook(com.lvr.aidldemo.Book book) throws android.os.RemoteException
{
..........
//第一个参数：识别调用哪一个方法的ID : Stub.TRANSACTION_addBoo
//第二个参数：Book的序列化传入数据: _data
//第三个参数：调用方法后返回的数据: _reply
//最后一个不用管
mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
_reply.readException();
}
 以下部分 主要完成对 Book 对象的序列化，最后调用 transact(处理)方法
..........
}
```
当 Proxy 对象的 transact 调用后，就把当前请求的数据 转发给 Binder 对象，那么 Binder 就会在 onTransact() 方法中收到 Proxy 对象传来的数据 ，并被 data 变量取出，进行逻辑判断，进行对应操作返回 reply 处理结果；
```java
case TRANSACTION_addBook:
{
data.enforceInterface(DESCRIPTOR);
com.lvr.aidldemo.Book _arg0;
if ((0!=data.readInt())) {
_arg0 = com.lvr.aidldemo.Book.CREATOR.createFromParcel(data);
}
else {
_arg0 = null;
}
//这里调用服务端实现的addBook方法
this.addBook(_arg0);
reply.writeNoException();
return true;
}
```
通过 transact 方法 获取 _reply 并返回结果；







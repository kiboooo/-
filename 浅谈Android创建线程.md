浅谈Android创建线程
---
>Android中创建线程和java中一样，通过java.lang.Thread类中的一系列方法进行线程的创建以及创建的约束操作。
#### 1.从java到native
启动一个线程：`.start();`方法；
> ##### 什么是native？
>  我们常说的**native**就是**Native Method**;
>  一个Native Method就是一个java调用非java代码的接口。
>  该方法的实现由**非java语言**实现，**比如C**。这个特征并非java所特有，很多其它的编程语言都有这一机制，比如在C＋＋中，你可以用extern "C"告知C＋＋编译器去调用一个C的函数。 
>  Native Method的一些特性：
>  +  在定义一个native method时，并不提供实现体（有些像定义一个java interface），因为其实现体是由非java语言在外面实现的。
>  +  与Java环境外交互，有时Java应用需要与java外面的环境交互。这是本地方法存在的主要原因，你可以想想Java需要与一些底层系统如操作系统或某些硬件交换信息时的情况。本地方法正是这样一种交流机制：它为我们提供了一个非常简洁的接口，而且我们无需去了解Java应用之外的繁琐的细节
>  + 与操作系统交互，JVM支持着Java语言本身和运行时库，它是Java程序赖以生存的平台，它由一个解释器（解释字节码）和一些连接到本地代码的库组成。然而不管怎样，它毕竟不是一个完整的系统，它经常依赖于一些底层（underneath在下面的）系统的支持。这些底层系统常常是强大的操作系统。通过使用本地方法，我们得以用Java实现了JRE的与底层系统的交互，甚至JVM的一些部分就是用C++写的，还有，如果我们要使用一些java语言本身没有提供封装的操作系统的特性时，我们也需要使用本地方法。
 
 简单实例：
```java
public class IHaveNatives
    {
      native public void Native1( int x ) ;
      native static public long Native2() ;
      native synchronized private float Native3( Object o ) ;
      native void Native4( int[] ary ) throws Exception ;
    } 
```
> + 标识符native可以与所有其它的java标识符连用，但是abstract除外。这是合理的，因为native暗示这些方法是有实现体的，只不过这些实现体是非java的，但是abstract却显然的指明这些方法无实现体
> +  一个native method方法可以返回任何java类型，包括非基本类型，而且同样可以进行异常控制。
> + native method的存在并不会对其他类调用这些本地方法产生任何影响，实际上调用这些方法的其他类甚至不知道它所调用的是一个本地方法。JVM将控制调用本地方法的所有细节。

我们看到最靠近栈顶的java方法调用的 Thread::start, 该方法内部调用了 native 方法 
```java
public synchronized void start(){
    /**
         * This method is not invoked for the main method thread or "system"
         * group threads created/set up by the VM. Any new functionality added
         * to this method in the future may have to also be added to the VM.
         *
         * A zero status value corresponds to state "NEW"
         */
         对于主方法线程或由VM创建/设置的“系统”组线程，此方法不会被调用。将来添加到这个方法中的任何新功能都可能需要添加到VM中。
        // Android-changed: throw if 'started' is true
        if (threadStatus != 0 || started)
            throw new IllegalThreadStateException();

        /* Notify the group that this thread is about to be started
         * so that it can be added to the group's list of threads
         * and the group's unstarted count can be decremented. */
         * 通知group，该线程即将启动
			这样就可以将它添加到group的线程列表中
					这个group的未开始计数可以被取消。
         
        group.add(this);

        started = false;
        try {
	       
	        重点是这个，请看下面解析！
            nativeCreate(this, stackSize, daemon);
           
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {
                /* do nothing. If start0 threw a Throwable then
                  it will be passed up the call stack */
            }
        }
    }
```

Thread::nativeCreate。如下：

```java
public synchronized void start() {
    ...
    nativeCreate(this, stackSize, daemon);
    ...
}
```

`nativeCreate(this,stackSize,Daemon)` 中我们主要关注传入的两个参数：
+  this : 即 Thread 对象自身；

+  stackSize :  指定了新创建的线程的栈大小，单位是字节（Byte）;
	+  Thread 类其中一个构造函数，接受stackSize参数;
	+  设置为0表示**忽略**之
	+  文档提到：提高stackSize会减少StackOverFlow的发生，而降低stackSize会减少OutOfMemory的发生;
	+  另外：**该参数是平台相关的，在一些平台上可能会直接被无视（有点类似Syste::gc的描述，然而目前来看gc在绝大多数平台都生效）**
+ daemon ： 表明新创建的线程是否是Daemon（守护进程；后台程序）线程；  

#### 2.从native到ART
（ART：Android Runtime：Android 4.4新增的一种应用运行模式）
native层的代码分析的是Android 8.0的ART虚拟机源码，相关文件会给出全路径。

首先我们看一下 Thread::nativeCreate 的native实现。
在`art/runtime/native/java_lang_thread.cc` 中。
主要逻辑会调用到 `art/runtime/thread.cc` 的 `art::Thread::CreateNativeThread `函数

逻辑代码如下：
```C++
void Thread::CreateNativeThread(
	JNIEnv* env, jobject java_peer, 
	size_t stack_size, bool is_daemon
	) {  // 代码1
	
	  Thread* child_thread = new Thread(is_daemon);  // 代码2
	  
	  std::unique_ptr<JNIEnvExt> child_jni_env_ext(
		      JNIEnvExt::Create(
			      child_thread,
			       Runtime::Current()->GetJavaVM(), 
			       &error_msg));
			       
  if (child_jni_env_ext.get() != nullptr) {    // 代码片段3
  
    if (success) return;
    
  }  
  // Either JNIEnvExt::Create or pthread_create(3) failed, so clean up.
  // 代码片段4
  }
```
**代码1：**
创建了` java.lang.Thread` 相对应的 native 层C++对象。

**代码2：**
java中每一个 java线程 对应一个 JniEnv 结构。
这里的JniEnvExt 就是ART 中的 JniEnv。这里源码中有一段注释
> Try to allocate a JNIEnvExt for the thread. We do this here as we might be out of memory and do not have a good way to report this on the child’s side.
> 尝试为线程分配一个JNIEnvExt。我们在这里做这个，因为我们可能已经失去了记忆，并且没有一个很好的方法来报告孩子的情况。

**JniEnv : 是一个线程相关的结构体, 该结构体代表了 Java 在本线程的运行环境 ; **
**作用：调用Java函数；操作Java对象；**


**代码片段4：**
代码3是创建线程的主要逻辑，4是执行创建流程失败的收尾逻辑。
我们先跳过3，看一下4的逻辑。
```C++
std::string msg(child_jni_env_ext.get() == nullptr ?
  StringPrintf("Could not allocate JNI Env: %s", error_msg.c_str()) :
  StringPrintf("pthread_create (%s stack) failed: %s",
               PrettySize(stack_size).c_str(), 
               strerror(pthread_create_result)));
               
ScopedObjectAccess soa(env);

soa.Self()->ThrowOutOfMemoryError(msg.c_str());
```

####  3.从ART到Pthread:
POSIX线程（POSIX threads），简称Pthreads，是线程的POSIX标准，该标准定义了创建和操纵线程的一整套API。在类Unix操作系统（Unix、Linux、Mac OS X等）中，都使用Pthreads作为操作系统的线程。
**代码3：**
```C++
if (child_jni_env_ext.get() != nullptr) {
    pthread_t new_pthread;
    pthread_attr_t attr;
    ...
    
    CHECK_PTHREAD_CALL(
	    pthread_attr_setstacksize, 
	    (&attr, stack_size),
	     stack_size);
	     
    pthread_create_result = pthread_create(&new_pthread,
                                           &attr,
                                           Thread::CreateCallback,
                                           child_thread); 
   if (pthread_create_result == 0) {
     ...     return;
    }
  }
```

可以看到，主要逻辑就是调用了pthread_create，该函数有几个参数：

>new_pthread: 新创建的线程的句柄。 

> attr: 指定了新线程的一些属性，其中包括栈大小。
 
> Thread::CreateCallback: 新创建的线程的routine函数,线程的入口函数。

> child_thread: callbac的唯一参数，此处是 native 层的 Thread 类。

对于 `pthread_create`中的实现逻辑为：bionic/lib/bionic/pthread_create.cpp：
```C++
int pthread_create(
pthread_t* thread_out,
pthread_attr_t const* attr, 
void* (*start_routine)(void*), 
void* arg)
  {

 ...  // 1. 分配栈。
 pthread_internal_t* thread = NULL; 
 void* child_stack = NULL; 
 int result = __allocate_thread(&thread_attr, &thread, &child_stack); 
 
 if (result != 0) { 
    return result;
  }
  
  ...  // 2. linux 系统调用 clone，执行真正的创建动作。
  int rc = clone(__pthread_start, child_stack, flags,
   thread, &(thread->tid), tls, &(thread->tid)); 
   
  if (rc == -1) { 
     return errno;
  }
  ...  return 0;
}
```
再来看一下分配栈中都做了些什么：
```C++
static int __allocate_thread(...)  {

  mmap_size = BIONIC_ALIGN(
  attr->stack_size + sizeof(pthread_internal_t), PAGE_SIZE);
  
  attr->stack_base = __create_thread_mapped_space(
							  mmap_size, attr->guard_size);
  
  if (attr->stack_base == NULL) { 
     return EAGAIN;
  }
  ...
}
```
我们再关注`__create_thread_mapped_space`中干了什么：
```C++
static void* __create_thread_mapped_space(
size_t mmap_size, size_t stack_guard_size) { 

 // Create a new private anonymous map.
 
 int prot = PROT_READ | PROT_WRITE; 
 //PROT_READ映射区域可被读取；
//PROT_WRITE  映射区域可被写入；
 
 int flags = MAP_PRIVATE | MAP_ANONYMOUS | MAP_NORESERVE; 
 
 //MAP_PRIVATE：对应射区域的写入操作会产生一个映射文件的复制，即私人的"写入时复制" (copy on write)对此区域作的任何修改都不会写回原来的文件内容。
 
 //MAP_ANONYMOUS: 建立匿名映射，此时会忽略参数fd，不涉及文件，而且映射区域无法和其他进程共享。

//mmap()用来将某个文件内容映射到内存中，对该内存区域的存取即是直接对该文件内容的读写。
 void* space = mmap(NULL, mmap_size, prot, flags, -1, 0); 
 
 if (space == MAP_FAILED) {
    ...   return NULL;
  }  // 代码片段1
  return space;
}
```

整个流程的主体已经很清晰：
调用mmap分配栈内存。这里mmap flag中指定了 MAP_ANONYMOUS，即匿名内存映射（mapping anonymous)。
这是在Linux中分配大块内存的常用方式。其分配的是虚拟内存，对应页的物理内存并不会立即分配，而是在用到的时候，触发内核的缺页中断，然后中断处理函数再分配物理内存。

#### 我们看一下创建的内存大小是怎么计算的。
在pthread的实现中，`mmap`分配的内存赋值给了`stack_base`，`stack_base`不光是线程执行的栈，其中还存储了线程的其他信息（如线程名，`ThreadLocal`变量等），这些信息定义在`pthread_internal_t`结构体中。
因此实际分配的内存大小是 `stack_size + sizeof(pthread_internal_t)`然后再向上取整，按照内存页大小对齐。

#### 那么我们默认的StackSize 是多少？
Java层的Thread类默认stackSize是0，传给native层也是0，于是在native层有这样一段代码。
```C++
static size_t FixStackSize(size_t stack_size) { 

 if (stack_size == 0) { 
 
   // GetDefaultStackSize 是启动art时命令行的 "-Xss=" 参数
    // Android 中没有该参数，因此为0.
    
    stack_size = Runtime::Current()->GetDefaultStackSize();
  } 
  
 // bionic pthread 默认栈大小是 1M
  stack_size += 1 * MB;

  ... 

  if (Runtime::Current()->ExplicitStackOverflowChecks()) { 
  
     // 8K
    stack_size += GetStackOverflowReservedBytes(kRuntimeISA);
  } else {   
  
   // 8K + 8K
    stack_size += Thread::kStackOverflowImplicitCheckSize +
        GetStackOverflowReservedBytes(kRuntimeISA);
  }
  ...  return stack_size;
}
```
**因此 默认的 stackSize = 1M + 8K + 8K = 1040K，和crash堆栈完全一致。**

#### 4.从 pthread 到 Linux 内核调用
这里主要涉及到 linux 的clone系统调用(SystemCall)

简单来说：

+ fork：创建新的进程，并把父进程的内存全部copy到子进程，两者的内存不共享。
+ （后来优化出了CopyOnWrite机制，几乎完全优化掉了Copy内存的开销）。


+ clone：创建新的进程，并且父进程和子进程共享内存。

因此当两个进程的内存共享之后，完全就符合“线程”的定义了。

**如果在创建线程过程中出现MMO，那就是因为进程内的虚拟内存地址空间耗尽了。**
> 我们学习编程语言的时候,基本上写的第一个程序都是输出一个hello world,  基本代码如下, 初学时,只知道使用`System.out.println`就可以将想要的内容输出到控制台上, 然而却并未关注过具体的细节,今天就先来简单了解一下`out`这个对象的赋值过程.

```java
public class Hello {
  public static void main(String[] args) {
       System.out.println("Hello world")
  }
}
```

## 实验环境

Ubuntu 16.04
debug jdk jdk21(基于[master](https://github.com/openjdk/jdk/)编译而成)
Source Insight 4.0

## out的定义

上述代码中使用到了`System.out`这个静态变量, 其定义如下:

```
/**
 * The "standard" output stream. This stream is already
 * open and ready to accept output data. Typically this stream
 * corresponds to display output or another output destination
 * specified by the host environment or user. The encoding used
 * in the conversion from characters to bytes is equivalent to
 * {@link Console#charset()} if the {@code Console} exists,
 * <a href="#stdout.encoding">stdout.encoding</a> otherwise.
 */
public static final PrintStream out = null;
```

上面可以看到这个静态变量最开始是被赋值为`null`, 如果该类被加载之后,没有其他地方再修改这个值,那么当我们调用`System.out.println()`的时候肯定会抛出`NullPointException`, 既然我们使用的时候没有问题, 那就证明这个属性在类加载之后被别的地方改动了.

我们都知道在JVM中如果想要使用一个类,则这个类必须经过加载,链接 ,初始化这几个步骤, 类加载,链接这两个阶段都是直接由JVM去控制的,只有初始化这个阶段可以插入我们自定义的逻辑,所以我初步推断out变量的值是在初始化这个步骤被修改的, 在初始化阶段JVM会调用类的`<clinit>`方法,  所以我们先去研究一下这个方法.

## 类的初始化方法

[虚拟机规范](https://docs.oracle.com/javase/specs/jvms/se19/html/jvms-2.html#jvms-2.9.1:~:text=Java%20programming%20language.-,2.9.2.%C2%A0Class%20Initialization%20Methods,-A%20class%20or)规定了一个类或者接口最多只能有一个初始化方法,并且这个方法只能由Java虚拟机去调用. 这个方法必须满足以下几个条件才能称之为初始化方法

* 名字必须为`<clinit>`
* 返回值必须为`void`
* 在JDK7之后该方法必须设置`ACC_STATIC`标志且无参

![image.png](https://b3logfile.com/file/2024/01/image-A07lBag.png)

上面是使用javap -v 反编译System类中`<clinit>`方法的字节码,可以看到在执行完`registerNatives`方法之后,将`in`,`out`,`err`这三个对象都赋值为了`null`(这也符合System源码中的定义),关于具体的字节码指令可以查看对应的[字节码](https://docs.oracle.com/javase/specs/jvms/se19/html/jvms-6.html#jvms-6.5).(JDK9之前 System类的class文件在rt.jar中, rt.jar位于JDK/JRE 的lib目录下, JDK9由于引入模块化,所以不存在rt.jar文件, 而是将所有的类文件打包为jmod, System类存在于java.base.jmod里, 这个mod的位置在jdk 目录下的jmod目录中,可以使用`jmod extract java.base.jmod` 命令对jmod进行解包, 解压缩之后的这个mod里的所有的类文件都存在于classes目录下, 之后可以使用javap 进行反编译.)

即然在`<clinit>`方法里没有修改out属性的值, 那么只能从别的地方入手, 首先out这个属性是静态成员, 所以要修改它的值只能在静态方法中去修改, 那么我们先观察一下System类的静态方法.![image.png](https://b3logfile.com/file/2024/01/image-aPo3ItV.png)

在System类中有两个`setOut`的静态方法, `setOut`中又调用了native的`setOut0`方法,那我们看一下native的`setOut0`的具体实现

![image.png](https://b3logfile.com/file/2024/01/image-gOriWYA.png)

上面的代码非常简单, 就是简单的赋值语句. 至此可以说是找到了真正的赋值逻辑.`setOut0`这个方法在System类中有两处调用, 一处是`setOut`,另一处是`initPhase1`, `setOut`这个方法在System类中是没有被调用的, 既然setOut方法在System类中没有调用, 那么只能在`initPhase1`里调用了.

## initPhase1的调用逻辑

```java
/**
* Initialize the system class.  Called after thread initialization.
 */
private static void initPhase1() {
     ...
    // FileDescriptor.in FileDescriptor.out FileDescriptor.err 对应的标准输入,标准输出,错误输出
    // 对应的linux /proc/${pid}/fd 目录下的0 1 2这三项
    FileInputStream fdIn = new FileInputStream(FileDescriptor.in);
    FileOutputStream fdOut = new FileOutputStream(FileDescriptor.out);
    FileOutputStream fdErr = new FileOutputStream(FileDescriptor.err);
    initialIn = new BufferedInputStream(fdIn);
    // 下面的三个方法即通过JNI调用设置 System类的in out err三个静态变量的值 
    setIn0(initialIn);
    setOut0(newPrintStream(fdOut, props.getProperty("stdout.encoding")));
    setErr0(newPrintStream(fdErr, props.getProperty("stderr.encoding")));
    ...		
}
```

看方法上的描述, 这个方法确实是用来初始化System类的,并且是在<clinit>方法调用之后. 这个方法没有在Java层面调用, 那只能在JVM中调用, 从JVM调用java的静态方法就需要走[JavaCalls::call_static](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/runtime/javaCalls.hpp#L239), 所以我们只需要关注从哪里调用了这个方法,就可以知道这个调用的链路.

## 调试JVM

要想知道initPhase1这个方法的具体调用逻辑, 需要对hotspot进行调试, 可以使用clion等GUI工具进行调试, 这里我为了方便,直接使用GDB进行简单的调试, 具体过程如下

```SHELL
# 启动gdb 
# -q表示不输出gdb的copyright等信息
# ~/jdk/build/linux-x86_64-server-slowdebug/jdk/bin/java 代表可执行程序的完整路径 
ght@ght-VirtualBox:~$ gdb -q ~/jdk/build/linux-x86_64-server-slowdebug/jdk/bin/java

# 开始调试 Hello.java代表了java命令的参数 jdk9之后 java命令可以直接接收源文件 不需要使用javac再提前编译
(gdb) run Hello.java

# 在JavaCalls::call_static 这个方法上打上断点 
# 仅当方法的参数 name == vmSymbols::initPhase1_name() 时该断点生效 
# name为call_static方法的一个参数
(gdb) b JavaCalls::call_static if name == vmSymbols::initPhase1_name()
# 由于之前在main方法打了断点 所以新增的这个断点的编号是2 
# call_static是重载方法 共有5处 这5个方法上都被打上了断点 有任意一个满足条件 都会被断下来
Breakpoint 2 at 0x7ffff5885818: JavaCalls::call_static. (5 locations)

# 继续执行 当断点生效时 会自动停下来
(gdb) c
Continuing.

# 线程2 命中了断点2  之后是方法的参数 以及文件的所在位置 
Thread 2 "java" hit Breakpoint 2, JavaCalls::call_static (result=0x7ffff7fc0b00, klass=0x83043318, 
    name=0x7fffdcdb6028, signature=0x7fffdcdc1ac0, args=0x7ffff7fc0a40, __the_thread__=0x7ffff002b210)
    at /home/ght/jdk/src/hotspot/share/runtime/javaCalls.cpp:249
# 方法的第一行
249	  CallInfo callinfo;

# 输出当前线程的栈
(gdb) bt
#0  JavaCalls::call_static (result=0x7ffff7fc0b00, klass=0x83043318, name=0x7fffdcdb6028, 
    signature=0x7fffdcdc1ac0, args=0x7ffff7fc0a40, __the_thread__=0x7ffff002b210)
    at /home/ght/jdk/src/hotspot/share/runtime/javaCalls.cpp:249
#1  0x00007ffff5885a32 in JavaCalls::call_static (result=0x7ffff7fc0b00, klass=0x83043318, 
    name=0x7fffdcdb6028, signature=0x7fffdcdc1ac0, __the_thread__=0x7ffff002b210)
    at /home/ght/jdk/src/hotspot/share/runtime/javaCalls.cpp:262
#2  0x00007ffff60a0228 in call_initPhase1 (__the_thread__=0x7ffff002b210)
    at /home/ght/jdk/src/hotspot/share/runtime/threads.cpp:290
#3  0x00007ffff60a0769 in Threads::initialize_java_lang_classes (main_thread=0x7ffff002b210, 
    __the_thread__=0x7ffff002b210) at /home/ght/jdk/src/hotspot/share/runtime/threads.cpp:371
#4  0x00007ffff60a11da in Threads::create_vm (args=0x7ffff7fc0e30, canTryAgain=0x7ffff7fc0d33)
    at /home/ght/jdk/src/hotspot/share/runtime/threads.cpp:644
#5  0x00007ffff5978c51 in JNI_CreateJavaVM_inner (vm=0x7ffff7fc0e88, penv=0x7ffff7fc0e90, 
    args=0x7ffff7fc0e30) at /home/ght/jdk/src/hotspot/share/prims/jni.cpp:3576
#6  0x00007ffff5979077 in JNI_CreateJavaVM (vm=0x7ffff7fc0e88, penv=0x7ffff7fc0e90, 
    args=0x7ffff7fc0e30) at /home/ght/jdk/src/hotspot/share/prims/jni.cpp:3667
#7  0x00007ffff7fcee04 in InitializeJVM (pvm=0x7ffff7fc0e88, penv=0x7ffff7fc0e90, ifn=0x7ffff7fc0f00)
    at /home/ght/jdk/src/java.base/share/native/libjli/java.c:1522
#8  0x00007ffff7fcaed7 in JavaMain (_args=0x7fffffffa880)
    at /home/ght/jdk/src/java.base/share/native/libjli/java.c:416
#9  0x00007ffff7fd27a0 in ThreadJavaMain (args=0x7fffffffa880)
    at /home/ght/jdk/src/java.base/unix/native/libjli/java_md.c:650
#10 0x00007ffff79a76ba in start_thread (arg=0x7ffff7fc1700) at pthread_create.c:333
#11 0x00007ffff74d951d in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:10
```

```c++
void Threads::initialize_java_lang_classes(JavaThread* main_thread, TRAPS) {
  ...
  // Initialize java_lang.System (needed before creating the thread)
  initialize_class(vmSymbols::java_lang_System(), CHECK);
  // Phase 1 of the system initialization in the library, 
  call_initPhase1(CHECK);
  ...
}
```

从上面的调试信息中可以看出,jvm在启动时, 在create_vm 这个步骤中, jvm调用了`initialize_java_lang_classes` 去初始化`java/lang`包下的这些类, 包括`String`,`System`,`Class`等, 初始化其实就是去调用类的`<clinit>`方法, 在执行完`<clinit>`方法之后, 再去调用System类中的`initPhase1`方法, 在java层将`in`,`out`,`err`对象构造出来, 之后再经过jni将这些对象再赋值给变量. 至此完成类的初始化工作, 从而使我们在代码中免受NPE的困扰.

## 总结

以上即是System类中out对象的赋值过程, 先是从JVM层调用到了java层, 在java层做了一些准备工作之后,又通过JNI调用回到了JVM层, 最终完成out对象的赋值. 基本流程如下.
![diagram8467815604994034344.png](https://b3logfile.com/file/2024/01/diagram-9183511484174953293-oMXCiQX.png)

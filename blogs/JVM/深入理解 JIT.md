---
深入 JVM 即时编译器 JIT，优化 Java 编译
---

### 目录

1. 前言
2. 类编译加载执行过程

### 前言

>摘自：https://time.geekbang.org/column/article/106953

说到编译，你一定会想到 java 文件被编译成 class 文件的过程，这个编译我们一般称为前端编译。Java 的编译和运行过程非常复杂，除了前端编译，还有运行时编译。由于机器无法直接运行 Java 生成的字节码，所以在运行时，JIT 或解释器会将字节码转换成机器码，这个过程就叫运行时编译。

类文件在运行时被进一步编译，它们可以变成高度优化的机器代码，由于 C/C++ 编译器的所有优化都是在编译期间完成的，运行期间的性能监控仅作为基础的优化措施则无法进行，例如调用频率预测、分支频率预测、裁剪未被选择的分支等，而 Java 在运行时的再次编译，就可以进行基础的优化措施。因此，JIT 编译器可以说是 JVM 中运行时编译最重要的部分之一。

### 类编译加载执行过程

在这之前，我们先了解一下 Java 从编译到运行的整个过程。如下图：
![](https://i.loli.net/2019/07/12/5d27ecc851a8286498.jpg)

#### 类编译

Java 源文件是通过 Javac 编译生成 class 文件，前端编译的过程其实是非常复杂的，包括词法分析、语法分析、填充符号表、注解处理、语义分析以及生成 class 文件，这个过程我们不用过多关注。只需要记住，编译后的字节码文件主要包括常量池和方法表集合这两部分就可以了。

常量池主要记录的是类文件中出现的字面量以及符号引用。字面常量包括字符串常量例如 String str = "abc"，其中 abc 就是常量），声明为 final 的属性以及一些基本类型的属性。符号引用包括类和接口的全限定名、类引用、方法引用以及成员变量引用（例如 String str = "abc"，其中 str 就是成员变量引用）等。

方法表集合中主要包含一些方法的字节码、方法访问权限、方法名索引、描述符索引、JVM 执行指令以及属性集合等。

#### 类加载

当一个类被创建实例或者被其他对象引用时，虚拟机在没有加载过该类的情况下，会通过类加载器将字节码文件加载到内存中。

不同的实现类由不同的类加载器加载，JDK 中的本地方法类一般由根加载器（BootstrapLoader）加载进来，JDK 中的内部实现的扩展类一般由扩展加载器（ExtClassLoader）实现加载，而程序中的类文件则由系统类加载器（AppClassLoader）实现加载。

在类加载后，class 类文件中的常量池信息以及其他数据会被保存到 JVM 内存的方法区中。

#### 类连接

类在加载进来之后，会进行连接、初始化，最后才会被使用。在连接过程中，又包括验证、准备和解析三个部分。

##### 验证

验证类符合 Java 规范和 JVM 规范，在保证符合规范的前提下，避免危害虚拟机安全。

##### 准备

为类的静态变量分配内存，初始化为系统的初始值。对于 final static 修饰的变量，直接赋值为用户的定义值。例如，private final static int value = 123，会在准备阶段分配内容，并初始化值为 123，而如果是 private static int value = 123，这个阶段 value 的值仍然为 0。

##### 解析

将符号引用转为直接引用的过程。在编译时，Java 类并不知道所引用的类的实际地址，因此只能使用符号引用来代替。类结构文件的常量池中存储了符号引用，包括类和接口的全限定名、类引用、方法引用以及成员变量引用等。如果要使用这些类和方法，就需要把它们转化为 JVM 可以直接获取的内存地址或指针，即直接引用。

#### 类初始化

类初始化阶段是类加载过程的最后阶段，在这个阶段中，JVM 首先将执行构造器 \<clinit> 方法，编译器会将在 java 文件时，收集所有类初始化代码，包括静态变量赋值语句、静态代码块、静态方法，收集在一起成为 \<clinit>() 方法。

初始化类的静态变量和静态代码块为用户自定义的值，初始化的顺序和 Java 源码从上到下的顺序一致。子类初始化时会首先调用父类的 \<clinit>() 方法，再执行子类的 \<clinit>() 方法。

JVM 会保证 \<clinit>() 方法的线程安全，保证同一时间只有一个线程执行。JVM 在初始化执行代码时，如果实例化一个新对象，会调用 \<init> 方法对实例变量进行初始化，并执行对应的构造方法内的代码。

### 即时编译

初始化完成后，类在调用执行过程中，执行引擎会把字节码转为机器码，然后在操作系统中才能执行。在字节码转换为机器码的过程中，虚拟机中还存在着一道编译，那就是即时编译。

最初，虚拟机中的字节码是由解释器（Interpreter）完成编译的，当虚拟机发现某个方法或代码块的运行特别频繁的时候，就会把这些代码认定为 “热点代码”。

为了提高热点代码的执行效率，在运行时，即时编译器会把这些代码编译成与本地平台相关的机器码，并进行各层次的优化，然后保存到内存中。

#### 即时编译器类型

在 HotSpot 虚拟机中，内置了两个 JIT，分别为 C1 编译器和 C2 编译器，这两个编译器的编译过程是不一样的。

C1 编译器是一个简单快速的编译器，主要的关注点在于局部性的优化，适用于执行时间较短或对启动性能有要求的程序，例如，GUI 应用对界面启动速度就有一定要求。

C2 编译器是为长期运行的服务器端应用程序做性能调优的编译器，适用于执行时间较长或对峰值性能有要求的程序。根据各自的适配性，这两种即时编译也被称为 Client Compiler 和 Server Compiler。

在 Java7 之前，需要根据程序的特性来选择对应的 JIT，虚拟机默认采用解释器和其中一个编译器配合工作。

Java7 引入了分层编译，这种方式综合了 C1 的启动性能优势和 C2 的峰值性能优势，我们也可以通过参数 “-client” ”-server“ 强制指定虚拟机的即时编译模式。分层编译将 JVM 的执行状态分为了 5 个层次：

* 第 0 层

  程序解释执行，默认开启性能监控（Profiling），如果不开启，可触发第二层编译；

* 第 1 层

  可称为 C1 编译，将字节码编译为本地代码，进行简单、可靠的优化，不开启 Profiling；

* 第 2 层

  也称为 C1 编译，开启 Profiling，仅执行带方法调用次数和循环回边执行次数 profiling 的 C1 编译；

* 第 3 层

  也称为 C1 编译，执行所有带 Profiling 的 C1 编译；

* 第 4 层

  可称为 C2 编译，也是将字节码编译为本地代码，但是会启动一些编译耗时较长的优化，甚至会根据性能监控信息进行一些不可靠的激进优化。

在 Java8 中，默认开启分层编译，除了这种默认的混合编译模式，我们还可以使用 “-Xint” 参数强制虚拟机运行在只有解释器的编译模式下，这时 JIT 完全不介入工作；我们还可以使用参数 “-XComp” 强制虚拟机运行于只有 JIT 的编译模式下。

#### 热点探测

在 HotSpot 虚拟机中的热点探测是 JIT 优化的条件，热点探测是基于计数器的热点探测，采用这种方法的虚拟机会为每个方法建立计数器统计方法的执行次数，如果执行次数超过一定的阈值就认为它是 “热点方法”。

虚拟机为每个方法准备了两类计数器：方法调用计数器和回边计数器。在确定虚拟机运行参数的前提下，这两个计数器都有一个确定的阈值，当计数器超过阈值溢出了，就会触发 JIT 编译。

##### 方法调用计数器

用于统计方法被调用的次数，方法调用计数器的默认阈值在 C1 模式下是 1500，在 C2 模式是在 10000 次。当方法计数器和回边计数器之和超过方法计数器阈值时，就会触发 JIT 编译。

##### 回边计数器

对于统计一个方法中循环体代码执行的次数，在字节码中遇到控制流向后跳转的指令称为 “回边”。在不开启分层编译的情况下，C1 默认为 13995，C2 默认为 10700。

建立回边计数器的主要目的是为了触发 OSR 编译，即栈上编译。在一些循环周期比较长的代码段中，当循环达到回边计数器阈值时，JVM 会认为这段是热点代码，JIT 编译器就会将这段代码编译成机器语言并缓存，在该循环时间段内，会直接将执行代码替换，执行缓存的机器语言。

### 编译优化技术




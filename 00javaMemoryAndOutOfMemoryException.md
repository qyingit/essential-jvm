JVM运行时数据区域

![](C:/Users/Administrator.WIN-C9942BDA1C2/AppData/Local/YNote/data/floor07@126.com/c75c6e27230247bcb8045a8faf6f249b/clipboard.png)![](/assets/java内存.png)

| 区域名称 | 作用 | 是否线程私有 | 是否会内存溢出 | 溢出原因 |
| :--- | :--- | :--- | :--- | :--- |
| 程序计数器 | 当前线程所执行的字节码的行号的指示器。每个线程都有独立的程序计数器 | 是 | 否 |  |
| Java虚拟机栈 | 与线程同生命周期存储局部变量表，操作数栈动态链接，方法出口，对象引用等。局部变量表存储基本数据类型boolean,int，float,long,double，byte,short.long,double占2其余占1局部空间 | 是 | 是 | 2种异常1.StackOverflowError,线程请求的栈深度大于虚拟机所允许的深度。2.OutOfMemoryError栈扩展时申请到不足够的内存。 |
| 本地方法栈 | Java虚拟机栈类似为Native服务 | 是 | 是 | 2种异常1.StackOverflowError,线程请求的栈深度大于虚拟机所允许的深度。2.OutOfMemoryError栈扩展时申请到不足够的内存。 |
| java堆 | 存放对象实例以及数组。GC堆。逻辑连续，物理不连续通过-Xmx和-Xms来控制。 | 否 | 是 | 堆中没有内存完成实例分配时，并且堆无法再扩展时。将抛出OutOfMemoryError |
| 方法区 | non-heap，“永久代”受到-XX：MaxPermSize | 否 | 是 | 当方法区无法满足内存分配需求时OutOfMemoryError |
| 运行时常量池 | 字面量，符号引用。Java语言不一定要求只有编译才会产生常量String的interm（）是方法区的一部分JDK1.7将它从方法区移除，使用直接内存 | 否 | 是 | 受方法区限制.当常量池无法再申请到内存时会内存溢出 |
| 直接内存 | 即本机直接内存，不受JVM控制。 | 否 | 是 | JVM各内存区域总和大于物理内存时，当动态扩展时会OutOfMemoryError |

创建对象：

new指令时，

1.首先将去检查这个指令的参数是否能在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否被加载，解析，初始化。若没有进行类加载过程。

2.之后类加载检查通过后，虚拟机为新生对象分配内存。对象的大小在类加载完成后便可完全确定。

（等同于Java堆中划分一部分内存）

分配内存方式：

a. “指针碰撞” ，内存是物理连续的，Serial,ParNew带Compact过程的收集器时。（已使用 指针 未使用）

b.“空间列表”，内存非物理连续的，CMS这种基于Mark-Sweep算法的收集器。（空间列表存储可用的内存地址）

分配内存在并发情况下非线程安全，2中解决方式：

a.CAS+失败重试

b.TLAB为每个线程分配单独的空间，-XX:+/-UseTLAB

3.虚拟机进行必要的设置。（这个对象是那个类的实例，如何找到类的元数据，对象的哈希码，对象的GC的分代年龄，这些信息存储在对象的对象头中）

4.虚拟机角度完成，之后执行init方法

---

对象布局

1. HotSpot消息头

包括两部分：

第一部分 Mark Word;对象自身运行时数据，包括：

哈希码，GC分代年龄，锁状态标示、线程持有的锁、偏向线程ID

偏向时间戳等。32和64位机子上，

未压缩的情况下分别为32bit和64bit。

考虑到空间效率，Mark Word被设计成一个非固定的数据结构。

例如：

对象处于未锁定的状态下。

32位的Mark Word，25bit存储对象哈希码，4bit分代年龄，2bit 存储锁标示位，

1 bit固定为0.

具体如下：

| 存储内容 | 标志位 | 状态 |
| :--- | :--- | :--- |
| 对象哈希码，对象分代年龄 | 01 | 未锁定 |
| 指向锁记录的指针 | 00 | 轻量级锁定 |
| 指向重量级锁的指针 | 10 | 重量级锁定 |
| 空，不需要记录信息 | 11 | GC标记 |
| 偏向线程ID，偏向时间戳，对象分代年龄 | 01 | 可偏向 |

第二部分：类型指针，即指向它的类元数据的指针，

虚拟机通过这个指针来确定这个对象是那个类的实例。（并非所有虚拟机都保存类型指针）

如果一个对象是Java数组，那对象头中还必须记录数组的长度，虚拟机可以通过Java

对象的元数据确定Java对象的大小。但是从数组的元数据中无法确定数组的大小。

1. 实例数据部分：存储顺序

受FieldAllocationStyle影响，相同宽度的字段总是被分配到一起，之后父类的变量会出现在子类之前，CompactFileds为true\(默认为true\)时，子类比较窄的变量可能会插入到父类变量空隙中。

3.对齐填充，非必须存在，Hotspot VM 要求对象头部分是8字节的倍数（1倍或2倍），对象大小必须为8字节的整数倍。就需要对齐填充。

---

对象访问

两种模式：

1.句柄和直接指针

句柄，垃圾回收时，对象移动时，reference不会变。

直接指针，访问更快，少一次指针定位，（Sum hotSpot使用）


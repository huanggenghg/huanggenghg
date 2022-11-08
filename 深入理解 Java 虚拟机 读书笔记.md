## 深入理解 Java 虚拟机 读书笔记

### 一、Java 虚拟机运行时数据区

![Java 虚拟机运行时数据区](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Java%20%E8%99%9A%E6%8B%9F%E6%9C%BA%E8%BF%90%E8%A1%8C%E6%97%B6%E6%95%B0%E6%8D%AE%E5%8C%BA.drawio.png)

- 程序计数器：唯一无 OOM Error 情况的区域。
- Java 虚拟机栈：每个方法被执行的时候，都会同步创建一个栈帧（Stack Frame）用于存储**局部变量表、操作数栈、动态连接、方法出口**等信息。局部变量表存放了编译期可知的各种 Java 虚拟机基本数据类型、对象引用和 returnAddress 类型。异常情况：StackOverFlowError 异常，OOM 异常。
- 本地方法栈：类似 Java 虚拟机栈，为虚拟机执行本地（Native）方法服务。异常情况：StackOverFlowError 异常，OOM 异常。
- Java 堆：存放对象实例，也为“GC 堆”。异常情况：OOM 异常。
- 方法区：存储已被虚拟机加载的类型信息、常量、静态变量、即时编译后的代码缓存等数据。异常情况：OOM 异常。
- 运行时常量池：方法区的一部分，存放 Class 文件中的常量池表（用于存放编译期生成的各种字面量与符号引用）。异常情况：OOM 异常。
- *直接内存：并不是虚拟机运行时数据区的一部分，也不是《Java 虚拟机规范》中定义的内存区域。JDK1.4 引入 NIO 的一种基于通道与缓存区的 I/O 方式，使得 Native 函数库可直接进行堆外内存分配。异常情况：OOM 异常。*

### 二、垃圾收集器与内存分配策略

怎么判定哪些对象需要被回收：

- 引用计数法：引用计数器。问题：相互引用无法被回收；
- 可达性分析算法：GC Roots search on Reference Chain，向下搜索过程所形成的“引用链”信息

可作为 GC Roots 的对象：

- 在虚拟机栈（栈帧中的本地变量表）中引用的对象，譬如各个线程被调用的方法堆中使用到的参数、局部变量、临时变量等。
- 在方法区中类静态属性引用的对象，譬如 Java 类的引用类型静态变量。
- 在方法区中常量引用的对象，譬如字符串常量池（String Table）里的引用。
- 在本地方法栈中 JNI（即通常所说的 Native 方法）引用的对象。
- Java 虚拟机内部的引用，如本地数据类型对应的 Class 对象，一些常驻的异常对象（比如 NullPointException、OutOfMemoryError）等，还有系统类加载器。
- 所有被同步锁（synchronized 关键字）持有的对象。
- 反映 Java 虚拟机内部情况的 JMXBean、JVMTI 中注册的回调、本地代码缓存等。
- 还可有其他对象“临时性”加入

引用：

- 强引用：`Object obj = new Object()`
- 软引用：`SoftReference`实现，在系统将要发生内存溢出异常前，会列入回收范围进行第二次回收，如果这次回收还没有足够的内存，才会抛出内存溢出异常
- 弱引用：`WeakReference`实现，比软引用更弱。弱引用对象只能生存到下一次垃圾收集发生为止。
- 虚引用：`PhantomReference`实现，最弱，为一个对象设置虚引用关联的唯一目的只是为了能在这个对象被收集器回收时能收到一个系统通知。

`finalize()`是对象在第一次被回收标记后逃脱死亡命运的最后一次机会，只要重新与引用链上的任一对象建立关联即可。

回收方法区：回收性价比低、回收内容为废弃的常量和不再使用的类型。

> 废弃的常量：如“java”曾经进入常量池中，但当前系统中已没有一个字符串对象的值是“java”，换句话说，已经没有任何字符串对象引用常量池中的“java”常量，且虚拟机中也没有其他地方引用这个字面量。
>
> 不再被使用的类（不一定会进行回收，虚拟机参数进行控制）：
>
> - 该类所有的实例都已被回收，也就是 Java 堆中不存在该类及其任何派生子类的实例。
> - 加载该类的类加载器已被回收。（很难达成。）
> - 该类对应的 java.lang.Class 对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。

垃圾收集算法：

- 标记-清除算法：内存碎片问题
- 标记-复制算法：内存使用率低，多数对象存活时会产生大量复制的开销
- 标记-整理算法：老年代移动存活对象，“Stop The World”

> 分代收集理论：
>
> 1. 弱分代假说：绝大多数对象都是朝生夕灭的。
> 2. 强分代假说：熬过越多次垃圾收集过程的对象就越难以消亡。
> 3. 跨代引用假说：跨代引用相对于同代引用来说仅占极少数。

### 三、虚拟机类加载机制

![类的生命周期](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E7%B1%BB%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F.drawio.png)

加载、验证、准备、初始化和卸载这五个阶段的顺序是确定的，类型的加载过程必须按照这种顺序按部就班地开始，而解析阶段则不一定：它在某些情况下可以在初始化阶段之后再开始，只是为了支持 Java 语言的运行时绑定特性（也称为动态绑定或晚期绑定）。

加载阶段：

1. 通过一个类的全限定名来获取定义此类的二进制字节流
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构
3. 在内存中生成一个代表这个类的 java.lang.Class 对象，作为方法区这个类的各种数据的访问入口

验证阶段：确保 Class 文件的字节流中包含的信息符合《Java 虚拟机规范》的全部约束要求，保证这些信息被当做代码运行后不会危害虚拟机自身的安全

1. 文件格式验证
2. 元数据验证
3. 字节码验证
4. 符号引用验证

准备阶段：为类中定义的变量（即静态变量，被 static 修饰的变量）分配内存并设置类变量初始值的阶段。仅包括类变量，不包括实例变量，实例变量将会在对象实例化时随着对象一起分配在 Java 堆中。初始值“通常情况”下是数据类型的零值。

解析：是 Java 虚拟机将常量池内的符号引用替换为直接引用的过程

- 符号引用：符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可。
- 直接引用：直接引用是可以直接指向目标的指针、相对偏移量或者一个能间接定位到目标的句柄。

类或接口的解析
字段解析
方法解析
接口方法解析

初始化 `<clinit>()`

类加载器：

对于任意一个类，都必须由它的类加载器和这个类本身一起共同确立其在 Java 虚拟机中的唯一性，每一个类加载器，都拥有一个独立的类名称空间

类加载器的类型：

Java 虚拟机的角度：启动类加载器（Bootstrap ClassLoader），这个类加载器使用 C++ 语言实现，是虚拟机自身的一部分；其他所有的类加载器，都由 Java 语言实现，独立存在于虚拟机外部，并且全部继承自抽象类 java.lang.ClassLoader。

Java 开发人员的角度：三层类加载器、双亲委派的类加载架构。

三层类加载器：

- 启动类加载器（Bootstrap Class Loader）：负责加载存放在 <JAVA_HOME>\lib，或者 -Xbootclasspath 参数指定路径存放的能够被识别的类库。
- 扩展类加载器（Extension Class Loader）：在类 sun.misc.Launcher$ExtClassLoader 中以 Java 代码的形式实现的。负责加载 <JAVA_HOME>\lib\ext 目录中，或者被 java.ext.dirs 系统变量所指定路径的所有的类库。
- 应用程序类加载器（Application Class Loader）：由 sun.misc.Launcher$AppClassLoader 来实现。负责加载用户类路径（ClassPath）上所有的类库。

![类加载器双亲委派模型](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8%E5%8F%8C%E4%BA%B2%E5%A7%94%E6%B4%BE%E6%A8%A1%E5%9E%8B.drawio.png)

双亲委派模型的工作过程是：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，二十把这个请求委托给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的类加载请求最终都应该传送到最顶层的启动类加载器中，只有当父类加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会去自己完成加载。

``` java
protected synchronized Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    // 首先，检查请求的类是否已经被加载过了
    Class c = findLoadedClass(name);
    if(c == null) {
        try {
            if(parent != null) {
            	c = parent.loadClass(name, false);    
            } else {
                c = findBootstrapClassOrNull(name);
            }
        } catch (ClassNotFoundException e) {
            // 如果父类加载器抛出 ClassNotException，说明父类加载器无法完成加载请求
        }
        if(c == null) {
            // 在父类加载器无法加载时，再调用本身的 findClass 方法来进行类加载
        	c = findClass(name);
        }
    }
    if(resolve) {
        resolveClass(c);  
    }
    return c;
}
```

JDK 9 类加载的委派关系发生了变动，当平台及应用程序类加载器收到类加载请求，在委派给父加载器加载前，要先判断该类是否能够归属到某一个系统模块中，如果可以找到这样的归属关系，就要先优先派给负责那个模块的类加载器完成加载，也许这可以算是对双亲委派的第四次破坏。

![JDK 9 后的类加载器委派关系](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/JDK%209%20%E5%90%8E%E7%9A%84%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8%E5%A7%94%E6%B4%BE%E5%85%B3%E7%B3%BB.drawio.png)

### 四、高效并发

硬件的效率与一致性：绝大多数的运算任务都不可能只靠处理器“计算”就能完成，由于计算机的存储设备与处理器的运算速度存在着几个数量级的差距，所以现代计算机系统都不得不加入一层或多层读写速度尽可能接近处理器运算速度的高速缓存（Cache）来作为内存与处理器之间的缓冲：**将运算需要使用的数据复制到缓存中，让运算能够快速进行，当运算结束后再从缓存同步回内存之中，这样处理器就无须等待缓慢的内存读写了。**

但这会引入缓存一致性（Cache Conherence）问题。为了解决缓存一致性问题，需要各个处理器访问缓存时都遵循一些协议，在读写时要根据协议进行操作。

“内存模型”可以理解为在特定的操作下，对特定的内存或高速缓存进行读写访问的过程抽象。

除了增加高速缓存之外，为了使处理器内部的运算单元能够尽量被充分利用，处理器可能会对输入代码进行乱序执行（Out-Of-Order Execution）。与处理器的乱序执行优化类似，Java 虚拟机的即时编译器也有指令重排序（Instruction Reorder）优化。

![处理器、高速缓存、主内存间的交互关系](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E5%A4%84%E7%90%86%E5%99%A8%E9%AB%98%E9%80%9F%E7%BC%93%E5%AD%98%E4%B8%BB%E5%86%85%E5%AD%98%E9%97%B4%E7%9A%84%E4%BA%A4%E4%BA%92%E5%85%B3%E7%B3%BB.drawio.png)

Java 内存模型的主要目的是定义程序中各种变量的访问规则，即关注在虚拟机中把变量值复制存储到内存和从内存中取出变量值这样的底层细节（此处的变量不包括线程私有概念的变量）。

Java 的内存模型规定了所有的变量都存储在主内存（Main Memory）中，每条线程有自己的工作内存，线程的工作内存中保存了被该线程使用的变量的主内存副本，线程对变量所有操作（读取、赋值等）都必须在工作内存中进行，而不是直接读写主内存中的数据。

![线程、主内存、工作内存三者的交互关系](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E7%BA%BF%E7%A8%8B%E4%B8%BB%E5%86%85%E5%AD%98%E5%B7%A5%E4%BD%9C%E5%86%85%E5%AD%98.drawio.png)

内存间交互操作中，原子的、不可再分的（double 和 long 类型的变量，load\stroe\read\write 操作在某些平台上允许有例外）操作：lock、unlock、read、load、use、assign、store、write。

把一个变量从主内存拷贝到工作内存中，那就要顺序执行 read 和 load 操作；如果要把变量从工作内存同步回主内存，要顺序执行 store 和 write 操作。要求顺序执行，但不要求是连续执行的。

执行上述八种操作时必须满足的规则：

- 不允许 read 和 load、store 和 write 操作 之一单独出现。
- 不允许一个线程丢弃它最近的 assign 操作，即变量在工作内存中改变了之后必须把该变化同步回主内存。
- 不允许一个线程无原因地（没有发生任何 assign 操作）把数据从线程的工作内存同步回主内存中。
- 一个新的变量只能在主内存中“诞生”，不允许在工作内存中直接使用一个未被初始化（load 或 assign）的变量，换句话说就是对一个变量实施 user、store 操作之前，必须先执行 assign 和 load 操作。
- 一个变量在同一时刻只允许一条线程对其进行 lock 操作，但 lock 操作可以被同一线程重复执行多次，多次执行 lock 后，只有执行同次数的 unlock 操作，变量才会被解锁。
- 如果对一个变量执行 lock 操作，将会清空工作内存中此变量的值，在执行引擎使用这个变量之前，需要重新执行 load 或 assign 操作以初始化变量的值。
- 如果一个变量事先没有被 lock 操作锁定，那就不允许对它执行 unlock 操作，也不允许去 unlock 一个呗其他线程锁定的变量。
- 对一个变量执行 unlock 操作之前，必须先把此变量同步回主内存中（执行 store、write 操作）。

volatile：最轻量级的同步机制，**保证修饰变量对所有线程的可见性（使用之前刷新），禁止指令重排序优化（内存屏障）**。某些情况下需要，该如何保证原子性：运算结果并不依赖变量的当前值，或者能够确保只有单一的线程修改变量的值；变量不需要与其他的状态变量共同参与不变约束。

针对 long 和 double 型变量的特殊规则：**允许虚拟机将没有被 volatile 修饰的 64 位数据的读写操作划分为两次 32 位的操作来进行**，所谓的“long 和 double 的非原子性协定”。在多线程读写此类变量时，可能某些线程会读到一个既不是原值，也不是其他线程修改值的代表了“半个变量”的数值。但在目前主流虚拟机中，不需要考虑这个原因刻意把用到的 long 个 double 变量专门声明为 volatile。

先行发生原则：Java 内存模型中定义的两项操作之间的偏序关系，前面操作所产生的影响会被后续操作所观察到，“并不是时间上的先发生”。

- 程序次序规则（Program Order Rule）：在一个线程内，按照**控制流顺序**，书写在前面的操作先行发生于书写在后面的操作。
- 管程锁定规则（Monitor Lock Rule）：一个 unlock 操作先行发生于后面对同一个锁的 lock 操作。
- volatile 变量规则（Volatile Variable Rule）：对一个 volatile 变量的写操作先行发生于后面对这个变量的读操作。
- 线程启动规则（Thread Start Rule）：Thread 对象的 start() 方法先行发生于此线程的每一个动作。
- 线程终止规则（Thread Termination Rule）
- 线程中断规则（Thread Interruption Rule）
- 对象终结规则（Finalizer Rule）：一个对象的初始化完成（构造函数执行结束）先行发生于它的 finalize() 方法的开始。
- 传递性（Transitivity）

线程的实现主要有三种方式：使用内核线程实现（1:1 实现），使用用户线程实现（1:N 实现 ），使用用户线程加轻量级进程混合实现（N:M 实现）

轻量级进程与内核线程之间 1:1 的关系：

![轻量级进程与内核线程之间 1:1 的关系](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E5%86%85%E6%A0%B8%E7%BA%BF%E7%A8%8B.drawio.png)

进程与用户线程之间 1:N 的关系：

![进程与用户线程之间 1:N 的关系](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E8%BF%9B%E7%A8%8B%E4%B8%8E%E7%94%A8%E6%88%B7%E7%BA%BF%E7%A8%8B.drawio.png)

用户线程与轻量级进程之间 M:N 的关系：

![用户线程与轻量级进程之间 M:N 的关系](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E7%94%A8%E6%88%B7%E7%BA%BF%E7%A8%8B%E4%B8%8E%E8%BD%BB%E9%87%8F%E7%BA%A7%E8%BF%9B%E7%A8%8B.drawio.png)

Java 的线程调度主要方式有两种，分别是协同式（Cooperative Threads-Scheduling）线程调度和抢占式（Preemptive Threads-Scheduling）线程调度。

线程状态转换关系：

![线程状态转换关系](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E7%BA%BF%E7%A8%8B%E7%8A%B6%E6%80%81%E8%BD%AC%E6%8D%A2%E5%85%B3%E7%B3%BB.drawio.png)

### 五、线程安全与锁优化

线程安全的定义：当多个线程同时访问一个对象时，如果不用考虑这些线程在运行环境下的调度和交替执行，也不需要进行额外的同步，或者在调用方进行任何其他的协调操作，调用这个对象的行为都可以获得正确的结果，那就称这个对象时线程安全的。

Java 语言的线程安全：不可变、绝对线程安全、相对线程安全、线程兼容和线程对立。

线程安全的实现方法：互斥同步、非阻塞同步。

互斥同步方式：临界区（Critical Section）、互斥量（Mutex）和信号量（Semaphore）都是常见的互斥实现方式。

在 Java 中，最基本的互斥同步手段就是`synchronized`关键字，这是一种块结构（Block Structured）的同步语法，同步块前后会有 monitorenter 和 monitorexit 这两个字节码指令，这两个字节码指令都需要一个 reference 类型的参数来指明要锁定和解锁的对象。如果 Java 源码中的 synchronized 明确指定了对象参数，那就以这个对象的引用作为 reference；**如果没有明确指定，那将根据 synchronized 修饰的方法类型（如实例方法或类方法），来决定是取代码所在的对象实例还是取类型对应的 Class 对象来作为线程要持有的锁。**

在执行 monitorenter 时，尝试获取锁，获取到或已持有，锁的计数器值 +1，而执行 monitorexit 会 -1。一旦计数器值为 0，锁随即就被释放。

- `synchronized`修饰的同步块对同一条线程来说是可重入的。
- `synchronized`修饰的同步块在持有锁的线程执行完毕并释放锁之前，会无条件地阻塞后面其他线程进入。

持有锁是一个重量级（Heavy-Weight）的操作。在主流 Java 虚拟机实现中，Java 的线程是映射到操作系统的原生内核线程之上的，如果要阻塞或唤醒一条线程，则需要操作系统来帮忙完成，这就不可避免地陷入用户态到核心态的转换中，进行这种状态转换需要耗费很多处理器的时间。

RerntrantLock 相比 synchronized 增加了一些高级功能：等待可中断、公平锁、锁绑定多个条件。

非阻塞同步：乐观并发策略，不管风险，先进行操作，如果没有其他线程争用共享数据，操作成功；若被争用，产生冲突，则进行其他补偿措施，最常用的补偿措施是不断地重试，直至出现没有竞争的共享数据为止。

“硬件指令集的发展”：

- 测试并设置（Test-And-Set）
- 获取并增加（Fetch-and-Increment）
- 交换（Swap）
- 比较并交换（Compare-and-Swap，CAS）
- 加载链接 / 条件存储（Load-Linked / Store-Conditional，LL/SC）

锁优化：

- 自旋锁与自适应自旋：挂起线程和恢复线程的操作都需要转入内核态中完成，在如今绝大多数电脑和多核处理器系统中，共享数据的锁定状态只会持续很短一段时间，为了这段时间去挂起和恢复线程并不值得。让后面请求锁而不得的线程等待一下，让其执行一个忙循环（自旋）。
- 锁消除：虚拟机及时编译器在运行时，对一些代码要求同步，但是虚拟机又检测到不可能存在共享数据竞争的锁进行消除。
- 锁粗化：一系列的连续操作都对同一个对象反复加锁和解锁，只需加锁一次即可。
- 轻量级锁
- 偏向锁


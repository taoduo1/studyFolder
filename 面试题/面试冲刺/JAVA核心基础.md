一、Java核心基础
JVM与性能调优

内存模型（堆、栈、方法区）、垃圾回收算法（CMS、G1、ZGC区别）

堆（Heap）：
    存储对象实例和数组。
    所有线程共享。
    是GC的主要区域。
    分为新生代（Eden、Survivor区）和老年代。

栈（Stack）：
    存储局部变量、方法参数和返回值。
    每个线程私有。
    栈帧用于方法调用，包含局部变量表、操作数栈、动态链接和方法返回地址。
    栈溢出通常由递归调用过深引起。

方法区（Method Area）：
    存储类信息、常量、静态变量、JIT编译后的代码。
    所有线程共享。
    在JDK 8之前称为“永久代”（PermGen），之后改为“元空间”（Metaspace），使用本地内存。

问题：如果线上服务出现OOM，如何快速定位问题？
回答：
    查看日志，通过异常提示，确认OOM类型（堆、栈、元空间）。
    使用工具（如jmap）生成dump文件。
    使用MAT或JProfiler分析堆转储文件，查找内存泄漏或大对象。
    dump文件里还有线程的堆栈跟踪信息，显示代码的执行路径，查看线程信息，分析代码走到哪里出现的oom
    结合代码逻辑，修复问题。

垃圾回收算法（CMS、G1、ZGC区别）

CMS（Concurrent Mark Sweep）
1. 特点
目标：最小化停顿时间（低延迟）。
算法：标记-清除（Mark-Sweep）。
工作阶段：
    初始标记（Initial Mark）：标记GC Roots直接关联的对象（STW）。
    并发标记（Concurrent Mark）：遍历对象图，标记存活对象。
    重新标记（Remark）：修正并发标记期间的变化（STW）。
    并发清除（Concurrent Sweep）：清除未标记的对象。
2. 优点
并发收集，减少停顿时间。
适合对延迟敏感的应用（如Web服务）。

3. 缺点
内存碎片问题（标记-清除算法的通病）。
并发阶段占用CPU资源，可能影响吞吐量。
无法处理浮动垃圾（并发清除期间产生的垃圾）。

4. 适用场景
适合老年代回收，对延迟敏感的应用。

二、G1（Garbage First）
1. 特点
目标：平衡吞吐量和延迟。
算法：分代收集 + 分区（Region）。

工作阶段：

初始标记（Initial Mark）：标记GC Roots直接关联的对象（STW）。
并发标记（Concurrent Mark）：遍历对象图，标记存活对象。
最终标记（Final Mark）：修正并发标记期间的变化（STW）。
筛选回收（Evacuation）：选择回收价值最高的Region进行回收（STW）。

2. 优点
分区设计，减少内存碎片。
可预测的停顿时间（通过 -XX:MaxGCPauseMillis 设置）。
适合大内存（>4GB）应用。

3. 缺点
相比CMS，停顿时间稍长。
小内存场景下性能不如CMS。

4. 适用场景
适合大内存、对延迟和吞吐量都有要求的应用。

三、ZGC（Z Garbage Collector）
1. 特点
目标：极低延迟（毫秒级停顿）。
算法：并发标记-整理（Concurrent Mark-Compact）。

工作阶段：
并发标记（Concurrent Mark）：标记存活对象。
并发预备重分配（Concurrent Prepare for Relocate）：选择需要回收的Region（区域）。
并发重分配（Concurrent Relocate）：将存活对象移动到新的Region（区域）。
并发重映射（Concurrent Remap）：更新引用关系。

2. 优点
停顿时间极短（通常<10ms）。
支持超大堆内存（TB级别）。
完全并发，几乎无STW。

3. 缺点
JDK 11引入，较新，生态支持可能不足。
需要64位系统，且对硬件要求较高。

4. 适用场景
适合超大内存、对延迟极其敏感的应用（如金融交易系统）。

四、CMS、G1、ZGC对比

特性	            CMS	                        G1	                        ZGC
目标      	    低延迟	                平衡吞吐量和延迟	            极低延迟
算法	            标记-清除	            分代 + 分区	                并发标记-整理
停顿时间	        较短	                    可预测	                    极短（<10ms）
内存碎片    	    有	                    较少	                        无
适用堆大小	    中小堆（<4GB）	        大堆（>4GB）	                超大堆（TB级别）
并发性	        部分并发	                部分并发	                    完全并发
JDK版本	        JDK 8及之前	            JDK 7及之后	                JDK 11及之后
适用场景	        对延迟敏感的应用	        大内存、平衡吞吐量和延迟	    超大内存、对延迟极其敏感

五、面试回答示例
面试官：请对比CMS、G1和ZGC的区别。
回答：

CMS：
目标是低延迟，采用标记-清除算法，适合中小堆内存和对延迟敏感的应用。
缺点是内存碎片问题，且无法处理浮动垃圾。
G1：
目标是平衡吞吐量和延迟，采用分代+分区算法，适合大堆内存。
优点是可预测的停顿时间和较少的内存碎片。
ZGC：
目标是极低延迟（毫秒级），采用并发标记-整理算法，适合超大堆内存和对延迟极其敏感的应用。
优点是几乎无停顿时间，但需要较新的JDK版本和硬件支持。

六、常见面试题
CMS和G1的区别是什么？

CMS注重低延迟，适合中小堆内存；G1注重平衡吞吐量和延迟，适合大堆内存。
ZGC的停顿时间为什么这么短？ ZGC采用完全并发的算法，几乎不需要STW（Stop-The-World）。

如何选择垃圾回收器？

根据应用场景选择：
中小堆内存、低延迟：CMS。
大堆内存、平衡吞吐量和延迟：G1。
超大堆内存、极低延迟：ZGC。

G1的Region设计有什么优势？
Region设计减少了内存碎片，同时可以优先回收价值最高的Region，提高回收效率。


类加载机制、双亲委派模型及破坏场景（如SPI）
Java 类加载机制是 JVM 将类的字节码文件（.class）加载到内存，并对字节码进行验证、准备、解析和初始化的过程。它是 Java 动态性的基础，支持类的动态加载和运行时绑定。

类加载的过程
类加载的过程可以分为以下几个阶段：

加载（Loading）：
通过类的全限定名获取类的字节码文件（.class）。
将字节码文件加载到内存，并生成一个 Class 对象。

验证（Verification）：
确保字节码文件的正确性和安全性。
包括文件格式验证、元数据验证、字节码验证和符号引用验证。

准备（Preparation）：
为类的静态变量分配内存，并设置默认初始值（如 0、null 等）。
如果静态变量是常量（final），则直接赋值为指定的值。

解析（Resolution）：
将常量池中的符号引用替换为直接引用。
符号引用是类的名称、方法的名称等，直接引用是内存地址。

初始化（Initialization）：
执行类的静态代码块（static {}）和静态变量的赋值操作。
这是类加载的最后一步，JVM 会确保类的初始化是线程安全的。


类加载器的层次结构
Java 类加载器采用双亲委派模型（Parent Delegation Model），类加载器的层次结构如下：

Bootstrap ClassLoader（启动类加载器）：
负责加载 JVM 核心类库（如 rt.jar），位于 JAVA_HOME/lib 目录下。
由 C++ 实现，是 JVM 的一部分。

Extension ClassLoader（扩展类加载器）：
负责加载扩展类库（如 javax.*），位于 JAVA_HOME/lib/ext 目录下。
由 Java 实现，是 sun.misc.Launcher$ExtClassLoader 的实例。

Application ClassLoader（应用程序类加载器）：
负责加载用户类路径（classpath）下的类。
由 Java 实现，是 sun.misc.Launcher$AppClassLoader 的实例。

自定义类加载器：
用户可以通过继承 ClassLoader 类实现自定义类加载器。
常用于热部署、模块化加载等场景。


双亲委派模型
双亲委派模型是类加载器的工作机制：
当一个类加载器收到类加载请求时，首先将请求委派给父类加载器。
如果父类加载器无法加载该类，子类加载器才会尝试加载。
这种机制确保了类的唯一性和安全性，避免了类的重复加载。

优点：
避免类的重复加载。
确保核心类库的安全性，防止用户自定义类替换核心类。

5. 打破双亲委派模型
在某些场景下，双亲委派模型会被打破：
SPI（Service Provider Interface）机制：
例如 JDBC 的 DriverManager，核心类库需要加载用户实现的驱动类。
通过线程上下文类加载器（Thread Context ClassLoader）实现。

热部署：

在应用服务器（如 Tomcat）中，每个 Web 应用需要独立的类加载器。
通过自定义类加载器实现。

6. 类加载机制的常见问题
ClassNotFoundException：
类加载器找不到指定的类。
通常是因为类路径配置错误或类未正确打包。

NoClassDefFoundError：
类加载器找到了类的定义，但在初始化时失败。
通常是因为类的静态初始化块或静态变量初始化失败。

LinkageError：
类加载器在链接阶段（验证、准备、解析）失败。
通常是因为类的版本不兼容或字节码文件损坏。

7. 实际应用场景
动态加载：
通过自定义类加载器实现插件化架构。

热部署：
在应用服务器中，通过类加载器实现代码的热更新。

模块化：
在 Java 9 及以上版本中，模块化系统（Module System）改进了类加载机制。

以下是一个简单的自定义类加载器示例：

public class MyClassLoader extends ClassLoader {
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classData = loadClassData(name);
        if (classData == null) {
            throw new ClassNotFoundException();
        }
        return defineClass(name, classData, 0, classData.length);
    }

    private byte[] loadClassData(String className) {
        // 从文件系统或网络中加载类的字节码
        // 这里省略具体实现
        return null;
    }
}




线程池参数设计（核心线程数计算、拒绝策略选择）
阿里为什么不推荐使用Excutores创建线程池
Excutores里面的阻塞队列或者创建最大线程数是Integer的maxValue，有可能会造成oom
我们自己创建自定义线程池，7个参数，在执行execute时，
先判断核心线程数是否满了（未满执行），如果满了判断阻塞队列数量是否已满（未满加入阻塞队列），如果阻塞队列满了判断最大线程数是否已满（未满创建临时线程），满了触发拒绝策略
临时线程每30秒-1个线程
如果线程执行期间报错并无捕捉的话，线程池会抛出异常，并将原来的线程销毁并重新创建一个线程，是因为线程可以配置统一异常处理的机制，线程池也是为了保持未捕捉异常的处理


核心线程数计算方式：CPU密集型N+1，IO密集型2N

拒绝策略：
如果不是强一致的业务，一般都是快速失败，如果不是快速失败的话，可能会导致内存过大，但是也看场景，像互联网的业务，设置的请求超时时间也比较短，就比较适合快速失败之后记个日志，像mysql的redo.log一样，空闲之后重做一遍


synchronized 和             ReentrantLock
java中的关键字               java中的一个类
jvm层面的锁                  api层面的锁
自动加锁和释放锁              手动加锁和释放锁
不可获取当前线程是否上锁       可获取当前线程是否上锁
非公平锁（对每个线程随机插队）  公平锁（按线程顺序一个一个）和非公平锁
不可中断                    可中断
锁的是对象，状态在对象头中     int类型的state表示来标识锁的状态
锁有升级过程                 没有锁升级过程

AQS原理（ReentrantLock、CountDownLatch源码）
LockSupport 提供park方法，可以指定唤醒某一个线程,底层由c++实现,
ReentrantLock 有两个实现，公平和非公平锁（默认），公平和非公平体现在公平会直接排队，而非公平在入队前会尝试获取锁，获取不到后还是会排队
AQS有三个非常重要的组成，一个是state，使用volatile修饰，代表同步状态，在ReentrantLock中，state=0代表没有被加锁，state=1代表当前资源已被加锁，state>1代表被当前线程多次加锁
第二个是Node节点，维护了一个双向链表，获取不到锁的线程会进入到双向链表中
第三个是当前持有锁的线程
ReentrantLock 加锁：
compareAndSetState(0,1) 修改state值从0修改为1
    1.1-> 修改成功后，
    1.2-> 修改失败后调用 acquire(1)
          1.2.1-> 调用 !tryAcquire(1) 尝试获取锁， 如果当前的state为0，尝试将state从0设置为1 如果设置成功，设置持有锁的线程为当前线程
                   1.2.1.1->如果当前state不为0，判断持有锁的线程是否为当前线程，如果为当前线程，则sate+1 如果不为当前线程，则返回false
          1.2.2-> 如果上一次没获取到锁，即加锁失败 调用 addWaiter() 作用是把没有获取到锁的线程创建为一个Node节点，加入到等待队列
          1.2.3-> 加入等待队列后，将node节点返回，调用 acquireQueued() 判断当前节点的前一个节点是否为头节点，如果为头节点，
          则再次尝试获取锁，如果获取锁失败，把上一个节点的waitStatus的状态设置为Signal ，再次尝试获取锁，获取锁失败的话会调用ParkAndCheckInterrupt()这个方法调用LockSupport的Park()将当前线程阻塞
解锁：
release(1) 调用tryRelease方法，将state-1，判断state是否为0 如果为是 将当前持有锁的线程设置为空，如果为否，则返回false
   1.1->如果为是，调用signalNext()方法，如果头节点不为空，并且头节点的后一个节点不为空，并且头节点的后一个节点的状态不为0（加入了等待队列还没有阻塞），则唤醒后一个节点


CountDownLatch
利用state，一开始设置state的初始值，然后将所有等待的线程阻塞在同步队列中，在state = 0后，唤醒全部线程
Semaphore 信号量
利用state，一开始设置state的初始值，然后将所有等待的线程阻塞在同步队列中，在state=0后，唤醒当前节点，当前节点唤醒后，将state+1并唤醒后一个线程，当state > 设置的初始值后，继续阻塞唤醒的线程



synchronized 锁优化（偏向锁、轻量级锁升级过程）
synchronized 原理是由一对monitorenter / monitorexit 指令实现的，锁的状态记录在对象头的Mark word中，jdk1.6之前monitor的实现是依靠操作系统内部的互斥锁实现的
jvm检测到不同的竞争状态的时候，会自动切换不同的锁实现，这个就是锁的升级和降级。
锁的状态有4种、无锁、偏向锁、轻量级锁、重量级锁
系统默认启动4秒后启动偏向锁，如果设置偏向锁参数为0的话,程序一进来就是偏向锁，当另一个线程进来的时候，触发偏向锁撤销，jvm首先会暂停原持有锁的线程，然后检查原线程的状态，如果退出同步块，对象恢复为无锁状态，新线程通过CAS竞争锁，
若还在同步块中，这个锁升级为轻量级锁，线程通过 CAS 操作 将对象头的 Mark Word 复制到自己的栈帧（Lock Record），并尝试将对象头替换为指向 Lock Record 的指针（锁标志位变为 00），
自旋失败（超过阈值）或竞争加剧（如多线程同时自旋失败）锁膨胀为 重量级锁（Mark Word 指向操作系统级的 Monitor 对象）
                    是否偏向     标志位
无锁对象头:           0           01
偏向锁对象头：         1           01
自旋锁对象头:         无           00
重量级锁对象头：       无          10


并发容器（ConcurrentHashMap分段锁 vs CAS） 暂时搁置

模块化（JPMS）、Record类、Pattern Matching
JPMS模块化 src/main/java 目录下创建module-info.java 文件，文件内声明引用的包，好处是包明确，之前没有的时候，使用jar包是使用找到的第一个包
Record类,java层面的pojo类，但是不允许修改，实际设置属性为final
Pattern Matching 模式匹配用于简化对象类型检查和提取的代码 16 支持 instanceof 模式匹配 21支持 switch 模式匹配


响应式编程（Reactor、WebFlux核心思想）




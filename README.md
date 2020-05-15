# Java_Learning.github.io
# Java学习笔记
## 一、《深入理解Java虚拟机》
### Java运行时数据区域：
1.程序计数器：一块较小的内存空间，可以看作是当前线程所执行的字节码的行号指示器。多线程时，为了线程切换后能恢复到正确的位置，每条线程都必须有一个独立的程序计数器，各个线程之间计数器互不影响，独立存储，即“**线程私有**”。<br>
2.Java虚拟机栈：也是“**线程私有**”的，生命周期与线程相同，它描述的是Java方法执行的内存模型：每个方法在执行时会创建一个**栈帧**(Stack Frame)，用于存储**局部变量表、操作数栈、动态链接、方法出口**等信息。局部变量表中存放了编译期可知的各种数据类型：基本类型(boolean,byte,short,int,float,long,double)，对象引用(reference类型，可能是指向一个对象起始地址的引用指针，也可能是指向一个对象代表的句柄)和returnAddress类型(指向了一条字节码指令的地址)。其中64位的double和long类型会占据两个局部变量表空间(Slot)，其余的数据类型只占据一个。局部变量表所需的内存空间大小在编译期间完成分配，在方法运行期间不会改变其大小。在Java虚拟机规范中，对这个区域规定了两种异常情况：如果线程请求的栈深度大于虚拟机栈所允许的深度(如递归)，则会报出**StackOverflowError**;如果虚拟机栈动态扩展时申请不到足够的内存，则会报**OutOfMemoryError**;<br>
3.本地方法栈：本地方法栈与虚拟机栈的作用类似，区别就是虚拟机栈为虚拟机执行Java方法服务，而本地方法栈为虚拟机执行Native方法服务。<br>
4.Java堆：**被所有线程共享的一块区域**，在虚拟机启动时创建。此内存区域的唯一目的就是**存放对象实例，几乎所有的对象实例都在这里分配内存**。Java堆是垃圾收集器管理的主要区域，由于现在收集器都采用分代收集算法，所以Java堆可以细分为**新生代和老年代**，再细致一点可以分为**Eden空间，From Survivor空间和 To Survivor空间**。<br>
5.方法区：也是**被所有线程共享的一块区域**，用于存放加载的类信息、常量、静态变量、即时编译后的代码等。<br>
6.运行时常量池：是方法区的一部分。Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池(Constant Pool Table)，用于存放编译期生成的字面量和符号引用。运行时常量池相对于Class文件常量池的另外一个重要特征是**动态性**，Java语言并不要求常量只有在编译期才能产生，运行期间也可以将新的常量放入常量池中，**例如String.inter()方法**。
### 垃圾收集器与内存分配策略
1.**引用计数算法**(Reference Counting)：给对象中添加一个引用计数器，每当有一个地方引用他时，计数器值就加1；当引用失效时，计数器值就减1；任何时刻计数器值为0的对象就是不可再被使用的。**优势：实现简单、判定效率高** **劣势:很难解决对象之间互相循环引用的问题**
2.**可达性分析算法(Reachability Analysis)** ：从"GC ROOTS"对象开始向下搜索，搜索所走过的路径称为引用链(Reference Chain)，当一个对象到GC ROOTS没有任何引用链连接时，则证明此对象不可用。**可被称为GC ROOTS的对象：1.虚拟机栈(栈帧中本地变量表)引用的对象 2.方法区中类静态属性引用的对象 3.方法区中常量引用的对象 4.本地方法栈JNI中引用的对象**
### 对象生存还是死亡？
即使在可达性分析算法中不可达的对象，也并非是“非死不可”的：**如果对象在进行可达性分析后没有与GC ROOTS相连的引用链，那它将会被第一次标记并进行一次筛选，筛选的条件是此对象是否有必要执行finalize()方法，当对象没有覆盖finalize()方法和此方法已被虚拟机调用过，都被视为“没有必要执行”。如果此对象有必要执行finalize()方法，这个对象会被放进F-QUEUE队列中，并在稍后由一个虚拟机自动建立的、低优先级的Finalizer线程去执行。finalize()方法是对象逃离死亡的最后一次机会--对象在此方法中与引用链重新链接上即可。如果这时他还没有逃脱，那他基本上真的被回收了。**<br>

## 二、Dubbo官方文档阅读笔记
### 1.架构
**调用关系说明**：<br>
0.服务容器负责启动，加载，运行服务提供者。<br>
1.服务提供者在启动时，向注册中心注册自己提供的服务。<br>
2.服务消费者在启动时，向注册中心订阅自己所需的服务。<br>
3.注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。<br>
4.服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。<br>
5.服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心。<br>
### 2.基于注解的配置
**服务提供方**：使用@Service注解暴露服务。**服务调用方**：使用@Reference注解引用服务。
### 3.配置中心
承担两个职责：<br>
1.外部化配置。启动配置的集中式存储 （简单理解为dubbo.properties的外部化存储）。<br>
2.服务治理。服务治理规则的存储与通知。<br>
<dubbo:config-center address="zookeeper://127.0.0.1:2181"/> <br>
**为了兼容2.6.x版本配置，在使用Zookeeper作为注册中心，且没有显示配置配置中心的情况下，Dubbo框架会默认将此Zookeeper用作配置中心，但将只作服务治理用途。** <br>
**外部化配置**：外部化配置默认较本地配置有更高的优先级，因此这里配置的内容会覆盖本地配置值，也可通过以下选项调整配置中心的优先级：-Ddubbo.config-center.highest-priority=false <br>
外部化配置有全局和应用两个级别，全局配置是所有应用共享的，应用级配置是由每个应用自己维护且只对自身可见的。**节点结构**：(namespace，用于不同配置的环境隔离)dubbo->config(Dubbo约定的固定节点，不可更改，所有配置和服务治理规则都存储在此节点下)->dubbo/application(分别用来隔离全局配置、应用级别配置：dubbo是默认group值，application对应应用名)->dubbo.properties(此节点的node value存储具体配置内容)。<br>
**配置加载流程**：Dubbo支持了多层级的配置，并按预定优先级自动实现配置间的覆盖，最终所有配置汇总到数据总线URL后驱动后续的服务暴露、引用等流程。<br>
Dubbo支持的配置来源说起，默认有四种配置来源（优先级从高到低）：<br>
JVM System Properties，-D参数<br>
Externalized Configuration，外部化配置<br>
ServiceConfig、ReferenceConfig等编程接口采集的配置<br>
本地配置文件dubbo.properties<br>

## 三、《重构》阅读笔记
这次分享的内容一部分来自于《重构-改善既有代码的设计》这本经典的书籍，一部分来自于本人维护项目中的一些实践经验。欢迎一起分享、讨论。

### No.1 重复代码的提炼 --《重构》第六章 Extract Method

提炼重复代码是重构收效最大的手法之一，进行这项重构的原因不需要多说，甚至在开发过程中就应该具备这种思维。它有很多好处，比如总代码量大大减少（因为抽象的到位，可以复用）、维护方便、代码条理更清晰、代码更易读等等。

它的重点就在于寻找代码中完成某项子功能的重复代码，找到以后请毫不犹豫的将它移动到合适的方法中，并存放在合适的类中。

Example：sf-unread-service          AgentOrgChangeListener

![Aaron Swartz](https://raw.githubusercontent.com/wangjc95/photos/master/image2020-4-8_16-12-2.png)

![Aaron Swartz](https://raw.githubusercontent.com/wangjc95/photos/master/image2020-4-8_16-10-59.png)



### No.2 冗长代码的分割 --《重构》第三章 Long Method

冗长代码的分割，与重复代码的提炼是有着不可分割的关系的，往往我们提炼重复代码的过程中，就不知不觉的完成了对某一个超长方法的分割。

值得注意的是：我们在分割大方法时，大部分是针对其中一些子功能分割，因此给每一个子功能起一个恰到好处的名字很重要。

Example：sf-agent-service          ManagerAttributionStoreInitJob

![Aaron Swartz](https://raw.githubusercontent.com/wangjc95/photos/master/image2020-4-10_16-57-26.png)



### No.3 嵌套条件分支的优化（1） --《重构》第九章 Replace Nested Conditional With Guard Clauses

大量的嵌套条件分支是很容易让人望而却步的代码，应该极力避免这种代码的出现。尽管结构化原则：一个函数只能有一个出口，但如果有大量的嵌套条件分支，很容易违背这规则。

卫语句可以治疗这种恐怖的嵌套条件。它的核心思想是：将不满足条件的情况放在方法前面，快速跳出，避免对后面的判断造成影响和额外变量占用内存。经过这项手术的代码看起来会非常清晰。

Example：sf-data-api          AgentJoinStoreListener

![Aaron Swartz](https://raw.githubusercontent.com/wangjc95/photos/master/image2020-4-8_16-39-57.png)


### No.4 嵌套条件分支的优化（2） --《重构》第九章 Consolidate Conditional Expression

此处所说的嵌套条件分支与上面的有些许不同，它无法使用卫语句进行优化，而是应该将条件分支合并，以此来达到代码清晰的目的。由这两条也可以看出，嵌套条件分支在编码中应当尽量避免。

Example：sf-bff-api          CouponInvoker

![Aaron Swartz](https://raw.githubusercontent.com/wangjc95/photos/master/image2020-4-8_16-58-58.png)



### No.5 去掉一次性的临时变量 --《重构》第六章 Replace Temp With Query

生活中我们常用一次性筷子，这无疑是对树木资源的摧残。然而在程序中，一次性的临时变量不仅是对性能的摧残，更是对代码可读性的亵渎。因此我们有必要对一次性的临时变量进行手术。



### No.6 消除过长参数列表 --《重构》第十章 Introduce Parameter Object

对于一些传递了大批参数的方法，我们可以尝试将这些参数封装成一个对象，再传递给方法，从而去除过长参数列表。这样封装久了以后，当你尝试寻找这样一个对象时，它往往已经存在了，可以省去多余的工作。



### No.7 提取类或继承体系中的常量 --《重构》第八章 Replace Magin Number With Symbolic Constant

这项重构的目的是为了消除一些魔数或者是字符串常量等，魔数会让人对程序的意图产生疑惑。把魔数替换为字面常量（或枚举），好处在于维护时的方便。

Example：sf-unread-service          AgentOrgChangeListener

![Aaron Swartz](https://raw.githubusercontent.com/wangjc95/photos/master/image2020-4-8_17-18-28.png)


### No.8 让类提供应该提供的方法 --《重构》第八章 Replace Magin Number With Symbolic Constant

很多时候，我们经常会操作一个类的大部分属性，从而得到一个最终结果。这种时候，我们应该让这个类做他该做的事，而不是我们编码替他做。如果我们替他做了，最终会成为重复代码的根源。这和DDD（Domain Drive Design）的思想不谋而合，要划分出领域，并提供这个领域该有的能力。

Bad Example：

![Aaron Swartz](https://raw.githubusercontent.com/wangjc95/photos/master/image2020-4-8_17-29-53.png)

Good Example：

![Aaron Swartz](https://raw.githubusercontent.com/wangjc95/photos/master/image2020-4-8_17-32-38.png)



### No.9 拆分冗长的类

这项技巧也是属于非常实用的一个技巧，它的难度也较高。大部分时候，我们拆分一个类的关注点应该主要集中在类的属性上面。拆分出来的多批属性应该在逻辑上是可以分离的，并且这两批属性的使用也都分别集中于某一些方法中。如果实在有一些属性同时存在于拆分后的两批方法内部，那么可以通过参数传递的方式解决。

类的拆分是一个相对较大的工程，因为大类往往被很多类使用着，因此一定要谨慎，做好足够的测试。



### No.10 提取继承体系中重复的属性与方法到父类 --《重构》第十一章 Extract Superclass

这项技巧大部分时候需要足够的判断力，在这个过程中，其实是在向模板方法迈进。

Example：sf-agent-service          BaseCompanyAttributionReq

![Aaron Swartz](https://raw.githubusercontent.com/wangjc95/photos/master/image2020-4-8_17-49-23.png)


## 四、学习笔记
### 1.Java引用类型：
1.强引用：类似于 Object a = new Object() 这类的引用，**只要垃圾强引用存在，垃圾回收器就不会回收被引用的对象**。<br>
2.软引用：对于软引用(使用SoftReference类实现软引用)关联的对象，**在JVM将要发生OOM(Out Of Memory)异常之前，会把这些对象列入第二次垃圾回收范围。如果本次回收还没有足够的内存，则抛出内存异常(当一个对象在堆上分配内存且不够时，先发生一次GC，清理掉弱引用和虚引用，如果够就分配，如果不够就进行第二次GC，清理掉软引用，如果清理完还不够，则抛出内存溢出异常。)**。<br>
3.弱引用：比软引用更弱，被弱引用(使用WeakReference类实现弱引用)关联的对象只能存活到下次垃圾回收之前。当发生GC时(Garbage Collection)时，**无论当前内存是否足够，都会清理掉只被弱引用关联的对象**。<br>
4.虚引用：一个对象是否有虚引用(使用PhantomReference类实现虚引用)的存在，完全不会对其生存时间构成影响，也无法通过虚引用取得一个对象的实例。**为一个对象设置虚引用关联的唯一目的就是能够在这个对象被垃圾回收器回收掉后收到一个通知**。
弱引用使用场景：假设有这样一个需求，每次创建一个数据库Connection时，需要将用户信息User与Connection关联。典型的做法就是在一个全局的Map中存储Connection与User的映射。<br>
public class ConnManager {<br>
    private Map<Connection,User> m = new HashMap<Connection,User>();

    public void setUser(Connection s, User u) {
        m.put(s, u);
    }
    public User getUser(Connection s) {
        return m.get(s);
    }
    public void removeUser(Connection s) {
        m.remove(s);
    }
}<br>
这样做的问题是User生命周期与Connection挂钩，我们无法准确预知Connection在什么时候结束，所以需要在每个Connection关闭之后，手动从Map中移除键值对，否则Connection和User将一直被Map引用，即使Connection的生命周期已经结束了，GC也无法回收对应的Connection和User。这些对象留在内存中不受控制，可能会造成内存溢出。<br>
**解决方法**：private Map<Connection,User> m = new WeakHashMap<Connection,User>();<br>
WeakHashMap 与 HashMap类似，但是在其内部，key是经过WeakReference包装的。使用WeakHashMap情况会变得怎样呢？
每当垃圾回收发生时，那些已经结束生命周期的Connection对象(没有强引用指向它)不受WeakHashMap中key(WeakReference)的影响，可以直接回收掉。同时，WeakHashMap利用ReferenceQueue(下文会提到) 可以做到删除那些已经被回收的Connection对应的User。<br>

### 2.可达性分析算法
Java执行GC时，需要判断对象是否存活。判断一个对象是否存活使用了"可达性分析算法"。
基本思路就是通过一系列称为GC Roots的对象作为起点，从这些节点开始向下搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots没有任何引用链相连，即从GC Roots到这个对象不可达时，证明此对象不可用。<br>
**可作为GC ROOTS的对象包括：**<br>
**1.虚拟机栈中引用的对象 2.方法区中类静态属性引用的对象 3.方法区中常量引用的对象 4.本地方法栈JNI引用的对象**<br>
往往到达一个对象的引用链会存在多条，垃圾回收时会依据两个原则来判断对象的可达性：<br>
**1.单一路径中，以最弱的引用为准。**<br>
**2.多路径中，以最强的引用为准。**<br>

### 3.AMQP(Advanced Message Queuing Protocol)高级消息队列协议
消息生产者Producer(Message) **->** 消息队列服务器实体Broker(消息交换机Exchange,绑定Bindings,消息队列载体Queues) **->** 消息消费者Consumer(Message)<br>
**Exchange**：消息交换机，它指定消息按照什么规则，路由到哪个队列。<br>
**Bindings**：绑定，它的作用是将Exchange和Queues按照路由规则绑定起来。<br>
**Queues**：消息队列载体，每个消息都会被投放到一个或多个队列。<br>
**那么为什么我们需要 Exchange 而不是直接将消息发送至队列呢？**
AMQP 协议中的核心思想就是**生产者和消费者的解耦**，生产者从不直接将消息发送给队列。生产者通常不知道是否一个消息会被发送到队列中，只是将消息发送到一个交换机。先由 Exchange 来接收，然后 Exchange 按照特定的策略转发到 Queue 进行存储。Exchange 就类似于一个交换机，将各个消息分发到相应的队列中。<br>
**Exchange收到消息后，它是怎样知道发送到哪些Queues呢？**
Binding 表示 Exchange 与 Queue 之间的关系，我们也可以简单的认为队列对该交换机上的消息感兴趣，**绑定可以附带一个额外的参数 RoutingKey**。Exchange 就是根据这个 RoutingKey 和当前 Exchange 所有绑定的 Binding 做匹配，如果满足匹配，就往 Exchange 所绑定的 Queue 发送消息，这样就解决了我们向 RabbitMQ 发送一次消息，可以分发到不同的 Queue。RoutingKey 的意义依赖于交换机的类型。<br>
**Exchange的三种常用类型:Fanout、Direct、Topic**<br>
**Fanout Exchange**：Fanout Exchange 会忽略 RoutingKey 的设置，直接将 Message **广播**到所有绑定的 Queue 中。<br>
应用场景：以日志系统为例：假设我们定义了一个 Exchange 来接收日志消息，同时定义了两个 Queue 来存储消息：一个记录将被打印到控制台的日志消息；另一个记录将被写入磁盘文件的日志消息。我们希望 Exchange 接收到的每一条消息都会同时被转发到两个 Queue，这种场景下就可以使用 Fanout Exchange 来广播消息到所有绑定的 Queue。<br>
**Direct Exchange**:Direct Exchange 是 RabbitMQ 默认的 Exchange，完全根据 RoutingKey 来路由消息。设置 Exchange 和 Queue 的 Binding 时需指定 RoutingKey（一般为 Queue Name），发消息时也指定一样的 RoutingKey，消息就会被路由到对应的Queue。<br>
应用场景：现在我们考虑只把重要的日志消息写入磁盘文件，例如只把 Error 级别的日志发送给负责记录写入磁盘文件的 Queue。这种场景下我们可以使用指定的 RoutingKey（例如 error）将写入磁盘文件的 Queue 绑定到 Direct Exchange 上。<br>
**Topic Exchange**：Topic Exchange 和 Direct Exchange 类似，也需要通过 RoutingKey 来路由消息，区别在于Direct Exchange 对 RoutingKey 是精确匹配，而 Topic Exchange 支持模糊匹配。分别支持 * 和 # 通配符，* 表示匹配一个单词，#则表示匹配没有或者多个单词。<br>
应用场景：假设我们的消息路由规则除了需要根据日志级别来分发之外还需要根据消息来源分发，可以将 RoutingKey 定义为 消息来源.级别 如 order.info、user.error等。处理所有来源为 user 的 Queue 就可以通过 user.* 绑定到 Topic Exchange 上，而处理所有日志级别为 info 的 Queue 可以通过 * .info 绑定到 Exchange上。<br>

### 4.String.intern()
String.intern()是一个Native方法，底层调用C++的 StringTable::intern方法实现。**当通过语句str.intern()调用intern()方法后，JVM 就会在当前类的常量池中查找是否存在与str等值的String，若存在则直接返回常量池中相应Strnig的引用；若不存在，则会在常量池中创建一个等值的String，然后返回这个String在常量池中的引用。** 因此，只要是等值的String对象，使用intern()方法返回的都是常量池中同一个String引用，所以，这些等值的String对象通过intern()后使用==是可以匹配的。**intern()方法优点：执行速度非常快，直接使用==进行比较要比使用equals()方法快很多；内存占用少。** 虽然intern()方法的优点看上去很诱人，但若不是在恰当的场合中使用该方法的话，便非但不能获得如此好处，反而还可能会有性能损失。由于intern()操作每次都需要与常量池中的数据进行比较以查看常量池中是否存在等值数据，同时JVM需要确保常量池中的数据的唯一性，这就涉及到加锁机制，这些操作都是有需要占用CPU时间的，所以如果进行intern操作的是大量不会被重复利用的String的话，则有点得不偿失。由此可见，String.intern()主要 适用于只有有限值，并且这些有限值会被重复利用的场景，如数据库表中的列名、人的姓氏、编码类型等。https://blog.csdn.net/u011635492/article/details/81048150

### 5.MVCC(MultiVersion Concurrency Control)多版本并发控制
是现代数据库（包括MySQL、Oracle、PostgreSQL等）引擎实现中常用的处理读写冲突的手段，目的在于提高 数据库 高并发场景下的吞吐性能 。

如此一来不同的事务在并发过程中， SELECT  操作可以不加锁而是通过  MVCC  机制读取指定的版本历史记录，并通过一些手段保证保证读取的记录值符合事务所处的隔离级别，从而解决并发场景下的读写冲突。

看一个栗子：在事务  A  提交前后，事务  B  读取到的  x  的值是什么呢？
![Aaron Swartz](https://raw.githubusercontent.com/wangjc95/photos/master/b4328925d1bda917b2b9e9991b64bd44.png)
答案是：**事务  B  在不同的隔离级别下，读取到的值不一样。**
如果事务  B  的隔离级别是读未提交（RU），那么两次读取均读取到  x  的最新值，即  20 。
如果事务  B  的隔离级别是读已提交（RC），那么第一次读取到旧值  10 ，第二次因为事务  A  已经提交，则读取到新值 20。
如果事务  B  的隔离级别是可重复读或者串行（RR，S），则两次均读到旧值  10 ，不论事务  A  是否已经提交。

可见在不同的隔离级别下，数据库通过  MVCC  和隔离级别，让事务之间并行操作遵循了某种规则，来保证单个事务内前后数据的一致性。


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

## 四、《MySQL技术内幕-InnoDB存储引擎》
### 第6章 锁
InnoDB中锁的类型：1、共享锁(Share Lock、S锁)，允许事务读一行数据。2、排它锁(Exclusive Lock、X锁)，允许事务更新或删除一行数据。<br>
//   X&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;S <br>
X  不兼容&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;不兼容<br>
S  不兼容&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;兼容<br>
此外，InnoDB存储引擎支持多粒度(granular)锁定，允许事务在行级锁和表级锁上的锁同时存在。为了支持这种操作，InnoDB支持了一种额外的锁方式,称之为**意向锁**(Intension Lock)。意向锁又分为IS Lock和IX Lock。**若需要对细粒度的对象上锁，那么首先需要对粗粒度的对象上锁**。若需要对记录上X锁，则需要分别对数据库、表、页上意向锁IX，最后对记录上X锁。若其中任何一个部分等待，则该操作阻塞。<br>
// IS&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;IX&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;S&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;X <br>
IS  兼容&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;兼容&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;兼容&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;不兼容<br>
IX  兼容&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;兼容&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;不兼容&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;不兼容<br>
S  兼容&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;不兼容&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;兼容&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;不兼容<br>
X  不兼容&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;不兼容&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;不兼容&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;不兼容<br>

行锁的三种算法：<br>
1、Record Lock：单个行记录上的锁。总是会去锁定索引记录，如果InnoDB引擎下的表在创建时没有设置任何索引，那么InnoDB引擎会使用隐式的主键来锁定。<br>
2、Gap Lock：间隙锁，锁定一个范围，但不包括记录本身<br>
3、Next Key Lock：等于 Record Lock + Gap Lock，锁定一个范围，并且包含记录本身<br>
例如一个索引列有10、11、13、20四个值，那该索引可能被 Next Key Locking的区间为：(-∞, 10], (10, 11], (11, 13], (13, 20], (20, +∞)。<br>
**设计Next Key Lock是为了解决Phantom Problem(幻读)问题。** 它是谓词锁(predict lock)的一种改进。除了Next Key Locking技术(左开右闭)，还有 Previous Key Locking技术(左闭右开)。<br>
**当查询的索引含有唯一属性时，InnoDB引擎会对Next Key Lock进行优化，降级为Record Lock，从而提高应用的并发性。** 若查询的索引是辅助索引，则使用Next Key Lock。**需要特别注意的是，InnoDB引擎还会对辅助索引的下一个值加上Gap Lock。** <br>


## 五、学习笔记
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
![Aaron Swartz](https://raw.githubusercontent.com/wangjc95/photos/master/b4328925d1bda917b2b9e9991b64bd44.png) <br>
答案是：**事务  B  在不同的隔离级别下，读取到的值不一样。** <br>
如果事务  B  的隔离级别是读未提交（RU），那么两次读取均读取到  x  的最新值，即  20 。<br>
如果事务  B  的隔离级别是读已提交（RC），那么第一次读取到旧值  10 ，第二次因为事务  A  已经提交，则读取到新值 20。<br>
如果事务  B  的隔离级别是可重复读或者串行（RR，S），则两次均读到旧值  10 ，不论事务  A  是否已经提交。<br>

可见在不同的隔离级别下，数据库通过  MVCC  和隔离级别，让事务之间并行操作遵循了某种规则，来保证单个事务内前后数据的一致性。

**为什么需要MVCC？** <br>
InnoDB  相比  MyISAM  有两大特点，一是支持事务二是支持行级锁，事务的引入带来了一些新的挑战。相对于串行处理来说，并发事务处理能大大增加数据库资源的利用率，提高数据库系统的事务吞吐量，从而可以支持可以支持更多的用户。但并发事务处理也会带来一些问题，主要包括以下几种情况：

1.更新丢失（ Lost Update ）：当两个或多个事务选择同一行，然后基于最初选定的值更新该行时，由于每个事务都不知道其他事务的存在，就会发生丢失更新问题 —— 最后的更新覆盖了其他事务所做的更新。如何避免这个问题呢，最好在一个事务对数据进行更改但还未提交时，其他事务不能访问修改同一个数据。

2.脏读（ Dirty Reads ）：一个事务正在对一条记录做修改，在这个事务未提交前，这条记录的数据就处于不一致状态；这时，另一个事务也来读取同一条记录，如果不加控制，第二个事务读取了这些尚未提交的脏数据，并据此做进一步的处理，就会产生未提交的数据依赖关系。这种现象被形象地叫做  “脏读” 。

3.不可重复读（ Non-Repeatable Reads ）：一个事务在读取某些数据已经发生了改变、或某些记录已经被删除了！这种现象叫做“不可重复读”。

4.幻读（ Phantom Reads ）：一个事务按相同的查询条件重新读取以前检索过的数据，却发现其他事务插入了满足其查询条件的新数据，这种现象就称为  “幻读” 。

以上是并发事务过程中会存在的问题，解决更新丢失可以交给应用，但是后三者需要数据库提供事务间的隔离机制来解决。 实现隔离机制的方法主要有两种 ：

1.加读写锁

2.一致性快照读，即  MVCC

但本质上，隔离级别是一种在并发性能和并发产生的副作用间的妥协，通常数据库均倾向于采用  Weak Isolation 。

**InnoDB MVCC实现原理** <br>
InnoDB  中  MVCC  的实现方式为：每一行记录都有两个隐藏列： **DATA_TRX_ID** 、 **DATA_ROLL_PTR** 、（如果没有主键，则还会多一个隐藏的主键列**DB_ROW_ID**）。

![Aaron Swartz](https://raw.githubusercontent.com/wangjc95/photos/master/bfa89998d4dea1104d7954d6ab03350e.png)<br>
**DATA_TRX_ID** <br>

记录最近更新这条行记录的 事务 ID ，大小为  6  个字节

**DATA_ROLL_PTR** <br>

表示指向该行回滚段 （rollback segment） 的指针，大小为  7  个字节， InnoDB  便是通过这个指针找到之前版本的数据。该行记录上所有旧版本，在  undo  中都通过链表的形式组织。

**DB_ROW_ID** <br>

行标识（隐藏单调自增  ID ），大小为  6  字节，如果表没有主键， InnoDB  会自动生成一个隐藏主键，因此会出现这个列。另外，每条记录的头信息（ record header ）里都有一个专门的  bit （ deleted_flag ）来表示当前记录是否已经被删除。

**如何组织版本链** <br>
在多个事务并行操作某行数据的情况下，不同事务对该行数据的 UPDATE 会产生多个版本，然后通过回滚指针组织成一条  Undo Log  链。
还是以上面的例子，事务  A  对值  x  进行更新之后，该行即产生一个新版本和旧版本。假设之前插入该行的事务  ID  为  100 ，事务  A  的  ID  为  200 ，该行的隐藏主键为  1 。
![Aaron Swartz](https://raw.githubusercontent.com/wangjc95/photos/master/ab36e68439be8c763c93505d34fcd72c.png) <br>
事务  A  的操作过程为：

1.对  DB_ROW_ID = 1  的这行记录加排他锁

2.把该行原本的值拷贝到  undo log  中， DB_TRX_ID  和  DB_ROLL_PTR  都不动

3.修改该行的值这时产生一个新版本，更新  DATA_TRX_ID  为修改记录的事务  ID ，将  DATA_ROLL_PTR  指向刚刚拷贝到  undo log  链中的旧版本记录，这样就能通过  DB_ROLL_PTR  找到这条记录的历史版本。如果对同一行记录执行连续的  UPDATE ， Undo Log  会组成一个链表，遍历这个链表可以看到这条记录的变迁

4.记录  redo log ，包括  undo log  中的修改

那么  INSERT  和  DELETE  会怎么做呢？其实相比  UPDATE  这二者很简单， INSERT  会产生一条新纪录，它的  DATA_TRX_ID  为当前插入记录的事务  ID ； DELETE  某条记录时可看成是一种特殊的  UPDATE ，其实是软删，真正执行删除操作会在  commit  时， DATA_TRX_ID  则记录下删除该记录的事务  ID 。<br>

**如何实现一致性读？**<br>
在  RU  隔离级别下，直接读取版本的最新记录就 OK，对于  SERIALIZABLE  隔离级别，则是通过加锁互斥来访问数据，因此不需要  MVCC  的帮助。因此  MVCC  运行在  RC  和  RR 这两个隔离级别下，当  InnoDB  隔离级别设置为二者其一时，在  SELECT  数据时就会用到版本链

核心问题是版本链中哪些版本对当前事务可见？

InnoDB  为了解决这个问题，设计了  **ReadView （可读视图）**的概念。

**RR下ReadView的生成**<br>
在  RR  隔离级别下，每个事务  touch first read  时（本质上就是执行第一个  SELECT 语句时，后续所有的  SELECT  都是复用这个  ReadView ，其它  update ,  delete ,  insert  语句和一致性读  snapshot  的建立没有关系），会将当前系统中的所有的活跃事务拷贝到一个列表生成  ReadView 。

下图中事务  A  第一条  SELECT  语句在事务  B  更新数据前，因此生成的  ReadView  在事务  A  过程中不发生变化，即使事务  B  在事务  A  之前提交，但是事务  A  第二条查询语句依旧无法读到事务  B  的修改。
![Aaron Swartz](https://raw.githubusercontent.com/wangjc95/photos/master/9ddb47dae87151dac6c8829bd98e30cd.png) <br>

**RC下ReadView的生成**<br>
在  RC  隔离级别下，每个  SELECT  语句开始时，都会重新将当前系统中的所有的活跃事务拷贝到一个列表生成  ReadView 。**二者的区别就在于生成  ReadView  的时间点不同，一个是事务之后第一个  SELECT  语句开始、一个是事务中每条  SELECT  语句开始。**<br>

ReadView  中是当前活跃的事务  ID  列表，称之为  m_ids ，其中最小值为  up_limit_id ，最大值为  low_limit_id ，事务  ID  是事务开启时  InnoDB  分配的，其大小决定了事务开启的先后顺序，因此我们可以通过  ID  的大小关系来决定版本记录的可见性，具体判断流程如下：
1.如果被访问版本的  trx_id  小于  m_ids  中的最小值  up_limit_id ，说明生成该版本的事务在  ReadView  生成前就已经提交了，所以该版本可以被当前事务访问。<br>

2.如果被访问版本的  trx_id  大于  m_ids  列表中的最大值  low_limit_id ，说明生成该版本的事务在生成  ReadView  后才生成，所以该版本不可以被当前事务访问。需要根据  Undo Log  链找到前一个版本，然后根据该版本的 DB_TRX_ID 重新判断可见性。<br>

3.如果被访问版本的  trx_id  属性值在  m_ids  列表中最大值和最小值之间（包含），那就需要判断一下  trx_id  的值是不是在  m_ids  列表中。如果在，说明创建  ReadView  时生成该版本所属事务还是活跃的，因此该版本不可以被访问，需要查找 Undo Log 链得到上一个版本，然后根据该版本的  DB_TRX_ID  再从头计算一次可见性；如果不在，说明创建  ReadView  时生成该版本的事务已经被提交，该版本可以被访问。<br>

4.此时经过一系列判断我们已经得到了这条记录相对  ReadView  来说的可见结果。此时，如果这条记录的  delete_flag  为  true ，说明这条记录已被删除，不返回。否则说明此记录可以安全返回给客户端。<br>

**RC和RR下MVCC判断流程** <br>
我们现在回看刚刚的查询过程，为什么事务  B  在  RC  隔离级别下，两次查询的  x  值不同。 RC  下  ReadView  是在语句粒度上生成的。
当事务  A  未提交时，事务  B  进行查询，假设事务  B  的事务  ID  为  300 ，此时生成  ReadView  的  m_ids  为 [200，300]，而最新版本的  trx_id  为  200 ，处于  m_ids 中，则该版本记录不可被访问，查询版本链得到上一条记录的 trx_id 为  100 ，小于  m_ids 的最小值  200 ，因此可以被访问，此时事务  B  就查询到值  10  而非  20 。
待事务  A  提交之后，事务  B  进行查询，此时生成的  ReadView  的  m_ids  为 [300]，而最新的版本记录中  trx_id  为  200 ，小于  m_ids  的最小值  300 ，因此可以被访问到，此时事务  B  就查询到  20 。


如果在  RR  隔离级别下，为什么事务  B  前后两次均查询到  10  呢？ RR  下生成  ReadView  是在事务开始时，m_ids 为 [200,300]，后面不发生变化，因此即使事务  A  提交了， trx_id  为  200  的记录依旧处于  m_ids  中，不能被访问，只能访问版本链中的记录  10 。

### 6.权限控制Cookie、Session、Token <br>
**Cookie**：服务端发送到用户浏览器并由浏览器保存的一小块数据，它会在下次浏览器向同一服务端再次发起请求时携带过去。**Cookie是不可跨域的**。例如：某个cookie属于www.baidu.com，那么www.taobao.com无法使用，但 www.baidu.com/1 和 www.baidu.com/2 可以共享使用，靠的是domain(也就是www.baidu.com)。<br>
**Session**：**基于Cookie实现**，session存储在服务端，sessionId会被存储在浏览器的cookie中。<br>
session认证流程：1、浏览器第一次请求服务器，服务器根据用户提交的信息创建session，并将sessionId返回给浏览器。2、浏览器将收到的sessionId放在cookie中，同时cookie会记录此sessionId属于哪个域名。3、浏览器再次请求服务器时，会将对应域名下的cookie信息发送给服务器，服务器进行鉴权。<br>
**Cookie和Session的区别**：1、Session比Cookie安全，因为Session存储在服务器，Cookie存储在浏览器。2、存储的值类型不同，Cookie只支持字符串数据，其他类型的数据需要转换成字符串，而Seesion可以存任意类型。3、有效期不同：Cookie可以设置长时间保持（例如常见的默认登陆功能），而Session的有效期较短，浏览器关闭或者Session超时都会失效。4、存储数据的大小不同：单个Cookie最多支持4K，而Session可存储的数据远大于Cookie，但当数据量过大和访问量过多，会占用服务器资源。<br>
**Token**：访问资源接口API时所需要的资源凭证。简单的Token由uid（用户唯一标识）、time（当前时间戳）、sign（签名：token的前几位以哈希算法压缩成十六进制字符串）。<br>
Token认证流程：1、客户端使用用户名密码登录。2、服务端收到用户名密码，验证。若验证成功，服务端会签发一个token并返回。3、客户端收到token后，会把它存储在cookie或者localStorage中。4、以后每次请求，客户端都要携带token（放在HTTP的Header里），服务端都要进行验证。<br>
Token特点：1、服务端无状态化，可拓展性好。服务端不存放token数据，用解析token的计算时间换取存储token的空间，从而减轻服务器压力。2、支持移动端设备。
**Session和Token的区别**：1、**Session使服务端状态化**，可以记录服务端和客户端的会话状态；而**Token是令牌，使服务端无状态化**，不存储会话信息。
2、类似于OAuth Token的Token机制，提供的是**认证**和**授权**，认证针对的是用户，授权针对的是App，目的是让某App有权利访问某用户的信息；而Session只提供简单的认证，即只要有SessionId便认为拥有此用户的全部权利，因此此数据需要保密，不能共享给其它网站或第三方App。**所以如果网站需要和第三方共享用户数据或提供API给第三方调用，用Token**。<br>
**JWT(JSON Web Token)**:目前最流行的跨域认证解决方案，是一种认证授权机制。使用HMAC或者RSA加密的公私秘钥对JWT签名，因此，这些传递的信息是安全可信的。
JWT是自包含的(内部包含了一些会话信息、用户信息)，减少了查询数据库的需要；JWT若不使用cookie，可以使用任何域名访问API服务而不需要担心跨域资源共享问题；因为用户的状态不保存在服务端，所以是无状态的认证机制。
JWT认证过程：1、客户端浏览器使用用户名密码登录，服务端认证成功后，使用密钥创建JWT并返回给浏览器。2、浏览器通常将JWT保存在localStorage，也可以使用cookie(不能跨域)。3、当客户端访问受保护的资源时，在请求Header中的Authorization字段使用Bearer模式添加JWT(内容：Authorization : Bearer <token>)，使服务端可以解密检查。

### 7.String、StringBuilder、StringBuffer
**String**：Immutable的类(final class)、内部的属性也是Immutable的(final)，所以原生的保证了性能安全；但每次对String对象的裁剪、拼接等操作会产生新的对象。底层使用char数组实现，在Java9中，底层修改为byte数组和编码标识符实现，相应的StringBuilder、StringBuffer、JVM Intrinsic机制(方法内联)都做了修改。<br>
**StringBuffer**：线程安全的，通过在所有的方法上加synchronized关键字实现，因为加了锁，所以性能不如StringBuilder好。默认长度为16。<br>

### 8.Java代理机制与反射
**反射**：赋予程序在运行时**自省**(introspect)的能力，通过反射可以直接操作类或者对象，比如获取某个对象的定义、获取类声明的属性和方法、调用方法或者构造对象、甚至在运行时修改类定义。例如AccessibleObject.setAccessible(boolean flag)方法就可以在运行时修改方法的访问权限。<br>
**动态代理**：运行时构建代理、动态处理代理方法调用的机制。常用于RPC、AOP。例如RPC中，通过代理可以实现调用者和提供者之间的**解耦**，RPC框架内部的寻址、序列化、反序列化等等对于调用者来说并不关心。<br>
**实现动态代理的两种方式**：1、JDK自带的动态代理，利用了反射机制实现。通过实现InvocationHandler接口来实现。2、ASM、cglib、Javassit等高性能的字节码操作机制。cglib通过生成目标类的子类来实现(MethodInterceptor)，适用于目标类没有实现接口但还是希望被动态代理。<br>



## 五、面试记录
### 字节跳动直播中台
一面：1、自我介绍
2、算法题：序列化二叉树
3、项目难点简介
4、Redis ZSET数据结构  ziplist底层数据结构
5、MVCC介绍 

### 唯品会
一面：1、自我介绍
2、为什么把单体拆分成微服务
3、进程和线程区别，上下文切换
4、Redis为什么用哨兵模式和集群模式
5、Dubbo服务挂了怎么保证高可用
6、算法题：缺失的数字<br>
二面：1、自我介绍
2、MySQL什么情况会不走索引
3、Dubbo SPI和API区别

### 步步高
一面：1、自我介绍
2、ES倒排索引及score机制
3、Spring父子容器，bean生命周期
4、MySQL哈希索引、全文索引、聚簇索引
5、Redis使用场景
6、MyBatis DAO接口可以重载吗？二级缓存机制
7、NIO多路复用IO

### 货拉拉
一面：1、自我介绍
2、JVM内存模型，GC原理
3、线程池参数及工作原理
3、加锁有哪几种方式？synchronized和Lock区别、性能
4、SpringBoot相关，了解不多
5、Dubbo架构、协议
6、Redis使用场景？redis和zk分布式锁的区别？
7、接口的设计原则
8、sql调优的经验
9、项目难点

### 腾讯
一面：1、介绍项目难点
2、RabbitMQ的消息确认与回执机制
3、算法题：分糖果

### 粉笔
一面：1、DDD的理解
2、SpringBoot 自动配置原理
3、怎么保证Dubbo高可用
4、synchronized和Lock区别，Lock底层实现，condition类的作用
5、Redis怎么保持数据一致性
6、Redis做分布式锁如果释放锁失败怎么办
7、JVM内存模型
8、算法题：二叉树的层序遍历（广度优先）、字符串是否为另一个字符串的子序列

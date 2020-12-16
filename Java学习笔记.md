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

### 9.Redis
Redis读取速度为11W次/s，写入速度为8W次/s。<br>
**Redis支持的数据类型**：最常用的5种：String、Hash、List、Set、Sorted Set。其他：HyperLogLog、Geo等等。<br>
**Redis比Memcached的优势？**：1、Memcached所有值均是简单字符串，而Redis支持丰富的数据类型。2、Redis速度比Memcached快很多。3、Redis可以持久化数据。<br>
单个字符串可存储的最大容量为512M。<br>
**Redis的持久化机制**：1、RDB(Redis DataBase)：在某个时间点记录所有键值对，生成临时的.rdb文件，生成完毕后替换原先的.rdb文件。触发机制：自动触发(一定时间内写操作次数，60s 1W次写入、300s 100次写入、 900s 1次写入)、手动触发(save(同步)、bgsave(异步，bgsave原理：fork + copyOnWrite)) <br>


### 10.Java IO
**字节流和字符流的区别**：1、字节流读取单个字节，字符流读取单个字符（一个字符根据编码的不同，对应的字节也不同，比如UTF-8编码是3个字节，中文编码是2个字节）。2、字节流用来处理二进制文件(图片、MP3、视频等)，字符流用来处理文本文件（可以看做是使用了某种编码的二进制文件，人可以阅读）。**简而言之，字节是给计算机看的，字符是给人看的**。<br>
Java的IO机制是由BIO、NIO、AIO逐步演化来的。<br>
**BIO**：传统的java.io包，基于流实现，提供了比如文件流、输入输出流等。交互方式是同步、阻塞，所以被称为BIO(Blocking IO)。很多时候，java.net包下的部分网络API，比如Socket、ServerSocket也归到BIO中，因为网络通信同样是IO行为。<br>
**NIO**：JDK 1.4中引入了 java.nio包，提供了**Channel**、**Buffer**、**Selector**等新的抽象，可以用来构建多路复用、同步非阻塞的IO程序。<br>
**AIO**：JDK 1.7中，NIO有了进一步的改进，也就是NIO 2，引入了**异步非阻塞IO方式**,也被称为AIO(Asynchronous IO)。AIO操作基于事件和回调机制，可简单理解为：应用操作直接返回，不会阻塞在某处；当处理完成，操作系统会通知相应线程进行后续工作。<br>
BIO响应请求：一个请求对应一个后台线程，当请求太多时，启动或销毁一个线程开销是很大的。<br>
![Aaron Swartz](https://github.com/wangjc95/photos/blob/master/WechatIMG2488.jpeg) <br>
NIO响应请求：单线程轮询事件，仅select阶段是阻塞的。
![Aaron Swartz](https://github.com/wangjc95/photos/blob/master/NIO.jpeg) <br>
**选择NIO还是BIO要看具体场景**，并不是说NIO就一定好，比如，多人聊天，每次发送的数据都是很小的内容，就很适合NIO；但连接数很小、传输的数据很多的时候，适合用BIO。<br>


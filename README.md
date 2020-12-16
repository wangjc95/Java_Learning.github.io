# Java_Learning.github.io
# Java学习笔记
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

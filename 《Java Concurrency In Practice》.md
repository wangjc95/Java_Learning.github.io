### 1、对象的结构
一个Object分为三部分：对象头、实例数据、填充字节。其中填充字节是为了满足一个Java对象的大小必须是8bit的倍数这一原则。
其中对象头存放了一些Runtime信息，这些信息属于额外的开销，所以被设计的极小来提高效率；
实例数据存放了对象的属性和方法。<br>

### 2、对象头的结构
对象头分为两部分：Mark Word、Class Pointer。Class Pointer就是类型指针，指向该对象所属的类在方法区中的类信息。Mark Word结构(32bit JVM)见下图<br>
![Mark Word](https://github.com/wangjc95/photos/blob/master/Mark%20Word.png?raw=true) <br>

**锁升级**：<br>
无锁：多个线程可以并发的访问同一资源。<br>
偏向锁：当只有一个线程会拿到对象的锁时，Mark Word中会记录偏向的线程id，把锁交给这个线程。epoch为偏向锁的时间戳。<br>
轻量级锁：当多于一个线程想要获取锁时，偏向锁会升级成轻量级锁，此时Mark Word中的前30bit会存储指向虚拟机栈中锁记录的指针(ptr_to_lock_record)。当线程判断要获取的锁为轻量级锁时，会在自己私有的虚拟机栈中开辟一块名为lock record的空间，分为两部分，一部分是Mark Word的副本，一部分是owner指针，指向锁的对象。这样就实现了对象锁和线程的绑定。此时如果有其他的线程想要获取锁，需要自旋等待，自旋即CPU空转，为了提高效率，JVM优化了自旋，提供了适应性自旋，即自旋的时间不再固定，根据上一轮自旋的时间和锁状态来决定。<br>
重量级锁：当轻量级锁自旋的线程超过CPU核心数的一半或者自旋次数超过10次，轻量级锁将会升级成重量级锁，此时完全使用monitor来对线程

### 3、synchronized
被synchronized关键字修饰的代码被编译后会被 monitor enter 和 monitor exit两条字节码指令包围。monitor依赖于操作系统的mutex lock实现。

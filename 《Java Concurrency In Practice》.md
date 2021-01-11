### 1、对象的结构
一个Object分为三部分：对象头、实例数据、填充字节。其中填充字节是为了满足一个Java对象的大小必须是8bit的倍数这一原则。
其中对象头存放了一些Runtime信息，这些信息属于额外的开销，所以被设计的极小来提高效率；
实例数据存放了对象的属性和方法。<br>

### 2、对象头的结构
对象头分为两部分：Mark Word、Class Pointer。Class Pointer就是类型指针，指向该对象所属的类在方法区中的类信息。Mark Word结构见下图<br>
![Mark Word](https://github.com/wangjc95/photos/blob/master/Mark%20Word.png?raw=true) <br>

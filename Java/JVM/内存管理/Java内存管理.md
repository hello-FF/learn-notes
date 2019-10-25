# 1. Java内存管理

## 1.1 运行时数据区域

![](D:\Workspace\Java\learn-notes\Java\images\memory-area.jpg)

### 1.1.1 程序计数器

程序计数器(Program Counter Register)

### 1.1.2 虚拟机栈

[TODO]

### 1.1.3 本地方法栈

[TODO]

### 1.1.4  Java堆

[TODO]

### 1.1.5 方法区

[TODO]

### 1.1.6 元空间

[TODO]

### 1.1.7 直接内存

[TODO]

### 1.1.8 常量池

#### 1.1.8.1 Class文件常量池

[TODO]

#### 1.1.8.2 运行池常量池

[TODO]

#### 1.1.8.3 字符串常量池

[TODO]

### 1.1.9 面试相关问题

#### Java内存区域划分

* 线程共享
  * Java堆
  * 方法区
* 线程私有
  * 程序计数器
  * 虚拟机栈
  * 本地方法栈
* 直接内存
  * NIO直接内存
  * 元空间

#### 堆和栈的区别

[TODO]

## 1.2 对象的内存布局

在HotSpot虚拟机中, 对象在内存中存储的布局可以分为3块区域: 对象头(Header), 实例数据(Instance Data), 对齐填充(Padding)

![java-object-structure](D:\Workspace\Java\learn-notes\Java\images\java-object-structure.jpg)

### 1.2.1 对象头

HotSpot虚拟机的对象头包括两部分信息. 第一部分是存储对象自身的运行时数据, 如HahCode, GC分代年龄, 锁状态标志, 线程持有的锁, 偏向线程ID, 偏向时间戳等, 这部分数据的长度在32位和64位(未开启指针压缩)的虚拟中, 分别为32bit和64bit, 称为**MarkWord**. 对象头的另一部分是类型指针, 即对象指向它的类元数据的指针, 虚拟机通过这个指针确定这个对象是哪个类的实例. 如果对象是一个Java数组, 那么对象头中还必须有一块用于记录数组长度的数据, 因为虚拟机可以通过普通Java对象的元数据信息确定Java对象的大小, 但是从数组的元数据中却无法确定数组的大小.



| 状态     | 内容                                           | 标志位 |
| -------- | ---------------------------------------------- | ------ |
| 无锁     | 对象的HashCode, 对象分代年龄, 是否偏向锁(0)    | 01     |
| 偏向锁   | 偏向线程ID, Epoch, 对象分代年龄, 是否偏向锁(1) | 01     |
| 轻量级锁 | 指向栈中锁记录的指针                           | 00     |
| 重量级锁 | 指向重量级锁的指针                             | 10     |
| GC标记   | 空                                             | 11     |

**32位虚拟机对象头**

![](D:\Workspace\Java\learn-notes\Java\images\java-object-header32.jpg)

**64位虚拟机对象头**

![](D:\Workspace\Java\learn-notes\Java\images\java-object-header64.jpg)

### 1.2.2 对象的访问定位

Java程序通过栈上的reference数据操作堆上的具体对象, 目前主流的访问方式有使用句柄和直接指针两种.

使用句柄访问, Java堆中将会划分出一块内存来作为句柄池，reference中存储的就是对象的句柄地址，而句柄中包含了对象实例数据和类型数据各自的具体地址信息

![](D:\Workspace\Java\learn-notes\Java\images\java-object-reference-handle.jpg)

如果使用的是直接指针访问方式，Java 堆对象的布局中就必须考虑如何放置访问类型数据的相关信息，reference中直接存储的就是对象地址

![](D:\Workspace\Java\learn-notes\Java\images\java-object-reference-direct.jpg)

使用句柄访问的最大好处是reference中存储的是稳定的句柄地址, 在对象被移动()时, 只会改变句柄中的实例数据指针, 而reference不需要改变.

使用直接指针访问方式的最大好处是速度更快, 节省了一次指针定位的时间开销.

Hotspot虚拟机使用的是直接指针访问.

## 1.3 实战




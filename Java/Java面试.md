#  Java面试

## Java

### 概念类

#### 面向对象

##### 面向对象设计七大原则

* 单一职责原则

  每个类专注做一件事

* 里氏替换原则

  超类存在的地方, 子类是可以替换的

* 依赖倒置原则

  实现尽量依赖抽象, 不依赖具体实现

* 接口隔离原则

  应当为客户提供尽可能小的接口, 而不是提供大的总的接口

* 迪米特法则(最小知识原则)

  一个软件应尽可能小的与其他实体发生相互作用

* 开闭原则

  面向扩展开放, 面向修改关闭

* 组合/聚合复用原则

  尽量使用合成/聚合达到复用, 尽量少用继承, 原则: 一个类中有另一个类的对象

##### Java面向对象的特征

* 抽象

* 封装
* 继承
* 多态

#### 设计模式

[TODO]

### JVM

#### 内存

##### Java内存结构

* 线程私有
  * 程序计数器
    * 当前线程字节码行号指示器
    * 唯一没有规定内存溢出区
  * 虚拟机栈
    * 描述Java方法执行的内存模型
    * 每个方法执行都会创建一个栈帧, 用于存储局部变量表, 操作数栈, 动态链接, 方法出口信息
    * 每个方法从调用到执行完成(异常), 就对应一个栈帧在虚拟机栈中的入栈到出栈过程
    * 栈帧随方法调用而创建, 方法结束而销毁, 无论方法是正常完成还是抛出异常
  * 本地方法栈
    * 与虚拟机栈类似, 为执行native方法服务
* 线程共享
  * 堆内存
    * Java对象在内存的存放区域
    * 从GC角度分为新生代和老年代
  * 方法区
    * 存放JVM加载的类信息, 常量, 静态变量
    * 从GC角度称为永久代
    * JDK8后使用元空间代替方法区
* 直接内存
  * NIO直接内存
  * 元空间



##### Java对象在内存中的结构

![java-object-structure](C:\Users\wolfc\Documents\学习\Java\images\java-object-structure.jpg)

* 对象头
* 实例数据
* 对象填充

#### 类加载

##### 类加载过程

1. 加载
   * 从本地系统加载
   * 从网络加载加载
   * 从zip, jar包中加载
   * 从专有数据库加载
   * 将Java源文件动态编译后加载
2. 连接
   * 验证
     * 文件格式验证
     * 元数据验证
     * 字节码验证
     * 符号引用验证
   * 准备
     * 为静态变量分配内存空间, 并初始化为类型默认值
   * 解析
     * 符号引用替换为直接引用
3. 初始化
   * 为类的静态变量赋予正确的初始值
   * 执行静态代码块

##### 内置类加载器

###### BootstrapClassLoader

引导类加载器, 由C++编写的类加载器, 由虚拟机启动加载, 主要负责加载` %JAVA_HOME%/jre/lib`, ` %JAVA_HOME%/jre/classes `和`-Xbootclasspath `参数指定目录中的类, 由BootStrapClassLoader加载的类`getClassLoader`时通常返回null.

###### ExtClassLoader

扩展类加载器, 全名`  sun.misc.Launcher$ExtClassLoader `, 父加载器为`BootstrapClassLoader`,  主要加载` %JAVA_HOME%/jre/lib/ext `目录和` java.ext.dirs `系统变量指定的目录下的类.

###### AppClassLoader

应用类加载器/系统类加载器, 全名` sun.misc.Launcher$AppClassLoader `, 父加载器为`ExtClassLoader`, Java默认的类加载器, `ClassLoader.getSystemClassLoader`返回的默认实现, 负责加载classpath目录和jar包中的类.

##### 双亲委托机制

###### 概念

类加载器在加载类时, 首先将加载任务委托给父类加载器, 依次递归, 如果存在父类加载器完成类加载任务, 就成功返回, 当父类加载无法加载类时, 才自己去加载.

###### 意义

* 防止同一个类被加载多次, 相同的类如果被不同类加载器加载的, 会被JVM认为是不同类型
* 为了系统安全, rt.jar包下的核心类, 都由`BootStrapClassLoader`加载

##### 线程上下文类加载器

###### 概念

`java.lang.Thread#getContextClassLoader`

`java.lang.Thread#setContextClassLoader`

每个Java线程都有一个关联的类加载器, 可以通过`Thread#setContextClassLoader`设置当前线程的上下文类加载器, 如果没有主动设置, 默认使用父线程的的类加载器. 使用线程上下文类加载器可以破坏双亲委托机制, 常应用于SPI机制, JNDI, WEB容器中.

###### 使用方法

```java
ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
try{
	Thread.currentThread().setContextClassLoader(targetTccl);//设置上下文类加载
	excute();// 执行相关类加载请求
} finally {
	Thread.currentThread().setContextClassLoader(classLoader);// 还原
}
```

线程上下文类加载器使用一般分为获取 -> 设置 -> 使用 -> 还原

###### 应用场景举例

当高层定义的接口由低层进行实现, 并需要由高层进行加载低层的实现类时, 就需要借助线程上下文类加载器来帮助高层加载这些类, 常见的举例就有JDBC4以后的`DriverManager`.

JDBC4以前, 加载jdbc驱动需要开发人员通过`Class.forName("xxx.Driver")`来装载驱动, JDBC4开始, 基于SPI机制来加载驱动, 驱动提供商通过`META-INF/services/java.sql.Driver`文件中指定实现类, 暴漏驱动即可.

```java
private static void loadInitialDrivers() {
    //...省略代码
    // If the driver is packaged as a Service Provider, load it.
    // Get all the drivers through the classloader
    // exposed as a java.sql.Driver.class service.
    // ServiceLoader.load() replaces the sun.misc.Providers()

    AccessController.doPrivileged(new PrivilegedAction<Void>() {
        public Void run() {

            ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
            Iterator<Driver> driversIterator = loadedDrivers.iterator();

            /* Load these drivers, so that they can be instantiated.
                 * It may be the case that the driver class may not be there
                 * i.e. there may be a packaged driver with the service class
                 * as implementation of java.sql.Driver but the actual class
                 * may be missing. In that case a java.util.ServiceConfigurationError
                 * will be thrown at runtime by the VM trying to locate
                 * and load the service.
                 *
                 * Adding a try catch block to catch those runtime errors
                 * if driver not available in classpath but it's
                 * packaged as service and that service is there in classpath.
                 */
            try{
                while(driversIterator.hasNext()) {
                    driversIterator.next();
                }
            } catch(Throwable t) {
                // Do nothing
            }
            return null;
        }
    });
    //...省略部分代码
}
```

以上是中在`DriverManager`静态代码块中执行的一段代码, 用于自动加载项目中的jdbc驱动. `DriverManager`是由在`rt.jar`包的类, 根据Java的类加载机制, `DriverManager`会由`BootStrapClassLoader`加载,而`BootStrapClassLoader`不能加载第三方服务提供的jar, 为了达到加载目的, 这里就使用了SPI机制, 其原理就是线程上下文类加载器.

```java
public static <S> ServiceLoader<S> load(Class<S> service) {
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    return ServiceLoader.load(service, cl);
}
```



##### 如何修改默认系统类加载器

系统类加载器第一次加载时通过参数获取, `SystemClassLoaderAction`

```java
System.getProperty("java.system.class.loader")
```

所以要修改, 只需要设置其参数即可

```
java -Djava.system.class.loader=xxx.XXXClassLoader
```

#### GC

##### 确定垃圾算法

###### 引用计数法

在对象头维护了一个counter, 每增加一次对对象的引用计数加1, 如果对该对象的引用失联, 则计数减1, 当counter为0时, 表示该对象废弃. 这种方法无法区分软, 虚, 弱, 强引用类别, 另一方面, 两个对象相互引用, 则counter永不为0 , 对象永远不会被GC释放

###### 根搜索算法/可达性分析

通过一系列的GC Roots对象作为起点, 从这些节点向下搜索, 搜索所走过的路径称为引用链, 当一个对象到GCRoots没有任何引用链时, 则证明对象不可用.

如果对象在进行可达性分析后, 发现没有与GCRoots相连的引用链, 也不会被立刻理解为死亡, 它会暂时被标记上并且进行一次筛选, 筛选的条件是是否与必要执行finalize()方法. 

如果被判定有必要执行finaliza()方法, 就会进入F-Queue队列中, 并有一个虚拟机自动建立的, 低优先级的线程去执行它. 稍后GC将对F-Queue中的对象进行第二次小规模标记. 

如果这时还是没有新的关联出现, 那基本上就真的被回收了.

##### 垃圾回收算法

###### 标记清除算法

最基础的垃圾回收算法, 分为标记阶段和回收阶段, 缺点是内存碎片化严重.

![](C:\Users\wolfc\Documents\学习\Java\images\java-gc-mark-sweep.png)

###### 复制算法

将内存分为等大小的两块, 每次只使用其中一块, 当这一块内存满后, 将尚存活的对象复制到另一块上, 把已使用的内存回收清除.

![](C:\Users\wolfc\Documents\学习\Java\images\java-gc-copying.png)

###### 标记整理算法(Mark-Compact)

结合以上两种算法, 标记后, 不是清除对象, 而是将尚存活的对象移到内存的另一端, 然后清除边界外的对象.

![](C:\Users\wolfc\Documents\学习\Java\images\java-gc-mark-compact.png)

###### 分代收集算法

目前大部分JVM所采用的算法, 根据对象存活的不同生命周期将内存划分为不同的区域, 一般情况下, 将GC堆划分为老年代和新生代, 老年代的对象存活时间久, 每次回收只有少量对象被回收, 新生代的对象存活时间短, 每次需要回收大量的对象, 因此, 不同区域选择不同的算法.

* 新生代-复制算法

  新生代每次回收垃圾多. 存活少, 因此复制操作少, 一般将新生代划分为Eden, From ,To三块区域

*  老年代-标记整理算法

  老年代每次回收垃圾少, 存活久, 因此使用标记整理算法

###### 分区收集算法

[TODO]

##### 垃圾回收过程

[TODO]

##### 垃圾收集器

[TODO]

##### CMS垃圾收集器

[TODO]

##### G1垃圾收集器

[TODO]

##### CMS和G1垃圾收集器的优劣

[TODO]

#### JVM调优

##### Java分析工具

###### jvisualvm

连接JVM, 监视系统运行

![](C:\Users\wolfc\Documents\学习\Java\images\java-jvisualvm.png)

###### jconsole 

基于JMX的图形化管理工具, 连接远程JVM, 查看内存, 线程, 使用状态, 还可以跟踪Java中类的状态和MBean信息

![](C:\Users\wolfc\Documents\学习\Java\images\java-jconsole.png)

###### jmap

**参数**

* -heap 打印堆的概要信息, GC算法, 堆配置信息
* -histo 打印每个class的实例数目, 类全名, 内存占用信息
* -clstats 打印类加载器信息, 名称, 活跃度, 地址, 父加载器, 加载的类的数量和大小信息
* -finalizerinfo 打印等待回收的类信息
* -dump 以hprof二进制格式转储Java堆到指定filename的文件中

###### jstat

* -class
* -compiler
* -gc
* -gccapacity
* -gccause
* -gcmetacapacity
* -gcnew
* -gcnewcapacity
* -gcold
* -gcoldcapacity
* -gcutil
* -printcompilation

###### jstack

java提供的线程堆栈跟踪工具

##### JVM常用参数

* 堆内存
  * 通用
    * -Xms 堆的最小空间
    * -Xmx 堆的最大空间
  * 新生代
    * -Xmn 设置年轻代大小
    * -XX:NewSize 新生代最小空间
    * -XX:MaxNewSize 新生代最大空间
  * 老年代
* 栈内存
  * -Xss 设置每个线程的堆栈大小 
* 方法区内存(jdk7)
  * -XX:PermSize 永久代最小空间
  *  -XX:MaxPermSize 永久代最大空间
* 元空间内存(jdk8)
* GC

##### JVM性能调优

[TODO]

### 基础

#### Object

##### Object类的方法

* hashCode: 计算对象的hash值
* equals: 比较两个对象是否相等
* toString: 返回对象的字符串形式
* getClass: 返回对象的Class对象
* clone: 复制对象
* wait: 阻塞当前线程, 需要在synchronized包裹的代码块或方法中执行
* wait(long timeOut): 具有超时的wait机制
* notify: 随机唤醒阻塞在当前对象监视器锁上的线程, 重新竞争锁
* notifyAll: 唤醒所有阻塞在当前对象监视器锁上的线程, 重新竞争锁
* finalize: GC回收前调用

#### String

##### String的特点

* final修饰的类, 不能被继承
* String对象是不可变的, 每次修改都会重新生成一个对象
* String对象创建时会放入常量池中, 重复对象会直接使用常量池对象的引用

##### String的缺点

* 循环拼接/修改字符串会创建大量对象, 浪费系统资源, 影响性能

##### String为什么要设计为不可变的

* 不可变类比较简单
* 线程安全
* String内部通过char数组实现, 数组长度是不可变的

##### String如何实现不可变的

* String类使用final修饰, 保证不能通过继承修改对象
* String内部value数组属性使用final修饰
* String提供的修改方法`substring` `replace` `replaceAll` `concat`都会创建新的对象

##### String是否真的不可变

​	否, 可以通过反射获取到value属性, 修改数组中的元素改变String对象的内容

```java
String str = "abc";
System.out.println(str);//abc
Field valueField = String.class.getDeclaredField("value");
valueField.setAccessible(true);
char[] value = (char[]) valueField.get(str);
value[0] = 'd';
System.out.println(str);//dbc
valueField.set(str, new char[]{'e', 'f'});
System.out.println(str);//ef
```

##### String的hashCode方法实现

```java
public int hashCode() {    
    int h = hash;    
    if (h == 0 && value.length > 0) {        
        char val[] = value;        
        for (int i = 0; i < value.length; i++) {           
            h = 31 * h + val[i];        
        }        
        hash = h;    
    }    
    return h;
}
```

数学公式: `s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]`

##### String的equals方法实现

```java
public boolean equals(Object anObject) {
    if (this == anObject) {
        return true;
    }
    if (anObject instanceof String) {
        String anotherString = (String)anObject;
        int n = value.length;
        if (n == anotherString.value.length) {
            char v1[] = value;
            char v2[] = anotherString.value;
            int i = 0;
            while (n-- != 0) {
                if (v1[i] != v2[i])
                    return false;
                i++;
            }
            return true;
        }
    }
    return false;
}
```

1. 比较两个对象地址是否相同, ==比较
2. 判断目标对象是不是String对象, instanceof比较
3. 比较value数组长度是否相等
4. 依次比较value数组中的每次元素是否相同

### 集合

#### Array

##### 数组特点

* 数组的Class实例是由JVM运行时生成的

* 连续的内存空间
* 数组内存对象头中有一个length属性表示数组长度

##### 数组是否可变

​	数组长度不可变, 数组内的元素可变

##### 数组扩容

​	数组长度不可变, 要实现扩容需要重新创建一个数组对象

* Arrays.copyOf
* System.arraycopy

#### List

##### ArrayList实现原理

​	ArrayList内部维护了一个Object数组, 通过数组复制实现动态扩容

##### LinkedList实现原理

​	双向链表

##### ArrayList, LinkedList, Vector的区别

* 实现区别
  * ArrayList, Vector数组实现
  *  LinkedList链表实现
* 线程安全
  * ArrayList, LinkedList非线程安全
  * Vector通过在方法上加synchronized实现线程安全
* 读取区别
  * ArrayList, Vector通过数组下标定位元素, 下标读取性能高
  * LinkedList下标读取需要遍历链表, 性能低
* 插入, 删除区别
  * ArrayList, Vector插入, 删除涉及数组的扩容和移动, 性能低
  * LinkedList通过链表的断开, 连接实现插入删除, 性能高
* 遍历区别
  - ArrayList, Vector实现了`RandomAccess`接口, 支持随机访问, 使用for遍历性能高
  - LinkedList不支持随机访问, 使用迭代器遍历性能高

##### 线程安全的List

* Vector
* Collections.synchronizedList
  * SynchronizedRandomAccessList
  * SynchronizedList
* CopyOnWriteArrayList

#### CopyOnWriteArrayList

##### 特点

* 读不加锁, 写加锁
* 每次写入都需要复制容器, 性能低

##### 原理

**读:**

```java
public E get(int index) {
    return get(getArray(), index);
}
```

普通读, 不加锁

**写:**

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

1. 内部维护了ReentrantLock, 每次写操作加锁
2. 复制扩容新的数组
3. 添加的元素写入到新数组
4. 替换旧数组

#### Set

##### Set和List的区别

* List可重复, Set不允许重复
* List是有序的, HashSet无序, LinkedHashSet有序
* List通过数组或链表实现, HashSet使用HashMap实现

##### HashSet实现原理

​	内部维护了一个HashMap, 插入的元素为HashMap的key, value为HashSet的一个Object常量

#### Map

##### Hashtable实现原理

[TODO]

##### TreeMap实现原理

[TODO]

##### HashMap实现原理(jdk7)

[TODO]

##### HashMap实现原理(jdk8)

* 数据结构

  数组 + 链表 + 红黑树

* 初始化

  初始化**数组长度**为大于等于传入长度的2的n次方, 默认是16, 注意仅是数组长度不是数组本身

  > 这里计算数组长度是一个精妙的位运算
  >
  > ```java
  > static final int tableSizeFor(int cap) {
  >        int n = cap - 1;
  >        n |= n >>> 1;
  >        n |= n >>> 2;
  >        n |= n >>> 4;
  >        n |= n >>> 8;
  >        n |= n >>> 16;
  >        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
  > }
  > ```

* 插入

  1. 第一次插入时会创建数组对象, 初始化数组本身
  2. 将key的hashcode进行运算后, 对数组长度取余(实际代码里使用的是位运算), 计算出key在数组中的位置
  3. 如果当前数组元素为空, 直接插入新增内容, 
  4. 如果当前数组元素为链表, 遍历链表, 如果存在key值与链表内元素equals比较相等, 替换旧元素, 否则将新增内容插入链表末尾, 当链表元素个数大于等于8个时, 执行链表转为红黑树
  5. 如果当前数组元素为红黑树, 遍历红黑树,  如果存在key值与树内元素equals比较相等, 替换旧元素, 根据红黑树规则插入新增内容, 并调整红黑树
  6. 插入新增内容后, Map内元素总个数达到阙值, 将执行数组扩容操作

* 扩容

  1. 创建新的数组, 数组长度新增一倍, 开始遍历数组, 迁移旧数组内的元素
  2. 如果数组元素为空, 不处理
  3. 如果数组元素为链表, 遍历链表, 将链表内的元素key值与新数组长度重新取余, 计算元素在新数组所在的位置, 组成新的两条链表, 一条位置`保持不变`, 一条位置在`旧位置+旧数组长度`
  4. 如果数组元素为红黑树, 执行红黑树工作, 与链表类似, 同样拆分成两条新树, 但如果新树的元素个数不足`6`个时, 会将树转换为链表
  5. 迁移工作完成后, 将新数组对象替换旧数组

* 删除

  1. 根据要删除的key值的hashcode, 计算出key所在数组的位置
  2. 如果数组元素为链表, 遍历链表, 找出与key值相等的元素, 删除元素
  3. 如果数组元素为红黑树, 遍历树, 找出与key值相等的元素, 删除元素后, 重新调整树, 如果元素个数已不能保持红黑树的特征(一般3<=元素个数<=6), 会将树转换为链表.

##### HashMap转换红黑树时机为什么是8

[TODO] 与红黑树和链表的遍历有关

##### Hashtable和HashMap的区别

* 使用:
  * Hashtable所有方法使用synchronized修饰, 属于线程安全, HashMap非线程安全
  * Hashtable的key和value都不允许为null, HashMap允许, key为null时, hash值按0计算
* 实现:
  * Hashtable结构是数组+链表, HashMap是数组+链表+红黑树(jdk8+)
  * HashMap数组长度限定为2的n次方, Hashtable不限制
  * Hashtable数组默认长度是`11`, HashMap默认是`16`
  * 计算key所在数组位置时,
    * Hashtable将hash取绝对值(与运算作用)后对数组长度取余: `(hash & 0x7FFFFFFF) % tab.length`
    * HashMap将hash运行异或和位运算后, 再对数组长度取余(位运算效果等同取余), `(n - 1) & hash`
  * Hashtable在创建对象时初始化数组, HashMap在第一次插入时
  * 链表插入时, Hashtable前插法(新元素链表最前端), HashMap后插法(新元素在链表最末端)
  * Hashtable在插入前判断扩容, HashMap在插入后
* 代码:
  * Hashtable继承Dictionary, HashMap继承AbstractMap

#### ConcurrentMap

##### ConcurrentHashMap实现原理(jdk7)

[TODO]

##### ConcurrentHashMap实现原理(jdk8)

* 数据结构

  数组 + 链表 + 红黑树

* 初始化

  将`sizeCtl`属性设值为将要初始化的数组长度, 但不初始化数组本身, 后续`sizeCtl`属性还将用于数组扩容

* 插入

  1. 判断key值是否`null`, 只允许插入非空的key
  2. 第一次插入会初始化数组, 长度为`sizeCtl`, 通过`CAS`操作将`sizeCtl`设值为`-1`, 成功的线程将执行数组扩容, 不成功的线程将循环判断等待(`Thread.yield`)直到有线程完成初始化操作, 初始化线程会创建`table`数组对象, 并将`sizeCtl`重新设值为下一次扩容所需的阙值, table长度的0.75, `n - (n >>> 2)`
  3. 根据key值的hashcode运算后对数组取余, 定位key在数组中的位置, 开始循环判断插入, 直到插入成功
  4. 如果数组元素为空, 通过`CAS`插入元素, 成功退出循环, 失败重新判断插入
  5. 如果数组元素为链表或红黑树, 通过`synchronized`同步锁执行插入, 获得锁的线程, 根据数组元素是链表还是红黑树, 执行插入操作, 同`HashMap`一样, 这里也存在链表转红黑树的操作
  6. 如果数组元素为扩容元素(`hash==-1`), 执行`helpTransfer`, 参与扩容
  7. 插入完成后, 通过`addCount`方法, 将`counterCells`元素个数加1,  并判断是否需要扩容

* 扩容

  1. 通过`CAS`修改`sizeCtl`为`-2`, 修改成功的线程将创建`nextTable`, 长度新增一倍, 失败的线程不再创建nextTable, 将通过`CAS`将`sizeCtl-1`参与协助扩容, 此时`sizeCtl <= -2`, `-sizeCtrl - 1`等于参与扩容的线程数
  2. 计算每个线程每次需要迁移的个数`stride = (NCPU > 1) ? (n >>> 3) / NCPU : tab.length`, 最少`16`个
  3. 计算当前线程迁移的元素索引`i`, 默认`transferIndex-1`, 从数组末尾开始迁移, `CAS`修改`transferIndex`, 帮助其它线程确定他们线程的迁移位置
  4. 如果数组元素为空, 通过`CAS`修改当前节点为`ForwardingNode`节点, `CAS`的作用是防止迁移线程和其它插入线程并发操作.
  5. 如果数组元素为链表或红黑树,  通过`synchronized`同步锁执行迁移, 获得锁的线程, 执行迁移工作
  6. 如果数组元素为链表, 遍历链表, 重新计算元素位置, 在新数组中`i`位置和`i+tab.length`形成新的链表, 将原数组该位置元素修改为`ForwardingNode`
  7. 如果数组元素是红黑树, 遍历树,执行树的迁移工作, 如果新树的元素个数不足`6`时, 将树转换为链表, 将原数组该位置元素修改为`ForwardingNode`
  8. 迁移完成, 更新`table`指向`nextTable`, 并更新`sizeCtl`为新数组大小的0.75倍.

* 删除

  [TODO]



##### ConcurrentHashMap的元素个数获取

```java
 final long sumCount() {
     CounterCell[] as = counterCells; CounterCell a;
     long sum = baseCount;
     if (as != null) {
         for (int i = 0; i < as.length; ++i) {
             if ((a = as[i]) != null)
                 sum += a.value;
         }
     }
     return sum;
 }
```

内部维护了一个CounterCell数组, 数组长度等于当前内容数组的长度, 数组的元素保存了数组的个数, 但是注意, 该数组每个位置并不和`table`数组对应, 是插入线程的随机因子`ThreadLocalRandom.getProbe`对`CounterCell`数组取余的位置

**为什么使用这种方式获取元素个数**?

减少并发冲突, 如果使用一个变量记录, 高并发下, 频繁的CAS`修改将会带来严重的性能影响

#### Queue

##### ArrayBlockingQueue 

基于数组实现的有界阻塞队列, 遵循FIFO(先进先出)原则

[TODO]

##### LinkedBlockingQueue

基于链表实现的有界阻塞队列, 遵循FIFO原则

[TODO]

##### PriorityBlockingQueue

支持优先级排序的无界阻塞队列

[TODO]

##### DelayQueue

使用优先级队列实现的无界阻塞队列

[TODO]

##### SynchronousQueue

> 经常与线程池newCachedThreadPool一起面试

不存储元素的阻塞队列

[TODO]

##### LinkedTransferQueue

基于链表实现的无界阻塞队列

[TODO]

##### LinkedBlockingDeque

基于链表实现的双向阻塞队列, 支持FIFO和FILO两种模式

[TODO]

### 并发编程

#### 线程

##### 线程状态

![](C:\Users\wolfc\Documents\学习\Java\images\thread-state.jpeg)

* NEW: 新建
* RUNNABLE: 运行/就绪
* BLOCKED: 阻塞
* WAITING: 等待
* TIMED_WAITING: 超时等待
* TERMINATED: 终止

##### 等待线程结束的方法

1. 
2. Thread.join()

##### 控制线程按顺序执行

1. Thread.sleep()
2. Thread.join()

#### happens-before

##### 规则

1. 程序次序规则: 一个线程内, 按照代码顺序执行, 书写在前面的操作先于书写在后面的操作
2. 锁定规则: 释放锁操作先行发生于后面对同一个锁的加锁操作
3. volatile规则: 对一个变量的写先行发生于后面对这个变量的读操作
4. 传递规则: 如果A操作先发生于B操作, B操作先发生于C操作, 则A操作先行发生于C操作
5. 线程启动规则: Thread对象的`start`方法先行发生于此线程的每一个动作
6. 线程中断规则: 对线程`interrupt`方法的调用先行发生于被中断线程的代码检测到中断事件
7. 线程终结规则: 线程中的所有操作先行发生于线程的终止检测, 我们可以通过`Thread.join`方法结束, `Thread.isAlive`的返回值手段检测到线程已经终止执行
8. 对象终结规则: 一个对象的初始化完成先行于它的`finalize`方法开始

#### 锁

##### synchronized

![](C:\Users\wolfc\Documents\学习\Java\images\java-synchronized.jpg)

JVM底层通过修改内存中对象的`markword`部分来实现锁和锁的升级

**无锁**

`markword`标记位为`01`,  偏向锁标记为`0` ,并记录对象的`hash`值和`age`, 线程在无锁的情况下获取锁时, 会通过`CAS`修改`markword`, 将其它一部分修改为当前线程ID, 修改成功则获得偏向锁, 并将偏向标志设置为1.

**偏向锁**

`markword`标记位为`01`,  偏向锁标记为1, 记录偏向线程ID, 当存在两个以上线程争抢锁时,  会开始偏向锁的撤销, 在原持有偏向锁线程达到全局安全点时, 会暂停工作线程, 升级为轻量级锁.

**轻量级锁**

`markword`标记位为`00`, 记录当前锁记录的指针. 在这个锁记录, 线程通过`CAS`修改`markword`锁记录的指针来获得锁, 未获得锁的线程开始自旋, 当自旋达到一定次数后, 升级为重量级锁. 

**重量级锁**

`markword`标记位为`10`, 记录monitor锁的指针, 当达到重量级锁, 其它未获得锁的线程会被挂起, 直到被唤醒继续争抢锁.

##### ReentrantLock

基于`AQS`实现的可重入锁锁, 分为公平模式和非公平模式, 分别对应两个AQS实现, 通过`CAS`修改`AQS`内的`state`属性来达到加锁和释放锁的目的. 

**原理:** 

1. `CAS`将`state`设置为`1`, 成功的线程持有锁, 失败则继续判断.
2. 失败的线程再次判断`state`为0时(锁已释放), 会再次尝试`CAS`加锁. (非公平和公平模式区别)
3. 如果是持有锁线程重入, `state+1`, 记录重入次数. (可重入提现)
4. 如果`state`不为0, 或者第二次`CAS`修改操作还是失败, 将线程入队, 调用`LockSupport.park`阻塞线程.
5. 线程释放锁, `state-1`, 当`state` 等于0时, 才会释放锁, 调用`LockSupport.unpark`唤醒等待队列中的线程, 重新争抢锁.

#### AQS

[TODO]

#### volatile

##### volatile作用

* 保证多线程之间共享变量的可见性
* 保证基本数据类型赋值的原子性
* 禁止指令重排序

##### volatile实现原理

volatile底层通过lock指令, 添加内存屏障

- 对volatile变量的写操作会立即刷新到主存
- 对volatile修饰的写操作会使缓存无效
- 阻止内存屏障的前后指令重排序

#### ThreadPoolExecutor

##### 构造参数

* corePoolSize 核心线程数
* maximumPoolSize 最大线程数
* keepAliveTime 空闲线程存活时间
* unit 时间单位
* workQueue 工作队列
* threadFactory 线程工厂
* handler 拒绝策略

##### 工作原理

1. 外部线程提交任务, 线程池创建核心线程执行任务, 达到核心线程数之前, 提交一个任务创建一个核心线程.
2. 达到核心线程数, 提交任务将被提交到队列中.
3. 当队列满后, 创建非核心线程执行任务.
4. 非核心线程达到最大线程数限制后, 且任务队列已满后, 使用拒绝策略拒绝任务提交.

##### 最佳线程数设置原则

* CPU密集型

  CPU密集型主要是计算任务, 需要提高CPU的使用率, 避免线程上下文切换 开销.

  核心线程数: CPU核心数

  最大线程数: CPU + 1

* IO密集型

  IO密集型线程池, 由于常常需要IO等待, CPU使用率不高, 所以可以提高线程数量, 在线程等待期间, 有其他线程处理任务, 提高CPU使用率.

  最大线程数: 2CPU+1

* 混合型

  混合型任务比较复杂, 需要根据实际情况调整, 原则上线程等待时间越长, 越需要提高线程数量

  计算公式: ( ( 线程等待时间+线程CPU时间 ) / 线程CPU时间 ) * CPU数目

##### 4种拒绝策略

*  **AbortPolicy**

  抛出`java.util.concurrent.RejectedExecutionException`异常

  ```java
  public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
      throw new RejectedExecutionException("Task " + r.toString() +
                                           " rejected from " +
                                           e.toString());
  }
  ```

  

* **DiscardPolicy**

  直接丢弃任务, 不执行任何操作

  ```java
  public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
  }
  ```

* **DiscardOldestPolicy**

  丢弃队尾的任务, 提交当前任务

  ```java
  public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
      if (!e.isShutdown()) {
          e.getQueue().poll();
          e.execute(r);
      }
  }
  ```

* **CallerRunsPolicy**

  直接使用调用线程执行任务

  ```java
  public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
      if (!e.isShutdown()) {
          r.run();
      }
  }
  ```


#### ForkJoinPool

[TODO]

#### Executors

>  Java提供的线程池创建工具类

##### Java内置提供了几种线程池

* **newFixedThreadPool** 固定线程数量
  * corePoolSize = maximumPoolSize = n
  * keepAliveTime = 0
  * LinkedBlockingQueue
* **newCachedThreadPool** 缓存线程池
  * corePoolSize = 0, maximunPooSize = Integer.MAX_VALUE
  * keepAliveTime  = 60s
  * SynchronousQueue
* **newSingleThreadExecutor** 单线程线程池
  * corePoolSize = maximumPoolSize = 1
  * keepAliveTime = 0
  * LinkedBlockingQueue
* **newScheduledThreadPool** 调度线程数
  * corePoolSize = n, maximunPooSize = Integer.MAX_VALUE
  * keepAliveTime  = 0
  * DelayedWorkQueue
* **newWorkStealingPool** 基于工作窃取算法的线程池
  * parallelism = CPU核心数

##### newCachedThreadPool使用的是哪种队列

​	SynchronousQueue. 队列详情请往前翻

## Spring

### SpringFamework

#### 概念

##### DIP, IOC, DI, IoC容器, AOP

* DIP 

  依赖倒置原则, 软件架构设计原则

* IoC

  控制反转, 遵循DIP原则的一种思想或者说是设计模式

* DI

  依赖倒置, 实现Ioc的一种方法

* IoC容器

  DI框架

* AOP

  面向切面编程

##### Spring中的设计模式

[TODO]

#### 流程

##### Spring启动流程

[TODO]

##### SpringBean的生命周期

[TODO]

#### AOP

##### 动态代理

###### Proxy

[TODO]

###### CGLIB

[TODO]

##### 源码实现

[TODO]

#### 事务

##### 事务传播级别

* PROPAGATION_REQUIRED: 支持当前事务, 没有就新建一个事务
* PROPAGATION_SUPPORTS: 支持当前事务, 没有就以非事务执行
* PROPAGATION_MANDATORY: 支持当前事务, 没有就抛出异常
* PROPAGATION_REQUIRES_NEW: 新建事务, 当前存在事务挂起
* PROPAGATION_NOT_SUPPORTED: 以非事务执行, 当前事务挂起
* PROPAGATION_NEVER: 以非事务执行, 当前存在事务就抛出异常
* PROPAGATION_NESTED: 当前存在事务就以嵌套事务执行

##### 实现原理

[TODO]

#### WebMVC

##### DispatchServlet流程

[TODO]

##### ControllerAdvice原理

[TODO]

### SpringBoot

#### 基础

##### SpringBoot的特性

* 独立的Spring应用
* 嵌入式Web容器
* 提供"starter"依赖, 简化配置
* 自动装配
* 提供运维特性, 如指标信息, 健康检查, 外部化配置
* 无代码生成, 不需要XML配置

##### SpringBoot和SpringMVC比较

* SpringBoot内置Web容器, SpringMVC依赖外部容器
* SpringBoot配置简单, 通过自动装配starters
* 提供运维特性

#### 原理

##### SpringBoot启动流程

1. 调用`静态`方法`SpringApplication#run(Class<?>, String...)`创建`SpringApplication`实例, 执行实例方法`SpringApplication#run(String...)`, 开始启动应用
2. 执行`SpringApplication#getRunListeners`创建`SpringApplicationRunListeners`实例
   1. 通过SpringFactoriesLoader加载`META-INF/spring.factories`文件, 获取文件中配置的`SpringApplicationRunListener`类
   2. 以SPI机制加载`SpringApplicationRunListener`的`Class`内容到内存中
   3. 通过反射创建所有的`SpringApplicationRunListener`对象
3. 发布`ApplicationStartingEvent`事件
4. 执行`prepareEnvironment`, 准备SpringBoot执行环境
   1. 根据`webApplicationType`创建`environment`
      * SERVLET: StandardServletEnvironment
      * REACTIVE: StandardReactiveWebEnvironment
      * default: StandardEnvironment
   2. 从系统变量和执行命令配置环境参数
   3. 发布`ApplicationEnvironmentPreparedEvent`事件
      * `ConfigFileApplicationListener`, 加载`application.properties`
5. 创建`ApplicationContext`实例, 如果指定`applicationContextClass`属性, 按指定对象创建, 否则按`webApplicationType`类型创建
   * SERVLET: AnnotationConfigServletWebServerApplicationContext
   * REACTIVE: AnnotationConfigReactiveWebServerApplicationContext
   * default: AnnotationConfigApplicationContext
6. 执行`prepareContext`准备上下文信息
   1. [TODO]
7. 执行`refreshContext`刷新上下文
   1. [TODO]
8. 执行`afterRefresh`上下文刷选完成处理
9. 发布`ApplicationStartedEvent`事件

##### SpringBoot自动装配原理

[TODO]

## 微服务

### 概念

### Dubbo

#### 基础

### SpringCloud

#### 基础

##### SpringCloud常用核心组件

* 注册中心
  * Spring Cloud Netflix Eureka
  * Spring Cloud Console
* 配置中心 Spring Cloud Config
* 服务调用 Spring Cloud Open Feign
* 负载均衡 Spring Cloud Netflix Ribbon
* 网关
  * Spring Cloud Netflix Zuul
  * Spring Cloud Gateway
* 熔断 Spring Cloud Netflix Hystrix
* 调用追踪 Spring Cloud Sleuth + Zipkin
* 消息总线 Spring Cloud Bus
* 消息驱动 Spring Cloud Stream

#### Eureka

##### 基本概念

![](C:\Users\wolfc\Documents\学习\Java\images\spring_cloud_eureka.png)

Eureka遵循AP理论设计, 采用Rest接口协议.

**服务对象及功能**

* 服务提供者
  * 服务上线
  * 服务下线
  * 服务续约
* 服务消费者
  * 获取服务列表
* 注册中心自身/集群
  * 剔除过期服务
  * 同步注册信息到集群中其它节点
* 其它
  * 强制修改服务状态

##### Eureka工作流程



##### Eureka自我保护机制





## 数据结构

### 数组

### 链表

### 树

#### 二叉树

#### B+树

#### 红黑树

### 排序算法

#### 冒泡排序

##### 算法描述

1. 比较相邻元素, 如果第一个比第二个大, 就交换他们两个
2. 依次向后比较相邻元素, 直到最大的元素排到末尾
3. 从头重复以上步骤

##### 动图演示

![](C:\Users\wolfc\Documents\学习\Java\images\sort-bubble.gif)

##### 代码实现

```java
int[] arr = {15, 3, 26, 46, 27, 2, 36, 4, 19, 38, 5, 50, 48, 44, 47};
int temp;
for (int i = 0; i < arr.length; i++) {
    for (int j = 0; j < arr.length - i - 1; j++) {
        if (arr[j] > arr[j + 1]) {
            temp = arr[j + 1];
            arr[j + 1] = arr[j];
            arr[j] = temp;
        }
    }
}
System.out.println(Arrays.toString(arr));
```

#### 快速排序

##### 算法描述

1. 从数组中选择一个"基准"(pivot)
2. 排序数组, 比基准小的在前, 基准大的在后
3. 递归把小于基准值的分区和大于基准值的分区排序

##### 动图演示

![](C:\Users\wolfc\Documents\学习\Java\images\sort-quick.gif)

##### 代码实现

```java
public static void main(String[] args) {
    int[] arr = {15, 3, 26, 46, 27, 2, 36, 4, 19, 38, 5, 50, 48, 44, 47};
    sortSub(arr, 0, arr.length);
    System.out.println(Arrays.toString(arr));
}

public static void sortSub(int[] arr, int left, int right) {
    int pivotVal = arr[left];
    int index = left + 1;
    int temp;
    for (int i = index; i < right; i++) {
        if (arr[i] < pivotVal) {
            temp = arr[i];
            arr[i] = arr[index];
            arr[index] = temp;
            index++;
        }
    }
    temp = arr[left];
    arr[left] = arr[index - 1];
    arr[index - 1] = temp;
    if (index - 1 > left + 1) {
        sortSub(arr, left, index - 1);
    }
    if (right - 1 > index) {
        sortSub(arr, index, right);
    }
}
```

#### 插入排序

#### 希尔排序

#### 选择排序

#### 堆排序

#### 二路归并排序

#### 多路归并排序

#### 计数排序

#### 桶排序

#### 基数排序

### Hash算法

## 数据库

### MySQL

#### 引擎

##### MyISAM

* 不支持事务
* 不支持外键

##### InnoDB

* 支持事务
* 支持外键

##### Memory

* 内存引擎

##### Merge

* 一组MyISAM表的组合

##### Archive

[TODO]

##### Federate

[TODO]

##### CSV

[TODO]

##### BLACKHOLE

[TODO]

#### 事务

##### 事务的四大特性(ACID)

* 原子性(Atomic)
* 一致性(Consistent)
* 隔离性(Isolated)
* 持久性(Durable)

##### 事务产生的问题

* 脏读
  * 读取未提交的数据
* 不可重复读
  * 同一事务多次读取, 结果不一致, 读取到已提交的数据
* 幻读
  * 同一查询条件, 读取结果不一致, 读取到其它事务新插入的数据

##### 事务隔离级别

* RU (Read Uncommit) 读未提交
* RC (Read Commit) 读已提交
* RR (Repeatable Read) 可重复读
* S (Serializable) 串行

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
| :------: | :--: | :--------: | :--: |
|    RU    |  Y   |     Y      |  Y   |
|    RC    |  N   |     Y      |  Y   |
|    RR    |  N   |     N      |  Y   |
|    S     |  N   |     N      |  N   |

##### 事务锁

* 共享锁(S): 允许事务去读一行, 阻止其它事务获得相同数据的排他锁
* 排他锁(X): 允许获得排他锁的数据更新数据, 阻止其它事务获得相同数据集的共享读锁和排他写锁
* 意向共享锁(IS): 事务打算给数据行加行共享锁, 事务在给一个数据行加共享锁前必须先取得该表的IS锁
* 意向排他锁(IX): 事务打算给数据行加行排他锁, 事务在给一个数据行加排他锁前必须先取得该表的IX锁

##### InnoDB中的锁

* record lock

  单条索引记录上加锁, record lock锁住的永远是索引, 而非记录本, 当一条sql没有走任何索引时，那么将会在每一条聚集索引后面加X锁, 这个类似于表锁, 但原理上和表锁应该是完全不同的

* gap lock

  在索引记录之间的间隙中加锁, 或者是在某一条索引记录之前或者之后加锁, 并不包括该索引记录本身.

  gap lock的机制主要是解决可重复读模式下的幻读问题

* next-key locks

  record lock和gap lock的结合, 除了锁住记录本身, 还要再锁住索引之间的间隙

[TODO]

##### MVCC

MVCC (Multiversion Concurrency Control), 即多版本并发控制技术,它使得大部分支持行锁的事务引擎, 不再单纯的使用行锁来进行数据库的并发控制, 取而代之的是,把数据库的行锁与行的多个版本结合起来, 只需要很小的开销,就可以实现非锁定读, 从而大大提高数据库系统的并发性能.

MVCC只在READ COMMITED 和 REPEATABLE READ 两个隔离级别下工作. READ UNCOMMITTED总是读取最新的数据行, 而不是符合当前事务版本的数据行. 而SERIALIZABLE 则会对所有读取的行都加锁

**实现原理**

InnoDB为每行数据增加3个隐藏列用于实现MVCC

| 列名        | 长度 | 作用                                                         |
| ----------- | ---- | ------------------------------------------------------------ |
| DB_TRX_ID   | 6    | 插入或更新行的最后一个事务ID(删除视为更新, 将其标记为已删除) |
| DB_ROLL_PTR | 7    | 写入回滚段的撤销日志记录(若行已更新, 则撤消日志记录包含在更新行之前重建行内容所需的信息) |
| DB_ROW_ID   | 6    | 行标识(隐藏单调自增id)                                       |

* Select
  * InnoDB只查找版本(DB_TRX_ID)早于当前事务版本号的数据行(行的版本号<=事务的系统版本号, 这样可以确保数据数据行要么是在开始之前已经存在了, 要么是事务自身插入或修改过的)
  * 行的删除版本号(DB_ROLL_PTR)要么未定义(未更新过), 要么大于当前事务版本号(在当前事务开始之后更新). 这样可以确保读取到的行, 在事务开始之前未被删除
* Insert
  * InnoDB为新插入的行保存当前系统版本号作为行版本号
* Update
  * InnoDB复制了一行, 新行的版本号使用系统版本号, 也把系统版本号作为删除行的版本号
* Delete
  * InnoDB为删除的每一行保存当前的系统版本号作为行删除标识符

#### 索引

##### 索引类型

* 主键索引
* 普通索引
* 唯一索引
* 组合索引
* 全文索引

##### 索引数据结构

* 哈希索引
* BTree索引

##### 聚簇索引/非聚簇索引

* 聚簇索引
  * 索引和数据存放在一起
  * 默认是主键, 没有主键会选择一个唯一非空索引, 如果没有InnoDB会隐式创建一个主键作为聚簇索引
* 非聚簇索引
  * 索引的叶子节点存放的数据是实际数据的地址, 查询完整数据需要根据地址查询

##### 最左匹配原则

* 组合索引最左匹配原则
* like左匹配

##### 覆盖索引

如果一个索引包含所有需要查询的字段, 称为覆盖索引.

覆盖索引的好处:

* 只查询索引, 不需要回表, 减少IO, 提升性能
* 分页优化

覆盖索引的限制:

* 覆盖索引必须包含查询的值, 因此索引中必须存储列值, 因此只有BTree索引能使用
* 不是所有存储引擎都实现了覆盖索引
* SELECT字段只能查询索引的字段

**怎么看是否使用了覆盖索引**

![](C:\Users\wolfc\Documents\学习\Java\images\mysql_explain_extra_index.png)

使用explain命令查询执行计划, 在extra中发现`Using index`表示使用了覆盖索引.

#### 集群

##### 主从复制原理

[TODO]

##### MySQL-Proxy

#### MySQL优化

##### SQL调优

[TODO]

##### 分页优化

```mysql
select * from table limit [offset], [rows]
```

一句很简单的SQL, 但是当offset一大后,  limit的性能就会急剧下降, 为什么?

MySQL分页的逻辑是:

1. 从数据表中读取第N条添加到数据集中
2. 重复第一步直到N=offset+row
3. 抛弃前offset条数据
4. 返回剩余的数据

很显然, 但offset较大时, 第2步会不断重复执行, 导致性能急剧下降.

**使用id优化**

```mysql
select * from table where id > 100000 limit 1
```

这种方法限制条件较高, 必须直到offset的位置, 适合情况有限

1. 查询前计算出offset, 必须要求id是连续不间断的, 而且, 不能有where条件干扰
2. 从第一次分页开始, 记录上一次分页的最后一个id, 传入下一次分页查询, 这意味着分页条件不能改变, 且不能跳页

**使用覆盖索引优化**

```mysql
select * from table t1 where id >= (select id from table limit 100000, 1) limit 10
```

```mysql
select * from table t1 join (select id from table limit 100000, 10) t2 on t1.id=t2.id
```

这两句SQL都是在子查询中使用覆盖索引, 直接在索引中查询分页后的id, 再回表查询少量的数据, 从而提高查询性能.

##### 分区/分库/分表

[TODO]

## 非关系型数据库

### Redis

#### 基础

##### Redis数据类型

* string 字符串
* list 有序, 可重复列表
* set 无序, 不可重复集合
* zset 有序, 不可重复集合
* hash map
* stream(redis5+)

##### Redis过期清除策略

* 惰性删除
  * 对redis的key进行读写操作时, 判断是否已过期, 过期则清除, 返回空
* 定期清除
  * redis周期性服务器检查时会清除过期的key
  * 随机选取一批key, 发现并清除过期的key, 如果删除的key超过25%, 重新随机选取删除
  * 每次清除有一个最大时长限制, 超过也停止清除, 计算公式: `1000000*ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC/server.hz/100`
* 内存不足
  * 当已用内存超过`maxmemory`时, 触发主动清除策略

##### Redis内存淘汰策略

内存淘汰是指redis内存已满的时候, 自动淘汰数据, 释放内存空间

* noeviction: 不删除, 达到最大内存时, 直接返回错误
* allkeys-lru: 所有key中选取最少使用(less recently used ,LRU)的
* volatile-lru: 从设置了过期时间中选取最少使用的
* allkeys-random: 所有key中随机选取一部分删除
* volatile-random: 从设置了过期时间中随机选择一部分删除
* volatile-ttl: 从设置了过期时间中优先选择剩余时间少(time to live,TTL)的

**Redis的近似LRU算法**:

* 抽取少量key样本, 选择访问时间最古老的key删除

##### Redis如何使用多路复用IO的

[TODO]

#### Redis集群

##### Redis主从

[TODO]

##### Redis-Sentinel

[TODO]

##### Redis-Cluster

![](C:\Users\wolfc\Documents\学习\Java\images\redis-cluster.png)

Redis Cluster是Redis官方推出的分布式Redis集群方案, 整个Redis由多个Redis节点组成, 区分于master/slave模式, 这里的每个Redis集群都是master节点, 每个节点地位平等, 没有中心节点, Redis Cluster将数据分片, 每个节点存储一部分数据

**数据分片规则**

Redis Cluster采用哈希槽的数据分片方法, 将总计`16384`个槽(slot)分配到集群各个节点中, 每个节点负责一部分哈希槽, 比如3个节点的集群:

* 节点A: 包含 0 到 5500号哈希槽
* 节点B: 节点 B 包含5501 到 11000 号哈希槽
* 节点C: 节点 C 包含11001 到 16384号哈希槽

**客户端路由**

Redis Cluster会将每个key通过CRC16校验后对16384取模来决定放置哪个槽, 当Client访问的key不在当前节点时, Redis Cluster会返回moved命令, 并告知正确的节点, Redis接收到moved命令后, 重新发送请求到对应的节点, 为了提高性能, 客户端应该缓存槽与节点的对应关系, 即槽位路由表. 这样下次指令执行时, 就可以直接请求对应的节点.

**数据分片迁移与ask**

>CLUSTER ADDSLOTS slot1 [slot2] … [slotN] – 指派槽位到节点
>CLUSTER DELSLOTS slot1 [slot2] … [slotN] – 从节点移除槽位
>CLUSTER SETSLOT slot NODE node – 设置槽位到某节点
>CLUSTER SETSLOT slot MIGRATING node – 将槽 slot 迁移出当前节点，移入 node 节点
>CLUSTER SETSLOT slot IMPORTING node – 接受从 node 节点迁移出的槽位 slot

Redis Cluster集群添加, 删除节点, 因数据分布不均衡时都会导致哈希槽重新分配, 数据迁移分为三个步骤：

1. 向目标节点发送状态变更命令，将目标节点的对应哈希槽状态置为importing
2. 向源节点发送状态变更命令，将源节点对应的哈希槽状态置为migrating
3. 针对源节点上的哈希槽的所有key, 向源节点发送migrate命令, 告知源节点将对应的key迁移到目标节点, 当源节点的状态置为migrating后, 此时源节点提供的服务和通常状态下有所区别
   1. 如果Client访问的key尚未迁出, 则正常的处理该key
   2. 如果key已经迁出或者key不存在, 则回复Client ASK, 信息让其跳转到目标节点处理
4. 当目标节点状态变成importing后,表示对应的slot正在向目标节点迁入。目标节点和通常情况下有所区别
   1. 对于该slot上所有非ask跳转的操作, 目标节点不会进行操作, 而是通过moved让Client跳转至源节点执行
   2. 迁移过程中, 新增加的key会在目标节点执行, 源节点不会新增key, 使得迁移可以在某个确定的时刻结束

单个key的迁移过程可以通过原子化的migrate命令完成, 对于源节点和目标节点的从节点, 是通过主备复制, 从而达到增删数据.

##### Codis

Codis是由豌豆荚团队开发的一个Redis集群方案

[TODO]

#### 问题处理

##### 缓存穿透

**发生场景:** 此时要查询的数据不存在, 缓存无法命中所以需要查询完数据库, 但是数据是不存在的, 此时数据库肯定会返回空,也就无法将该数据写入到缓存中,那么每次对该数据的查询都会去查询一次数据库.

**解决方案:**

* 布隆过滤: 预先缓存所有的key到一个大的`map`中, 然后在过滤器中过滤掉不存在的key, 需要考虑数据库的key更新, 此时需要考虑`map`的更新频率
* 缓存空值: 缓存不存在的数据到缓存中, 设置较短过期时间

##### 缓存雪崩

**发生场景:**  当Redis服务器重启或者大量缓存在同一时期失效时, 此时大量的流量会全部冲击到数据库上面, 数据库有可能会因为承受不住而宕机.

**解决方案:** 

* 均匀分布: 设置均匀分布的过期时间
* 熔断机制: 类似SpringCloud的熔断器, 达到预定阙值时, 直接返回
* 隔离机制: 
* 限流机制:
* 多级缓存机制: 

### Memcached

### MongoDB

## 消息

### 基础

### RabbitMQ

#### AMQP协议

##### AMQP是什么

AMQP（Advanced Message Queuing Protocol，高级消息队列协议）是一个进程间传递**异步消息**的**网络协议**。

##### AMQP中的基本概念

- Broker: MQ Server
- Virtual host: 虚拟主机, 多用户使用同一个MQ Server时, 可以划分多个vhost, 每个vhost创建exchange/queue
- Connection: publisher/consumer和broker之间的tcp连接
- Channel: 信道, 用于减少Connection连接
- Exchange: 交换机, 常用类型包括direct, topic, fanout, header
- Queue: 消息队列
- Binding: exchange和queue之间的关联

##### RabbitMQ实现AMQP

[TODO]

#### 确认机制

##### 生产者AMQP事务机制

[TODO]

##### 生产者confirm机制

[TODO]

##### 消费者ack机制

[TODO]

#### 持久化

##### 消息什么情况下会持久化

- 消息publish时要求持久化
- 内存不足, 需要将内存中的消息转移到磁盘

##### 消息什么时候会刷新到磁盘

- 写入文件前会有一个Buffer, 大小为1M, 如果Buffer已满, 则将写入文件
- 固定刷新, 频率25ms

#### 使用RabbitMQ遇到的问题

##### 消息丢失

- 生产者消息丢失
  - 事务机制, 影响性能
  - confirm机制
- RabbitMQ丢失消息
  - 开启持久化
  - 持久化前前MQ挂了, 消息可能丢失, 需要配合生产者消息确认机制, 持久化后才发送`ack`
- 消费者丢失消息
  - 开启消费者ack机制

##### 重复消费

​	消息重复消费, 一般情况下是consumer消费了消息, 但因网络问题, 未发送`ack`给RabbitMQ, 此时, MQ会将消息发送给同队列其它消费者处理, 导致消息重复消费.

​	解决办法可以在收到消息后查询数据库有没有被消费

## 分布式

### 分布式锁

#### 概念

[TODO]

#### 实现

##### ZK实现

[TODO]

##### Redis实现

[TODO]

#### 问题

[TODO]

### 分布式事务

## 软件开发

[TODO ]

## 人事问题

#### 前公司相关

##### 哪个公司给你的印象比较深刻

[TODO]

##### 离职原因

[TODO]

##### 有什么没做好的地方

[TODO]

#### 职业岗位

##### 高级Java的岗位职责

一般来说, 专业公司的程序员从高级岗位开始分为技术和管理两条路线.

**管理岗位:**

* 分配开发人员任务
* 跟踪项目进度

**技术岗位:**

* 参与或负责项目框架设计
* 项目技术难点把关攻克
* 核心模块代码开发
* 公司技术预研
* 公司技术培训, 积累

##### 架构师的岗位职责

[TODO]

##### 近期规划

* Java
  * JVM底层
  * JVM调优
* 框架
  * Spring源码
  * Spring Boot源码
  * Spring WebFlux
* Docker, K8S
* 分布式
  * Dubbo
  * Spring Cloud
  * Spring Cloud Ailibaba
* 数据库
  * MySQL
  * Redis
* 数据结构
* 算法
* 其他
  * Python

##### 3年规划

[TODO]

##### 5年规划

[TODO]

#### 学习

##### 学习目的

[TODO]

##### 学习途径

* 开源社区, github, gitee
* 官网
* 书籍
* 博客
* 视频

##### 在读什么书

[TODO]

#### 性格

##### 缺点

[TODO]

##### 三个词形容自己

乐观, 真诚, 内敛


<!-- GFM-TOC -->
* [一、线程安全性](#线程安全性)
    * [1 什么是线程安全](#什么是线程安全)
    * [2 原子性](#原子性)
    * [3 加锁机制](#加锁机制)
* [二、对象的共享](#对象的共享)
	* [1 可见性](#可见性)
	* [2 发布与逸出](#发布与逸出)
	* [3 线程封闭](#线程封闭)
    * [4 安全发布](#安全发布)
* [三、对象的组合](#对象的组合)
    * [1 设计线程安全的类](#设计线程安全的类)
    * [2 实例封闭](#实例封闭)
    * [3 线程安全性的委托](#线程安全性的委托)

----------


# 线程安全性

## 什么是线程安全
线程安全性：当多个线程访问某个类时，这个类始终都能表现出正常的行为，那么就称这个类是线程安全的。（无状态的对象一定是线程安全的）

## 原子性
原子性指的是整个程序中的所有操作，要么全部完成，要么全部不完成，不可能停滞在中间的某个环节
```java
++count;  // 非原子操作
```
实际上，++count包含了3个独立的操作：读取count的值，将值+1，然后将计算结果写入count，这是一个“读取——修改——写入”的操作序列，并且其结果状态依赖于之前的状态。

要保持状态的一致性，就需要在单个原子操作中更新所有相关的状态变量。

## 竞态条件
由于不恰当的执行时序而出现的不正确的结果叫作**竞态条件**

## 加锁机制
### 内置锁
Java提供了一种内置的锁机制来支持原子性：同步代码块(Synchronized Block)，其包含两个部分
- 一个作为锁的对象引用
- 一个作为由这个锁保护的代码块，其中该同步代码块的锁就是方法调用所在的对象。
静态的synchronize方法以Class对象作为锁。
``` java
synchronized (lock){
// 访问或修改锁保护的共享状态
}
```
每一个Java对象都可以用作一个实现同步的锁，这些锁被称为**内置锁(Intrinsic Lock)**或**监视器锁(Monitor Locl)**。
线程在进入同步代码块之前会自动获得锁，并且在退出同步代码块时会自动释放锁，而无论是通过正常的路径退出，还是通过从代码块中抛出异常退出。

**获得内置锁的唯一途径** 就是进入由这个锁保护的同步代码块或者方法。
Java的内置锁相当于一种**互斥体(或互斥锁)** ，这意味着同时最多只有一个线程能持有这种锁。当线程A尝试获得由线程B持有的锁时，线程A必须等待或者阻塞，等到B释放锁之后才有可能获得这个锁，如果B永远不释放锁，则A永远等待下去。

### 重入
当某个线程请求一个由其他线程持有的锁时，发出请求的线程就会阻塞。然而，由于内置锁是可以**重入**的，因此某个线程试图获得一个已由它自己持有的锁，那么这个请求就会成功。

重入进一步提升了加锁行为的封装性，因此简化了面向对象并发代码的开发
```java
public class Widget{
    public synchronized void doSomething(){
        ...
    }
}
public class LoggingWidget extends Widget{
    public synchronized void doSomething(){
        System.out.println(toString() + ": calling doSomething");
        super.doSomething();
    }
}
```
上述程序清单中，子类重写了父类的synchronized方法，然后调用父类中的方法，由于Widget和loggingWidget中的doSomething方法在执行前都会获得Widget上的锁，此时如果没有可重入的锁，子类调用父类的doSomething方法时，将永远无法获得Widget上的锁，因此这段代码将产生死锁。

### 用锁来保护状态
如果在复合操作的执行过程中持有一个锁，则会使复合操作变成原子操作。对于可能被多个线程同时访问的可变状态变量，在访问它们的时候都需要持有一个锁，我们称这个状态变量是由这个锁保护的。

一种常见的加锁约定是，把所有可变的状态都封装到对象的内部。

如果只是将每个方法作为同步方法，例如Vector，那么并不足以确保Vector上的复合操作都是原子的，例如：
```java
if(!vector.contains(element))
	vector.add(element);
```
虽然contains和add 方法都是原子的，假设contains方法由A线程占有，add方法由B线程占有，但是在执行完if判断条件之后，被其它线程C抢先执行了一次add方法，之后线程B再执行add方法，也就是if之后执行了2次add，显然与目标程序设计不符。

虽然synchronized方法可以确保单个操作的原子性，但如果要把多个操作合并为一个复合操作，还需要额外的加锁机制(了解如何在线程安全对象中添加原子操作的方法)，否则仍然会产生**竞态条件**。

----------

# 对象的共享

## 可见性
为了确保多个线程之间对内存写入操作的可见性，必须使用同步机制。

### 加锁与可见性
内置锁可以用于确保某个线程以一种可预测的方式来查看另一个线程的执行结果。
<div align="center"> <img src="../pics//1545703873(1).png" width=""/> </div><br>
访问某个共享且可变的变量时，要求所有的线程都在同一个锁上同步，就是为了确保某个线程写入该变量的值对于其它线程来说是可见的。否则，如果一个线程在未持有正确的锁的情况下读取某个变量，则读取到的值可能是实效值。

加锁的含义不仅仅局限于互斥行为，还包括内存可见性。为了确保所有的线程都能看到共享变量的最新值，所有执行读操作或者写操作的线程都必须在同一个锁上同步。

### Volatile变量
Java语言提供了一种稍弱的同步机制，即volatile变量，用来确保将变量的更新操作通知其它线程。当把一个变量声明为volatile类型后，编译器与运行时都会注意到这个变量是共享的，因此不会将该变量上的操作与其它内存操作一起重排序。volatile不会被缓存在寄存器或者对其它处理器不可见的地方，因此在读取volatile类型的变量时，总会返回最新写入的值。

volatile变量是一种比synchronized关键字更轻量级的同步机制。

volatile变量的一种经典用法：检查某个状态变量以标记是否退出循环。
```java
volatile boolean asleep;
...
while(!asleep)
	countSomeSheep();
```
为了使这个实例能正确执行，asleep必须设置为volatile类型，否则其它线程修改了asleep后，执行判断的线程缺发现不了。


**局限性：** volatile变量通常用作某个操作完成、发生或中断的状态标志。volatile的语义不足以确保递增操作(count++)的原子性。

**当且仅当** 满足以下条件时，才应该使用volatile关键字：
- 对变量的写入操作不依赖于变量的当前值，或者你能确保只有单个线程对变量进行更新。
- 该变量不会与其它状态变量一起纳入[不变性条件](#不变性)中。
- 在访问变量时不需要加锁。

## 发布与逸出
**发布(Publish)** 一个对象的意思是指，使对象能够在当前作用域之外的代码中使用。当某个不应该发布的对象被发布时，这种情况叫作 **逸出(Escape)**。

发布对象对简单的方法是将对象的引用保存到一个公有的静态变量中。
```java
public static Set<secret> knowScrets;
public void initialize(){
	knowScrets = new HashSet<>();
}
```
当发布一个对象时，该对象的非私有域中引用的所有对象同样会被发布。

发布对象或者其内部状态的机制就是发布一个内部的类实例，例如：
```java
public class ThisEscape{
	public ThisEscape(EventSource source){
		source.registerListener(new EventListener(){
			public void onEvent(Event e){
				doSomething(e);
			}
		})
	}
}
```
ThisEscape发布EventListener时，也隐含的发布了ThisEscape本身，因为在这个内部类实例中包含了对ThisEscape实例的隐含引用。但是不推荐这么做。

## 线程封闭
访问共享的可变数据时，通常需要使用同步。一种避免使用同步的方式就是不共享数据。如果仅在单线程内访问数据，就不需要同步。这种技术称为 **线程封闭(Thread Confinement)**，它是实现线程安全性的最简单方式之一。常见的应用有JDBC(Java Database Connectivity)的Connection对象。

### Ad-hoc线程封闭
Ad-hoc线程封闭是指，维护线程封闭性的职责完全由程序来实现。由于Ad-hoc线程封闭技术的脆弱性，因此在程序中要尽量少使用。

### 栈封闭
栈封闭是线程封闭的一种特例，在栈封闭中，只能通过局部变量才能访问到对象。局部变量(在JVM虚拟机栈中，线程私有)的固有属性之一就是封闭在执行线程中。

### ThreadLocal类
ThreadLocal类能使线程中的某个值与保存值的对象关联起来。ThreadLocal提供了get和set等访问接口或方法这些方法为每个使用该变量的线程都存有一份独立的副本，因此get总是返回由当前线程执行在调用set时设置的最新值。

ThreadLocal对象通常用于防止对可变的单实例变量(Singleton)或全局变量进行共享。

由于JDBC连接对象不一定是线程安全的，通过将JDBC的连接保存到ThreadLocal对象中，每个线程都会有自己的连接。
```java
private static ThreadLocal<Connection> connectionHolder = new ThreadLocal<Connection>(){
	public Connection initialValue(){
		return DriverManager.getConnection(DB_URL);
	}
};
public static Connection getConnection(){
	return connectionHolder.get();
}
```
当某个频繁执行的操作需要一个临时对象，例如一个缓冲区，而同时又希望避免在每次执行时都重新分配该临时对象，就可以使用该技术。

## 不变性
如果某个对象在创建后，其状态就不能被修改，那么这个对象就称为**不可变对象**，不可变对象一定是线程安全的。
当满足以下条件时，对象才是不可变的
- 对象创建后其状态就不能修改
- 对象的所有域都是final类型
- 对象是正确创建(在创建对象期间，this引用没有逸出)

在不可变对象的内部，仍可以用可变对象来管理其状态。
```java
public final class ThreeStooges{
	private final Set<String> stooges = new HashSet<>();

	public ThreeStooges(){
		stooges.add("Moe");
		stooges.add("A");
		stooges.add("B");
	}

	public boolean isStooge(String name){
		return stooges.contains(name);
	}
}
```
尽管保存姓名的set对象可变，但是从ThreeStooges的设计中可以看到，在set对象构造完成之后无法改变。

### final域
关键字final用于构造不可变性对象，final类型的域是不能修改的。但是如果final域引用的对象是可变的，那么这些被引用的对象是可以修改的。
```java
public finalTarget{
    final String[] a;
    finalTarget(){
        a = {"a","b","c"};  // 或者其它的字符串
    }
}
```
final域能确保初始化过程的安全性，从而可以不受限制的访问不可变对象，也不需要在共享时同步。

## 安全发布

### 不正确的发布
确保要发布的对象对其他线程可见(因此要使用同步)。未正确发布对象示例：
```java
public class Holder{

    private int a;
    Holder(int a){
        this.a = a;
    }
    public void assertSanity(){
        if (a != a) {
            throw new AssertionError("This statement is false.");
        }
    }
}
```
由于没有使用同步来确保Holder对象对其它线程可见，因此将Holder称为“未被正确发布”。在未被正确发布的对象中存在2个问题：
- 除了发布对象的线程外，其它线程可能看到Holder域是一个失效值，例如默认被JVM初始化为a=0，或者是个空的引用。
- 线程看到Holder引用的值是最新的，但是Holder的状态却是失效的。
某个线程在第一次读取域的是一个失效值，可能第二次读取就能得到一个更新后的值，因此assertSanity中的判断条件就为真，抛出异常。

**解决办法** 将Holder中所有的域都定义为final类型。

任何线程都可以在不需要额外同步的情况下安全地访问不可变对象。这种保证还将延伸到被正确创建对象中所有的final类型的域。**注意：如果final类型的域所指向的是可变对象，那么在访问这些域时仍然要做同步。**

### 安全发布的常用模式
可变对象必须通过安全的方式进行发布，意味着在发布和适用这些对象时必须进行同步。要安全地发布一个对象，对象的 **引用** 及其 **状态** 必须同时对其它线程可见，一个正确的构造对象可通过以下的方式安全发布：
- 在静态初始化函数中初始化一个对象的引用；
- 将对象的引用保存到volatile类型的域，或者是AtomicReferance对象中；
- 将对象的引用保存到一个正确构造对象的final域中；
- 将对象的引用保存到一个由锁保护的域中；

通常，要发布一个静态构造对象，最简单个最安全的方式是使用静态的初始化器：
```java
public static Holder holder = new Holder(66);
```

### 事实不可变对象
对象从技术上来看是可变的，但其状态在发布后不会再改变，需要安全发布。
例如，Date本身是可变的，但是如果将它作为不可变对象来使用，那么在多线程之间共享Date的时候就可以省去加锁。假设需要维护一个Map对象，其中保存了每位用户最近的登录时间：
```java
public Map<String, Date> lastLoggin = Collections.synchronizedMap(new HashMap<>());
```
### 可变对象
如果对象在构造后可以修改，那么安全发布只能确保在“发布当时”状态的可见性。对于可变对象，不仅在发布的时候需要同步，且每次访问或者修改的时候都要进行同步，以确保后续操作的可见性。要安全地共享可变对象，这些对象就必须安全地发布，并且必须是 **线程安全** 或者由 **锁** 保护起来。

对象的发布需求取决于它的可变性：
- 不可变对象可以通过任意机制来发布；
- 事实不可变对象必须通过安全方式来发布；
- 可变对象必须通过安全方式来发布，且必须是线程安全的或者由某个锁保护起来；

### 安全地共享对象
在并发程序中使用和共享对象时，可以使用一些实用的策略：
- **线程封闭：** 线程封闭的对象只能由一个线程拥有，对象被封闭在线程中，且只能由这个线程修改。
- **只读共享：** 在没有额外的同步下，共享的只读对象可以供多个线程并发访问，但任何线程都不可以修改它。共享只读对象包括：不可变对象、事实不可变对象。
- **线程安全共享：** 线程安全的对象在其内部实现同步，因此多个线程可以通过对象的共有接口来进行访问，而不需要进一步的同步。
- **保护对象：** 被保护的对象只能通过持有特定的锁来访问。保护对象包括封装在其他线程安全对象中的对象，以及已发布的并且由某个特定的锁保护的对象。

----------

# 对象的组合
在开发过程中，我们并不希望每一次内存访问都进行分析以确保程序是线程安全的，而是希望将一些现有的线程安全组件组合为更大规模的程序或组件。

## 设计线程安全的类

通过使用 **封装技术** ，可以使得在不对整个程序进行分析的情况下，就可以判断一个类是否为线程是安全的。
在设计线程安全类的过程中，需要包含以下三个基本要素：
- 找出构成对象状态的所有变量；
- 找出约束状态变量的不变性条件；
- 建立对象状态的并发访问管理策略；

**同步策略(Synchronization Policy)** 定义了如何在不违背对象不变性条件或后验条件的情况下对其状态的访问操作进行协同。同步策略规定了如何将不可变性、线程封闭、加锁机制等结合起来以维护线程的安全性，并且还规定了哪些变量由哪些锁来保护。

### 收集同步需求
如果不了解对象的不变性条件和后验条件，那么久不能确保线程的安全性。要满足在状态变量的有效值或者状态转换上的各种约束条件，就需要借助于 **原子性** 和 **封装性** 。

### 依赖状态的操作
如果在某个状态中包含有基于状态的先验条件(Precondition)，那么这个操作就称为**依赖状态的操作**，例如：
```java
public MyStack{

    public void pop(){
        ...
    }
    public boolean isEmpty(){
        ...
    }
    public static void main(String[] args) {
        MyStack stack = new MyStack();
        if (!stack.isEmpty()) {
            stack.pop();
        }
    }
}
```
删除栈中的元素前，应该先判断栈是否为空。

## 实例封闭
封装简化了线程安全类的实现过程，它提供了一种 **实例封闭机制(Instance Confinement)**，通常也称为**封闭**。将封闭机制与合适的加锁机制结合起来，可以确保以线程安全的方式去访问非线程安全的对象。

将数据封装在对象内部，可以将数据的访问限制在对象的方法上，从而更容易确保线程在访问数据时能持有正确的锁。

被封闭对象一定不能超出它们既定的作用域，对象可以封闭在：
- 类的一个实例（例如作为类的一个私有成员）；
- 某个作用域内（例如作为一个局部变量）；
- 线程内

以下说明如何通过封闭与加锁机制使一个类称为线程安全的（即使这个类的状态变量并不是线程安全的）。
```java
public class PersonSet{
    private final Set<Person> mySet = new HashSet<>();
    public synchronized void addPerson(Person p){
        mySet.add(p);
    }
    public synchronized boolean containsPerson(Person p){
        return mySet.contains(p);
    }
}
```
PersonSet的状态由HashSet来管理，而HashSet并不是线程安全的(即使mySet是final类型的，但是它所指向的是可变对象)。但由于mySet是私有的，且不会逸出，因此HashSet被封闭在了PersonSet中。唯一能访问mySet的代码路是addPerson和ContainsPerson，在执行它们时都需要获取PersonSet上面的锁。PersonSet的状态完全由它的内置锁保护，因而PersonSet是一个线程安全的类。需要注意的是，本例假设Person为线程安全的，否则在访问Person时还要做额外的同步操作。

### Java监视器模式
从线程封闭原则及其逻辑推论可以得出Java监视器模式(主要优势在于简单性)。遵循Java监视器模式的对象会把对象所有可变的状态都封装起来，并由自己的内置锁来保护。
```java
public final class Counter{
    private long value = 0;
    public synchronized long getValue(){
        return value;
    }
    public synchronized long increment(){
        if (value == Long.MAX_VALUE) {
            throw new IllegalStateException("Counter overflow");
        }
        return ++value;
    }
}
```
在Counter中封装了变量value，对该变量的所有访问都要经过Counter的方法来执行，且方法都是同步的。

**示例：车辆追踪器：**
一个用于调度车辆的“车辆追踪器”，例如出租车、警车、货车等。首先使用监视器模式来构建车辆追踪器。每辆车都有一个String对象来标识，并且拥有一个相应的位置坐标(x,y)。在VehicleTracker类中封装了车辆的标识和位置。
```java
public class MonitorVehicleTracker{
    private final Map<String,MutablePoint> locations;
    // constructor
    public MonitorVehicleTracker(Map<String,MutablePoint> locations){
        this.locations = deepCopy(locations);
    }
    // 获得所有车的位置
    public synchronized Map<String,MutablePoint> getLocations(){
        return deepCopy(locations);
    }
    // 获得某一辆车的位置
    public synchronized MutablePoint getLocations(String id){
        MutablePoint loc = locations.get(id);
        return loc == null ? null : new MutablePoint(loc);
    }
    // 设置一辆车的位置
    public synchronized void setLocations(String id, int x, int y){
        MutablePoint loc = locations.get(id);
        if (loc == null) {
            throw new IllegalArgumentException("No such id: " + "id" );
        }
        loc.x = x;
        loc.y = y;
    }
    /* unmodifiableMap方法返回的Map, 它本身不允许修改, 就是说其中每一个entry引用不允许修改，但是entry中的value如果是对象，value引用的对象的属性值是可以修改的*/
    private static Map<String,MutablePoint> deepCopy(Map<String,MutablePoint> m){
        Map<String,MutablePoint> result = new HashMap<String,MutablePoint>();
        for (String id : m.keySet()) {
            result.put(id, new MutablePoint(m.get(id)));
        }
        return Collections.unmodifiableMap(result);
    }
}
// MutablePoint不是线程安全类，因此当需要返回车辆的位置时，通过MutablePoint拷贝构造函数或者deepCopy方法来复制正确的值
public class MutablePoint{
    public int x,y;
    public MutablePoint(){
        x = 0;
        y = 0;
    }
    public MutablePoint(MutablePoint p){
        this.x = p.x;
        this.y = p.y;
    }
}
```
这种实现方式是通过返回客户代码之前复制可变的数据来维持线程的安全性。当数据量很大的时候，性能差。

## 线程安全性的委托
**示例：基于委托的车辆追踪器：**
```java
public class DelegatingVechicleTracker{
    private final ConcurrentMap<String,Point> locations;
    private final Map<String,Point> unmodifiableMap;

    public DelegatingVechicleTracker(Map<String,Point> points){
        locations = new ConcurrentHashMap<>();
        unmodifiableMap = new Collections.unmodifiableMap(locations);
    }
    // 获取所有的车辆位置
    public Map<String,Point> getLocations(){
        return unmodifiableMap;
    }
    // 获取指定的车辆位置
    public Point getLocations(String id){
        return unmodifiableMap.get(id);
    }
    // 设置车辆位置
    public void setLocations(String id, int x, int y){
        if (locations.replace(id, new Point(x,y)) == null) {
            throw new IllegalArgumentException("Invalid vehicle name:" + id);
        }
    }
}
// 线程安全的类，可以自由共享和发布，因此不要在返回locations时复制
public class Point{
    public final int x,y;
    public Point(int x, int y){
        this.x = x;
        this.y = y;
    }
}
```

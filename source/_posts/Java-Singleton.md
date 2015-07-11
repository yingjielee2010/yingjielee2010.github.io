title: Java Singleton
date: 2015-01-19 22:15:39
categories:
- Java
tags: Java
---
![Java Singleton](http://7xkb27.com1.z0.glb.clouddn.com/singleton.png)
# Singleton 单例模式
单例模式是指只允许存在一个实例的类，通常的做法会将类的构造器设为私有，这样可以阻止在类外部对它进行实例化。由于无法在类外部进行实例化，单例类会提供一个公有的 static 方法来获取这个类的唯一实例。
单例模式常见用在日志、驱动、缓存和线程池等中，Java 类库中部分类也使用了单例模式，如 `java.lang.Runtime`。
# 简单实用的写法
根据单例类的定义，单例类需要满足以下两个要求：
1. 构造器私有
2. 提供公有静态方法来获取类唯一的实例

一个简单粗暴的实现方法如下：
```java
public class Singleton {
	private static final Singleton INSTANCE = new Singleton();
	private Singleton() {}

	public static Singleton getInstance() {
		return INSTANCE;
	}
}
```
这种方法直接使用 static final 初始化类一个的实例，可以保证每次通过公有静态方法获取到的都是同一个唯一的类实例，且在类外部无法再对类再次进行实例化，很简单，但也很实用，对于那些程序运行前就必須做一些初始化工作的单例类，是最简单快捷的实现。
## 在 Static 区域中初始化
上面的实现中，如果单例类在实例化的时候抛出异常了，将没有机会对异常进行处理，因此可以改进成另一种常见的实现：
```java
public class Singleton {
	private static final Singleton INSTANCE;
	static{
		try{
			INSTANCE = new Singleton();
		}catch(Exception e){
			//处理异常情况
		}
	}

	private Singleton() {}
	public static Singleton getInstance() {
		return INSTANCE;
	}
}
```

# 单例类延迟(懒)加载
上面提到的单例类的实现方法有一个缺点，就是在类加载的时候类的实例就被初始化了，即使没有调用单例类的静态方法 getInstance()，类的唯一实例也随着类加载而被初始化了。这可能会引起一些问题，导致类实例初始化的时机将不再受程序所控制，而是取决于类加载的时机。如果程序中有很多个单例类或者单例类在初始化的时候很耗时且占用很大的内存，并且在加载程序时引用到了这些单例类，那么启动程序的时候将非常缓慢且占用过多不必要的内存。更加常见的需求是，可能在初始化单例的时候需要等待某些资源，比如资源文件、数据库连接等等，这种情况下需要在这些资源准备完成之后，再调用 getInstance() 来初始化类的实例。
因此对于需要控制初始化实例时机的单例类，最好的做法就是在调用单例类的 getInstance() 方法时再实例化类的唯一实例，这样的实现即是单例类延迟加载，也称懒加载(lazy initialization)。

# 懒加载错误的实现方式
实现懒加载需要做的是在调用单例类 getInstance() 静态方法时才会去初始化单例类的实例，如果已经实例化过了，就直接返回。看上去很简单，最容易想到的是下面这种做法：
```java
public class Singleton {
	private static Singleton INSTANCE = null;
	private Singleton() {}

	public static Singleton getInstance() {
		if (INSTANCE == null) {
			INSTANCE = new Singleton();
		}
		return INSTANCE;
	}
}
```
以上实现看上去没有问题，调用 getInstance() 时加上个判断，类的唯一实例 INSTANCE 为 null 则实例化一个，否则直接返回。问题是上面的这种写法忽略了多线程调用时的情况，如果有多个线程同时调用 getInstance()，这多个线程都同时发现 INSTANCE 为 null ，于是都执行了一遍 `INSTANCE = new Singleton();`，最终导致该单例类被实例化了多次。
### 糟糕的解决方案
上面的实现方式没有考虑多线程调用，于是很容易一拍脑袋就写出下面的做法：
```java
public static synchronized Singleton getInstance() {
	if (INSTANCE == null) {
		INSTANCE = new Singleton();
	}
	return INSTANCE;
}
```
加个 synchronized 阻止多线程同时调用这下解决了吧。没那么简单！这种方法之所以糟糕，在于其简单粗暴的头痛医头脚痛医脚的直接将 getInstance() 方法限制成线程独占执行，然而对于 getInstance() 方法，只有在第一次实例化的时候需要保证没有多个线程在同时执行初始化操作，在实例化完之后，这个独占锁就成了累赘，获得锁和去锁的过程不仅大大降低了获取实例方法的性能，还同时阻止了多个线程同时的调用。
# 懒加载正确的多线程解决方案
下面这种实现方式正确的使用多线程思路来解决以上懒加载出现的问题，既可以保证类实例的唯一性，又不影响多线程的调用：
```java
public class Singleton {
	private volatile static Singleton INSTANCE = null;
	private Singleton() {}

	public static Singleton getSingleton() {
		if (INSTANCE == null) { 		// Single Checked
			synchronized (Singleton.class) {
				if (INSTANCE == null) { // Double Checked
					INSTANCE = new Singleton();
				}
			}
		}
		return INSTANCE;
	}
}
```
在上述的实现中，当调用 getSingleton() 时首先检查是否已实例化过，如果已经实例化了就直接返回唯一实例，这一过程并未加锁，这么做可以保证在实例化之后多线程调用时不存在互斥访问导致访问性能低下的问题。当发现类还没有实例化之后，需要对单例类唯一的实例进行初始化，这个时候就有必要对这一过程加上锁了，这样可以防止多个线程进行实例化。同时最为关键的是，进行上锁代码之后，再一次对 INSTANCE 变量进行了一次判断，如果不为空才初始化。这一步很重要，因为如果不加上这个判断，则当两个或多个线程执行到 synchronized 上锁的代码时，第一个线程获得了锁进去做了实例化操作，而其它线程则被阻塞住了，当第一个线程实例化结束归还锁时，其它阻塞的线程获得锁，再次进入 synchronized 上锁的代码，这个时候如果不判断 INSTANCE 是否为空，则又会对其实例化一次。上面的过程被称为双重检验锁(double checked locking pattern)，代码里进行了两次检查，一次是在同步块外，一次是在同步块内。
然而以上过程仍不能保证类只实例化一次，问题在 `INSTANCE = new Singleton();`这句代码上，由于 JVM 存在指令重排序，有可能 INSTANCE 在还未初始化完时就已经被指向了分配的内存空间，导致其它线程在第一个线程还未初始化结束时进入到了同步代码区内，再次对类进行了一次实例化。
解决方法是将 INSTANCE 声明为 volatile，这样可以告诉 JVM，不要在 INSTANCE 进行初始化时进行重排序，防止 INSTANCE 在还未初始化结束之前就已经为非 null。

# 静态内部类实现懒加载
可以看出以多线程思路来解决单例问题实在是过于复杂，很容易就会因疏忽写错。下面这种做法可以利用静态内部类来实现单例类懒加载：
```java
public class Singleton {
	private static class SingletonHolder {
		private static final Singleton INSTANCE = new Singleton();
	}

	private Singleton() {}
	public static Singleton getSingleton() {
		return SingletonHolder.INSTANCE;
	}
}
```
这种方式利用了 Classloader 的机制来保证线程安全性问题，当调用 getSingleton() 时，Java 类加载器加载类 SingletonHolder，同时实例化单例类的唯一实例，而这一过程是由 JVM 来保证线程独占执行的。

# 使用枚举实现**非懒加载**的 Singleton
《Effective Java》里所推荐的写法，将类改为枚举：
```
public enum Singleton{
    INSTANCE;
}
```
Java中的枚举类型是实例受控的，枚举类型不允许有可访问的构造器，客户端无法创建枚举类的实例，这种写法不仅能避免多线程创建多个实例的问题，而且还能防止反序列化重新创建新的对象。
枚举方式实现的单例类是**不支持懒加载**的，有很多介绍关于枚举单例的文章中都没有提到这一点或者是直接误认为这种方法就是懒加载。实际上这种实现方式和上面提到的第一种区别不大，在类/枚举被类加载器加载的时候会初始化唯一实例，这个时机是不受程序控制的。
用枚举来写单例类从实现方式上看是比较独特的，如果没有见过这种实现方式，看到这种写法的代码可能会造成不必要的困惑。

# 最后
总结一下，对于程序启动初始化时就需要使用的单例类，直接使用非懒加载的实现即可。事实上即使对于那些不需要在程序启动过程中使用的单例类，在绝大部分情况下也是不需要使用懒加载的，虽然单例类实例初始化的时机是在类加载的时候，但在此之前只要在程序代码中没有引用到单例类，类加载器就不会加载这些单例类，也就不会引发实例的初始化(也算是另一种方式的懒加载，不过这个过程是由JVM控制而不是在程序代码中控制的)。
对于需要那些懒加载的单例类，可以考虑以上不同的解决方案，个人比较倾向于静态内部类的实现方式，原因在于多线程解决的方案过于复杂，同时也让以后阅读/维护代码的人不得不小心翼翼的对待这种写法的代码，而对于静态内部类的实现，即使阅读代码的人不理解为什么这么做至少也不会造成太大的困扰。

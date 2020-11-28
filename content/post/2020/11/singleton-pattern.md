---
title: "设计模式之单例模式"
categories: ["设计模式"]
tags: ["设计模式", "单例模式"]
date: 2020-11-28T05:41:57+08:00
draft: false
---

## 定义

单例模式保证了一个类只有一个实例，并且提供了全局唯一的访问方式。

## 核心

单例模式主要是将构造方法私有化，防止其他线程创建新的实例。

## 实现方式

### 饿汉式

饿汉式，就是直接创建好对象，缺点就是浪费内存空间，尤其对于jvm来说，过多的对象会造成频繁的fullgc。实现代码如下：
```java
public class Singleton {
    private static Singleton instance = new Singleton();
    private Singleton() {}
    public static Singleton getInstance() {
        return instance;
    }
}
```
所以我们能不能在只需要对象的时候才创建呢？有饱必有饿，接下来看懒汉式。

### 懒汉式

```java
public class Singleton {
    private static Singleton instance;
    private Singleton() {}
    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

懒汉式是最简单的实现，也符合我们一般思考方式，在Java中多线程是基本要求，所以我们需要实现线程安全的懒汉式写法，最简单实现是在getInstance方法上加synchronized锁，或者是:
```java
synchronized(Singleton.class) {   
    if (instance == null) {
        instance = new Singleton();
    }
}
```
但是这种写法存在很严重的问题，每次去获取对象都需要先获取锁，并发性能非常地差，极端情况下，可能会出现卡顿现象。所以只有在对象不存在的时候才会加锁去创建对象，其实就是下面的DCL双重检查模式。

### 双重检查模式(DCL)
双重检查其实就是在加锁前后进行判断，当对象为空的时候创建对象。需要特别注意的是静态属性要加上volatile保证加锁的时候能够及时写入缓存，保证其他线程拿到的是最新的对象。
```java
public class Singleton {
    private volatile static Singleton instance;
    private Singleton() {}
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized(Singleton.class) {   
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```
双重检查模式即能保证了线程安全又能节约内存。双重检查模式写起来比较复杂，能否简化呢？虽然java语法很繁琐，但是答案还是肯定的。接下来的静态内部类和枚举方式是实现比较优雅的方式。

### 静态内部类

```java

public class Singleton {
  private Singleton(){}
 
  private static class SingletonHoler {
     private static Singleton INSTANCE = new Singleton();
 }
 
  public static Singleton getInstance(){
    return SingletonHoler.INSTANCE;
  }

```

静态内部类的效果和双重检查模式差不多，但是实现上更简单。当Singleton第一次被加载时，并不需要去加载SingletonHoler，只有当getInstance()方法第一次被调用时，才会去初始化INSTANCE,第一次调用getInstance()方法会导致虚拟机加载SingletonHoler类，这种方法不仅能确保线程安全，也能保证单例的唯一性，同时也延迟了单例的实例化。那么静态内部类是如何实现线程安全的呢？我们来看下类加载的方式：

JAVA虚拟机在有且仅有的5种场景下会对类进行初始化。
- 遇到new、getstatic、setstatic或者invokestatic这4个字节码指令时，对应的java代码场景为：new一个关键字或者一个实例化对象时、读取或设置一个静态字段时(final修饰、已在编译期把结果放入常量池的除外)、调用一个类的静态方法时。
- 使用java.lang.reflect包的方法对类进行反射调用的时候，如果类没进行初始化，需要先调用其初始化方法进行初始化。
- 当初始化一个类时，如果其父类还未进行初始化，会先触发其父类的初始化。
- 当虚拟机启动时，用户需要指定一个要执行的主类(包含main()方法的类)，虚拟机会先初始化这个类。
- 当使用JDK 1.7等动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解析结果REF_getStatic、REF_putStatic、REF_invokeStatic的方法句柄，并且这个方法句柄所对应的类没有进行过初始化，则需要先触发其初始化。

这5种情况被称为是类的主动引用，注意，这里《虚拟机规范》中使用的限定词是"有且仅有"，那么，除此之外的所有引用类都不会对类进行初始化，称为被动引用。静态内部类就属于被动引用的行列。那instance的创建过程是如何保证线程安全的呢？在《深入理解JAVA虚拟机》中，有这么一句话:

> 虚拟机会保证一个类的<clinit>()方法在多线程环境中被正确地加锁、同步，如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的<clinit>()方法，其他线程都需要阻塞等待，直到活动线程执行<clinit>()方法完毕。如果在一个类的<clinit>()方法中有耗时很长的操作，就可能造成多个进程阻塞(需要注意的是，其他线程虽然会被阻塞，但如果执行<clinit>()方法后，其他线程唤醒之后不会再次进入<clinit>()方法。同一个加载器下，一个类型只会初始化一次。)，在实际应用中，这种阻塞往往是很隐蔽的。

故而，可以看出INSTANCE在创建过程中是线程安全的，所以说静态内部类形式的单例可保证线程安全，也能保证单例的唯一性，同时也延迟了单例的实例化。

那么，是不是可以说静态内部类单例就是最完美的单例模式了呢？其实不然，静态内部类也有着一个致命的缺点，就是传参的问题，由于是静态内部类的形式去创建单例的，故外部无法传递参数进去，例如Context这种参数，所以，我们创建单例时，可以在静态内部类与DCL模式里自己斟酌。

### 枚举式

枚举式是比较新的写法，在jdk1.5后支持，这种写法简单明了，并且无需关心多线程问题。枚举式还有个特别的优点就是反序列化也不会新创建对象，这点在前面的方式都不能方便的支持。
```java
public enum Singleton {
    INTSTANCE;
    public void anyMethod() {
        
    }
}
```

所以DCL和静态内部类等模式如果防止反序列化创建多个对象呢？答案是继承Serializable接口实现readResolve方法：
```java
public class Singleton implement Serializable {
    private Singleton(){}
 
    private static class SingletonHoler {
        private static Singleton INSTANCE = new Singleton();
    }
    
    public static Singleton getInstance(){
        return SingletonHoler.INSTANCE;
    }
	
	private Object readResolve(){
		return SingletonHoler.INSTANCE;
	}
}
```
为什么实现了readResolve方法就能防止反序列化创建多个对象呢？ObjectInputStream对象在反序列化的时候判断是否存在readResolve方法，如果存在则调用此方法进行创建。
```java
if (obj != null && this.handles.lookupException(this.passHandle) == null && desc.hasReadResolveMethod()) {
    Object rep = desc.invokeReadResolve(obj);
    if (unshared && rep.getClass().isArray()) {
        rep = cloneArray(rep);
    }

    if (rep != obj) {
        if (rep != null) {
            if (rep.getClass().isArray()) {
                this.filterCheck(rep.getClass(), Array.getLength(rep));
            } else {
                this.filterCheck(rep.getClass(), -1);
            }
        }

        obj = rep;
        this.handles.setObject(this.passHandle, rep);
    }
}
```

## 引申

从上面java的例子中可以看出单单一个单例模式就存在如此多的方式和坑，那么我们看看其他语言是怎么实现的？

### python

python由于存在GIL（全局解释锁），即同一时刻只有一个线程在运行，所以当一个类使用import引入的时候就是单例:
```python
class Singleton(object):
    def foo(self):
        pass
singleton = Singleton()
# 从其他类import进来
from foo import singleton
```

另外python中还支持通过函数装饰器，类装饰器，实现__new__和metaclass方式。其实前3中方式大同小异，都是拦截或者覆盖了new方法，创建出一个新的对象。至于metaclass则是创建类的方式，创建类的时候会调用new方式，当然是可以实现单例的。具体的代码不再赘述了。

### kotlin

kotlin几年前谷歌宣布作为安卓的官方开发语言后炙手可热，kotlin在语法上确实抛弃了java的一些历史包袱，采用了更加现代的语法，降低了编码时的心智负担。 我们来看下kotlin下DCL模式的实现：
```kotlin
class Singleton private constructor() {
    companion object {
        val instance: Singleton by lazy(mode = LazyThreadSafetyMode.SYNCHRONIZED) {
        Singleton() }
    }
}
```
kotlin的语法更加简洁， 在语言设计上kotlin增加了不少语法糖来减少手动实现的难度，比如上面的lazy语法。另外kotlin官方也有一个简单的单例模式适合无参单例:
```kotlin
object Singleton {
  fun sayHi(){}
}
```

### js

js一般认为是单进程单线程模型，所以单例模式写起来非常简单，更多的是结合js的语法使用闭包避免污染全局空间来实现：

```js
var singleton = function(fn) {
    var instance; 
    return function() { 
        return instance || (instance = fn.apply(this, arguments)); 
    } 
};
```

结合es6的class语法糖也可以写出java饿汉式，但是没有java的线程安全问题，代码不再赘述。

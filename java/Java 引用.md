### Java 引用

#### 强引用

强引用是最普遍的引用，如果一个对象具有强引用，垃圾回收器不会回收该对象，当内存空间不足时，JVM 宁愿抛出 `OutOfMemoryError`异常；只有当这个对象没有被引用时，才有可能会被回收

```java
package com.lzumetal.jvmtest;

import java.util.ArrayList;
import java.util.List;

public class StrongReferenceTest {

    static class BigObject {
        private Byte[] bytes = new Byte[1024 * 1024];
    }


    public static void main(String[] args) {
        List<BigObject> list = new ArrayList<>();
        while (true) {
            BigObject obj = new BigObject();
            list.add(obj);
        }
    }
}
```

`BigObject obj = new BigObject()`创建的这个对象时就是强引用，上面的main方法最终将抛出OOM异常：

```java
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
    at com.lzumetal.jvm.StrongReferenceTest$BigObject.<init>(StrongReferenceTest.java:9)
    at com.lzumetal.jvm.StrongReferenceTest.main(StrongReferenceTest.java:16)
```

#### 软引用

如果一个对象只具有软引用，则

- 当内存空间足够，垃圾回收器就不会回收它。
- 当内存空间不足了，就会回收该对象。
  JVM会优先回收长时间闲置不用的软引用的对象，对那些刚刚构建的或刚刚使用过的“新”软引用对象会尽可能保留。
- 如果回收完还没有足够的内存，才会抛出内存溢出异常。只要垃圾回收器没有回收它，该对象就可以被程序使用。

软引用是用来描述一些有用但并不是必需的对象，适合用来实现缓存(比如浏览器的‘后退’按钮使用的缓存)，内存空间充足的时候将数据缓存在内存中，如果空间不足了就将其回收掉。

```java
package com.lzumetal.jvmtest;

import java.lang.ref.SoftReference;

public class SoftReferenceTest {

    static class Person {

        private String name;
        private Byte[] bytes = new Byte[1024 * 1024];

        public Person(String name) {
            this.name = name;
        }
    }


    public static void main(String[] args) throws InterruptedException {
        Person person = new Person("张三");
        SoftReference<Person> softReference = new SoftReference<>(person);
        
        person = null;  //去掉强引用，new Person("张三")的这个对象就只有软引用了     

        System.gc();
        Thread.sleep(1000);

        System.err.println("软引用的对象 ------->" + softReference.get());
    }
}
```

**ReferenceQueue**

SoftReference对象是用来保存软引用，但它同时也是一个Java对象。所以，当软可及对象被回收之后，虽然这个SoftReference对象的get()方法返回null，但SoftReference对象本身并不是null，而此时这个SoftReference对象已经不再具有存在的价值，需要一个适当的清除机制，避免大量SoftReference对象带来的内存泄漏.

在java.lang.ref包里还提供了ReferenceQueue。如果在创建SoftReference对象的时候，使用了一个ReferenceQueue对象作为参数提供给SoftReference的构造方法，如

```java
    Person person = new Person("张三");
    ReferenceQueue<Person> queue = new ReferenceQueue<>();
    SoftReference<Person> softReference = new SoftReference<Person>(person, queue);
```

在SoftReference所软引用的Person对象被垃圾回收时，JVM会先将softReference对象添加到ReferenceQueue这个队列中。当我们调用ReferenceQueue的poll()方法，如果这个队列中不是空队列，那么将返回并移除前面添加的那个Reference对象。
还是上面的那个例子，测试代码

```java
public static void main(String[] args) throws InterruptedException {
    Person person = new Person("张三");
    ReferenceQueue<Person> queue = new ReferenceQueue<>();
    SoftReference<Person> softReference = new SoftReference<Person>(person, queue);

    person = null;//去掉强引用，new Person("张三")的这个对象就只有软引用了

    Person anotherPerson = new Person("李四");
    Thread.sleep(1000);

    System.err.println("软引用的对象 ------->" + softReference.get());

    Reference softPollRef = queue.poll();
    if (softPollRef != null) {
        System.err.println("SoftReference对象中保存的软引用对象已经被GC，准备清理SoftReference对象");
        //清理softReference
    }
}
```
#### 弱引用

弱引用与软引用的区别在于：只具有弱引用的对象拥有更短暂的生命周期，它只能生存到下一次垃圾收集发生之前。当垃圾回收器扫描到只具有弱引用的对象时，无论当前内存空间是否足够，都会回收它。不过，由于垃圾回收器是一个优先级很低的线程，因此不一定会很快发现那些只具有弱引用的对象。

弱引用也可以和一个引用队列（ReferenceQueue）联合使用。

使用场景：一个对象只是偶尔使用，希望在使用时能随时获取，但也不想影响对该对象的垃圾收集，则可以考虑使用弱引用来指向该对象。

参考上面的代码示例，测试弱引用

```java
    public static void main(String[] args) throws InterruptedException {
        Person person = new Person("张三");
        ReferenceQueue<Person> queue = new ReferenceQueue<>();
        WeakReference<Person> weakReference = new WeakReference<Person>(person, queue);

        person = null;//去掉强引用，new Person("张三")的这个对象就只有软引用了

        System.gc();
        Thread.sleep(1000);
        System.err.println("弱引用的对象 ------->" + weakReference.get());

        Reference weakPollRef = queue.poll();   //poll()方法是有延迟的
        if (weakPollRef != null) {
            System.err.println("WeakReference对象中保存的弱引用对象已经被GC，下一步需要清理该Reference对象");
            //清理softReference
        } else {
            System.err.println("WeakReference对象中保存的软引用对象还没有被GC，或者被GC了但是获得对列中的引用对象出现延迟");
        }
    }
```

#### 虚引用

与其他三种引用都不同，虚引用并不会决定对象的生命周期。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收。

虚引用主要用来跟踪对象被垃圾回收的活动。虚引用与软引用和弱引用的一个区别在于：虚引用必须和引用队列（ReferenceQueue）联合使用。当垃 圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列中。

```java
Object object = new Object();
ReferenceQueue queue = new ReferenceQueue ();
PhantomReference pr = new PhantomReference (object, queue); 
```

程序可以通过判断引用队列中是 否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收。程序如果发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取必要的行动。

在实际程序设计中一般很少使用弱引用与虚引用，使用软引用的情况较多，这是因为软引用可以加速JVM对垃圾内存的回收速度，可以维护系统的运行安全，防止内存溢出（OutOfMemory）等问题的产生。
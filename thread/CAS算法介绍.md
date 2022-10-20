#### CAS算法介绍

CAS(V,E,N)函数包含三个参数：V表示内存地址，E表示旧的预期值，N表示新值。仅当V地址处的值等于E时，才会将V地址处的值设置为N。CAS操作是报着乐观的态度进行的，它总是认为自己可以成功完成操作(乐观锁，相反的sync是被关锁)。当多个操作同时使用CAS操作一个变量时，只有一个会胜出，并成功更新，其余均会失败。失败的线程不会被挂起，尽被告知失败，并且允许再次尝试。

与锁相比，使用CAS会使程序看起来更加复杂一些，但由于其非阻塞的，它对死锁问题天生免疫，并且，线程间的相互影响也非常小。更为重要的是，使用无锁的方式完全没有锁竞争带来的系统开销，也没有线程间频繁调度带来的开销，因此，他要比基于锁的方式拥有更优越的性能。

我们以AtomicLong的源码为例子，来看看CAS函数的原理是怎么样的：

```java
//expect 期望值，update 新值
public final boolean compareAndSet(long expect, long update) {
        return unsafe.compareAndSwapLong(this, valueOffset, expect, update);
 }

public final native boolean compareAndSwapLong(Object var1, long var2, long var4, long var6);
```

从上往下看，可以看到最终代码调用的是一个本地方法，即它的实现是有c/c++来实现：

compareAndSwapInt()方法在```http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/69087d08d473/src/share/vm/prims/unsafe.cpp```目录下：

```c
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapLong(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jlong e, jlong x))

  UnsafeWrapper("Unsafe_CompareAndSwapLong");

  Handle p (THREAD, JNIHandles::resolve(obj));
//获取要更新的地址
  jlong* addr = (jlong*)(index_oop_from_field_offset_long(p(), offset));

#ifdef SUPPORTS_NATIVE_CX8
//调用Atomic::cmpxchg方法
//x 更新值， e 期望值
  return (jlong)(Atomic::cmpxchg(x, addr, e)) == e;
```

接着这个方法中调用cmpxchg方法，实现类```openjdk/hotspot/src/os_cpu/linux_x86/vm/atomic_linux_x86.inline.cpp```目录下：

```c++
inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
  int mp = os::is_MP();
  __asm__ volatile (LOCK_IF_MP(%4) "cmpxchgl %1,(%3)"
                    //输出寄存器
                    : "=a" (exchange_value) 
                    //输入
                    : "r" (exchange_value), "a" (compare_value), "r" (dest), "r" (mp)
               
                    : "cc", "memory");
  return exchange_value;
}
```

我们继续看代码：我们通过文件名可以知道，针对不同的操作系统,JVM 对于 Atomic::cmpxchg 应该有不同的实现。由于我们服务基本都是使用的是64位linux，所以我们就看看linux_x86 的实现。

- `__asm__` 的意思是这个是一段内嵌汇编代码。也就是在 C 语言中使用汇编代码。
- 这里的 `volatile`和 JAVA 有一点类似，但不是为了内存的可见性，而是告诉编译器对访问该变量的代码就不再进行优化。
- `LOCK_IF_MP(%4)` 的意思就比较简单，就是如果操作系统是多线程的，那就增加一个 LOCK。
- `cmpxchgl` 就是汇编版的“比较并交换”。但是我们知道比较并交换，有三个步骤，不是原子的。所以在多核情况下加一个 LOCK，由CPU硬件保证他的原子性。
- 我们再看看 LOCK 是怎么实现的呢？我们去Intel的官网上看看，可以知道LOCK在的早期实现是直接将 cup  的总线阻塞，这样的实现可见效率是很低下的。后来优化为X86 cpu  有锁定一个特定内存地址的能力，当这个特定内存地址被锁定后，它就可以阻止其他的系统总线读取或修改这个内存地址。

**内嵌汇编代码说明**

```objectivec
https://www.cnblogs.com/wanghuizhao/p/16388211.html
asm (
  "汇编语句模板"
  :输出寄存器
  :输入寄存器
  :会被修改的寄存器
)   
```

```: "=a" (exchange_value)```：代码执行完成后，将寄存器```eax```内的值赋值给```exchange_value，``` ,```"=a"```中的```a```称为限制符，```=```表示这是输出寄存器并且其中的值将被输出值替代。

```: "r" (exchange_value), "a" (compare_value), "r" (dest), "r" (mp)```：表示```compare_value```存入```eax```寄存器，而```exchange_value```、```dest```、```mp```的值存入任意的通用寄存器。常用寄存器限制符中，```a```一般表示```eax```寄存器，```r```任意动态分配的寄存器。

```"cmpxchgl %1,(%3)"```：比较并交互操作数。嵌入汇编程序规定把输出和输入寄存器统一按顺序编号（相当于 C 语言中的占位符），顺序是从输出寄存器序列从左到右、从上到下以```%0 开始。```，```%3```表示存储操作地址的寄存器，所以```(%3)```是一个内存操作即获取```dest```地址存储的值。整段代码的意思是：先取出```eax```寄存器存储的```compare_value```与```(%3)```进行比较，如果相等，则将```%1```的值替换```dest```地址指向的值，否则将```(%3)```的值设置到```eax```寄存器，然后通过输出段```"=a" (exchange_value) ```，将```eax```寄存器的值设置到```exchange_value```然后返回。

**intel手册对lock前缀的说明如下**：

1. 确保对内存的读-改-写操作原子执行。在Pentium及Pentium之前的处理器中，带有lock前缀的指令在执行期间会锁住总线，使得其他处理器暂时无法通过总线访问内存。很显然，这会带来昂贵的开销。从Pentium 4，Intel Xeon及P6处理器开始，intel在原有总线锁的基础上做了一个很有意义的优化：如果要访问的内存区域（area of memory）在lock前缀指令执行期间已经在处理器内部的缓存中被锁定（即包含该内存区域的缓存行当前处于独占或以修改状态），并且该内存区域被完全包含在单个缓存行（cache line）中，那么处理器将直接执行该指令。由于在指令执行期间该缓存行会一直被锁定，其它处理器无法读/写该指令要访问的内存区域，因此能保证指令执行的原子性。这个操作过程叫做缓存锁定（cache locking），缓存锁定将大大降低lock前缀指令的执行开销，但是当多处理器之间的竞争程度很高或者指令访问的内存地址未对齐时，仍然会锁住总线。
2. 禁止该指令与之前和之后的读和写指令重排序（有序性）。
3. 把写缓冲区中的所有数据刷新到内存中（可见性）。

上面的第1点保证了CAS操作是一个原子操作，第2点和第3点所具有的内存屏障效果，保证了CAS同时具有volatile读和volatile写的内存语义。

#### 缺点

1. 循环时间开销很大。
2. ABA问题。即在A的基础上修改成B然后再修改成A，整个过程，cas函数无法感知。ava并发包为了解决这个问题，提供了一个带有标记的原子引用类“AtomicStampedReference”，它可以通过控制变量值的版本来保证CAS的正确性。
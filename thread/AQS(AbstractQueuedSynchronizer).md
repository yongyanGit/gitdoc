### AQS(AbstractQueuedSynchronizer)

AQS是一个用来构建锁和同步器，各种并发包下ReentrantLock、ReadWriteLock，以及Semaphone、CountDownLatch等都是基于AQS来构建的。

1、AQS在内部定义了一个volatitle int state 变量，表示同步状态：当线程调用lock方法时，如果state=0，说明没有任何线程占用共享资源的锁，可以获得锁并将state=1；如果state=1，说明有线程目前正在使用共享锁，其他线程必须加入同步队列进行等待。

```java
/**
 * The synchronization state.
 */
private volatile int state;
```

2、AQS在内部定义了一个内部类：Node。它是对要访问同步代码块的线程的封装，它包含了当前线程的状态waitStatus、prev节点、next节点。通过node节点构成一个双向链表结构的同步队列，来完成获取锁的排队工作，当有线程获取锁失败后，就会被添加到队列末尾。

```java
/**
         * Status field, taking on only the values:
         *   SIGNAL:     The successor of this node is (or will soon be)
         *               blocked (via park), so the current node must
         *               unpark its successor when it releases or
         *               cancels. To avoid races, acquire methods must
         *               first indicate they need a signal,
         *               then retry the atomic acquire, and then,
         *               on failure, block.
         *   CANCELLED:  This node is cancelled due to timeout or interrupt.
         *               Nodes never leave this state. In particular,
         *               a thread with cancelled node never again blocks.
         *   CONDITION:  This node is currently on a condition queue.
         *               It will not be used as a sync queue node
         *               until transferred, at which time the status
         *               will be set to 0. (Use of this value here has
         *               nothing to do with the other uses of the
         *               field, but simplifies mechanics.)
         *   PROPAGATE:  A releaseShared should be propagated to other
         *               nodes. This is set (for head node only) in
         *               doReleaseShared to ensure propagation
         *               continues, even if other operations have
         *               since intervened.
         *   0:          None of the above
         *
         * The values are arranged numerically to simplify use.
         * Non-negative values mean that a node doesn't need to
         * signal. So, most code doesn't need to check for particular
         * values, just for sign.
         *
         * The field is initialized to 0 for normal sync nodes, and
         * CONDITION for condition nodes.  It is modified using CAS
         * (or when possible, unconditional volatile writes).
         */
		//当前线程状态
        volatile int waitStatus;

        //它的上一个线程
        volatile Node prev;

       //下一个它负责唤醒的线程
        volatile Node next;

        //node节点持有的线程
        volatile Thread thread;
		
		/** Marker to indicate a node is waiting in shared mode */
		//共享模式
        static final Node SHARED = new Node();
        /** Marker to indicate a node is waiting in exclusive mode */
		//独占模式
        static final Node EXCLUSIVE = null;
```

3、AQS通过内部类ConditionObject构建等待队列，当Condition调用wait()方法后，线程将会被加入到等待队列中，而当Condition调用signal()方法后，线程将从等待队列转移到同步队列中进行锁竞争。

4、AQS和Condition各自维护了不同的队列，在使用Lock和Condition的时候，其实是两个队列的相互移动。


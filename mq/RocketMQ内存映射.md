# RocketMQ内存映射

RocketMQ通过使用内存映射文件来提高IO访问性能，无论是CommitLog、  ConsumeQueue还是IndexFile，单个文件都被设计为固定长度，如果一个文件写满以后再创建一个新文件，文件名就为该文件第一条消息对应的全局物理偏移量。例如CommitLog的文件组织方式如下图所示。
![](/Users/user/Documents/workSpace/gitdoc/images/mq/49.png)

RocketMQ使用 MappedFile、 MappedFileQueue来封装存储文件。 

![](/Users/user/Documents/workSpace/gitdoc/images/mq/50.png)

### MappedFileQueue

MappedFileQueue是MappedFile的管理容器，MappedFileQueue是对存储目录的封装。MappedFileQueue类的核心属性如下：

```java
private final String storePath; // 存储目录
private final int mappedFileSize; // 单个文件的存储大小
private final CopyOnWriteArrayList<MappedFile> mappedFiles = new CopyOnWriteArrayList<MappedFile>(); //MappedFile文件集合
private final AllocateMappedFileService allocateMappedFileService; // 创建MappedFile服务类
private long flushedWhere = 0; // 当前刷盘指针，表示该指针之前的所有数据全部持久化到磁盘
private long committedWhere = 0; // 当前数据提交指针，内存中ByteBuffer当前的写指针，该值大于等于flushedWhere
private volatile long storeTimestamp = 0; // 刷盘时间戳
```

### MappedFile

MappedFile是RocketMQ内存映射文件的具体实现，其核心属性如下：

```java
/ 操作系统每页大小，默认4k
public static final int OS_PAGE_SIZE = 1024 * 4;
// 当前JVM实例中MappedFile虚拟内存
private static final AtomicLong TOTAL_MAPPED_VIRTUAL_MEMORY = new AtomicLong(0);
// 当前JVM实例中MappedFile对象个数
private static final AtomicInteger TOTAL_MAPPED_FILES = new AtomicInteger(0);
// 当前该文件的写指针，从0开始(内存映射文件中的写指针)
protected final AtomicInteger wrotePosition = new AtomicInteger(0);
// 当前文件的提交指针，如果开启transientStorePoolEnable，则数据会存储在TransientStorePool中，然后提交到内存映射ByteBuffer中，再刷写到磁盘。
protected final AtomicInteger committedPosition = new AtomicInteger(0);
// 刷写到磁盘指针，该指针之前的数据持久化到磁盘中
private final AtomicInteger flushedPosition = new AtomicInteger(0);
// 文件大小
protected int fileSize;
// 文件通道
protected FileChannel fileChannel;
/**
 * Message will put to here first, and then reput to FileChannel if writeBuffer is not null.
 */
//堆外内存ByteBuffer，如果不为空，数据首先将存储在该Buffer中，然后提交到MappedFile对应的内存映射文件Buffer。transientStorePoolEnable为true时不为空。
protected ByteBuffer writeBuffer = null;
// 堆外内存池，transientStorePoolEnable为true时启用。
protected TransientStorePool transientStorePool = null;
// 文件名称
private String fileName;
// 该文件的初始偏移量
private long fileFromOffset;
// 物理文件
private File file;
// 物理文件对应的内存映射Buffer
private MappedByteBuffer mappedByteBuffer;
// 文件最后一次内容写入时间
private volatile long storeTimestamp = 0;
// 是否是MappedFileQueue队列中第一个文件
private boolean firstCreateInQueue = false;
```

在详细介绍RocketMQ的MappedFile之前，我们先插播一段关于MappedByteBuffer的介绍，它是RocketMQ实现内存映射的关键，也是Java官方给出的内存映射方案。

### MappedByteBuffer

在深入MappedByteBuffer之前，先看看计算机内存管理的几个术语：

- MMC：CPU的内存管理单元。
- 物理内存：即内存条的内存空间。
- 虚拟内存：计算机系统内存管理的一种技术。它使得应用程序认为它拥有连续的可用的内存（一个连续完整的地址空间），而实际上，它通常是被分隔成多个物理内存碎片，还有部分暂时存储在外部磁盘存储器上，在需要时进行数据交换。
- 页面文件：物理内存被占满后，将暂时不用的数据移动到硬盘上。
- 缺页中断：当程序试图访问已映射在虚拟地址空间中但未被加载至物理内存的一个分页时，由MMC发出的中断。如果操作系统判断此次访问是有效的，则尝试将相关的页从虚拟内存文件中载入物理内存。

如果正在运行的一个进程，它所需的内存是有可能大于内存条容量之和的，如内存条是256M，程序却要创建一个2G的数据区，那么所有数据不可能都加载到内存（物理内存），必然有数据要放到其他介质中（比如硬盘），待进程需要访问那部分数据时，再调度进入物理内存。

假设你的计算机是32位，那么它的地址总线是32位的，也就是它可以寻址0xFFFFFFFF（4G）的地址空间，但如果你的计算机只有256M的物理内存0x0FFFFFFF（256M），同时你的进程产生了一个不在这256M地址空间中的地址，那么计算机该如何处理呢？

计算机会对虚拟内存地址空间（32位为4G）进行分页，从而产生页（page），对物理内存地址空间（假设256M）进行分页产生页帧（page frame），页和页帧的大小一样，所以虚拟内存页的个数势必要大于物理内存页帧的个数。在计算机上有一个页表（page  table），就是映射虚拟内存页到物理内存页的，更确切的说是页号到页帧号的映射，而且是一对一的映射。

那么问题来了，虚拟内存页的个数 >  物理内存页帧的个数，岂不是有些虚拟内存页的地址永远没有对应的物理内存地址空间？不是的，操作系统是这样处理的：如果要用的页没有找到，操作系统会触发一个页面失效（page  fault）功能，操作系统找到一个最少使用的页帧，使之失效，并把它写入磁盘，随后把需要访问的页放到页帧中，并修改页表中的映射，保证了所有的页都会被调度。

FileChannel提供了map方法把文件映射到虚拟内存：

```java
// 只保留了核心代码
public MappedByteBuffer map(MapMode mode, long position, long size)  throws IOException {
    // allocationGranularity一般等于64K，它是虚拟内存的分配粒度，由操作系统指定
    // 这里将position与分配粒度取余，然后真实映射起始位置为mapPosition = position-pagePosition,position 是参数指定的 position，pagePosition是根据内存分配粒度取余的结果，最终算出映射起始地址，这样算是为了内存对齐
    // 这样无论position为多少，得出的各个MappedByteBuffer实例之间的内存都是成块对齐的
    // 对齐的好处：如果两个不同的MappedByteBuffer，即便它们的position不同，但是只要它们有公共映射区域的话，这些公共区域在物理内存上的分页会被共享
    // 如果它们的MapMode是PRIVATE的话，那么会copy-on-write的方式来对修改内容进行私有化
    // 而如果它们的MapMode是SHARED的话，那么对映射的修改，其他实例均可见
    // 实际上，上述的过程都是内核来做的，我们要做的只是调用map0时将对齐好的position输入即可，这实际上是map0下层使用的mmap系统调用的约束
    
  //计算余数
  int pagePosition = (int)(position % allocationGranularity);
  	//计算起始地址，整数，用来对齐
    long mapPosition = position - pagePosition;
    long mapSize = size + pagePosition;
    try {
      	//指定文件应被映射到进程空间的起始地址
        addr = map0(imode, mapPosition, mapSize);
    } catch (OutOfMemoryError x) {
      	//内存溢出，进行gc
        System.gc();
        try {
            Thread.sleep(100);
        } catch (InterruptedException y) {
            Thread.currentThread().interrupt();
        }
        try {
          	//重新映射
            addr = map0(imode, mapPosition, mapSize);
        } catch (OutOfMemoryError y) {
            // After a second OOME, fail
            throw new IOException("Map failed", y);
        }
    }
    int isize = (int)size;
    Unmapper um = new Unmapper(addr, mapSize, isize, mfd);
    if ((!writable) || (imode == MAP_RO)) {
        return Util.newMappedByteBufferR(isize,
                                         addr + pagePosition,
                                         mfd,
                                         um);
    } else {
        return Util.newMappedByteBuffer(isize,
                                        addr + pagePosition,
                                        mfd,
                                        um);
    }
}
```

上述代码可以看出：

1. map通过native函数map0完成文件的映射工作，下层使用系统调用mmap
2. 如果第一次文件映射导致OOM，则手动触发垃圾回收，休眠100ms后再次尝试映射，如果失败，则抛出异常。
3. 如果映射成功，会得到虚拟内存地址address
4. 根据得到的虚拟内存地址，通过newMappedByteBuffer方法初始化MappedByteBuffer实例，其最终返回的是DirectByteBuffer，如下就是从内存地址生成DirectByteBuffer实例的过程

```java
static MappedByteBuffer newMappedByteBuffer(int size, long addr, FileDescriptor fd, Runnable unmapper) {
    MappedByteBuffer dbb;
    if (directByteBufferConstructor == null)
        initDBBConstructor();
    dbb = (MappedByteBuffer)directByteBufferConstructor.newInstance(
          new Object[] { new Integer(size),
                         new Long(addr),
                         fd,
                         unmapper }
    return dbb;
}
// 访问权限
private static void initDBBConstructor() {
    AccessController.doPrivileged(new PrivilegedAction<Void>() {
        public Void run() {
            Class<?> cl = Class.forName("java.nio.DirectByteBuffer");
                Constructor<?> ctor = cl.getDeclaredConstructor(
                    new Class<?>[] { int.class,
                                     long.class,
                                     FileDescriptor.class,
                                     Runnable.class });
                ctor.setAccessible(true);
                directByteBufferConstructor = ctor;
        }});
}
```

由于FileChannelImpl和DirectByteBuffer不在同一个包中，所以有权限访问问题，通过AccessController类获取DirectByteBuffer的构造器进行实例化。

map0()函数返回一个虚拟内存地址address，这样就无需调用read或write方法对文件进行读写，通过address就能够操作文件。底层采用unsafe.getByte方法，通过（address + 偏移量）获取指定内存的数据。

- 第一次访问address所指向的内存区域，导致缺页中断，中断响应函数会在交换区中查找相对应的页面，如果找不到（也就是该文件从来没有被读入内存的情况），则从硬盘上将文件指定页读取到物理内存中（非jvm堆内存）。
- 如果在拷贝数据时，发现物理内存不够用，则会通过虚拟内存机制（swap）将暂时不用的物理页面交换到硬盘的虚拟内存中。

MappedByteBuffer的效率之所以比read/write高，主要是因为read/write过程会涉及到用户内存拷贝到内核缓冲区，而MappedByteBuffer在发生缺页中断时，也是直接将硬盘内容拷贝到了内核，但是缺少内核到用户空间这一步骤，这也就是我们所说的零拷贝技术。所以，采用内存映射的读写效率要比传统的read/write性能高。

MappedByteBuffer使用虚拟内存，因此分配(map)的内存大小不受JVM的-Xmx参数限制，但是也是有大小限制的。如果当文件超出大小限制Integer.MAX_VALUE时，可以通过position参数重新map文件后面的内容。

至此，我们已经了解了文件内存映射的技术，既然Java已经提供了内存映射的方案，还有MappedFile什么事呢？这一层封装又有何意义呢？接下来再回到MappedFile的介绍中来，我将详细介绍RocketMQ的MappedFile都对原生内存映射方案做了哪些增强。


# TLAB

LAB的全称是Thread Local Allocation Buffer，即线程本地分配缓存区，这是一个线程专用的内存分配区域。 
  由于对象一般会分配在堆上，而堆是全局共享的。因此在同一时间，可能会有多个线程在堆上申请空间。因此，每次对象分配都必须要进行同步（虚拟机采用CAS配上失败重试的方式保证更新操作的原子性），而在竞争激烈的场合分配的效率又会进一步下降。JVM使用TLAB来避免多线程冲突，在给对象分配内存时，每个线程使用自己的TLAB，这样可以避免线程同步，提高了对象分配的效率。 
 TLAB本身占用eEden区空间，在开启TLAB的情况下，虚拟机会为**每个Java线程分配一块TLAB空间**。参数-XX:+UseTLAB开启TLAB，默认是开启的。TLAB空间的内存非常小，缺省情况下仅占有整个Eden空间的1%，当然可以通过选项-XX:TLABWasteTargetPercent设置TLAB空间所占用Eden空间的百分比大小。 
  由于TLAB空间一般不会很大，因此大对象无法在TLAB上进行分配，总是会直接分配在堆上。TLAB空间由于比较小，因此很容易装满。比如，一个100K的空间，已经使用了80KB，当需要再分配一个30KB的对象时，肯定就无能为力了。这时虚拟机会有两种选择，第一，废弃当前TLAB，这样就会浪费20KB空间；第二，将这30KB的对象直接分配在堆上，保留当前的TLAB，这样可以希望将来有小于20KB的对象分配请求可以直接使用这块空间。实际上虚拟机内部会维护一个叫作refill_waste的值，当请求对象大于refill_waste时，会选择在堆中分配，若小于该值，则会废弃当前TLAB，新建TLAB来分配对象。这个阈值可以使用TLABRefillWasteFraction来调整，它表示TLAB中允许产生这种浪费的比例。默认值为64，即表示使用约为1/64的TLAB空间作为refill_waste。默认情况下，TLAB和refill_waste都会在运行时不断调整的，使系统的运行状态达到最优。如果想要禁用自动调整TLAB的大小，可以使用-XX:-ResizeTLAB禁用ResizeTLAB，并使用-XX:TLABSize手工指定一个TLAB的大小。 
 -XX:+PrintTLAB可



   
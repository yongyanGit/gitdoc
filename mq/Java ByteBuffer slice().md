# Java ByteBuffer slice()

java.nio.ByteBuffer类的slice()方法用于创建一个新的字节缓冲区，其内容是给定缓冲区内容的共享子序列。

新缓冲区的内容将从该缓冲区的当前位置开始。对该缓冲区内容的更改将在新缓冲区中可见，反之亦然。这两个缓冲区的位置，限制和标记值将是独立的。

新缓冲区的位置将为零，其容量和限制将为该缓冲区中剩余的浮点数，并且其标记将不确定。当且仅当该缓冲区是直接缓冲区时，新缓冲区才是直接缓冲区；当且仅当该缓冲区是只读缓冲区时，新缓冲区才是只读缓冲区。

用法:

public abstract ByteBuffer slice()

 public static void main(String[] args) {
        ByteBuffer buffer = ByteBuffer.allocate(10);

```java
    for (int i = 0; i< buffer.capacity(); i++) {
        buffer.put((byte)i);
    }

    buffer.position(2);
    buffer.limit(6);

    ByteBuffer sliceBuffer = buffer.slice();

    for (int i = 0; i < sliceBuffer.capacity(); i++) {
        byte b = sliceBuffer.get(i);
        b *= 2;
        sliceBuffer.put(i, b);
    }
    buffer.position(0);
    buffer.limit(10);
    while (buffer.hasRemaining()) {
        System.out.print(buffer.get() + " ");
    }
}
```
结果：0 1 4 6 8 10 6 7 8 9
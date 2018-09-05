# First Chapter: ByteBuf

## ByteBuf

### ![](/assets/ByteBuf.jpg)

### 按内存分配：

* 堆内存 （使用字节数组）
* 直接内存（使用jdk  ByteBuffer）

### 按内存回收：

* 基于对象池（可以循环利用已创建的ByteBuf）

* 普通

## AbstractByteBuf

### 主要成员变量

```java
static final ResourceLeakDetector<ByteBuf> leakDetector =
            ResourceLeakDetectorFactory.instance().newResourceLeakDetector(ByteBuf.class);

    int readerIndex;
    int writerIndex;
    private int markedReaderIndex;
    private int markedWriterIndex;
    private int maxCapacity;
```

所有ByteBuf共享同一个leakDetector检测对象是否泄露

### 读操作

### 写操作

```java
    @Override
    public ByteBuf writeBytes(byte[] src, int srcIndex, int length) {
        ensureWritable(length);
        setBytes(writerIndex, src, srcIndex, length);
        writerIndex += length;
        return this;
    }

    @Override
    public ByteBuf ensureWritable(int minWritableBytes) {
        if (minWritableBytes < 0) {
            throw new IllegalArgumentException(String.format(
                    "minWritableBytes: %d (expected: >= 0)", minWritableBytes));
        }
        ensureWritable0(minWritableBytes);
        return this;
    }

    final void ensureWritable0(int minWritableBytes) {
        ensureAccessible();
        if (minWritableBytes <= writableBytes()) {
            return;
        }

        if (minWritableBytes > maxCapacity - writerIndex) {
            throw new IndexOutOfBoundsException(String.format(
                    "writerIndex(%d) + minWritableBytes(%d) exceeds maxCapacity(%d): %s",
                    writerIndex, minWritableBytes, maxCapacity, this));
        }

        // Normalize the current capacity to the power of 2.
        int newCapacity = alloc().calculateNewCapacity(writerIndex + minWritableBytes, maxCapacity);

        // Adjust to the new capacity.
        capacity(newCapacity);
    }
```

Netty的ByteBuf的优势在于可以动态扩容，而JDK的ByteBuffer并不能。分析写入如下：

1. 先判断待写入长度minWritableBytes，如果不合法（小于0），抛异常。
2. 进入方法ensureWritable0，如果minWritableBytes小于剩余可写入长度，返回。
3. 判断minWritableBytes + 已写入长度writerIndex，如果大于最大容量maxCapacity，抛异常。
4. 扩容，扩容代码如下。




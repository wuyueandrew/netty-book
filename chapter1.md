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

#### 扩容

```java
    @Override
    public int calculateNewCapacity(int minNewCapacity, int maxCapacity) {
        if (minNewCapacity < 0) {
            throw new IllegalArgumentException("minNewCapacity: " + minNewCapacity + " (expected: 0+)");
        }
        if (minNewCapacity > maxCapacity) {
            throw new IllegalArgumentException(String.format(
                    "minNewCapacity: %d (expected: not greater than maxCapacity(%d)",
                    minNewCapacity, maxCapacity));
        }
        final int threshold = CALCULATE_THRESHOLD; // 4 MiB page

        if (minNewCapacity == threshold) {
            return threshold;
        }

        // If over threshold, do not double but just increase by threshold.
        if (minNewCapacity > threshold) {
            int newCapacity = minNewCapacity / threshold * threshold;
            if (newCapacity > maxCapacity - threshold) {
                newCapacity = maxCapacity;
            } else {
                newCapacity += threshold;
            }
            return newCapacity;
        }

        // Not over threshold. Double up to 4 MiB, starting from 64.
        int newCapacity = 64;
        while (newCapacity < minNewCapacity) {
            newCapacity <<= 1;
        }

        return Math.min(newCapacity, maxCapacity);
    }
```

1. 判断写入后最小长度minNewCapacity，如果不合法（小于0或大于最大长度maxCapacity），抛异常。
2. 设置阈值threshold为4MB，如果minNewCapacity等于threshold，返回threshold。
3. 如果minNewCapacity大于阈值threshold，按照每次4MB步进扩容，直至达到最大容量maxCapacity。

4. 如果minNewCapacity小于阈值threshold，从64开始倍增扩容。

原因为，如果以minNewCapacity为目标容量，下次写入又需要扩容，当内存比较小时，用倍增不会浪费太多容量，当内存达到阈值时，如果继续倍增就会造成内存浪费。

### 操作索引

### 重用缓冲区

```java
    protected final void adjustMarkers(int decrement) {
        int markedReaderIndex = this.markedReaderIndex;
        if (markedReaderIndex <= decrement) {
            this.markedReaderIndex = 0;
            int markedWriterIndex = this.markedWriterIndex;
            if (markedWriterIndex <= decrement) {
                this.markedWriterIndex = 0;
            } else {
                this.markedWriterIndex = markedWriterIndex - decrement;
            }
        } else {
            this.markedReaderIndex = markedReaderIndex - decrement;
            markedWriterIndex -= decrement;
        }
    }
```




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

## AbstractReferenceCountedByteBuf

## 




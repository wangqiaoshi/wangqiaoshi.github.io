---
layout:     default
title:      Drill Vector 源码分析
category: blog
description: 
---
# Drill Vectors
vectors是用于drill分布式计算交互的数据结构,它的存储方式也是引用了netty的ByteBuf.

为什么Drill要用Vectors来作为它的存储结构,Drill官方中是这样介绍:

## Verctor特性

### 支持多语言的操作
ValueVectors支持C/C++/Assmebly的写操作.这样,ByteBuffer在执行JNI接口的时候不需要被修改.ValueVector一旦被构造就可以在底层操作.直接顺序不需要考虑.

### 访问
从ValueVector访问随机的元素的时间是恒定.为了达到这种性能,利用buffer的start,offset来限定元素位置.Repeated,nullable,variable width ValueVectors这些变量通过添加index指针指向另外的verctor.ValueVector一旦构造，写操作不能修改原来的数据.

### Value Vectors子集合
当一个操作返回ValueVector返回一个子集合,它需要重用ValueVector.达到这种效果,一种方式是跳过不需要的。这种方式,需要一系列的offsets(包括需要返回的ValueVector和子集合的个数).

### 内存池分配
ValueVectors利用一个或者多个缓冲区。这些buffers都来自一个池.Value vectors在迭代过程中随着schema的改变它们自己被创建和销毁.

### 同质的变量类型
同一个Value Vector的每一个变量是相同的类型.在schema变化的时候,Record Batch是有责任的创建一个新的变量.

## 定义

### 数据类型
变量的定义在[Drill DateTypes](https://docs.google.com/spreadsheet/ccc?key=0AvUC_YMxQ9UkdE5kbHpfMzE1YlBSSkx0a0NrRnh1cFE#gid=0)文档中.个别类型列在'Basic Data Types',value vector类型能被找到在‘Value Vectors’下面找到.

### 操作
一个操作是用于转化stream的列.它操作在Record Batches或者常量.

### Record Batch
一段范围的记录的字段变量集合.Record Batch是有Value Vectors的组成,每一个batch组成一个完整的schema.

### Value Vector
value vector是有一个或多个连续的buffes组成.它可能有序列变量,zero or more which store any metadata associated with the ValueVector.

## 数据结构
ValueVector存储值在Bytebuf(一块连续内存区).附加了支持变量的宽度,空值,repeated值和可选的值.  
这些是主要类型,对于由一种或多种混合的ValueVectors的tables需要组合这些类型.固定宽度ValueVectors空列上,不重复的是不需要直接查找;元素能被直接访问通过位置乘以跨步.

### Fixed Width Values
固定宽度ValueVector简单的含有序列值.ByteBuf[0]+Index*Stride(Index between 0 and base)可随机访问元素.下面举例INT4型的缓冲区 values[1..6]:



![](http://drill.apache.org/docs/img/value1.png)

### Nullable Values
Nullable values代表了比特值.每个比特值就是ValueVector的元素.假如bit值没有被设置,value就为空.否者检索到潜在的buffer.下面举例INT4型的2,3,6的NullableValueVector:



![](http://drill.apache.org/docs/img/value2.png)

### Repeated Values
用于存储一个元素(含有多个值,比如json array).含有offset和count pairs的表去代表repeated element在valuevector.当count=0表示这个元素没有值(offset是没有被使用的).下面举例3个fields,一个有两个值,一个没有值,一个只有一个值:


![](http://drill.apache.org/docs/img/value3.png)

ValueVector等价于json

x:[1,2]

x:[]

x:[3]

### Variable Width Values
variable width values存储是连续的ByteBuf.每一个元素代表了entry(固定长度的offset).entry的长度是下一个entry的offset减去上一个的offset.正因为如此,offset表将会比实际存储的多一个entry,最后一个entry来指向buffer的末端.


![](http://drill.apache.org/docs/img/value4.png)

### Repeated Map Vectors
repeated map vector含有一个或多个maps(类似于在json中多个对象数组).在map的值是连续存储在ByteBuf中.为了能访问到特定的记录,查找表(count,offset).这张查找表指向第一个repeated field在每一列,count指明这列元素的最大个数.下面举例,有两条记录的RepeatedMap,一个有两个object,一个有一个object:


![](http://drill.apache.org/docs/img/value5.png)
ValueVector等价于JSON:

x:[{name:'Sam',age:1},{name:'Max',age:2}]

x:[{name:'Joe',age:3}]


### Selection Vectors
selection Vector是ValueVector的子集.确定ValueVector每一个元素的offset的列表包含在SelectionVector. 对于一定宽度的ValueVector,offsets参考于ByteBufs.对于nulltable,repeated,variable with ValuevVector,offset参考于相应的查找表.下面举例,INT4(固定宽度)的Selection Vector值2,3,5来自上面的[1..6]:


![](http://drill.apache.org/docs/img/value6.png)

下面举例,nulltable的ValueVector:


![](http://drill.apache.org/docs/img/value7.png)


[参考原文](http://drill.apache.org/docs/value-vectors/)

# Drill Vectors源码图解分析
上面是对Vectors的设计文档,Drill也是完全根据这个思路来完成Vectors的代码.


针对源码,设计文档画了它的UML class图:


![](/images/vectorClass.jpg)

ValueVector作为原始接口,它本身继承了Iterable,Accessor和Mutator是其内部类,这样Accessor和Mutator共享ValueVector的资源.Accessor是用于当前ValueVector的读操作,Mutator是对ValueVector进行写操作.一下是它需要实现的方法:

    void allocateNew() throws OutOfMemoryRuntimeException;
    boolean allocateNewSafe();
    public void setInitialCapacity(int numRecords);
    int getValueCapacity();
    void close(); 
    void clear();
    MaterializedField getField();
    TransferPair getTransferPair();
    TransferPair getTransferPair(FieldReference ref);
    TransferPair makeTransferPair(ValueVector target);
    A getAccessor();
    M getMutator();
    FieldReader getReader();
    SerializedField getMetadata();
    int getBufferSize();
    DrillBuf[] getBuffers(boolean clear);
    void load(SerializedField metadata, DrillBuf buffer);
 在设计文档指出,Value分为两种,一种固定长度,一种不固定长度.对此,设计两个接口,FixedWidthVector(固定长度的),VariableWidthVector(不是固定长度的).固定长度和不固定长度最大的区别,FixedWidthVector可以根据固定长度和index来寻址来获得数据, 而VariableWidthVector不行.在方法中可以看出,FixedWidthVector不需要getAccessor.
### VariableWidthVector的接口方法
    public void allocateNew(int totalBytes, int valueCount);
    public int getByteCapacity();
    public int load(int dataBytes, int valueCount, DrillBuf buf);
    public abstract VariableWidthMutator getMutator();
    public abstract VariableWidthAccessor getAccessor();

### FixedWidthVector的接口方法
    public void allocateNew(int valueCount);
    public int load(int valueCount, DrillBuf buf);
    public abstract Mutator getMutator();
    public void zeroVector();
BaseDataValueVector是抽象类,聚合了DrillBuf,DrillBuf只是对netty的ByteBuf重新包装了,也是数据真正存储的地方.


所以值是固定长度的Vector需要实现FixedWidthVector和BaseDataValueVector,不固定长度的Vector需要实现VariableWidthVector和BaseDataValueVector.














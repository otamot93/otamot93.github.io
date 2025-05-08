---
title: Spark常见优化
date: 2025-04-11 14:34:42
tags:
    - Spark

category: Spark
---

# 数据序列化
序列化在任何分布式应用性能中都扮演了重要的角色。如果传输对象序列化和反序列化性能不佳，则会让降低计算速度。一般情况下，Spark应用性能优化首先应该考虑序列化。Spark的目标是寻求易用性（能使用任何java类型）和性能。Spark提供两种序列化方式:
- `Java序列化`:默认情况下,Spark使用java的ObjectOutputStream框架对任何实现了[`java.io.Serializable`](https://docs.oracle.com/javase/8/docs/api/java/io/Serializable.html).接口的对象进行序列化。当然也能通过继承[`java.io.Externalizable`](https://docs.oracle.com/javase/8/docs/api/java/io/Externalizable.html).接口（实现writeExternal方法和readExternal方法）来精细化的控制序列化的表现。Java序列化方案虽然灵活，但通常相当慢，同时也会导致许多类产生较大的序列化对象。
- `Kryo序列化`:Spark 也支持使用Kryo 库(Version 4)更快的序列化对象。Kyro 序列化比Java序列化显著得更快、序列化大小更小（通常快达10倍），但是不支持所有可序列化的类型，需要手动注册哪些类需要被序列化。

<!-- more -->

通过初始化一个 `SparkConfig` ，调用`conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")`。通过此设置，不仅会序列化在不同节点shuffle的数据，也会序列化持久化到磁盘上的RDD数据。Kyro不被用于默认序列化的原因是需要自定义注册哪些类需要被序列化（降低了易用性），但是在网络交互的应用中，我们还是推荐使用kryo。从Spark2.0.0开始，Kyro序列化被用于shuffle 是简单类型、元素为简单类型的数组、字符串的RDD。

Spark自动包含Kryo，许多常用的核心Scala类被包含在Twitter chill库的AllScalaRegistrar 中。

使用 `registerKryoClasses` 方法注册自定义的类

```scala
val conf = new SparkConf().setMaster(...).setAppName(...)
conf.registerKryoClasses(Array(classOf[MyClass1], classOf[MyClass2]))
val sc = new SparkContext(conf)
```

阅读 [Kryo 文档](https://github.com/EsotericSoftware/kryo)了解更多使用方法。
如果需要序列化的对象太大，可能需要增加配置`spark.kryoserializer.buffer`的值。配置的值需要足够放下需要序列化最大的对象。

最后，如果没有注册自定义类，Kryo也同样能工作，但是他会存储类的全限定名称，这会浪费存储空间和传输带宽。注册类可以避免这种浪费，因为注册后Kryo只需要存储一个简短的整数ID。


# 内存调节

调节内存时有三个需要考虑的因素:对象占用的内存、访问这些对象的成本、垃圾回收的开销(如果数据频繁更替)。

默认情况下，Java对象都能被快速访问，但是相较于对象本身的数据，这些对象会消耗2-5倍的存储空间，有以下原因:
- 每一个不同的对象都有`对象头`,包含指向对应类的指针等信息，会占用16个字节。如果一个对象数量很小的话，对象头所占的大小都要比数据本身大。
- Java 字符串有40字节的额外花销(因为他们将数据保存在字节数组中并且保存了数据的长度)，因为String类内部使用UTF-16编码，一个字符会使用2个字节。所以一个长度为10字符的字符串能轻易消耗60个字节。
- 如HashMap和LinkedList等通用集合类，使用了链表的数据结构，每一个元素都是一个封装对象（如Map.Entry），封装对象不仅有对象头，同样有指向链表下一个元素的指针(通常是8个字节)。
- 原始类型的集合通常被保存为“装箱”对象，例如int值会被保存为java.lang.Integer

这个章节将会从Spark内存管理的概述开始，讨论有些方法能使应用变得更加高效。尤其是我们会介绍如何去确认对象的内存使用和提升方法：通过改变数据的结构或者序列化存储对象。接下来我们会讨论如何调整Spark的缓存大小和Java垃圾回收。


## 内存管理概述
Spark中的内存使用主要分为两个类别: 执行和存储。执行内存用于shuffle、join、sort和aggregation计算使用，存储内从用于缓存和传播内部数据。在Spark中，执行和存储共享一块内存区域(M)。当没有使用执行内存时，存储能占用所有可用内存，反之亦然。执行在必要时会驱除存储内存，但只会将存储内存降低到一个确定的阈值(R)。换句话说，阈值R能决定在M的一块区域中存储的缓存块永远不会被驱除。由于实现的复杂度，存储内存可能不会驱除计算内存。

这个设计有以下好处。首先，应用如果不需要缓存，计算能使用所有的内存，避免不必要的磁盘溢出。其次，应用如果使用缓存，能至少保存在一块不会被驱除的存储空间中（R）。最后，这将为各种工作负载提供合理的开箱即用的性能，而不需要需要懂得内存隔离的专业人士。

虽然有两个相关的配置，最大数情况下用户只需要使用默认值即可。
- `spark.memory.fraction` 表示内存区域M占JVM中堆内存的比例，默认为0.6.堆内存剩余的40%空间用于保存用户的数据结构、Spark内部元数据，以及在稀疏和异常大的记录情况下防止OOM错误。（在处理稀疏数据时，Spark 需要确保不会因为数据的稀疏性导致内存浪费。例如，保留一定比例的内存用于防止因数据稀疏性引起的内存不足）
- `spark.memory.storageFraction` 表示R所占M的比例,默认为0.5。R是M中存储缓存块并不能被执行驱除的存储区域

`spark.memory.fraction`的值需要被设置为了堆空间能更舒适地使适应老年代或“永久代”（Java8之后永久代被元空间取代）
- `spark.memory.fraction`设置的值应使Spark的内存使用主要分布在JVM的老年代内存中。这是因为Spark的任务通常是长时间运行的，需要稳定的内存分配，而老年代适合存放生命周期较长的对象。
- 如果`spark.memory.fraction`设置得太高，Spark内存管理系统会占用过多的JVM堆内存，导致年轻代内存不足。这会导致频繁的垃圾回收，尤其是年轻代的垃圾回收，进而影响性能。因此，适当的设置能确保老年代有足够的空间，避免频繁的垃圾回收。

## 推断内存消耗

了解数据集内存消耗最好的方式就是创建一个RDD，通过cache方法缓存，然后在Spark-UI的Storage页面查看。
估算一个特定对象的内存消耗，可以使用`SizeEstimator`类的estimate方法。这对于尝试不同的数据布局以减少内存使用，以及确认广播变量在每个执行器堆上占用的空间量非常有效。


## 调整数据结构
减少内存消耗的第一种方法就是避免使用会增加开销的Java特性，例如基于指针的数据结构和包装对象。
- 设计数据结构优先选择使用数组、原始类型而不是Java或Scala中的集合类，如HashMap。fastutil库提供了方便的原始类型集合类，这些类与Java标准库兼容。
- 尽量避免包含大量小对象和指针的嵌套结构。大量小对象和指针会导致内存分散，增加内存碎片，降低缓存命中率，从而影响性能。大量小对象会增加垃圾回收器的工作量、导致频繁的垃圾回收。
例如链表结构对象
```
class Node { int value; Node next; }
```
- 考虑使用数值ID或枚举对象作为键，而不是字符串。
- 如果内存小雨32 GiB，设置JVM参数`-XX:+UseCompressedOops`,从而让指针的长度从8个字节变为4个字节。

## 序列化RDD存储

经过这些调优，对象依然过大而无法高效存储时，一个更简单的减少内存使用的方式是序列化存储，使用[RDD persistence API](https://spark.apache.org/docs/latest/rdd-programming-guide.html#rdd-persistence)中提供的序列化存储级别，例如`MEMORY_ONLY_SER`,Spark将会存储RDD分区数据为一个大的序列化后的字节数组。序列化存储的一个缺点是降低了访问时间，这是因为使用时都要先反序列化。推荐使用Kryo进行序列化，序列化后的大小会被Java序列化更小。

## 调整垃圾回收（GC）

当你的程序存储的RDD数量发生大量变动时，JVM垃圾回收可能成为一个问题。在那些只读取一次RDD然后对其运行多次操作的程序中，通常不会出现出现这个问题。（RDD持久化内存，对同一个RDD再进行transfer，不重新加载RDD）。当Java程序需要空间存放新对象时，就需要去查找所有对象并找到那些没用的对象。此处关键的是垃圾回收的消耗与对象数量成比例关系，所以使用有较少对象的数据结构有助于减少垃圾回收的消耗（如使用int数组替代LinkedList）。一个更好的方法就是持久化保存:这样一个RDD分区就只有一个对象，也就是一个字节数组。在尝试其他技术之前，如果垃圾回收一个问题，首先要尝试的就是序列化缓存。

在工作内存(运行任务的内存)和节点的RDD缓存之间同样也存在垃圾回收问题。我们将讨论如何通过控制RDD缓存的空间分配来减缓垃圾回收问题。

### 评估GC的影响

调整GC的第一步需要收集GC的回收频率、GC的回收时间等信息。可以通过在java参数中添加`-verbose:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps`，下次程序启动时，每次发生GC时，worker（executor）日志中就会打印出GC的日志。需要注意的是，需要在worker程序上设置并查看，而不是在driver程序上。

### 高级GC设置

在对GC调整之前，我们需要了解JVM内存管理的基础信息:
- 堆内存被分为新生代(Young region)和老年代(Old region)，新生代保存短生命周期的对象，而老年代则用于存放生命周期较长的对象。
- 新生代被分为三个区域：Eden、Survivor1、Survivor2
- GC程序工作的简单描述为：如果Eden区满了，将会发生minor GC（Young GC）,也就是Eden区和Survivor1区的存活对象将会被复制到Survivor2区，然后Survivor2区和Survivor1区的角色将对换（也就是下次发生minor GC时，将会将Eden区和Survivor2区的对象拷贝到Survivor1中）。如果一个对象的存活对象足够长（经过一定次数的minor GC）或者Survivor2区满了，该对象则会被移到老年代。最后，如果Old区也满了，将会触发full GC（major GC）。

Spark GC调整的目标是长时间存在的RDD数据存储在老年代，新生代有足够的空间保存短期生存对象。这将避免为了收集在任务执行期间创建的临时对象而进行的full GC。一些可能有用步骤包括:
- 通过GC日志确认是否发生很多次垃圾回收。如果在一个任务完成之前，就触发了多次full GC，这意外着没有足够的内存执行任务。
- 如果发生了多次minor GC，但是没有发生full GC，调整Eden区的大小会有作用。可以调整Eden区的大小比任务的预估值要多一些。定义Eden区的大小为E，通过设置`-Xmn=4/3*E`。将比例放大到4/3是因为要考虑到Survivor区的大小。
- 通过GC日志，如果看到老年代即将耗尽，可以通过减少参数`spark.memory.fraction`的值以减少缓存内存的使用。与其降低执行速度，不如减少缓存（为了缓存，结果减慢了执行速度，得不偿失，缓存应当缓存那些频繁使用且内存占用较少的）。当然也可以通过设置 `-xmn`减少新生代的大小。如果不这么做，也可以设置JVM的`NewRatio`的大小，很多JVM是2，也就是老年代占用堆内存的2/3。老年代应该足够大，以满足设置的`spark.memory.fraction`会使用的内存大小。
- 尝试通过设置`-XX:+UseG1GC`设置G1垃圾回收器，如果是垃圾回收存在瓶颈，使用G1能解决一些性能问题。如果exuector有比较大的内存，设置`-XX:G1HeapRegionSize`是很重要的。
- 举个例子，如果从HDFS读取数据，一个任务所使用的内存可以估算为HDFS一个块（block）的大小。block被解压后的大小通过会大2-3倍，所以我们需要设置3-4倍任务的空间，HDFS  block的大小是128Mib（通常情况下），可以估算Eden区的大小为`4*3*128 Mib`
- 设置新的JVM参数后，观察日志才看GC的频率和耗时。

我们的经验表明，GC调优的效果取决于您的应用程序（逻辑与使用的数据）和可用内存的大小。在
[GC调优参数](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/index.html)可以查看更多调优参数，但从一个高层次的维度来看，管理full GC的频率有助于减少开销。

GC调整的参数通过设置`spark.executor.defaultJavaOptions`或者`spark.executor.extraJavaOption`来实现。
```
$SPARK_HOME/bin/spark-submit --class org.example.FtMatchApp \
    ....
    --conf "spark.executor.extraJavaOptions= -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps  -XX:+UseCompressedOops  -DdatasetAFilePath='${dataset_a}' -DdatasetAMeta='${dataset_a_meta}' -DdatasetAIdColumnName='${dataset_a_id_column_name}'  -DdatasetBFilePath='${dataset_b}' -DoutputType='${output_type}' -DoutputPath='${output_path}' -DoutputColumns='${output_columns}' -DencryptedAesKey='${encrypted_aes_key}' " \
    --conf "spark.driver.extraJavaOptions=-DdatasetAFilePath='${dataset_a}' -DdatasetAMeta='${dataset_a_meta}' -DdatasetAIdColumnName='${dataset_a_id_column_name}'  -DdatasetBFilePath='${dataset_b}' -DoutputType='${output_type}' -DoutputPath='${output_path}' -DoutputColumns='${output_columns}' -DencryptedAesKey='${encrypted_aes_key}' " \
 ....
```


# 其他优化方式

## 并行度
要充分利用集群，需要为每个操作设置高并行度。Spark会根据每个文件的大小自动设置要运行的"map"阶段的任务(当然也可以手动设置参数，比如对方法`SparkContext.textFile`可以设置第二个参数作为并行度)。对于分布式的`reduce`(如`groupByKey`,`reduceByKey`)操作，默认最大并行度由上层RDD最大的数量决定。可以通过设置参数(比如`defgroupByKey(numPartitions: Int)`)或者设置`spark.default.parallelism` 修改并行度默认值。一般推荐每个cpu运行2-3个任务。

>1.该建议是为了让每个cpu都有活干。比如原始数据就两个文件，默认并行度可能就就两个，这样就只会有两个任务运行，设置再多的executor数量和cpu核数，也发挥不了集群的作用，因为干活的只会有两个任务，并且容易导致OOM。
>2.`spark.default.parallelism`另一个更高优先级的设置原则为，如果数据量特别大，每个executor内存不足以处理当前并行度的数据大小，需要将该参数设置的尽量大，让executor能处理`总数量/并行度`的数据，且能被垃圾回收掉。也就是下文`Reduce任务内存使用`的内容。



## Reduce任务内存使用
有些时候，发生`OutOfMemoryError`并不是因为存放不下RDD得数据，而是因为工作的的任务，比如正在执行`groupByKey`的reduce任务产生的工作数据太大。Spark的shuffle操作(比如sortByKey,groupByKey,reduceByKey,join等)会构建一张大的hash表来提升聚合的性能，通常占用内存会很大。最简单的一个解决方案是增加任务的并行度，这样会让每个任务的输入输入都很小。得益于多个任务复用JVM(如每个executor有两核，分配到这个executor上面的任务是100个，则JVM同时运行了两个任务，运行完一个再处理下一个任务的数据)，任务的启动成本低，Spark能高效的支持短至200毫秒的任务。所以可以安全地将并行度提高到超过集群中核数数量。


## 广播大变量

使用SparkContext中的广播功能可以大大减少每个序列化任务的大小，并降低在集群上启动任务的的成本。如果(executor)任务从driver中获取大的对象，如一张静态的查找表，考虑使用广播变量。
Spark会在master节点上打印每个任务的序列化大小，因此可以查看这些信息来判断任务是否过大，一般来说，超过20KiB的任务可能值得优化。


## 数据本地化
本地数据会对Spark的任务性能产生较大的影响，如果数据和代码（executor 程序）是在一起的，那计算就会变得更快。如果代码和数据是分开的，则一方必须要移动到另一方。通常，将序列化代码从一个地方传输得到另一个地方比传输一块数据更快，因为代码的大小远小于数据。Spark的调度是基于数据本地化这一基本原则构建的。

数据本地化是指数据与代码的接近程度。以下根据数据所处的位置划分了几种本地化级别，按从近到远排序:
- `PROCESS_LOCAL`数据和代码在同一个JVM中，这是最好的本地化界别
- `NODE_LOCAL` 数据和代码在同一个节点。比如HDFS数据和executor在同一个节点。这比`PROCESS_LOCAL`会慢一点，因为数据需要在不同进程中参数。
- `NO_PREF`数据可以从任何地方以同样的方式访问，并且没有本地性偏好。
- `RACK_LOCAL`数据和代码在同一个服务器机架。在同一个机器上的不同服务器中，需要通过网络传输，典型的场景需要通过一个简单的网关。
- ANY 数据在别的网络上，不在一个机架上

Spark倾向于所有的任务都使用最好的本地化级别，但是不是总能实现的。比如在空闲的executor上没有未处理的数据，Spark会选择降低本地化级别。此时有两个选择:a) 等待一个和数据在同一服务上的运行中的CPU空闲后，处理这份数据。b）立马在别的服务上（有空闲的CPU）运行一个任务并将数据传输过去。

Spark通常会稍微等待一下，希望繁忙的CPU能空闲下来。如果超过等待时间，将会将数据传输到空闲的CPU的服务上，并开启任务。等待的超时时间可以对每个本地化级别单独设置活通过一个参数设置，可以查看[配置文档](https://spark.apache.org/docs/latest/configuration.html#scheduling)上的`spark.locality`说明。如果您的任务时间较长且本地化较差(Driver日志中查看),您应该增加这些设置，但默认设置通常效果良好。


# 总结

以上内容为简单的调整Spark应用指南，最重要的调整应当是数据序列化和内存调整。对于大多数程序，选择使用`Kyro`序列化和持久化数据会解决大多数性能问题。

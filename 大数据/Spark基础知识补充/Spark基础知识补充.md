# StorageLevel等级
RDD数据保存时，persist的StorageLevel各个等级的定义以及说明

## StorageLevel类定义
``` scala
class StorageLevel private(
      private var _useDisk: Boolean,
      private var _useMemory: Boolean,
      private var _useOffHeap: Boolean,
      private var _deserialized: Boolean,
      private var _replication: Int = 1)
```
## StorageLevel各等级说明
|模式|存储等级|描述|
|-|-|-|
|`NONE`|StorageLevel(false,  false, false, false)|在内存和磁盘中都不进行存储|
|`DISK_ONLY`|StorageLevel(true,  false, false, false)|只使用磁盘对RDD数据进行存储，效率低|
|`DISK_ONLY_2`|StorageLevel(true,  false, false, false, 2)|备份到两个节点的磁盘中，效率低|
|`MEMORY_ONLY`|StorageLevel(false,  true, false, true)|默认选项，RDD数据直接以java对象的形式存储于JVM的内存中，如果内存空间不足，某些分区数据将不会被缓存，需要在使用的时候根据血统信息重新计算，效率低|
|`MEMORY_ONLY_2`|StorageLevel(false,  true, false, true, 2)|备份到两个节点的内存中，效率低|
|`MEMORY_ONLY_SER`|StorageLevel(false,  true, false, false)|RDD数据序列化之后存储于jvm内存中，但是序列化之后能够有效节约内存，效率高|
|`MEMORY_ONLY_SER_2`|StorageLevel(false,  true, false, false, 2)|备份到两个节点中，效率高|
|`MEMORY_AND_DISK`|StorageLevel(true,  true, false, true)|RDD数据以java对象的形式存储在jvm内存中，如果内存不足，部分数据将会被存储至磁盘中，使用的时候从磁盘中读取|
|`MEMORY_AND_DISK_2`|StorageLevel(true,  true, false, true, 2)|备份到两个节点的内存、磁盘中|
|`MEMORY_AND_DISK_SER`|StorageLevel(true,  true, false, false)|序列化保存数据，提升存储效率，在内存、磁盘中，效率高|
|`MEMORY_AND_DISK_SER_2`|StorageLevel(true,  true, false, false, 2)|备份到两个节点中，在内存、磁盘中，效率高|
|`OFF_HEAPS`|StorageLevel(true,  true, true, false, 1)|内存、磁盘次高使用堆外内存，能够减少垃圾回收的cpu开销（目前使用alluxio以内存为中心的分布式存储系统）|


# RDD的transformation操作
## 1.map
对RDD中的每一个元素都执行一个指定的函数来产生一个新的RDD， RDD之间的元素关系是一对一的关系

``` scala
val list = List(("sun", 20), ("sun", 20), ("earth", 40), ("earth", 40), ("moon", 50))
val myRDD = sparkContext.parallelize(list)
//每个元素的第二个数据执行+1操作
myRDD.map(x=>(x._1,x._2+1)).foreach(println)
(sun,21)
(sun,21)
(earth,41)
(earth,41)
(moon,51)
```
## 2.mapValues操作
对键值对数据的值进行操作，这个比较容易理解
``` scala
val list=List(("sun",20),("earth",40),("moon",50))
val mapValuesRDD=sparkContext.parallelize(list).mapValues(_+2)
mapValuesRDD.foreach(println)
(sun,22)
(earth,42)
(moon,52)
```
### 3.filter
对RDD的元素进行过滤，返回一个新的数据集，由经过传入函数处理后返回值为true的元素组成返回结果
//获取数据集中的第一个数据为sun的元素
``` scala
myRDD.filter(_._1.equals("sun")).foreach(println)
(sun,20)
(sun,20)
```
## 4.flatMap
每输入一个元素，会被映射为0到多个输出元素
``` scala
//给每一个元素增加1或者n个关系
myRDD.flatMap(x=>Seq(x,"flat something")).foreach(println)
(sun,20)
flat something
(sun,20)
flat something
(earth,40)
flat something
(earth,40)
flat something
(moon,50)
flat something
```
## 5.flatMapValues操作
该操作相当于是先进行了map操作，然后再进行flatten操作，返回值是Seq类型，RDD之间的元素是一对多的关系
``` scala
myRDD.flatMapValues(x=>Seq(x,"female")).foreach(println)
(sun,20)
(sun,test)
(sun,20)
(sun,test)
(earth,40)
(earth,test)
(earth,40)
(earth,test)
(moon,50)
(moon,test)
```
## 6.mapPartitions
是map函数的一个改变形式，map函数的输入是作用于每一个元素，而mapPartitions作用于每一个分区中的数据，把一个分区中的数据当成一个小整体，在并行计算中效率较高，如map函数在使用时需要为每一个元素创建一个连接对象，而mapPartitions只需要为每一个分区创建连接对象即可。
``` scala
//每个分区中的内容将以Iterator[T]的形式传递给它，返回为Iterator[C]
def myPartitionFunc(iterator: Iterator[(String,Int)]):Iterator[(String,Int)]={
var sun = List[(String,Int)]()
while (iterator.hasNext){
val next = iterator.next()
    next match {
case ("sun",20) => sun =(next) :: sun
case _=>
   }
}
sun.iterator
}
myRDD.mapPartitions(myPartitionFunc).foreach(println)
(sun,20)
(sun,20)
```
## 7.mappartitionsWithIndex
``` scala
//添加一个分区索引,由于我只有一个分区，所以索引值只有一个
def myPartitionFuncIndex(i:Int,iterator: Iterator[(String,Int)]):Iterator[(String,Int)]={
var sun = List[(String,Int)]()
while (iterator.hasNext){
val next = iterator.next()
  next match {
   case ("sun",20) => sun =(next._1+i,next._2) :: sun
   case _=>
 }
}
   sun.iterator
}
myRDD.mapPartitionsWithIndex(myPartitionFuncIndex).foreach(println)

(sun0,20)
(sun0,20)
```
### 8.sample
根据随机种子seed，随机抽样出比例为fraction的样本，withReplacement设置为true样本抽出后放回，false则不放回。
相同的随机种子得到的随机序列是一样的
```
myRDD.sample(false,0.2,1).foreach(println)

(moon,50)
```
## 9.union
将数据合并，返回一个新的数据集
``` scala
val people = List(("middle", "Mobin"), ("male", "Kpop"), ("female", "Lucy"), ("male", "Lufei"), ("female", "Amy"))
val myRDD2 = sparkContext.parallelize(people)
//在前一个rdd的尾端接上后一个rdd
myRDD2.union(myRDD.mapValues(_.toString)).foreach(println)
```
## 10.intersection
返回两个RDD的交集
```
//获取两个数据集的交集
myRDD2.intersection(myRDD.mapValues(_.toString)).foreach(println)
```
## 11.distinct
对元素进行去重，可以在入参中设置任务数量，增加并行运行数，提高效率
```
//对数据集中的元素去重
myRDD.distinct(5).foreach(println)

(moon,50)
(earth,40)
(sun,20)
```
## 12.combineByKey操作
该函数用途为，将[k,v]转成[k,c]
``` scala
val people = List(("middle", "Mobin"), ("male", "Kpop"), ("female", "Lucy"), ("male", "Lufei"), ("female", "Amy"))
val myRDD2 = sparkContext.parallelize(people)
myRDD2.combineByKey(
//createCombiner操作，将V类型转成C类型
(x: String) => (List(x), 1),
//mergeValue操作，将（V,C）转换成C，其中::作用是往peo的list中插入x，组成一个新的列表，每个分区内进行
(peo: (List[String], Int), x: String) => (x :: peo._1, peo._2 + 1),
//mergeCombiners将（C,C）转换成C,这个操作在不同分区间进行，前期因为没看到这个调了半天以为这个函数不起作用
(sex1: (List[String], Int), sex2: (List[String], Int)) => (sex1._1:::sex2._1, sex1._2 + sex2._2)).foreach(println)

(male,(List(Lufei, Kpop),2))
(female,(List(Amy, Lucy),2))
(middle,(List(Mobin),1))
```
## 13.foldByKey操作
自行编写函数按key将数据进行聚合
``` scala
val list = List(("sun", 20), ("sun", 20),("earth", 40),("earth", 40), ("moon", 50))
val myRDD = sparkContext.parallelize(list)
//zeroValue 累加的起始值
//numPartitions
分区数量
//内部实现是依赖combineByKey实现的
myRDD.foldByKey(2,1)(_+_).foreach(println)

(earth,82)
(sun,42)
(moon,52)
```
## 14.reduceByKey操作

使用聚合函数聚合相同键值对应的value
``` scala
//可以输入一个聚合函数来处理value，同时设置分区数
//内部实现是依赖combineByKey实现的
myRDD.reduceByKey(_+_,1).foreach(println)

(earth,80)
(moon,50)
(sun,40)
```
## 15.aggregateByKey
输入的是一个RDD[K,V],返回的是RDD[K,U]，U和V可以相同也可以不同
``` scala
//可以设置一个初始值，设置分区数目myRDD.aggregateByKey(List(("wcf",2)),5)(
//seqOp: (U, V) => U 因为初始值是List，因此设置一个U为List，将V转成List
(x:List[(String,Int)],y:Int)=>(("hello",y)::x),
//combOp: (U, U) => U  将两个分区中的U合并成一个U
(x1:List[(String,Int)],x2:List[(String,Int)])=>(x1)).foreach(println)
```
## 16.treeAggregate
采用多层树模式进行聚合操作，输入的初始值和输出值类型需要一致
``` scala
val test= myRDD.treeAggregate(("wcf", 1))(
//seqOp: (U, C) => U 这里U和C的类型需要一致的
(u, t) => (u._1 + t._1, u._2 + t._2),
//combOp: (U, U) => U  将两个分区中的U合并成一个U
(u1, u2) => (u1._1+u2._1 , u1._2 + u2._2),3).toString()
println(test)
```
## 17.groupByKey操作
在一个由键值对（K,V）组成的数据集上使用，返回（K，Seq[V]）
``` scala
//可以设置分区数或者使用自定义的分区函数
//但是它的入参中不包含自定义函数，实际处理性能较低，也会造成网络传输消耗
//当机子少或者数据量少的时候使用较为便捷
//内部实现是依赖combineByKey实现的
myRDD.groupByKey(2).foreach(println)

(earth,CompactBuffer(40, 40))
(sun,CompactBuffer(20, 20))
(moon,CompactBuffer(50))
```

## 18.sortByKey操作
按照键值进行排序
``` scala
//可设置排序方式和分区数目，true和不填都是升序，false为降序
myRDD.sortByKey(true, 1)

(earth,40)
(earth,40)
(moon,50)
(sun,20)
(sun,20)
```
## 19.cogroup操作
``` scala
//假设前一个rdd1[k,v],后一个rdd2[k,c]
//分别对rdd1，rdd2进行聚合，返回(k,(v的CompactBuffer,c的CompactBuffer))
//同时设置分区数
myRDD.cogroup(myRDD2,2).foreach(println)

(earth,(CompactBuffer(40, 40),CompactBuffer()))
(male,(CompactBuffer(),CompactBuffer(Kpop, Lufei)))
(moon,(CompactBuffer(50),CompactBuffer()))
(female,(CompactBuffer(),CompactBuffer(Lucy, Amy)))
(sun,(CompactBuffer(20, 20),CompactBuffer()))
(middle,(CompactBuffer(),CompactBuffer(Mobin)))
```
## 20.Join操作
原理：对两个rdd先进行cogroup操作形成新的RDD，再对每个Key下的元素求笛卡尔积，同时可以在操作的同时指定分区数目，提高作业的并行度。
Join操作类似于sql的inner join，返回结果是前面和后面集合中共同拥有的键的选项
LeftJoin返回结果和数据库中的leftjoin操作类似，以前者为基准返回相关数据，若不存在则返回None
rightJoin返回结果和数据库中的rightjoin操作类似，以前者为基准，返回相关数据，若不存在，则返回None
``` scala
val rdd1=sparkContext.makeRDD(Array(("1","Spark"),("2","Hadoop"),("3","Scala"),("4","Java")),2)
val rdd2=sparkContext.makeRDD(Array(("1","30K"),("5","15K"),("3","25K"),("4","10K")),2)
println("join performance")
val rddJoin=rdd1.join(rdd2,1).collect().foreach(println)

(4,(Java,10K))
(3,(Scala,25K))
(1,(Spark,30K))
println("leftjoin performance")
val rddLeft=rdd1.leftOuterJoin(rdd2).collect().foreach(println)

(4,(Java,Some(10K)))
(2,(Hadoop,None))
(3,(Scala,Some(25K)))
(1,(Spark,Some(30K)))

println("rightjoin performance"val rddRight=rdd1.rightOuterJoin(rdd2).collect().foreach(println)

(4,(Some(Java),10K))
(5,(None,15K))
(3,(Some(Scala),25K))
(1,(Some(Spark),30K))
```
## 21.cartesian
求两个RDD的笛卡尔积，这个只要懂笛卡尔积原理就能应用
``` scala
myRDD.cartesian(myRDD2).foreach(println)
```
## 22.pipe
执行linux脚本语句
``` scala
//执行已有的脚本文件或者执行命令行操作
myRDD.pipe("/wcf/spark/test.sh").foreach(println)
```
## 23.randomSplit
划分RDD元素
``` scala
//对RDD按照权重对数据进行划分，第一个参数为分割权重的数组，第二个参数为随机种子myRDD.randomSplit(Array(0.2,0.8),2)(1).foreach(println)
```
## 24.substract
对存在的RDD进行减法操作，减去两者共有的元素
``` scala
val rdd1=sparkContext.parallelize(1 to 10,1)
val rdd2=sparkContext.parallelize(2 to 6,1)
rdd1.subtract(rdd2,1).foreach(println)

1
7
8
9
10
```
## 25.zip

对两个RDD进行拉链操作，两个rdd的元素个数必须相同
``` scala
val rdd3=sparkContext.parallelize(1 to 5)
val rdd4=sparkContext.parallelize(2 to 6)
rdd3.zip(rdd4).foreach(println)
```
## 26.coalesce和repartition
Repartition调用coalesce函数，将shuffle置成true
``` scala
//默认不进行混洗
myRDD.coalesce(2)
//进行混洗
myRDD.repartition(2)
```
## 27.glom
将RDD的T类型转换成Array[T]
``` scala
myRDD.glom().foreach(rdd=>println(rdd.getClass.getSimpleName()))
```
初始类型Tuple2，输出类型Tuple2[]

# Implicit用法
Scala编译器在运行时发现对象类型不匹配会去寻找能匹配相关类型的implicit声明的方法
``` scala
def main(args: Array[String]): Unit = {
  display(1)
}
def display(inp: String): Unit = {
  println()
}
//如果没有这个implicit声明的转换，display(1)会报错
implicit def convertor(inp: Int): String = inp.toString
``` 
# 使用implicit定义的Object
``` scala
//这里我们定义一个隐式转换的class，其中包含一个方法
object AppleUtils {
implicit class get(s:String){
def getApple=new Apple(s)
  }
}
case class Apple(name:String)
//理论上我们是不能通过String调用getApple的方法的
//但是我们通过语句import TestCla.AppleUtils._使得String转成了Apple对象
def main(args: Array[String]): Unit = {
import TestCla.AppleUtils._
val apple:Apple="".getApple
}
```
注意：当有多个隐式转换存在时，需要在使用它之前import，而不能在一开始就import，否则会引起多个隐式转换产生冲突。
# 广播变量的用法
Broadcast只是将变量广播到各个节点，而不是广播到每一个task，因为task属于线程，他们可以共享同一个进程的变量，所以在每个节点广播一份就行了。
``` scala
//对变量进行广播
 val bb=sparkContext.broadcast(myRDD)
//获取广播中的变量
 val vv=bb.value
//删除各个节点的广播变量副本
 bb.unpersist()
```
# RDD Action操作

## 1.reduce

按照自定义聚集函数执行操作，返回聚集结果
``` scala
val list = List(("sun", 20), ("sun", 20), ("earth", 40), ("earth", 40), ("moon", 50))
val myRDD = sparkContext.parallelize(list)
println(myRDD.reduce((x,y)=>(x._1+y._1,x._2+y._2)))

(sunsunearthearthmoon,170)
```
## 2.collect
将RDD[T]数据集中的数据以Array[T]的形式返回
## 3.collectAsMap
将RDD[T]数据集中的数据以Map[K,V]的形式返回
## 4.count
返回数据集中的元素个数
## 5.countByKey
返回Map[K,long]类型的数据
## 6.first
返回数据集中的第一个数
## 7.take
返回数据集中的前n个数据，貌似当前这个操作还不能并行
## 8.takeSample
第一个参数是抽样后是否放回，第二个参数精确指定抽样个数而不是比例，第三个参数是随机数种子，返回Array[]
``` scala
myRDD.takeSample(true,5,1)
```
## 9.takeOrdered
返回排序好的n个元素，默认升序
``` scala
val rdd1=sparkContext.parallelize(List(1,5,2,5,3,5))
rdd1.takeOrdered(3).foreach(println)

1
2
3
```
### 10.saveAsTextFile
把数据集中的数据写入到文本文件中，spark会对每个元素调用toString方法，并且每个元素占用一行
### 11.foreach
对数据集中的每个元素都执行foreach中输入的功能函数
### 12.top
返回排序好的n个元素，默认倒序
``` scala
rdd1.top(4).foreach(println)

5
5
5
3
```
### 13.lookup
传入key值，返回key值对应的所有value值
``` scala
val list = List(("sun", 20), ("sun", 20), ("earth", 40), ("earth", 40), ("moon", 50))
val myRDD = sparkContext.parallelize(list)
myRDD.lookup("sun").foreach(println)

20
20
```
### 14.aggregate
首先分别计算每个分区的结果：初始值+分区聚合值，最后再计算总的聚合结果：初始值+各个分区的结果
``` scala
val result = myRDD.aggregate(("wcf", "0"))(//y的类型与rdd的元素类型相同
(x, y)=> (x._1 + y._1, x._2 + y._2.toString),(a, b) => (a._1 + b._1, a._2 + b._2)).toString()
println(result)

(wcfwcfsunearthwcfearthmoonwcfsun,00204004050020)
```
### 15.fold
初始值类型和RDD元素的类型需要是相同的，然后返回多个元素聚合成一个元素，感觉上像是aggregate的简写版本。
``` scala
val result1 = myRDD.fold(("wcf", 0))((x, y)=> (x._1 + y._1, x._2 + y._2)).toString()

println(result1)
```
### 16.saveAsSequenceFile

将数据集保存成sequence的格式在HDFS中

## DataFrame使用方法

（1）连接pg数据库，获取表中的数据

需要配置数据库url、数据库驱动、用户名、密码、数据库表名

连接完毕后可以使用show显示表中的数据，会在命令窗口中显示

val sparkConf = new SparkConf().setAppName("wcf").setMaster("local")

val sparkContext = new SparkContext(sparkConf)

val sparkSession = SparkSession.builder().config(sparkConf).getOrCreate()

val url = "jdbc:postgresql://10.10.65.24:5432/gidc_db"

//加载两张表的数据
``` scala
val jdbcDF1 =sparkSession.read.format("jdbc").options(
   Map("url" -> url,
           "driver" -> "org.postgresql.Driver",
           "user" -> "postgres",
           "password" -> "88075998",
           "dbtable" -> "face_statistics_data")).load()
val jdbcDF2 =sparkSession.read.format("jdbc").options(
     Map("url" -> url,
            "driver" -> "org.postgresql.Driver",
            "user" -> "postgres",
            "password" -> "88075998",
            "dbtable" -> "history_passengerflow_data")).load()
jdbcDF1.show(10)
jdbcDF2.show(10)
//还有一些相关显示的设置
//设置是否缩略显示，默认为缩略显示，只显示20个字符jdbcDF1.show(false)
//同时设置显示行数与是否缩略jdbcDF1.show(1,false)
``` 
（2）获取表中的数据转成List和Array
``` scala
//返回表中的所有数据，包含了它的数据类型，结果类型为Array
val array= jdbcDF2.collect()
//返回类型为List
val list=jdbcDF2.collectAsList()
``` 
（3）数值化统计
Describe中填写的都是表中字段的名称，如果在表中不存在会提示异常
``` scala
//对表中各个字段中的信息进行统计，一般用于数值统计，比如有多少条数据，最小值，最大值等
dataFrame2 .describe("id" , "object_type", "camera_index_code" ).show()
``` 
（4）获取数据
获取数据的时候需要注意数据的大小，方式内存溢出
``` scala
//获取第一行数据，返回的数据格式为：GenericRowWithSchema
val first=dataFrame2.first()
//感觉下面两个方法的使用基本一致，都是获取前n行数据
val take=dataFrame2.take(2)
val header=dataFrame2.head(2)
//获取前n行数据，并返回成list形式
val takeList=dataFrame2.takeAsList(2)
```
（4）数据检索操作
``` scala
//可以直接在一个where中输入所有条件，中间用and或者or连接，and也可以直接使用多个where连接
//返回类型为DataSet
val where = dataFrame2.where("id=1 or id=2").where("camera_index_code=001190").show()
//和上面的where使用方式怎么看怎么一样
val filter = dataFrame2.filter("id=1").show()
//获取指定列的相关数据
val select = dataFrame2.select("id", "camera_index_code").show()
//还可以对列进行数值操作，如果是字符操作的话，会改变列名，因为没有列名而获取不到数据
val select2 = dataFrame2.select(dataFrame2("id") + 1, dataFrame2("camera_index_code")).show()
//可以对各个字段指定别名，或者调用UDF函数处理信息
val selectExpr = dataFrame2.selectExpr("id as index", "round(id,3)").show()
val column = dataFrame2.col("id")
val column2 = dataFrame2.apply("id")
//除去id字段信息
val drop = dataFrame2.drop("id").show()
//limit操作不是action操作，和前面的head和take操作不一样
val limit = dataFrame2.limit(3).show()
//按照指定字段进行排序，加上负号为倒序
dataFrame2.sort("id").show()
dataFrame2.orderBy(-dataFrame2("id")).show()
//按照分区进行排序
dataFrame2.sortWithinPartitions(-dataFrame2("id")).show()
//返回值类型为RelationalGroupedDataset，不能直接使用
//一般groupby都和sum/max/min/mean/count合并使用，先分组再累加相同分组的值
val groupBy = dataFrame2.groupBy("camera_index_code").sum().show()
val groupBy2 = dataFrame2.groupBy(dataFrame2("camera_index_code"))
//貌似是建立一个数据立方
dataFrame2.cube("id").count().orderBy("id").show()
//使用当前DataFrame给定的cols创建一个多维的rollup，这样我们可以在其上进行聚合
dataFrame2.rollup("id")
```
（5）去除重复数据以及合并操作
``` scala
//返回一个不包含重复记录的Dataset
dataFrame2.distinct()
//按照指定字段去除重复内容
dataFrame2.dropDuplicates("camera_index_code").show()
//复杂聚合操作,一般和groupBy合并使用
dataFrame2.agg("id"->"max","camera_index_code"->"sum")
//组合操作
dataFrame2.union(dataFrame2.limit(1))
```
（6）Join操作
``` scala
//连接两张表，通过id字段，或者通过多个字段,指定是内连接还是外链接
dataFrame1.join(dataFrame2,"id").show(1)
dataFrame1.join(dataFrame2,Seq("id","camera_index_code"),"inner").show(1)
```
（7）其他操作
``` scala
//获取两张表都同时具有的
dataFrame1.intersect(dataFrame2.limit(1)).show()
//获取其中一张表具有而另一张表没有的
dataFrame1.except(dataFrame2.limit(1)).show()
//重命名字段
dataFrame1.withColumnRenamed("id","wcf")
//新增一列，若存在，则覆盖
dataFrame1.withColumn("id",dataFrame1("id")).show()
```
## RDD有两种操作方式分为Transformation和Action
转换操作时延迟计算，可以理解为，该操作并不实际执行，只是记录下该操作。执行操作触发转换操作，真正进行执行。
怎么区分是action操作还是transformation操作
通过查看源码，可以发现action操作都有一个共同的特征，即它都调用了一个runJob()的方法，而transformation操作都没有调用该方法。
原创文章转载请标明出处
更多文章请查看 
[http://www.canfeng.xyz](http://www.canfeng.xyz)
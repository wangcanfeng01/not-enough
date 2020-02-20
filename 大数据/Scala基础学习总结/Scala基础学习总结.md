# 引言：
Scala是一门多范式的编程语言，运行在java虚拟机上，并兼容现有的java程序，它运行在JVM上，并可以调用现有的java类库。Scala特性：面向对象特性、函数式编程、静态类型、扩展性、并发性。使用Scala必须要先安装java（JDK版本1.5以后）。
# Scala访问修饰符：
Scala访问修饰符基本和java一致，有private、protected、public，如果没有指定修饰符，默认为public，Scala外层类不能访问内部类的私有变量。
# Scala运算符：
Scala运算符主要有算术运算符、关系运算符、逻辑运算符、位运算符、赋值运算符，基本运算符和java一致。
::作用

X::List(x)相当于把x往list里面注入

:::作用

List1:::List2连接两个集合

:+作用

在尾部追加元素

+:作用

在头部追加元素
# Scala条件语句和循环语句：

与java一致

# Scala函数：

（1）函数声明：def functionName(参数)：返回类型

（2）函数定义：def functionName(参数)：返回类型{ 函数功能体，返回值}

（3）函数调用和java类似。

# Scala闭包：

闭包是一个函数，返回值依赖于声明在函数外部的一个或多个变量。闭包通常来讲是可以访问一个函数里面局部变量的另外一个函数。

``` scala
object Test { 
def main(args: Array[String]) { 
println( "muliplier(1) value = " +  multiplier(1) ) 
println( "muliplier(2) value = " +  multiplier(2) ) 
} 
var factor = 3 
val multiplier = (i:Int) => i * factor  //这个计算值就是返回值
}
```

# Scala字符串：

用法与java相似

# Scala数组：

（1）数组声明

``` scala
var z:Array[String] = new Array[String](3)
```
或

``` scala
var z = new Array[String](3)
```

或

``` scala
var z = Array("Runoob",  "Baidu", "Google")
```
索引访问格式使用：  数组名（index）
（2）多维数组
``` scala
var myMatrix = Array.ofDim[Int](3,3)
```
（3）合并数组

``` scala
var  myList3 =  concat( myList1, myList2)
```
（4）创建区间数组

``` scala
var myList1 = range(10, 20, 2)
```
最后一个参数确定步长，不填默认为1

（5）其他方法：

可以进入Array类中查看scala数组中具有的相关方法

# Scala集合：

有List、Set、Map、元组、Option、Iterator

其中元组中可以插入不同的类型的值

``` scala
val t = new Tuple3(1, 3.14, "Fred")
```

其中Option中表示有可能包含值，有可能不包含值的容器

``` scala
    val myMap: Map[String, String] =  Map("key1" -> "value")
    val value1: Option[String] =  myMap.get("key1")
    val value2: Option[String] =  myMap.get("key2")
    println(value1) // Some("value1") 
    println(value2) // None
```
# Scala迭代器：

基本使用方法和java一致

# Scala类和对象：

（1）继承

父类中的字段如果有val修饰，子类中必须要override和val修饰，两者都不可缺少。

Scala只允许继承一个父类

（2）单例对象

Scala中是没有static这个东西的，但是它也提供了单例模式的实现方法，使用object关键字。Scala使用单例模式时，除了定义的类之外还要定义一个同名的object对象，它和类的区别是，object对象不能携带参数。当单例对象和某个类重名的时候，它被称为这个类的伴生对象，companion object。必须在同一个源文件里定义类和它的伴生对象，类和它的伴生对象可以互相访问其私有成员。

# Scala特征：

Trait相当于java的接口，也可以说是抽象类，可以多重继承

# Scala模式匹配：

模式匹配类似于java的switch，用箭头符号隔开模式和表达式。

样例类常用于优化模式匹配

``` scala
object Test {

  def main(args: Array[String]) {
  val alice = new Person("Alice", 25)
  val bob = new Person("Bob", 32)
  val charlie = new Person("Charlie", 32)
  for (person <- List(alice, bob, charlie)) {
  person match {
  case Person("Alice", 25) => println("Hi  Alice!")
  case  Person("Bob", 32) => println("Hi Bob!")
  case Person(name, age) => println("Age: " +  age + " year, name: " + name + "?")
    }
  }
}
   //样例类
  case class Person(name: String, age: Int)
}
```

# Scala正则表达式：

``` scala
def main(args: Array[String]) {
  val pattern = new Regex("(S|s)cala")  //用管道设置不同的匹配模式  首字母可以是大写 S 或小写s
  val str = "Scala is scalable and cool"
  println((pattern findAllIn str).mkString(","))   //使用逗号 , 连接返回结果
}
```
用findAllIn可以找到所有匹配项，用mkString连接返回结果，中间逗号隔开

# Scala异常处理：

异常处理和java相似，但是异常抛出的方式和捕获与java在形式上存在区别：

通过模式匹配的机制来对不同的异常进行捕获

``` scala
try {
  val f = new FileReader("input.txt")
} catch {
  case ex: FileNotFoundException =>{
  println("Missing file exception")
}
  case ex: IOException => {
  println("IO Exception")
  }
}finally {
  println("Exiting finally...")
}
```

# Scala提取器：

``` scala
def  main(args: Array[String]) {
      println ("Unapply方法: " + unapply("Zara@gmail.com"));
      println ("Unapply方法: " + unapply("Zara Ali"));
 }

   //提取方法（必选）
   def unapply(str: String): Option[(String,  String)] = {
      val parts = str split "@"
      if (parts.length == 2){
         Some(parts(0), parts(1))
      }else{
         None
      }
 }
```
编译器在实例化的时候自动使用apply ，在match中自动使用unapply

# Scala文件I/O：

``` scala
//输出流
def main(args: Array[String]) {
  val writer = new PrintWriter(new File("test.txt" ))
  writer.write("菜鸟教程")
  writer.close()
}
//屏幕输入
def main(args: Array[String]) {
  print("请输入菜鸟教程官网: " )
  val line = Console.readLine
  println("谢谢，你输入的是:  " + line)
   }
//从文件中读取
def main(args: Array[String]) {
  println("文件内容为:"  )
  Source.fromFile("test.txt" ).foreach{print}
}
```
原创文章转载请标明出处
更多文章请查看 
[http://www.canfeng.xyz](http://www.canfeng.xyz)

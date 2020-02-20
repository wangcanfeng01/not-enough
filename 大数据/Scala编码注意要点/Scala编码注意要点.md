## 1.val 和var的使用
应该尽量使用val，即在定义一个不可重新赋值的对象。
## 2.eq 和equals的区别
==方法和！=不能重写，==本质上等同于equals
eq判断的是值引用是否相等，equals判断的是它的值是否相等，这个跟java存在区别
## 3.scala会将非字段或方法定义的代码编译进类的主构造方法中
## 4.require的使用
可以使用require(boolean)设置主构造器的前置条件,判断传入的值是否合法，不合法则抛出异常
## 5.scala遵循的是java的驼峰命名法
## 6.柯里化
def(x,y)可以写成def(x)(y)
## 7.控制抽象
往函数接口里传递函数减少代码
## 8.特质不能有类的参数
  Class MyQueue extends BasicIntQueue with Doubling 没有定义新的代码，在实例化的时候可以直接给出 new BasicIntQueue with Doubling
  混入多个特质时，最先调用的是最右侧的特质中的方法，如果那个方法调用了super，依次往左调用。·	
## 9.命名空间
java存在4个命名空间：字段、方法、类型、包；scala的两个命名空间：值（字段、方法、包和单例对象），类型（类和特质名）
## 10.zip
zip操作符在处理两个数组长度不同的情况是，会扔掉较长一个数组的中部分元素
## 11.for
for()循环中通过yield交出结果
## 12.import
import Fruits.{Apple=>BigApple,Orangle} 引入时重命名类名，类名相同需要连带包名引入
## 13.排除引入
隐藏子句x=>_,这会从引入名称集里排除掉x。import Fruits.{Apple=>_,_},引入所有，除了Apple，排除项必须要放在前面。
## 14.list匹配
list匹配未知元素个数用var x::b::rest=list对象
## 15.模式匹配
（y:@unchecked）禁用警告
## 16.列表增加元素
列表增加元素需要考虑列表长度
## 17.私有构造函数
calss Queue[T] private(private val a,private val b)，类名和参数之间的private表示构造器方法是私有的
## 18.泛型上下界
scala编译器会对类或特质定义中的所有能出现类型参数的点归为协变的、逆变的、和不变的。用+注解的只能用在协变点，用-注解的只能用在逆变点，没有注解的可以用在任何能出现类型参数的点，因此也是唯一的一种能用在不变点的类型参数。<:表示的是定义的泛型上界，如T<:Animal表示的是T类型必须是Animal的子类或者本身
## 19.懒加载
使用lazy标示字段，只有字段在被调用时才会初始化；
## 20.隐式标记规则
只有标记为implict的定义才可用，关键字implict用来标记任何变量、函数或对象定义。隐式转换必须在当前作用域内，转换时采用显式转换优先的原则，参数列表也可以提供隐式转换。
## 21.隐式转换选择原则
选择更为具体的方式：（1）前者的入参类型是后者入参类型的子类型；（2）两者都是方法，而前者所在的类扩展自后者所在的类。
## 22.隐式 
def implicty[T]（implicty t:T）=t;调用implicty[Foo]的作用是编译器会查找一个类型为Foo的隐式定义。
## 23.右结合
中缀操作（：：，：：：）是右结合的。
## 24.scala列表有三个基本操作
（1）head返回 第一个元素（2）tail返回一个列表，除了第一个元素；（3）isEmpty在列表为空时返回true。
## 25.for其他表示方法
每个for表达式都可以用三个高阶函数map，flatMap，withFilter来表示。
## 26.seq的update和updated区别
后者是返回新的序列，而不是修改原序列
## 27.链式与普通集合区别
链式哈希映射或哈希集跟常规的哈希区别在于链式的版本额外包含了一个按照元素添加顺序保存的元素链表。对于这样集合的遍历总是按照元素添加顺序来进行的。
## 28.系统会优先选择predef中定义的隐式转换。
## 29.减少中间计算结果的方式
通过转换成视图的方式，如：（v.view map(_+1) map(_*2)）.force，但这种操作只支持一般化的集合类型。
## 30.scala和java集合互转
需要引入import collection.JavaConversions._
双向互转包括 
Iterator-java.util.Iterator 
Iterator-java.util.Enumeration
Iterable—java.lang.Iterable
Iterable-java.util.collection
mutable.Buffer-java.util.List
mutable.set-java.util.set
mutable.Map-java.util.Map.
## 31.常用注解
 @serializable+@SerialVersionUID(232131)注解序列化类，@scala.reflect.BeanProperty注解自动生成get和set方法。
## 32.在方法上添加@throws（classof[IOException]）
## 33.scala中future和java的future并不一致。
原创文章转载请标明出处
更多文章请查看 
[http://www.canfeng.xyz](http://www.canfeng.xyz)
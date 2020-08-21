---
layout:     post
title:      Kotlin学习
subtitle:   Kotlin笔记
date:       2020-08-08
author:     HYC
header-img: img/post-2020-05-05-header.jpg
catalog: true
tags:
    - Android
    - Kotlin

---
## Kotlin 笔记
### 1. 编译运行Kotlin
创建一个kotlin文件，并命名为hello.kt
``` kotlin
fun main() {
    println("Hello, Kotlin!")
}
```
编译源码：
``` shell
> kotlinc-jvm hello.kt

> ls
hello.kt HelloKt.class
```
这时我们可以看到一个.class文件，然后我们可以让JVM执行这个文件：
``` shell
> kotlin HelloKt
Hello, Kotlin!

Kotlin does not generate Java source code —it's not a transpiler. It generates bytecodes that can be interpreted by the JVM.
```
也可以将kotlin代码生成为可以被Java命令执行的JAR文件：
``` shell
> kotlinc-jvm hello.kt -include-runtime -d hello.jar
```
然后使用java命令执行这个JAR文件：
``` shell
> java -jar hello.jar
Hello, Kotlin!
```
### 2. Kotlin REPL
使用kotlinc命令，进入kotlin REPL模式：
``` shell
    > kotlinc
    Welcome to Kotlin version 1.3.72 (JRE 1.8.0_251-b08)
    Type :help for help, :quit for quit
    >>> println("Hello world!")
    Hello world!
    >>> var name = "Dolly"
    >>> println("Hello, $name!")
    Hello, Dolly!
    >>> :help
    Available commands:
    :help                   show this help
    :quit                   exit the interpreter
    :dump bytecode          dump classes to terminal
    :load <file>            load script from specified file
    >>> :quit
```

### 3.  基础Kotlin
#### 3.1 Using Nullable Types in Kotlin
> Ensure that a variable is never null

Kotlin的一大特色就是它消除了所有可能的nulls。如果你在定义一个变量时，在尾部没有加上？，那么编译器会要求那个变量是非空的。
``` kotlin
var name: String

name = "Dolly"
// name = null
```
`name = null`不会通过编译，因为不能将`null`赋值给一个非空变量。

如果想让变量可以是空的，那么就在尾部加上❓。
``` kotlin
class Person(val first: String,
             val middle: String?,
             val last: String)

val jkRowling = Person("Joanne", null, "Rowling")
val northWest = Person("North", null, "West")
```
如果middle不是空的时候，Kotlin会对它做一个小的转型，将它看作是一个String而不是String?类型。
``` kotlin
val p = Person(first = "North", middle = null, last = "West")

if (p.middle != null) {
    val = middleNameLength = p.middle.length
}
```
- Safe Call Operator: `?.`
返回`null`，如果`middle`为`null`
`val middleNameLength = p.middle?.length`
- Elvis Operator: ?:
返回`0`，如果`middle`是`null`
`val middleNameLength = p.middle?.length ?: 0`
#### 3.2 `var` 和 `val`的区别
- `var`指代可变变量
- `val`指代不可变变量，只能赋值一次

常量与变量都可以没有初始化值,但是在引用前必须初始化
编译器支持自动类型判断,即声明时可以不指定类型,由编译器判断。

#### 3.3 类的继承
Kotlin中任何一个非抽象类默认都是不可以被继承的，相当于Java中使用了`final`关键字。如果想让一个类能够被继承，那么就需要加上`open`关键字。
##### 3.3.1 open 关键字
``` kotlin
open class Person {
    ...
}
```
加上了`open`关键字之后，我们就主动告诉Kotlin编译器，`Person`这个类就是专门为继承而设计的，这样`Person`类就允许被继承了。

##### 3.3.2 继承
Java中的继承关键字到`Person`里面就变成了一个冒号：
``` kotlin
class Student : Person() {
    var sno = ""
    var grade = 0
}
```
至于`Person`后面为什么要加上括号呢，就是因为构造函数的问题：

##### 3.3.3 主次构造函数和`init`关键字
- 主构造函数
将会是最常使用的构造函数，每个类默认都会有一个无参的构造函数。它的特点是没有函数体，直接定义在类名的后面即可。比如：
``` kotlin
class Student(val sno: String, val grade: Int) : Person() { 

}

// ....
// 实例化
val student = Student("a123", 5)
```
既然主构造函数中没有函数体，如果想要在构造函数中加入一点逻辑，应该怎么办呢？
``` kotlin 
class Student(val sno: String, val grade: Int) : Person() { 
    init {
        println("sno is " + sno)
        println("grade is " + grade)
    }
}
```
我们将逻辑写入`init`代码块中即可。

- 次构造函数
任何一个类只能有一个主构造函数，可以有多个次构造函数，而且所有的次构造函数都要调用主构造函数。
次构造函数是通过`constructor`关键字定义的：
``` kotlin
class Student(val sno: String, val grade: Int, name: String, age: Int) :
    Person(name, age) {
    constructor(name: String, age: Int) : this("", 0, name, age) {
    }

    constructor() : this("", 0) {
    }
}
```
特殊情况：类中只有次构造函数，没有主构造函数。这个时候就只能调用父类的构造函数，也是将`this`关键字换成了`super`关键字。

##### 3.3.4 接口的实现
假设有这样一个接口：
``` kotlin
interface Study {
    fun readBooks()
    fun doHomework()
}
```
让后让一个`Student`类去实现它：
``` kotlin
class Student(name: String, age: Int) : Person(name, age), Study {
    override fun readBooks() {
        println(name + " is reading.")
    }

    override fun doHomework() {
        println(name + " is doing homework.")
    }
}
```
在Kotlin中，继承和实现都是使用冒号表示，中间使用逗号隔开。
##### 3.3.5 数据类和单例类
用`data`关键字声明数据类：
``` kotlin
data class Cellphone(val brand: String, val price: Double)
```
当你声明`data`关键字的时候，就表明你希望这个类是一个数据类，Kotlin会根据主构造函数中的参数帮你将`equals()`、`hashCode()`、`toString()`等固定且无逻辑意义的方法自动生成，从而大大减少了开发的工作量。
``` kotlin
fun main() {
    val cellphone1 = Cellphone("Samsung", 1299.99)
    val cellphone2 = Cellphone("Samsung", 1299.99)
    println(cellphone1)
    println("cellphone1 equals cellphone2" + (cellphone1 == cellphone2))
}
```
用`object`关键字声明单例类：
``` kotlin
object Singleton {
    fun singletonTest() {
        println("singletonTest is called")
    }    
}
```
3.4 Kotlin中的可见修饰符
- `private`：和Java中一样，表示只对当前类可见。
- `public`：和Java中一样，表示对所有类可见，但是在Kotlin中，public是默认的可见属性。
- `protected`：在Kotlin中表示只对当前类和子类可见，而java中表示对当前类、子类和同一包路径下的类都可见。
- `internal`：只对同一模块中的类可见。

#### 3.5 Kotlin中的Getter和Setter
在Kotlin类中给属性添加get和set函数，kotlin中一个属性的定义语法是这样的：
``` kotlin
var <propertyName> [: <PropertyType>] [= <property_initializer>]
    [<getter>]
    [<setter>]
```
也就是说，`get`和`set`在我们定义变量的时候，就可以同时被定义：
``` kotlin
val isLowPriority 
    get() = priority < 3
```
这个例子展示了`isLowPriority`这个变量指的是当`priority`小于3是就是`true`，否则为`false`。很明显它的类型是`boolean`类型。

``` kotlin
val priority = 3
    set(value) {
        field = value.coerceIn(1..5)
    }
```    
这个setter的例子展示了在`set`中，一般我们如果需要引用原来的那个属性，就要使用`field`这个标识符。这个setter的功能是将输入的值保证是在1到5这个区间内。

#### 3.6 Kotlin的scoping functions
##### 3.6.1 `apply`关键字
`apply`函数是一个扩展函数，它接受`this`作为参数，然后返回的也是`this`。

`apply`函数的定义：
``` kotlin
inline fun <T> T.apply(block: T.() -> Unit): T
```
##### 3.6.2 also关键字
`also`也是标准库中的扩展函数，其定义如下：
``` kotlin
public inline fun <T> T.also(
    block: (T) -> Unit
): T
```
因为`also`返回的也是原来那个上下文变量，所以也可以链式调用。
``` kotlin
val book = createBook()
    .also{ println(it) }
    .also{ Logger.getAnonymousLogger().info(it.toString()) }
```    
在`also`的函数体中，传入的对象被给到一个叫`it`的引用上。

##### 3.6.3 let关键字
`let`函数也是一个对于任何泛型T定义的扩展函数，它的定义是：
``` kotlin
public inline fun<T, R> T.let(
    block: (T) -> R
): R
```
`let`的关键的一点就是它返回的是代码块中的结果，而不是上下文。
加入我们想要使一个字符串中的字符全部变成大写，我们可以这么做：
``` kotlin
fun processString(str: String) = 
    str.let {
        when {
            it.isEmpty() -> "Empty"
            it.isBlank() -> "Blank"
            else -> it.capitalize()
        }
    }
```
而且传入的参数也可以是空：
``` kotlin
fun processString(str: String) = 
    str?.let {
        when {
            it.isEmpty() -> "Empty"
            it.isBlank() -> "Blank"
            else -> it.capitalize()
        }
    } ?: "Null"
```
#### 3.7 静态的方法
在Java中创建一个静态方法就只需要加上`static`关键字就好了，但是在Kotlin中却极度弱化了静态方法这个概念，为什么Kotlin要这样设计呢？因为Kotlin提供了比静态方法更好用的语法特性，就是单例类。

如果是工具类的话，在Kotlin中就可以使用单例类的方式来实现：
object Util {
    fun doAction() {
        println("do action")
    }
}

虽然这里的doAction并不是静态的方法，但是我们也可以使用Util.doAction()调用。但是如果我们只希望类中的一个方法变成静态方法，那么应该怎么办呢？这个时候我们就可以使用companion object了：
companion object {
    fun doAction() {
        println("do action")
    }
}

其实上面定义的也不是一个静态的方法，companion object实际上会在Util类的内部创建一个伴生类。虽然kotlin没有直接定义静态方法的关键字，但是提供了一些语法特性来支持类似于静态方法调用的写法。如果我们给单例类或companion object中的方法加上@JvmStatic注解的话，Kotlin就会将这些方法编译成真正的静态方法：
companion object {
    @JvmStatic
    fun doAction() {
        println("do action")
    }
}

再来看顶层方法，指的是那些没有定义在任何类中的方法，比如main()方法。Kotlin会将所有的顶层方法全部编译为静态方法。因此只要你定义了一个顶层方法，那么它就一定是静态方法：
fun doSomething() {
    println("do something")
}

Kotlin会将放在一个类中，方便Java调用，比如类叫Helper，那么Java中就可以用HelperKT调用。

3.8 Kotlin 类架构
3.8.1 Any类型
Any其实等同于Java中的Object，而Kotlin中原始类型和用户定义的类型之间其实没有什么明显的区别。
- 如果声明了一个类但却不指定它继承自哪个类的话，那么它就会是Any类的一个直接子类：
class Fruit(val ripeness: Double)

- 如果声明类这个类的父类的话，那么它就是它的父类的直接子类，而最后的祖先类会是Any。
- 如果你的类实现了一个或者多个接口，它就会有多个父类型，而Any也会是它的最终祖先类型。

3.8.2 可空类型
和Java不同的是，Kotlin将“非空（Non-null）”和“可空（Nullable）”类型区分开。刚刚我们谈到的都是非空类型，而且Kotlin不允许将null作为那些类型的值：
var s: String = null
// Error: Null can not be a value of a non-null type String

var s: String? = null
// Yes

这些可空类型的类的架构其实和那些非空类型的架构其实一样，举个🌰 ：String是Any的直接子类，String?是Any?的直接子类。Banana是Fruit的子类，而Banana?是Fruit?的子类。

除此之外，一个非空类型也是它的可空类型的子类型，比如String是String?的子类。

3.8.3 Unit类型
因为Kotlin是一个面向表达式的语言，Kotlin没有void类型的函数，所有的函数都会有返回值，那些类似void函数的Kotlin函数都会返回一个Unit类型的值。Unit中只有一个值，也叫Unit。

大部分情况下我们不需要显式地声明一个返回Unit类型的函数的返回类型；如果你写了一个没有声明返回类型的函数，那么编译器就会自动地将它作为一个Unit函数对待。

Unit类型也是Any子类，Unit？是Any？的子类，Unit也是Unit？的子类。

Unit?是一个比较特别的例子，它只有两个成员：一个Unit值和一个null。

Unit类型的出现其实是让整个类型系统一致对待所有的函数。 

3.8.4 Nothing类型
在Kotlin类架构的最底部是Nothing类型，它并没有一个实例，一个返回Nothing类型的表达式不会产生任何值。也就是说在一个返回Nothing表达式之后的代码是unreachable。因此返回Nothing的函数一般是用于捕获异常。


4. Kotlin特性
4.1 Kotlin的协程
4.1.1 runBlocking
runBlocking函数会创建一个协程的作用域，它可以保证在协程作用域内的所有代码和子协程没有全部执行玩之前一直阻塞当前线程。需要注意的是，runBlocking函数通常只应该在测试环境下使用，在正式的环境中使用容易产生一些性能上的问题：
fun main() {
    runBlocking {
        println("codes run in coroutine scope")
        delay(1500)
        println("codes run in coroutine scope finished")
    }
    Thread.sleep(1000)
}

那么如何才能创建多个协程呢？很简单，使用launch函数就行了：
fun main() {
    runBlocking {
        launch {
            println("launch1")
            delay(1000)
            println("launch1 finished")
        }

        launch {
            println("launch2")
            delay(1000)
            println("launch2 finished")
        }
    }
    Thread.sleep(1000)
}

4.2 密封类
密封类用来表示受限的类继承结构：当一个值为有限几种的类型、而不能有任何其他类型时。在某种意义上，他们是枚举类的扩展：枚举类型的值集合也是受限的，但每个枚举常量只存在一个实例，而密封类的一个子类可以有可包含状态的多个实例。

fun eval(expr: Expr): Int =
        when (expr) {
            is Num -> expr.value
            is Sum -> eval(expr.left) + eval(expr.right)
        }

sealed class Expr
class Num(val value: Int) : Expr()
class Sum(val left: Expr, val right: Expr) : Expr()

一个密封类是自身抽象的，它不能直接实例化并可以有抽象（abstract）成员。密封类不允许有非-private 构造函数（其构造函数默认为 private）。

4.3 内联函数
- inline关键字：将函数代码拷贝到调用的地方。如果不使用inline而且当参数是lambda表达式时，多次调用会引起比较大的性能开销，因为要经常生成新的lambda表达式对象。相反，如果是inline，lambda也会被拷贝到调用的地方。
- noninline关键字：当在一个inline函数中，有多个lambda作为参数时，可以在不想内联的lambda表达式前加上这个声明。
- crossinline关键字：显示声明inline函数的形参lambda不能有return语句，避免lambda中的return影响外部程序流程。

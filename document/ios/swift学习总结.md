###  1. 读后概述

阅读完[《the swift programming language 中文版》](https://swiftgg.gitbook.io/swift/)，对swift语言有了初步的认识。在特点是：swift既是项目开发的编程语言，又有脚本语言的特点；在学习上：swift可以在xcode的playground进行编写，所见即所得，非常适合语法学习。

对于swift语言，个人认为就是"站在巨人的肩膀上"的一门语言，及众家之所长。

众所周知，编程语言都是相同的。swift和Java语言在某些语法和关键字上面甚是相似，因此在学习时困难不大。当然，对于熟悉C++的同学可以进行类比。

同时，swift是一门新生语言，大部分时代潮流技术也在其中，而且有自己独有的一些技巧在其中，非常值得学习。

### 2. 部分语法梳理

#### 2.1 说明

对不同的且重要的语法进行了较为详细的说明, 对于已知或类似的语法没有进行细致的说明.

#### 2.2 基础类型

**1）变量与常量**

swift用`let`定义常量，用`var`定义变量。对于数据类型则swift自行推断(类型推倒在C++中相当于`auto`这个类型占位符一样)。常量在设定后，将不能再更改它的值，变量则可以。

如果需要指定类型，则使用类型注解

```swift
var welcomeMessage: String
// 一般不需要写类型注解, 在声明常量或者变量的时候赋了一个初始值，Swift 可以推断出这个常量或者变量的类型
```

常量和变量名可以包含几乎所有的字符，包括 Unicode 字符：

```swift
let π = 3.14159
let 你好 = "你好世界"
let 🐶🐮 = "dogcow"
```

值得注意的是：

a）在实际项目开发中，应该先定义为`let`类型，如果需要修改时，再修改类型为`var`

b）let指的是指向的地址不可修改，但是该对象内容是可修改的(常量不可修改)

**2）元组**

swift提供了元组：多个值组合成一个复合值。

元组内的值可以是任意类型，并不要求是相同类型。有了元组，函数的返回值就可以返回多个不同类型的值。

```swift
// 元组定义
// 1. 字面量定义
var tuple1 = (1, 2.0, "string", true)
// 2. 带参数标签的定义
var tuple2 = (parameter1:int, parameter2:double parameter3:String)
tuple2 = (1, 2.0, "string")
let tuple3 = (parameter1: 1, parameter2: "string")
// 元组取值
// 1. 序列号取值
tuple.0, tuple.1, tuple.2
// 2. 标签取值
tuple2.parameter1
tuple2.parameter2
```

**3）可选类型**

让程序更安全！

在OC中，有一种写法。

```objective-c
NSString *str = [obj getSomeStr] ?: @"";
```

为了保证str不是`nil`，在赋值时确定一下。swift提供了可选类型，来处理可能缺失的情况。注意可选类型是一个类型。

```swift
var surveyAnswer: String? = "123"
// surveyAnswer 可选值为"123""
// 注意：可选类型为类型后面+？
```

构造器返回值可选值类型

```swift
let possibleNumber = "123"
let convertedNumber = Int(possibleNumber)
// 因为该构造器可能会失败，convertedNumber 被推测为类型 "Int?"， 或者类型 "optional Int"
// 构造器就是构造函数, 在C++与Java中的叫法
```

使用if或者while可以进行判断是否有值。这个过程又叫做可选绑定

```swift
// 强制解包 
let value = surveyAnswer!
// 强制解包如果没值，则会crash，因此使用强制解包要保证有值

// 可选绑定
// if语法在后面介绍
if let newstring = surveyAnswer {

}

// 隐式展开
// 在确定有值的时候，可以直接解包
var a:Int! = 10
var b:Int = a
// 疑问：已经确定有值，还用可选类型干嘛？如：
// var a:Int = 10
// 解答：用可选类型是因为可以使用nil，如果不是可选类型则不能使用nil。
```

**4）值类型与引用类型**

值类型：当其进行常量、变量赋值操作，或在函数/方法中传递时，会进行值拷贝。

引用类型：增加引用计数，不进行拷贝。

**5）其他**

5.1)类型别名: 

`swift`中不是使用，`typedef` , 且有等号。

```swift
typealias AudioSample = UInt16
```

5.2)分号:

在swift中不用使用分号, 但是在同一行的两个语句使用分号.

#### 2.3 运算符

**1）空合运算符**

如果 `a` 包含一个值就进行解包，否则就返回一个默认值 `b`

```swift
a != nil ? a! : b
// a必须是可选类型
// 
```

**2）区间运算符**

Swift 提供了几种方便表达一个区间的值的*区间运算符*。

闭区间运算符（`a...b`）定义一个包含从 `a` 到 `b`（包括 `a` 和 `b`）的所有值的区间。`a` 的值不能超过 `b`。

半开区间运算符（`a..<b`）定义一个从 `a` 到 `b` 但不包括 `b` 的区间。 

单侧区间运算符（`a...`/`...b`）可以表达往一侧无限延伸的区间

#### 2.4 字符与字符串

Swift 的 `String` 和 `Character` 类型提供了一种快速且兼容 Unicode 的方式来处理代码中的文本内容, 因此可以用任何字符。有趣的是，变量或者常量命名可以采用emoji表情或者汉字。

```swift
for character in "Dog!🐶" {
    print(character)
}
// D
// o
// g
// !
// 🐶
```

在书中介绍了String是基于 *Unicode 标量* 建立的. 因此需要了解一下Unicode中的码点, utf-8等概念. 在说到 `count`和objective-c的length返回的值不一样, 也是由于采取的编码格式不同. 

String是**值类型**，对于String的使用不再赘述。

#### 2.4 控制流

**1）if**

if的条件必须是一个**布尔表达式**——这意味着像 `if score { ... }` 这样的直接传值的代码将报错，swift不会隐形地与 0 做对比。

```swift
let individualScores = [75, 43, 103, 87, 12]
var teamScore = 0
for score in individualScores {
  // swift的if语句可以像其他编程语言的一样，格式一样
	// 可以采用之前的写法
	if (score > 50) {
    // ...
  }
	// 也可以不写括号，裸奔~
  if score > 50 {
      teamScore += 3
   } else {
       teamScore += 1
   }
}
print(teamScore)

// guard语句对比：相当于if 相反的执行
if 条件为真 {
  // 执行
}

guard 条件为真else {
  // 执行
}

// guard也可以进行可选绑定
let str : String?;
guard let value = str else {
	//解绑失败，即数据为空进入  
}

```

**2）switch**

`switch` 支持任意类型的数据（区间、元组…………）以及各种比较操作——不仅仅是整数以及测试相等。

case还支持多个条件

```swift
let vegetable = "red pepper"
switch vegetable {
case "celery":
    print("Add some raisins and make ants on a log.")
case "cucumber", "watercress":
    print("That would make a good tea sandwich.")
case let x where x.hasSuffix("pepper"):
    print("Is it a spicy \(x)?")
default:
    print("Everything tastes good in soup.")
}
```

在 Swift 中，当匹配的 case 分支中的代码执行完毕后，程序会终止 `switch` 语句，而不会继续执行下一个 case 分支。这也就是说，不需要在 case 分支中显式地使用 `break` 语句。如果需要产生case穿透，则可以在case结束后，增加`fallthrough`

#### 2.5 函数

**1）函数的定义**

```swift
// 定义格式
func funcName(parameters) -> returnType {
    function body
}
// 例子
func greet(person: String) -> String {
    let greeting = "Hello, " + person + "!"
    return greeting
}
```

这里函数的格式类似JavaScript的格式，只是多了返回值类型

**2）参数列表**

这里重点说明参数列表。

a）参数标签和参数名称：每个函数参数都有一个*参数标签（argument label）*以及一个*参数名称（parameter name）*。参数标签在调用函数的时候使用；调用的时候需要将函数的参数标签写在对应的参数前面。参数名称在函数的实现中使用。默认情况下，**参数名称来作为它们的参数标签。**

```swift
// 参数列表
func someFunction(parameterName: Int) {
    // 在函数体内，parameterName 代表参数值
}
```

b）如果你不希望为某个参数添加一个标签，可以使用一个下划线（`_`）来代替一个明确的参数标签。

```swift
func someFunction(_ firstParameterName: Int, secondParameterName: Int) {
     // 在函数体内，firstParameterName 和 secondParameterName 代表参数中的第一个和第二个参数值
}
someFunction(1, secondParameterName: 2)
```

c）当然也可以类似`C++`使用缺省参数，即就是提供默认值

```swift
func someFunction(parameterWithoutDefault: Int, parameterWithDefault: Int = 12) {
    // 如果你在调用时候不传第二个参数，parameterWithDefault 会值为 12 传入到函数体中。
}
someFunction(parameterWithoutDefault: 3, parameterWithDefault: 6) 
// parameterWithDefault = 6
someFunction(parameterWithoutDefault: 4) 
// parameterWithDefault = 12
```

d）当然你还可以用可变参数来指定函数参数可以被传入不确定数量的输入值。通过在变量类型名后面加入（`...`）的方式来定义可变参数。可变参数的传入值在函数体中变为此类型的一个数组。例如，一个叫做 `numbers` 的 `Double...` 型可变参数，在函数体内可以当做一个叫 `numbers` 的 `[Double]` 型的数组常量。

```swift
func arithmeticMean(_ numbers: Double...) -> Double {
    var total: Double = 0
    for number in numbers {
        total += number
    }
    return total / Double(numbers.count)
}
arithmeticMean(1, 2, 3, 4, 5)
// 返回 3.0, 是这 5 个数的平均数。
arithmeticMean(3, 8.25, 18.75)
// 返回 10.0, 是这 3 个数的平均数。
```

e）函数参数默认是常量。试图在函数体中更改参数值将会导致编译错误。如果你想要一个函数可以修改参数的值，并且想要在这些修改在函数调用结束后仍然存在，那么就应该把这个参数定义为*输入输出参数*

定义一个输入输出参数时，在参数定义前加 `inout` 关键字，且这个值需要时变量而不是常量

当传入的参数作为输入输出参数时，需要在参数名前加 `&` 符，表示这个值可以被函数修改。

```swift
func swapTwoInts(_ a: inout Int, _ b: inout Int) {
    let temporaryA = a
    a = b
    b = temporaryA
}
```

**3）函数类型**

函数类型就像是函数指针一样。实则，还是那句话"万物皆对象"。

在 Swift 中，每个函数都有种特定的函数类型，函数的类型由函数的参数类型和返回类型组成。使用函数类型就像使用其他类型一样。

```swift
func addTwoInts(_ a: Int, _ b: Int) -> Int {
    return a + b
}

func multiplyTwoInts(_ a: Int, _ b: Int) -> Int {
    return a * b
}

var mathFunction: (Int, Int) -> Int = addTwoInts
```

函数类型可以作为参数类型，返回值类型等。

#### 2.6 闭包

**1) 闭包简介**

OC有Block，C++有lambda表达式，swift则有闭包。

闭包采用如下三种形式之一：

- 全局函数是一个有名字但不会捕获任何值的闭包
- 嵌套函数是一个有名字并可以捕获其封闭函数域内值的闭包
- 闭包表达式是一个利用轻量级语法所写的可以捕获其上下文中变量或常量值的匿名闭包

闭包的语法

```swift
{ (parameters) -> return type in
    statements
}

// 使用
var blockOne:(String, Int) = {(parma1:String, parma2:Int) -> void in 
                             }
```

```swift
// 还有很多精巧的语法
let names = ["Chris", "Alex", "Ewa", "Barry", "Daniella"]
// 单行表达式闭包可以通过省略 return 关键字来隐式返回单行表达式的结果
reversedNames = names.sorted(by: { s1, s2 in s1 > s2 } )
// 直接通过 $0，$1，$2 来顺序调用闭包的参数，以此类推。
reversedNames = names.sorted(by: { $0 > $1 } )
// Swift 的 String 类型定义了关于大于号（>）的字符串实现，其作为一个函数接受两个 String 类型的参数并返回 Bool 类型的值
reversedNames = names.sorted(by: >)
```

另外要注意的是闭包表达式是引用类型。

**2) 逃逸闭包**

逃逸闭包用于网络通信等异步操作。

一种能使闭包“逃逸”出函数的方法是，将这个闭包保存在一个函数外部定义的变量中。举个例子，很多启动异步操作的函数接受一个闭包参数作为 completion handler。这类函数会在异步操作开始之后立刻返回，但是闭包直到异步操作结束后才会被调用。在这种情况下，闭包需要“逃逸”出函数，因为闭包需要在函数返回之后被调用。

```swift
//  @escaping，用来指明这个闭包是允许“逃逸”出这个函数的。
var completionHandlers: [() -> Void] = []
func someFunctionWithEscapingClosure(completionHandler: @escaping () -> Void) {
    completionHandlers.append(completionHandler)
}
```



#### 2.7 枚举、结构体、类

**1）枚举**

Swift 中的枚举更加灵活，不必给每一个枚举成员提供一个值。如果给枚举成员提供一个值（称为原始值），则该值的类型可以是字符串、字符，或是一个整型值或浮点数。

```swift
// 枚举的定义
enum SomeEnumeration {
    // 枚举定义放在这里
    // 使用 case 关键字来定义一个新的枚举成员值。
}
// 枚举的定义例子
enum CompassPoint {
    case north
    case south
    case east
    case west
}
// 可以在一个枚举中定义一组相关的枚举成员，每一个枚举成员都可以有适当类型的关联值。
enum Barcode {
    case upc(Int, Int, Int, Int)
    case qrCode(String)
}
// 枚举的遍历
enum TestEnum:CaseIterable {
  
}
for item in TestEnum.allCases {
  
}
```

**2）结构体与类**

定义语法：

```swift
struct SomeStructure {
    // 在这里定义结构体
}
class SomeClass {
    // 在这里定义类
}

// 结构体是值类型
// 类是引用类型
// 结构体也可以当类来使用，但是不建议
// 还要很多相同点, 但是class就是定义类的, 不过多赘述struct
```

**3）类的属性与方法**

a）属性

swift中类的属性分为存储属性与计算属性。

存储属性就是存储在特定类或结构体实例里的一个常量或变量。

```swift
class FixedLengthRange {
    var firstValue: Int
    let length: Int
}
```

计算属性就是不直接存储值，而是提供一个 getter 和一个可选的 setter，来间接获取和设置其他属性或变量的值。

```swift
struct Point {
    var x = 0.0, y = 0.0
}
struct Size {
    var width = 0.0, height = 0.0
}
struct Rect {
    var origin = Point()
    var size = Size()
    var center: Point {
        get {
            let centerX = origin.x + (size.width / 2)
            let centerY = origin.y + (size.height / 2)
            return Point(x: centerX, y: centerY)
        }
        set(newCenter) {
            origin.x = newCenter.x - (size.width / 2)
            origin.y = newCenter.y - (size.height / 2)
        }
    }
}
```

swift还有一个类属性，类似C++的static成员。

使用关键字 `static` 来定义类型属性。在为类定义计算型类型属性时，可以改用关键字 `class` 来支持子类对父类的实现进行重写。

```swift
struct SomeStructure {
    static var storedTypeProperty = "Some value."
    static var computedTypeProperty: Int {
        return 1
    }
}
enum SomeEnumeration {
    static var storedTypeProperty = "Some value."
    static var computedTypeProperty: Int {
        return 6
    }
}
class SomeClass {
    static var storedTypeProperty = "Some value."
    static var computedTypeProperty: Int {
        return 27
    }
    class var overrideableComputedTypeProperty: Int {
        return 107
    }
}
```

b）方法

实例方法同普通方法一样，方法体可以使用`self`关键字

类方法在方法的 `func` 关键字之前加上关键字 `static`，来指定类型方法。类还可以用关键字 `class` 来指定，从而允许子类重写父类该方法的实现。

#### 2.8 类的构造与析构

类的构造与析构和C++的构造与析构类似，唯一不同的可能就是在语法上不同了。

比如：全缺省构造，默认构造等等。

构造函数语法

```swift
// 默认构造
init() {
  // 在构造函数中，如果没有明企鹅super.init()，那么系统会自动添加
  // 使用KVC构造时，一定要先super.init()
}


// 缺省构造
init(parmaterNmae : type) {
	// 有歧义则加self
  self.xxx = parmaterNmae;
}
init?(parmaterNmae : type){
  // 可能返回空，获取时为可选类型
}


// 必要构造
required init(parmaterNmae : type) {
  
}


```

#### 2.9 属性监听器

```swift
class SomeClass : NSObject {
  var name : String {
    // 属性即将改变
    willSet {
      
    }
    // 属性已经改变
    didSet {
      
    }
  }
}
// 即使设置相同的值仍会触发
// 不能同setter同时存在
// 
```

#### 2.10 类的继承

```swift
// 重载：同C++，参数顺序，名称，数量等不同
// 重写：需要override关键字，表示是重写的，不允许子类重写则使用final关键字
```



#### 2.11 访问控制



#### 2.12 扩展与协议



#### 2.10 泛型

对于泛型可以类比C++的模板，C++的标准模板库STL就是泛型编程的经典。

swift的泛型语法也和模板类似——`<T>`。

```swift
// swift
func swapTwoValues<T>(_ a: inout T, _ b: inout T) {
    let temporaryA = a
    a = b
    b = temporaryA
}
```

```c++
// C++
void swapTwoValues(T& a, T& b) {
  T tmp = a;
  a = b;
  b = tmp;
}
```

C++有函数模板，类模板，全特化模板。片特化模板等。

swift有泛型函数，泛型类型，泛型扩展等。

### 3. 总结

由于时间关系，总结的比较草率，还有很多需要补充的地方。对于swift语言，只是对语法进行了简单的学习。其核心思想还需进一步的深入学习——实践+概念。
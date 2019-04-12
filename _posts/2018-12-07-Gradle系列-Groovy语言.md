## 1. Groovy简介

Groovy是Apache旗下的一种基于JVM的面向对象编程语言，即可以面向对象编程，也可以作为纯粹的脚本语言，Groovy完全支持Java语法，可以和Java互相调用，同时Groovy吸收了其他语言的优秀特性，支持动态类型转换、闭包、元编程等。

## 2. Groovy调试

Groovy语言的编写和调试，可以在IDE中编写调试，也可以在文本中直接编写调试，新建目录下新建build.gradle文件，在文件中编译gradle task，在task中编写Groovy代码，运行task调试Groovy代码。

## 3. Groovy语法

### 3.1 变量

Groovy用def关键字定义变量，可以不指定变量类型，默认修饰符是public，语句后边的";"可以省略。

```groovy
def name = 'wx';
def a = 1
def int b = 10;
```

### 3.2 方法

方法使用返回类型或者def关键字定义，方法可以接受任意数量的参数，参数可以不声明类型，默认方法的修饰符是public，如果不使用return，方法返回值为最后一行代码执行的结果。

```groovy
task method <<{
    add(1,2)   //语句后的；可以省略
    minus 1,2  //方法括号可以省略
    def is = test 2,1
}

def add(int a, int b)  {
    println a+b;
}
int minus(a, b) { //参数类型可以省略
    printl a-b
}

boolean test(a, b) {
    a > b;       //return 可以省略
}
```

### 3.3 类

Groovy类和Java类相似，但Groovy类默认修饰符为public，没有可见性修饰符时自动为成员变量生成对应的set和get方法。

### 3.4 字符串

在Groovy中单引号字符串被解释成`java.lang.string`，不支持内插值。当双引号字符串中没有插值表达式时，字符串的类型为`java.lang.String`，当双引号字符串中包含插值表达式时，字符串类型为`groovy.lang.GString`。三引号字符串不支持内插值，但保留文本的换行和缩进格式。

```groovy
def name = 'wx' 
def greeting = "Hello ${name}"
def name = '''Android测试
       Android换行测试
？''''''
```

### 3.5 数据类型

Groovy中的数据类型主要有三种：Java中的基本数据类型，Groovy中的容器类，闭包。

#### 3.5.1 Groovy集合

Groovy没有自己的集合类，它是在Java的集合类继承上进行增强和简化，Groovy的List对应Java的List接口，默认实现是ArrayList。

#### 3.5.2 Closures（闭包）

Groovy闭包是一种可执行代码块的方法，闭包也是对象，可以向方法一样传递参数，它可以访问到其外部的变量或方法。

```groovy
{ [closureParameters -> ] statements }
```

闭包分为两个部分：[closureParameters -> ]参数列表部分和statements语句部分，参数部分是可选的，如果闭包只有一个参数，参数名是可选的，Groovy会隐式指定it作为参数名，如下所示：

```groovy
//1.无参数
{ println "Hello Word！"}

//2.有参数，需要用->进行分离
{ param-> println "${param}" }

//3.只有一个参数时可以用it来访问该参数
{ println "Hello ${it}" }
```

#### 2.5.3 闭包使用

闭包是groovy.lang.Cloush类的一个实例，可以负责给变量或者字段，如下所示：

```groovy
//1.将闭包赋值给一个变量
def println ={ it -> it * it }

//2.将闭包赋值给Closure类型变量
Closure do = { println 'do!' }

//3.打印true
assert println instanceof Closure
```

调用闭包，可以当方法调用，也可以显示调用call方法，如下所示：

```groovy
def code = { 123 }

//1.闭包当做方法调用
assert code() == 123 

//2.显示调用call方法
assert code.call() == 123 

//3.调用带参数的闭包
def isOddNumber = { int i -> i%2 != 0 }   
assert isOddNumber(3) == true 
```


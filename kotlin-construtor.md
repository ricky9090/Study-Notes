# Kotlin中的构造函数

假设有个Person类，含有两个属性name和age先看一个Java版本
```java
public class PersonJava {
    private final String mName;
    private final int mAge;

    public PersonJava(String name) {
        this.mName = name;
        this.mAge = 0;
    }

    public PersonJava(String name, int age) {
        this.mName = name;
        this.mAge = age;
    }
}
```

两个属性被声明为``final``类型，在构造函数中一次性赋值。其中只有``name``的构造函数设定了一个默认年龄0，这可以进一步优化：
```java
public class PersonJava {
    private final String mName;
    private final int mAge;

    public PersonJava(String name) {
        this(name, 0);
    }

    public PersonJava(String name, int age) {
        this.mName = name;
        this.mAge = age;
    }
}
```

一个参数的构造函数内部调用两个参数的构造函数，传入默认值。
下面来看一下这个类的Kotlin版本，首先是仿照第一个版本的代码：
```kotlin
class Person {
    private val mName: String
    private val mAge: Int

    constructor(name: String, age: Int) {
        this.mName = name
        this.mAge = age
    }
    
    constructor(name: String) {
        this.mName = name
        this.mAge = 0
    }
}
```
在这里声明了两个常量，同样通过两个构造函数进行赋值。进一步改造成第二个版本：
```kotlin
class Person {
    private val mName: String
    private val mAge: Int

    constructor(name: String, age: Int) {
        this.mName = name
        this.mAge = age
    }

    constructor(name: String) : this(name, 0)
}
```
在kotlin中，代理调用类中的其他构造函数，需要在当前声明的构造函数后使用``: this(parameter)``的语法，在这里我们除了给属性赋值，没有其他操作，所以后面的函数实现也可以省略了。

Kotlin还支持在声明类时，声明首要构造函数，可以进一步改写代码：
```kotlin
class Person(name: String, age: Int) {
    private val mName: String
    private val mAge: Int

    init {
        this.mName = name
        this.mAge = age
    }

    constructor(name: String) : this(name, 0)
}
```
直接将两个参数的构造函数声明在类的开始，这种声明方式下，构造函数不能跟随实现代码块，相应的初始化语句需要写在``init``代码块中。
这时编译器提示我们可以将``init``代码块直接替换成赋值语句：
```kotlin
class Person(name: String, age: Int) {
    private val mName: String = name
    private val mAge: Int = age

    constructor(name: String) : this(name, 0)
}
```
Kotlin还支持将类的属性直接声明在首要构造函数中，因此代码可以进一步简化：
```kotlin
class Person(val mName: String, val mAge: Int) {
    constructor(name: String) : this(name, 0)
}
```
再进一步，函数形参的声明可以包含默认值，因此对``val``属性也可以做同样处理：
```kotlin
class Person(val mName: String,val mAge: Int = -1) {
    constructor(name: String) : this(name, 0)
}
```
为了作区分，这里将默认值与重载传入不同值，编译器并没有报错。但默认值本就是在这个参数缺失情况下起作用的，
而我们又声明了只有一个参数的构造函数，并赋值给``age=0``，因此实际``-1``这个值并没有起作用。
以下是调用的效果：
```kotlin
Person("Jack")  // mName = "Jack", mAge = 0
Person("Jack", 25)  // mName = "Jack", mAge = 25 
```
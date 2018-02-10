---
title: 让Java类在kotlin中也支持解构声明
date: 2018-02-08 11:34:57
categories: Android
tags: 
    - Android
    - Kotlin
    - Java
---


关于`kotlin`的一大段介绍我就不说了，首先我们来看看结构声明在`kotlin`中是如何使用的。

```kotlin
//首先定义一个Person类
data class Person(val name: String? = null, val age: Int = 0)

fun main(args: Array<String>) {
    val (name, age) = Person("zhangsan", 20)
    println(name) // zhangsan
    println(age) // 20
// ===等价于===
    val p = Person("zhangsan", 20)
    println(p.name) // zhangsan
    println(p.age) // 20
}
```
有时把一个对象 解构 成很多变量会很方便，我们可以同时声明多个变量
一个解构声明会被编译成以下代码：

```kotlin
val name = p.component1() // zhangsan
val age = p.component2() // 20
```
其中`component1()`,`component2()`,`componentN()`函数是由Kotlin编译器自动生成的，方法的数量取决于属性的数量，方法的顺序取决于属性的顺序。

我们用Kotlin插件的`Show Kotlin Bytecode`功能将`Person`反编译可以得到如下Java代码：

```java
public final class Person {
   @Nullable // String?类型隐含添加上@Nullable注解，在Java调用Kotlin时能给出友好地非空提示
   private final String name;
   // Int类型会被翻译成基本类型int，只有需要用到方法时才会转成Integer
   private final int age;

   ... setter/getter ... 

   public Person(@Nullable String name, int age) {
      this.name = name;
      this.age = age;
   }

   ......
   
   @Nullable
   public final String component1() { // 第一个属性
      return this.name;
   }

   public final int component2() { // 第二个属性
      return this.age;
   }

   @NotNull
   public final Person copy(@Nullable String name, int age) {
      return new Person(name, age);
   }

   ... toString/hashCode/equals ... 
  
}
```
从上面的代码可以看出，`data class`除了自动生成`setter/getter`，`toString`，`equals`，`hashCode`之外，还有`component1`和`component2`。但是`Java`编译器是不会自动生成的，也就无法使用结构声明。

我们可以手动添加吗？？？？ 答案是可以的，你可以在`Person`类(Java)里面添加如下代码

```java
    @NotNull
    public String component1() {
        return getName();
    }
    public int component2() {
        return getAge();
    }
```

上面的方法可行，但似乎不够完美，我们大部分使用的API都是无法更改其内部结构的。根本没办法手动添加。

其实`Kotlin`是支持结构声明扩展的，你可以这么做

```kotlin
// 1. operator关键字必不可少
// 2. 为什么我这里显示声明**String?**呢？
// Person是Java类，getName()返回的是Java的平台类型(String!),而不是String或者String?，可空性未知
// 你应当显示声明name是否可空，以便更好运用Kotlin空安全的特性。
// 当然你也可以不添加，因此你会得到一个波浪线和一个黄色感叹号，但是这并不会影响程序的运行
// 而int类型是基本类型，编译成Int类型是一定非空的，开发工具不会因此报出警告。
operator fun Person.component1(): String? = name 
operator fun Person.component2() = age

fun main(args: Array<String>) {
    val (name, age) = Person("zhangsan", 20)
    println(name)
    println(age)
}

```

讲到这里，我想同学们都很清楚知道如何使用结构声明扩展让Java类也支持结构声明了，真是可以可贺。

你可能见过下面一段代码：

```kotlin
for ((key, value) in map) {
// 使用该 key、value 做些事情
}
```

这是一段遍历map的方法，实现它我们只需要这样做

* 通过提供一个 `iterator()` 函数将映射表示为一个值的序列；
* 通过提供函数component1() 和 component2() 来将每个元素呈现为一对。

```
operator fun <K, V> Map<K, V>.iterator(): Iterator<Map.Entry<K, V>> = entrySet().itera
tor()
operator fun <K, V> Map.Entry<K, V>.component1() = getKey()
operator fun <K, V> Map.Entry<K, V>.component2() = getValue()
```

题外话：其实kotlin标准库已经实现了一些很方便的扩展，包括上述的。`kotlin`拥有`java`所不具有的诸多优点，在谷歌大力支持下，我觉得Android开发者都应该学习并使用这门语言。



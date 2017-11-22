---
title: 如何理解Kotlin中属性
date: 2017-10-27 15:59:12
tags:
    - Android
    - kotlin
categories: Android
---

## 属性的定义
在**kotlin**中，**var**和**val**是用来声明属性的两个关键字，在**kotlin**官方参考文档上是这么说的。

> **Kotlin**的类可以有属性。 属性可以用关键字 **var**声明为可变的，否则使用只读关键字**val**。

那什么是可变属性，什么又是可读属性呢？我们在问这个问题前可以先回顾一下java对属性的定义。

> 属性可以通过**get**、**set**、**is**（可以替代**get**，用在布尔型属性上）方法或遵循特定命名规范的其他方法访问。

在这里我们可以理解为，在**java**中，对一个字段生成**public**的**set get**方法，那么他就是一个可变（可读可写）属性，仅提供**public get**或者**set**被**private**修饰的就是一个只读属性。

在kotlin中我们是如何定义属性的呢

```java
class Person {
    //定义一个String类型的属性name
    var name: String? = null
    //定义一个Boolean类型的属性deceased
    var deceased: Boolean? = false
}
```
kotlin和java同属于jvm语言，编译后会生成class文件。我将上面的kotlin代码生成的class再次反编译成java代码，如下：

```java
public final class Person {
   @Nullable
   private String name;
   @Nullable
   private Boolean deceased = false;

   @Nullable
   public final String getName() {
      return this.name;
   }

   public final void setName(@Nullable String var1) {
      this.name = var1;
   }

   @Nullable
   public final Boolean getDeceased() {
      return this.deceased;
   }

   public final void setDeceased(@Nullable Boolean var1) {
      this.deceased = var1;
   }
}
```

看到这里你会发现，反编译后代码和java属性的定义是一毛一样有木有。所以**kotlin**的属性，在**java**代码中可以通过**set get**方法来调用。

## 只读属性和常量

有的地方说，**var**是声明变量的，**val**是声明常量的。这个其实不是完全正确，至少**val**在官方参考文档上说的是只读属性，而不是常量。比方说温度，我们可以获取温度的变化，但是不能人为直接改变温度，那么温度就是只读属性，而不是常量。
再比如说list的isEmpty属性

```kotlin
val isEmpty get() = this.size == 0
```

isEmpty属性受到size的改变而改变，但是无法主动去设置isEmpty的值。那么isEmpty就是个只读属性。

在这里要分为几种情况来理解**val**。

* **val**声明的属性被字面量或者表达式赋值的时候
* **val**声明的属性实现了自定义的**get**方法

看下面的例子


```kotlin
class Person {
    //这里的name属性被字面量“zhangsan”赋值了
    val name: String? = "zhangsan"
     //age属性的值受birthday的影响
    val age: Int get() {
        return Calendar.getInstance().get(Calendar.YEAR) - birthday
    }
    var birthday: Int = 0
}
```
 
 反编译成**java**代码是这样的
    
```java
public final class Person {
    //name属性被字面量赋值，编译后会加上final关键字，成为真正的常量
   @Nullable
   private final String name = "zhangsan";
   private int birthday;

   @Nullable
   public final String getName() {
      return this.name;
   }
    
    //age属性只实现了get方法，无法通过set方法来改变值，是个只读属性
   public final int getAge() {
      return Calendar.getInstance().get(1) - this.birthday;
   }

   public final int getBirthday() {
      return this.birthday;
   }

   public final void setBirthday(int var1) {
      this.birthday = var1;
   }
}

```

## 什么是变量的幕后字段

在**kotlin**中可以通过关键字声明属性，那可不可以声明字段呢？它有没有字段呢？
答：**kotlin**中不能声明字段，但是**kotlin**中是有字段的，他有一个幕后字段的概念，每一个属性可以有一个幕后字段，也可以没有。

拿上面的例子再讲一遍

```kotlin
val isEmpty get() = this.size == 0
```

isEmpty的值只和size有关，那么在反编译之后，不会出现如下代码

```java
private boolean isEmpty = false;

```

这里可以得出一个结论，属性不一定非得有字段。对**java**属性定义的后半句是这么说的，“*或遵循特定命名规范的其他方法访问*”，所以关键看方法的定义，比如说在一个类里面有**setWidth()**,**getWidth()**这两个方法。就可以认定为这个类有**width**属性，但是不一定有**width**这个字段。

也就是说这个属性没有幕后字段。


再看下面这个例子

```kotlin
//name属性没有实现自定义的set get方法
var name:String? = null;
......

name = "zhangsan"

val len = name.length()
```
**name**没有实现自定义的**set get**方法，字面值"zhangsan"需要一个字段来赋值。那么在class里面会生成 

```java
//这个就是name属性的幕后字段
private String name = null;
```

那么幕后字段可以用代码访问到么，这个是可以的，使用 **field** 关键字


```kotlin
var name: String? = null

//上面这句代码其实和下面的代码是等价的，没有区别。
var name: String? = null
   set(value) {
       field = value
   }
   get() {
       return field
   }
```

讲到这里我相信大家对幕后字段已经有了一定的了解。

## 扩展属性
在kotlin声明一个扩展属性的方法如下。

```kotlin
val Person.city:String get() { return "city" } 
```

> 注：扩展属性只能用生成自定义set get方法来实现，不能直接用字面量来赋值。所以下面的代码是错的

```kotlin
val Person.city:String = "city"
```
第二句代码是无法编译通过的，那是因为

> 扩展是静态解析的.扩展不能真正的修改他们所扩展的类。通过定义一个扩展，你并没有在一个类中插入新成 员， 仅仅是可以通过该类型的变量用点表达式去调用这个新函数,或者属性。由于扩展没有实际的将成员插入类中，因此对扩展属性来说幕后字段是无效的。这就 是为什么扩展属性不能有初始化器。他们的行为只能由显式提供的 **get/set** 定义。

静态解析的结果是

```java
  @NotNull
   public static final String getCity(@NotNull Person $receiver) {
      return "city";
   }
```

所以扩展仅仅是可以通过点语法来调用，并不是真正插入类里面，也不存在幕后字段。


## 总结

* 属性可以通过**get**、**set**、**is**（可以替代**get**，用在布尔型属性上）方法或遵循特定命名规范的其他方法访问。
* 只读属性和常量不是同一个概念，具体看上头。
* 属性可以有一个幕后字段，也可以没有
* 扩展是静态解析的，扩展属性是没有幕后字段的
* 扩展只是能通过点语法调用而已。



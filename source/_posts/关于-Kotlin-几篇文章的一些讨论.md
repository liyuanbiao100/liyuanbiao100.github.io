---
title: 关于 Kotlin 几篇文章的一些讨论
tags: Kotlin
abbrlink: 95d4684d
date: 2018-06-26 13:25:29
---


## 抛弃 Java 改用 Kotlin 的六个月后，我后悔了
原文地址：https://blog.csdn.net/csdnnews/article/details/80746096

这篇文章我很早之前之前就看过了，是一片批判 Kotlin 的文章，一时间在各大技术论坛疯狂转载。由于我本人很喜欢 Kotlin ，所以只是粗略的看了一眼，并觉得作者是在放屁。上面做法并不符合我认真求学的风格，今天我决定认真的看看，仔细分析下。

文中举了很多例子说明 Kotlin 的不完美，甚至会降低工作效率。我觉得，尽管 Kotlin 不完美，因为任何一门编程语言都不可能尽善尽美，但是他是在 Java 的基础上，为了解决 Java 的一些设计缺陷而设计出来的语言，使用 Kotlin 能极大增加工作效率。

### 名称遮蔽

作者认为这是 Koltin 的设计缺陷，试图通过警告信息来解决这个问题。我认为这是 Kotlin 的一个功能，并不属于缺陷，名称遮蔽并不会造成编程错误，在有些地方也有名称遮蔽的应用。

### 类型推断

Kotlin 的类型推断比 Java 强大不止一点点，Java 在 8 开始才对类型推断有比较好的支持，到了 10 才可以使用`var`来声明变量，通过类型推断而不需要显示声明变量类型。这一特性也只在局部作用域生效。现在   
Java 的使用者里面有很多都是Android开发者，本人就是。Android 对 Java 新语法的支持并不是很良好，为了兼容以前的 Android 系统，开发者们不得不使用 Java7 甚至 Java6 的语法。Kotlin 的出现让这一切有了很大的改观。

### Null 安全类型

作者在描述 Kotlin 与Java 互操作的过程中不可避免的会产生第三种类型，**平台类型**，作者认为平台类型会让代表变得糟糕，误导你，会禁用 Kotlin 的 NULL 安全机制。

我觉得 Kotlin 为了要与 Java 实现100% 互操作而产生平台类型这是不可避免的，因为 Java 从一开始就没有这种安全机制，在 Java 语言中，每个类变量都有可能为 null 。正是因为这样，Kotlin 编译器才为了我们做了更多的事情，他并没有将所有的 Java 变量翻译成可空类型。

* 基础类型是不可能为 null 的，在 Kotlin 中会作为不可空类型的存在
* Kotlin编译器还支持识别`@Nullable` 和 `@NonNull` 注解，他们分别会被翻译为可空类型和非空类型，现在很多类库为了适应 Kotlin 而加上了这些注解。
* 当平台类型不可避免的时候，可以显示为 Java 变量声明可空类型或者非空类型，避免在 Kotlin 中使用平台类型。如果你没有遵守这一规则，Kotlin 编译器则会以警告的形式提示你。
* Kotlin 有强大的 IDE 支持，在编辑器中会显示类型推断的实际类型。平台类型则是类型名称后面加上！。这只在编辑器中作为提示显示，语法中并不存在。

Kotlin 并不会误导你，对语言的不熟悉才是你被误导的真正原因。我觉得 Kotlin有很多吸引我的地方，其中 Null 安全机制是我最喜欢的。

### Maybe

在 Java8 中为了解决空安全的问题，而引入了Optional。作者称 如今，Optional 是在 API 边界处理返回类型中的空值的非常流行的方式。而我倒觉得这是非常无奈的做法，比起 Kotlin 从根源上解决查太多。
 
为了论证 Kotlin 很差，作者举了下面的例子：
例如，在 Java 中： 

```java
public int parseAndInc(String number) {
    return Optional.ofNullable(number)
                   .map(Integer::parseInt)
                   .map(it -> it + 1)
                   .orElse(0);
}
```

在 Kotlin 中，需要这样做

```kotlin
fun parseAndInc(number: String?): Int {
    return number?.let { Integer.parseInt(it) }
                 ?.let { it -> it + 1 } ?: 0
}
```
然后比较这两段代码的可读性。

可读性对于每个人都不一样吧，而且谁会在 kotlin 使用 `Integer.parseInt` 呢？如果作者喜欢 Optional ，你可以使用它。 Kotlin 在 JVM 上运行。即使是自己实现一个类似于 Optional 的类也是很容易的。

### 类名称字面常量

这点我就不想多说了，作者纯粹为了找茬嘛。
文中说：Kotlin 把 Kotlin 类和 Java 类进行了区分，并为其提供了语法规范：

```kotlin
val kotlinClass : KClass<LocalDate> = LocalDate::class
val javaClass : Class<LocalDate> = LocalDate::class.java
```
这看起来非常丑陋。

丑陋，呵呵呵呵呵呵呵。。。。。
还有其他的一些点论证 Kotlin 无比丑陋的说法就不一一说明了。

另外作为认为 Kotlin 具有陡峭的学习路线，这点我是不认同的，我只用了两天就将 Kotlin 用于生产环境了。Groovy 虽然很容易从 Java 转换过来，因为Java代码是正确的 Groovy 代码，但是想要深入学习一点都不比 Kotlin 容易，特别是没有接触过弱类型语言的开发者来说。Groovy 我也学习过，我更认为同样作为静态类型的 Kotlin 与 Java 更接近，即使他的语法与 Scala 接近。

### 关于作者最后的想法

> 学习新技术就像一项投资。我们投入时间，新技术让我们得到回报。但我并不是说 Kotlin 是一种糟糕的语言，只是在我们的案例中，成本远超收益。

作者说的是普遍的道理，所以我不能说他错，但是就目前而言我不认为 Kotlin 是糟糕的语言。使用 Kotlin 一年多时间以来，我感受到 Kotlin 带来的诸多好处，一种学习新语言，新特性带来的心灵洗礼，我不会说 Java 和 Kotlin 谁更好，他们都很好，我都很喜欢。


## 谷歌大牛说：为什么 Kotlin 比你们用的那些垃圾语言都好
原文地址：http://blog.jobbole.com/111249/

这篇文章在安卓团队在谷歌 I/O 2017 大会上宣布 Kotlin 成为官方头等支持语言之后发布，同样获得疯狂的转载。这篇文章的标题有点标题党，甚至有点引战的味道。要让我用一个词来形容就是跪舔。

作者讲述了十年来的经历，以及为什么使用 Kotlin 。
比如 Android 有几个很糟糕的 API。因为这样，接下来一年半没有碰 Android 编程。来自俄罗斯的救星 Kotlin，对他第一印象是简洁，而且有一种似曾相识的感觉，和 Swift 很像，然后简述 Kotlin 和 Swift 的历史，是 Kotlin 在前，Swift 在后，所以是 Swift 和 Kotlin 很像。
其实谁和谁像已经无所谓了， Kotlin 和 Swift 都是现代化语言，而现在化语言都是很像的，作者花了大篇幅只为争个先后，这是跪舔。

接下里说为什么要从 Java 叛逃到 Kotlin。大部分是讲和 Java8 的对比，还有在 Android开发上的便利性，这和标题有个毛关系。回到糟糕的Android API，原文说：
> Kotlin 致力于让大家绕过 Android API 那些恶心的东西，并且能让你充分发挥你的经验，这一点甚至比 iOS 做得还要好。好吧，至少来说比 Objective-C 做得好，因为我觉得 Swift 肯定也不会差。知道为什么吗？因为 Swift 和 Kotlin 很像啊。

Swift 和 Objective-C 只是在互操作性没有 Kotlin 与 Java 这么好，没办法，他们都是基础JVM的（我知道有人要杠了），这个 Android API 本身没有关系吧，API就在哪里，你用与不用，他都在哪里，请问 Kotlin 是如何绕过去的。 可能作者根本不喜欢 Android 吧，不然怎么会因为 Java 拒绝 Android，语句中看不出丝毫热爱。可能是为了跪舔而扯上的Android吧，但似乎有点用力过猛了。这也是跪舔。



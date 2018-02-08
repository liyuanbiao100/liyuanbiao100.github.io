---
title: kotlin编译 Kotlin home does not exist or is not a directory
date: 2018-02-04 03:19:36
categories: Android
tags: 
	- Android
	- Kotlin
---

虽然kotlin成为谷歌干儿子才半年之久，但是我依然早早的在项目中使用它。秉承着早入坑早脱坑的理念，我遇到了下面这个问题。代码没有报错，但是编译不通过。

```log
e: Kotlin home does not exist or is not a directory: 
```

这个问题原因不明，莫名其妙出现的。代码完全没有问题，但是就是不能编译。
下面是解决方法


```shell
killall java
./gradlew clean assemble
```

解决原理不清楚。

下面是相关连接
https://discuss.kotlinlang.org/t/task-compilekotlin2js-fails-in-gradle-4-4-kotlin-home-does-not-exist-or-is-not-a-directory/5706/3


---
title: Java转Kotlin boolean属性的坑
date: 2017-10-27 11:59:22
categories: Android
tags: 
    - Java 
    - Kotlin 
    - Android
---

### 踩坑

假设一个 `JavaBean` 类中有 `boolean` 类型的字段叫做 `enabled` 
它生成的`setter`/`getter` 方法将是

* setter: `setEnabled(boolean enabled)`
* getter: `isEnabled()`

```java
public static class JavaBean{
        private boolean enabled;

        public boolean isEnabled() {
            return enabled;
        }

        public void setEnabled(boolean enabled) {
            this.enabled = enabled;
        }
    }
```
使用 `Android Studio` 的 `kotlin` 插件转换后将生成 `isEnabled` 属性，而不是 `enabled` 属性。

```kotlin
class JavaBean {
    var isEnabled: Boolean = false
}
```

我们知道`kotlin`编译后也是生成`class`文件，所以也是可以转成`java`文件的，通过查看反编译后的结果来学习`kotlin`不失为一种好方法。

可以在 `Android Studio` 执行下面的操作将 `kotlin` 代码重新转成`java`代码。
Tools / Kotlin / Show Kotlin Bytecode / Decompile. 

```java

public final class JavaBean {
   private boolean isEnabled;

   public final boolean isEnabled() {
      return this.isEnabled;
   }

   public final void setEnabled(boolean var1) {
      this.isEnabled = var1;
   }
}
```

可以看到是 `isEnabled` ，不是 `enabled`。

#### 可能遇到的问题
* 我在项目中使用的是**Gson**来解析后台返回来的数据。**Gson**是通过反射字段的名称来赋值的，然而数据中是 `enbaled` ，代码里是 `isEnabled` ，它将永远不能被赋值，永远为`false`，解决方法可以是

    * 给字段添加 `@SerializedName(value = "enbaled")`
    * 直接将 `isEnabled` 改成 `enbaled`，保证kotlin的幕后字段与后台返回数据一致即可.

* 如果你项目中用的是**fastjson**，那么需要为每个字段添加`setter`/`getter` 方法。**fastjson**是根据**Javabean**属性来赋值的，所以必须保证属性名正确。由于之前**fastjson**出现过序列化成功，但是反序列化失败的bug，致使我放弃使用**fastjson**。虽然**fastjson**以快著称，但是快不是唯一的需求，它只在数据量大的时候有明显差异，数据量小的时候差异几乎可是忽略不计，所以我选择**Gson**。

### 细谈Java属性

在Java里面生成 `setter`/`getter` 方法是有一定规则的。

* 如果类的成员变量的名字是xxx，那么为了更改或获取成员变量的值，即更改或获取属性，在类中可以使用两个方法：
    * getXxx()，用来获取属性xxx。
    * setXxx()，用来修改属性xxx.。
* 对于boolean类型的成员变量，即布尔逻辑类型的属性，允许使用"is"代替上面的"get"。
* 类中访问属性的方法都必须是public的，一般属性是private的。
* 类中如果有构造方法，那么这个构造方法也是public的并且是无参数的。

因为Java属性是由 `setter`/`getter` 方法决定的，而不是字段。在一个Java类里面，如果有`void setName(String name)`和`String getName()`方法，那么可以认为这个类有`name`这个属性（**可读可写**），如果仅有get，那么这个`name`属性就是个**只读属性**。即使没有`name`这个字段也是一样成立的。

也就是说，对于 `boolean` 类型，`enabled` 和 `isEnabled` 这两个字段生成的`setter`/`getter` 方法是一致的。当这两个字段同时存在时，只能针对其中一个生成`setter`/`getter` 方法。看下图你就明白了

![](/media/15057241768106.jpg)

`isEnabled` 已经无法再生成了，因为已经存在了。在项目当中应该避免则这样的命名，选择其一即可。




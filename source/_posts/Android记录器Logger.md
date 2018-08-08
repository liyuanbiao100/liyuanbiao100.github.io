---
title: Android记录器Logger
tags: Android
abbrlink: de93668b
date: 2017-09-08 22:16:24
---

# Logger

简单，漂亮，功能强大的Android记录器

## 安装

```groovy
implementation 'com.orhanobut:logger:2.2.0'
```
## 初始化
```java
Logger.addLogAdapter(new AndroidLogAdapter());
```

## 使用

```java
Logger.d("hello");
```

## 输出
![](/media/15337379282287.jpg)

## 可选项

```java
Logger.d("debug");
Logger.e("error");
Logger.w("warning");
Logger.v("verbose");
Logger.i("information");
Logger.wtf("What a Terrible Failure");
```

支持字符串格式参数

```java
Logger.d("hello %s", "world");
```

支持集合（仅适用于调试日志）

```java
Logger.d(MAP);
Logger.d(SET);
Logger.d(LIST);
Logger.d(ARRAY);
```

Json和Xml支持（输出将处于调试级别）

```java
Logger.json(JSON_CONTENT);
Logger.xml(XML_CONTENT);
```

## 高级

```java
FormatStrategy formatStrategy = PrettyFormatStrategy.newBuilder()
  .showThreadInfo(false)  // (可选) 是否显示线程信息。默认为true
  .methodCount(0)         // (可选) 要显示多少个方法行。默认2
  .methodOffset(7)        // (可选) 隐藏内部方法调用以抵消偏移。默认5
  .logStrategy(customLog) // (可选) 更改打印日志策略。默认LogCat
  .tag("My custom tag")   // (可选) 每个日志的全局TAG。默认“PRETTY_LOGGER”
  .build();

Logger.addLogAdapter(new AndroidLogAdapter(formatStrategy));
```

## 可打印
日志适配器通过检查此功能来检查是否应打印日志。如果要禁用/隐藏输出日志，请覆盖isLoggable方法。 true将打印日志消息，false将忽略它。
```java
Logger.addLogAdapter(new AndroidLogAdapter() {
  @Override public boolean isLoggable(int priority, String tag) {
    return BuildConfig.DEBUG;
  }
});
```

## 将日志保存到文件中
// TODO：稍后会添加更多信息

```java
Logger.addLogAdapter(new DiskLogAdapter());
```
将自定义 TAG 添加到Csv格式策略

```java
FormatStrategy formatStrategy = CsvFormatStrategy.newBuilder()
  .tag("custom")
  .build();
  
Logger.addLogAdapter(new DiskLogAdapter(formatStrategy));
```

## 怎么运行的
![](/media/15337379584559.jpg)

## 更多

* 使用过滤器可获得更好的效果。 PRETTY_LOGGER或您的自定义标记
* 确保禁用换行选项
* 您还可以通过更改设置来简化输出

![](/media/15337379723935.jpg)


* Timber Integration
    
```java
// 将methodOffset设置为5以隐藏内部方法调用
Timber.plant(new Timber.DebugTree() {
  @Override protected void log(int priority, String tag, String message, Throwable t) {
    Logger.log(priority, tag, message, t);
  }
});
```
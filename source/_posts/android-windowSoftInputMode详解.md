---
title: 'android:windowSoftInputMode详解'
date: 2017-12-07 11:25:56
categories: Android
tags: Android
---

Activity 的主窗口与包含屏幕软键盘的窗口的交互方式。 该属性的设置影响两个方面：

* 当 Activity 成为用户注意的焦点时软键盘的状态 — 隐藏还是可见。
* 对 Activity 主窗口所做的调整 — 是否将其尺寸调小以为软键盘腾出空间，或者当窗口部分被软键盘遮挡时是否平移其内容以使当前焦点可见。

该设置必须是下表所列的值之一，或者是一个“state...”值加上一个“adjust...”值的组合。 在任一组中设置多个值（例如，多个“state...”值）都会产生未定义结果。各值之间使用垂直条 (|) 分隔。 例如：

```
<activity android:windowSoftInputMode="stateVisible|adjustResize" . . . >
```

此处设置的值（`stateUnspecified`和`adjustUnspecified`除外）替换主题中设置的值。


* `stateUnspecified`
    不指定软键盘的状态（隐藏还是可见）。<br/>将由系统选择合适的状态，或依赖主题中的设置。这是对软键盘行为的默认设置。

* `stateUnchanged`
    当 Activity 转至前台时保留软键盘最后所处的任何状态，无论是可见还是隐藏。
* `stateHidden`
    当用户选择 Activity 时 — 也就是说，当用户确实是向前导航到 Activity，而不是因离开另一 Activity 而返回时 — 隐藏软键盘。
* `stateAlwaysHidden`
    当 Activity 的主窗口有输入焦点时始终隐藏软键盘。
* `stateVisible`
    在正常的适宜情况下（当用户向前导航到 Activity 的主窗口时）显示软键盘。
* `stateAlwaysVisible`
    当用户选择 Activity 时 — 也就是说，当用户确实是向前导航到 Activity，而不是因离开另一 Activity 而返回时 — 显示软键盘。
* `adjustUnspecified`
    	不指定 Activity 的主窗口是否调整尺寸以为软键盘腾出空间，或者窗口内容是否进行平移以在屏幕上显露当前焦点。 系统会根据窗口的内容是否存在任何可滚动其内容的布局视图来自动选择其中一种模式。 如果存在这样的视图，窗口将进行尺寸调整，前提是可通过滚动在较小区域内看到窗口的所有内容。这是对主窗口行为的默认设置。
* `adjustResize`
    始终调整 Activity 主窗口的尺寸来为屏幕上的软键盘腾出空间。
* `adjustPan`
    不调整 Activity 主窗口的尺寸来为软键盘腾出空间， 而是自动平移窗口的内容，使当前焦点永远不被键盘遮盖，让用户始终都能看到其输入的内容。 这通常不如尺寸调正可取，因为用户可能需要关闭软键盘以到达被遮盖的窗口部分或与这些部分进行交互。

该属性是在 API 级别 3 引入的。

### 引入的版本：
API 级别 1，为 `noHistory` 和 `windowSoftInputMode` 之外的所有属性引入，这两个属性则是在 API 级别 3 中增加。


---
title: javascript 模板字符串
tags: javascript
categories: javascript
abbrlink: 7f9876e8
date: 2017-10-09 14:44:25
---

模板字面量/Template literals 是允许嵌入表达式的字符串字面量。你可以使用多行字符串和字符串插值功能。它们在ES2015规范的先前版本中被称为“模板字符串/template strings”。

## 语法


```javascript
`string text`

`string text line 1
 string text line 2`

`string text ${expression} string text`

tag `string text ${expression} string text`
```

> Note: 模板字面量也可以使用三元运算符( condition ?  true : false ) 和  嵌套 nested！

## 描述

模板字符串使用反引号 (\` \`) 来代替普通字符串中的用双引号和单引号。模板字符串可以包含特定语法(`${expression}`)的占位符。占位符中的表达式和周围的文本会一起传递给一个默认函数，该函数负责将所有的部分连接起来，如果一个模板字符串由表达式开头，则该字符串被称为带标签的模板字符串，该表达式通常是一个函数，它会在模板字符串处理后被调用，在输出最终结果前，你都可以通过该函数来对模板字符串进行操作处理。在模版字符串内使用反引号（\`）时，需要在它前面加转义符（\）。


```javascript
`\`` === "`" // --> true
```

### 多行字符串

在新行中插入的任何字符都是模板字符串中的一部分，使用普通字符串，你可以通过以下的方式获得多行字符串：


```javascript
console.log("string text line 1\n\
string text line 2");
// "string text line 1
// string text line 2"
```

要获得同样效果的多行字符串，只需使用如下代码：

```javascript
console.log(`string text line 1
string text line 2`);
// "string text line 1
// string text line 2"
```

### 表达式插补
在普通字符串中嵌入表达式，必须使用如下语法：


```javascript
var a = 5;
var b = 10;
console.log("Fifteen is " + (a + b) + " and\nnot " + (2 * a + b) + ".");
// "Fifteen is 15 and
// not 20."
```
现在通过模板字符串，我们可以使用一种更优雅的方式来表示：

```javascript
var a = 5;
var b = 10;
console.log(`Fifteen is ${a + b} and\nnot ${2 * a + b}.`);
// "Fifteen is 15 and
// not 20."
```

### 带标签的模板字符串
模板字符串的一种更高级的形式称为带标签的模板字符串。它允许您通过标签函数修改模板字符串的输出。标签函数的第一个参数是一个包含了字符串字面值的数组（在本例中分别为“Hello”,“world”和""）；第二个参数，在第一个参数后的每一个参数，都是已经被处理好的替换表达式（在这里分别为“15”和“50”）。 最后，标签函数返回处理好的字符串。在下面的例子中，命名这个标签并没有什么特殊的地方，这个函数的名字可以是任何你想要的。


```javascript
var a = 5;
var b = 10;

function tag(strings, ...values) {
  console.log(strings[0]); // "Hello "
  console.log(strings[1]); // " world "
  console.log(strings[2]); // ""
  console.log(values[0]);  // 15
  console.log(values[1]);  // 50

  return "Bazinga!";
}

tag`Hello ${ a + b } world ${ a * b}`;
// "Bazinga!"
```

正如下面例子所展示的，标签函数并不一定需要返回一个字符串。


```javascript
function template(strings, ...keys) {
  return (function(...values) {
    var dict = values[values.length - 1] || {};
    var result = [strings[0]];
    keys.forEach(function(key, i) {
      var value = Number.isInteger(key) ? values[key] : dict[key];
      result.push(value, strings[i + 1]);
    });
    return result.join('');
  });
}

var t1Closure = template`${0}${1}${0}!`;
t1Closure('Y', 'A');  // "YAY!" 
var t2Closure = template`${0} ${'foo'}!`;
t2Closure('Hello', {foo: 'World'});  // "Hello World!"
```
### 原始字符串

在标签函数的第一个参数中，存在一个特殊的属性raw ，我们可以通过它来访问模板字符串的原始字符串。


```javascript
function tag(strings, ...values) {
  console.log(strings.raw[0]); 
  // "string text line 1 \\n string text line 2"
}

tag`string text line 1 \n string text line 2`;
```

另外，使用`String.raw() `方法创建原始字符串和使用默认模板函数和字符串连接创建是一样的。


```javascript
String.raw`Hi\n${2+3}!`;
// "Hi\\n5!"
```




---
title: Kotlin 创建数组的正确方式
---
# Kotlin 创建数组的正确方式

数组在 Java 中构建十分的简单，可是到了 Kotlin 这里，很多人（包括我）在初期却出现了不少问题。

我来总结一下，顺便写一下我之前犯过的错误。

本文暂时没写协变逆变的不同，先写点比较基础的，如果有时间会再补。

本文结合 Kotlin 和 Java，方便大家区分其中的区别。

## 基本类型数组

在 Java 中，创建一个基本类型数组非常的简单：

```java
int[] arr1 = new int[2]; // [0, 0]
char[] arr2 = new char[] {'a', 'b', 'c'}; // ['a', 'b', 'c']
boolean[] arr3 = {true}; // [true]
```

了解 Kotlin 的一般都知道，在 Kotlin 中，数组创建不采用`T[]`，而是`Array<T>`。

### 错误用法

**以`Int`数组为例，假设你需要创建一个对标 Java 中`int[]`的数组**

#### 其一

你一想，Java 的非空的`Integer`和`int`在 Kotlin 中不都统一成`Int`了吗？我直接

```kotlin
val arr = Array<Int>(2)
```

怎么报错了，一看`Array`类的构建参数有两个

```kotlin
public class Array<T> {
    public inline constructor(size: Int, init: (Int) -> T)
}
```

这不简单？Java 里不是创建`int`数组初始化自动补 0 吗？我给他填满 0 不就行了？

```kotlin
val arr = Array<Int>(2) { 0 }
```

#### 其二

Kotlin 提供了`arrayOf<T>(vararg T)`方法来构建数组

```kotlin
public inline fun <reified @PureReifiable T> arrayOf(vararg elements: T): Array<T>
```

我直接

```kotlin
val arr = arrayOf(1, 2, 3)
```

之后测试了一番，发现这两种方法确实不会报错，而且数组也能正常使用。但这增加了非常多的开销。

### 错误分析

分析一下这段代码

```kotlin
val arr = Array<Int>(2) { 0 }
```

把这反编译成 Java 再看看

```java
Integer[] var2 = new Integer[2];

for (int var3 = 0; var3 < 2; ++var3) {
	Integer var8 = 0;
	var2[var3] = var8;
}
```

显而易见，用这种方法创建的是`Integer[]`而不是`int[]`，而且后面补 0 的操作更是多此一举。

第二个方法也是错误的，`arrayOf<T>(vararg T)`方法返回值是`Array<T>`，本质和上者的反编译是差不多的，用这种方法创建的是`Integer[]`而不是`int[]`。

用这种方法确实能起到假性`int[]`的作用，因为由于 Kotlin 严格的判空，所以你不能往这个数组添加非空元素。但是你若要用真正的`int[]`，一定不要采用这种方法。

### 正确使用

**以基本类型`int`为例，别的也都一样，改个名就是了**

在 Kotlin 中，提供了以下方法用来构建基本类型数组

```kotlin
public class IntArray(size: Int)
public class IntArray(size: Int, init: (Int) -> Int)
public fun intArrayOf(vararg elements: Int): IntArray
```

举例

```kotlin
val arr1 = IntArray(2)
val arr2 = intArrayOf(1, 2, 3)
```

对应 Java

```java
int[] arr1 = new int[2];
int[] arr2 = new int[] {1, 2, 3};
```

这方法才叫真正的不拖泥带水，给你真正的基本类型数组。

要是想构建元素可空的`Integer`数组怎么办，别急，这就是`Array<T>`的范畴了，一会就会说。

## 对象数组

**以 String 为例**

在 Java 中，创建一个对象数组

```java
String[] arr = new String[2]; // 1
String[] arr1 = new String[] {"Ace Taffy"}; // 2
```

### 错误用法

从上文可知，Kotlin 提供了`arrayOf<T>(vararg T)`方法来构建对象数组，如果创建像上者 2 这样的数组，因为 String 是一个类，不是基本数据类型，直接这样即可。

```kotlin
val arr1 = arrayOf("Ace Taffy")
```

但如果你想要上者 1 这样的数组，你该如何去做呢？

一般肯定又会想到之前提到的 Array 类，我们知道 Java 对象数组初始化自动补 null，然后这样构建

```kotlin
val arr = Array<String?>(2) { null }
```

### 错误分析

Array 类的构造器后的`init: (Int) -> T`参数会导致一轮不必要的循环，让我们再看一下反编译的结果

```java
String[] var2 = new String[2]; // 本身就已经是 [null, null]

// 再给它循环赋值 null，完全没必要
for (int var3 = 0; var3 < 2; ++var3) {
	Object var8 = null;
	var2[var3] = (String)var8;
}
```

### 正确用法

其实官方也早有对策，有专门的方法来创建上者 1 这样的数组

```kotlin
public fun <reified @PureReifiable T> arrayOfNulls(size: Int): Array<T?>
```

举例

```kotlin
val arr = arrayOfNulls<String>(2)
```

反编译后

```java
String[] var1 = new String[2];
```

完美符合需求。

上文说的构建元素可空的`Integer`数组，用该方法就可以实现。

```kotlin
val arr = arrayOfNulls<Int>(2)
```

如果你需要一个长度为 0 的元素非空数组，官方也给你提供了一个方法

```kotlin
public inline fun <reified @PureReifiable T> emptyArray(): Array<T> =
        @Suppress("UNCHECKED_CAST")
        (arrayOfNulls<T>(0) as Array<T>)
```

## 总结

以一个大的例子来总结一下，先用 Java 描述基本要求，再用 Kotlin 改写

``` java
// 1, [' ', ' ']
char[] arr1 = new char[2];
// 2, ['a', 'a']
char[] arr2 = new char[] {'a', 'a'};
// 3, [0, 2, 4]
int[] arr3 = new int[3];
for (int i = 0; i < arr3.length; ++i) {
    arr3[i] = i * 2;
}

// 4, [null, null]
String[] arr4 = new String[2];
// 5, ["Ace Taffy"]
String[] arr5 = new String[] {"Ace Taffy"};
// 6，"!"意思是永不为null，Java中没这个概念也不能实现, []
String![] arr6 = new String![0];
// 7, ["0", "2"]
String[] arr7 = new String[2];
for (int i = 0; i < arr7.length; ++i) {
    arr7[i] = String.valueOf(i * 2);
}
// 8, ["Ace Taffy", "Ace Taffy", "Ace Taffy"]
String[] arr8 = new String[3];
for (int i = 0; i < arr8.length; ++i) {
    arr8[i] = "Ace Taffy";
}
```

```kotlin
// 1
val arr1 = CharArray(2)
// 2
val arr2 = charArrayOf('a', 'a')
// 3
val arr3 = IntArray(3) { i -> i * 2 }

// 4
val arr4 = arrayOfNulls<String>(2)
// 5，也可以用 8 的方式。
val arr5 = arrayOf<String or String?>("Ace Taffy")
// 6
val arr6 = emptyArray<String>()
// 7，如需要该数组为元素非空数组，String? 也可以改成 String，
// 保证数组内的元素不为 null 是 Kotlin 特有，Java 不能实现。
// Array<String> 和 Array<String?> 在 Java 中都是 String[].
val arr7 = Array<String or String?>(2) { i -> (i * 2).toString() }
// 8，同 7
val arr8 = Array<String or String?>(3) { "Ace Taffy" }
```


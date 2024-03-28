---
title: 一起读 JEP430 —— Java 终于要有字符串模板了
---
# 一起读 JEP430 —— Java 终于要有字符串模板了

> 本文大部分是由 [JEP430](https://openjdk.org/jeps/430) 翻译而来，版权归原作者所有。

```java
void main() {
	var a = 1;
	var b = 2;
	var c = a + b;
	var str = STR."\{a} + \{b} = \{c}";
	System.out.println(str); // 输出结果：1 + 2 = 3
}
```

JDK21 确实是添加了不少新鲜的东西，其中最让我“耳目一新”的就是这个**字符串模板**。为什么加引号呢，因为在现代语言当中，你确实很少再见到这种不能在字符串中内嵌变量的语言了。当你每次就想加个变量在中间，就得多加两个引号和加号，一个忍忍就过去了，要是有好多个的话，想想就头疼。

字符串模板在 JDK21 中也只算个**预览**功能，所以说在正式生产环境能用到得猴年马月了。但是我为什么这么想让大家一起看一下 Java 的字符串模板呢？一是虽然说这是 JEP 提案，但解释的比较详细，可以当作一个简单的使用文档；二是分析的很到位，因为字符串基本是大家第一次接触的东西，这篇提案能够让新手逐步深入探究自己做的东西的可行性和拓展性。

我们在例子中可以看到，“莫名其妙”的`STR.`，“怪异”的`\{}`。到底是什么原因才使 Java 选择了看起来如此“奇怪”的写法呢？这种写法相比其他语言，比如 Kotlin，又有何优势呢？

别急，让我们把 JEP430 从头到尾读一遍。

## 概要

用字符串模板来增强 Java 编程语言的便利性。字符串模板通过将文本与嵌入的表达式和模板处理器耦合来产生专门的结果，从而补充 Java 现有的字符串文本（`"foo"`）和文本块（`"""bar"""`）。

## 目标

- 让包含运行时计算值的字符串易于表达从而简化 Java 程序的书写难度。
- 无论文本是在单个源行上（如字符串文字）还是跨越多个源行（如文本块），都能增强混合文字和表达式的表达式的可读性。
- 通过支持模板及其嵌入表达式的值的验证和转换，提高从用户提供的值组成字符串并将它们传递给其他系统（例如，构建数据库查询）的 Java 程序的安全性。
- 通过允许 Java 库定义字符串模板中使用的格式化语法来保持灵活性。
- 简化接受用非 Java 语言（如 SQL、XML 和 JSON）编写的字符串的API的使用。
- 允许创建根据文字文本和嵌入表达式计算的非字符串值，而不必通过中间字符串表示形式。

## 非目标

- 我们并不是想为 Java 的字符串连接运算符（+）引入语法糖，因为这会绕过验证的目标。
- 我们并不是想弃用或删除传统上用于复杂或编程字符串组合的 StringBuilder 和 StringBuffer 类。

## 动机

开发人员通常根据文本和表达式的组合来编写字符串。Java 为字符串组合提供了几种机制，但实际用起来怎么都麻烦。

先是最经典的**连接运算符（+）**：

```java
String s = x + " plus " + y + " equals " + (x + y);
```

简单的字符串还好，但凡有点复杂的（比如 SQL）会繁琐到直骂娘。

再是冗长的 **StringBuilder**：

```java
String s = new StringBuilder()
                 .append(x)
                 .append(" plus ")
                 .append(y)
                 .append(" equals ")
                 .append(x + y)
                 .toString();
```

**String::format** 和 **String::formatted** 的问题在于它们将格式字符串与参数分开了，可能导致数量和类型不匹配：

```java
String s = String.format("%2$d plus %1$d equals %3$d", x, y, x + y);
String t = "%2$d plus %1$d equals %3$d".formatted(x, y, x + y);
```

而 **java.text.MessageFormat** 需要太多“仪式”，然后用起来的语法也不是让人很熟悉：

```java
MessageFormat mf = new MessageFormat("{0} plus {1} equals {2}");
String s = mf.format(x, y, x + y);
```

### 字符串插值

许多编程语言都提供了字符串插值作为字符串连接的替代方案。通常采用字符串文本的形式，其中包含嵌入表达式和文本。在原位嵌入表达式，例如文章开头的那样，读者可以更容易的看出预期的结果。在运行时，嵌入的表达式将替换为它们的 toString 值——这些值被称为插值到字符串中。这里有几个其他编程语言插值的例子：

```c#
$"{x} plus {y} equals {x + y}"
```

```vb
$"{x} plus {y} equals {x + y}"
```

```python
f"{x} plus {y} equals {x + y}"
```

```scala
s"$x plus $y equals ${x + y}"
```

```groovy
"$x plus $y equals ${x + y}"
```

```kotlin
"$x plus $y equals ${x + y}"
```

```javascript
`${x} plus ${y} equals ${x + y}`
```

```ruby
"#{x} plus #{y} equals #{x + y}"
```

```swift
"\(x) plus \(y) equals \(x + y)"
```

这些语言中的部分语言允许对所有字符串文字进行插值，而另一些则要求在需要时启用插值功能，比如说通过在字符串分隔符前面添加`$`或者`f`。嵌入表达式的语法也各不相同，但通常包含了`$`或者`{ }`，这意味着想用这些字符必须进行转义。

插值不仅让写字符串连接的时候更省事，同时呈现出了更清晰的代码。这种清晰在复杂字符串中尤其显著。接下来举一个 JavaScript 的例子：

```javascript
const title = "My Web Page";
const text  = "Hello, world";

var html = `<html>
              <head>
                <title>${title}</title>
              </head>
              <body>
                <p>${text}</p>
              </body>
            </html>`;
```

### 可是字符串插值也有危险

字符串插值即使很便利，但仍有缺陷：很容易构造出一些将会被其他系统解释，但是对那个系统来说是严重错误的字符串。

包含 SQL 语句、HTML/XML 文档、JSON 片段、shell 脚本和自然语言文本的字符串都需要根据特定于域的规则进行验证（validate）和净化（sanitize）。由于 Java 不可能强制执行所有这些规则，所以由开发人员使用插值来验证和净化。通常意味着要记住在调用中包装嵌入的表达式以转义（`escape`）或验证（`validate`）方法，并依靠 IDE 或静态分析工具来帮助验证文本。

插值在 SQL 语句中尤其危险，因为稍有不慎就有可能造成**注入攻击**。接下来举一个例子，首先假设 Java 的插值使用的是`${ }`模式：

```java
String query = "SELECT * FROM Person p WHERE p.last_name = '${name}'";
ResultSet rs = connection.createStatement().executeQuery(query);
```

如果有个人把 name 传入这样的形式

`Smith' OR p.last_name <> 'Smith`

最后`query`这个字符串就会成这样

`SELECT * FROM Person p WHERE p.last_name = 'Smith' OR p.last_name <> 'Smith'`

可以看出这是个典型的注入攻击，本来我们想着要选择符合姓为某的列，结果最后变成了选择所有列。这样很不安全，容易暴露机密信息。然而旧的字符串连接法同样不安全：

```java
String query = "SELECT * FROM Person p WHERE p.last_name = '" + name + "'";
```

### 还能做的更好一点吗？

我们希望 Java 有一个既可以实现插值的清晰性，又可以开箱即用地获得更安全的结果的字符串组合功能，也许可以牺牲少量的便利性来获得大量的安全性。

例如，在编写SQL语句时，嵌入表达式的值中的任何引号都必须转义，并且整个字符串必须具有平衡的引号。上面那个 SQL 语句，应该这么写才是安全的：

`SELECT * FROM Person p WHERE p.last_name = '\'Smith\' OR p.last_name <> \'Smith\''`

几乎每一次使用字符串插值都涉及到构建适应某种模板的字符串：

- SQL 一般遵循模板`SELECT ... FROM ... WHERE ...`
- HTML 文档一般遵循模板`<html> ... </html>`
- 自然语言消息也遵循在文本中穿插动态值（例如用户名）的模板。

每种模板都有用于验证和转换的规则：

- SQL 语句——“转义所有引号”
- HTML 文档——“只允许合法字符实体”
- 自然语言消息——“根据操作系统中配置的语言本地化”。

理想情况下，字符串的模板可以直接在代码中表示，就像注释字符串一样，Java 运行时会自动将模板特定的规则应用于字符串。结果将是带有转义引号的 SQL 语句，没有非法实体的 HTML 文档以及本地化的、无样板的消息。从模板组合字符串将使开发人员不必费力地转义每个嵌入表达式，在整个字符串上调用`validate()`，或使用 java.util.ResourceBundle 查找本地化字符串。

再举一个例子，我们可能会构造一个表示 JSON 文档的字符串，然后将其提供给 JSON 解析器以获得强类型的 JSONObject：

```javascript
String name    = "Joan Smith";
String phone   = "555-123-4567";
String address = "1 Maple Drive, Anytown";
String json = """
    {
        "name":    "%s",
        "phone":   "%s",
        "address": "%s"
    }
    """.formatted(name, phone, address);

JSONObject doc = JSON.parse(json);
... doc.entrySet().stream().map(...) ...
```

理想情况下，字符串的 JSON 结构可以直接在代码中表达，Java 运行时会自动将字符串转换为 JSONObject。手动调用解析器是不必要的。

综上所述，如果我们有一个一流的、基于模板的字符串编写机制，几乎可以提高每个 Java 程序的可读性和可靠性。这种功能将会提供插值的优势，就像在其他编程语言中看到的那样，并且更不容易引入安全漏洞。它还会减少一些需要提供复杂输入作为字符串的字符串处理库的“仪式”。

## 说明

模板表达式是 Java 编程语言中一种新的表达式。模板表达式可以执行字符串插值，但也可以通过一种帮助开发人员安全高效地编写字符串的方式进行编程。此外，模板表达式并不局限于组成字符串——它们可以根据特定于域的规则将结构化文本转换为任何类型的对象。

从语法上讲，模板表达式类似于带前缀的字符串文字。下面展示代码的第二行就是一个模板表达式：

```java
String name = "Joan";
String info = STR."My name is \{name}";
assert info.equals("My name is Joan");   // true
```

模板表达式`STR."My name is \{name}"`由以下三点构成：

1. `STR`——模板处理器。
2. `.`(U+002E)——就是个点。
3. `"My name is \{name}"`——一个包含嵌入表达式的模板。

当在运行时计算模板表达式时，其模板处理器会将模板中的文字文本与嵌入表达式的值相结合，从而生成结果。模板处理器的结果以及对模板表达式求值的结果通常是一个字符串，只是通常，也有其他情况。

### STR 模板处理器

`STR`是由 Java 平台定义的一个模板处理器。它通过将模板中的每个嵌入表达式替换为该表达式的 toString 值来执行字符串插值。

在日常对话中，我们可能会在提及整个模板表达式（包括模板处理器）或仅提及模板表达式的模板部分（模板处理器的参数）时都会使用“模板”一词。只要注意不要混淆这些概念，这种非正式的用法是没什么问题的。

`STR`是一个`public static final`字段，**会被自动导入进所有 Java 源文件**。

这里有几个使用 STR 模板处理器的模板表达式例子（符号“|”后面是上一条语句的输出结果，类似 jshell）：

```java
// Embedded expressions can be strings
String firstName = "Bill";
String lastName  = "Duck";
String fullName  = STR."\{firstName} \{lastName}";
| "Bill Duck"
String sortName  = STR."\{lastName}, \{firstName}";
| "Duck, Bill"

// Embedded expressions can perform arithmetic
int x = 10, y = 20;
String s = STR."\{x} + \{y} = \{x + y}";
| "10 + 20 = 30"

// Embedded expressions can invoke methods and access fields
String s = STR."You have a \{getOfferType()} waiting for you!";
| "You have a gift waiting for you!"
String t = STR."Access at \{req.date} \{req.time} from \{req.ipAddress}";
| "Access at 2022-03-25 15:34 from 8.8.8.8"
```

为了方便重构，可以在嵌入表达式中使用双引号字符，而无需将其转义为`\"`。这意味着嵌入表达式可以在模板表达式中和在模板表达式外一样出现，从而简化了从字符串连接（+）到模板表达式的切换。例如：

```java
String filePath = "tmp.dat";
File   file     = new File(filePath);
String old = "The file " + filePath + " " + file.exists() ? "does" : "does not" + " exist";
String msg = STR."The file \{filePath} \{file.exists() ? "does" : "does not"} exist";
| "The file tmp.dat does exist" or "The file tmp.dat does not exist"
```

为了增加可读性，一个嵌入表达式可以在源文件中多行书写，而不需要引入换行符。嵌入表达式的值在嵌入表达式的`\`的位置插入到结果中。然后，该模板在与`\`的同一行上继续。比如：

```java
String time = STR."The time is \{
    // The java.time.format package is very useful
    DateTimeFormatter
      .ofPattern("HH:mm:ss")
      .format(LocalTime.now())
} right now";
| "The time is 12:34:56 right now"
```

字符串模板表达式中嵌入的表达式的数量是没有限制的。嵌入的表达式是**从左到右**计算的，就像方法调用表达式中的参数一样。例如：

```java
// Embedded expressions can be postfix increment expressions
int index = 0;
String data = STR."\{index++}, \{index++}, \{index++}, \{index++}";
| "0, 1, 2, 3"
```

任何 Java 表达式都可以用作嵌入表达式，套娃也可以。比如：

```java
// Embedded expression is a (nested) template expression
String[] fruit = { "apples", "oranges", "peaches" };
String s = STR."\{fruit[0]}, \{STR."\{fruit[1]}, \{fruit[2]}"}";
| "apples, oranges, peaches"
```

这样套娃比较难读懂，因为太多`\`、`{}`和`""`混在一行了。所以最好这么写：

```java
String s = STR."\{fruit[0]}, \{
    STR."\{fruit[1]}, \{fruit[2]}"
}";
```

因为嵌入表达式没什么副作用，所以也可以把它拆出来：

```java
String tmp = STR."\{fruit[1]}, \{fruit[2]}";
String s = STR."\{fruit[0]}, \{tmp}";
```

### FMT 模板处理器

`FMT`是 Java 平台中定义的另一个模板处理器。`FMT`与`STR`类似，它执行插值，但它也解释出现在嵌入表达式左侧的格式说明符。格式说明符与 java.util.Formatter 中定义的相同。以下是区域表示例，由模板中的格式说明符整理：

```java
record Rectangle(String name, double width, double height) {
    double area() {
        return width * height;
    }
}
Rectangle[] zone = new Rectangle[] {
    new Rectangle("Alfa", 17.8, 31.4),
    new Rectangle("Bravo", 9.6, 12.4),
    new Rectangle("Charlie", 7.1, 11.23),
};
String table = FMT."""
    Description     Width    Height     Area
    %-12s\{zone[0].name}  %7.2f\{zone[0].width}  %7.2f\{zone[0].height}     %7.2f\{zone[0].area()}
    %-12s\{zone[1].name}  %7.2f\{zone[1].width}  %7.2f\{zone[1].height}     %7.2f\{zone[1].area()}
    %-12s\{zone[2].name}  %7.2f\{zone[2].width}  %7.2f\{zone[2].height}     %7.2f\{zone[2].area()}
    \{" ".repeat(28)} Total %7.2f\{zone[0].area() + zone[1].area() + zone[2].area()}
    """;
| """
| Description     Width    Height     Area
| Alfa            17.80    31.40      558.92
| Bravo            9.60    12.40      119.04
| Charlie          7.10    11.23       79.73
|                              Total  757.69
| """
```

### 确保安全

模板表达式`STR."..."`是调用 STR 模板处理器中`process`方法的捷径，可以说是个语法糖。下面是个熟悉的例子：

```java
String name = "Joan";
String info = STR."My name is \{name}";
```

这个与下列的代码是完全一致的：

```java
String name = "Joan";
StringTemplate st = RAW."My name is \{name}";
String info = STR.process(st);
```

其中`RAW`是生成未处理的 StringTemplate 对象的标准模板处理器。

模板表达式的设计，故意使其无法直接从嵌入表达式的字符串文字或文本块，转到插入表达式值的字符串。这可以防止危险的错误字符串在程序中传播。字符串文字由模板处理器处理，该处理器明确负责安全地插入和验证结果（字符串或其他）。因此，如果我们忘记使用 STR、RAW 或 FMT 等模板处理器，则会**报编译时错误**：

```java
String name = "Joan";
String info = "My name is \{name}";
| error: processor missing from template expression
```

### 语法及语义

模板表达式中的四种模板由其语法显示，语法从 TemplateExpression 开始：

```
TemplateExpression:
  TemplateProcessor . TemplateArgument

TemplateProcessor:
  Expression

TemplateArgument:
  Template
  StringLiteral
  TextBlock

Template:
  StringTemplate
  TextBlockTemplate

StringTemplate:
  Resembles a StringLiteral but has one or more embedded expressions,
    and can be spread over multiple lines of source code

TextBlockTemplate:
  Resembles a TextBlock but has one or more embedded expressions
```

Java 编译器检测术语`"..."`，然后根据嵌入表达式是否存在，来将其转化成一个 StringLiteral 或者 StringTemplate。相同地，编译器也会检测术语`"""..."""`，然后决定是解析成 TextBlock 还是 TextBlockTemplate。我们统一将这些术语的`...`部分称为字符串文字、字符串模板、文本块或文本块模板的内容。

强烈建议 IDE 在视觉上区分字符串模板和字符串文字，以及文本块模板和文本块。在字符串模板或文本块模板的内容中，IDE 应该在视觉上将嵌入的表达式与文字文本区分开来。

Java 语言区分字符串文字和字符串模板，区分文本块和文本块模板，主要是因为字符串模板或文本块模板的类型不是我们熟悉的 java.lang.String，而是 java.lang.StringTemplate，是一个接口，String并没有实现 StringTemplate。因此，当模板表达式的模板是字符串字面量或文本块时，Java 编译器会自动将模板表示的 String 转换为没有嵌入表达式的 StringTemplate。

1. 计算`.`左侧的表达式以获得嵌套接口 StringTemplate.Processor 的实例，即模板处理器。

2. 计算`.`右侧的表达式以获得 StringTemplate 的实例。

3. StringTemplate 实例被传递给 StringTemplate.Processor 实例的`process`方法来构成一个结果。

模板表达式的类型是 StringTemplate.Processor 实例的处理方法的返回类型。

模板处理器在**运行时执行**，而不是编译时，所以不能再编译时处理模板。也**无法获得源代码中模板中出现的确切字符**（比如`\{x}`，你并不能拿到那个`x`字符）。只有嵌入表达式的值可用，而不是嵌入表达式本身。

### 模板表达式中的字符串模板

使用字符串文字或文本块作为模板参数的能力，提高了模板表达式的灵活性。开发人员可以编写最初在字符串文本中具有占位符文本的模板表达式，比如说：

```java
String s = STR."Welcome to your account";
| "Welcome to your account"
```

不需要任何分隔符或者特殊前缀，就能一个接一个将表达式嵌入进去以创建一个字符串模板：

```java
String s = STR."Welcome, \{user.firstName()}, to your account \{user.accountNumber()}";
| "Welcome, Lisa, to your account 12345"
```

### 自定义模板处理器

我们之前看到模板处理器 STR 和 FMT，他们的使用方式看起来更像是通过字段访问的对象。这样想确实很好记，但更准确地说，模板处理器是一个对象，它是单方法接口 StringTemplate.Processor 的一个实例。这个对象实现了接口的唯一方法`process`，该方法接受一个 StringTemplate 参数并返回一个对象。像`STR`这样的`static`字段仅仅储存了该类的实例（实例存储在`STR`中的实际类中具有一个`process`方法，该方法执行适用于单例的无状态插值，因此使用了大写字段名。）

开发者想创建用于模板表达式的模板处理器是非常容易的。然而我们在讨论怎么创建模板处理器之前，还要了解一下什么是 StringTemplate。

StringTemplate 的实例表示在模板表达式中作为模板出现的字符串模板或文本块模板。看下面这代码：

```java
int x = 10, y = 20;
StringTemplate st = RAW."\{x} plus \{y} equals \{x + y}";
String s = st.toString();
| StringTemplate{ fragments = [ "", " plus ", " equals ", "" ], values = [10, 20, 30] }
```

结果有点出乎意料，也没见`10`，`20`，`30`插入进`" plus "`，`" equals "`啊？回想一下，模板表达式的目标之一就是提供安全的字符串组合。单纯让`StringTemplate::toString`简单地就把`"10"`，`" plus "`，`"20"`，`" equals "`和`"30"`连接成一个字符串并不能很好地贯彻我们的目标。相反，`toString`方法给我们展示了 StringTemplate 比较有用的两个部分：

- 文本片段（text fragments）——`"", " plus ", " equals ", ""`
- 值（values）——`10`, `20`, `30`

StringTemplate 类直接暴露了这些部分：

- `StringTemplate::fragments`返回字符串模板或文本块模板中嵌入表达式前后的文本片段列表：

  ```java
  int x = 10, y = 20;
  StringTemplate st = RAW."\{x} plus \{y} equals \{x + y}";
  List<String> fragments = st.fragments();
  String result = String.join("\\{}", fragments);
  | "\{} plus \{} equals \{}"
  ```

- `StringTemplate::values`返回一个值列表，这些值是按照嵌入表达式在源代码中出现的顺序求值而生成的。在当前示例中，这等效于`List.of(x, y, x + y)`。

  ```java
  int x = 10, y = 20;
  StringTemplate st = RAW."\{x} plus \{y} equals \{x + y}";
  List<Object> values = st.values();
  | [10, 20, 30]
  ```

字符串模板的`fragments()`在模板表达式的所有求值中都是常量，而`values()`则是每次求值都计算一遍的。例如：

```java
int y = 20;
for (int x = 0; x < 3; x++) {
    StringTemplate st = RAW."Adding \{x} and \{y} yields \{x + y}";
    System.out.println(st);
}
| ["Adding ", " and ", " yields ", ""](0, 20, 20)
| ["Adding ", " and ", " yields ", ""](1, 20, 21)
| ["Adding ", " and ", " yields ", ""](2, 20, 22)
```

使用`fragments()`和`values()`，我们可以通过往`StringTemplate.Processor::of`这个工厂方法里传入一个 lambda 表达式，来轻松地创建插值模板处理器：

```java
var INTER = StringTemplate.Processor.of((StringTemplate st) -> {
    String placeHolder = "•";
    String stencil = String.join(placeHolder, st.fragments());
    for (Object value : st.values()) {
        String v = String.valueOf(value);
        stencil = stencil.replaceFirst(placeHolder, v);
    }
    return stencil;
});

int x = 10, y = 20;
String s = INTER."\{x} plus \{y} equals \{x + y}";
| 10 plus 20 equals 30
```

我们可以利用每个模板代表片段和值的交替序列这一事实，通过从片段和值构建其结果，使这种插值模板处理器更加高效：

```java
var INTER = StringTemplate.Processor.of((StringTemplate st) -> {
    StringBuilder sb = new StringBuilder();
    Iterator<String> fragIter = st.fragments().iterator();
    for (Object value : st.values()) {
        sb.append(fragIter.next());
        sb.append(value);
    }
    sb.append(fragIter.next());
    return sb.toString();
});

int x = 10, y = 20;
String s = INTER."\{x} plus \{y} equals \{x + y}";
| 10 and 20 equals 30
```

工具方法（Util 里的方法）`StringTemplate::interpolate`跟上面这个代码的效果是一致的，依次连接片段和值：

```java
var INTER = StringTemplate.Processor.of(StringTemplate::interpolate);
```

鉴于嵌入式表达式的值通常是不可预测的，对于模板处理器来说，将其生成的字符串进行内部化（intern）通常是不值得的。例如，STR 不会对其结果进行内部化。但是，如果需要，创建一个内部化和插值的模板处理器是很简单的。

```java
var INTERN = StringTemplate.Processor.of(st -> st.interpolate().intern());
```

### 模板处理器 API

咱们之前所有的例子都用了工厂方法 `StringTemplate.Processor::of`创建了模板处理器。这些例子处理器返回了 String 的实例而且不抛出任何异常，所以用它们的模板表达式求值永远不会失败。

相比之下，直接实现 StringTemplate.processor 接口的模板处理器可以是完全通用的。它不仅能返回 String，还能返回除了 String 的任何类型。而且如果处理发生了错误，或者模板不合法，或者其他原因，比如 I/O 错误，可以抛出检查异常。如果一个处理器抛出了检查异常，在模板表达式中用这个处理器的开发者就得用 try-catch 来处理错误，要不就传播异常给调用者。

String Template.Processor 接口的声明是：

```java
package java.lang;
public interface StringTemplate {
    ...
    @FunctionalInterface
    public interface Processor<R, E extends Throwable> {
        R process(StringTemplate st) throws E;
    }
    ...
}
```

之前咱们写过这样的插值字符串的代码：

```java
var INTER = StringTemplate.Processor.of(StringTemplate::interpolate);
...
String s = INTER."\{x} plus \{y} equals \{x + y}";
```

等同于：

```java
StringTemplate.Processor<String, RuntimeException> INTER =
    StringTemplate.Processor.of(StringTemplate::interpolate);
...
String s = INTER."\{x} plus \{y} equals \{x + y}";
```

模板处理器 INTER 的返回类型是由第一个类型参数 String 指定。处理器抛出的异常由第二个类型参数指定，在这个例子里是 RuntimeException，因为这个处理器不抛出任何检查异常。

这里有个返回值为 JSONObject 而不是字符串的模板处理器：

```java
var JSON = StringTemplate.Processor.of(
        (StringTemplate st) -> new JSONObject(st.interpolate())
    );

String name    = "Joan Smith";
String phone   = "555-123-4567";
String address = "1 Maple Drive, Anytown";
JSONObject doc = JSON."""
    {
        "name":    "\{name}",
        "phone":   "\{phone}",
        "address": "\{address}"
    };
    """;
```

上面`JSON`的定义和下面这个是相同的：

```java
StringTemplate.Processor<JSONObject, RuntimeException> JSON =
    StringTemplate.Processor.of(
        (StringTemplate st) -> new JSONObject(st.interpolate())
    );
```

比较一下第一个类型参数 JSONObject 与上面给 INTER 的第一个类型自变量 String。

这个假设的 JSON 处理器永远看不到由`st.interpolate()`生成的字符串。然而，以这种方式使用`st.interpolate()`有可能将注入漏洞传播到 JSON 结果中。我们可以谨慎修改代码，首先检查模板的值，如果值不太对劲，就抛出检查异常 JSONException：

```java
StringTemplate.Processor<JSONObject, JSONException> JSON_VALIDATE =
    (StringTemplate st) -> {
        String quote = "\"";
        List<Object> filtered = new ArrayList<>();
        for (Object value : st.values()) {
            if (value instanceof String str) {
                if (str.contains(quote)) {
                    throw new JSONException("Injection vulnerability");
                }
                filtered.add(quote + str + quote);
            } else if (value instanceof Number ||
                       value instanceof Boolean) {
                filtered.add(value);
            } else {
                throw new JSONException("Invalid value type");
            }
        }
        String jsonSource =
            StringTemplate.interpolate(st.fragments(), filtered);
        return new JSONObject(jsonSource);
    };

String name    = "Joan Smith";
String phone   = "555-123-4567";
String address = "1 Maple Drive, Anytown";
try {
    JSONObject doc = JSON_VALIDATE."""
        {
            "name":    \{name},
            "phone":   \{phone},
            "address": \{address}
        };
        """;
} catch (JSONException ex) {
    ...
}
```

这个版本的处理器抛出一个检查异常，所以我们不能使用工厂方法`StringTemplate.Processor::of`创建它。相反，我们直接在右侧使用 lambda 表达式。这同样意味着我们不能在左侧使用 var，因为 Java 需要 lambda 表达式的显式目标类型。

为了提高效率，我们可以通过将模板的片段编译成一个带有占位符值的 JSONObject，并缓存结果来对这个处理器进行记忆化。如果下一次调用处理器使用相同的片段，那么它可以将嵌入式表达式的值注入到缓存对象的一个新的深拷贝中，这样就不会有中间字符串存在。

### 安全地组合和执行数据库查询

下面的模板处理器类 QueryBuilder 首先从字符串模板创建 SQL 查询字符串。然后，它根据该查询字符串创建一个 JDBC PreparedStatement 语句，并将其参数设置为嵌入表达式的值。

```java
record QueryBuilder(Connection conn)
  implements StringTemplate.Processor<PreparedStatement, SQLException> {

    public PreparedStatement process(StringTemplate st) throws SQLException {
        // 1. Replace StringTemplate placeholders with PreparedStatement placeholders
        String query = String.join("?", st.fragments());

        // 2. Create the PreparedStatement on the connection
        PreparedStatement ps = conn.prepareStatement(query);

        // 3. Set parameters of the PreparedStatement
        int index = 1;
        for (Object value : st.values()) {
            switch (value) {
                case Integer i -> ps.setInt(index++, i);
                case Float f   -> ps.setFloat(index++, f);
                case Double d  -> ps.setDouble(index++, d);
                case Boolean b -> ps.setBoolean(index++, b);
                default        -> ps.setString(index++, String.valueOf(value));
            }
        }

        return ps;
    }
}
```

如果我们为特定的 Connection 实例化这个假设的 QueryBuilder：

```java
var DB = new QueryBuilder(conn);
```

相比以下这种不安全，容易造成注入攻击的写法：

```java
String query = "SELECT * FROM Person p WHERE p.last_name = '" + name + "'";
ResultSet rs = conn.createStatement().executeQuery(query);
```

我们可以换成这种更安全的写法：

```java
PreparedStatement ps = DB."SELECT * FROM Person p WHERE p.last_name = \{name}";
ResultSet rs = ps.executeQuery();
```

你可能在想为什么不直接返回一个 ResultSet，比如`ResultSet rs = DB."SELECT ...";`，这样难道不更方便吗？**然而，让模板处理器触发可能长时间运行的操作来输出一个结果不太明智，触发可能产生副作用的操作，例如更新数据库也是不明智的。**强烈建议模板处理程序的作者专注于验证他们的输入，并编写一个为客户端提供最大灵活性的结果。

### 简化本地化

我们之前提到的 FMT 模板处理器，是模板处理器类 java.util.FormatProcessor 的一个实例。虽然 FMT 使用的是默认语言环境，但通过以不同的方式实例化类来为不同的语言环境创建模板处理器是很简单的。比如说，这些代码创建了一个泰语环境下的模板处理器：

```java
Locale thaiLocale = Locale.forLanguageTag("th-TH-u-nu-thai");
FormatProcessor THAI = new FormatProcessor(thaiLocale);
for (int i = 1; i <= 10000; i *= 10) {
    String s = THAI."This answer is %5d\{i}";
    System.out.println(s);
}
| This answer is     ๑
| This answer is    ๑๐
| This answer is   ๑๐๐
| This answer is  ๑๐๐๐
| This answer is ๑๐๐๐๐
```

### 简化资源包的使用

下面的模板处理器类 LocalizationProcessor 简化了资源包的使用。对于给定的区域设置，它将字符串映射到资源包中的相应属性。

```java
record LocalizationProcessor(Locale locale)
  implements StringTemplate.Processor<String, RuntimeException> {

    public String process(StringTemplate st) {
        ResourceBundle resource = ResourceBundle.getBundle("resources", locale);
        String stencil = String.join("_", st.fragments());
        String msgFormat = resource.getString(stencil.replace(' ', '.'));
        return MessageFormat.format(msgFormat, st.values().toArray());
    }
}
```

假设每个区域设置都有一个属性文件资源包：

```
# resources_en_CA.properties file
no.suitable._.found.for._(_)=\
    no suitable {0} found for {1}({2})

# resources_zh_CN.properties file
no.suitable._.found.for._(_)=\
    \u5BF9\u4E8E{1}({2}), \u627E\u4E0D\u5230\u5408\u9002\u7684{0}

# resources_jp.properties file
no.suitable._.found.for._(_)=\
    {1}\u306B\u9069\u5207\u306A{0}\u304C\u898B\u3064\u304B\u308A\u307E\u305B\u3093({2})
```

则程序可以基于属性来组成本地化字符串：

```java
var userLocale = Locale.of("en", "CA");
var LOCALIZE = new LocalizationProcessor(userLocale);
...
var symbolKind = "field", name = "tax", type = "double";
System.out.println(LOCALIZE."no suitable \{symbolKind} found for \{name}(\{type})");
```

并且模板处理器会将字符串映射到合适语言环境的资源包中的相应属性里：

```
no suitable field found for tax(double)
```

如果程序写了这一行：

```java
var userLocale = Locale.of("zh", "CN");
```

输出结果将是：

```
对于tax(double), 找不到合适的field
```

如果写了这一行：

```java
var userLocale = Locale.of("ja");
```

将会输出：

```
taxに適切なfieldが見つかりません(double)
```

## 备选方案

- 当一个字符串模板没有模板处理器时，我们可以简单地进行基本的插值。但是，这样做会违反安全性的目标。比如使用插值构造 SQL 查询诱惑太大了，但这会降低 Java 程序的安全性。总是要求一个模板处理器，可以确保开发者至少认识到字符串模板中可能存在领域特定规则。

- 模板表达式的语法，以模板处理器为首，并不是严格必要的。也可以将模板处理器作为`StringTemplate::process`的一个参数。例如：

  ```java
  String s = "The answer is %5d{i}".process(FMT); 
  ```

  让模板处理器出现在第一位是更好的，因为计算模板表达式的结果完全取决于模板处理器的操作。

- 对于嵌入表达式的语法，我们考虑过使用`${...}`，但是这需要在字符串模板上加一个标签（要么是一个前缀，要么是一个不同于`"`的分隔符），来避免与以前的旧代码冲突。我们也考虑过`[…]`和`(…)`，但是`[ ]`和`( )`很可能出现在嵌入表达式中，而`{ }`出现的可能性较小，所以从视觉上确定嵌入表达式的开始和结束会更容易。

- 将格式说明符内置到字符串模板中也是可能的，就像 C# 中这样：

  ```csharp
  var date = DateTime.Now; Console.WriteLine($"The time is {date:HH:mm}"); 
  ```

  但是这样做就需要在引入新的格式说明符时修改 Java 的语言规范。

## 风险和假设

java.util.FormatProcessor 的实现在很大程度上依赖于 java.util.Formatter，这可能需要大量地重写。

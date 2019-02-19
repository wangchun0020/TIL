# You Don't Know JS: Scope & Closures

# Chapter 2: Lexical Scope

在第1章中，我们将“作用域”定义为一组规则，这些规则控制引擎如何通过其标识符名称查找变量并在当前作用域内或在其中包含的任何 *嵌套作用域* 中查找它。

对于作用域是如何工作，这里有两种占有统治地位的模型。其中第一个是迄今为止最常见的，绝大多数编程语言都使用它们。它被称为 **词法作用域** ，我们将深入研究它。另一个模型仍然被一些语言（如bash脚本、Perl中的一些模式等）使用，称为 **动态作用域** 。

附录A中介绍了动态作用域。我在这里提到它只是为了与词法作用域进行对比，词法作用域是JavaScript所使用的作用域模型。

## Lex-time

正如我们在第1章中讨论的那样，标准语言编译器的第一个传统阶段称为 *lexing-词法分析*（就是 *分词*，也可叫 *令牌化*）。如果你还记得，词法分析过程将检查一系列源代码字符，并将语义意义分配给令牌，这是一些有状态解析的结果。

正是这个概念为理解词法作用域和名称来源提供了基础。

为了在一定程度上循环地定义它, 词法作用域是在 词法分析时定义的范围。换句话说，词法作用域基于你在编写时编写的变量和作用域块的位置，因此(大多数情况下)在词法分析器处理你的代码时就已经固定下来了。

**注意：** 我们会看到，有一些方法可以欺骗词法作用域，从而在词法程序通过后对其进行修改，但这些方法让人很不爽。实际上，最好的做法是将词法作用域视为仅用于词法的，因此在本质上完全是编写时决定的。

让我们考虑一下这段代码：

```js
function foo(a) {

	var b = a * 2;

	function bar(c) {
		console.log( a, b, c );
	}

	bar(b * 3);
}

foo( 2 ); // 2 4 12
```

在这个代码示例中有三个固有的嵌套作用域。将这些作用域看作彼此内部的气泡可能会有所帮助。

![](https://github.com/getify/You-Dont-Know-JS/raw/master/scope%20%26%20closures/fig2.png)

**气泡1** 包含全局作用域，其中只有一个标识符：`foo`。

**气泡2** 包含`foo`作用域，其中包括三个标识符：`a`，`bar`和`b`。

**气泡3** 包含`bar`的作用域，它只包含一个标识符：`c`。

作用域气泡是由写入作用域块的位置定义的，其中一个块嵌套在另一个块中，等等。在下一章中，我们将讨论不同的作用域单元，但是现在，让我们假设每个函数都创建了一个新的作用域气泡。

`bar`的气泡完全包含在`foo`的气泡中，因为（并且只是因为）我们选择定义函数`bar`的位置。

请注意，这些嵌套的气泡是严格嵌套的。我们不是在讨论气泡可以跨越边界的维恩图( Venn diagrams )。换句话说，某些函数的气泡不能同时(部分地)存在于其他两个外部作用域气泡中，就像没有函数可以部分地存在于两个父函数中一样。

### Look-ups

这些作用域气泡的结构和相对位置完全向引擎解释了查找标识符所需的所有位置。

在上面的代码片段中，引擎执行`console.log(..)`语句并查找三个引用的变量`a`，`b`和`c`。它首先从最里面的作用域气泡开始，即`bar(..)`函数的作用域。它在这里没有找到`a`，所以它上升了一个级别，到下一个最近的作用域气泡，`foo(..)`的作用域。它在这里找到了`a`，所以它使用了`a`。 对于`b`有同样的过程。但是`c`，它确实在`bar(..)`内找到了。

如果在`bar(..)`和`foo(..)`内部都有一个`c`，则`console.log(..)`语句会找到并使用`bar(..)`中的那个，永远不会到达`foo(..)`去寻找。

**一旦找到第一个匹配，范围查找就会停止。** 相同的标识符名称可以在嵌套作用域的多层指定，称为“阴影(shadowing)”(内部标识符“阴影”外部标识符)。无论阴影是什么，范围查找总是从当前执行的最内部的范围开始，并向外/向上工作，直到第一个匹配，然后停止。

**注意：** 全局变量也是全局对象的自动属性(浏览器中的`window`等)，因此可以不直接引用全局变量的词法名称，而是间接引用全局对象的属性引用。

```js
window.a
```

此技术允许访问全局变量，否则将无法访问该变量，因为该变量被隐藏。但是，无法访问非全局阴影变量。

无论函数在何处被调用，甚至如何调用它，其词法作用域 **仅** 由声明函数的位置定义。

词法作用域查找过程仅适用于第一类标识符，例如`a`，`b`和`c`。如果在一段代码中引用了`foo.bar.baz`，则词法作用域查找将查找`foo`标识符，但一旦找到该变量，则对象属性访问规则将接管，分别解析`bar`和`baz`属性。

## Cheating Lexical

如果词法作用域仅由声明函数的位置定义，这完全是写作时所决定，那么在运行时怎么可能有一种方法来“修改”（又称作弊）词法作用域？

JavaScript有两种这样的机制。在更广泛的社区中，它们都被认为是在代码中使用的不好的实践，因此都是不受欢迎的。但反对它们的典型论点往往忽略了最重要的一点: **欺骗词法作用域会导致较差的性能。**

不过，在我解释性能问题之前，让我们来看看这两种机制是如何工作的。

### `eval`

JavaScript中的eval(..)函数接受一个字符串作为参数，并将该字符串的内容当作是在程序中的那个时候编写的代码一样对待。换句话说，你可以在编写的代码中以编程方式生成代码，并且运行生成的代码，就好像它在编写时就在那里一样。

从这个角度来评估`eval(..)`(双关语)，应该可以清楚地看到`eval(..)`是如何通过欺骗和假装始终存在编写时的(也就是词法)代码来修改词法作用域环境的。

在`eval(..)`执行后的后续代码行中，引擎将“不知道”或“关心”前面的代码是否被动态解释，从而修改了词法作用域环境。引擎将像往常一样简单地执行其词法作用域查找。

请考虑以下代码：

```js
function foo(str, a) {
	eval( str ); // cheating!
	console.log( a, b );
}

var b = 2;

foo( "var b = 3;", 1 ); // 1 3
```

在`eval(..)`调用时，字符串`"var b = 3;"`被视为始终存在的代码。因为该代码恰好声明了一个新变量`b`，所以它修改了`foo(..)`的现有词法作用域。实际上，如上所述，此代码实际上在`foo(..)`中创建了变量`b`，它隐藏了在外部（全局）作用域内声明的`b`。

当发生`console.log(..)`调用时，它会在`foo(..)`的范围内找到`a`和`b`，并且永远不会找到外部的`b`。因此，我们通常会打印出"1 3"而不是"1 2"。

**注意：** 在这个例子中，为简单起见，我们传入的字符串“代码”是一个固定的文字。但是，根据程序的逻辑将字符添加到一起，可以很容易地通过编程方式创建它。`eval(..)`通常用于执行动态创建的代码，因为从字符串文本中动态地评估本质上是静态的代码不会给直接编写代码带来真正的好处。

默认情况下，如果`eval(..)`执行的代码字符串包含一个或多个声明（变量或函数），则此操作将修改`eval(..)`所在的现有词法作用域。从技术上讲，`eval(..)`可以通过各种技巧“间接”调用(超出我们在这里讨论的范围)，这将导致它在全局作用域的上下文中执行，从而修改它。但在任何一种情况下，`eval(..)`都可以在运行时修改编写时的词法作用域。

**注意：** 在严格模式程序中使用`eval(..)`时，它在自己的词法作用域中操作，这意味着在`eval()`中所做的声明实际上并不修改所包围的作用域。

```js
function foo(str) {
   "use strict";
   eval( str );
   console.log( a ); // ReferenceError: a is not defined
}

foo( "var a = 2" );
```

JavaScript中还有其他一些功能，其效果与`eval(..)`非常相似。`setTimeout(..)`和`setInterval(..)` *可以* 为各自的第一个参数接收一个字符串，其中的内容被`eval`动态生成的函数的代码进行计算。这是旧的、遗留的行为，早就被弃用了。不要这样做！

`new Function(..)`函数构造函数类似地在其 **最后** 一个参数中使用一串代码转换为动态生成的函数（第一个参数（如果有的话）是新函数的命名参数）。这个函数构造函数语法比`eval(..)`稍微安全一些，但在代码中仍然应该避免使用它。

在你的程序中动态生成代码的用例非常罕见，因为性能下降几乎不值得使用这种功能。

### `with`

JavaScript中另一个不受欢迎(现在已弃用了!)的欺骗词法作用域的功能是`with`关键字。`with`有多种有效的解释方法，但我将选择从它如何与词法作用域交互并影响词法作用域的角度来解释它。

`with`通常被解释为对对象进行多个属性引用而不每次重复对象引用本身的简写。

例如：

```js
var obj = {
	a: 1,
	b: 2,
	c: 3
};

// more "tedious" to repeat "obj"
obj.a = 2;
obj.b = 3;
obj.c = 4;

// "easier" short-hand
with (obj) {
	a = 3;
	b = 4;
	c = 5;
}
```

然而，这里要做的不仅仅是一个方便对象属性访问的简写。考虑:

```js
function foo(obj) {
	with (obj) {
		a = 2;
	}
}

var o1 = {
	a: 3
};

var o2 = {
	b: 3
};

foo( o1 );
console.log( o1.a ); // 2

foo( o2 );
console.log( o2.a ); // undefined
console.log( a ); // 2 -- Oops, leaked global!
```

在此代码示例中，创建了两个对象`o1`和`o2`。一个有`a`属性，另一个没有。`foo(..)`函数将对象引用`obj`作为参数，并在引用上使用`with (obj) { .. }`进行调用。在`with`块中，我们对变量`a`(实际上是一个LHS引用)进行常规的词法引用，并将其赋值为`2`。

当我们传入`o1`时，`a = 2`赋值找到属性`o1.a`并为其赋值`2`，如后续的`console.log(o1.a)`语句所反映的那样。但是，当我们传入`o2`时，因为它没有`a`属性，所以没有创建这样的属性，并且`o2.a`是`undefined`。

但是我们注意到一个特殊的副作用，即全局变量`a`是由`a = 2`赋值创建的。怎么会这样呢?

`with`语句接受一个具有零个或多个属性的对象，**并将该对象视为一个完全独立的词法作用域** ，因此将该对象的属性视为该“作用域”中词法定义的标识符。

**注意：** 尽管`with`块将对象视为词法作用域，但`with`块内部的普通`var`声明的作用域不会限定为`with`块，而是包含函数的作用域。

虽然`eval(..)`函数可以修改现有的词法作用域(如果它接受一串包含一个或多个声明的代码)，但是`with`语句实际上凭空创建了一个 **全新的词法作用域** ，从你传递给它的对象开始。

这样理解，当我们传入`o1`时，`with`语句声明的“作用域”是`o1`，而“作用域”中有一个与`o1.a`属性对应的“标识符”。但是当我们使用`o2`作为“作用域”时，它没有这样的`a`“标识符”，所以LHS标识符查找的正常规则(参见第1章)就出现了。

无论是`o2`的“作用域”，还是`foo(..)`的作用域，甚至全局作用域，都没有要找到的`a`标识符，因此当执行`a = 2`时，它会导致创建个全局变量(因为我们处于非严格模式)。

在运行时将对象及其属性转换为带有“标识符”的“作用域”是一种奇怪的思维扭曲。但对于我们看到的结果，这是我能给出的最清晰的解释。

**注意：**`eval(..)`和`with`都受到严格模式的影响(限制)，使用起来不仅不好。`with`是完全不允许的，而各种形式的间接或不安全的`eval(..)`在保留核心功能的同时是不允许的。

### Performance

`eval(..)`和`with` 在运行时通过修改或创建新的词法作用域来欺骗在其他情况下由编写时定义的词法作用域。

你会问，这有什么大不了的?如果它们提供更复杂的功能和编码灵活性，这些不是很好吗?**不。** 

JavaScript 引擎在编译阶段执行了许多性能优化。其中一些归结为能够基本上静态地分析代码，并预先确定所有变量和函数声明的位置，以便在执行期间解析标识符所需的工作量更少。

但如果引擎发现`eval(. .)`或`with`在代码中,它本质上必须假定所有的标识符位置可能是无效的,因为它在词法分析时无法知道什么代码可以传递给`eval(. .)`以修改词法作用域, 或你可以传递给`with`的对象内容创建一个新的词法作用域。

换句话说，在悲观意义上，如果`eval(..)`或`with`存在，那么它所做的大多数优化 *将* 是没有意义的，因此它 *根本* 不执行优化。

你的代码几乎肯定会因为代码中的任何位置包含`eval(..)`或`with`而变得更慢。无论引擎在试图限制这些悲观假设的副作用方面有多聪明，**都无法回避这样一个事实：没有优化，代码运行速度会变慢。** 

## Review (TL;DR)

词法作用域意味着作用域由函数声明位置的编写时所决定。编译的词法分析阶段基本上能够知道所有标识符在哪里以及如何声明，从而预测在执行期间如何查找它们。

JavaScript中的两种机制可以“欺骗”词法作用域：`eval(..)`和`with`。前者可以通过计算包含一个或多个声明的“代码”字符串来修改现有的词汇作用域（在运行时）。后者本质上创建了一个全新的词法作用域(同样是在运行时)，方法是将对象引用视为“作用域”，并将该对象的属性视为作用域标识符。

这些机制的缺点是它会破坏引擎对作用域查找执行编译时优化的能力，因为引擎必须悲观地假设这样的优化将是无效的。由于使用任一功能，代码运行速度 *将* 较慢。**不要使用它们。**
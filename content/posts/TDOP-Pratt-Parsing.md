---
title: 'TDOP Pratt Parsing'
date: 2023-05-06T15:52:47+08:00
draft: false
author: Zijian Zang
toc: true
tags: 
 - 
---

Top Down Operator Precedence（TDOP）是Vaughan R. Pratt在1973年POPL（Principles of Programming Languages Symposium）第一期年刊上提出的一种简洁、易于实现、高效、类似自顶向下的表达式分析方法。

该篇论文的电子版本维护于一个[github仓库](https://tdop.github.io/)。

<!--more-->

## Backus-Naur Form

一般使用巴科斯范式来描述上下文无关语法的解析方式。LR语法分析和LL语法分析全部基于BNF。

```
Item ::=
    StructItem
  | EnumItem
  | ...

StructItem ::= 
    'Struct' Name '{' FieldList '}'
```
但是这种描述方式有显著的缺点：模糊性，例如优先级不明确。类似下式的BNF
```
Expr ::=
    Expr '+' Expr
  | Expr '*' Expr
  | '(' Expr ')'
  | 'Number'
```

这种描述方式并没有体现出`*`的优先级高于`+`的特点。以`a + b * c`为例，它既可以分析为：
```
Expr1 -> Expr2 + Expr3
Expr2 -> a
Expr3 -> Expr4 * Expr5
Expr4 -> b
Expr5 -> c
```

即`a + (b * c)`，也可以分析为
```
Expr1 -> Expr2 * Expr3
Expr2 -> Expr4 + Expr5
Expr3 -> c
Expr4 -> a
Expr5 -> b
```

即`(a + b) * c`，两种分析方式都符合语法。要想唯一地确定分析方式，必须要为乘法和括号表达式建立额外的非终结符。

```
Expr ::= 
    Factor
  | Expr '+' Factor

Factor ::=
    Atom
  | Factor '*' Atom

Atom ::=
    Number
  | '(' Expr ')'
```

这种情况下，`a + (b * c)`才能被唯一分析

```
Expr1 -> Expr2 + Factor1
Expr2 -> Factor2
Factor1 -> Factor3 * Atom1
Factor2 -> Atom2
Factor3 -> Atom3
Atom1 -> a
Atom2 -> b
Atom3 -> c
```

可以看到，无论是文法本身还是解析过程，都变得十分复杂。而考虑到编程语言中常见的运算符有十种以上，实际情况下还会要复杂得多；同时，由于每添加一种运算符，都需要定义新的非终结符和语法规则，这使得语法的扩展性非常差。

相较而言，Pratt递归下降方法不基于BNF，能自然处理表达式优先级问题，能用相同的逻辑处理所有表达式分析的问题，不仅包括基本运算符，还包括函数调用、数组寻址、字典寻址等；同时，Pratt分析方法的扩展性非常好，新定义一钟算符时，只需要给定它的优先级即可。

Pratt分析法还有一个优点：不需要考虑左递归问题。

Pratt分析法的程序结构类似于

```go
func ParseExp() {
    // ...
    for {
        // ...
        ParseExp()
        // ...
    }
    // ... 
}
```

它在循环中无限递归，只在满足条件时跳出循环。

## 结合问题（association problem）

对于一个表达式

```
A + B * C
```

按一般的数学常识，显然应该先将`B`乘`C`，再与`A`相加，运算顺序应当为

```
(A + (B * C))
```

但机器是没有数学常识的。在一般情况下，解决这种问题会使用后缀表达式。但是在语法分析中，读入的token序列自然就是一个中缀表达式，没有简单转化为后缀表达式的方法。

我们可以发现，如果将`()`括起来的一部分看作一颗子树，那么只要明确了优先级关系，就可以直接构建表达式的AST，而不需要使用BNF进行分析。Pratt分析法就是基于优先级来进行的。

为了处理算符的优先级，提出结合性问题：给定子串`AEB`，其中`E`是一个表达式，`A`是一个缺少右操作数的表达式，`B`是一个缺少左操作数的表达式，那么`E`应该与`A`结合还是与`B`结合？

## 算符的吸引能力

对于`A + B * C`，可以将其看作三部分：`A + `、`B`、` * C`，对应了前文中的`AEB`格式。

对于这样的表达式，可以认为：`B`同时被左侧的`+`和右侧的`*`吸引，而`*`有着更强的吸引能力，所以结果为
```
A + (B * C)
```
`B`会参与到`*`的运算中而不是`+`的运算中，这就是Pratt分析法中对算符优先级的理解。

优先级是一个整型。如果定义`*`优先级是`4`，而`+`优先级是`3`，我们可以将表达式序列写为
```
expr:      A   +   B   *   C
power:   0   3   3   4   4   0
```
对于每个符号，其可以与左侧表达式和右侧表达式结合。哪个符号优先级高，就与哪个表达式结合。表达式末尾和开头定义为最低优先级。我们可以直观的看到`B`左右两侧算符的优先级分别为`3`和`4`，所以`B`将参与右侧算符的运算。

那么，对于一串具有相同算法的表达式序列该如何处理其优先级呢？
```
A + B + C
```
实际情况中，需要从左到右依次计算，为了实现这一操作，我们在定义操作符优先级时，令操作符左右侧优先级不对称，且右侧略高于左侧。
```
expr:      A   +   B   +   C
power:   0   3  3.1  3  3.1   0
```

由于第一个`+`对`B`的吸引力是`3.1`，所以`B`将参与第一个`+`表达式中，这就使得计算结果为
```
(A + B) + C
```

假如有些算符具有右结合性，只需要将其左侧的优先级定义为略高于右侧即可。

## 简化版Pratt分析

```go
func (p *Parser) ParseExp(precedence int) ast.Expression {
	prefix := p.prefixParseFns[p.peekToken().Type]
	if prefix == nil {
		p.noPrefixParseFnError(p.peekToken().Type)
		return nil
	}
	leftExp := prefix()
	for p.peekToken().Type != token.SEMICOLON && precedence < p.peekPrecedence() {
		infix := p.infixParseFns[p.peekToken().Type]
		if infix == nil {
			return leftExp
		}
		leftExp = infix(leftExp)
	}
	return leftExp
}
```
```go
func (p *Parser) ParseInfixExpression(left ast.Expression) ast.Expression {
	expression := &ast.InfixExpression{
		Token:    *p.peekToken(),
		Left:     left,
		Operator: p.peekToken().Literal,
	}
	precedence := p.peekPrecedence()
	p.nextToken()
	expression.Right = p.ParseExp(precedence)
	return expression
}
```

核心函数为`ParseExp`，它接受名为`precedence`的参数，其含义为`AEB`格式中`A`表达式的算符优先级。在循环语句中，它将与下一个符号，即右侧算符的优先级进行比较。如果左侧优先级小于右侧优先级，则`E`将参与右侧算符的运算，否则跳出循环。

以`a + b * c`为例，由于左侧无算符，初始优先级为`LOWEST`。首先解析标识符`a`。

```
Expression: [ 'a' ]
Parse Tree:
a
```

接下来将`LOWEST`与`+`优先级比较，`+`优先级高，则`a`参与到`+`运算中，进入`ParseInfixExp`函数。

```
Expression: [ 'a', '+' ]
Parse Tree:
+
├── a
└── Unparsed
```

在`ParseInfixExp`中，将递归地调用`ParseExp`产生右侧表达式，此时左侧算符优先级为`+`。解析`b`并比较`+`与`*`的优先级，发现`*`优先级较高，故`b`参与到右侧`*`中，最后解析`c`并返回，得到`a + (b * c)`。

```
Expression: [ 'a', '+', 'b']
+
├── a
└── b
```
```
Expression: [ 'a', '+', 'b', '*']
+
├── a
└── *
	├── b
	└── Unparsed
```
```
Expression: [ 'a', '+', 'b', '*', 'c']
+
├── a
└── *
	├── b
	└── c
```

如果初始表达式为`a + b + c`，则在解析`b`后直接返回，于是在原函数中得到`a + b`的结点，并继续解析`+`和`c`，最终得到`(a + b) + c`。
```
Expression: [ 'a']
a
```
```
Expression: [ 'a', '+']
+
├── a
└── Unparsed
```
```
Expression: [ 'a', '+', 'b']
+
├── a
└── b
```
```
Expression: [ 'a', '+', 'b', '+']
+
├── +
│	├── a
│	└── b
└── Unparsed
```
```
Expression: [ 'a', '+', 'b', '+', 'c']
+
├── +
│	├── a
│	└── b
└── c
```

## Pratt分析的泛用性

Pratt分析法可以解决运算符表达式的分析，而它的功能还不止于此。

例如，对于`x + a[2]`，Pratt如何识别数组寻址符号？

答案将`a[2]`看作操作符为`[`的中缀表达式，并令`[`算符的优先级高于`+`，则可以得到如下ast:
```
+
├── x
└── [
	├── a
	└── 2
```

假如在原表达式使用了`()`来自定义运算规则，如何在Pratt分析中体现这点？

答案是为`(`定义独特的前缀表达式分析函数，当分析到token`(`时，进入该特定函数，为括号内表达式调用`ParseExp`函数，并定义优先级为`LOWEST`，相当于另开一棵ast。

```
Expression: [ '(', 'a', '+', 'b', ')' , '*', 'c']
*
├── +
│	├── a
│	└── b
└── c
```

表达式中常常包含函数调用，Pratt分析如何处理这种情况？

答案是将`(`看作一个中缀表达式。

```
Expression: ['x' + 'double' + '(' + 'y' + ')']
+
├── x
└── (
	├── double
	└── params
		└── y
```

## 参考文献

[1]Vaughan R. Pratt.Top Down Operator Precedence[J].Principles of Programming Languages Symposium,1973(1):41-51
[2]matklad.Simple but Powerful Pratt Parsing[CP/OL].https://matklad.github.io/2020/04/13simple-but-powerful-pratt-parsing.html ,2020-4-13
[3]Thorsten Ball.Writing An Interpreter In Go[M].Thorsten Ball, 2018: 46-76

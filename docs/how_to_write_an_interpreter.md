# 如何写一个（Scheme）解释器

## 目标读者

菜鸟。

## 0.熟悉目标实现语言

请找相关Scheme教程 :d，[R5RS.PDF](schem-r5rscn.pdf)

## 1.解释器

&emsp;&emsp;我们要写的解释器，是做解释工作的一种程序，它能够理解代码的意义，并执行所要求的动作。

## 2.熟悉实现所用语言

&emsp;&emsp;在用一种语言实现另一种语言的解释器之前，你应该先熟悉实现所用语言。它就像用中文表达英语世界中的概念，必须先熟悉中文，然后才能选用合适的中文词语，去描述英语中对应的概念。  
如果你熟悉JavaScript（JS），那么至少应该理解对象，函数等等。我这里只介绍JS :)。


## 3.词法+语法分析

&emsp;&emsp;我们知道看到一段Scheme或者其它语言的文本时，我们会理解它是由不同的单词，并且会以语句，关键字等单位组成的。我们的第一步是语法分析，是从代码文本转换到一种数据结构的过程。
我们会对该解释器输入一段代码例如`(+ 1 2)`，这样的代码之前通常是通过输入到文本编辑器编辑得到的，它们存储在内存中或者编码为文本文件存在硬盘里，然后通过复制等程序会到达某个地址的内存中，在JS程序中被抽象成字符串变量。而我们知道，字符串是字符数组，是一维的，那么我们怎么取出运算符`+`，数字`1`和数字`2`呢？取子字符串：
```js
var str = "(+ 1 2)";
var operator = str.substring(1, 2);
var operand1 = str.substring(3, 4);
var operand2 = str.substring(5, 6);
```
一种更好的方法，用正则:
```js
var str = "(add 12 23)";
var m = str.match(/[\+\-\*\/\w]+|[\(\)]/g);
var operator = m[1];
for(var n = 1; n < m.length - 1; n++) {
    var operandN = m[n];
    console.log(operandN);
}
```
可是我们的Scheme程序不会像`(add 12 23)`这么简单，如何解析复杂表达式呢？例如下面这样的：
```scheme
; Scheme+过程调用表达式
(
  +         ; operator
  (* 3 5)   ; operand1
  (- 10 6)  ; operand2
            ; operandN
)
```
这个问题可以暂时放下，先考虑下面的问题。
我们应该将这些结果（`operator`，`operand1`和`operand2`）缓存起来，后续就不用再次取出，方便后续解释器模块进一步处理，去慢慢接近代码的意义。同样的这个原因，我们在此阶段需要输出`12`是一个数值，而`add`和`+`则是一个标识符。  
它们就像中文中的单词，那么我们该如何表示这样一个单词呢，以及它们之间的关系？我们可以看到，Scheme语法如此简洁统一，Scheme语言元素的关系在于它们在不在同一对括号里：`*`、`3`和`5`在同一对括号里，`-`、`10`和`6`在同一对括号里，以及`+`、`(* 3 5)`和`(- 10 6)`在同一对括号里，即它们是“粘”在一起的，operand1和operand2是整个+表达式单元中的两个子单元。稍加思考，我们发现可以用JS数组表示那样的关系：
```scheme
/* Scheme+过程调用表达式的JS数组表示 */
[
  '+',          // operator
  ['*', 3, 5],  // operand1
  ['-', 10, 6]  // operand2
                // operandN
]
```
上面这种表示法其实是一些人曾尝试过的做法，但是后来我们会发现，如果语法分析的结果是Scheme对象，那么对于`quote`运算符等重要特性的实现，仅仅只要返回Scheme对象即可，而不用去解释时转换。下面是quote的例子：
```scheme
; quote用法示例
(define l (quote (* 7 8 9)))  ; quote将(* 7 8 9)看作表的外部表示
(list? l)                     ; 判断是否一个表，返回#t
(map display l)               ; 输出l的4个元素：+、7、8、9和空表'()
(car l)                       ; 返回符号*
(cdr l)                       ; 返回表(7 8 9)
(quote +)                     ; 返回符号+
(symbol? (quote +))           ; 判断是否一个符号，返回#t
(quote 3)                     ; 返回数字3
(integer? (quote 3))          ; 判断是否一个整数，返回#t
```
我们将表达式`(+ (* 3 5) (- 10 6))`用Scheme表表示，Scheme表基于Scheme序对表示，因此我们要实现Scheme序对，然后是表。  
只要我们实现了Scheme序对和表，我们也就是完成了Scheme数据的表示和Scheme程序的表示。接下来我们考虑Scheme序对在JS中的实现和表示。  
你应该已经知道了序对是什么。序对是下面的抽象定义，用JS描述：
```js
var z = cons(x, y); // x和y组成的序对
car(z) === x        // true
cdr(z) === y        // true
setCar(z, a);
car(z) === a        // true
setCdr(z, b);
cdr(z) === b        // true
```
在具体层面上，一个包含2个元素的数组可以实现：
```js
function cons(x, y) { return [x, y]; }
function car(z) { return z[0]; }
function cdr(z) { return z[1]; }
function setCar(z, x) { z[0] = x; }
function setCdr(z, y) { z[1] = y; }
```
在Scheme中，我们是这样构造Scheme程序的数据表示的：
```scheme
; 构造Scheme数据的Scheme表达式
(cons '+
      (cons (cons '* (cons 3 (cons 5 '())))
            (cons (cons '- (cons 10 (cons 6 '())))
                  '())))
```
在JS中，我们以同样的原理构造Scheme程序的数据表示：
```js
; 构造Scheme数据的JS表达式
cons("+",
     cons(cons("*", cons(3, cons(5, []))),
          cons(cons("-", cons(10 ,cons(6, []))), [])));
```

&emsp;&emsp;要认识到我们看不了太远，但要尽可能的往前看，并在当前勇于做一些决定。抽象可以延迟某些表示的决定。一种数据抽象方法是基于选择器函数和构造函数：由于本质上是函数，我们保持函数在"使用"层面的行为上，它的"实现"是可替换和修改的。一次完成不了所有事，要做着反复重写重构程序的准备。

## 4.执行语义
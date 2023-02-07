# 48 小时使用 F#写一个 Scheme!

本教程是 F#版本的[48 小时写一个 Scheme](https://en.wikibooks.org/wiki/Write_Yourself_a_Scheme_in_48_Hours)(一个著名的 Haskell 教程)。

本教程使用实践的教学手段。你将从使用并解析命令行开始，然后逐渐完成一个完全工作的实现了 R5RS 部分优秀特性的 Scheme 解释器。 在这个过程中，你会学到 F#的 I/O、 可修改状态、动态类型、错误处理以及解析器等功能。当你完成本教程时，你应当对于 F#和 Scheme 都比较熟悉了。

本教程面向下列人群

- 知道 Lisp 或 Scheme, 想学习 F#的人
- 不懂编程，但有相当强的相关背景，且对计算机比较熟悉

The second group will likely find this challenging, as this tutorial glosses over several concepts in Scheme and general programming in order to stay focused on F#. A good textbook such as [Structure and Interpretation of Computer Programs](https://mitpress.mit.edu/sites/default/files/sicp/full-text/book/book.html) or [The Little Schemer](https://mitpress.mit.edu/sites/default/files/sicp/full-text/book/book.html) should be very helpful.

Users of a procedural or object-oriented languages like [C](<https://en.wikipedia.org/wiki/C_(programming_language)>), [Java](<https://en.wikipedia.org/wiki/Java_(programming_language)>), or [Python](<https://en.wikipedia.org/wiki/Java_(programming_language)>) should beware: You'll have to forget most of what you already know about programming. F# is quite different from those languages, and requires a different way of thinking about programming. It's best to go into this tutorial with a blank slate; try not to compare F# to imperative languages, because many concepts (including classes, functions, and return) have significantly different meanings in F#. Although you can use imperative programming style in F#, we mostly focus on functional programming in F# in this tutorial.

Since each lesson builds upon code written in previous ones, it's generally best to go through them in order.

This tutorial assumes that you have the latest DotNet Core runtime installed. The command "dotnet" should be available and in your path.

The source code can found at the [Write Your a Scheme in 48 Hours using F# repo](https://github.com/pangwa/write-yourself-a-scheme). The main branch of this repo is a empty project to start with. The source code of each chapter can be found in the coresponding branch.

# Write Yourself a Scheme in 48 hours using F#!
This book is the F# fork of [Write Yourself a Scheme in 48 Hours](https://en.wikibooks.org/wiki/Write_Yourself_a_Scheme_in_48_Hours) which is a great and practical Haskell tutorial. 

This tutorial takes a practical approach . You'll start off using and parsing the command-line, then progress to writing a fully-functional Scheme interpreter that implements a decent subset of R5RS Scheme. Along the way, you'll learn F#'s I/O, mutable state, dynamic typing, error handling, and parsing features. By the time you finish, you should become fairly fluent in F# and Scheme.

This tutorial targets two main audiences:

* People who already know Lisp or Scheme, and want to learn F#
* People who don't know any programming language, but have a strong quantitative background, and are familiar with computers

The second group will likely find this challenging, as this tutorial glosses over several concepts in Scheme and general programming in order to stay focused on F#. A good textbook such as [Structure and Interpretation of Computer Programs](https://mitpress.mit.edu/sites/default/files/sicp/full-text/book/book.html) or [The Little Schemer](https://mitpress.mit.edu/sites/default/files/sicp/full-text/book/book.html) should be very helpful.

Users of a procedural or object-oriented languages like [C](https://en.wikipedia.org/wiki/C_(programming_language)), [Java](https://en.wikipedia.org/wiki/Java_(programming_language)), or [Python](https://en.wikipedia.org/wiki/Java_(programming_language)) should beware: You'll have to forget most of what you already know about programming. F# is quite different from those languages, and requires a different way of thinking about programming. It's best to go into this tutorial with a blank slate; try not to compare F# to imperative languages, because many concepts (including classes, functions, and return) have significantly different meanings in F#. Although you can use imperative programming style in F#, we mostly focus on functional programming in F# in this tutorial.

Since each lesson builds upon code written in previous ones, it's generally best to go through them in order.

This tutorial assumes that you have the latest DotNet Core runtime installed. The command "dotnet" should be available and in your path.

The source code can found at the [Write Your a Scheme in 48 Hours using F# repo](https://github.com/pangwa/write-yourself-a-scheme). The main branch of this repo is a empty project to start with. The source code of each chapter can be found in the coresponding branch.

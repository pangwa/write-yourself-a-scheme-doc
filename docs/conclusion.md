# Conclusion & Further Resources
You now have a working Scheme interpreter that implements a large chunk of the standard, including functions, lambdas, lexical scoping, symbols, strings, integers, list manipulation, and assignment. You can use it interactively, with a REPL, or in batch mode, running script files. You can write libraries of Scheme functions and either include them in programs or load them into the interactive interpreter. With a little text processing via awk or sed, you can format the output of UNIX commands as parenthesized Lisp lists, read them into a Scheme program, and use this interpreter for shell scripting.

There are still a number of features you could add to this interpreter. Hygienic macros let you perform transformations on the source code before it's executed. They're a very convenient feature for adding new language features, and several standard parts of Scheme (such as let-bindings and additional control flow features) are defined in terms of them. [Section 4.3](https://schemers.org/Documents/Standards/R5RS/HTML/r5rs-Z-H-7.html#%_sec_4.3) of R5RS defines the macro system's syntax and semantics, and there is a [whole collection](https://web.archive.org/web/20180810194414/http://library.readscheme.org/page3.html) of papers on implementation. Basically, you'd want to intersperse a function between readExpr and eval that takes a form and a macro environment, looks for transformer keywords, and then transforms them according to the rules of the pattern language, rewriting variables as necessarily.

Continuations are a way of capturing "the rest of the computation", saving it, and perhaps executing it more than once. Using them, you can implement just about every control flow feature in every major programming language. The easiest way to implement continuations is to transform the program into [continuation-passing style](https://web.archive.org/web/20180720154748/http://library.readscheme.org/page6.html), so that eval takes an additional continuation argument and calls it, instead of returning a result. This parameter gets threaded through all recursive calls to eval, but is only manipulated when evaluating a call to call-with-current-continuation.

Dynamic-wind could be implemented by keeping a stack of functions to execute when leaving the current continuation and storing (inside the continuation data type) a stack of functions to execute when resuming the continuation.

If you're just interested in learning more F#, there are a large number of links that may help:
- [FSharp Official Site](https://fsharp.org/).
- [Official F# Documents](https://docs.microsoft.com/en-us/dotnet/fsharp/).
- [Awesome Fsharp](https://github.com/fsprojects/awesome-fsharp) github repo.


This should give you a starting point for further investigations into the language. Happy hacking!


## Acknowledgements
This tutorial is mainly translated from the [original version](https://en.wikibooks.org/wiki/Write_Yourself_a_Scheme_in_48_Hours) written using Haskell.

Thanks to Ketil Malde, Pete Kazmier, and Brock for sending in questions and suggestions about the original tutorial.


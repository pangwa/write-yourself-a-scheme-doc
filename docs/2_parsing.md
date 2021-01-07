# Parsing

## Writing a Simple Parser
Now, let's try writing a very simple parser. We'll be using the  [fparsec](https://www.quanttec.com/fparsec/) to write a scheme parser in this chapter. You need to install `fparsec` using below command:

```bash
dotnet add package fparsec
dotnet add package FSharpPlus
```

Start by adding these lines to the import section:
```fsharp
open FParsec
```

This makes the `FParsec` library functions available to us.

Now, we'll define a parser that recognizes one of the symbols allowed in Scheme identifiers:

```fsharp
type LispState = unit // doesn't have to be unit, of course
type Parser<'t> = Parser<'t, LispState>

let pSymbol: Parser<_> = anyOf "!#$%&|*+-/:<=>?@^_~"
```

This is an example of a monad: in this case, the "extra information" that is being hidden is all the info about position in the input stream, backtracking record, first and follow sets, etc. `FParsec` takes care of all of that for us. We need only use the `FParsec` library function oneOf, and it'll recognize a single one of any of the characters in the string passed to it. `FParsec` provides a number of pre-built parsers: for example, letter and digit are library functions. And as you're about to see, you can compose primitive parsers into more sophisticated productions.

You may also noted that we add the explict type information for pSymbol, because we'll get a compile error without it. The error mentions F#'s "value restriction" which was explained in the  [documentation](https://www.quanttec.com/fparsec/tutorial.html#fs-value-restriction).

Let's define a function to call our parser and handle any possible errors:
```fsharp
let readExpr input = 
    match run pSymbol input with
    | Failure (_, err, _) -> sprintf "No match: %s"  (err.ToString())
    | Success _ -> "Found value"
```

As you can see from the type signature, readExpr is a function (->) from a string to a string. We name the parameter `input`, and pass it, along with the symbol parser we defined above to the `FParsec` function `run`. 

`run` can return either the parsed value or an error, so we need to handle the error case. Following the `FParsec` convention, `FParsec`` returns an `ParseResult` data type, using the Failure constructor to indicate an error and the Success one for a normal value.

We use a `match ... with ` construction to match the result of parse against these alternatives. If we get a `Failure` value (error), then we bind the error itself to err and return "No match" with the string representation of the error. If we get a `Success` value, weignore it, and return the string "Found value".

The `match ... with ` construction is an example of pattern matching, which we will see in much greater detail later on.

Finally, we need to change our `main` function to call `readExpr` and print out the result:
```fsharp
let main argv =
    let input = if argv.Length = 0 then "" else argv.[0]
    let result = readExpr input
    printfn "%s\n" result
    0 // return an integer exit code
```

Use `dotnet run -- ` and pass the string to the parser.
```bash
$ dotnet run -- '|'
Found value

$ dotnet run -- 1
No match: Error in Ln: 1 Col: 1
Expecting: any char in ‘!#$%&|*+-/:<=>?@^_~’
```


## Whitespace
Next, we'll add a series of improvements to our parser that'll let it recognize progressively more complicated expressions. The current parser chokes if there's whitespace preceding our symbol:
```bash
$ dotnet run -- "   %"
No match: Error in Ln: 1 Col: 1
Expecting: any char in ‘!#$%&|*+-/:<=>?@^_~’
```
Let's fix that, so that we ignore whitespace.

`FParsec` provides the `spaces` parser to match any number of space characters.

Now, let's edit our parse function so that it uses this new parser:
```fsharp
let readExpr input = 
    match run (spaces >>. pSymbol) input with
    | Failure (_, err, _) -> sprintf "No match: %s"  (err.ToString())
    | Success _ -> "Found value"
```
The `>>.` is a combinator that `FParsc` provides. The parser ``p1 >>. p2`` parses p1 and p2 in sequence and returns the result of p2.
- There is another companation operator `.>>` which also parses p1 and p2 in sequence but returns the result of p1 instead of p2. In each case the point points to the side of the parser whose result is returned. By combining both operators in p1 >>. p2 .>> p3 we obtain a parser that parses p1, p2 and p3 in sequence and returns the result from p2. As a mnemonic the dot at the composer helpers `>>.` and `.>>` mean which of the parsers are meant to be kept.

Compile and run this code. 
```bash
$ dotnet run -- '|'
Found value

$ dotnet run -- '    %'
Found value

$ dotnet run -- '    abc'
No match: Error in Ln: 1 Col: 4
Expecting: any char in ‘!#$%&|*+-/:<=>?@^_~’
```

## Return Values
Right now, the parser doesn't do much of anything—it just tells us whether a given string can be recognized or not. Generally, we want something more out of our parsers: we want them to convert the input into a data structure that we can traverse easily. In this section, we learn how to define a data type, and how to modify our parser so that it returns this data type.

First, we need to define a data type that can hold any Lisp value:

```fsharp
type LispVal = 
    | LispAtom of string
    | LispList of List<LispVal>
    | LispDottedList of List<LispVal> * LispVal
    | LispNumber of int64
    | LispString of string
    | LispBool of bool
```

This is an example of an `algebraic data type` (which called a [discriminated union](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/discriminated-unions) in F#): it defines a set of possible values that a variable of type LispVal can hold. Each alternative (called a constructor and separated by |) contains a name for the constructor along with the type of data that the constructor can hold. In this example, a LispVal can be:

* An Atom, which stores a String naming the atom
* A List, which stores a list of other LispVals (F# lists are denoted by the List<'T> generic type); also called a proper list
> We prefer List over the more generic Seq type here, because List can be patten matched which will become useful later on.
* A DottedList, representing the Scheme form (a b . c); also called an improper list. This stores a list of all elements but the last, and then stores the last element as another field
* A Number, containing an F# Integer
* A String, containing an F# String
* A Bool, containing an F# boolean value

The types are prefixed with the `Lisp` prefix to avoid conflicting with the existing types or keywords in F#.

Next, let's add a few more parsing functions to create values of these types. A string is a double quote mark, followed by any number of non-quote characters, followed by a closing quote mark:

```fsharp
let parseString: Parser<LispVal> = 
      between (pstring "\"") (pstring "\"") (manyChars (noneOf (Seq.toList "\""))) 
            |>> LispString
```

We're using the `between` combinator, the parser `between pOpen pClose p` applies `pOpen`, `p` and `pClose` in sequence. It returns the result of `p`.

Once we've finished the parse and have the F# string returned from many. We use the operator `|>>`, it pipes the parser result to the `LispString` function. The `LispString` is the constructor (from our `LispVal` data type) to turn it into a `LispVal`. Every constructor in an Record type also acts like a function that turns its arguments into a value of its type. It also serves as a pattern that can be used in the left-hand side of a pattern-matching expression; we saw an example of this in Lesson 3.1 when we matched our parser result against the two constructors in the `Result` data type.

Now let's move on to Scheme variables. An [atom](http://www.schemers.org/Documents/Standards/R5RS/HTML/r5rs-Z-H-5.html#%_sec_2.1) is a letter or symbol, followed by any number of letters, digits, or symbols:

```fsharp
let parseAtom = 
    pipe2 (letter <|> pSymbol) 
          (manyChars (letter <|> digit <|> pSymbol)) 
          (fun s rest -> 
                let atom = sprintf "%c%s" s rest
                match atom with
                | "#t" -> LispBool true
                | "#f" -> LispBool false
                | _ -> LispAtom atom) 
```

Here, we introduce another `FParsec` combinator, the choice operator `<|>`. This tries the first parser, then if it fails, tries the second. If either succeeds, then it returns the value returned by that parser. The first parser must fail before it consumes any input: we'll see later how to implement backtracking.

Once we've read the first character and the rest of the atom, we need to put them together. The "`let`" statement defines a new variable `atom`. We use the `sprintf` for this. Instead of `sprintf`, we could have used the concatenation operator `+` like this `s1 + s2`, but we need to convert s from `char` to `string` using `string s`, it would writes like: `let atom = (string s) + rest`.

Then we use a [match expression](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/match-expressions) to determine which `LispVal` to create and return, matching against the literal strings for `true` and `false`. The underscore `_` alternative is a readability trick: match blocks continue until a `_` case (or fail any case which also causes the failure of the whole case expression), think of `_` as a wildcard. So if the code falls through to the `_` case, it always matches, and returns the value of `atom`.

Finally, we create one more parser, for numbers. 

```fsharp
let parseNumber: Parser<_> = pint64 |>> LispNumber
```
The `FParsec` parser `pint64` parses an  `int64` value. We'd like to construct a number `LispVal` from the resulting number using `LispNumber` constructor.

Let's create a parser that accepts either a string, a number, or an atom:

```fsharp
let parseExpr = parseAtom <|> 
                parseString <|> 
                parseNumber
```

Now update the `readExpr` function to use `parseExpr`
```fsharp
let readExpr input =
    match run (spaces >>. parseExpr) input with
        | Failure (_, err, _) -> sprintf "No match: %s" (err.ToString())
        | Success _ -> "Found Value"
```


Compile and run this code, and you'll notice that it accepts any number, string, or symbol, but not other strings:
```bash
$ dotnet run -- '"this is a string"'
Found value
$ dotnet run -- '25'
Found value
$ dotnet run -- symbol
Found value
$ dotnet run -- '(symbol)'
No match: Error in Ln: 1 Col: 1
Expecting: any char in ‘!#$%&|*+-/:<=>?@^_~’, integer number (64-bit, signed),
letter or '"'
```

## Refactor the code
Now we put all the source code in the `Program.fs` which is not a good programming practice. Let's refactor the code and break the type definitions into a separated `LispTypes.fs` and the parsing stuff into `Parser.fs`.

`LispTypes.fs`
```fsharp
module LispTypes

type LispVal = 
    | LispAtom of string
    | ListAtom of List<LispVal>
    | LispDottedList of List<LispVal> * LispVal
    | LispNumber of int64
    | LispString of string
    | LispBool of bool

```
`Parser.fs`
```fsharp
module Parser
open NUnit.Framework
open FsUnit
open FParsec
open LispTypes

type LispState = unit // doesn't have to be unit, of course
type Parser<'t> = Parser<'t, LispState>

let pSymbol: Parser<_> = anyOf "!#$%&|*+-/:<=>?@^_~"
let parseString: Parser<LispVal> = 
      between (pstring "\"") (pstring "\"") (manyChars (noneOf (Seq.toList "\""))) 
            |>> LispString
let parseAtom = 
    pipe2 (letter <|> pSymbol) 
          (manyChars (letter <|> digit <|> pSymbol)) 
          (fun s rest -> 
                let atom = sprintf "%c%s" s rest
                match atom with
                | "#t" -> LispBool true
                | "#f" -> LispBool false
                | _ -> LispAtom atom) 

let parseNumber: Parser<_> = pint64 |>> LispNumber

let parseExpr = parseAtom <|> 
                parseString <|> 
                parseNumber

let readExpr input = 
    match run parseExpr input with
    | Failure (_, err, _) -> sprintf "No match: %s"  (err.ToString())
    | Success _ -> "Found value"

```

Now go back to the main file `Program.fs` Remove all LispVal and parsing code and add one line:
```fsharp
open Parser
```
Which imports the Parser module.

You must add `LispTypes.fs` and `Parser.fs` to the F# project file, just open the .fsproj file and add two lines in the `ItemGroup` section:
```xml
    <Compile Include="LispTypes.fs" />
    <Compile Include="Parser.fs" /
```
The two line should be placed before `Program.fs` and their order should be the same as the compile order. In F# the file ordering is important, a file can only import the files which compiles before it.

We added `module` definitions in `LispTypes.fs` and `Parser.fs` which can be imported in other files using the `open` statement.

## Testing
We can add some tests to the `Parser.fs`, for example we could add some tests to the `atom` parser.

Add below test cases at the end of `Parser.fs`.
```fsharp

let checkResult v r = match r with
                      | ParserResult.Success(e, _, _) -> e |> should equal v
                      | _ -> Assert.Fail "parse failed"
  
let checkParseFailed r = match r with
                         | ParserResult.Success(_, _, _) -> Assert.Fail("Expect parse fail")
                         | _ -> ()

[<Test>]
let ``parse atom test`` () =
  run parseAtom "#t" |> checkResult (LispBool true)
  run parseAtom "#f" |> checkResult (LispBool false)
  run parseAtom "#test" |> checkResult (LispAtom "#test")
  run parseAtom "test" |> checkResult (LispAtom "test")
  run parseAtom "+" |> checkResult (LispAtom "+")
  run parseAtom "1" |> checkParseFailed
```

Run the test and check the result:
```bash
$ dotnet test
...
Test Pased - Failed:     0, Passed:     2, Skipped:     0, Total:     2, Duration: 72 ms- /
```

## Exercises
1. Our strings aren't quite [R5RS compliant](http://www.schemers.org/Documents/Standards/R5RS/HTML/r5rs-Z-H-9.html#%_sec_6.3.5), because they don't support escaping of internal quotes within the string. Change parseString so that \" gives a literal quote character instead of terminating the string. You may want to replace noneOf "\"" with a new parser action that accepts either a non-quote character or a backslash followed by a quote mark.
2. Add a Character constructor to LispVal, and create a parser for [character literals](http://www.schemers.org/Documents/Standards/R5RS/HTML/r5rs-Z-H-9.html#%_sec_6.3.4) as described in R5RS.
3. Add a `Float` constructor to LispVal, and support R5RS syntax for [decimals](https://schemers.org/Documents/Standards/R5RS/HTML/r5rs-Z-H-9.html#%_sec_6.2.4).
4. Add data types and parsers to support the [full numeric tower](http://www.schemers.org/Documents/Standards/R5RS/HTML/r5rs-Z-H-9.html#%_sec_6.2.1) of Scheme numeric types. F# has built-in types to represent many of these; check the documents. For the others, you can define compound types (e.g. Records) that represent e.g. a `Rational` as a numberator and denominator, or a `Complex` as a real and imaginary part (each itself a `Real`).
5. Add more tests to `Parser.fs`, so we can be more confident with our parser functionality.


## Recursive Parsers: Adding lists, dotted lists, and quoted datums
Next, we add a few more parser actions to our interpreter. Start with the parenthesized lists that make Lisp famous:

```fsharp
let parseList = sepBy parseExpr spaces1 |>> LispList
```

This works analogously to `parseNumber`, first parsing a series of expressions separated by whitespace (`sepBy parseExpr spaces1`) and then apply the List constructor to it within the Parser result using the `|>>` combinator. Note too that we can pass `parseExpr` to `sepBy`, even though it's an action we wrote ourselves.

The dotted-list parser is somewhat more complex, but still uses only concepts that we're already familiar with:
```fsharp
let parseDottedList =
    pipe2 (sepEndBy parseExpr spaces1) (pchar '.' >>. spaces >>. parseExpr) (fun head tail -> LispDottedList(head, tail))
```
Here we use the parser `pipe2 p1 p2 f` applies the parsers `p1` and `p2` in sequence. It returns the result of the function application `f a b`, where `a` and `b` are the results returned by `p1` and `p2`.

We also use `sepEndby` and the `>>.` combinators to build parsers.

Next, let's add support for the single-quote syntactic sugar of Scheme:

```fsharp
let parseQuoted =
    pchar '\'' >>. parseExpr
    |>> (fun expr -> [ LispAtom "quote"; expr ] |> LispList)
```

Most of this is fairly familiar stuff: it reads a single quote character, reads an expression and binds it to expr using the `|>>` combinator, and then returns `(quote x)`, to use Scheme notation. The `LispAtom` constructor works like an ordinary function: you pass it the String you're encapsulating, and it gives you back a `LispVal`. You can do anything with this `LispVal` that you normally could, like put it in a list.

Finally, edit our definition of parseExpr to include our new parsers:

```fsharp
let parseExpr =
    choice [ parseAtom
             parseString
             parseNumber
             parseQuoted
             (between (pchar '(') (pchar ')') (attempt parseList <|> parseDottedList)) ]
```
Here we used the `choice` combinator instead of `<|>`, `choice` is a optimized implementation of the `<|>` combinator.

This illustrates one last feature of `Parsec`: backtracking. `parseList` and `parseDottedList` recognize identical strings up to the dot; this breaks the requirement that a choice alternative may not consume any input before failing. The `attempt` combinator attempts to run the specified parser, but if it fails, it backs up to the previous state. This lets you use it in a choice alternative without interfering with the other alternative.

Try to compile the project and bong! You would get a bounch of compile errors like
```
Error FS0039: undefined value or constructure "parseExpr".
```

This error happens because we defined the `parseExpr` as a recursive value, referencing in `parseList` and `parseQuoted` which also need to reference `parseExpr`. Fortunately `FParsec` provides a workaround for such case the `createParserForwardedToRef`, 
Add codes to the `Parser.fs`
```fsharp
let parseExpr, parseExprRef = createParserForwardedToRef () // add before parseList and parseDottedList

// put after the parseQuoted function definition
parseExprRef
:= choice [ parseAtom
            parseString
            parseNumber
            parseQuoted
            (between (pchar '(') (pchar ')') (attempt parseList <|> parseDottedList)) ]
```

The `createParserForwardedToRef()` creates a parser which will actually use the parserRef (which is a [reference cell](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/reference-cells) in F#) later on. And we use the  `:=` syntax to rebind the `parseExpRef` value. 

Now compile and run the code:
```bash
$ dotnet run -- "(a test)"
Found value
$ dotnet run -- "(a (nested) test)"
$ dotnet run -- "(a (dotted . list) test)"
$ dotnet run -- "(a '(quoted (dotted . list)) test)"
Found value
$ dotnet run -- "(a '(imbalanced parens)"
No match: Error in Ln: 1 Col: 24
Expecting: whitespace or ')'
```

Note that by referring to `parseExpr` within our parsers, we can nest them arbitrarily deep. Thus, we get a full Lisp reader with only a few definitions. That's the power of recursion.


## Exercises
1. Add support for the [backquote](http://www.schemers.org/Documents/Standards/R5RS/HTML/r5rs-Z-H-7.html#%_sec_4.2.6) syntactic sugar: the Scheme standard details what it should expand into (quasiquote/unquote).
2. Add support for [vectors](https://schemers.org/Documents/Standards/R5RS/HTML/r5rs-Z-H-9.html#%_sec_6.3.6). The F# representation is up to you: List, Array, Seq. You may have a better idea how to do this after the section on set!, later in this tutorial.
3. Instead of using the `attempt` combinator, left-factor the grammar so that the common subsequence is its own parser. You should end up with a parser that matches a string of expressions, and one that matches either nothing or a dot and a single expression. Combining the return values of these into either a List or a DottedList is left as a (somewhat tricky) exercise for the reader: you may want to break it out into another helper function.
4. Add more tests to check the result of the `parseList` and `parseDottedList` functions

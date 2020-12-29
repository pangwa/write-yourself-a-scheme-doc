# Evaluation, Part 1
## Beginning the Evaluator
Currently, we've just been printing out whether or not we recognize the given program fragment. We're about to take the first steps towards a working Scheme interpreter: assigning values to program fragments. We'll be starting with baby steps, but fairly soon you'll be progressing to working with computations.

Let us start by telling F# how to print out a string representation of the various possible `LispVals`, add below method to `LispVal`, it should be indented as the constructors. modify Lisp declaration as below:

```fsharp
type LispVal =
    | LispAtom of string
    | LispList of List<LispVal>
    | LispDottedList of List<LispVal> * LispVal
    | LispNumber of int64
    | LispString of string
    | LispBool of bool

    override this.ToString() =
        match this with
        | LispAtom s -> s
        | LispString s -> sprintf "\"%s\"" s
        | LispNumber v -> sprintf "%d" v
        | LispBool true -> "#t"
        | LispBool false -> "#f"
        | LispList v -> unwordsList v |> sprintf "(%s)"
        | LispDottedList (head, tail) ->
            head
            |> unwordsList
            |> (sprintf "(%s . %s)")
            <| (tail.ToString())
```
Here we add a member function to the `LispVal` record type which overrides the default `ToString` function.  Also, in the example above, you see the word `this` in front of the `ToString` method name. `this` is a `self-identifier` that can be used to refer to the current instance of the class. Every non-static member must have a self-identifier, even it is not used (as in the properties above). There is no requirement to use a particular word, just as long as it is consistent. You could use `this` or `self` or `me` or any other word that commonly indicates a self reference.

And This is our first real introduction to pattern matching. Pattern matching is a way of destructuring an algebraic data type, selecting a code clause based on its constructor and then binding the components to variables. Any constructor can appear in a pattern; that pattern matches a value if the tag is the same as the valueus tag and all subpatterns match their corresponding components. Patterns can be nested arbitrarily deep, with matching proceeding in an inside → outside, left → right order. The clauses of a function definition are tried in textual order, until one of the patterns matches. If this is confusing, you'll see some examples of deeply-nested patterns when we get further into the evaluator.

For now, you only need to know that each clause of the above definition matches one of the constructors of `LispVal`, and the right-hand side tells what to do for a value of that constructor.

The `LispList` and `LispDottedList` clauses work similarly, but we need to define a helper function unwordsList to convert the contained list into a string:
```fsharp
        | LispList v -> unwordsList v |> sprintf "(%s)"
        | LispDottedList (head, tail) ->
            head
            |> unwordsList
            |> (sprintf "(%s . %s)")
            <| (tail.ToString())
```

The `unwordsList` function works like the F#'s `String.concat` function which glues together a list of words with a delimeter. Since we're dealing with a list of `LispVals` instead of words, we define a function that first converts the `LispVals` into their string representations and then applies `String.concat` to it:
```fsharp
let unwordsList list = list
                       |> List.map (fun p -> p.ToString())
                       |> String.concat " "
```
> The `|>` operator is used everywhere in F#, this works like the `|` operator in bash which feeds the output of the left call to the right side.


Let's try things out by changing our `readExpr` function so it returns the string representation of the value actually parsed, instead of just "Found value":

```fsharp
let readExpr input =
    match run parseExpr input with
    | Failure (_, err, _) -> sprintf "No match: %s" (err.ToString())
    | Success (v, _, _) -> sprintf "Found value: %s" (v.ToString())
```
And compile and run…

```bash
$ dotnet run -- "(1 2 2)"
Found (1 2 2)
$ dotnet run -- "'(1 3 (\"this\" \"one\"))'"
Found value: (quote (1 3 ("this" "one")))
```
## Beginnings of an evaluator: Primitives
Now, we start with the beginnings of an evaluator. The purpose of an evaluator is to map some "code" data type into some "data" data type, the result of the evaluation. In Lisp, the data types for both code and data are the same, so our evaluator will return a `LispVal. Other languages often have more complicated code structures, with a variety of syntactic forms.

Let's start with adding a new file `Eval.fs` to the F# project, as we mentioned before, F# files orders matters, so we need to put `Eval.fs` after `Parser.fs` in the `.fsproj` file.

Evaluating numbers, strings, booleans, and quoted lists is fairly simple: return the datum itself.

```fsharp
module Eval

open LispTypes

let eval = 
  function
  | LispString _ as v -> v
  | LispNumber _ as v -> v
  | LispBool _ as v -> 
  | LispList [ LispAtom "quote"; v ] -> v
```

Here we introuced the syntax function pattern matching syntax `let eval = function ... |` which is a shortcut for 
```fsharp
let eval args = match args with
                | ...
```
This introduces a new type of pattern. The notation `LispString _ as v` matches against any `LispVal` that's a string and then binds v to the whole `LispVal`, and not just the contents of the `LispString` constructor. The result has type `LispVal` instead of type String. The underbar is the "don't care" variable, matching any value yet not binding it to a variable. It can be used in any pattern, but is especially useful with as (where you bind the variable to the whole pattern) and with simple constructor-tests where you're just interested in the tag of the constructor.

The last clause is our first introduction to nested patterns. The type of data contained by `LispList` is `List<LispVal>`, a list of `LispVals`. We match that against the specific two-element list `[LispAtom "quote", val]`, a list where the first element is the symbol "quote" and the second element can be anything. Then we return that second element.

Let's integrate `eval` into our existing code. Start by changing `readExpr` back so it returns the expression instead of a string representation of the expression:

```fsharp
let readExpr input =
    match run parseExpr input with
    | Failure (_, err, _) -> sprintf "No match: %s" (err.ToString()) |> LispString
    | Success (v, _, _) -> v
```

And then change our `main` function to read an expression, evaluate it, convert it to a string, and print it out. As you already know the `|>` pipe operator, we can make it a little bit concise:
```fsharp
// Learn more about F# at http://fsharp.org
open System
open LispTypes
open Parser
open Eval

[<EntryPoint>]
let main argv =
    argv |> Array.tryHead |> Option.defaultValue "" |>
    readExpr |> eval |> (fun v -> v.ToString()) |> printfn "%s\n"
    0 // return an integer exit code
```

Here, we take the input from the argv array and pass it into the composition of:
1. take the first value (`Array.tryHead` and `Option.defaultValue`), use "" if the array is empty;
2. parse it (`readExpr`);
3. evaluate it (`eval`);
4. convert it to string;
5. print it (`printfn`).

Compile and run the code the normal way:

```bash
$ dotnet run -- "'atom" 
atom
$ dotnet run -- 2
2
$ dotnet run -- "\"a string\""
"a string"
$ dotnet run --  "(+ 2 2)"
Unhandled exception. Microsoft.FSharp.Core.MatchFailureException: The match cases were incomplete
   at Eval.eval(LispVal _arg1) in write-yourself-a-scheme/Eval.fs:line 10
   at Program.main(String[] argv) in write-yourself-a-scheme/Program.fs:line 12
```
We still can't do all that much useful with the program (witness the failed (+ 2 2) call), but the basic skeleton is in place. Soon, we'll be extending it with some functions to make it useful.

## Adding basic primitives
Next, we'll improve our Scheme so we can use it as a simple calculator. It's still not yet a "programming language", but it's getting close.

Begin by adding a clause to eval to handle function application. Remember that all clauses of a function definition must be placed together and are evaluated in textual order, so this should go after the other eval clauses:

```fsharp
  | LispList (LispAtom func:: args) -> args |> List.map eval |> apply func
```

This is another nested pattern, but this time we match against the cons operator "`::`" instead of a literal list. Lists in F# are really syntactic sugar for a chain of cons applications and the empty list: [1, 2, 3, 4] = 1::(2::(3::(4::[]))). By pattern-matching against cons itself instead of a literal list, we're saying "give me the rest of the list" instead of "give me the second element of the list". For example, if we passed `(+ 2 2)` to eval, `func` would be bound to "`+`" and `args` would be bound to [LispNumber 2, LispNumber 2].

The rest of the clause consists of a couple of functions we've seen before and one we haven't defined yet. We have to _recursively_ evaluate each argument, so we map `eval` over the args. This is what lets us write compound expressions like `(+ 2 (- 3 1) (* 5 4))`. Then we take the resulting list of evaluated arguments, and pass it and the original function to `apply`, `apply` should be defined right after `eval`:
```fsharp
and apply func args =
    Map.tryFind func primitives|> Option.map (fun f -> f args) |> Option.defaultWith (fun () -> LispBool false) 
```
As the `eval` and the `apply` function are recusive with each other, we can't define them separately, we use the syntax `let .... and ...` to define them at the same time.

The static function `Map.tryFind` looks up a key (its first argument) in a Map. However, `tryFind` will fail if no key in the map contains the matching key. To express this, it returns an instance of the built-in type `Option`. We use the function `Option.defaultWith` to specify what to do in case of either success or failure. If the function isn't found, we return a `LispBool false` value, equivalent to `#f` (we'll add more robust error-checking later). If it is found, we apply it to the arguments using (`Option.map`).

Next, we define the list of primitives that we support:

```fsharp
open FSharpPlus // for the divRem function
open FSharpPlus.Data

let primitives: Map<string, List<LispVal> -> LispVal> =
    Map.empty.
        Add("+", numbericBinOp (+)).
        Add("-", numbericBinOp (-)).
        Add("*", numbericBinOp (*)).
        Add("/", numbericBinOp (/)).
        Add("mod", numbericBinOp (%)).
        Add("quotient", numbericBinOp (/)).
        Add("remainder", numbericBinOp (fun a b -> (divRem a b) |> snd))
```

Look at the type of `primitives`. It is a map with the string for the key and the values of the pairs are functions from `Lisp<LispVal>` to `LispVal`. In F#, you can easily store functions in other data structures, though the functions must all have the same type.

Also, the functions that we store are themselves the result of a function, `numericBinop`, which we haven't defined yet. This takes a primitive F# function (often an operator section) and wraps it with code to unpack an argument list, apply the function to it, and wrap the result up in our `LispNumber` constructor.

```fsharp
let rec unpackNum =
    function
    | LispNumber v -> v
    | LispString s -> match run pint64 s with
                        | Success (v, _, _) -> v
                        | Failure (err, _, _) -> 0L // parse failed, use 0
    | LispList [n] -> unpackNum n
    | _ -> 0L

let numbericBinOp op args = args |> List.map unpackNum |> List.reduce op |> LispNumber

```
As with R5RS Scheme, we don't limit ourselves to only two arguments. Our numeric operations can work on a list of any length, so `(+ 2 3 4)` = `2 + 3 + 4`, and `(- 15 5 3 2)` = `15 - 5 - 3 - 2`. We use the built-in function `Lisp.reduce` to do this. It essentially changes every cons operator in the list to the binary function we supply, `op`.

Unlike R5RS Scheme, we're implementing a form of `weak typing`. That means that if a value can be interpreted as a number (like the string "2"), we'll use it as one, even if it's tagged as a string. We do this by adding a couple extra clauses to `unpackNum`. If we're unpacking a string, attempt to parse it with FParsc's built-in pint64 function, which returns the number if it success.

For lists, we pattern-match against the one-element list and try to unpack that. Anything else falls through to the next case.

If we can't parse the number, for any reason, we'll return 0 for now. We'll fix this shortly so that it signals an error.

The last thing to notice is the `rec` keyword before in `let rec unpackNumber = `, this is because F# doesn't support [recusive functions](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/functions/recursive-functions-the-rec-keyword) definiton by default, you must add the `rec` annonation to the function signature if it's recusive.  This also applies to the `eval` function as it calles the `apply` function in it which calls itself.

Compile and run this the normal way. Note how we get nested expressions "for free" because we call eval on each of the arguments of a function:

```bash
$ dotnet run -- "(+ 2 2)"
4
$ dotnet run -- "(+ 2 (-4 1))"
2
$ dotnet run -- "(+ 2 (- 4 1))"
5
$ dotnet run -- "(- (+ 4 6 3) 3 5 2)"
3
```

## Exercises
1. Add primitives to perform the various [type-testing](https://schemers.org/Documents/Standards/R5RS/HTML/r5rs-Z-H-9.html#%_sec_6.3) functions of R5RS: `symbol?`, `string?`, `number?`, etc.
2. Change `unpackNum` so that it always returns 0 if the value is not a number, even if it's a string or list that could be parsed as a number.
3. Add the [symbol-handling functions](https://schemers.org/Documents/Standards/R5RS/HTML/r5rs-Z-H-9.html#%_sec_6.3.3) from R5RS. A symbol is what we've been calling an `LispAtom` in our data constructors.
4. Add more unit tests to our `Eval.fs` to test our new eval code.

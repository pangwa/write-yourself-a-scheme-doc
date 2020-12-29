# Evaluation, Part 2
## Additional Primitives: Partial Application
Now that we can deal with type errors, bad arguments, and so on, we'll flesh out our primitive list so that it does something more than calculate. We'll add boolean operators, conditionals, and some basic string operations.

Start by adding the following into the list of primitives:

```fsharp
        Add("=", numBoolBinop (=)).
        Add("<", numBoolBinop (<)).
        Add(">", numBoolBinop (>)).
        Add("/=", numBoolBinop (<>)).
        Add(">=", numBoolBinop (>=)).
        Add("<=", numBoolBinop (<=)).
        Add("&&", boolBoolBinop (&&)).
        Add("||", boolBoolBinop (||)).
        Add("string=?", strBoolBinop (=)).
        Add("string<?", strBoolBinop (<)).
        Add("string>?", strBoolBinop (>)).
        Add("string<=?", strBoolBinop (<=)).
        Add("string>=?", strBoolBinop (>=))
```
These depend on helper functions that we haven't written yet: `numBoolBinop`, `boolBoolBinop` and `strBoolBinop`. Instead of taking a variable number of arguments and returning an integer, these both take exactly two arguments and return a boolean. They differ from each other only in the type of argument they expect, so let's factor the duplication into a generic `boolBinop` function that's parametrized by the unpacker function it applies to its arguments:

```fsharp
let boolBinop unpacker op (args: List<LispVal>) =
    if args.Length <> 2 then
        NumArgs(2, args) |> throwError
    else
        monad {
            let! left = args.[0] |> unpacker
            let! right = args.[1] |> unpacker
            return op left right |> LispBool
        }
```
Because each argument may throw a type mismatch, we have to unpack them sequentially, in a `monad` block (for the `Result` monad). We then apply the operation to the two arguments and wrap the result in the `LispBool` constructor. 

Here used the `monad` construct which was defined in the `FSharpPlus` library. The code block can be written without the `monad` block, which can be replaced by two `Result.bind` function calls. It will a little bit more tedious here.

Also, take a look at the type signature. `boolBinop` takes two functions as its first two arguments: the first is used to unpack the arguments from `LispVals` to native F# types, and the second is the actual operation to perform. By parameterizing different parts of the behavior, you make the function more reusable.

Now we define three functions that specialize `boolBinop` with different unpackers:

```fsharp
let numBoolBinop = boolBinop unpackNum
let strBoolBinop = boolBinop unpackStr
let boolBoolBinop = boolBinop unpackBoo
```
We haven't told F# how to unpack strings from `LispVal`s yet. This works similarly to `unpackNum`, pattern matching against the value and either returning it or throwing an error. Again, if passed a primitive value that could be interpreted as a string (such as a number or boolean), it will silently convert it to the string representation.

```fsharp
let unpackStr =
    function
    | LispString s -> Result.Ok s
    | LispNumber v -> sprintf "%d" v |> Result.Ok
    | LispBool v -> v.ToString() |> Result.Ok
    | notString -> TypeMismatch("string", notString) |> throwError
```

And we use similar code to unpack booleans:
```fsharp
let unpackBool =
    function
    | LispBool v -> Result.Ok v
    | notBool -> TypeMismatch("boolean", notBool) |> throwError
```

Let's compile and test this to make sure it's working, before we proceed to the next feature:
```bash
$ dotnet run -- "(< 2 3)"
#t
$ dotnet run -- "(> 2 3)"
#f
$ dotnet run -- "(>= 3 3)"
#t
$ dotnet run -- "(string=? \"test\"  \"test\")"
#t
$ dotnet run -- "(string<? \"abc\" \"bba\")"
#t
```

## Conditionals: Pattern Matching 2
Now, we'll proceed to adding an if-clause to our evaluator. As with standard Scheme, our evaluator considers `#f` to be false and any other value to be true:

```fsharp
    | LispList [ LispAtom "if"; pred; conseq; alt ] ->
        eval pred
        |> Result.bind
            (fun v ->
                match v with
                | LispBool false -> eval alt
                | _ -> eval conseq)
```
As the function definitions are evaluated in order, be sure to place this one above `| LispList (LispAtom func :: args) -> args |> mapM eval |> Result.bind (apply func)` or it will throw a `Unrecognized primitive function args: "if"` error.

This is another example of nested pattern-matching. Here, we're looking for a 4-element list. The first element must be the atom `if`. The others can be any Scheme forms. We take the first element, evaluate, and if it's false, evaluate the alternative. Otherwise, we evaluate the consequent.

Compile and run this, and you'll be able to play around with conditionals:
```bash
$ dotnet run -- "(if (> 2 3) \"no\" \"yes\")"
evaluated: "yes"
$ dotnet run -- "(if (= 3 3) (+ 2 3 (- 5 1)) \"unequal\")"
evaluated: 9
```
## List Primitives: car, cdr, and cons
For good measure, let's also add in the basic list-handling primitives. Because we've chosen to represent our lists as F# algebraic data types instead of pairs, these are somewhat more complicated than their definitions in many Lisps. It's easiest to think of them in terms of their effect on printed S-expressions:
1. `(car '(a b c))` = a
1. `(car '(a))` = a
1. `(car '(a b . c))`= a
1. `(car 'a)` = error – not a list
1. `(car 'a 'b)` = error – `car` only takes one argument

We can translate these fairly straightforwardly into pattern matching clauses, recalling that (x :: xs) divides a list into the first element and the rest:

```fsharp
let car =
    function
    | [ LispList (x :: _) ] -> Result.Ok x
    | [ LispDottedList (x :: _, _) ] -> Result.Ok x
    | [ badArg ] -> TypeMismatch("pair", badArg) |> throwError
    | badArgList -> NumArgs(1, badArgList) |> throwError
```
Let's do the same with `cdr`:
1. `(cdr '(a b c))` = `(b c)`
1. `(cdr '(a b))` = `(b)`
1. `(cdr '(a))` = `NIL`
1. `(cdr '(a . b))` = `b`
1. `(cdr '(a b . c))` = `(b . c)`
1. `(cdr 'a)` = error – not a list
1. `(cdr 'a 'b)` = error – too many arguments

We can represent the first three cases with a single clause. Our parser represents `'()` as `List []`, and when you pattern-match `(x :: xs)` against `[x]`, xs is bound to `[]`. The other ones translate to separate clauses:

```fsharp
let cdr v =
    match v with
    | [ LispList (_ :: xs) ] -> Result.Ok(LispList xs)
    | [ LispDottedList ([ _ ], x) ] -> Result.Ok x
    | [ LispDottedList (_ :: xs, x) ] -> Result.Ok(LispDottedList(xs, x))
    | [ badArg ] -> TypeMismatch("pair", badArg) |> throwError
    | badArgList -> NumArgs(1, badArgList) |> throwError
```

`cons` is a little tricky, enough that we should go through each clause case-by-case. If you cons together anything with `Nil`, you end up with a one-item list, the `Nil` serving as a terminator:
```fsharp
let cons =
    function
    | [ x1; LispList [] ] -> Result.Ok(LispList [ x1 ])
```
If you `cons` together anything and a list, it's like tacking that anything onto the front of the list:
```fsharp
    | [ x; LispList xs ] -> x :: xs |> LispList |> Result.Ok
```
However, if the list is a `LispDottedList`, then it should stay a `LispDottedList`, taking into account the improper tail:
```fsharp
    | [ x; LispDottedList (xs, xlast) ] -> LispDottedList(x :: xs, xlast) |> Result.Ok
```
If you `cons` together two non-lists, or put a list in front, you get a `LispDottedList`. This is because such a `cons` cell isn't terminated by the normal `Nil` that most lists are.
```fsharp
    | [ x1; x2 ] -> LispDottedList([ x1 ], x2) |> Result.Ok
```
Finally, attempting to `cons` together more or less than two arguments is an error:

```fsharp
    | badArgList -> NumArgs(1, badArgList) |> throwError
```

Our last step is to implement `eqv?`. Scheme offers three levels of equivalence predicates: `eq?`, `eqv?`, and `equal?`. For our purposes, `eq?` and `eqv?` are basically the same: they recognize two items as the same if they print the same, and are fairly slow. So we can write one function for both of them and register it under `eq?` and `eqv?`.

Firstly we add the `Equals` override to `LispVal`, we need to add some attributes to the `LispVal`:
```fsharp
[<CustomEquality; NoComparison>]
type LispVal 
```
And implement the `Equals` method, it should be placed right after the `ToString()` method.
```fsharp
      override x.Equals(yObj) =
        let compareList (s1: List<LispVal>) (s2: List<LispVal>) =
           s1.Length = s2.Length
           && (List.zip s1 s2
               |> List.forall (fun (x, y) -> x = y))

        match yObj with
        | :? LispVal as y ->
            match (x, y) with
            | (LispAtom s1, LispAtom s2) -> s1 = s2
            | (LispString s1, LispString s2) -> s1 = s2
            | (LispNumber v1, LispNumber v2) -> v1 = v2
            | (LispBool b1, LispBool b2) -> b1 = b2
            | (LispList s1, LispList s2) -> compareList s1 s2
            | (LispDottedList (h1, t1), LispDottedList (h2, t2)) -> compareList h1 h2 && t1 = t2
            | _ -> false

```

The `Equals` method just as the scheme `eq?` function it checks the equality deeply.

Most of these clauses are self-explanatory, the exception being the one for two Lists. This, after checking to make sure the lists are the same length, `List.zip` zip the two lists of pairs, and then uses the function `List.forall` to return `false` if any pair is not equal in this list. `compareList` is an example of a local definition: it is defined using the let keyword inside the function body, just like a normal function, but is available only within that particular clause of `Equal`. 

Now the `eqv` function could be simply as:
```fsharp
let rec eqv =
    function
    | [ l; r ] -> l = r |> LispBool |> Result.Ok
    | badArgList -> NumArgs(2, badArgList) |> throwError
```


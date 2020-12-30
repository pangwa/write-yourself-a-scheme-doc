# Defining Scheme Functions: Closures and Environments
Now that we can define variables, we might as well extend it to functions. After this section, you'll be able to define your own functions within Scheme and use them from other functions. Our implementation is nearly finished.

Let's start by defining new `LispVal` constructors:

```fsharp
type LispVal =
    | LispAtom of string
    | LispList of List<LispVal>
    | LispDottedList of List<LispVal> * LispVal
    | LispNumber of int64
    | LispString of string
    | LispBool of bool
    | LispPrimitiveFunc of (List<LispVal> -> ThrowsError<LispVal>)
    | LispFunc of List<string> * Option<string> * List<LispVal> * Env
```
We've added a separate constructor for primitives, because we'd like to be able to store `+`, `eqv?`, etc. in variables and pass them to functions. The `PrimitiveFunc` constructor stores a function that takes a list of arguments to a `ThrowsError LispVal`, the same type that is stored in our primitive list.
We also want a constructor to store user-defined functions. We store four pieces of information:
1. the names of the parameters, as they're bound in the function body;
1. whether the function accepts a variable-length list of arguments, and if so, the variable name it's bound to;
1. the function body, as a list of expressions;
1. the environment that the function was created in.

You might notice we'll getting complile error after adding the two types constructures because we're defining `LispVal` and `LispError` recusively. We need to adjust the definition to be together:
```fsharp
[<CustomEquality; NoComparison>]
type LispVal =
    | LispAtom of string
    | LispList of List<LispVal>
    | LispDottedList of List<LispVal> * LispVal
    | LispNumber of int64
    | LispString of string
    | LispBool of bool
    | LispPrimitiveFunc of (List<LispVal> -> ThrowsError<LispVal>)
    | LispFunc of List<string> * Option<string> * List<LispVal> * Env

    override this.ToString() =
        match this with
        | LispAtom s -> s
        | LispString s -> sprintf "\"%s\"" s
        | LispNumber v -> sprintf "%d" v
        | LispBool true -> "#t"
        | LispBool false -> "#f"
        | LispList v -> unwordsList v |> sprintf "(%s)"
        | LispDottedList (head, tail) ->
            head |> unwordsList |> (sprintf "(%s . %s)")
            <| (tail.ToString())

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
and LispError =
    | NumArgs of int32 * List<LispVal>
    | TypeMismatch of string * LispVal
    | ParseError of string
    | BadSpecialForm of string * LispVal
    | NotFunction of string * string
    | UnboundVar of string * string
    | UnspecifiedReturn of string
    | DefaultError of string

    override this.ToString() =
        match this with
        | NumArgs (expected, found) ->
            sprintf
                "Expected %d args; found values: %s"
                expected
                (found
                 |> List.map (fun v -> v.ToString())
                 |> String.concat " ")
        | TypeMismatch (expected, found) -> sprintf "Invalid type expected %s, found %s" expected (found.ToString())
        | ParseError err -> sprintf "Parse error at %s" (err.ToString())
        | BadSpecialForm (message, form) -> sprintf "%s: %s" message (form.ToString())
        | NotFunction (message, func) -> sprintf "%s: %s" message func
        | UnboundVar (message, varname) -> sprintf "%s: %s" message varname
        | UnspecifiedReturn message -> message
        | DefaultError e -> sprintf "Error: %s" e

and Env = System.Collections.Generic.Dictionary<string, ref<LispVal>>
and ThrowsError<'T> = Result<'T, LispError>
```

And the `ToString` and `Equals` method should handle the new types too:

_ToString()_
```fsharp
        | LispPrimitiveFunc _ -> "<primitive>"
        | LispFunc (p, vargs, _, _) ->
            let paramStr =
                p
                |> List.map (fun v -> v.ToString())
                |> String.concat " "

            let vargStr =
                match vargs with
                | None -> ""
                | Some arg -> sprintf " . %s" arg

            sprintf """(lambda (%s%s) ...) """ paramStr vargStr
```
Instead of showing the full function, we just print out the word <primitive> for primitives and the header info for user-defined functions. This is an example of pattern-matching for records: as with normal algebraic types, a pattern looks exactly like a constructor call. Field names come first and the variables they'll be bound to come afterwards.

_Equals()_
```fsharp
            | (LispPrimitiveFunc f1, LispPrimitiveFunc f2) -> LanguagePrimitives.PhysicalEquality f1 f2
            | (LispFunc (params1, varg1, body1, closure1), LispFunc (params2, varg2, body2, closure2)) ->
                params1 = params2
                && varg1 = varg2
                && body1 = body2
                && closure1 = closure2
```

We use the `LanguagePrimitives.PhysicalEquality` to check the equality of `primitiveFunc`. It checkes the reference equality which simplifies the problem here.

Next, we need to change `apply`. Instead of being passed the name of a function, it'll be passed a `LispVal` representing the actual function. For primitives, that makes the code simpler: we need only read the function out of the value and apply it.

```fsharp
and apply f args =
    match f with
    | LispPrimitiveFunc fn -> fn args
```
The interesting code happens when we're faced with a user defined function. Records let you pattern match on both the field name (as shown above) or the field position, so we'll use the latter form:

```fsharp
and apply func args =
    match func with
    | LispPrimitiveFunc fn -> fn args
    | LispFunc (fparams, vargs, body, clojure) ->
        if fparams.Length <> args.Length && vargs = None then
            NumArgs(fparams.Length, args) |> throwError
        else
            let remainingArgs = List.drop fparams.Length args

            let evalBody (env: EnvType) =
                evalAllReturnLast env body

            let bindVarArgs arg env =
                match arg with
                | Some argName -> bindVars env (Map.empty.Add(argName, LispList remainingArgs))
                | None -> env

            let newEnv =
                List.zip fparams (List.take fparams.Length args)
                |> Map.ofList
                |> bindVars (new Env(clojure))
                |> bindVarArgs vargs

            evalBody newEnv
    | _ -> DefaultError "runtime error" |> throwError

and evalAllReturnLast env =
    function
    | [x] -> eval env x
    | x :: xs -> match eval env x with
                | Result.Error _ as v -> v
                | Result.Ok(_) -> evalAllReturnLast env xs
```

The very first thing this function does is check the length of the parameter list against the expected number of arguments. It throws an error if they don't match.

Assuming the call is valid, we do the bulk of the call in the pipeline that binds the arguments to a new environment and executes the statements in the body. The first thing we do is zip the list of parameter names and the list of (already evaluated) argument values together into a list of pairs. Then, we take that and the function's closure (not the current environment – this is what gives us lexical scoping) and use them to create a new environment to evaluate the function in. 

Now it's time to bind the remaining arguments to the `varargs` variable, using the local function `bindVarArgs`. If the function doesn't take `varargs` (the Nothing clause), then we just return the existing environment. Otherwise, we create a singleton map that has the variable name as the key and the remaining args as the value, and pass that to `bindVars`. We define the local variable `remainingArgs` for readability, using the function `List.drop` to ignore all the arguments that have already been bound to variables.

The final stage is to evaluate the body in this new environment. We use the local function `evalBody` for this, which maps the function `eval env` over every statement in the body, and then returns the value of the last statement.

Since we're now storing primitives as regular values in variables, we have to bind them when the program starts up:

```fsharp
let primitiveBindings () =
    Map.mapValues LispPrimitiveFunc primitives
    |> bindVars (nullEnv ())
```
This takes the initial `null` environment, makes a bunch of name/value pairs consisting of `LispPrimitiveFunc` wrappers, and then binds the new pairs into the new environment. We also want to change `runOne` and `runRepl` to `primitiveBindings` instead:

```fsharp

let runOne expr = evalAndPrint (primitiveBindings()) expr

let runRepl () =
    until ((=) "quit") (fun () -> readPrompt "Lisp>>>") (evalAndPrint (primitiveBindings ()))
```

Finally, we need to change the evaluator to support [lambda](https://schemers.org/Documents/Standards/R5RS/HTML/r5rs-Z-H-7.html#%_sec_4.1.4) and function [define](https://schemers.org/Documents/Standards/R5RS/HTML/r5rs-Z-H-8.html#%_sec_5.2). We'll start by creating a few helper functions to make it a little easier to create function objects.
```fsharp
let makeFunc varargs env (args: List<LispVal>) body =
    LispFunc((List.map (fun p -> p.ToString()) args), varargs, body, env)
    |> Result.Ok

let makeNormalFunc (envType: Env) args body = makeFunc None envType args body

let makeVarArgs =
    makeFunc << Some << (fun v -> v.ToString()
```
Here, `makeNormalFunc` and `makeVarArgs` should just be considered specializations of `makeFunc` with the first argument set appropriately for normal functions vs. variable args. This is a good example of how to use first-class functions to simplify code.

Now, we can use them to add our extra eval clauses. They should be inserted after the define-variable clause and before the function-application one:
```fsharp
    | LispList (LispAtom "define" :: LispList (LispAtom var :: args) :: body) ->
                makeNormalFunc env args body |> Result.bind (defineVar env var)
    | LispList (LispAtom "define" :: LispDottedList (LispAtom var :: args, LispAtom vargs) :: body) ->
                makeVarArgs vargs env args body |> Result.bind (defineVar env var)
    | LispList (LispAtom "lambda" :: LispList p :: body) ->
                makeNormalFunc env p body
    | LispList (LispAtom "lambda" :: LispDottedList (LispAtom var :: args, LispAtom vargs) :: body) ->
                makeVarArgs vargs env args body |> Result.bind (defineVar env var)
    | LispList (LispAtom "lambda" :: (LispAtom vargs) :: body) ->
                makeVarArgs vargs env [] body
```

The following needs to replace the prior function-application eval clause.
```fsharp
    | LispList (func :: args) ->
            monad {
                let! fn = eval env func
                let! argVals = mapM (eval env) args
                let! ret = apply fn argVals
                return ret
            }
```
As you can see, they just use pattern matching to destructure the form and then call the appropriate function helper. In define's case, we also feed the output into `defineVar` to bind a variable in the local environment. 

Here is just an another example to use the `monad` block to simplify the code.

We can now compile and run our program, and use it to write real programs!
```bash
$ dotnet run
Lisp>>>(define (f x y) (+ x y))
(lambda (x y) ...)
Lisp>>>(f 1 2)
3
Lisp>>>(f 1 2 3)
Eval failed: Expected 2 args; found values: 1 2 3
Lisp>>>(f 1)
Eval failed: Expected 2 args; found values: 1
Lisp>>>(define (factorial x) (if (= x 1) 1 (* x (factorial (- x 1)))))
(lambda (x) ...)
Lisp>>>(factorial 10)
3628800
Lisp>>>(define (counter inc) (lambda (x) (set! inc (+ x inc)) inc))
(lambda (inc) ...)
Lisp>>>(define my-count (counter 5))
(lambda (x) ...)
Lisp>>>(my-count 3)
8
Lisp>>>(my-count 6)
14
Lisp>>>(my-count 5)
19
Lisp>>>quit
```

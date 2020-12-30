# Adding Variables and Assignment
Finally, we get to the good stuff: variables. A variable lets us save the result of an expression and refer to it later. In Scheme, a variable can also be reset to new values, so that its value changes as the program executes. Doing this is not hard in F#, as in F# we have mutable variables and [reference cells](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/reference-cells).

let's start out by defining the type of our environments. Add it in `LispTypes.fs` and after the `LisaVal` definition.

```fsharp
type Env = System.Collections.Generic.Dictionary<string, ref<LispVal>>
```
This declares an Env as an Map holding a list that maps strings to `LispVals` references. There are two ways that a program can mutate the environment. It might use `set!` to change the value of an individual variable, a change visible to any function that shares that environment (Scheme allows nested scopes, so a variable in an outer scope is visible to all inner scopes). Or it might use `define` to add a new variable, which should be visible on all subsequent statements. It's important that we use `ref cells` for the individual values here in order to support sharing variables to inner scopes.

We use `Dictionary` instead of `Map` because `Map` is a inmutable data structure but `Dictionary` is mutable. It means we can update the `Dictionary` directly instead of creating a copy of it.

We define a helper function to create an empty environment:
```fsharp
let nullEnv () = new Env()
```

Now we're ready to return to environment handling. We'll start with a function to determine if a given variable is already bound in the environment, necessary for proper handling of define:

_Eval.fs_
```fsharp
let isBound (env: Env) var = env.ContainsKey(var)
```

It simply check whether the env contains the key.

Next, we'll want to define a function to retrieve the current value of a variable:

```fsharp
let getVar (env: Env) var =
    let mutable ret = ref (LispBool false)

    if env.TryGetValue(var, &ret) then
        Result.Ok !ret
    else
        UnboundVar("Getting an unbound variable", var)
        |> throwError
```
Here we declared a mutable variable `ret` and uses the `env.TryGetValue` to get the value from the `env`. And return an error if there is no such value in the env.

Now we create a function to set values:

```fsharp
let setVar (env: Env) var value =
    if isBound env var then
        env.[var] := value
        Result.Ok value
    else
        UnboundVar("Setting an unbound variable", var)
        |> throwError
```
Firstly we check if the var is defined in env and return an error if it's not defined. Then we use the `:=` operator to update the value in the reference cell. Finally, we return the value we just set, for convenience.

We'll want a function to handle the special behavior of `define`, which sets a variable if already bound or creates a new one if not. Since we've already defined a function to set values, we can use it in the former case:

```fsharp
let defineVar (env: Env) var value =
    env.[var] <- ref value
    Result.Ok value
```
This is simple, we just always creating a `ref cell` using the `ref` keyword and update it to the env.

There's one more useful environment function: being able to bind a whole bunch of variables at once, as happens when a function is invoked. We might as well build that functionality now, though we won't be using it until the next section:

```fsharp
let bindVars (env: Env) (vars: Map<string, LispVal>) =
    for kv in vars do
        env.[kv.Key] <- ref kv.Value

    env
```
It looks simliar to `defineVar` and we just use a `for...do` loop to looop over the values.

Now that we have all our environment functions, we need to start using them in the evaluator. We'll pass the environment to the `eval` function and make the changes as below and add the [set!](https://schemers.org/Documents/Standards/R5RS/HTML/r5rs-Z-H-7.html#%_sec_4.1.6) and [define](https://schemers.org/Documents/Standards/R5RS/HTML/r5rs-Z-H-8.html#%_sec_5.2) special forms as well.
```fsharp
let rec eval (env: Env)=
    function
    | LispString _ as v -> Result.Ok v
    | LispNumber _ as v -> Result.Ok v
    | LispBool _ as v -> Result.Ok v
    | LispAtom id -> getVar env id
    | LispList [ LispAtom "quote"; v ] -> Result.Ok v
    | LispList [ LispAtom "if"; pred; conseq; alt ] ->
        eval env pred
        |> Result.bind
            (fun v ->
                match v with
                | LispBool false -> eval env alt
                | _ -> eval env conseq)
    | LispList [ LispAtom "set!"; LispAtom var; form ] -> eval env form |> Result.bind (setVar env var)
    | LispList [ LispAtom "define"; LispAtom var; form ] -> eval env form |> Result.bind (defineVar env var)
    | LispList (LispAtom func :: args) -> args |> mapM (eval env) |> Result.bind (apply func)
    | badform ->
        BadSpecialForm("Unrecognized special form", badform)
        |> throwError
```
Since a single environment gets threaded through a whole interactive session, we need to change a few of our functions to take an environment.
```fsharp
let readAndEval env = readExpr >> Result.bind (eval env)

let evalString env expr =
    match readAndEval env expr with
    | Ok v -> v.ToString()
    | Error e -> sprintf "Eval failed: %s" (e.ToString())

let evalAndPrint env expr =
  evalString env expr |> Console.WriteLine
```
Next, we initialize the environment with a null variable before starting the program:
```fsharp

let runOne expr = evalAndPrint (nullEnv()) expr

let runRepl () =
    until ((=) "quit") (fun () -> readPrompt "Lisp>>>") (evalAndPrint (nullEnv()))
```
We've created an additional helper function runOne to handle the single-expression case, since it's now somewhat more involved than just running evalAndPrint. The changes to runRepl are a bit more subtle: notice how we added a function composition operator before evalAndPrint. That's because evalAndPrint now takes an additional Env parameter, fed from nullEnv. The function composition tells until_ that instead of taking plain old evalAndPrint as an action, it ought to apply it first to whatever's coming down the monadic pipeline, in this case the result of nullEnv. Thus, the actual function that gets applied to each line of input is (evalAndPrint env), just as we want it.

```fsharp
let main argv =
    if argv.Length = 0 then
        runRepl()
    else
        argv |> Array.tryHead |> Option.defaultValue ""
        |> runOne
    0 // return an integer exit code
```
And we can compile and test our program:

```bash
$ dotnet run
Lisp>>>(define x 3)
3
Lisp>>>(+ x 2)
5
Lisp>>>(+ y 2)
Eval failed: Getting an unbound variable: y
Lisp>>>(define y 5)
5
Lisp>>>(+ x (- y 2))
6
Lisp>>>(define str "A string")
"A string"
Lisp>>>(< str "The string")
Eval failed: Invalid type expected number, found "A string"
Lisp>>>(string<? str "The string")
#t
Lisp>>>quit
$
```

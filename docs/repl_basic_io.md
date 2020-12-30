# Building a REPL: Basic I/O
So far, we've been content to evaluate single expressions from the command line, printing the result and exiting afterwards. This is fine for a calculator, but isn't what most people think of as "programming". We'd like to be able to define new functions and variables, and refer to them later. But before we can do this, we need to build a system that can execute multiple statements without exiting the program.

Instead of executing a whole program at once, we're going to build a _read-eval-print_ loop. This reads in expressions from the console one at a time and executes them interactively, printing the result after each expression. Later expressions can reference variables set by earlier ones (or will be able to, after the next section), letting you build up libraries of functions.

First add a file `Repl.hs` to the fsharp project(placed right after `Eval.fs`).

We start with creating a function that prints out a prompt and reads in a line of input

 ```fsharp
 open System

let readPrompt (prompt: string) =
    Console.Write prompt
    let v = Console.ReadLine()
    if isNull v then "" else v
 ```

Pull the code to parse and evaluate a string and trap the errors out of main into its own function:

```fsharp
let readAndEval = readExpr >> Result.bind eval
let evalString expr =
    match readAndEval expr with
    | Ok v -> v.ToString()
    | Error e -> sprintf "Eval failed: %s" (e.ToString())
```
And write a function that evaluates a string and prints the result:
```fsharp
let evalAndPrint expr =
  evalString expr |> Console.WriteLine
```
Now it's time to tie it all together. We want to read input, perform a function, and print the output, all in an infinite loop. 

```fsharp
let rec until pred prompt action =
    let input = prompt()
    if not (pred input) then
      action input
      until pred prompt action
```

`until` takes a predicate that signals when to stop, an action to perform before the test, and a function-returning-an-action to do to the input. 

Now that we have all the machinery in place, we can write our REPL easily:
```fsharp
let runRepl () =
    until ((=) "quit") (fun () -> readPrompt "Lisp>>>") evalAndPrint
```

And change our `main` function so it either executes a single expression, or enters the REPL and continues evaluating expressions until we type quit:

```fsharp
let main argv =
    if argv.Length = 0 then
        runRepl()
    else
        let result = argv |> Array.tryHead |> Option.defaultValue ""
                     |> readExpr |> Result.bind eval
        match result with
        | Ok v -> printfn "%s" (v.ToString())
        | Error e -> printfn "error: %s" (e.ToString())
    0 // return an integer exit code
```
Compile and run the program, and try it out:
```bash
$ dotnet run
Lisp>>> (+ 2 3)
5
Lisp>>> (cons this '())
Eval failed: Unrecognized special form: this
Lisp>>> (cons 2 3)
(2 . 3)
Lisp>>> (cons 'this '())
(this)
Lisp>>> quit
$
```

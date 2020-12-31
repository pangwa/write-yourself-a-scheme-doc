# Creating IO Primitives
Our Scheme can't really communicate with the outside world yet, so it would be nice if we could give it some I/O functions. Also, it gets really tedious typing in functions every time we start the interpreter, so it would be nice to load files of code and execute them.

The first thing we'll need is a new constructor for `LispVals`. We want a dedicated constructor for primitive functions that perform IO, (this is not quite necessary in F# as any function can do IO and do not require the IO monad, but I kept this structure to distinguish pure and IO functions):
```fsharp
    | LispIOFunc of (List<LispVal> -> ThrowsError<LispVal>)
```

While we're at it, let's also define a constructor for the Scheme data type of a [port](https://schemers.org/Documents/Standards/R5RS/HTML/r5rs-Z-H-9.html#%_sec_6.6.1). Most of our IO functions will take one of these to read from or write to:

```fsharp
    | LispPort of FileHandle
```

where the `FileHandle` is defined as:
```fsharp
type FileHandle =
    | ReadHandle of System.IO.StreamReader
    | WriteHandle of System.IO.StreamWrite
```
A `FileHandle` is basically a stream in F#, for the `ReadHandle` it's a `StreamReader` and it's `StreamWrite` for `WriteHandle`. It's important to understand that we use different record constructors to distinguish different type of handle instead of implementing a generalized FileHandle class. `Data` and `Behavior` is kind of separated.

For completeness, we ought to provide `ToString()` and `Equals()` handlers:
```fsharp
// ..ToString:
        | LispIOFunc _ -> "<IO primitive>"
        | LispPort (ReadHandle _) -> "<IO port: read>"
        | LispPort (WriteHandle _) -> "<IO port: write>

// ...Equals:
        | (LispIOFunc f1, LispIOFunc f2) -> LanguagePrimitives.PhysicalEquality f1 f2
        | (LispPort p1, LispPort p2) -> p1 = p2
```

This'll let the REPL function properly and not crash when you use a function that returns a port.

We also need to update apply, so that it can handle `LispIOFuncs`:
```fsharp
and apply func args =
    match func with
    | LispPrimitiveFunc fn -> fn args
    | LispIOFunc fn -> fn args
    //....
```

We'll need to make some minor changes to our parser to support [load](https://schemers.org/Documents/Standards/R5RS/HTML/r5rs-Z-H-9.html#%_sec_6.6.4). Since Scheme files usually contain several definitions, we need to add a parser that will support several expressions, separated by whitespace. And it also needs to handle errors. We can reuse much of the existing infrastructure by factoring our basic `readExpr` so that it takes the actual parser as a parameter:

```fsharp
let readOrThrow parser input =
    match run parser input with
    | Success (v, _, _) -> Result.Ok(v)
    | Failure (err, _, _) ->
        sprintf "No match %s" err
        |> ParseError
        |> throwError

let readExpr = readOrThrow (spaces >>. parseExpr .>> spaces .>> eof)
let readExprList = readOrThrow (sepEndBy parseExpr spaces)
```
Again, think of both `readExpr` and `readExprList` as specializations of the newly-renamed `readOrThrow`. We'll be using `readExpr` in our REPL to read single expressions; we'll be using `readExprList` from within load to read programs.

Next, we'll want a new list of IO primitives, structured just like the existing primitive list:

```fsharp

let ioPrimitives =
    Map
        .empty
        .Add("apply", applyProc)
        .Add("open-input-file", makePort ModeRead)
        .Add("open-output-file", makePort ModeWrite)
        .Add("close-input-port", closePort)
        .Add("close-output-port", closePort)
        .Add("read", readProc)
        .Add("write", writeProc)
        .Add("read-contents", readContents)
        .Add("read-all", readAll
```
The type signature is actually the same as primitives. We defined it separately to be consistent with the original tutorial.

We also need to change the definition of `primitiveBindings` to add our new primitives:
```fsharp
let primitiveBindings () =
    Map.mapValues LispIOFunc ioPrimitives
    |> Map.union (Map.mapValues LispPrimitiveFunc primitives)
    |> bindVars (nullEnv ())
```

Now we start defining the actual functions. `applyProc` is a very thin wrapper around `apply`, responsible for destructuring the argument list into the form apply expects:

```fsharp
let applyProc =
  function
  | [func; LispList args] -> apply func args
  | func :: args -> apply func args
  | [] -> BadSpecialForm ("function", LispList []) |> throwError
```

`makePort` wraps the F#'s IO stream functions, converting them to the right type and passing them to the `LispPort` constructor. It's intended to be partially-applied to the `FileMode`, `ModeRead` for open-input-file and `ModeWrite` for open-output-file:
```fsharp
type FileMode =
    | ModeRead
    | ModeWrite

let runIOSafe<'T> (action: unit -> 'T) =
  try
    action() |> Result.Ok
  with
    | e -> e.ToString() |> DefaultError |> throwError

let makePort mode args =
    match args with
    | [LispString filename] ->
        match mode with
        | ModeRead -> runIOSafe (fun () -> new System.IO.StreamReader(filename) |> ReadHandle |> LispPort)
        | ModeWrite -> runIOSafe (fun () -> new System.IO.StreamWriter(filename) |> WriteHandle |> LispPort)
    | v -> TypeMismatch ("makePort: filename", LispList v) |> throwError
```
We defined the `FileMode` type which has read and write modes.

We also defined a helper function `runIOSafe` to ensure all IO actions not throwing any exceptions as we're using the .Net frameworks IO library, they will throw out exceptions instead of return as an `Error`.

Now let's extend the `FileHandle` class to add a member function `Close` for it.
```fsharp
type FileHandle with
    member this.Close () = match this with
                           | ReadHandle reader -> runIOSafe (fun () -> reader.Dispose())
                           | WriteHandle writer -> runIOSafe (fun () -> writer.Dispose())

```
We used the [type extension](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/type-extensions) feature of F# to add new members to a previously defined object type.

`closePort` also wraps the 'FileHandle.Close()' method:
```fsharp
let closePort  =
  function
  | [LispPort p] -> p.Close() |> Result.map (fun _ -> LispBool true)
  | _ -> LispBool false |> Result.Ok
```

`readProc` wraps the `StreamReader.ReadLine()` and then sends the result to `parseExpr`, to be turned into a `LispVal` suitable for Scheme:

```fsharp
let makePortFromStdIn () =
     runIOSafe (fun () -> new System.IO.StreamReader(System.Console.OpenStandardInput()) |> ReadHandle |> LispPort)

let safeIO<'T> (fn: LispVal -> ThrowsError<'T>) =
    function
    | LispPort handle as v ->
        match fn v with
        | Result.Ok(v) ->
                handle.Close() |> ignore
                Result.Ok(v)
        | Result.Error(e) ->
                handle.Close() |> ignore
                Result.Error(e)
    | _ -> DefaultError "safeIO: unexpected param" |> Result.Error

let rec readProc =
  function
  | [] -> makePortFromStdIn() |> Result.bind (safeIO (fun p -> readProc [p]))
  | [LispPort (ReadHandle reader)] ->
            runIOSafe (fun () -> reader.ReadLine()) |> Result.map LispString
  | args  -> BadSpecialForm("readProc", LispList args) |> throwError
```

`readProc` opens the `stdio` if it's called without args or it'ss called with with a `ReadHandle`. Here we implemented two helpers: `makePortFromStdIn` which makes a `StreamReader` from standard input. The `safeIO` function take an action and a `LispPort`, it applys the action to the port and then it ensure the port is closed.

`writeProc` converts a `LispVal` to a string and then writes it out on the specified port:

```fsharp
let rec writeProc =
  function
  | [obj] -> makePortFromStdIn() |> Result.bind (safeIO (fun p -> writeProc [obj; p]))
  | [obj; LispPort (WriteHandle writer)] ->
            runIOSafe (fun () -> writer.Write(sprintf "%s" (obj.ToString()))) |> Result.map (fun _ -> LispBool true)
  | args  -> BadSpecialForm("readProc", LispList args) |> throwError
```

`readContents` reads the whole file into a string in memory. It's a thin wrapper around .net's `System.IO.File.ReadAllText` function:
```fsharp
let readFile filename = runIOSafe (fun() -> System.IO.File.ReadAllText(filename))

let readContents =
    function
    | [LispString filename] -> readFile filename |> Result.map LispString
    | args  -> BadSpecialForm("readContents", LispList args) |> throwError
```

The helper function `load` doesn't do what Scheme's load does (we handle that later). Rather, it's responsible only for reading and parsing a file full of statements. It's used in two places: `readAll` (which returns a list of values) and `load` (which evaluates those values as Scheme expressions).

```fsharp
let load filename = readFile filename |> Result.bind readExprList
```

`readAll` then just wraps that return value with the `LispList` constructor:
```fsharp
let readAll =
    function
    | [LispString filename] -> load filename |> Result.map LispList
    | args  -> BadSpecialForm("readAll", LispList args) |> throwError
```

Implementing the actual Scheme `load` function is a little tricky, because `load` can introduce bindings into the local environment. Apply, however, doesn't take an environment argument, and so there's no way for a primitive function (or any function) to do this. We get around this by implementing `load` as a special form:

```fsharp
//eval = ...
  | LispList [Atom "load"; LispString filename] ->
             load filename |> Result.bind (mapM (eval env)) |> Result.map List.last
  
```

Finally, we might as well change our `runOne` function so that instead of evaluating a single expression from the command line, it takes the name of a file to execute and runs that as a program. Additional command-line arguments will get bound into a list args within the Scheme program:

```fsharp
let runOne args = 
  let env = List.tail args |> List.map LispString |> LispList
            |> fun x -> Map.add "args" x (Map.empty) 
            |> bindVars (primitiveBindings())
  match eval env (LispList [LispAtom "load"; LispString (args.[0])]) with
  | Ok v -> Console.WriteLine(v)
  | Error e -> sprintf "Eval failed: %s" (e.ToString()) |> Console.WriteLin
```
That's a little involved, so let's go through it step-by-step. The first line takes all tail of the args but the first and convert them to a `LispList` . Then it creates a Map contains one key `args` and the value is the `LispList` and bind `args` to the original primitive bindings. Then it creates a Scheme form `load "args1"`, just as if the user had typed in it, and evaluates it. The result is transformed to a string and write to the stdout as it did before. In this case, we'll also be printing the return value of the last statement in the program, which generally has no meaning to anything.)

Then we change main so it uses our new `runOne` function. 

```fsharp
let main argv =
    if argv.Length = 0 then
        runRepl()
    else
        argv |> List.ofArray |>runOne 
    0 // return an integer exit code
```

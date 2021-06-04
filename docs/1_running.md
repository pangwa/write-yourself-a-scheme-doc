# First Steps: Compiling and running
First, you'll need to install .NET. .NET is a free, cross-platform, open source developer platform for building many different types of applications. It's downloadable from https://dotnet.microsoft.com/download. Follow the install instructions on the offical site (which contains the latest and up to date information) to install it.

For Windows users, Visual Studio Communtity Version is a pretty great IDE to develop F# applications. VS Code with the [Ionide Extensions](https://ionide.io/Editors/Code/getting_started.html) extension installed is also a great alternative to use on other platforms, they can be used on MacOS or Linux.


Now, it's time for your first F# program. This program will read a name off of the command line and then print a greeting. Create a new project using command `dotnet new console -lang F# --name write-yourself-a-scheme`. You can find more on the `dotnet new` command by checking its [documentation](https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-new)

Go to the directory using `cd write-yourself-a-scheme`, edit `Program.fs` and update its content to the code shown below:
```fsharp
open System

[<EntryPoint>]
let main argv =
    let who = if argv.Length = 0 then "F#" else argv.[0]
    printfn "Hello world from %s" who
    0 // return an integer exit code
```

Let's go through this code. The first line specifies that we open the System module. The next section is the `[<EntryPoint>]` annotation, every F# program must have one entrypoint which is specified by the `[<>]` annotation. In F# these kinds of annotations are called attributes. Attributes enable metadata to be applied to a programming construct. We'll later see another annotation to mark a function for unit testing. You can check the [documentation on attributes](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/attributes) for futher understanding.

The line `let main argv = ` is a type declaration: it says that main is a function that takes one argument. In F# functions are normally just defined as variables; they both use the `let` keyword. Then we associate the `who` name to either 'F#' or the first command line argument passed in. `argv` is an array of arguments user passed while executing the program. You may notice that we didn't specify the `argv` type or the return value type of the main function, F# has a very strong [type inference](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/type-inference) feature and normally it will infer the types based on the context. Most times you don't have to specify the types explictly while writing F# program. You can still apply explict type annotations if it's really necessary.

Readers with a background in another programming language might also find we're using single equal sign to test the equality. And yes! In F#, equality is checked using a single `=` not `==`!

We use the `printfn` function to write text to the console which was provided by the System module. F# is not a pure functional programming language, unlike Haskell we don't have to provide an IO monad to perform IO operations in F#.

To compile and run the program, try something like this:
```bash
$ dotnet run
$ dotnet run -- jacky
```

The command `dotnet run` compiles and runs the program. And in the second command we use the `--` to pass arguments to the program.

## Exercises
1. Change the program so it reads two arguments from the command line, and prints out a message using both of them
1. Change the program so it performs a simple arithmetic operation on the two arguments and prints out the result. You can use `Int32.tryParse` to convert a string to a number, and `sprintfn` function to convert it to string. Play around with different operations.
1. Console.ReadLine() reads a line from the console and returns it as a string. Change the program so it prompts for a name, reads the name, and then prints that instead of the command line value

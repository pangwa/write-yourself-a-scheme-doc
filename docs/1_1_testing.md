# Testing
.Net has a very simple unit testing system. We're going to go through it quickly using a simple example.

First you need add some required packages using the `dotnet add package` command:
```bash
dotnet add package FSUnit
dotnet add package NUnit
dotnet add package NUnit3TestAdapter
dotnet add package Microsoft.Net.Test.Sdk
```

Add below code to your `Program.fs`, they should be placed at the `[<EntryPoint>]` annotation.

```fsharp
open NUnit.Framework
open FsUnit

[<Test>]
let ``test hello`` () =
    5 + 1 |> should equal 6
```

Here we opened two modules for testing `NUnit.Framework` and `FsUnit`. We declared a testing method whose name is `test hello` and it add one assertion that 5 + 1 should equal to 6. We use the `[<Test>]` annotation to mark this method is a unit test method. 

You can test it using command:
```bash
dotnet test
```
You should get a report contains passed and failed tests likes below:
```
Passed! - Failed:     0, Passed:     1, Skipped:     0, Total:     1, Duration: 13 ms- /Users/pangwa/fun/hello-world/bin/Debug/net5.0/hello-world.dll (net5.0)
```

You may found that we named the function name as `test hello` which contains whitespaces! In F#, names can include spaces and the must be quoted using double `` ` ``.  and for functions which takes no argumetns, we need add an explict `()` after its name to distinguish it with values.

We also used the `|>` operator which is the 'pipe' operator that feeds the result of the left expression to the function at the right side. so `5 + 1 |> should equal 6` equals to `should equal 6 (5 + 1)`.

## Exercises
1. Modify the test to make it fail and run it, check the outputs.

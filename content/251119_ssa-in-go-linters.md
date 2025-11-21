---
title: "Deep Dive: Static Single Assignment (SSA) in Go Linters"
date: 2025-11-19
description: "Understanding how SSA form powers advanced static analysis and custom linters in Go."
draft: false
params:
  author: Nick Westendorf
tags: ["go", "golang", "static analysis", "ssa", "compilers"]
---

When writing custom linters or static analysis tools for Go, you often start with the Abstract Syntax Tree (AST). The AST is great for syntax-level checks—like ensuring variable names follow a convention or checking for missing comments. However, when you need to understand the *behavior* of the code—data flow, control flow, or value propagation the AST becomes cumbersome.

This is where **Static Single Assignment (SSA)** form comes in.

## What is SSA?

Static Single Assignment is an intermediate representation (IR) used by compilers (like LLVM and the Go compiler itself) to optimize code. The core property of SSA is simple but powerful: **every variable is assigned exactly once.**

In standard imperative programming, we reassign variables all the time:

```go
x := 1
x = 2
y := x
```

In SSA form, this would look something like this (conceptually):

```text
x_1 = 1
x_2 = 2
y_1 = x_2
```

By versioning the variables, we make the data flow explicit. If we want to know where `y_1` got its value, we just look at the definition of `x_2`. We don't need to scan backwards through the code to see if `x` was changed in between.

### Phi Nodes

The magic of SSA happens at control flow merge points (like after an `if` statement). If a variable `x` can have a value from the `if` branch or the `else` branch, SSA introduces a special function called a **Phi (Φ) node**.

```go
// Original Go
var x int
if condition {
    x = 1
} else {
    x = 2
}
return x
```

```text
// SSA Form
if condition goto BlockA else goto BlockB

BlockA:
    x_1 = 1
    goto BlockMerge

BlockB:
    x_2 = 2
    goto BlockMerge

BlockMerge:
    x_3 = phi(x_1, x_2)
    return x_3
```

The `phi` instruction says: "If we came from BlockA, the value is `x_1`. If we came from BlockB, the value is `x_2`."

## Why Use SSA for Linters?

Using the `golang.org/x/tools/go/ssa` package allows you to build much more sophisticated tools than `go/ast` alone.

1.  **Control Flow Graph (CFG):** SSA provides a built-in graph of basic blocks. You can easily analyze reachability, loops, and branching logic.
2.  **Data Flow Analysis:** Because of the "single assignment" property, "def-use" chains (finding where a variable is defined and where it is used) are trivial.
3.  **Type Information:** The Go SSA package integrates deeply with the type checker. Every instruction has precise type information.

## Building a Tool with `go/ssa`

To use SSA, you typically follow these steps:

1.  **Load the Program:** Use `golang.org/x/tools/go/packages` to load the source code, parse it, and type-check it.
2.  **Create SSA Builder:** Use `ssa.NewProgram` with the loaded packages.
3.  **Build:** Call `program.Build()` to generate the SSA IR for all functions.

### Example: Inspecting Function Calls

Imagine we want to find every call to a specific function `danger.Zone()` and analyze the arguments passed to it. In the AST, this requires traversing nested expressions. In SSA, it's a flat list of instructions.

Here is a simplified conceptual snippet of how you might traverse SSA:

```go
import (
    "golang.org/x/tools/go/ssa"
    "golang.org/x/tools/go/ssa/ssautil"
)

func Analyze(pkg *ssa.Package) {
    // Iterate over all functions in the package
    for _, member := range pkg.Members {
        if fn, ok := member.(*ssa.Function); ok {
            // Iterate over all basic blocks in the function
            for _, block := range fn.Blocks {
                // Iterate over all instructions in the block
                for _, instr := range block.Instrs {
                    // Check if instruction is a function call
                    if call, ok := instr.(*ssa.Call); ok {
                        analyzeCall(call)
                    }
                }
            }
        }
    }
}

func analyzeCall(call *ssa.Call) {
    // We can inspect the Common().Value to see what's being called
    // and Common().Args to see the arguments.
    // Since these are SSA values, we can trace them back to their origins!
}
```

### Real-World Use Case: Taint Analysis

One of the most powerful applications of SSA is **taint analysis**. This involves tracking "tainted" data (e.g., user input) as it flows through the program to ensure it doesn't reach sensitive sinks (e.g., SQL execution) without being sanitized.

In SSA, this becomes a graph traversal problem:
1.  **Source:** Identify instructions that introduce tainted data (e.g., `http.Request.FormValue`).
2.  **Propagation:** Follow the def-use chains. If `x` is tainted and `y = x + 1`, then `y` is also tainted.
3.  **Sink:** Check if any tainted value reaches a dangerous function (e.g., `sql.Exec`).

Because SSA handles control flow merges (via Phi nodes) and variable versioning, you don't need to manually track complex branching logic. The graph structure does the heavy lifting for you.

## The Trade-off

While powerful, SSA is more expensive to compute than the AST. It requires full type checking, which can be slow for large codebases. It also has a steeper learning curve.

**Use AST when:**
*   You care about code style or formatting.
*   You only need local, syntactic context.
*   Speed is the absolute priority.

**Use SSA when:**
*   You need to track values across functions or blocks.
*   You are detecting bugs like nil pointer dereferences, race conditions, or taint analysis.
*   You need to understand the *logic* of the program.

## Conclusion

The `go/ssa` package is a superpower for Go tooling developers. It bridges the gap between simple syntax checking and deep program analysis. If you've hit the limits of what `go/ast` can do, it's time to embrace the graph.
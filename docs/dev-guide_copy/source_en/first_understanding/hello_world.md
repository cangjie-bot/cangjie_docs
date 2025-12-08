# Running Your First Cangjie Program

Everything is ready—let's start writing and running your first Cangjie program!

First, create a new text file named `hello.cj` in an appropriate directory and write the following Cangjie code into it:

<!-- verify -->

```cangjie
// hello.cj
main() {
    println("Hello, Cangjie")
}
```

In this code snippet, we use Cangjie's comment syntax. Single-line comments can be written after the `//` symbol, while multi-line comments can be written between `/*` and `*/` symbols, which is identical to the comment syntax in languages like C/C++. Comment content does not affect program compilation and execution.

Next, execute the following command in this directory:

```bash
cjc hello.cj -o hello
```

Here, the Cangjie compiler will compile the source code in `hello.cj` into an executable file named `hello` for the current platform. When you run this file in a command-line environment, you'll see the program output the following content:

```text
Hello, Cangjie
```

> **Note:**
>
> The above compilation command is for Linux and macOS platforms. If you're using Windows, simply modify the compilation command to `cjc hello.cj -o hello.exe`.
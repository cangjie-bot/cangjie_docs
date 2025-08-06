# Formatting Tool

## Feature Overview

`CJFMT (Cangjie Formatter)` is a code auto-formatting tool developed based on the Cangjie language programming specifications.

## Usage Instructions

Command line operation: `cjfmt [option] file [option] file`

`cjfmt -h` displays help information and option descriptions

```text
Usage:
     cjfmt -f fileName [-o fileName] [-l start:end]
     cjfmt -d fileDir [-o fileDir]
Options:
   -h            Show usage
                     eg: cjfmt -h
   -v            Show version
                     eg: cjfmt -v
   -f            Specifies the file in the required format. The value can be a relative path or an absolute path.
                     eg: cjfmt -f test.cj
   -d            Specifies the file directory in the required format. The value can be a relative path or an absolute path.
                     eg: cjfmt -d test/
   -o <value>    Output. If a single file is formatted, '-o' is followed by the file name. Relative and absolute paths are supported;
                 If a file in the file directory is formatted, a path must be added after -o. The path can be a relative path or an absolute path.
                     eg: cjfmt -f a.cj -o ./fmta.cj
                     eg: cjfmt -d ~/testsrc -o ./testout
   -c <value>    Specify the format configuration file, relative and absolute paths are supported.
                 If the specified configuration file fails to be read, cjfmt will try to read the default configuration file in CANGJIE_HOME.
                 If the default configuration file also fails to be read, will use the built-in configuration.
                     eg: cjfmt -f a.cj -c ./config/cangjie-format.toml
                     eg: cjfmt -d ~/testsrc -c ~/home/project/config/cangjie-format.toml
   -l <region>   Only format lines in the specified region for the provided file. Only valid if a single file was specified.
                 Region has a format of [start:end] where 'start' and 'end' are integer numbers representing first and last lines to be formated in the specified file.
                 Line count starts with 1.
                     eg: cjfmt -f a.cj -o ./fmta.cj -l 1:25
```

### File Formatting

`cjfmt -f`

- Format and overwrite the source file, supporting relative and absolute paths.

```shell
cjfmt -f ../../../test/uilang/Thread.cj
```

- Option `-o` creates a new `.cj` file to export the formatted code. Both source and output files support relative and absolute paths.

```shell
cjfmt -f ../../../test/uilang/Thread.cj -o ../../../test/formated/Thread.cj
```

### Directory Formatting

`cjfmt -d`

- Option `-d` allows developers to specify a Cangjie source code directory for scanning and formatting all Cangjie source files within the folder, supporting relative and absolute paths.

```shell
cjfmt -d test/              // Source directory as relative path

cjfmt -d /home/xxx/test     // Source directory as absolute path
```

- Option `-o` specifies the output directory, which can be an existing path. If it doesn't exist, the relevant directory structure will be created. Both relative and absolute paths are supported. The maximum path length (MAX_PATH) varies across systems. For example, on `Windows`, this value generally cannot exceed 260 characters, while on `Linux`, it is recommended not to exceed 4096 characters.

```shell
cjfmt -d test/ -o /home/xxx/testout

cjfmt -d /home/xxx/test -o ../testout/

cjfmt -d testsrc/ -o /home/../testout   // Error if source directory doesn't exist: "error: Source file path not exist!"
```

### Formatting Configuration File

`cjfmt -c`

- Option `-c` allows developers to specify a custom formatting tool configuration file.

```shell
cjfmt -f a.cj -c ./cangjie-format.toml
```

The default cangjie-format.toml configuration file contains the following settings, which also serve as the built-in configuration options for the `cjfmt` tool:

```toml
# indent width
indentWidth = 4 # Range of indentWidth: [0, 8]

# limit length
linelimitLength = 120 # Range of indentWidth: [1, 120]

# line break type
lineBreakType = "LF" # "LF" or "CRLF"

# allow Multi-line Method Chain when it's level equal or greater than multipleLineMethodChainLevel
allowMultiLineMethodChain = false

# if allowMultiLineMethodChain's value is true,
# and method chain's level is equal or greater than multipleLineMethodChainLevel,
# method chain will be formatted to multi-line method chain.
# e.g. A.b().c() level is 2, A.b().c().d() level is 3
# ObjectA.b().c().d().e().f() =>
# ObjectA
#     .b()
#     .c()
#     .d()
#     .e()
#     .f()
multipleLineMethodChainLevel = 5 # Range of multipleLineMethodChainLevel: [2, 10]

# allow Multi-line Method Chain when it's length greater than linelimitLength
multipleLineMethodChainOverLineLength = true
```

> **Note:**
>
> If the custom formatting tool configuration file fails to load, the tool will attempt to read the default configuration file `cangjie-format.toml` from the CANGJIE_HOME environment.
> If the default configuration file in CANGJIE_HOME also fails to load, the tool will use the built-in formatting configuration options.
> If any configuration option in the formatting tool configuration file fails to load, that option will use the built-in formatting configuration.

### Partial Formatting

`cjfmt -l`

- Option `-l` allows developers to specify a portion of the file to be formatted. The formatter will only apply rules to the source code within the provided line range.
- The `-l` option is only applicable when formatting a single file (option `-f`). If a directory is specified (option `-d`), the `-l` option will be ignored.

```shell
cjfmt -f a.cj -o .cj -l 10:25 // Only formats lines 10 to 25
```

## Formatting Rules

- A source file sequentially includes copyright, package, import, and top-level elements, separated by blank lines.

[Correct Example]

```cangjie
// Part 1: Copyright information
/*
 * Copyright (c) [Year of First Pubication]-[Year of Latest Update]. [Company Name]. All rights reserved.
 */

// Part 2: Package declaration
package com.myproduct.mymodule

// Part 3: Import declarations
import std.collection.HashMap   // Standard library

// Part 4: Public element definitions
public class ListItem <: Component {
    // CODE
}

// Part 5: Internal element definitions
class Helper {
    // CODE
}
```

> **Note:**
>
> The Cangjie formatter does not enforce blank lines between the copyright section and other sections. If developers leave one or more blank lines below the copyright information, the formatter will retain one blank line.

- Use consistent indentation with 4 spaces per level.

[Correct Example]

```cangjie
class ListItem {
    var content: Array<Int64> // Correct: 4-space indentation relative to class declaration
    init(
        content: Array<Int64>, // Correct: 4-space indentation for function parameters relative to function declaration
        isShow!: Bool = true,
        id!: String = ""
    ) {
        this.content = content
    }
}
```

- Use a unified brace style. For non-empty block structures, use the K&R style.

[Correct Example]

```cangjie
enum TimeUnit { // Correct: Placed at the end of the declaration line with one preceding space
    Year | Month | Day | Hour
} // Correct: Closing brace on its own line

class A { // Correct: Placed at the end of the declaration line with one preceding space
    var count = 1
}

func fn(a: Int64): Unit { // Correct: Placed at the end of the declaration line with one preceding space
    if (a > 0) { // Correct: Placed at the end of the declaration line with one preceding space
    // CODE
    } else { // Correct: Closing brace and 'else' on the same line
        // CODE
    } // Correct: Closing brace on its own line
}

// Lambda function
let add = {
    base: Int64, bonus: Int64 => // Correct: Non-empty blocks in lambda expressions follow K&R style
    print("Correct news")
    base + bonus
}
```

- Follow rule G.FMT.10 from the Cangjie language programming specifications to use spaces to highlight keywords and important information.

[Correct Example]

```cangjie
var isPresent: Bool = false  // Correct: One space after the colon in variable declarations
func method(isEmpty!: Bool): RetType { ... } // Correct: One space after colons in function definitions (named parameters/return types)

method(isEmpty: isPresent) // Correct: One space after the colon in named parameter passing

0..MAX_COUNT : -1 // Correct: No spaces around range operators, one space before and after the step colon

var hundred = 0
do { // Correct: One space between the 'do' keyword and the following brace
    hundred++
} while (hundred < 100) // Correct: One space between the 'while' keyword and the preceding brace

func fn(paramName1: ArgType, paramName2: ArgType): ReturnType { // Correct: No spaces between parentheses and adjacent internal characters
    ...
    for (i in 1..4) { // Correct: No spaces around range operators
        ...
    }
}

let listOne: Array<Int64> = [1, 2, 3, 4] // Correct: No spaces inside square or round brackets

let salary = base + bonus // Correct: Spaces around binary operators

x++ // Correct: No spaces between unary operators and operands
```

- Minimize unnecessary blank lines to keep code compact.

[Incorrect Example]

```cangjie
class MyApp <: App {
    let album = albumCreate()
    let page: Router
    // Blank line
    // Blank line
    // Blank line
    init() {           // Incorrect: Consecutive blank lines within type definitions
        this.page = Router("album", album)
    }

    override func onCreate(): Unit {

        println( "album Init." )  // Incorrect: Blank lines inside braces

    }
}
```

- Minimize unnecessary semicolons, prioritizing code conciseness.

[Before Formatting]

```cangjie
package demo.analyzer.filter.impl; // Redundant semicolon

internal import demo.analyzer.filter.StmtFilter; // Redundant semicolon
internal import demo.analyzer.CJStatment; // Redundant semicolon

func fn(a: Int64): Unit {
    println( "album Init." );
}
```

[After Formatting]

```cangjie
package demo.analyzer.filter.impl // Redundant semicolon removed

internal import demo.analyzer.filter.StmtFilter // Redundant semicolon removed
internal import demo.analyzer.CJStatment // Redundant semicolon removed

func fn(a: Int64): Unit {
    println("album Init.");
}
```

- Arrange modifier keywords according to the priority specified in rule G.FMT.12 of the Cangjie language programming specifications.Here is the recommended modifier ordering priority for top-level elements:

```cangjie
public
open/abstract
```

Here is the recommended modifier ordering priority for instance member functions or instance member properties:

```cangjie
public/protected/private
open
override
```

Here is the recommended modifier ordering priority for static member functions:

```cangjie
public/protected/private
static
redef
```

Here is the recommended modifier ordering priority for member variables:

```cangjie
public/protected/private
static
```

- Formatting behavior for multi-line comments

For comments starting with `*`, the `*` symbols will be aligned with each other. For comments not starting with `*`, the original comment format will be preserved.

```cangjie
// Before formatting
/*
      * comment
      */

/*
        comment
        */

// After formatting
/*
 * comment
 */

/*
        comment
 */
```

## Notes

- The Cangjie formatting tool currently does not support formatting code with syntax errors.

- The Cangjie formatting tool currently does not support metaprogramming formatting.
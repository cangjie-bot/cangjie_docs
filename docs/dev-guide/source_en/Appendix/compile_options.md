# `cjc` Compilation Options

This chapter introduces the commonly used `cjc` compilation options. If an option also applies to `cjc-frontend`, it will be marked with the <sup>[frontend]</sup> superscript; if the behavior of an option differs between `cjc-frontend` and `cjc`, additional explanations will be provided.

- Options starting with two hyphens are long options, such as `--xxxx`.
  If a long option has an optional parameter, the option and the parameter must be connected with an equals sign, e.g., `--xxxx=<value>`.
  If a long option has a mandatory parameter, the option and the parameter can be separated by a space or connected with an equals sign, e.g., `--xxxx <value>` is equivalent to `--xxxx=<value>`.

- Options starting with one hyphen are short options, such as `-x`.
  For short options that require a parameter, the option and the parameter can be separated by a space or concatenated directly, e.g., `-x <value>` is equivalent to `-x<value>`.

## Basic Options

### `--output-type=[exe|staticlib|dylib]` <sup>[frontend]</sup>

Specifies the type of output file. In `exe` mode, an executable file is generated; in `staticlib` mode, a static library file (`.a` file) is generated; in `dylib` mode, a dynamic library file is generated (`.so` file on Linux, `.dll` file on Windows, `.dylib` file on macOS).

The default mode for `cjc` is `exe`.

In addition to compiling `.cj` files into an executable, you can also compile them into a static or dynamic link library. For example:

```shell
$ cjc tool.cj --output-type=dylib
```

This command compiles `tool.cj` into a dynamic link library. On Linux, `cjc` generates a dynamic link library file named `libtool.so`.

**Note:** When linking against Cangjie's dynamic library files to compile an executable program, you must specify the `--dy-std` option. For details, see the [`--dy-std` option description](#--dy-std).

<sup>[frontend]</sup> In `cjc-frontend`, the compilation process only proceeds to `LLVM IR`, so the output is always a `.bc` file. However, different `--output-type` values still affect the frontend compilation strategy.

### `--package`, `-p` <sup>[frontend]</sup>

Compiles a package. When using this option, you need to specify a directory as input, and the source files in the directory must belong to the same package.

Assume there is a file `log/printer.cj`:

<!-- compile -p -->
<!-- cfg="-p log --output-type=staticlib" -->

```cangjie
package log

public func printLog(message: String) {
    println("[Log]: ${message}")
}
```

And a file `main.cj`:

<!-- compile -p -->
<!-- cfg="liblog.a" -->

```cangjie
import log.*

main() {
    printLog("Everything is great")
}
```

You can use the following command to compile the `log` package:

```shell
$ cjc -p log --output-type=staticlib
```

`cjc` generates a `liblog.a` file in the current directory.

You can then use the `liblog.a` file to compile `main.cj` with the following command:

```shell
$ cjc main.cj liblog.a
```

`cjc` compiles `main.cj` together with `liblog.a` into an executable file named `main`.

### `--module-name <value>` <sup>[frontend]</sup>

> **Note:**
>
> This option is deprecated and will be removed in a future version. Using this option has no functional effect in the current version.

### `--output <value>`, `-o <value>`, `-o<value>` <sup>[frontend]</sup>

Specifies the path of the output file. The compiler's output is written to the specified file.

For example, the following command specifies the name of the output executable file as `a.out`:

```shell
cjc main.cj -o a.out
```

### `--library <value>`, `-l <value>`, `-l<value>`

Specifies the library files to link against.

The given library files are passed directly to the linker. This compilation option generally needs to be used together with `--library-path <value>`.

The filename format should be `lib[arg].[extension]`. When you need to link against library `a`, you can use the option `-l a`. The linker searches for files such as `liba.a` and `liba.so` (or `liba.dll` when linking for Windows targets) in the library file search directories and links them to the final output as needed.

### `--library-path <value>`, `-L <value>`, `-L<value>`

Specifies the directory where the library files to be linked are located.

When using the `--library <value>` option, you usually need to use this option to specify the directory containing the library files to be linked.

The paths specified by `--library-path <value>` are added to the linker's library file search path. Additionally, paths specified in the `LIBRARY_PATH` environment variable are also added to the linker's library file search path. Paths specified via `--library-path` have higher priority than those in `LIBRARY_PATH`.

Assume there is a dynamic library file `libcProg.so` compiled from the following C source file using a C compiler:

```c
#include <stdio.h>

void printHello() {
    printf("Hello World\n");
}
```

And a Cangjie file `main.cj`:

<!-- code_check_manual -->

```cangjie
foreign func printHello(): Unit

main(): Int64 {
  unsafe {
    printHello()
  }
  return 0
}
```

You can use the following command to compile `main.cj` and specify the `cProg` library to link against:

```shell
cjc main.cj -L . -l cProg
```

`cjc` outputs an executable file `main`.

Executing `main` produces the following output:

```shell
$ LD_LIBRARY_PATH=.:$LD_LIBRARY_PATH ./main
Hello World
```

**Note:** Since a dynamic library is used, you need to add the directory containing the library file to `LD_LIBRARY_PATH` to ensure that `main` can be dynamically linked at runtime.

### `-g` <sup>[frontend]</sup>

Generates an executable file or library file with debug information.

> **Note:**
>
> `-g` can only be used with `-O0`. Using a higher optimization level may cause debugging functionality to behave abnormally.

### `--trimpath <value>` <sup>[frontend]</sup>

Removes the prefix of the source file path information from the debug information.

When compiling Cangjie code, `cjc` saves the absolute path information of the source files (`.cj` files) to provide debugging and exception information at runtime.

Using this option allows you to remove the specified path prefix from the source file path information. The source file path information in the output files generated by `cjc` will not include the user-specified part.

You can use `--trimpath` multiple times to specify multiple different path prefixes. For each source file path, the compiler removes the first matching prefix from the path.

### `--coverage` <sup>[frontend]</sup>

Generates an executable program that supports code coverage statistics. The compiler generates a code information file with the suffix `.gcno` for each compilation unit. After executing the program, an execution statistics file with the suffix `.gcda` is generated for each compilation unit. Using these two files together with the `cjcov` tool, you can generate a code coverage report for the current execution.

> **Note:**
>
> `--coverage` can only be used with `-O0`. If a higher optimization level is used, the compiler will issue a warning and force the use of `-O0`. `--coverage` is used to compile and generate executable programs. If used to generate static or dynamic libraries, link errors may occur when the library is ultimately used.

### `--int-overflow=[throwing|wrapping|saturating]` <sup>[frontend]</sup>

Specifies the overflow strategy for fixed-precision integer operations. The default is `throwing`.

- In `throwing` strategy, an exception is thrown when an integer operation overflows.
- In `wrapping` strategy, when an integer operation overflows, it wraps around to the corresponding integer type.
- In `saturating` strategy, when an integer operation overflows, the extreme value of the corresponding fixed precision is chosen as the result.

### `--diagnostic-format=[default|noColor|json]` <sup>[frontend]</sup>

> **Note:**
>
> The Windows version does not support outputting error messages with color rendering.

Specifies the output format of error messages. The default is `default`.

- `default`: Output error messages in the default format (with color).
- `noColor`: Output error messages in the default format (without color).
- `json`: Output error messages in JSON format.

### `--verbose`, `-V` <sup>[frontend]</sup>

When using this option,the compiler prints compiler version information, toolchain dependency information, and the commands executed during the compilation process.

### `--help`, `-h` <sup>[frontend]</sup>

Prints the available compilation options.

When using this option, the compiler only prints information about the compilation options and does not compile any input files.

### `--version`, `-v` <sup>[frontend]</sup>

Prints the compiler version information.

When using this option, the compiler only prints version information and does not compile any input files.

### `--save-temps <value>`

Retains the intermediate files generated during the compilation process and saves them to the `<value>` path.

The compiler retains intermediate files such as `.bc` and `.o` generated during the compilation process.

### `--import-path <value>` <sup>[frontend]</sup>

Specifies the search path for AST files of imported modules.

Assume the following directory structure exists, where the `libs/myModule` directory contains the library files of the `myModule` module and the AST export files of the `log` package:

```text
.
├── libs
|   └── myModule
|       ├── myModule.log.cjo
|       └── libmyModule.log.a
└── main.cj
```

And there is the `main.cj` file:

<!-- code_check_manual -->

```cangjie
import myModule.log.printLog

main() {
    printLog("Everything is great")
}
```

You can add `./libs` to the search path for AST files of imported modules by using `--import-path ./libs`. `cjc` uses the `./libs/myModule/myModule.log.cjo` file to perform semantic checks and compilation on the `main.cj` file.

`--import-path` provides the same functionality as the `CANGJIE_PATH` environment variable, but paths set via `--import-path` have higher priority.

### `--scan-dependency` <sup>[frontend]</sup>

The `--scan-dependency` command can be used to obtain the direct dependencies of other packages and other information for the specified package source code or a package's `cjo` file, output in JSON format.

<!-- code_check_manual -->

```cangjie
// this file is placed under directory pkgA
macro package pkgA
import pkgB.*
import std.io.*
import pkgB.subB.*
```

```shell
cjc --scan-dependency --package pkgA
```

Or

```shell
cjc --scan-dependency pkgA.cjo
```

```json
{
  "package": "pkgA",
  "isMacro": true,
  "dependencies": [
    {
      "package": "pkgB",
      "isStd": false,
      "imports": [
        {
          "file": "pkgA/pkgA.cj",
          "begin": {
            "line": 2,
            "column": 1
          },
          "end": {
            "line": 2,
            "column": 14
          }
        }
      ]
    },
    {
      "package": "pkgB.subB",
      "isStd": false,
      "imports": [
        {
          "file": "pkgA/pkgA.cj",
          "begin": {
            "line": 4,
            "column": 1
          },
          "end": {
            "line": 4,
            "column": 19
          }
        }
      ]
    },
    {
      "package": "std.io",
      "isStd": true,
      "imports": [
        {
          "file": "pkgA/pkgA.cj",
          "begin": {
            "line": 3,
            "column": 1
          },
          "end": {
            "line": 3,
            "column": 16
          }
        }
      ]
    }
  ]
}
```

### `--no-sub-pkg` <sup>[frontend]</sup>

Indicates that the currently compiled package has no sub-packages.

After enabling this option, the compiler can further reduce the code size.

### `--warn-off`, `-Woff <value>` <sup>[frontend]</sup>

Turns off all or part of the warnings generated during compilation.

`<value>` can be `all` or a predefined warning group. When the parameter is `all`, the compiler does not print any warnings generated during the compilation process. When the parameter is another predefined group, the compiler does not print warnings of that group generated during the compilation process.

When each warning is printed, there is a `#note` line indicating which group the warning belongs to and how to turn it off. You can print all available compilation option parameters via `--help` to check the specific group names.

### `--warn-on`, `-Won <value>` <sup>[frontend]</sup>

Turns on all or part of the warnings generated during compilation.

The `<value>` for `--warn-on` has the same range as that for `--warn-off`. `--warn-on` is usually used in combination with `--warn-off`. For example, you can set `-Woff all -Won <value>` to allow only warnings of group `<value>` to be printed.

**Important Note:** `--warn-on` and `--warn-off` are order-sensitive in use. For the same group, the later option overrides the earlier setting. For example, swapping the positions of the two compilation options in the previous example to `-Won <value> -Woff all` results in all warnings being turned off.

### `--error-count-limit <value>` <sup>[frontend]</sup>

Limits the maximum number of errors printed by the compiler.

The parameter `<value>` can be `all` or a non-negative integer. When the parameter is `all`, the compiler prints all errors generated during the compilation process. When the parameter is a non-negative integer `N`, the compiler prints at most `N` errors. The default value of this option is 8.

### `--output-dir <value>` <sup>[frontend]</sup>

Controls the directory where intermediate files and output files generated by the compiler are saved.

Controls the directory where intermediate files generated by the compiler are saved, such as `.cjo` files. When `--output-dir <path1>` and `--output <path2>` are both specified, intermediate files are saved to `<path1>`, and the final output is saved to `<path1>/<path2>`.

> **Note:**
>
> When this option and the `--output` option are specified together, the parameter of the `--output` option must be a relative path.

### `--static`

Links against the Cangjie library statically.

This option only takes effect when compiling executable files.

**Note:**

The `--static` option is only applicable to the Linux platform and has no effect on other platforms.

### `--static-std`

Links against the `std` module of the Cangjie library statically.

This option only takes effect when compiling dynamic link libraries or executable files.

When compiling an executable program (i.e., when `--output-type=exe` is specified), `cjc` links against the `std` module of the Cangjie library statically by default.

### <span id="--dy-std">`--dy-std`

Links against the `std` module of the Cangjie library dynamically.

This option only takes effect when compiling dynamic link libraries or executable files.

When compiling a dynamic library (i.e., when `--output-type=dylib` is specified), `cjc` links against the `std` module of the Cangjie library dynamically by default.

**Note:**

1. If the `--static-std` and `--dy-std` options are used together, only the last option takes effect.
2. When compiling an executable program that links against Cangjie dynamic libraries (i.e., products compiled via the `--output-type=dylib` option), you must explicitly specify the `--dy-std` option to link against the standard library dynamically. Otherwise, multiple copies of the standard library may appear in the program assembly, which may eventually lead to runtime issues.

### `--static-libs`

> **Note:**
>
> This option is deprecated and will be removed in a future version. Using this option has no functional effect in the current version.

### `--dy-libs`

> **Note:**
>
> This option is deprecated and will be removed in a future version. Using this option has no functional effect in the current version.

### `--stack-trace-format=[default|simple|all]`

Specifies the format for printing exception call stacks, which controls the display of stack frame information when exceptions are thrown. The default is `default`.

The formats for exception call stacks are described as follows:

- `default` format: `Function name with generic parameters omitted (filename:line number)`
- `simple` format: `filename:line number`
- `all` format: `Complete function name (filename:line number)`

### `--lto=[full|thin]`

Enables and specifies the `LTO` (Link Time Optimization) optimization compilation mode.

**Note:**

1. This feature is not supported on Windows and macOS platforms.
2. When enabling and specifying the `LTO` (Link Time Optimization) optimization compilation mode, the following optimization compilation options are not allowed to be used simultaneously: `-Os`, `-Oz`.

`LTO` optimization supports two compilation modes:

- `--lto=full`: `full LTO` merges all compilation modules together and performs optimizations globally. This method can achieve maximum optimization potential but also requires longer compilation time.
- `--lto=thin`: Compared to `full LTO`, `thin LTO` uses parallel optimization on multiple modules and supports incremental compilation during linking by default. The compilation time is shorter than that of `full LTO`. However, due to the loss of more global information, the optimization effect is not as good as that of `full LTO`.

    - Generally, the optimization effect comparison: `full LTO` **>** `thin LTO` **>** conventional static linking compilation.
    - Generally, the compilation time comparison: `full LTO` **>** `thin LTO` **>** conventional static linking compilation.

Usage scenarios for `LTO` optimization:

1. Compile an executable file using the following command:

    ```shell
    $ cjc test.cj --lto=full
    or
    $ cjc test.cj --lto=thin
    ```

2. Compile a static library (`.bc` file) required for `LTO` mode using the following command, and use this library file to participate in executable file compilation:

    ```shell
    # Generate a static library as a .bc file
    $ cjc pkg.cj --lto=full --output-type=staticlib -o libpkg.bc
    # Input the .bc file together with the source file to the Cangjie compiler to compile the executable file
    $ cjc test.cj libpkg.bc --lto=full
    ```

    > **Note:**
    >
    > When inputting a static library (`.bc` file) in `LTO` mode, you need to input the path of the file to the Cangjie compiler.

3. In `LTO` mode, when statically linking the standard library (`--static-std`), the code of the standard library also participates in `LTO` optimization and is statically linked to the executable file. When dynamically linking the standard library (`--dy-std`), the dynamic library in the standard library is still used for linking in `LTO` mode.

    ```shell
    # Static linking, the standard library code also participates in LTO optimization
    $ cjc test.cj --lto=full --static-std
    # Dynamic linking, still uses dynamic libraries for linking, and the standard library code does not participate in LTO optimization
    $ cjc test.cj --lto=full --dy-std
    ```

### `--pgo-instr-gen`

Enables instrumentation compilation to generate an executable program carrying instrumentation information.

This feature is not supported when compiling for macOS and Windows targets.

`PGO` (short for `Profile-Guided Optimization`) is a commonly used compilation optimization technique that further improves program performance by using runtime profiling information. `Instrumentation-based PGO` is a `PGO` optimization method that uses instrumentation information. It usually includes three steps:

1. The compiler instruments and compiles the source code to generate an instrumented executable program.
2. Run the instrumented executable program to generate a profile file.
3. The compiler uses the profile file to compile the source code again.

```shell
# Generate an executable program test that supports source code execution information statistics (carrying instrumentation information)
$ cjc test.cj --pgo-instr-gen -o test
# After running the executable program `test`, a `default.profraw` profile file is generated
$ ./test
```

### `--pgo-instr-use=<.profdata>`

Uses the specified `profdata` profile file to guide compilation and generate an optimized executable program.

This feature is not supported when compiling for macOS targets.

> **Note:**
>
> The `--pgo-instr-use` compilation option only supports profile files in `profdata` format. You can use the `llvm-profdata` tool to convert `profraw` profile files to `profdata` profile files.

```shell
# Convert the `profraw` file to a `profdata` file.
$ LD_LIBRARY_PATH=$CANGJIE_HOME/third_party/llvm/lib:$LD_LIBRARY_PATH $CANGJIE_HOME/third_party/llvm/bin/llvm-profdata merge default.profraw -o default.profdata
# Use the specified `default.profdata` profile file to guide compilation and generate an optimized executable program `testOptimized`
$ cjc test.cj --pgo-instr-use=default.profdata -o testOptimized
```

### `--target <value>` <sup>[frontend]</sup>

Specifies the target triple for compilation.

The parameter `<value>` is generally a string in the following format: `<arch>(-<vendor>)-<os>(-<env>)`. Among them:

- `<arch>` represents the manufature of the target platform, such as `aarch64`, `x86_64`, etc.
- `<vendor>` represents the manufacturer developing the target platform, such as apple for darwin. When there is no clear platform manufacturer or the manufacturer is not important, it is often written as `unknown` or omitted.
- `<os>` represents the operating system of the target platform, such as `Linux`, `Win32`, etc.
- `<env>` represents the ABI or standard specification of the target platform, used to distinguish different runtime environments of the same operating system in more detail, such as `gnu`, `musl`, etc. This item can also be omitted when the operating system does not need to be distinguished in more detail based on `<env>`.

Currently, the local platforms and target platforms supported by `cjc` for cross-compilation are as shown in the following table:

| Local Platform (host) | Target Platform (target) |
| --------------------- | ------------------------ |
| x86_64-linux-gnu      | x86_64-windows-gnu       |
| aarch64-linux-gnu     | x86_64-windows-gnu       |

Before using `--target` to specify a target platform for cross-compilation, please prepare the cross-compilation toolchain for the corresponding target platform and the corresponding Cangjie SDK version that can be compiled for that target platform and run on the local platform.

### `--target-cpu <value>`

> **Note:**
>
> This option is an experimental feature. Binaries generated using this feature may have potential runtime issues. Please be aware of the risks of using this option. This option must be used together with the `--experimental` option.

Specifies the CPU type of the compilation target.

When specifying the CPU type of the target, the compiler attempts to use the unique extended instruction set of that CPU type and apply optimizations applicable to that CPU type when generating binaries. Binaries generated for a specific CPU type usually lose portability and may not run on other CPUs (with the same architecture instruction set).

This option supports the following tested CPU types:

**x86-64 architecture:**

- generic

**aarch64 architecture:**

- generic
- tsv110

`generic` is a general-purpose CPU type. When `generic` is specified, the compiler generates general-purpose instructions applicable to that architecture. Thus, the generated binaries can run on various CPUs based on that architecture, regardless of the specific CPU type, provided that the operating system and the dynamic dependencies of the binaries themselves are consistent. The default value of the `--target-cpu` option is `generic`.

This option also supports the following CPU types, but these CPU types have not been tested and verified. Please be aware that binaries generated using the following CPU types may have runtime issues.

**x86-64 architecture:**

- alderlake
- amdfam10
- athlon
- athlon-4
- athlon-fx
- athlon-mp
- athlon-tbird
- athlon-xp
- athlon64
- athlon64-sse3
- atom
- barcelona
- bdver1
- bdver2
- bdver3
- bdver4
- bonnell
- broadwell
- btver1
- btver2
- c3
- c3-2
- cannonlake
- cascadelake
- cooperlake
- core-avx-i
- core-avx2
- core2
- corei7
- corei7-avx
- geode
- goldmont
- goldmont-plus
- haswell
- i386
- i486
- i586
- i686
- icelake-client
- icelake-server
- ivybridge
- k6
- k6-2
- k6-3
- k8
- k8-sse3
- knl
- knm
- lakemont
- nehalem
- nocona
- opteron
- opteron-sse3
- penryn
- pentium
- pentium-m
- pentium-mmx
- pentium2
- pentium3
- pentium3m
- pentium4
- pentium4m
- pentiumpro
- prescott
- rocketlake
- sandybridge
- sapphirerapids
- silvermont
- skx
- skylake
- skylake-avx512
- slm
- tigerlake
- tremont
- westmere
- winchip-c6
- winchip2
- x86-64
- x86-64-v2
- x86-64-v3
- x86-64-v4
- yonah
- znver1
- znver2
- znver3

**aarch64 architecture:**

- a64fx
- ampere1
- apple-a10
- apple-a11
- apple-a12
- apple-a13
- apple-a14
- apple-a7
- apple-a8
- apple-a9
- apple-latest
- apple-m1
- apple-s4
- apple-s5
- carmel
- cortex-a34
- cortex-a35
- cortex-a510
- cortex-a53
- cortex-a55
- cortex-a57
- cortex-a65
- cortex-a65ae
- cortex-a710
- cortex-a72
- cortex-a73
- cortex-a75
- cortex-a76
- cortex-a76ae
- cortex-a77
- cortex-a78
- cortex-a78c
- cortex-r82
- cortex-x1
- cortex-x1c
- cortex-x2
- cyclone
- exynos-m3
- exynos-m4
- exynos-m5
- falkor
- kryo
- neoverse-512tvb
- neoverse-e1
- neoverse-n1
- neoverse-n2
- neoverse-v1
- saphira
- thunderx
- thunderx2t99
- thunderx3t110
- thunderxt81
- thunderxt83
- thunderxt88

In addition to the above optional CPU types, this option can also use `native` as the current CPU type. The compiler attempts to identify the CPU type of the current machine and uses that CPU type as the target type to generate binaries.

### `--toolchain <value>`, `-B <value>`, `-B<value>`

Specifies the path where the binary files in the compilation toolchain are located.

These binary files include the compiler, linker, and C runtime object files provided by the toolchain (such as `crt0.o`, `crti.o`, etc.).

After preparing the compilation toolchain, you can store it in a custom path and then pass that path to the compiler via `--toolchain <value>`, allowing the compiler to call the binary files in that path for cross-compilation.

### `--sysroot <value>`

Specifies the root directory path of the compilation toolchain.

For cross-compilation toolchains with a fixed directory structure, if there is no need to specify paths to binary, dynamic library, or static library files outside this directory, you can directly pass the root directory path of the toolchain to the compiler using `--sysroot <value>`. The compiler analyzes the corresponding directory structure according to the target platform type and automatically searches for the required binary files, dynamic library files, and static library files. After using this option, there is no need to specify the `--toolchain` and `--library-path` parameters.

If cross-compiling to a platform with a `triple` of `arch-os-env` and the cross-compilation toolchain has the following directory structure:

```text
/usr/sdk/arch-os-env
├── bin
|   ├── arch-os-env-gcc (cross-compiler)
|   ├── arch-os-env-ld  (linker)
|   └── ...
├── lib
|   ├── crt1.o          (C runtime object file)
|   ├── crti.o
|   ├── crtn.o
|   ├── libc.so         (dynamic library)
|   ├── libm.so
|   └── ...
└── ...
```

For the Cangjie source file `hello.cj`, you can use the following command to cross-compile `hello.cj` to the `arch-os-env` platform:

```shell
cjc --target=arch-os-env --toolchain /usr/sdk/arch-os-env/bin --toolchain /usr/sdk/arch-os-env/lib --library-path /usr/sdk/arch-os-env/lib hello.cj -o hello
```

You can also use abbreviated parameters:

```shell
cjc --target=arch-os-env -B/usr/sdk/arch-os-env/bin -B/usr/sdk/arch-os-env/lib -L/usr/sdk/arch-os-env/lib hello.cj -o hello
```

If the directory of the toolchain conforms to the conventional directory structure, you can directly use the following command without using the `--toolchain` and `--library-path` parameters:

```shell
cjc --target=arch-os-env --sysroot /usr/sdk/arch-os-env hello.cj -o hello
```

### `--strip-all`, `-s`

When compiling an executable file or dynamic library, specifies this option to remove the symbol table from the output file.

### `--discard-eh-frame`

When compiling an executable file or dynamic library, specifies this option to remove part of the information in the eh_frame section and eh_frame_hdr section (information related to crt is not processed), reducing the size of the executable file or dynamic library, but affecting debugging information.

This feature is not supported when compiling for macOS targets.

### `--set-runtime-rpath`

Writes the absolute path of the directory where the Cangjie runtime library is located into the RPATH/RUNPATH section of the binary. After using this option, there is no need to set the Cangjie runtime library directory using LD_LIBRARY_PATH (applicable to Linux platforms) or DYLD_LIBRARY_PATH (applicable to macOS platforms) when running the Cangjie program in the build environment.

This feature is not supported when compiling for Windows targets.

### `--link-option <value>`<sup>1</sup>

Specifies linker options.

`cjc` passes the value of this option as a parameter directly to the linker. The available parameters vary depending on the (system or specified) linker. You can use `--link-option` multiple times to specify multiple linker options.

### `--link-options <value>`<sup>1</sup>

Specifies linker options.

`cjc` passes multiple parameters of this option to the linker, separated by spaces. The available parameters vary depending on the (system or specified) linker. You can use `--link-options` multiple times to specify multiple linker options.

<sup>1</sup> The superscript indicates that linker passthrough options may vary depending on the linker. For specific supported options, refer to the linker documentation.

### `--disable-reflection`

Disables the reflection feature, i.e., does not generate relevant reflection information during compilation.

> **Note:**
>
> When cross-compiling aarch64-linux-ohos target, reflection information is disabled by default, and this option has no effect.

### `--profile-compile-time` <sup>[frontend]</sup>

Outputs the data of time elapsed in each compilation phase to a file with the suffix `.time.prof`, which is saved in the directory specified by `output`. If `output` specifies a file, `.time.prof` is in the same directory as that file.

### `--profile-compile-memory` <sup>[frontend]</sup>

Outputs the memory consumption data of each compilation phase to a file with the suffix `.mem.prof`, which is saved in the directory specified by `output`. If `output` specifies a file, `.mem.prof` is in the same directory as that file.

## Unit Test Options

### `--test` <sup>[frontend]</sup>

The entry point provided by the `unittest` testing framework, automatically generated by macros. When compiling with the `--test` option, the program entry point is no longer `main` but `test_entry`. For usage of the `unittest` testing framework, refer to the *Cangjie Programming Language Standard Library API* documentation.

For the Cangjie file `a.cj` in the `pkgc` directory:

<!-- run -->

```cangjie
import std.unittest.*
import std.unittest.testmacro.*

@Test
public class TestA {
    @TestCase
    public func case1(): Unit {
        print("case1\n")
    }
}
```

You can use the following command in the `pkgc` directory:

```shell
cjc a.cj --test
```

To compile `a.cj`. Executing `main` produces the following output:

> **Note:**
>
> The execution time of test cases is not guaranteed to be the same each time.

```text
case1
--------------------------------------------------------------------------------------------------
TP: default, time elapsed: 29710 ns, Result:
    TCS: TestA, time elapsed: 26881 ns, RESULT:
    [ PASSED ] CASE: case1 (16747 ns)
Summary: TOTAL: 1
    PASSED: 1, SKIPPED: 0, ERROR: 0
    FAILED: 0
--------------------------------------------------------------------------------------------------
```

For the following directory structure:

```text
application
├── src
├── pkgc
|   ├── a1.cj
|   └── a2.cj
└── a3.cj
```

You can use the `-p` compilation option in the `application` directory to compile the entire package:

```shell
cjc --test -p pkgc
```

To compile the test cases `a1.cj` and `a2.cj` under the entire `pkgc` package.

<!-- code_check_manual -->

```cangjie
/*a1.cj*/
package pkgc

import std.unittest.*
import std.unittest.testmacro.*

@Test
public class TestA {
    @TestCase
    public func caseA(): Unit {
        print("case1\n")
    }
}
```

<!-- code_check_manual -->

```cangjie
/*a2.cj*/
package pkgc

import std.unittest.*
import std.unittest.testmacro.*

@Test
public class TestB {
    @TestCase
    public func caseB(): Unit {
        throw IndexOutOfBoundsException()
    }
}
```

Executing `main` produces the following output (**output information is for reference only**):

```text
case1
--------------------------------------------------------------------------------------------------
TP: a, time elapsed: 367800 ns, Result:
    TCS: TestA, time elapsed: 16802 ns, RESULT:
    [ PASSED ] CASE: caseA (14490 ns)
    TCS: TestB, time elapsed: 347754 ns, RESULT:
    [ ERROR  ] CASE: caseB (345453 ns)
    REASON: An exception has occurred:IndexOutOfBoundsException
        at std/core.Exception::init()(std/core/exception.cj:23)
        at std/core.IndexOutOfBoundsException::init()(std/core/index_out_of_bounds_exception.cj:9)
        at a.TestB::caseB()(/home/houle/cjtest/application/pkgc/a2.cj:7)
        at a.lambda.1()(/home/houle/cjtest/application/pkgc/a2.cj:7)
        at std/unittest.TestCases::execute()(std/unittest/test_case.cj:92)
        at std/unittest.UT::run(std/unittest::UTestRunner)(std/unittest/test_runner.cj:194)
        at std/unittest.UTestRunner::doRun()(std/unittest/test_runner.cj:78)
        at std/unittest.UT::run(std/unittest::UTestRunner)(std/unittest/test_runner.cj:200)
        at std/unittest.UTestRunner::doRun()(std/unittest/test_runner.cj:78)
        at std/unittest.UT::run(std/unittest::UTestRunner)(std/unittest/test_runner.cj:200)
        at std/unittest.UTestRunner::doRun()(std/unittest/test_runner.cj:75)
        at std/unittest.entryMain(std/unittest::TestPackage)(std/unittest/entry_main.cj:11)
Summary: TOTAL: 2
    PASSED: 1, SKIPPED: 0, ERROR: 1
    FAILED: 0
--------------------------------------------------------------------------------------------------
```

### `--test-only` <sup>[frontend]</sup>

The `--test-only` option is used to compile only the test part of a package.

If this option is enabled, the compiler compiles only the test files in the package (ending with `_test.cj`).

> **Note:**
>
> When using this option, you should compile the same package in normal mode separately, then add dependencies via the `-L`/`-l` link options, or add the dependent `.bc` files when using the `LTO` option. Otherwise, the compiler will report an error of missing dependent symbols.

Example:

<!-- code_check_manual -->

```cangjie
/*main.cj*/
package my_pkg

func concatM(s1: String, s2: String): String {
    return s1 + s2
}

main(): Int64 {
    println(concatM("a", "b"))
    0
}
```

<!-- code_check_manual -->

```cangjie
/*main_test.cj*/
package my_pkg

@Test
class Tests {
    @TestCase
    public func case1(): Unit {
        @Expect("ac", concatM("a", "c"))
    }
}
```

The commands are as follows:

```shell
# Compile the production part of the package first, only the `main.cj` file is compiled here
cjc -p my_pkg --output-type=staticlib -o=output/libmain.a
# Compile the test part of the package, Only the `main_test.cj` file is compiled here
cjc -p my_pkg --test-only -L output -lmain --import-path output
```

### `--mock <on|off|runtime-error>` <sup>[frontend]</sup>

If `on` is passed, the package enables mock compilation. This option allows mocking classes in the package in test cases. `off` is an explicit way to disable mock.

> **Note:**
>
> In test mode (when `--test` is enabled), mock support for this package is automatically enabled, and there is no need to explicitly pass the `--mock` option.

`runtime-error` is only available in test mode (when `--test` is enabled). It allows compiling packages with mock code but does not perform any mock-related processing in the compiler (these processes may cause some overhead and affect the runtime performance of tests). This may be useful for benchmarking test cases with mock code. When using this compilation option, avoid compiling and running tests with mock code, otherwise, runtime exceptions will be thrown.

## Macro Options

`cjc` supports the following macro options. For more information about macros, refer to the ["Introduction to Macros"](../Macro/macro_introduction.md) chapter.

### `--compile-macro` <sup>[frontend]</sup>

Compiles macro definition files and generates default macro definition dynamic library files.

### `--debug-macro` <sup>[frontend]</sup>

Generates Cangjie code files after macro expansion. This option can be used to debug macro expansion functionality.

### `--parallel-macro-expansion` <sup>[frontend]</sup>

Enables parallel macro expansion. This option can be used to shorten the macro expansion compilation time.

## Conditional Compilation Options

`cjc` supports the following conditional compilation options. For more information about conditional compilation, refer to ["Conditional Compilation"](../compile_and_build/conditional_compilation.md).

### `--cfg <value>` <sup>[frontend]</sup>

Specifies custom compilation conditions.

## Parallel Compilation Options

`cjc` supports the following parallel compilation options to achieve higher compilation efficiency.

### `--jobs <value>`, `-j <value>` <sup>[frontend]</sup>

Sets the maximum number of parallel tasks allowed during parallel compilation. `<value>` must be a reasonable non-negative integer. When `<value>` is greater than the maximum parallel capability supported by the hardware, the compiler performs parallel compilation with the maximum parallel capability supported by the hardware.

If this compilation option is not set, the compiler automatically calculates the maximum number of parallel tasks based on hardware capabilities.

> **Note:**
>
> `--jobs 1` means compiling in serialised mode.

### `--aggressive-parallel-compile`, `--apc`, `--aggressive-parallel-compile=<value>`, `--apc=<value>` <sup>[frontend]</sup>

After enabling this option, the compiler adopts a more aggressive strategy (which may affect optimizations and thus cause a decrease in program runtime performance) to perform aggressive parallel compilation to achieve higher compilation efficiency. `<value>` is an optional parameter indicating the maximum number of parallel tasks allowed for the aggressive parallel compilation part:

- If `<value>` is used, it must be a reasonable non-negative integer. When `<value>` is greater than the maximum parallel capability supported by the hardware, the compiler automatically calculates the maximum number of parallel tasks based on hardware capabilities. It is recommended to set `<value>` to a non-negative integer less than the number of physical cores of the hardware.
- If `<value>` is not used, aggressive parallel compilation is enabled by default, and the number of parallel tasks for the aggressive parallel compilation part is the same as that for `--jobs`.

In addition, if the `value` of this option is different or the switch state of this option is different when compiling the same code twice, the compiler does not guarantee the binary consistency of the products.

The rules for enabling or disabling aggressive parallel compilation are as follows:

- In the following scenarios, aggressive parallel compilation is forcibly disabled by the compiler and cannot be enabled:

    - `--fobf-string`
    - `--fobf-const`
    - `--fobf-layout`
    - `--fobf-cf-flatten`
    - `--fobf-cf-bogus`
    - `--lto`
    - `--coverage`
    - Compiling for Windows targets
    - Compiling for macOS targets

- If `--aggressive-parallel-compile=<value>` or `--apc=<value>` is used, the switch of aggressive parallel compilation is controlled by `value`:

    - `value <= 1`: Disables aggressive parallel compilation.
    - `value > 1`: Enables aggressive parallel compilation, and the number of parallel tasks for aggressive parallel compilation depends on `value`.

- If `--aggressive-parallel-compile` or `--apc` is used, aggressive parallel compilation is enabled by default, and the number of parallel tasks for aggressive parallel compilation is the same as that for `--jobs`.

- If this compilation option is not set, the compiler enables or disables aggressive parallel compilation by default according to the scenario:

    - `-O0` or `-g`: Aggressive parallel compilation is enabled by default by the compiler, and the number of parallel tasks for aggressive parallel compilation is the same as that for `--jobs`. Aggressive parallel compilation can be disabled by using `--aggressive-parallel-compile=<value>` or `--apc=<value>` with `value <= 1`.
    - Non `-O0` and non `-g`: Aggressive parallel compilation is disabled by default by the compiler. Aggressive parallel compilation can be enabled by using `--aggressive-parallel-compile=<value>` or `--apc=<value>` with `value > 1`.

## Optimization Options

### `--fchir-constant-propagation` <sup>[frontend]</sup>

Enables CHIR constant propagation optimization.

### `--fno-chir-constant-propagation` <sup>[frontend]</sup>

Disables CHIR constant propagation optimization.

### `--fchir-function-inlining` <sup>[frontend]</sup>

Enables CHIR function inlining optimization.

### `--fno-chir-function-inlining` <sup>[frontend]</sup>

Disables CHIR function inlining optimization.

### `--fchir-devirtualization` <sup>[frontend]</sup>

Enables CHIR devirtualization optimization.

### `--fno-chir-devirtualization` <sup>[frontend]</sup>

Disables CHIR devirtualization optimization.

### `--fast-math` <sup>[frontend]</sup>

After enabling this option, the compiler makes some aggressive assumptions about floating-point numbers that may lose precision to optimize floating-point operations.

### `-O<N>` <sup>[frontend]</sup>

Uses the code optimization level specified by the parameter.

A higher optimization level means the compiler performs more code optimizations to generate more efficient programs, but may also require longer compilation time.

`cjc` uses O0 level code optimization by default. Currently, `cjc` supports the following optimization levels: O0, O1, O2, Os, Oz.

When the optimization level is 2, `cjc` also enables the following options in addition to performing corresponding optimizations:

- `--fchir-constant-propagation`
- `--fchir-function-inlining`
- `--fchir-devirtualization`

When the optimization level is `s`, `cjc` optimizes for code size in addition to performing O2 level optimizations.

When the optimization level is `z`, `cjc` further reduces the code size in addition to performing Os level optimizations.

> **Note:**
>
> When the optimization level is `s` or `z`, the link-time optimization compilation option `--lto=[full|thin]` is not allowed to be used simultaneously.

### `-O` <sup>[frontend]</sup>

Uses O1 level code optimization, equivalent to `-O1`.

## Code Obfuscation Options

`cjc` supports code obfuscation functionality to provide additional security protection for code, which is disabled by default.

`cjc` supports the following code obfuscation options:

### `--fobf-string`

Enables string obfuscation.

Obfuscates string constants appearing in the code, so that attackers cannot directly read the original identifiers.

### `--fno-obf-string`

Disables string obfuscation.

### `--fobf-const`

Enables constant obfuscation.

Obfuscates numerical constants used in the code, replacing numerical operation instructions with equivalent and more complex sequences of numerical operation instructions.

### `--fno-obf-const`

Disables constant obfuscation.

### `--fobf-layout`

Enables layout obfuscation.

The layout obfuscation function obfuscates symbols (including function names and global variable names), path names, code line numbers, and function arrangement order in the code. After using this compilation option, `cjc` generates a symbol mapping output file `*.obf.map` in the current directory. If the `--obf-sym-output-mapping` option is configured, the parameter value of `--obf-sym-output-mapping` is used as the name of the symbol mapping output file generated by `cjc`. The symbol mapping output file contains the mapping relationship between symbols before and after obfuscation. The symbol mapping output file can be used to deobfuscate obfuscated symbols.

> **Note:**
>
> The layout obfuscation function conflicts with the parallel compilation function. Do not enable them simultaneously. If enabled simultaneously with parallel compilation, parallel compilation will be invalidated.

### `--fno-obf-layout`

Disables layout obfuscation.

### `--obf-sym-prefix <string>`

Specifies the prefix string added when obfuscating symbols by the layout obfuscation function.

After setting this option, all obfuscated symbols are prefixed with this string. Symbol conflicts may occur when compiling and obfuscating multiple Cangjie packages. This option can be used to specify different prefixes for different packages to avoid symbol conflicts.

### `--obf-sym-output-mapping <file>`

Specifies the symbol mapping output file for layout obfuscation.

The symbol mapping output file records the original name of the symbol, the obfuscated name, and the path of the file to which it belongs. The symbol mapping output file can be used to deobfuscate obfuscated symbols.

### `--obf-sym-input-mapping <file,...>`

Specifies the symbol mapping input file for layout obfuscation.

The layout obfuscation function obfuscates symbols using the mapping relationships in these files. Therefore, when compiling Cangjie packages with calling relationships, please use the symbol mapping output file of the called package as the parameter of the `--obf-sym-input-mapping` option during the obfuscation of the calling package to ensure that the same symbol is obfuscated consistently in both the calling package and the called package.

### `--obf-apply-mapping-file <file>`

Provides a custom symbol mapping file for layout obfuscation. The layout obfuscation function obfuscates symbols according to the mapping relationships in the file.

The file format is as follows:

```text
<original_symbol_name> <new_symbol_name>
```

Among them, `original_symbol_name` is the name before obfuscation, and `new_symbol_name` is the name after obfuscation. `original_symbol_name` consists of multiple `fields`. A `field` represents a field name, which can be a module name, package name, class name, struct name, enum name, function name, or variable name. `fields` are separated by the separator `'.'`. If a `field` is a function name, the parameter types of the function need to be modified with parentheses `'()'` and appended to the function name. For functions without parameters, the content inside the parentheses is empty. If a `field` has generic parameters, the specific generic parameters need to be appended to the `field` with parentheses `'<>'`.

The layout obfuscation function replaces `original_symbol_name` with `new_symbol_name` in the Cangjie application. For symbols not in this file, the layout obfuscation function normally replaces them with random names. If the mapping relationships specified in this file conflict with those in `--obf-sym-input-mapping`, the compiler throws an exception and stops compiling.

### `--fobf-export-symbols`

Allows the layout obfuscation algorithm to obfuscate exported symbols. This option is enabled by default when the layout obfuscation algorithm is enabled.

After enabling this option, the layout obfuscation algorithm obfuscates exported symbols.

### `--fno-obf-export-symbols`

Prohibits the layout obfuscation function from obfuscating exported symbols.

### `--fobf-source-path`

Allows the layout obfuscation function to obfuscate path information in symbols. This option is enabled by default when the layout obfuscation function is enabled.

After enabling this option, the layout obfuscation function obfuscates the path information in the exception stack information, replacing the path name with the string `"SOURCE"`.

### `--fno-obf-source-path`

Prohibits the layout obfuscation function from obfuscating path information in stack information.

### `--fobf-line-number`

Allows the layout obfuscation function to obfuscate line number information in stack information.

After enabling this option, the layout obfuscation function obfuscates the line number information in the exception stack information, replacing the line number with `0`.

### `--fno-obf-line-number`

Prohibits the layout obfuscation function from obfuscating line number information in stack information.

### `--fobf-cf-flatten`

Enables control flow flattening obfuscation.

Obfuscates the existing control flow in the code, making its transfer logic more complex.

### `--fno-obf-cf-flatten`

Disables control flow flattening obfuscation.

### `--fobf-cf-bogus`

Enables bogus control flow obfuscation.

Inserts bogus control flow into the code to make the code logic more complex.

### `--fno-obf-cf-bogus`

Disables bogus control flow obfuscation.

### `--fobf-all`

Enables all obfuscation functions.

Specifying this option is equivalent to specifying the following options simultaneously:

- `--fobf-string`
- `--fobf-const`
- `--fobf-layout`
- `--fobf-cf-flatten`
- `--fobf-cf-bogus`

### `--obf-config <file>`

Specifies the path of the code obfuscation configuration file.

In the configuration file, you can prohibit the obfuscation tool from obfuscating certain functions or symbols.

The specific format of the configuration file is as follows:

```text
obf_func1 name1
obf_func2 name2
...
```

The first parameter `obf_func` is a specific obfuscation function:

- `obf-cf-bogus`: Bogus control flow obfuscation
- `obf-cf-flatten`: Control flow flattening obfuscation
- `obf-const`: Constant obfuscation
- `obf-layout`: Layout obfuscation

The second parameter `name` is the object to be retained, consisting of multiple `fields`. A `field` represents a field name, which can be a package name, class name, struct name, enum name, function name, or variable name.

`fields` are separated by the separator `'.'`. If a `field` is a function name, the parameter types of the function need to be modified with parentheses `'()'` and appended to the function name. For functions without parameters, the content inside the parentheses is empty.

For example, assume the following code exists in the package `packA`:

<!-- compile -->

```cangjie
package packA
class MyClassA {
    func funcA(a: String, b: Int64): String {
        return a
    }
}
```

To prohibit the control flow flattening function from obfuscating `funcA`, the user can write the following rule:

```text
obf-cf-flatten packA.MyClassA.funcA(std.core.String, Int64)
```

Users can also use wildcards to write more flexible rules to retain multiple objects with one rule. Currently supported wildcards include the following three types:

# Obfuscation Function Wildcards
| Obfuscation Function Wildcards | Description |
| :------------------------------ | :---------- |
| `?`                             | Matches a single character in the name |
| `*`                             | Matches any number of characters in the name |

# Field Name Wildcards
| Field Name Wildcards | Description |
| :-------------------- | :---------- |
| `?`                   | Matches a single character that is not the separator `'.'` in the field name |
| `*`                   | Matches any number of characters in the field name that do not include the separator `'.'` and parameters |
| `**`                  | Matches any number of characters in the field name, including the separator `'.'` between fields and parameters. `'**'` only takes effect when used as a single `field`; otherwise, it is treated as `'*'` |

# Function Parameter Type Wildcards
| Parameter Type Wildcards | Description |
| :------------------------ | :---------- |
| `...`                     | Matches any number of parameters |
| `***`                     | Matches one parameter of any type |

> **Note:**
>
> Parameter types are also composed of field names, so field name wildcards can also be used to match a single parameter type.

The following are examples of wildcard usage:

Example 1:

```text
obf-cf-flatten pro?.myfunc()
```

This rule prohibits the `obf-cf-flatten` function from obfuscating the function `pro?.myfunc()`. `pro?.myfunc()` can match `pro0.myfunc()` but cannot match `pro00.myfunc()`.

Example 2:

```text
* pro0.**
```

This rule prohibits any obfuscation function from obfuscating any functions and variables under the package `pro0`.

Example 3:

```text
* pro*.myfunc(...)
```

This rule prohibits any obfuscation function from obfuscating the function `pro*.myfunc(...)`. `pro*.myfunc(...)` can match the `myfunc` function in any single-layer package starting with `pro` and with any parameters.

To match multi-layer package names, such as `pro0.mypack.myfunc()`, use `pro*.**.myfunc(...)`. Note that `'**'` only takes effect when used as a field name. Therefore, `pro**.myfunc(...)` is equivalent to `pro*.myfunc(...)` and cannot match multi-layer package names. To match all `myfunc` functions (including `myfunc` functions in classes) under all packages starting with `pro`, use `pro*.**.myfunc(...)`.

Example 4:

```text
obf-cf-* pro0.MyClassA.myfunc(**.MyClassB, ***, ...)
```

This rule prohibits the `obf-cf-*` function from obfuscating the function `pro0.MyClassA.myfunc(**.MyClassB, ***, ...)`. Among them, `obf-cf-*` matches two obfuscation functions: `obf-cf-bogus` and `obf-cf-flatten`. `pro0.MyClassA.myfunc(**.MyClassB, ***, ...)` matches the function `pro0.MyClassA.myfunc`, and the first parameter of the function can be the `MyClassB` type under any package, the second parameter can be any type, and zero or more arbitrary parameters can be followed.

### `--obf-level <value>`

Specifies the strength level of the obfuscation function.

The strength level can be specified as 1-9. The default strength level is 5. A higher level number indicates higher strength. This option affects the size of the output file and execution overhead.

### `--obf-seed <value>`

Specifies the random number seed for the obfuscation algorithm.

By specifying the random number seed of the obfuscation algorithm, the same Cangjie code can have different obfuscation results in different builds. By default, for the same Cangjie code, the same obfuscation result is obtained after each obfuscation.

## Security Compilation Options

`cjc` generates position-independent code by default and generates position-independent executable files by default when compiling executable files.

When building a Release version, it is recommended to enable/disable compilation options according to the following rules to improve security.

### Enable `--trimpath <value>` <sup>[frontend]</sup>

Removes the specified absolute path prefix from debugging and exception information. Using this option can prevent build path information from being written into the binary output.

After using this option, the source code path information in the binary is usually no longer complete, which may affect the debugging experience. It is recommended to disable this option when building a debugging version.

### Enable `--strip-all`, `-s`

Removes the symbol table from the binary. Using this option can delete symbol-related information that is not needed at runtime.

After using this option, the binary cannot be debugged. Please disable this option when building a debugging version.

### Disable `--set-runtime-rpath`

If the executable program is to be distributed to different environments for running, or other ordinary users have write permissions to the directory of the currently used Cangjie runtime library, using this option may cause security risks. Therefore, it is recommended to disable this option.

This option does not take effect when compiling for Windows targets.

### Enable `--link-options "-z noexecstack"`<sup>1</sup>

Sets the thread stack to be non-executable.

Only available when compiling for Linux targets.

### Enable `--link-options "-z relro"`<sup>1</sup>

Sets the GOT table relocation to be read-only.

Only available when compiling for Linux targets.

### Enable `--link-options "-z now"`<sup>1</sup>

Sets immediate binding.

Only available when compiling for Linux targets.

## Code Coverage Instrumentation Options

> **Note:**
>
> The Windows and macOS versions do not currently support code coverage instrumentation options.

Cangjie supports code coverage instrumentation (SanitizerCoverage, hereinafter referred to as SanCov), providing an interface consistent with LLVM's SanitizerCoverage. The compiler inserts coverage feedback functions at the function level or BasicBlock level. Users only need to implement the agreed callback functions to perceive the program's runtime status.

The SanCov functionality provided by Cangjie is on a per-package basis, meaning that an entire package is either fully instrumented or not instrumented at all.

### `--sanitizer-coverage-level=0/1/2`

Instrumentation level:

- 0: No instrumentation.
- 1: Function-level instrumentation, inserting callback functions only at function entrances.
- 2: BasicBlock-level instrumentation, inserting callback functions at each BasicBlock.

If not specified, the default value is 2.

This compilation option only affects the instrumentation level of `--sanitizer-coverage-trace-pc-guard`, `--sanitizer-coverage-inline-8bit-counters`, and `--sanitizer-coverage-inline-bool-flag`.

### `--sanitizer-coverage-trace-pc-guard`

Enabling this option inserts a function call `__sanitizer_cov_trace_pc_guard(uint32_t *guard_variable)` at each Edge, affected by `sanitizer-coverage-level`.

**Notably**, this feature differs from gcc/llvm implementations: it does not insert `void __sanitizer_cov_trace_pc_guard_init(uint32_t *start, uint32_t *stop)` in the constructor. Instead, it inserts a function call `uint32_t *__cj_sancov_pc_guard_ctor(uint64_t edgeCount)` during package initialization.

The `__cj_sancov_pc_guard_ctor` callback function needs to be implemented by the developer. The package with SanCov enabled calls this callback function as early as possible. The input parameter is the number of Edges of the Package, and the return value is usually a memory area created by `calloc`.

If you need to call `__sanitizer_cov_trace_pc_guard_init`, it is recommended to call it in `__cj_sancov_pc_guard_ctor`, using the dynamically created buffer to calculate the input parameters and return value of the function.

A standard reference implementation of `__cj_sancov_pc_guard_ctor` is as follows:

```cpp
uint32_t *__cj_sancov_pc_guard_ctor(uint64_t edgeCount) {
    uint32_t *p = (uint32_t *) calloc(edgeCount, sizeof(uint32_t));
    __sanitizer_cov_trace_pc_guard_init(p, p + edgeCount);
    return p;
}
```

### `--sanitizer-coverage-inline-8bit-counters`

After enabling this option, an accumulator is inserted at each Edge. Each time it is passed through, the accumulator is incremented by 1, affected by `sanitizer-coverage-level`.

**Note:** There is an inconsistency between this feature and the gcc/llvm implementation: `void __sanitizer_cov_8bit_counters_init(char *start, char *stop)` is not inserted in the constructor. Instead, the function call `uint8_t *__cj_sancov_8bit_counters_ctor(uint64_t edgeCount)` is inserted during package initialization.

The `__cj_sancov_pc_guard_ctor` callback function needs to be implemented by the developer. The package with SanCov enabled calls this callback function as early as possible. The input parameter is the number of Edges of the Package, and the return value is usually a memory area created by `calloc`.

If you need to call `__sanitizer_cov_8bit_counters_init`, it is recommended to call it in `__cj_sancov_8bit_counters_ctor`, using the dynamically created buffer to calculate the input parameters and return value of the function.

A standard reference implementation of `__cj_sancov_8bit_counters_ctor` is as follows:

```cpp
uint8_t *__cj_sancov_8bit_counters_ctor(uint64_t edgeCount) {
    uint8_t *p = (uint8_t *) calloc(edgeCount, sizeof(uint8_t));
    __sanitizer_cov_8bit_counters_init(p, p + edgeCount);
    return p;
}
```

### `--sanitizer-coverage-inline-bool-flag`

After enabling this option, a boolean value is inserted at each Edge. The boolean value corresponding to the passed Edge is set to True, affected by `sanitizer-coverage-level`.

**Note:** There is an inconsistency between this feature and the gcc/llvm implementation: `void __sanitizer_cov_bool_flag_init(bool *start, bool *stop)` is not inserted in the constructor. Instead, the function call `bool *__cj_sancov_bool_flag_ctor(uint64_t edgeCount)` is inserted during package initialization.

The `__cj_sancov_bool_flag_ctor` callback function needs to be implemented by the developer. The package with SanCov enabled calls this callback function as early as possible. The input parameter is the number of Edges of the Package, and the return value is usually a memory area created by `calloc`.

If you need to call it within `__sanitizer_cov_bool_flag_init`, it is recommended to invoke it within `__cj_sancov_bool_flag_ctor`, using a dynamically created buffer to calculate the function's input parameters and return value.

A standard reference implementation of `__cj_sancov_bool_flag_ctor` is as follows:

```cpp
bool *__cj_sancov_bool_flag_ctor(uint64_t edgeCount) {
    bool *p = (bool *) calloc(edgeCount, sizeof(bool));
    __sanitizer_cov_bool_flag_init(p, p + edgeCount);
    return p;
}
```

### `--sanitizer-coverage-pc-table`

This compilation option provides the correspondence between instrumentation points and source code, currently only supporting function-level precision. It must be used together with at least one of `--sanitizer-coverage-trace-pc-guard`, `--sanitizer-coverage-inline-8bit-counters`, or `--sanitizer-coverage-inline-bool-flag` (multiple can be enabled simultaneously).

**Notable Inconsistency**: Unlike gcc/llvm implementations, this feature does **not** insert `void __sanitizer_cov_pcs_init(const uintptr_t *pcs_beg, const uintptr_t *pcs_end);` in constructors. Instead, it inserts the following function call during package initialization:
```cangjie
void __cj_sancov_pcs_init(int8_t *packageName, uint64_t n, int8_t **funcNameTable, int8_t **fileNameTable, uint64_t *lineNumberTable)
```
The meanings of each parameter are as follows:
- `int8_t *packageName`: A string representing the package name (C-style int8 arrays are used as input parameters to represent strings in instrumentation; the same applies below).
- `uint64_t n`: The total number of instrumented functions.
- `int8_t **funcNameTable`: An array of strings of length `n`, where `funcNameTable[i]` is the function name corresponding to the `i`-th instrumentation point.
- `int8_t **fileNameTable`: An array of strings of length `n`, where `fileNameTable[i]` is the file name corresponding to the `i`-th instrumentation point.
- `uint64_t *lineNumberTable`: An array of `uint64` of length `n`, where `lineNumberTable[i]` is the line number corresponding to the `i`-th instrumentation point.

If you need to call `__sanitizer_cov_pcs_init`, you must manually convert the Cangjie pc-table to a C-language pc-table.

### `--sanitizer-coverage-stack-depth`

When this compilation option is enabled, since Cangjie cannot obtain the value of the SP (Stack Pointer), it can only insert a call to `__updateSancovStackDepth` at each function entry. Implementing this function on the C side allows you to obtain the SP pointer.

A standard implementation of `__updateSancovStackDepth` is as follows:

```cpp
thread_local void* __sancov_lowest_stack;

void __updateSancovStackDepth()
{
    register void* sp = __builtin_frame_address(0);
    if (sp < __sancov_lowest_stack) {
        __sancov_lowest_stack = sp;
    }
}
```

### `--sanitizer-coverage-trace-compares`

When this option is enabled, function callbacks are inserted before all `compare` and `match` instruction calls. The specific list is as follows, with functionality consistent with LLVM-based APIs. Refer to [Tracing data flow](https://clang.llvm.org/docs/SanitizerCoverage.html#tracing-data-flow).

```cpp
void __sanitizer_cov_trace_cmp1(uint8_t Arg1, uint8_t Arg2);
void __sanitizer_cov_trace_const_cmp1(uint8_t Arg1, uint8_t Arg2);
void __sanitizer_cov_trace_cmp2(uint16_t Arg1, uint16_t Arg2);
void __sanitizer_cov_trace_const_cmp2(uint16_t Arg1, uint16_t Arg2);
void __sanitizer_cov_trace_cmp4(uint32_t Arg1, uint32_t Arg2);
void __sanitizer_cov_trace_const_cmp4(uint32_t Arg1, uint32_t Arg2);
void __sanitizer_cov_trace_cmp8(uint64_t Arg1, uint64_t Arg2);
void __sanitizer_cov_trace_const_cmp8(uint64_t Arg1, uint64_t Arg2);
void __sanitizer_cov_trace_switch(uint64_t Val, uint64_t *Cases);
```

### `--sanitizer-coverage-trace-memcmp`

This compilation option is used to feed back prefix comparison information in comparisons of `String`, `Array`, and other types. When enabled, function callbacks are inserted before the comparison functions of `String` and `Array`. Specifically, the corresponding stub functions are inserted for each of the following `String` and `Array` APIs:

- String==: __sanitizer_weak_hook_memcmp
- String.startsWith: __sanitizer_weak_hook_memcmp
- String.endsWith: __sanitizer_weak_hook_memcmp
- String.indexOf: __sanitizer_weak_hook_strstr
- String.replace: __sanitizer_weak_hook_strstr
- String.contains: __sanitizer_weak_hook_strstr
- CString==: __sanitizer_weak_hook_strcmp
- CString.startswith: __sanitizer_weak_hook_memcmp
- CString.endswith: __sanitizer_weak_hook_strncmp
- CString.compare: __sanitizer_weak_hook_strcmp
- CString.equalsLower: __sanitizer_weak_hook_strcasecmp
- Array==: __sanitizer_weak_hook_memcmp
- ArrayList==: __sanitizer_weak_hook_memcmp

## Experimental Feature Options

### `--enable-eh` <sup>[frontend]</sup>

When this option is enabled, Cangjie will support **Effect Handlers**—an advanced control flow mechanism for implementing modular, recoverable side effect handling.

Effect Handlers allow programmers to decouple side effect operations from their handling logic, enabling clearer, more composable code. This mechanism elevates abstraction levels and is particularly suitable for operations like logging, input/output, and state changes, preventing the main flow from being cluttered with side effect logic.

The working mechanism of effects is similar to exception handling but does not use `throw` or `catch`. Instead, effects are executed via `perform` and caught/handled via `handle`. Each effect must be defined by inheriting the `stdx.effect.Command` class.

Unlike traditional exception mechanisms, Effect Handlers can choose to **resume execution** after handling an effect—injecting a value into the original call site and continuing execution. This "resumption" capability enables fine-grained control over the control flow, making it ideal for scenarios requiring high control (e.g., simulators, interpreters, or cooperative multitasking systems).

Example:

<!-- verify -->
<!-- cfg="--enable-eh --experimental" -->

```cangjie
import stdx.effect.Command

// Define a Command named GetNumber
class GetNumber <: Command<Int64> {}

main() {
    try {
        println("About to perform")

        // Perform the GetNumber effect
        let a = perform GetNumber()

        // Execution resumes here after the handler executes the effect
        println("It is resumed, a = ${a}")
    } handle(e: GetNumber) {
        // Handle the GetNumber effect
        println("It is performed")

        // Resume execution and inject the value 9
        resume with 9
    }
    0
}
```

In the above example:
1. A new subclass `GetNumber` of `Command` is defined.
2. In the `main` function, a `try-handle` structure is used to handle the effect.
3. In the `try` block:
   - A prompt message ("About to perform") is printed first.
   - The effect is performed via `perform GetNumber()`, and the return value of the `perform` expression is assigned to variable `a`.
   - Executing an effect jumps the control flow to the `handle` block that catches it.
4. In the `handle` block:
   - The `GetNumber` effect is caught and processed, and a message ("It is performed") is printed.
   - The `resume with 9` statement injects the constant `9` back into the original call site and resumes execution after `perform`, printing ("It is resumed, a = 9").

Output:

```text
About to perform
It is performed
It is resumed, a = 9
```

> **Note:**
>
> - Effect Handlers are currently an experimental feature and may change in future versions. Use with caution.
> - Using Effect Handlers requires importing the `stdx.effect` library.

### `--experimental` <sup>[frontend]</sup>

Enables experimental features, allowing other experimental options to be used on the command line.

> **Note:**
>
> Binaries generated with experimental features may have potential runtime issues. Please be aware of the risks when using this option.

## Compilation Plugin Option

### `--plugin <value>` <sup>[frontend]</sup>

Provides compiler plugin capabilities but is **experimental** and intended only for internal validation. Custom plugins by developers are not supported (may cause errors).

## Other Features

### Colored Compiler Error Messages

For the Windows version of the Cangjie compiler, colored error messages are only displayed when running on Windows 10 version 1511 (Build 10586) or later. No colors are displayed on older Windows versions.

### Setting `build-id`

Linker options can be passed through using `--link-options "--build-id=<arg>"`<sup>1</sup> to set a `build-id`.

This feature is not supported when compiling for Windows targets.

### Setting `rpath`

Linker options can be passed through using `--link-options "-rpath=<arg>"`<sup>1</sup> to set an `rpath`.

This feature is not supported when compiling for Windows targets.

### Incremental Compilation

Incremental compilation is enabled via `--incremental-compile`<sup>[frontend]</sup>. After enabling, `cjc` speeds up the current compilation by leveraging cached files from the previous build.

> **Note:**
>
> - This is an experimental feature. Binaries generated with it may have potential runtime issues. Use with caution.
> - Must be used together with the `--experimental` option.
> - When enabled, incremental compilation caches and logs are saved to the `.cached` directory under the output path.

### Outputting CHIR

The serialized output of the CHIR (Cangjie Intermediate Representation) compilation phase is specified via `--emit-chir=[raw|opt]`<sup>[frontend]</sup>:
- `raw`: Outputs unoptimized CHIR (before compiler optimizations).
- `opt`: Outputs optimized CHIR (after compiler optimizations).
- Using `--emit-chir` without arguments defaults to `opt`.

### `--no-prelude` <sup>[frontend]</sup>

Disables the automatic import of the standard library's `core` package.

> **Note:**
>
> This option is used only to compile the Cangjie standard library core package. It is not intended for any other purpose.

## Environment Variables Used by `cjc`

Here are some environment variables that the Cangjie compiler may use during code compilation.

### `TMPDIR` or `TMP`

The Cangjie compiler saves temporary files generated during compilation in a temporary directory:
- By default:
  - Linux/macOS: `/tmp`
  - Windows: `C:\Windows\Temp`
- Customization:
  - On Linux/macOS: Set the `TMPDIR` environment variable.
  - On Windows: Set the `TMP` environment variable.

Examples:
In Linux shell:

```shell
export TMPDIR=/home/xxxx
```

In Windows cmd:

```shell
set TMP=D:\\xxxx
```

# `cjc` Compilation Options

This chapter introduces commonly used `cjc` compilation options. If an option is also applicable to `cjc-frontend`, it will be marked with a <sup>[frontend]</sup> superscript; if the behavior differs between `cjc-frontend` and `cjc`, additional explanations will be provided.

- Options starting with two hyphens are long options, such as `--xxxx`.
  If a long option has an optional parameter, the option and parameter must be connected with an equals sign, e.g., `--xxxx=<value>`.
  If a long option has a required parameter, the option and parameter can be separated by either a space or an equals sign, e.g., `--xxxx <value>` and `--xxxx=<value>` are equivalent.

- Options starting with a single hyphen are short options, such as `-x`.
  For short options, if followed by a parameter, the option and parameter can be separated by a space or not, e.g., `-x <value>` and `-x<value>` are equivalent.

## Basic Options

### `--output-type=[exe|staticlib|dylib]` <sup>[frontend]</sup>

Specifies the type of output file. In `exe` mode, an executable file is generated; in `staticlib` mode, a static library file (`.a` file) is generated; in `dylib` mode, a dynamic library file is generated (`.so` file on Linux, `.dll` file on Windows, and `.dylib` file on macOS).

`cjc` defaults to `exe` mode.

In addition to compiling `.cj` files into executable files, they can also be compiled into static or dynamic link libraries. For example:

```shell
$ cjc tool.cj --output-type=dylib
```

This compiles `tool.cj` into a dynamic link library. On Linux, `cjc` will generate a dynamic link library file named `libtool.so`.

**Note:** If an executable program links to a Cangjie dynamic library file, both `--dy-std` and `--dy-libs` options must be specified. For details, refer to the [`--dy-std` option description](#--dy-std).

<sup>[frontend]</sup> In `cjc-frontend`, the compilation process stops at `LLVM IR`, so the output is always a `.bc` file. However, different `--output-type` values still affect the frontend compilation strategy.

### `--package`, `-p` <sup>[frontend]</sup>

Compiles a package. When using this option, a directory must be specified as input, and the source files in the directory must belong to the same package.

Assume there is a file `log/printer.cj`:

```cangjie
package log

public func printLog(message: String) {
    println("[Log]: ${message}")
}
```

And a file `main.cj`:

```cangjie
import log.*

main() {
    printLog("Everything is great")
}
```

You can compile the `log` package using:

```shell
$ cjc -p log --output-type=staticlib
```

`cjc` will generate a `liblog.a` file in the current directory.

You can use the `liblog.a` file to compile `main.cj` as follows:

```shell
$ cjc main.cj liblog.a
```

`cjc` will compile `main.cj` and `liblog.a` together into an executable file named `main`.

### `--module-name <value>` <sup>[frontend]</sup>

Specifies the name of the module to be compiled.

Assume there is a file `my_module/src/log/printer.cj`:

```cangjie
package log

public func printLog(message: String) {
    println("[Log]: ${message}")
}
```

And a file `main.cj`:

```cangjie
import my_module.log.*

main() {
    printLog("Everything is great")
}
```

You can compile the `log` package and specify its module name as `my_module` using:

```shell
$ cjc -p my_module/src/log --module-name my_module --output-type=staticlib -o my_module/liblog.a
```

`cjc` will generate a `my_module/liblog.a` file in the `my_module` directory.

You can then use the `liblog.a` file to compile `main.cj`, which imports the `log` package:

```shell
$ cjc main.cj my_module/liblog.a
```

`cjc` will compile `main.cj` and `liblog.a` together into an executable file named `main`.

### `--output <value>`, `-o <value>`, `-o<value>` <sup>[frontend]</sup>

Specifies the output file path. The compiler's output will be written to the specified file.

For example, the following command specifies the output executable file name as `a.out`:

```shell
cjc main.cj -o a.out
```

### `--library <value>`, `-l <value>`, `-l<value>`

Specifies the library file to link.

The given library file will be passed directly to the linker. This option is typically used in conjunction with `--library-path <value>`.

The filename format should be `lib[arg].[extension]`. When linking library `a`, you can use the option `-l a`. The linker will search for files like `liba.a`, `liba.so` (or `liba.dll` when targeting Windows) in the library search directories and link them as needed.

### `--library-path <value>`, `-L <value>`, `-L<value>`

Specifies the directory where the library files to be linked are located.

When using the `--library <value>` option, this option is usually needed to specify the directory containing the library files.

The path specified by `--library-path <value>` will be added to the linker's library search path. Additionally, paths specified in the `LIBRARY_PATH` environment variable will also be added to the linker's search path. Paths specified via `--library-path` take precedence over those in `LIBRARY_PATH`.

Assume there is a dynamic library file `libcProg.so` compiled from the following C source file:

```c
#include <stdio.h>

void printHello() {
    printf("Hello World\n");
}
```

And a Cangjie file `main.cj`:

```cangjie
foreign func printHello(): Unit

main(): Int64 {
  unsafe {
    printHello()
  }
  return 0
}
```

You can compile `main.cj` and link the `cProg` library using:

```shell
cjc main.cj -L . -l cProg
```

`cjc` will output an executable file named `main`.

Running `main` will produce the following output:

```shell
$ LD_LIBRARY_PATH=.:$LD_LIBRARY_PATH ./main
Hello World
```

**Note:** Since a dynamic library is used, the library directory must be added to `$LD_LIBRARY_PATH` to ensure dynamic linking during execution.

### `-g` <sup>[frontend]</sup>

Generates an executable or library file with debug information.

> **Note:**
>
> `-g` can only be used with `-O0`. Using higher optimization levels may cause debugging features to malfunction.

### `--trimpath <value>` <sup>[frontend]</sup>

Removes the specified prefix from source file path information in debug information.

When compiling Cangjie code, `cjc` saves the absolute path of source files (`.cj` files) to provide debugging and exception information at runtime.

This option removes the specified path prefix from the source file path information. The output file's source file path information will not include the user-specified prefix.

Multiple `--trimpath` options can be used to specify different prefixes. For each source file path, the compiler will remove the first matching prefix.

### `--coverage` <sup>[frontend]</sup>

Generates an executable program that supports code coverage statistics. The compiler generates a code information file with the `.gcno` suffix for each compilation unit. After execution, each compilation unit generates an execution statistics file with the `.gcda` suffix. Using these files with the `cjcov` tool can generate a code coverage report for the execution.

> **Note:**
>
> `--coverage` can only be used with `-O0`. If a higher optimization level is specified, the compiler will issue a warning and force `-O0`. `--coverage` is used to compile executables. If used to generate static or dynamic libraries, linking errors may occur when using the library.

### `--int-overflow=[throwing|wrapping|saturating]` <sup>[frontend]</sup>

Specifies the overflow strategy for fixed-precision integer operations. Defaults to `throwing`.

- `throwing`: Throws an exception on integer overflow.
- `wrapping`: Wraps around to the other end of the fixed-precision integer on overflow.
- `saturating`: Clamps to the extreme value of the fixed-precision integer on overflow.

### `--diagnostic-format=[default|noColor|json]` <sup>[frontend]</sup>

> **Note:**
>
> The Windows version does not currently support colored error message output.

Specifies the output format for error messages. Defaults to `default`.

- `default`: Error messages are output in the default format (with color).
- `noColor`: Error messages are output in the default format (without color).
- `json`: Error messages are output in JSON format.

### `--verbose`, `-V` <sup>[frontend]</sup>

`cjc` prints compiler version information, toolchain dependency details, and commands executed during compilation.

### `--help`, `-h` <sup>[frontend]</sup>

Prints available compilation options.

When this option is used, the compiler only prints compilation option information and does not compile any input files.

### `--version`, `-v` <sup>[frontend]</sup>

Prints compiler version information.

When this option is used, the compiler only prints version information and does not compile any input files.

### `--save-temps <value>`

Retains intermediate files generated during compilation and saves them to the specified `<value>` path.

The compiler retains intermediate files like `.bc` and `.o` generated during compilation.

### `--import-path <value>` <sup>[frontend]</sup>

Specifies the search path for imported module AST files.

Assume the following directory structure, where the `libs/myModule` directory contains the `myModule` module's library files and the `log` package's AST export files:

```text
.
├── libs
|   └── myModule
|       ├── log.cjo
|       └── libmyModule.a
└── main.cj
```

And the following `main.cj` file:

```cangjie
import myModule.log.printLog

main() {
    printLog("Everything is great")
}
```

You can add `./libs` to the AST file search path using `--import-path ./libs`. `cjc` will use the `./libs/myModule/log.cjo` file for semantic checking and compilation of `main.cj`.

`--import-path` provides the same functionality as the `CANGJIE_PATH` environment variable, but paths specified via `--import-path` take precedence.

### `--scan-dependency` <sup>[frontend]</sup>

The `--scan-dependency` command outputs direct dependencies and other information for a specified package's source code or `cjo` file in JSON format.

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

Or:

```shell
cjc --scan-dependency pkgA.cjo
```

```json
{
  "package": "pkgA",
  "isMacro": true,
```

```json
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

Indicates that the current compilation package has no sub-packages.

When this option is enabled, the compiler can further reduce the code size.

### `--warn-off`, `-Woff <value>` <sup>[frontend]</sup>

Suppresses all or specific categories of compilation warnings.

`<value>` can be `all` or a predefined warning group. When set to `all`, the compiler will not print any warnings generated during compilation. When set to a specific group, the compiler will suppress warnings belonging to that group.

Each warning message includes a `#note` line indicating its group and how to disable it. Use `--help` to print all available compilation options and view specific group names.

### `--warn-on`, `-Won <value>` <sup>[frontend]</sup>

Enables all or specific categories of compilation warnings.

`<value>` for `--warn-on` follows the same range as `--warn-off`. This option is typically used in combination with `--warn-off`; for example, `-Woff all -Won <value>` allows only warnings from the specified group to be printed.

**Important Note:** The order of `--warn-on` and `--warn-off` matters. For the same group, the latter option overrides the former. For instance, reversing the order in the above example to `-Won <value> -Woff all` will result in suppressing all warnings.

### `--error-count-limit <value>` <sup>[frontend]</sup>

Limits the maximum number of errors printed by the compiler.

`<value>` can be `all` or a non-negative integer. When set to `all`, the compiler prints all errors encountered during compilation. When set to a non-negative integer `N`, the compiler prints at most `N` errors. The default value for this option is 8.

### `--output-dir <value>` <sup>[frontend]</sup>

Controls the output directory for intermediate and final files generated by the compiler.

Specifies the directory for intermediate files (e.g., `.cjo` files). If both `--output-dir <path1>` and `--output <path2>` are specified, intermediate files are saved to `<path1>`, while the final output is saved to `<path1>/<path2>`.

> **Note:**
>
> When using this option with `--output`, the argument for `--output` must be a relative path.

### `--static`

Statically links the Cangjie library.

This option only takes effect when compiling executable files.

**Important Note:**

The `--static` option is only applicable on Linux platforms and has no effect on other platforms.

### `--static-std`

Statically links the std module of the Cangjie library.

This option only takes effect when compiling dynamic libraries or executable files.

When compiling executables (i.e., with `--output-type=exe`), `cjc` defaults to statically linking the std module of the Cangjie library.

### <span id="--dy-std">`--dy-std`

Dynamically links the std module of the Cangjie library.

This option only takes effect when compiling dynamic libraries or executable files.

When compiling dynamic libraries (i.e., with `--output-type=dylib`), `cjc` defaults to dynamically linking the std module of the Cangjie library.

**Important Notes:**

1. If both `--static-std` and `--dy-std` are used, only the last option takes effect.
2. `--dy-std` cannot be used with `--static-libs`; doing so will result in an error.
3. When compiling executables that link to a Cangjie dynamic library (i.e., output from `--output-type=dylib`), you must explicitly specify `--dy-std` to dynamically link the standard library. Otherwise, multiple copies of the standard library may appear in the program, potentially causing runtime issues.

### `--static-libs`

Statically links non-std and non-runtime modules of the Cangjie library.

This option only takes effect when compiling dynamic libraries or executable files. By default, `cjc` statically links non-std and non-runtime modules of the Cangjie library.

### `--dy-libs`

Dynamically links non-std modules of the Cangjie library.

This option only takes effect when compiling dynamic libraries or executable files.

**Important Notes:**

1. If both `--static-libs` and `--dy-libs` are used, only the last option takes effect.
2. `--static-std` cannot be used with `--dy-libs`; doing so will result in an error.
3. Using `--dy-std` alone implicitly enables `--dy-libs`, with a warning message.
4. Using `--dy-libs` alone implicitly enables `--dy-std`, with a warning message.

### `--stack-trace-format=[default|simple|all]`

Specifies the format for printing exception stack traces, controlling how stack frame information is displayed when exceptions are thrown. The default format is `default`.

Stack trace formats are defined as follows:

- `default`: `Function name without generic parameters (filename:line number)`
- `simple`: `filename:line number`
- `all`: `Full function name (filename:line number)`

### `--lto=[full|thin]`

Enables and specifies the `LTO` (`Link Time Optimization`) compilation mode.

**Important Notes:**

1. This feature is not supported on `Windows` or `macOS` platforms.
2. When `LTO` is enabled, the following optimization options cannot be used simultaneously: `-Os`, `-Oz`.

`LTO` supports two compilation modes:

- `--lto=full`: `Full LTO` merges all compilation modules for global optimization, offering the highest optimization potential at the cost of longer compilation times.
- `--lto=thin`: Compared to `full LTO`, `thin LTO` performs parallel optimizations across modules and supports incremental linking by default. It compiles faster than `full LTO` but may yield less optimal results due to reduced global information.

    - Typical optimization effectiveness: `full LTO` **>** `thin LTO` **>** conventional static linking.
    - Typical compilation time: `full LTO` **>** `thin LTO` **>** conventional static linking.

**Usage Scenarios for `LTO`:**

1. Compile an executable with:
    ```shell
    $ cjc test.cj --lto=full
    or
    $ cjc test.cj --lto=thin
    ```

2. Compile a static library (`.bc` file) for `LTO` mode and use it in executable compilation:
    ```shell
    # Generate a .bc static library
    $ cjc pkg.cj --lto=full --output-type=staticlib -o libpkg.bc
    # Compile the executable with the .bc file and source
    $ cjc test.cj libpkg.bc --lto=full
    ```

    > **Note:**
    >
    > In `LTO` mode, the path to the static library (`.bc` file) must be provided to the Cangjie compiler.

3. In `LTO` mode:
    - Statically linking the standard library (`--static-std` & `--static-libs`) includes standard library code in `LTO` optimization.
    - Dynamically linking the standard library (`--dy-std` & `--dy-libs`) uses the standard dynamic library without `LTO` optimization.
    ```shell
    # Static linking: standard library code participates in LTO
    $ cjc test.cj --lto=full --static-std
    # Dynamic linking: standard library is linked dynamically (no LTO)
    $ cjc test.cj --lto=full --dy-std
    ```

### `--pgo-instr-gen`

Enables instrumentation compilation, generating an executable with profiling instrumentation.

This feature is not supported for macOS or Windows targets.

`PGO` (`Profile-Guided Optimization`) is a common compilation optimization technique that uses runtime profiling data to enhance performance. `Instrumentation-based PGO` involves three steps:

1. The compiler instruments the source code to generate an instrumented executable.
2. Running the instrumented executable produces a profile file.
3. The compiler uses the profile file to recompile the source code with optimizations.

```shell
# Generate an instrumented executable 'test'
$ cjc test.cj --pgo-instr-gen -o test
# Run 'test' to generate a 'default.profraw' profile
$ ./test
```

### `--pgo-instr-use=<.profdata>`

Uses a specified `.profdata` profile to guide compilation and generate an optimized executable.

This feature is not supported for macOS targets.

> **Note:**
>
> The `--pgo-instr-use` option only supports `.profdata` profile files. Use the `llvm-profdata` tool to convert `.profraw` files to `.profdata`.

```shell
# Convert .profraw to .profdata
$ LD_LIBRARY_PATH=$CANGJIE_HOME/third_party/llvm/lib:$LD_LIBRARY_PATH $CANGJIE_HOME/third_party/llvm/bin/llvm-profdata merge default.profraw -o default.profdata
# Use the profile to generate an optimized executable 'testOptimized'
$ cjc test.cj --pgo-instr-use=default.profdata -o testOptimized
```

### `--target <value>` <sup>[frontend]</sup>

Specifies the target platform triple for compilation.

`<value>` typically follows the format: `<arch>(-<vendor>)-<os>(-<env>)`, where:
- `<arch>`: Target architecture (e.g., `aarch64`, `x86_64`).
- `<vendor>`: Platform vendor (e.g., `apple`). Often omitted or set to `unknown` if irrelevant.
- `<os>`: Target OS (e.g., `Linux`, `Win32`).
- `<env>`: ABI or runtime environment (e.g., `gnu`, `musl`). Omitted if unnecessary.

Currently supported host and target platforms for cross-compilation:

| Host Platform       | Target Platform       |
| ------------------- | -------------------- |
| x86_64-linux-gnu    | x86_64-windows-gnu   |
| aarch64-linux-gnu   | x86_64-windows-gnu   |

Before cross-compiling with `--target`, ensure the target platform's cross-compilation toolchain and a compatible Cangjie SDK version are available on the host platform.

### `--target-cpu <value>`

> **Note:**
>
> This is an experimental feature. Binaries generated with this option may have potential runtime issues. Use with caution. Requires `--experimental` to be enabled.

Specifies the target CPU type for compilation.

When specified, the compiler attempts to use CPU-specific instruction sets and optimizations. Binaries optimized for a specific CPU may lose portability and might not run on other CPUs (even with the same architecture).

Tested CPU types:

**x86-64 Architecture:**
- generic

**aarch64 Architecture:**
- generic
- tsv110

`generic` is the default CPU type, producing portable binaries that run on any CPU of the target architecture (assuming OS and dynamic dependencies match). 

The following CPU types are also supported but untested; binaries may exhibit runtime issues:

**x86-64 Architecture:**
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

**aarch64 Architecture:**  

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

In addition to the above optional CPU types, the `native` option can also be used to specify the current CPU type. The compiler will attempt to identify the host machine's CPU type and generate binaries targeting that CPU type.

### `--toolchain <value>`, `-B <value>`, `-B<value>`  

Specifies the path where binary files in the compilation toolchain are stored.  

These binary files include the compiler, linker, and C runtime object files provided by the toolchain (e.g., `crt0.o`, `crti.o`, etc.).  

After preparing the compilation toolchain, you can store it in a custom path and then pass this path to the compiler via `--toolchain <value>`. This allows the compiler to invoke the binaries under this path for cross-compilation.  

### `--sysroot <value>`  

Specifies the root directory path of the compilation toolchain.  

For cross-compilation toolchains with fixed directory structures, if there is no need to specify paths for binaries, dynamic libraries, or static library files outside this directory, you can directly use `--sysroot <value>` to pass the toolchain's root directory path to the compiler. The compiler will analyze the corresponding directory structure based on the target platform type and automatically search for the required binary files, dynamic libraries, and static library files. When using this option, there is no need to specify the `--toolchain` or `--library-path` parameters.  

For cross-compilation targeting a platform with the `triple` format `arch-os-env`, if the cross-compilation toolchain has the following directory structure:  

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

Alternatively, you can use the abbreviated parameters:  

```shell  
cjc --target=arch-os-env -B/usr/sdk/arch-os-env/bin -B/usr/sdk/arch-os-env/lib -L/usr/sdk/arch-os-env/lib hello.cj -o hello  
```  

If the toolchain's directory conforms to conventional structures, you can omit the `--toolchain` and `--library-path` parameters and directly use the following command:  

```shell  
cjc --target=arch-os-env --sysroot /usr/sdk/arch-os-env hello.cj -o hello  
```  

### `--strip-all`, `-s`  

When compiling executable files or dynamic libraries, specifying this option removes the symbol table from the output file.  

### `--discard-eh-frame`  

When compiling executable files or dynamic libraries, specifying this option removes the eh_frame section and partial information from the eh_frame_hdr section (information related to crt remains unprocessed), reducing the size of the executable or dynamic library but potentially affecting debugging information.  

This feature is currently unsupported when compiling for macOS targets.  

### `--set-runtime-rpath`  

Writes the absolute path of the Cangjie runtime library directory into the binary's RPATH/RUNPATH section. Using this option eliminates the need to set the Cangjie runtime library directory via LD_LIBRARY_PATH (for Linux platforms) or DYLD_LIBRARY_PATH (for macOS platforms) when running the Cangjie program in the build environment.  

This feature is unsupported when compiling for Windows targets.  

### `--link-options <value>`<sup>1</sup>  

Specifies linker options.  

`cjc` will transparently pass the arguments of this option to the linker. Available arguments may vary depending on the system or specified linker. Multiple linker options can be specified by using `--link-options` multiple times.  

<sup>1</sup> Superscript indicates that linker passthrough options may vary depending on the linker. Refer to the linker documentation for supported options.  

### `--disable-reflection`  

Disables the reflection option, meaning no reflection-related information is generated during compilation.  

> **Note:**  
>  
> When cross-compiling for the aarch64-linux-ohos target, reflection information is disabled by default, and this option has no effect.  

### `--profile-compile-time` <sup>[frontend]</sup>  

Prints time consumption data for each compilation phase.  

### `--profile-compile-memory` <sup>[frontend]</sup>  

Prints memory consumption data for each compilation phase.  

## Unit Test Options  

### `--test` <sup>[frontend]</sup>  

The entry point provided by the `unittest` testing framework, automatically generated by macros. When compiling with the `cjc --test` option, the program entry is no longer `main` but `test_entry`. For usage of the unittest testing framework, refer to the *Cangjie Programming Language Standard Library API* documentation.  

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

You can compile `a.cj` in the `pkgc` directory using:  

```shell  
cjc a.cj --test  
```  

Executing `main` will produce the following output:  

> **Note:**  
>  
> Execution time for test cases is not guaranteed to be consistent across runs.  

```cangjie  
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

You can compile the entire test suite in the `pkgc` directory (including `a1.cj` and `a2.cj`) from the `application` directory using the `-p` compilation option:  

```shell  
cjc pkgc --test -p  
```  

```cangjie  
/*a1.cj*/  
package a  

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

```cangjie
/*a2.cj*/
package a

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

Executing `main` will produce the following output (**output is for reference only**):

```cangjie
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

The `--test-only` option is used to compile only the test portion of a package.

When this option is enabled, the compiler will only compile test files (those ending with `_test.cj`) within the package.

> **Note:**
>
> When using this option, the same package should first be compiled in regular mode separately, then dependencies should be added via `-L`/`-l` linking options, or by adding dependent `.bc` files when using the `LTO` option. Otherwise, the compiler will report errors about missing dependency symbols.

Example:

```cangjie
/*main.cj*/
package my_pkg

func concatM(s1: String, s2: String): String {
    return s1 + s2
}

main() {
    println(concatM("a", "b"))
    0
}
```

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

The compilation commands are as follows:

```shell
# Compile the production part of the package first, only `main.cj` file would be compiled here
cjc -p my_pkg --output-type=static -o=output/libmain.a
# Compile the test part of the package, Only `main_test.cj` file would be compiled here
cjc -p my_pkg --test-only -L output -lmain
```

### `--mock <on|off|runtime-error>` <sup>[frontend]</sup>

If `on` is passed, the package will enable mock compilation, allowing classes in this package to be mocked in test cases. `off` is an explicit way to disable mocking.

> **Note:**
>
> Mock support for this package is automatically enabled in test mode (when `--test` is enabled), without needing to explicitly pass the `--mock` option.

`runtime-error` is only available in test mode (when `--test` is enabled). It allows compiling packages with mock code but performs no mock-related processing in the compiler (which may cause some overhead and affect test runtime performance). This can be useful when benchmarking test cases with mock code. When using this compilation option, avoid compiling test cases with mock code and running tests, otherwise a runtime exception will be thrown.

## Macro Options

`cjc` supports the following macro options. For more details about macros, refer to the ["Macros"](./Chapter_15_Macro.md) chapter.

### `--compile-macro` <sup>[frontend]</sup>

Compiles macro definition files, generating default macro definition dynamic library files.

### `--debug-macro` <sup>[frontend]</sup>

Generates Cangjie code files after macro expansion. This option can be used to debug macro expansion functionality.

### `--parallel-macro-expansion` <sup>[frontend]</sup>

Enables parallel macro expansion. This option can be used to reduce macro expansion compilation time.

## Conditional Compilation Options

`cjc` supports the following conditional compilation options. For more details about conditional compilation, refer to ["Conditional Compilation"](../compile_and_build/conditional_compilation.md).

### `--cfg <value>` <sup>[frontend]</sup>

Specifies custom compilation conditions.

## Parallel Compilation Options

`cjc` supports the following parallel compilation options to achieve higher compilation efficiency.

### `--jobs <value>`, `-j <value>` <sup>[frontend]</sup>

Sets the maximum allowed parallelism during parallel compilation. Here, `value` must be a reasonable non-negative integer. When `value` exceeds the hardware's maximum parallel capability, the compiler will use the hardware's maximum parallel capability for compilation.

If this compilation option is not set, the compiler will automatically calculate the maximum parallelism based on hardware capabilities.

> **Note:**
>
> `--jobs 1` indicates using purely serial compilation.

### `--aggressive-parallel-compile`, `--apc`, `--aggressive-parallel-compile=<value>`, `--apc=<value>` <sup>[frontend]</sup>

When enabled, the compiler will adopt a more aggressive strategy (which may impact optimizations and thus reduce program runtime performance) to perform aggressive parallel compilation for higher compilation efficiency. Here, `value` is an optional parameter indicating the maximum allowed parallelism for the aggressive parallel compilation portion:

- If `value` is used, it must be a reasonable non-negative integer. When `value` exceeds the hardware's maximum parallel capability, the compiler will automatically calculate the maximum parallelism based on hardware capabilities. It is recommended to set `value` to a non-negative integer less than the hardware's physical core count.
- If `value` is not used, aggressive parallel compilation is enabled by default, and the parallelism for the aggressive portion matches `--jobs`.

Additionally, if the `value` differs between two compilations of the same code, or if the option's enabled/disabled state differs, the compiler does not guarantee binary consistency between the outputs of these two compilations.

Rules for enabling/disabling aggressive parallel compilation:

- In the following scenarios, aggressive parallel compilation will be forcibly disabled by the compiler and cannot be enabled:

    - `--fobf-string`
    - `--fobf-const`
    - `--fobf-layout`
    - `--fobf-cf-flatten`
    - `--fobf-cf-bogus`
    - `--lto`
    - `--coverage`
    - Compiling for Windows targets
    - Compiling for macOS targets

- If `--aggressive-parallel-compile=<value>` or `--apc=<value>` is used, the enable/disable state is controlled by `value`:

    - `value <= 1`: Disables aggressive parallel compilation.
    - `value > 1`: Enables aggressive parallel compilation, with parallelism determined by `value`.

- If `--aggressive-parallel-compile` or `--apc` is used, aggressive parallel compilation is enabled by default, with parallelism matching `--jobs`.

- If this compilation option is not set, the compiler will default to enabling or disabling aggressive parallel compilation based on the scenario:

    - `-O0` or `-g`: Aggressive parallel compilation is enabled by default, with parallelism matching `--jobs`; can be disabled via `--aggressive-parallel-compile=<value>` or `--apc=<value>` with `value <= 1`.
    - Not `-O0` and not `-g`: Aggressive parallel compilation is disabled by default; can be enabled via `--aggressive-parallel-compile=<value>` or `--apc=<value>` with `value > 1`.

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

When enabled, the compiler will make aggressive and potentially precision-losing assumptions about floating-point operations to optimize them.

### `-O<N>` <sup>[frontend]</sup>

Uses the specified code optimization level.

Higher optimization levels cause the compiler to perform more optimizations to generate more efficient programs, though compilation time may increase.

`cjc` defaults to O0 optimization level. Currently, `cjc` supports the following optimization levels: O0, O1, O2, Os, Oz.

When the optimization level is 2, `cjc` performs corresponding optimizations and also enables the following options:

- `--fchir-constant-propagation`
- `--fchir-function-inlining`
- `--fchir-devirtualization`

When the optimization level is s, `cjc` performs O2-level optimizations and additionally optimizes for code size.

When the optimization level is z, `cjc` performs Os-level optimizations and further reduces code size.

> **Note:**
>
> When the optimization level is s or z, the link-time optimization option `--lto=[full|thin]` cannot be used simultaneously.

### `-O` <sup>[frontend]</sup>

Uses O1-level code optimization, equivalent to `-O1`.

## Code Obfuscation Options

`cjc` supports code obfuscation functionality to provide additional security protection for code, disabled by default.

`cjc` supports the following code obfuscation options:

### `--fobf-string`

Enables string obfuscation.

Obfuscates string constants in the code, preventing attackers from statically reading string data directly from the binary.

### `--fno-obf-string`

Disables string obfuscation.

### `--fobf-const`

Enables constant obfuscation.

Obfuscates numeric constants used in the code, replacing numeric operation instructions with equivalent but more complex instruction sequences.

### `--fno-obf-const`

Disables constant obfuscation.

### `--fobf-layout`

Enables layout obfuscation.

Layout obfuscation obfuscates symbols (including function names and global variable names), path names, line numbers, and function layout order in the code. When this compilation option is used, `cjc` generates a symbol mapping output file `*.obf.map` in the current directory. If `--obf-sym-output-mapping` is configured, its parameter value will be used as the filename for the symbol mapping output file. The symbol mapping output file contains the mapping between original and obfuscated symbols, which can be used to deobfuscate obfuscated symbols.

> **Note:**
>
> Layout obfuscation conflicts with parallel compilation. Do not enable both simultaneously. If both are enabled, parallel compilation will be disabled.

### `--fno-obf-layout`

Disables layout obfuscation.

### `--obf-sym-prefix <string>`

Specifies the prefix string added to symbols during layout obfuscation.

When set, all obfuscated symbols will have this prefix added. When compiling multiple obfuscated Cangjie packages, symbol conflicts may occur. Using this option to assign different prefixes to different packages can avoid symbol conflicts.

### `--obf-sym-output-mapping <file>`

Specifies the symbol mapping output file for layout obfuscation.

The symbol mapping output file records original symbol names, obfuscated names, and file paths. This file can be used to deobfuscate obfuscated symbols.

### `--obf-sym-input-mapping <file,...>`

Specifies symbol mapping input files for layout obfuscation.

The layout obfuscation functionality will use the mappings in these files to obfuscate symbols. Therefore, when compiling Cangjie packages with dependencies, use the symbol mapping output files of dependent packages as parameters for `--obf-sym-input-mapping` to ensure consistent obfuscation results for the same symbols across dependent packages.

### `--obf-apply-mapping-file <file>`

Provides a custom symbol mapping file for layout obfuscation. The layout obfuscation functionality will obfuscate symbols according to the mappings in this file.

File format is as follows:

```text
<original_symbol_name> <new_symbol_name>
```

Where `original_symbol_name` is the pre-obfuscation name, and `new_symbol_name` is the post-obfuscation name. `original_symbol_name` consists of multiple `field`s. A `field` represents a field name, which can be a module name, package name, class name, struct name, enum name, function name, or variable name. Fields are separated by the delimiter `'.'`. If a `field` is a function name, the function's parameter types must be appended in parentheses `'()'`. For parameterless functions, the parentheses remain empty. If a `field` has generic parameters, they must also be appended in angle brackets `'<>'`.

The shape obfuscation feature will replace `original_symbol_name` in the Cangjie application with `new_symbol_name`. For symbols not listed in this file, the shape obfuscation feature will use random names for replacement. If the mapping relationships specified in this file conflict with those in `--obf-sym-input-mapping`, the compiler will throw an exception and halt compilation.

### `--fobf-export-symbols`

Allows the shape obfuscation feature to obfuscate exported symbols. This option is enabled by default when shape obfuscation is active.

When enabled, the shape obfuscation feature will obfuscate exported symbols.

### `--fno-obf-export-symbols`

Disables the shape obfuscation feature from obfuscating exported symbols.

### `--fobf-source-path`

Allows the shape obfuscation feature to obfuscate path information in symbols. This option is enabled by default when shape obfuscation is active.

When enabled, the shape obfuscation feature will obfuscate path information in exception stack traces, replacing path names with the string `"SOURCE"`.

### `--fno-obf-source-path`

Disables the shape obfuscation feature from obfuscating path information in stack traces.

### `--fobf-line-number`

Allows the shape obfuscation feature to obfuscate line number information in stack traces.

When enabled, the shape obfuscation feature will obfuscate line number information in exception stack traces, replacing line numbers with `0`.

### `--fno-obf-line-number`

Disables the shape obfuscation feature from obfuscating line number information in stack traces.

### `--fobf-cf-flatten`

Enables control flow flattening obfuscation.

Obfuscates existing control flow in the code, making its transfer logic more complex.

### `--fno-obf-cf-flatten`

Disables control flow flattening obfuscation.

### `--fobf-cf-bogus`

Enables bogus control flow obfuscation.

Inserts bogus control flow into the code, making the logic more complex.

### `--fno-obf-cf-bogus`

Disables bogus control flow obfuscation.

### `--fobf-all`

Enables all obfuscation features.

Specifying this option is equivalent to specifying the following options simultaneously:

- `--fobf-string`
- `--fobf-const`
- `--fobf-layout`
- `--fobf-cf-flatten`
- `--fobf-cf-bogus`

### `--obf-config <file>`

Specifies the path to the code obfuscation configuration file.

The configuration file can instruct the obfuscation tool to exclude certain functions or symbols from obfuscation.

The configuration file format is as follows:

```text
obf_func1 name1
obf_func2 name2
...
```

The first parameter `obf_func` specifies the obfuscation feature:

- `obf-cf-bogus`: Bogus control flow obfuscation
- `obf-cf-flatten`: Control flow flattening obfuscation
- `obf-const`: Constant obfuscation
- `obf-layout`: Shape obfuscation

The second parameter `name` specifies the object to be preserved, consisting of multiple `field`s. A `field` represents a field name, which can be a package name, class name, struct name, enum name, function name, or variable name.

Fields are separated by the delimiter `'.'`. If a `field` is a function name, the function's parameter types must be appended in parentheses `'()'`. For parameterless functions, the parentheses remain empty.

For example, assume the following code exists in package `packA`:

```cangjie
package packA
class MyClassA {
    func funcA(a: String, b: Int64): String {
        return a
    }
}
```

To exclude `funcA` from control flow flattening obfuscation, the user can write the following rule:

```text
obf-cf-flatten packA.MyClassA.funcA(std.core.String, Int64)
```

Users can also use wildcards to create more flexible rules, allowing a single rule to preserve multiple objects. Currently supported wildcards include the following three types:

Obfuscation feature wildcards:

| Wildcard       | Description                  |
| :------------- | :--------------------------- |
| `?`           | Matches a single character in a name |
| `*`           | Matches any number of characters in a name |

Field name wildcards:

| Wildcard       | Description                                                                 |
| :------------- | :-------------------------------------------------------------------------- |
| `?`           | Matches a single non-delimiter `'.'` character in a field name              |
| `*`           | Matches any number of characters in a field name, excluding delimiters `'.'` and parameters |
| `**`          | Matches any number of characters in a field name, including delimiters `'.'` and parameters. `'**'` only takes effect when used as a standalone `field`; otherwise, it is treated as `'*'` |

Function parameter type wildcards:

| Wildcard       | Description                  |
| :------------- | :--------------------------- |
| `...`         | Matches any number of parameters |
| `***`         | Matches a single parameter of any type |

> **Note:**
>
> Parameter types also consist of field names, so field name wildcards can be used to match individual parameter types.

Examples of wildcard usage:

Example 1:

```text
obf-cf-flatten pro?.myfunc()
```

This rule excludes the function `pro?.myfunc()` from `obf-cf-flatten` obfuscation. `pro?.myfunc()` can match `pro0.myfunc()` but not `pro00.myfunc()`.

Example 2:

```text
* pro0.**
```

This rule excludes all functions and variables in package `pro0` from any obfuscation feature.

Example 3:

```text
* pro*.myfunc(...)
```

This rule excludes the function `pro*.myfunc(...)` from any obfuscation feature. `pro*.myfunc(...)` can match any `myfunc` function in a single-level package starting with `pro`, with any parameters.

To match multi-level package names, such as `pro0.mypack.myfunc()`, use `pro*.**.myfunc(...)`. Note that `'**'` only takes effect as a standalone field name, so `pro**.myfunc(...)` and `pro*.myfunc(...)` are equivalent and cannot match multi-level package names. To match all `myfunc` functions in any package starting with `pro` (including functions named `myfunc` in classes), use `pro*.**.myfunc(...)`.

Example 4:

```text
obf-cf-* pro0.MyClassA.myfunc(**.MyClassB, ***, ...)
```

This rule excludes the function `pro0.MyClassA.myfunc(**.MyClassB, ***, ...)` from `obf-cf-*` obfuscation. Here, `obf-cf-*` matches both `obf-cf-bogus` and `obf-cf-flatten` obfuscation features. `pro0.MyClassA.myfunc(**.MyClassB, ***, ...)` matches the function `pro0.MyClassA.myfunc`, where the first parameter can be of type `MyClassB` from any package, the second parameter can be of any type, and any number of additional parameters can follow.

### `--obf-level <value>`

Specifies the obfuscation intensity level.

Accepts values from 1 to 9. The default intensity level is 5. Higher values increase the intensity, affecting output file size and execution overhead.

### `--obf-seed <value>`

Specifies the random seed for the obfuscation algorithm.

By specifying a random seed, the same Cangjie code can produce different obfuscation results across builds. By default, the same Cangjie code produces identical obfuscation results each time.

## Secure Compilation Options

`cjc` generates position-independent code by default and produces position-independent executables when compiling executable files.

For Release builds, it is recommended to enable/disable compilation options according to the following rules to enhance security.

### Enable `--trimpath <value>` <sup>[frontend]</sup>

Removes specified absolute path prefixes from debugging and exception information. This option prevents build path information from being written into the binary.

After enabling this option, source path information in the binary is typically incomplete, which may affect debugging. It is recommended to disable this option for debug builds.

### Enable `--strip-all`, `-s`

Removes symbol tables from the binary. This option deletes symbol-related information not required at runtime.

After enabling this option, the binary cannot be debugged. Disable this option for debug builds.

### Disable `--set-runtime-rpath`

If the executable will be distributed to different environments or if other users have write permissions to the Cangjie runtime library directory currently in use, enabling this option may pose security risks. Therefore, disable this option.

This option is not applicable when compiling for Windows targets.

### Enable `--link-options "-z noexecstack"`<sup>1</sup>

Makes the thread stack non-executable.

Only available when compiling for Linux targets.

### Enable `--link-options "-z relro"`<sup>1</sup>

Makes the GOT table read-only after relocation.

Only available when compiling for Linux targets.

### Enable `--link-options "-z now"`<sup>1</sup>

Enables immediate binding.

Only available when compiling for Linux targets.

## Code Coverage Instrumentation Options

> **Note:**
>
> Windows and macOS versions currently do not support code coverage instrumentation options.

Cangjie supports code coverage instrumentation (SanitizerCoverage, hereafter SanCov), providing interfaces consistent with LLVM's SanitizerCoverage. The compiler inserts coverage feedback functions at the function or BasicBlock level, allowing users to monitor program execution by implementing predefined callback functions.

Cangjie's SanCov feature operates at the package level, meaning an entire package is either fully instrumented or not instrumented at all.

### `--sanitizer-coverage-level=0/1/2`

Instrumentation level:

- 0: No instrumentation.
- 1: Function-level instrumentation, inserting callback functions only at function entry points.
- 2: BasicBlock-level instrumentation, inserting callback functions at each BasicBlock.

If not specified, the default value is 2.

This compilation option only affects the instrumentation level of `--sanitizer-coverage-trace-pc-guard`, `--sanitizer-coverage-inline-8bit-counters`, and `--sanitizer-coverage-inline-bool-flag`.

### `--sanitizer-coverage-trace-pc-guard`

Enabling this option inserts a function call `__sanitizer_cov_trace_pc_guard(uint32_t *guard_variable)` at each Edge, influenced by `sanitizer-coverage-level`.

**Note:** This feature differs from gcc/llvm implementations in one key aspect: it does not insert `void __sanitizer_cov_trace_pc_guard_init(uint32_t *start, uint32_t *stop)` in the constructor. Instead, it inserts the function call `uint32_t *__cj_sancov_pc_guard_ctor(uint64_t edgeCount)` during package initialization.

The `__cj_sancov_pc_guard_ctor` callback function must be implemented by the developer. Packages with SanCov enabled will call this callback as early as possible, passing the number of Edges in the package as an argument. The return value is typically a memory region allocated via `calloc`.

If `__sanitizer_cov_trace_pc_guard_init` needs to be called, it is recommended to call it within `__cj_sancov_pc_guard_ctor`, using dynamically allocated buffers to compute the function's arguments and return value.

A standard implementation of `__cj_sancov_pc_guard_ctor` is as follows:

```cpp
uint32_t *__cj_sancov_pc_guard_ctor(uint64_t edgeCount) {
    uint32_t *p = (uint32_t *) calloc(edgeCount, sizeof(uint32_t));
    __sanitizer_cov_trace_pc_guard_init(p, p + edgeCount);
    return p;
}
```

### `--sanitizer-coverage-inline-8bit-counters`

Enabling this option inserts an 8-bit counter at each Edge, incrementing the counter each time the Edge is traversed, influenced by `sanitizer-coverage-level`.

**Note:** This feature differs from gcc/llvm implementations in one key aspect: it does not insert `void __sanitizer_cov_8bit_counters_init(char *start, char *stop)` in the constructor. Instead, it inserts the function call `uint8_t *__cj_sancov_8bit_counters_ctor(uint64_t edgeCount)` during package initialization.

The `__cj_sancov_pc_guard_ctor` callback function must be implemented by the developer. Packages with SanCov enabled will call this callback as early as possible, passing the number of Edges in the package as an argument. The return value is typically a memory region allocated via `calloc`.

If `__sanitizer_cov_8bit_counters_init` needs to be called, it is recommended to call it within `__cj_sancov_8bit_counters_ctor`, using dynamically allocated buffers to compute the function's arguments and return value.

A standard implementation of `__cj_sancov_8bit_counters_ctor` is as follows:

```cpp
uint8_t *__cj_sancov_8bit_counters_ctor(uint64_t edgeCount) {
    uint8_t *p = (uint8_t *) calloc(edgeCount, sizeof(uint8_t));
    __sanitizer_cov_8bit_counters_init(p, p + edgeCount);
    return p;
}
```

### `--sanitizer-coverage-inline-bool-flag`

Enabling this option inserts a boolean flag at each Edge, setting the flag to `True` when the Edge is traversed, influenced by `sanitizer-coverage-level`.

**Note:** This feature differs from gcc/llvm implementations in one key aspect: it does not insert `void __sanitizer_cov_bool_flag_init(bool *start, bool *stop)` in the constructor. Instead, it inserts the function call `bool *__cj_sancov_bool_flag_ctor(uint64_t edgeCount)` during package initialization.

The `__cj_sancov_bool_flag_ctor` callback function must be implemented by the developer. Packages with SanCov enabled will call this callback as early as possible, passing the number of Edges in the package as an argument. The return value is typically a memory region allocated via `calloc`.

If `__sanitizer_cov_bool_flag_init` needs to be called, it is recommended to call it within `__cj_sancov_bool_flag_ctor`, using dynamically allocated buffers to compute the function's arguments and return value.A standard reference implementation of `__cj_sancov_bool_flag_ctor` is as follows:

```cpp
bool *__cj_sancov_bool_flag_ctor(uint64_t edgeCount) {
    bool *p = (bool *) calloc(edgeCount, sizeof(bool));
    __sanitizer_cov_bool_flag_init(p, p + edgeCount);
    return p;
}
```

### `--sanitizer-coverage-pc-table`

This compilation option provides the mapping between instrumentation points and source code, currently offering only function-level granularity. It must be used in conjunction with at least one of the following options: `--sanitizer-coverage-trace-pc-guard`, `--sanitizer-coverage-inline-8bit-counters`, or `--sanitizer-coverage-inline-bool-flag`. Multiple options can be enabled simultaneously.

**Notably**, there is an implementation discrepancy with gcc/llvm: instead of inserting `void __sanitizer_cov_pcs_init(const uintptr_t *pcs_beg, const uintptr_t *pcs_end);` in the constructor, it inserts the function call `void __cj_sancov_pcs_init(int8_t *packageName, uint64_t n, int8_t **funcNameTable, int8_t **fileNameTable, uint64_t *lineNumberTable)` during package initialization. The parameters are defined as follows:

- `int8_t *packageName`: A string representing the package name (using C-style int8 arrays for string parameters, same below).
- `uint64_t n`: Indicates n functions are instrumented.
- `int8_t **funcNameTable`: A string array of length n, where the i-th instrumentation point corresponds to funcNameTable\[i\].
- `int8_t **fileNameTable`: A string array of length n, where the i-th instrumentation point corresponds to fileNameTable\[i\].
- `uint64_t *lineNumberTable`: A uint64 array of length n, where the i-th instrumentation point corresponds to lineNumberTable\[i\].

If `__sanitizer_cov_pcs_init` needs to be called, manual conversion from Cangjie pc-table to C language pc-table is required.

### `--sanitizer-coverage-stack-depth`

When this compilation option is enabled, since Cangjie cannot obtain the SP pointer value, it can only insert calls to `__updateSancovStackDepth` at each function entry point. Implementing this function on the C side will provide access to the SP pointer.

A standard `updateSancovStackDepth` implementation is as follows:

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

Enabling this option inserts callback functions before all compare and match instructions. The following list matches the functionality of LLVM's API. Refer to Tracing data flow.

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

This compilation option provides prefix comparison feedback for String and Array comparisons. When enabled, it inserts callback functions before comparison functions for String and Array operations. The following APIs will have corresponding instrumentation functions inserted:

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

### `--experimental` <sup>[frontend]</sup>

Enables experimental features, allowing the use of other experimental options in the command line.

> **Note:**
>
> Binaries generated with experimental features may have potential runtime issues. Use this option with caution.

## Additional Features

### Compiler Error Message Colors

For the Windows version of the Cangjie compiler, colored error messages are only displayed when running on Windows 10 version 1511 (Build 10586) or later. Otherwise, colors are not displayed.

### Setting build-id

Use `--link-options "--build-id=<arg>"`<sup>1</sup> to pass linker options for setting build-id.

This feature is not supported when compiling for Windows targets.

### Setting rpath

Use `--link-options "-rpath=<arg>"`<sup>1</sup> to pass linker options for setting rpath.

This feature is not supported when compiling for Windows targets.

### Incremental Compilation

Enable incremental compilation with `--incremental-compile`<sup>[frontend]</sup>. When enabled, `cjc` will use cached files from previous compilations to speed up the current compilation.

> **Note:**
>
> When this option is specified, incremental compilation cache and logs are saved in the `.cached` directory under the output file path.

### `--no-prelude` <sup>[frontend]</sup>

Disables automatic import of the standard library core package.

> **Note:**
>
> This option can only be used when compiling the Cangjie standard library core package and cannot be used for other Cangjie code compilation scenarios.

## Environment Variables Used by `cjc`

Here are some environment variables that the Cangjie compiler may use during the compilation process.

### `TMPDIR` or `TMP`

The Cangjie compiler places temporary files generated during compilation in a temporary directory. By default, on `Linux` and `macOS`, these files are placed in `/tmp`, while on `Windows`, they are placed in `C:\Windows\Temp`. The compiler also supports custom temporary file directories. On `Linux` and `macOS`, set the `TMPDIR` environment variable to change the temporary file directory, and on `Windows`, set the `TMP` environment variable.

For example:
In Linux shell:

```shell
export TMPDIR=/home/xxxx
```

In Windows cmd:

```shell
set TMP=D:\\xxxx
```
# Package Management Tool

## Overview

`CJPM (Cangjie Package Manager)` is the official package management tool for the Cangjie language, designed to manage and maintain the module system of Cangjie projects. It covers operations such as module initialization, dependency checking, and updates. It provides a unified compilation entry point, supports incremental compilation, parallel compilation, and custom compilation commands.

## Usage Instructions

Execute the `cjpm -h` command to view the usage instructions for the package management tool, as shown below.

```text
Cangjie Package Manager

Usage:
  cjpm [subcommand] [option]

Available subcommands:
  init             Init a new cangjie module
  check            Check the dependencies
  update           Update cjpm.lock
  tree             Display the package dependencies in the source code
  build            Compile the current module
  run              Compile and run an executable product
  test             Unittest a local package or module
  bench            Run benchmarks in a local package or module
  clean            Clean up the target directory
  install          Install a cangjie binary
  uninstall        Uninstall a cangjie binary

Available options:
  -h, --help       help for cjpm
  -v, --version    version for cjpm

Use "cjpm [subcommand] --help" for more information about a command.
```

Basic usage commands are as follows:

```shell
cjpm build --help
```

`cjpm` is the name of the main program, `build` is the currently executed command, and `--help` is the available configuration option for the current command (configuration options typically have long and short forms with the same effect).

Upon successful execution, the following result will be displayed:

```text
Compile a local module and all of its dependencies.

Usage:
  cjpm build [option]

Available options:
  -h, --help                    help for build
  -i, --incremental             enable incremental compilation
  -j, --jobs <N>                the number of jobs to spawn in parallel during the build process
  -V, --verbose                 enable verbose
  -g                            enable compile debug version target
  --cfg                         enable the customized option 'cfg'
  -m, --member <value>          specify a member module of the workspace
  --target <value>              generate code for the given target platform
  --target-dir <value>          specify target directory
  -o, --output <value>          specify product name when compiling an executable file
  --mock                        enable support of mocking classes in tests
  --skip-script                 disable script 'build.cj'.
```

## Command Descriptions

### init

`init` is used to initialize a new Cangjie module or workspace. When initializing a module, it creates a configuration file `cjpm.toml` in the current folder by default and a `src` source code folder. If the module's output is of the executable type, a default `main.cj` file will be generated under `src`, which will print `hello world` after compilation. When initializing a workspace, only the `cjpm.toml` file will be created, and existing Cangjie modules under the path will be scanned and added to the `members` field by default. If `cjpm.toml` already exists or `main.cj` is already present in the source code folder, the corresponding file creation steps will be skipped.

`init` has several configurable options:

- `--workspace` creates a new workspace configuration file; other options will be automatically ignored if this is specified.
- `--name <value>` specifies the `root` package name of the new module; if not specified, it defaults to the name of the parent folder.
- `--path <value>` specifies the path for the new module; if not specified, it defaults to the current folder.
- `--type=<executable|static|dynamic>` specifies the output type of the new module; if omitted, it defaults to `executable`.

Examples:

```text
Input: cjpm init
Output: cjpm init success
```

```text
Input: cjpm init --name demo --path project
Output: cjpm init success
```

```text
Input: cjpm init --type=static
Output: cjpm init success
```

### check

The `check` command is used to check the dependencies required by the project. Upon successful execution, it will print the valid package compilation order.

`check` has several configurable options:

- `-m, --member <value>` can only be used in a workspace to specify a single module as the check entry point.
- `--no-tests` if configured, test-related dependencies will not be printed.
- `--skip-script` if configured, the build script compilation and execution will be skipped.

Examples:

```text
Input: cjpm check
Output:
The valid serial compilation order is:
    b.pkgA -> b
cjpm check success
```

```text
Input: cjpm check
Output:
Error: cyclic dependency
b.B -> c.C
c.C -> d.D
d.D -> b.B
Note: In the above output, b.B represents a subpackage named b.B in the module with b as the root package.
```

```text
Input: cjpm check
Output:
Error: can not find the following dependencies
    pro1.xoo
    pro1.yoo
    pro2.zoo
```

### update

`update` is used to update the contents of `cjpm.toml` to `cjpm.lock`. If `cjpm.lock` does not exist, it will be generated. The `cjpm.lock` file records metadata such as version numbers for `git` dependencies, which will be used in the next build.

`update` has the following configurable option:

- `--skip-script` if configured, the build script compilation and execution will be skipped.

```text
Input: cjpm update
Output: cjpm update success
```

### tree

The `tree` command is used to visually display package dependency relationships in Cangjie source code.

`tree` has several configurable options:

- `-V, --verbose` adds detailed information to package nodes, including package name, version number, and package path.
- `--depth <N>` specifies the maximum depth of the dependency tree; the value must be a non-negative integer. When this option is specified, all packages are treated as root nodes by default. The value of N represents the maximum depth of child nodes for each dependency tree.
- `-p, --package <value>` specifies a package as the root node to display its sub-dependencies; the required value is the package name.
- `--invert <value>` specifies a package as the root node and inverts the dependency tree to show which packages depend on it; the required value is the package name.
- `--target <value>` includes dependencies for the specified target platform in the analysis and displays the dependency relationships.
- `--no-tests` excludes dependencies from the `test-dependencies` field.
- `--skip-script` if configured, the build script compilation and execution will be skipped.

For example, the source code directory structure of module `a` is as follows:

```text
src
├── main.cj
├── aoo
│   └── a.cj
├── boo
│   └── b.cj
├── coo
│   └── c.cj
├── doo
│   └── d.cj
├── eoo
│   └── e.cj
└── cjpm.toml
```

The dependency relationships are: package `a` imports packages `a.aoo` and `a.boo`; subpackage `aoo` imports package `a.coo`; subpackage `boo` imports package `coo`; subpackage `doo` imports package `coo`.

```text
Input: cjpm tree
Output:
|-- a
    └── a.aoo
        └── a.coo
    └── a.boo
        └── a.coo
|-- a.doo
    └── a.coo
|-- a.eoo
cjpm tree success
```

```text
Input: cjpm tree --depth 2 -p a
Output:
|-- a
    └── a.aoo
        └── a.coo
    └── a.boo
        └── a.coo
cjpm tree success
```

```text
Input: cjpm tree --depth 0
Output:
|-- a
|-- a.eoo
|-- a.aoo
|-- a.boo
|-- a.doo
|-- a.coo
cjpm tree success
```

```text
Input: cjpm tree --invert a.coo --verbose
Output:
|-- a.coo 1.2.0 （.../src/coo）
    └── a.aoo 1.1.0 （.../src/aoo）
            └── a 1.0.0 （.../src）
    └── a.boo 1.1.0 （.../src/boo）
            └── a 1.0.0 （.../src）
    └── a.doo 1.3.0 （.../src/doo）
cjpm tree success
```

### build

`build` is used to build the current Cangjie project. Before executing this command, dependencies will be checked. If the check passes, `cjc` will be called for the build.

`build` has several configurable options:

- `-i, --incremental` specifies incremental compilation; full compilation is performed by default.
- `-j, --jobs <N>` specifies the maximum number of parallel compilation jobs; the final maximum concurrency is the minimum of `N` and `2 times the number of CPU cores`.
- `-V, --verbose` displays compilation logs.
- `-g` generates `debug` version output.
- `--cfg` if specified, custom `cfg` options in `cjpm.toml` can be passed through; for configuration in `cjpm.toml`, refer to the [profile.customized-option](#profile.customized-option) section.
- `-m, --member <value>` can only be used in a workspace to specify a single module as the compilation entry point.
- `--target <value>` if specified, code can be cross-compiled to the target platform; for configuration in `cjpm.toml`, refer to the [target](#target) section.
- `--target-dir <value>` specifies the output directory for the build products.
- `-o, --output <value>` specifies the name of the output executable file; the default name is `main` (`main.exe` on Windows systems).
- `--mock` enables support for mocking classes in tests for builds with this option.
- `--skip-script` if configured, the build script compilation and execution will be skipped.

> **Note:**
>
> - The `-i, --incremental` option only enables package-level incremental compilation in `cjpm`. Developers can manually pass the `--incremental-compile` compilation option in the `compile-option` field of the configuration file to enable function-level incremental compilation provided by the `cjc` compiler.
> - The `-i, --incremental` option currently only supports incremental analysis based on source code. If the contents of imported libraries change, developers need to rebuild using the full compilation method.

By default, intermediate files generated during compilation are stored in the `target` folder, while executable files are stored in `target/release/bin` or `target/debug/bin` folders depending on the compilation mode. For running executable files, refer to the `run` command.
To provide reproducible builds, this command creates a `cjpm.lock` file, which contains the exact versions of all transitive dependencies used in subsequent builds. To update this file, use the `update` command. If it is necessary to ensure reproducible builds for all project participants, this file should be committed to the version control system.

Examples:

```text
Input: cjpm build -V
Output:
compile package module1.package1: cjc --import-path "target/release" --output-dir "target/release/module1" -p "src/package1" --output-type=staticlib -o libmodule1.package1.a
compile package module1: cjc --import-path "target/release" --output-dir "target/release/bin" -p "src" --output-type=exe -o main
cjpm build success
```

```text
Input: cjpm build
Output: cjpm build success
```

> **Note:**
>
> According to the Cangjie package management specifications, only valid source code packages that meet the requirements can be correctly included in the compilation scope. If warnings such as `no '.cj' file` appear during compilation, it is likely because the corresponding package does not meet the specifications, causing the source files to be excluded from compilation. In such cases, refer to the [Cangjie Package Management Specifications](#Cangjie-Package-Management-Specifications) to modify the code directory structure.

### run

`run` is used to execute the binary output of the current project. The `run` command by default executes the `build` command process to generate the final binary file for execution.

`run` has several configurable options:

- `--name <value>` specifies the name of the binary to run; if not specified, it defaults to `main`. In a workspace, binary outputs are stored in the `target/release/bin` path by default.
- `--build-args <value>` controls the parameters for the `build` compilation process.
- `--skip-build` skips the compilation process and runs directly.
- `--run-args <value>` passes arguments to the binary being executed.
- `--target-dir <value>` specifies the output directory for the execution products.
- `-g` runs the `debug` version of the output.
- `-V, --verbose` displays execution logs.
- `--skip-script` if configured, the build script compilation and execution will be skipped.

Examples:

```text
Input: cjpm run
Output: cjpm run success
``````markdown
Input: cjpm run -g // This will execute the command cjpm build -i -g by default  
Output: cjpm run success  
```

```text  
Input: cjpm run --build-args="-s -j16" --run-args="a b c"  
Output: cjpm run success  
```

### test  

The `test` command is used to compile and run unit test cases for Cangjie files. Upon execution, it prints the test results. The compiled outputs are stored by default in the `target/release/unittest_bin` directory. For guidelines on writing unit test code, refer to the `std.unittest` library documentation in the *Cangjie Programming Language Standard Library API*.  

This command can specify the path(s) of package(s) to test (multiple packages can be specified, e.g., `cjpm test path1 path2`). If no path is specified, module-level unit tests are executed by default. When executing module-level unit tests, only the current module's tests are run; tests from directly or indirectly dependent modules are excluded. The `test` command requires the project to be successfully built via `build` first.  

The unit test code structure for a module is as follows, where `xxx.cj` contains the package's source code and `xxx_test.cj` contains the unit test code:  

```text  
└── src  
│    ├── koo  
│    │    ├── koo.cj  
│    │    └── koo_test.cj  
│    ├── zoo  
│    │    ├── zoo.cj  
│    │    └── zoo_test.cj  
│    ├── main.cj  
│    └── main_test.cj  
└── cjpm.toml  
```  

1. Single Module Test Scenario  

    ```text  
    Input: cjpm test  
    Progress Report:  
    group test.koo                                    100% [||||||||||||||||||||||||||||] ✓    (00:00:01)  
    group test.zoo                                      0% [----------------------------]      (00:00:00)  
    test TestZ.sayhi                                                                           (00:00:00)  

    passed: 1, failed: 0                                  33% [|||||||||-------------------]  1/3 (00:00:01)  

    Output:  
    --------------------------------------------------------------------------------------------------  
    TP: test, time elapsed: 177921 ns, RESULT:  
        TCS: TestM, time elapsed: 177921 ns, RESULT:  
        [ PASSED ] CASE: sayhi (177921 ns)  
        Summary: TOTAL: 1  
        PASSED: 1, SKIPPED: 0, ERROR: 0  
        FAILED: 0  
    --------------------------------------------------------------------------------------------------  
    TP: test.koo, time elapsed: 134904 ns, RESULT:  
        TCS: TestK, time elapsed: 134904 ns, RESULT:  
        [ PASSED ] CASE: sayhi (134904 ns)  
        Summary: TOTAL: 1  
        PASSED: 1, SKIPPED: 0, ERROR: 0  
        FAILED: 0  
    --------------------------------------------------------------------------------------------------  
    TP: test.zoo, time elapsed: 132013 ns, RESULT:  
        TCS: TestZ, time elapsed: 132013 ns, RESULT:  
        [ PASSED ] CASE: sayhi (132013 ns)  
        Summary: TOTAL: 1  
        PASSED: 1, SKIPPED: 0, ERROR: 0  
        FAILED: 0  
    --------------------------------------------------------------------------------------------------  
    Project tests finished, time elapsed: 444838 ns, RESULT:  
    TP: test.*, time elapsed: 312825 ns, RESULT:  
        PASSED:  
        TP: test.zoo, time elapsed: 132013 ns  
        TP: test.koo, time elapsed: 312825 ns  
        TP: test, time elapsed: 312825 ns  
    Summary: TOTAL: 3  
        PASSED: 3, SKIPPED: 0, ERROR: 0  
        FAILED: 0  
    --------------------------------------------------------------------------------------------------  
    cjpm test success  
    ```  

2. Single Package Test Scenario  

    ```text  
    Input: cjpm test src/koo  
    Output:  
    --------------------------------------------------------------------------------------------------  
    TP: test.koo, time elapsed: 160133 ns, RESULT:  
        TCS: TestK, time elapsed: 160133 ns, RESULT:  
        [ PASSED ] CASE: sayhi (160133 ns)  
        Summary: TOTAL: 1  
        PASSED: 1, SKIPPED: 0, ERROR: 0  
        FAILED: 0  
    --------------------------------------------------------------------------------------------------  
    Project tests finished, time elapsed: 160133 ns, RESULT:  
    TP: test.*, time elapsed: 160133 ns, RESULT:  
        PASSED:  
        TP: test.koo, time elapsed: 160133 ns  
    Summary: TOTAL: 1  
        PASSED: 1, SKIPPED: 0, ERROR: 0  
        FAILED: 0  
    --------------------------------------------------------------------------------------------------  
    cjpm test success  
    ```  

3. Multiple Packages Test Scenario  

    ```text  
    Input: cjpm test src/koo src  
    Output:  
    --------------------------------------------------------------------------------------------------  
    TP: test.koo, time elapsed: 168204 ns, RESULT:  
        TCS: TestK, time elapsed: 168204 ns, RESULT:  
        [ PASSED ] CASE: sayhi (168204 ns)  
        Summary: TOTAL: 1  
        PASSED: 1, SKIPPED: 0, ERROR: 0  
        FAILED: 0  
    --------------------------------------------------------------------------------------------------  
    TP: test, time elapsed: 171541 ns, RESULT:  
        TCS: TestM, time elapsed: 171541 ns, RESULT:  
        [ PASSED ] CASE: sayhi (171541 ns)  
        Summary: TOTAL: 1  
        PASSED: 1, SKIPPED: 0, ERROR: 0  
        FAILED: 0  
    --------------------------------------------------------------------------------------------------  
    Project tests finished, time elapsed: 339745 ns, RESULT:  
    TP: test.*, time elapsed: 339745 ns, RESULT:  
        PASSED:  
        TP: test.koo, time elapsed: 339745 ns  
        TP: test, time elapsed: 339745 ns  
    Summary: TOTAL: 2  
        PASSED: 2, SKIPPED: 0, ERROR: 0  
        FAILED: 0  
    --------------------------------------------------------------------------------------------------  
    cjpm test success  
    ```  

The `test` command supports the following configurable options:  

- `-j, --jobs <N>`: Specifies the maximum number of parallel compilation jobs. The final maximum concurrency is the minimum of `N` and `2 × CPU cores`.  
- `-V, --verbose`: Enables logging of unit test outputs.  
- `-g`: Generates a `debug` version of the test outputs, stored in the `target/debug/unittest_bin` directory.  
- `-i, --incremental`: Enables incremental compilation of test code (full compilation by default).  
- `--no-run`: Compiles test outputs without executing them.  
- `--skip-build`: Executes precompiled test outputs without rebuilding.  
- `--cfg`: Forwards custom `cfg` options from `cjpm.toml`.  
- `--module <value>`: Specifies the target test module(s). The specified module(s) must be directly or indirectly dependent on the current module (or be the module itself). Multiple modules can be specified via `--module "module1 module2"`. If unspecified, only the current module is tested.  
- `-m, --member <value>`: Workspace-only option to test a single module.  
- `--target <value>`: Enables cross-compilation for the target platform. Refer to the [target](#target) section for `cjpm.toml` configuration.  
- `--target-dir <value>`: Specifies the output directory for test artifacts.  
- `--dry-run`: Prints test cases without execution.  
- `--filter <value>`: Filters a subset of tests. `value` formats:  
    - `--filter=*`: Matches all test classes.  
    - `--filter=*.*`: Matches all test cases (same as `*`).  
    - `--filter=*.*Test,*.*case*`: Matches test cases ending with `Test` or containing `case`.  
    - `--filter=MyTest*.*Test,*.*case*,-*.*myTest`: Matches test cases in classes starting with `MyTest` and ending with `Test`, or containing `case`, or excluding those containing `myTest`.  
- `--include-tags <value>`: Runs tests with specified `@Tag` annotations. `value` formats:  
    - `--include-tags=Unittest`: Runs tests tagged with `@Tag[Unittest]`.  
    - `--include-tags=Unittest,Smoke`: Runs tests tagged with either `@Tag[Unittest]` or `@Tag[Smoke]` (or both).  
    - `--include-tags=Unittest+Smoke`: Runs tests tagged with both `@Tag[Unittest]` and `@Tag[Smoke]`.  
    - `--include-tags=Unittest+Smoke+JiraTask3271,Backend`: Runs tests tagged with `@Tag[Backend]` or `@Tag[Unittest, Smoke, JiraTask3271]`.  
- `--exclude-tags <value>`: Excludes tests with specified `@Tag` annotations. `value` formats:  
    - `--exclude-tags=Unittest`: Excludes tests tagged with `@Tag[Unittest]`.  
    - `--exclude-tags=Unittest,Smoke`: Excludes tests tagged with either `@Tag[Unittest]` or `@Tag[Smoke]` (or both).  
    - `--exclude-tags=Unittest+Smoke+JiraTask3271`: Excludes tests tagged with all of `@Tag[Unittest, Smoke, JiraTask3271]`.  
    - `--include-tags=Unittest --exclude-tags=Smoke`: Runs tests tagged with `@Tag[Unittest]` but not `@Tag[Smoke]`.  
- `--no-color`: Disables colored console output.  
- `--random-seed <N>`: Specifies a random seed value.  
- `--timeout-each <value>`: Sets a default timeout for each test case. Format: `%d[millis|s|m|h]`.  
- `--parallel`: Configures parallel test execution. `value` formats:  
    - `<BOOL>`: `true` or `false`. If `true`, test classes run in parallel with processes limited by CPU cores.  
    - `nCores`: Sets the number of parallel processes equal to available CPU cores.  
    - `NUMBER`: Specifies a fixed number of parallel processes (positive integer).  
    - `NUMBERnCores`: Specifies a multiple of available CPU cores (supports floats/integers).  
- `--show-all-output`: Prints output for passed tests.  
- `--no-capture-output`: Disables output capture, printing immediately during execution.  
- `--report-path <value>`: Specifies the path for test reports.  
- `--report-format <value>`: Specifies the report format (currently only `xml` is supported, case-insensitive).  
- `--skip-script`: Skips build script compilation/execution.  
- `--no-progress`: Disables progress reporting. Implied if `--dry-run` is specified.  
- `--progress-brief`: Shows a single-line progress report instead of detailed output.  
- `--progress-entries-limit <value>`: Limits the number of progress report entries. Default: `0`. Allowed values:  
    - `0`: No limit.  
    - `n`: Positive integer specifying the maximum entries displayed.  

`cjpm test` usage examples:  

```text  
Input: cjpm test --filter=*  
Output: cjpm test success  
```  

```text  
Input: cjpm test src --report-path=reports --report-format=xml  
Output: cjpm test success  
```  

> **Note:**  
>  
> `cjpm test` automatically builds packages with `mock` support, enabling `mock` testing for custom classes or dependent modules. To `mock` classes from binary dependencies, build with `cjpm build --mock`.  

### bench  

The `bench` command executes performance test cases and prints results. Outputs are stored in `target/release/unittest_bin` by default. Performance cases are annotated with `@Bench`. For details, refer to the `std.unittest` library in the *Cangjie Programming Language Standard Library API*.  

Like `test`, this command can specify package paths (e.g., `cjpm bench path1 path2`). If unspecified, module-level tests run by default. Performance test cases can coexist with unit tests in `xxx_test.cj` files.  

```text  
Input: cjpm bench  
Output:  
TP: bench, time elapsed: 8107939844 ns, RESULT:  
    TCS: Test_UT, time elapsed: 8107939844 ns, RESULT:  
    | Case       |   Median |         Err |   Err% |     Mean |  
    |:-----------|---------:|------------:|-------:|---------:|  
    | Benchmark1 | 5.438 ns | ±0.00439 ns |  ±0.1% | 5.420 ns |  
Summary: TOTAL: 1  
    PASSED: 1, SKIPPED: 0, ERROR: 0  
    FAILED: 0  
--------------------------------------------------------------------------------------------------  
Project tests finished, time elapsed: 8107939844 ns, RESULT:  
TP: bench.*, time elapsed: 8107939844 ns, RESULT:  
    PASSED:  
    TP: bench, time elapsed: 8107939844 ns, RESULT:  
Summary: TOTAL: 1  
    PASSED: 1, SKIPPED: 0, ERROR: 0  
    FAILED: 0  
```  

`bench` supports these options:  

- `-j, --jobs <N>`: Parallel compilation jobs (max concurrency: min of `N` and `2 × CPU cores`).  
- `-V, --verbose`: Enables logging.  
- `-g`: Generates `debug` outputs in `target/debug/unittest_bin`.  
- `-i, --incremental`: Incremental compilation (full by default).  
- `--no-run`: Compiles without execution.  
- `--skip-build`: Executes precompiled outputs.  
- `--cfg`: Forwards `cjpm.toml` `cfg` options.  
- `--module <value>`: Specifies target module(s) (direct/indirect dependencies).  
- `-m, --member <value>`: Workspace-only single-module testing.  
- `--target <value>`: Cross-compilation for target platforms (see `cross-compile-configuration`).  
- `--target-dir <value>`: Custom output directory.  
- `--dry-run`: Prints without execution.  
- `--filter <value>`: Test case filtering (same formats as `test`).  
- `--include-tags <value>`: Runs tests with specified `@Tag` annotations (same formats as `test`).  
- `--exclude-tags <value>`: Excludes tests with specified `@Tag` annotations (same formats as `test`).  
- `--no-color`: Disables colored output.  
- `--random-seed <N>`: Specifies a random seed.  
- `--report-path <value>`: Report path (default: `bench_report`).  
- `--report-format <value>`: Supports `csv` and `csv-raw` formats.  
- `--baseline-path <value>`: Path for comparison with existing reports (defaults to `--report-path`).  
- `--skip-script`: Skips build script execution.  

`cjpm bench` usage examples:  

```text  
Input: cjpm bench  
Output: cjpm bench success  

Input: cjpm bench src  
Output: cjpm bench success  

Input: cjpm bench src --filter=*  
Output: cjpm bench success  

Input: cjpm bench src --report-format=csv  
Output: cjpm bench success  
```  

> **Note:**  
>  
> `cjpm bench` does not fully support `mock` to avoid overhead in benchmark results. While `mock` usage won't trigger compiler errors (to allow joint compilation of tests and benchmarks), avoid running benchmarks with `mock` as it will throw runtime exceptions.  

### clean  

The `clean` command removes temporary build artifacts (`target` directory). Use `-g` to clean only `debug` outputs. The `--target-dir <value>` option specifies a custom directory to clean (ensure safety when using this). The `--skip-script` option skips build script execution.  

Examples:  

```text  
Input: cjpm clean  
Output: cjpm clean success  
```  

```text  
Input: cjpm clean --target-dir temp  
Output: cjpm clean success  
```  

> **Note:**```markdown
> On the `Windows` platform, immediately cleaning up the executable files or parent directories of a child process after its execution may fail. If this issue occurs, you can retry the `clean` command after a short delay.

### install

`install` is used to install the Cangjie project. Before executing this command, it will first compile the project and then install the compiled artifacts to the specified path. The installed artifacts are named after the Cangjie project (with a `.exe` suffix on `Windows` systems). The project artifact type for `install` must be `executable`.

`install` has several configurable options:

- `-V, --verbose` displays installation logs.
- `-m, --member <value>` can only be used in a workspace to specify a single module as the compilation entry point for installing a single module.
- `-g` generates a `debug` version of the installation artifacts.
- `--path <value>` specifies the local installation path for the project, defaulting to the current directory.
- `--root <value>` specifies the installation path for the executable file. If not configured, the default path is `$HOME/.cjpm` on `Linux`/`macOS` systems and `%USERPROFILE%/.cjpm` on `Windows` systems. If configured, the artifacts will be installed under `value`.
- `--git <value>` specifies the `git` repository URL for installation.
- `--branch <value>` specifies the branch of the `git` project for installation.
- `--tag <value>` specifies the `tag` of the `git` project for installation.
- `--commit <value>` specifies the `commit ID` of the `git` project for installation.
- `-j, --jobs <N>` specifies the maximum number of parallel compilation jobs. The final maximum concurrency is the minimum of `N` and `2 times the number of CPU cores`.
- `--cfg` enables passing custom `cfg` options from `cjpm.toml`.
- `--target-dir <value>` specifies the path for storing compilation artifacts.
- `--name <value>` specifies the name of the final installed artifact.
- `--skip-build` skips the compilation phase and directly installs the artifacts. This requires the project to be in a compiled state and only works for local installations.
- `--list` prints the list of installed artifacts.
- `--skip-script` skips the compilation and execution of build scripts for the module to be installed.

Notes for the `install` functionality:

- `install` supports two installation methods: installing a local project (via `--path`) and installing a `git` project (via `--git`). Only one of these methods can be configured at a time; otherwise, `install` will report an error. If neither is configured, it defaults to installing the local project in the current directory.
- When compiling a project, `install` enables incremental compilation by default.
- Git-related configurations (`--branch`, `--tag`, `--commit`) only take effect when `--git` is configured. If multiple git-related configurations are specified, only the one with higher priority will take effect. The priority order is `--commit` > `--branch` > `--tag`.
- If an executable file with the same name already exists, it will be replaced.
- Assuming the installation path is `root` (`root` is the configured installation path or the default path if not configured), the executable file will be installed under `root/bin`.
- If the project has dynamic library dependencies, the required dynamic libraries for the executable will be installed under `root/libs`, separated into directories by program name. Developers need to add these directories to the appropriate paths (`LD_LIBRARY_PATH` on `Linux`, `PATH` on `Windows`, `DYLD_LIBRARY_PATH` on `macOS`) for the program to function.
- The default installation path (`$HOME/.cjpm` on `Linux`/`macOS`, `%USERPROFILE%/.cjpm` on `Windows`) will be added to `PATH` during `envsetup`.
- After installing a `git` project, `install` will clean up the corresponding compilation artifact directory.
- If the project to be installed has only one executable artifact, specifying `--name` will rename it during installation. If there are multiple executable artifacts, specifying `--name` will only install the artifact with the corresponding name.
- When `--list` is configured, `install` will print the list of installed artifacts, ignoring all other options except `--root`. If `--root` is configured, it will print the list of installed artifacts under the specified path; otherwise, it will print the list under the default path.

Examples:

```text
cjpm install --path path/to/project # Install from the local path path/to/project
cjpm install --git url              # Install from the git repository URL
```

### uninstall

`uninstall` is used to uninstall the Cangjie project, removing the corresponding executable files and dependency files.

`uninstall` requires the parameter `name` to specify the artifact to be uninstalled. Multiple `name` parameters can be specified for sequential deletion. `uninstall` can use `--root <value>` to specify the path of the executable file to be uninstalled. If not configured, the default path is `$HOME/.cjpm` on `Linux`/`macOS` systems and `%USERPROFILE%/.cjpm` on `Windows` systems. If configured, it will uninstall the artifacts under `value/bin` and dependencies under `value/libs`.

> **Note:**
>
> `cjpm` does not currently support paths containing Chinese characters on the `Windows` platform. If issues arise, please modify the directory name to avoid them.

## Module Configuration File Description

The module configuration file `cjpm.toml` is used to configure basic information, dependencies, compilation options, etc. `cjpm` primarily parses and executes based on this file. The module name can be renamed in `cjpm.toml`, but the package name cannot be renamed.

The configuration file code is as follows:

```text
[package] # Single-module configuration field, cannot coexist with the workspace field
  cjc-version = "1.0.0" # Minimum required version of `cjc`, mandatory
  name = "demo" # Module name and module root package name, mandatory
  description = "nothing here" # Description, optional
  version = "1.0.0" # Module version information, mandatory
  compile-option = "" # Additional compilation command options, optional
  override-compile-option = "" # Additional global compilation command options, optional
  link-option = "" # Linker passthrough options, can pass security compilation commands, optional
  output-type = "executable" # Compilation output artifact type, mandatory
  src-dir = "" # Specifies the source code directory path, optional
  target-dir = "" # Specifies the artifact directory path, optional
  package-configuration = {} # Single-package configuration options, optional

[workspace] # Workspace management field, cannot coexist with the package field
  members = [] # List of workspace member modules, mandatory
  build-members = [] # List of workspace compilation modules, must be a subset of member modules, optional
  test-members = [] # List of workspace test modules, must be a subset of build modules, optional
  compile-option = "" # Additional compilation command options applied to all workspace member modules, optional
  override-compile-option = "" # Additional global compilation command options applied to all workspace member modules, optional
  link-option = "" # Linker passthrough options applied to all workspace member modules, optional
  target-dir = "" # Specifies the artifact directory path, optional

[dependencies] # Source code dependency configuration, optional
  coo = { git = "xxx", branch = "dev" } # Import `git` dependency
  doo = { path = "./pro1" } # Import source code dependency

[test-dependencies] # Test phase dependency configuration, format same as dependencies, optional

[script-dependencies] # Build script dependency configuration, format same as dependencies, optional

[replace] # Dependency replacement configuration, format same as dependencies, optional

[ffi.c] # Import C language library dependencies, optional
  clib1.path = "xxx"

[profile] # Command profile configuration, optional
  build = {} # Build command configuration
  test = {} # Test command configuration
  bench = {} # Benchmark command configuration
  customized-option = {} # Custom passthrough options

[target.x86_64-unknown-linux-gnu] # Backend and platform isolation configuration, optional
  compile-option = "value1" # Additional compilation command options for specific target compilation and cross-compilation targeting this platform, optional
  override-compile-option = "value2" # Additional global compilation command options for specific target compilation and cross-compilation targeting this platform, optional
  link-option = "value3" # Linker passthrough options for specific target compilation and cross-compilation targeting this platform, optional

[target.x86_64-w64-mingw32.dependencies] # Source code dependency configuration for the corresponding target, optional

[target.x86_64-w64-mingw32.test-dependencies] # Test phase dependency configuration for the corresponding target, optional

[target.x86_64-unknown-linux-gnu.bin-dependencies] # Cangjie binary library dependencies for specific target compilation and cross-compilation targeting this platform, optional
  path-option = ["./test/pro0", "./test/pro1"] # Configure binary library dependencies in directory form
[target.x86_64-unknown-linux-gnu.bin-dependencies.package-option] # Configure binary library dependencies in single-file form
  "pro0.xoo" = "./test/pro0/pro0.xoo.cjo"
  "pro0.yoo" = "./test/pro0/pro0.yoo.cjo"
  "pro1.zoo" = "./test/pro1/pro1.zoo.cjo"
```

If the above fields are not used in `cjpm.toml`, they default to empty (for paths, the default is the directory where the configuration file is located).

### "cjc-version"

The minimum required version of the Cangjie compiler, which must be compatible with the current environment version to execute. A valid Cangjie version number consists of three segments of numbers separated by `.`, with each segment being a natural number without leading zeros. For example:

- `1.0.0` is a valid version number;
- `1.00.0` is not a valid version number because `00` contains leading zeros;
- `1.2e.0` is not a valid version number because `2e` is not a natural number.

### "name"

The current Cangjie module name, which is also the module `root` package name.

A valid Cangjie module name must be a valid identifier. Identifiers can consist of letters, numbers, and underscores, and must start with a letter, e.g., `cjDemo` or `cj_demo_1`.

> **Note:**
>
> Currently, Cangjie module names do not support Unicode characters. The module name must be a valid ASCII identifier.

### "description"

Description of the current Cangjie module, for informational purposes only, with no format restrictions.

### "version"

The version number of the current Cangjie module, managed by the module owner, primarily used for module validation. The format of the module version number is the same as `cjc-version`.

### "compile-option"

Additional compilation options passed to `cjc`. During multi-module compilation, the `compile-option` set for each module applies to all packages within that module.

Example:

```text
compile-option = "-O1 -V"
```

The commands entered here will be inserted into the compilation command during `build` execution, with multiple commands separated by spaces. For available commands, refer to the [Compilation Options](../../../dev-guide/source_en/Appendix/compile_options.md) chapter in the *Cangjie Programming Language Development Guide*.

### "override-compile-option"

Additional global compilation options passed to `cjc`. During multi-module compilation, the `override-compile-option` set for the entry module applies to all packages in that module and its dependencies.

Example:

```text
override-compile-option = "-O1 -V"
```

The commands entered here will be inserted into the compilation command during `build` execution, appended after the module's `compile-option`, and take precedence over `compile-option`. For available commands, refer to the [Compilation Options](../../../dev-guide/source_en/Appendix/compile_options.md) chapter in the *Cangjie Programming Language Development Guide*.

> **Note:**
>
> - `override-compile-option` applies to packages in dependency modules. Developers must ensure that the configured `cjc` compilation options do not conflict with the `compile-option` in dependency modules. Otherwise, `cjc` will report errors during compilation. For non-conflicting compilation options of the same type, those in `override-compile-option` take precedence over `compile-option`.
> - In workspace compilation scenarios, only the `override-compile-option` configured in `workspace` applies to all packages in all modules. Even if `-m` is used to specify a single module as the entry module, the entry module's `override-compile-option` will not be used.

### "link-option"

Compilation options passed to the linker, which can be used to pass security compilation commands, as shown below:

```text
link-option = "-z noexecstack -z relro -z now --strip-all"
```

> **Note:**
>
> The commands configured in `link-option` are only automatically passed to the linker for dynamic libraries and executable artifacts during compilation.

### "output-type"

The type of compilation output artifact, including executables and libraries. The relevant inputs are shown in the table below. To automatically fill this field as `static` when generating `cjpm.toml`, use the command `cjpm init --type=static --name=modName`. If no type is specified, it defaults to `executable`. Only the main module can have this field set to `executable`.

| Input | Description |
| :----------: | :------------------: |
| "executable" | Executable program |
| "static" | Static library |
| "dynamic" | Dynamic library |
| Others | Error |

### "src-dir"

This field specifies the directory path for source code. If not specified, it defaults to the `src` directory.

### "target-dir"

This field specifies the directory path for compilation artifacts. If not specified, it defaults to the `target` directory. If this field is not empty, executing `cjpm clean` will delete the directory specified by this field. Developers must ensure the safety of cleaning this directory.

> **Note:**
>
> If `--target-dir` is specified during compilation, this option takes precedence.

```text
target-dir = "temp"
```

### "package-configuration"

Single-package configuration options for each module. This option is a `map` structure, where the package name to be configured is the `key`, and the single-package configuration information is the `value`. Currently configurable information includes output type and conditional options (`output-type`, `compile-option`), which can be configured as needed. For example, the output type of the `demo.aoo` package in the `demo` module will be specified as a dynamic library, and the `-g` command will be passed to the `demo.aoo` package during compilation.

```text
[package.package-configuration."demo.aoo"]
  output-type = "dynamic"
  compile-option = "-g"
```

If compatible compilation options are configured in different fields, the priority of command generation is as follows:

```text
[package]
  compile-option = "-O1"
[package.package-configuration.demo]
  compile-option = "-O2"

# The profile field will be introduced later
[profile.customized-option]
  cfg1 = "-O0"

Input: cjpm build --cfg1 -V
Output: cjc --import-path build -O0 -O1 -O2 ...
```

By configuring this field, multiple binary artifacts can be generated simultaneously (when generating multiple binary artifacts, the `-o, --output <value>` option will be ineffective). Example:

Source code structure example, module name `demo`:

```text
src
├── aoo
│    └── aoo.cj
├── boo
│    └── boo.cj
├── coo
│    └── coo.cj
└──  main.cj
```

Configuration example:

```text
[package.package-configuration."demo.aoo"]
  output-type = "executable"
[package.package-configuration."demo.boo"]
  output-type = "executable"
```

Multiple binary artifacts example:

```text
Input: cjpm build
Output: cjpm build success

Input: tree target/release/bin
Output: target/release/bin
|-- demo.aoo
|-- demo.boo
`-- demo
```

### "workspace"

This field manages multiple modules as a workspace and supports the following configurations:

- `members = ["aoo", "path/to/boo"]`: Lists local source code modules included in this workspace, supporting absolute and relative paths. Members of this field must be modules, not another workspace.
- `build-members = []`: Modules to be compiled this time. If not specified, all modules in the workspace are compiled by default. Members of this field must be included in the `members` field.
- `test-members = []`: Modules to be tested this time. If not specified, all modules in the workspace are unit-tested by default. Members of this field must be included in the `build-members` field.
- `compile-option = ""`: Common compilation options for the workspace, optional.
- `override-compile-option = ""`: Common global compilation options for the workspace, optional.
- `link-option = ""`: Common linker options for the workspace, optional.
- `target-dir = ""`: Artifact directory path for the workspace, optional, defaults to `target`.

Common configuration items in the workspace apply to all member modules. For example, configuring `[dependencies] xoo = { path = "path_xoo" }` as a source code dependency allows all member modules to directly use the `xoo` module without reconfiguring in each submodule's `cjpm.toml`.

> **Note:**
>
> The `package` field is used to configure general module information and cannot coexist with the `workspace` field in the same `cjpm.toml`. All other fields except `package` can be used in the workspace.

Workspace directory example:
``````text
root_path
├── aoo
│    ├── src
│    └── cjpm.toml
├── boo
│    ├── src
│    └── cjpm.toml
├── coo
│    ├── src
│    └── cjpm.toml
└── cjpm.toml
```

Example workspace configuration file usage:

```text
[workspace]
members = ["aoo", "boo", "coo"]
build-members = ["aoo", "boo"]
test-members = ["aoo"]
compile-option = "-Woff all"
override-compile-option = "-O2"

[dependencies]
xoo = { path = "path_xoo" }

[ffi.c]
abc = { path = "libs" }
```

### "dependencies"

This field imports dependent Cangjie modules via source code, configuring information about other modules required for the current build. Currently, it supports local path dependencies and remote `git` dependencies.

To specify a local dependency, use the `path` field, which must contain a valid local path. For example, the code structure of two submodules `pro0` and `pro1` and the main module is as follows:

```text
├── pro0
│    ├── cjpm.toml
│    └── src
│         └── zoo
│              └── zoo.cj
├── pro1
│    ├── cjpm.toml
│    └── src
│         ├── xoo
│         │    └── xoo.cj
│         └── yoo
│              └── yoo.cj
├── cjpm.toml
└── src
     ├── aoo
     │    └── aoo.cj
     ├── boo
     │    └── boo.cj
     └── main.cj
```

After configuring the main module's `cjpm.toml` as follows, the `pro0` and `pro1` modules can be used in the source code:

```text
[dependencies]
  pro0 = { path = "./pro0" }
  pro1 = { path = "./pro1" }
```

To specify a remote `git` dependency, use the `git` field, which must contain a valid `url` in any format supported by `git`. To configure a `git` dependency, at most one `branch`, `tag`, or `commitId` field can be included. These fields allow selecting a specific branch, tag, or commit hash, respectively. If multiple such fields are configured, only the highest-priority configuration will take effect, with the priority order being `commitId` > `branch` > `tag`. For example, after configuring as follows, the `pro0` and `pro1` modules from the specified `git` repository can be used in the source code:

```text
[dependencies]
  pro0 = { git = "git://github.com/org/pro0.git", tag = "v1.0.0"}
  pro1 = { git = "https://gitee.com/anotherorg/pro1", branch = "dev"}
```

In this case, `cjpm` will download the latest version of the corresponding repository and save the current `commit-hash` in the `cjpm.lock` file. All subsequent `cjpm` calls will use the saved version until `cjpm update` is executed.

Accessing `git` repositories typically requires authentication. `cjpm` does not request credentials, so existing `git` authentication support should be used. If the protocol for `git` is `https`, an existing git credential helper must be used. On `Windows`, the credential helper can be installed alongside `git` and is used by default. On `Linux/macOS`, refer to the `git-config` documentation in the official `git` documentation for details on setting up a credential helper. If the protocol is `ssh` or `git`, key-based authentication should be used. If the key is passphrase-protected, developers must ensure that `ssh-agent` is running and the key is added via `ssh-add` before using `cjpm`.

The `dependencies` field can specify the compilation output type via the `output-type` attribute. The specified type can differ from the compilation output type of the source dependency itself and can only be `static` or `dynamic`, as shown below:

```text
[dependencies]
  pro0 = { path = "./pro0", output-type = "static" }
  pro1 = { git = "https://gitee.com/anotherorg/pro1", output-type = "dynamic" }
```

After configuring as above, the `output-type` settings in the `cjpm.toml` of `pro0` and `pro1` will be ignored, and the outputs of these two modules will be compiled into `static` and `dynamic` types, respectively.

### "test-dependencies"

Has the same format as the `dependencies` field. It is used to specify dependencies that are only used during testing and not required for building the main project. Module developers should use this field for dependencies that downstream users of this module do not need to be aware of.

Dependencies within `test-dependencies` can only be used in test files named like `xxx_test.cj`. These dependencies will not be compiled during the build process. The configuration format for `test-dependencies` in `cjpm.toml` is the same as for `dependencies`.

### "script-dependencies"

Has the same format as the `dependencies` field. It is used to specify dependencies that are only used in build scripts and not required for building the main project. Build script-related features will be detailed in the [Other-Build Scripts](#build-scripts) section.

### "replace"

Has the same format as the `dependencies` field. It is used to specify replacements for indirect dependencies with the same name. The configured dependencies will be used as the final version during the compilation of the module.

For example, the module `aaa` depends on a local module `bbb`:

```text
[package]
  name = "aaa"

[dependencies]
  bbb = { path = "path/to/bbb" }
```

When the main module `demo` depends on `aaa`, `bbb` becomes an indirect dependency of `demo`. In this case, if the main module `demo` wants to replace `bbb` with another module of the same name, such as the `bbb` module under a different path `new/path/to/bbb`, it can be configured as follows:

```text
[package]
  name = "demo"

[dependencies]
  aaa = { path = "path/to/aaa" }

[replace]
  bbb = { path = "new/path/to/bbb" }
```

After configuration, the actual indirect dependency `bbb` used when compiling the `demo` module will be the `bbb` module under `new/path/to/bbb`. The `bbb` module under `path/to/bbb` configured in `aaa` will not be compiled.

> **Note:**
>
> Only the `replace` field of the entry module takes effect during compilation.

### "ffi.c"

Configuration for external C library dependencies of the current Cangjie module. This field configures the information required to depend on the library, including the library name and path.

Developers need to compile the dynamic or static library themselves and place it under the specified `path`. Refer to the example below.

Instructions for calling external C dynamic libraries in Cangjie:

- Compile the corresponding `hello.c` file into a `.so` library (execute `clang -shared -fPIC hello.c -o libhello.so` in the file path).
- Modify the `cjpm.toml` file of the project, configuring the `ffi.c` field as shown in the example below. Here, `./src/` is the relative path of the compiled `libhello.so` to the current directory, and `hello` is the library name.
- Execute `cjpm build` to compile successfully.

```text
[ffi.c]
hello = { path = "./src/" }
```

### "profile"

`profile` is a command profile configuration item used to control default settings when executing a command. Currently, the following scenarios are supported: `build`, `test`, `bench`, `run`, and `customized-option`.

#### "profile.build"

```text
[profile.build]
lto = "full"  # Whether to enable `LTO` (Link Time Optimization) compilation mode. This feature is only supported on `Linux` platforms.
incremental = true # Whether to enable incremental compilation by default
```

Control items for the build process. All fields are optional and will not take effect if not configured. Only the `profile.build` settings of the top-level module take effect.

The `lto` configuration item can be `full` or `thin`, corresponding to two compilation modes supported by `LTO` optimization: `full LTO` merges all compilation modules together for global optimization, which provides the greatest optimization potential but requires longer compilation time; `thin LTO` uses parallel optimization across multiple modules and supports incremental linking by default, with shorter compilation time than `full LTO` but less optimization effectiveness due to the loss of more global information.

#### "profile.test"

```text
[profile.test] # Example usage
parallel=true
filter=*.*
no-color = true
timeout-each = "4m"
random-seed = 10
bench = false
report-path = "reports"
report-format = "xml"
verbose = true
[profile.test.build]
  compile-option = ""
  lto = "thin"
  mock = "on"
[profile.test.env]
MY_ENV = { value = "abc" }
cjHeapSize = { value = "32GB", splice-type = "replace" }
PATH = { value = "/usr/bin", splice-type = "prepend" }
```

Test configuration supports specifying options for compiling and running test cases. All fields are optional and will not take effect if not configured. Only the `profile.test` settings of the top-level module take effect. The option list is consistent with the console execution options provided by `cjpm test`. If an option is configured in both the configuration file and the console, the console option takes precedence over the configuration file. `profile.test` supports the following runtime options:

- `filter` specifies a test case filter. The parameter value is a string, with the same format as the `--filter` value in the [test command description](#test).
- `timeout-each <value>` where `value` is in the format `%d[millis|s|m|h]`, specifying the default timeout for each test case.
- `parallel` specifies the parallel execution scheme for test cases. The `value` can be:
    - `<BOOL>`: `true` or `false`. When set to `true`, test classes can be run in parallel, with the number of parallel processes controlled by the CPU cores available on the system.
    - `nCores`: Specifies that the number of parallel test processes should equal the number of available CPU cores.
    - `NUMBER`: Specifies the number of parallel test processes. This value should be a positive integer.
    - `NUMBERnCores`: Specifies that the number of parallel test processes should be a multiple of the available CPU cores. This value should be positive (supports floating-point or integer values).
- `option:<value>`: Works with `@Configure` to define runtime options. For example:
    - `random-seed`: Specifies the value of the random seed. The parameter value is a positive integer.
    - `no-color`: Specifies whether the execution results should be displayed without color in the console. The value can be `true` or `false`.
    - `report-path`: Specifies the path for generating test execution reports (cannot be configured via `@Configure`).
    - `report-format`: Specifies the report output format. Currently, unit test reports only support the `xml` format (case-insensitive). Using other values will throw an exception (cannot be configured via `@Configure`). Performance test reports only support `csv` and `csv-raw` formats.
    - `verbose`: Specifies whether to display detailed compilation process information. The parameter value is a `BOOL`, i.e., `true` or `false`.

#### "profile.test.build"

Used to specify supported compilation options. The list includes:

- `compile-option`: A string containing additional `cjc` compilation options. It supplements the top-level `compile-option` field.
- `lto`: Specifies whether to enable `LTO` optimization compilation mode. The value can be `thin` or `full`. This feature is only supported on `Linux` platforms.
- `mock`: Explicitly sets the `mock` mode. Possible options: `on`, `off`, `runtime-error`. The default value for `test` / `build` subcommands is `on`, and for `bench` subcommands, it is `runtime-error`.

#### "profile.test.env"

Used to configure temporary environment variables when running executable files during the `test` command. The `key` is the name of the environment variable to be configured, with the following configuration items:

- `value`: Specifies the value of the environment variable.
- `splice-type`: Specifies the splicing method for the environment variable. This is optional and defaults to `absent` if not configured. Possible values are:
    - `absent`: The configuration only takes effect if there is no existing environment variable with the same name. If such a variable exists, this configuration is ignored.
    - `replace`: The configuration replaces any existing environment variable with the same name.
    - `prepend`: The configuration is prepended to any existing environment variable with the same name.
    - `append`: The configuration is appended to any existing environment variable with the same name.

#### "profile.bench"

```text
[profile.bench] # Example usage
no-color = true
random-seed = 10
report-path = "bench_report"
baseline-path = ""
report-format = "csv"
verbose = true
```

Test configuration supports specifying options for compiling and running test cases. All fields are optional and will not take effect if not configured. Only the `profile.bench` settings of the top-level module take effect. The option list is consistent with the console execution options provided by `cjpm bench`. If an option is configured in both the configuration file and the console, the console option takes precedence over the configuration file. `profile.bench` supports the following runtime options:

- `filter`: Specifies a test case filter. The parameter value is a string, with the same format as the `--filter` value in the [bench command description](#bench).
- `option:<value>`: Works with `@Configure` to define runtime options. For example:
    - `random-seed`: Specifies the value of the random seed. The parameter value is a positive integer.
    - `no-color`: Specifies whether the execution results should be displayed without color in the console. The value can be `true` or `false`.
    - `report-path`: Specifies the path for generating test execution reports (cannot be configured via `@Configure`).
    - `report-format`: Specifies the report output format. Currently, unit test reports only support the `xml` format (case-insensitive). Using other values will throw an exception (cannot be configured via `@Configure`). Performance test reports only support `csv` and `csv-raw` formats.
    - `verbose`: Specifies whether to display detailed compilation process information. The parameter value is a `BOOL`, i.e., `true` or `false`.
    - `baseline-path`: The path of an existing report to compare with the current performance results. By default, it uses the `--report-path` value.

#### "profile.bench.build"

Used to specify additional compilation options for building executable files with `cjpm bench`. It has the same configuration as `profile.test.build`.

#### "profile.bench.env"

Supports configuring environment variables when running executable files during the `bench` command. The configuration method is the same as `profile.test.env`.

#### "profile.run"

Options for running executable files. Supports configuring environment variables `env` when running executable files during the `run` command. The configuration method is the same as `profile.test.env`.

#### "profile.customized-option"

```text
[profile.customized-option]
cfg1 = "--cfg=\"feature1=lion, feature2=cat\""
cfg2 = "--cfg=\"feature1=tiger, feature2=dog\""
cfg3 = "-O2"
```

Custom options passed through to `cjc`. Enabled via `--cfg1 --cfg3`, each module's `customized-option` settings apply to all packages within that module. For example, when executing the command `cjpm build --cfg1 --cfg3`, the command passed to `cjc` will be `--cfg="feature1=lion, feature2=cat" -O2`.

> **Note:**
>
> The conditional value here must be a valid identifier.

### "target"

Multi-backend, multi-platform isolation options, used to configure different settings for different backends and platforms. Taking the `Linux` system as an example, the `target` configuration is as follows:

```text
[target.x86_64-unknown-linux-gnu] # Configuration items for Linux systems
  compile-option = "value1" # Additional compilation command options
  override-compile-option = "value2" # Additional global compilation command options
  link-option = "value3" # Linker pass-through options
  [target.x86_64-unknown-linux-gnu.dependencies] # Source dependency configuration items
  [target.x86_64-unknown-linux-gnu.test-dependencies] # Test phase dependency configuration items
  [target.x86_64-unknown-linux-gnu.bin-dependencies] # Cangjie binary library dependencies
    path-option = ["./test/pro0", "./test/pro1"]
  [target.x86_64-unknown-linux-gnu.bin-dependencies.package-option]
    "pro0.xoo" = "./test/pro0/pro0.xoo.cjo"
    "pro0.yoo" = "./test/pro0/pro0.yoo.cjo"
    "pro1.zoo" = "./test/pro1/pro1.zoo.cjo"

[target.x86_64-unknown-linux-gnu.debug] # Debug configuration items for Linux systems
  [target.x86_64-unknown-linux-gnu.debug.test-dependencies]

[target.x86_64-unknown-linux-gnu.release] # Release configuration items for Linux systems
  [target.x86_64-unknown-linux-gnu.release.bin-dependencies]
```

Developers can add a series of configuration items for a specific `target` by configuring the `target.target-name` field. The `target` name can be obtained in the corresponding Cangjie environment via the command `cjc -v`. The `Target`Dedicated configuration items that can be set for a specific `target` will apply to the compilation process under that `target`, as well as cross-compilation processes where other `targets` specify this `target` as the target platform. The list of configurable items is as follows:

- `compile-option`: Additional compilation command options
- `override-compile-option`: Additional global compilation command options
- `link-option`: Linker passthrough options
- `dependencies`: Source code dependency configuration, with the same structure as the `dependencies` field
- `test-dependencies`: Test phase dependency configuration, with the same structure as the `test-dependencies` field
- `bin-dependencies`: Cangjie binary library dependencies, whose structure is described below
- `compile-macros-for-target`: Macro package control items for cross-compilation. This option does not support distinguishing between `debug` and `release` compilation modes mentioned below.

Developers can configure the `target.target-name.debug` and `target.target-name.release` fields to specify additional configurations unique to the `debug` and `release` compilation modes for this `target`. The configurable items are the same as above. Items configured in these fields will only apply to the corresponding compilation mode of the `target`.

#### "target.target-name[.debug/release].bin-dependencies"

This field is used to import precompiled Cangjie library artifacts suitable for the specified `target`. The following example demonstrates importing three packages from the `pro0` and `pro1` modules.

> **Note:**
>
> Unless there are special requirements, it is not recommended to use this field. Instead, use the `dependencies` field described earlier to import module source code.

```text
├── test
│    ├── pro0
│    │    ├── libpro0.xoo.so
│    │    ├── pro0.xoo.cjo
│    │    ├── libpro0.yoo.so
│    │    └── pro0.yoo.cjo
│    └── pro1
│         ├── libpro1.zoo.so
│         └── pro1.zoo.cjo
├── src
│    └── main.cj
└── cjpm.toml
```

Method 1: Import via `path-option`:

```text
[target.x86_64-unknown-linux-gnu.bin-dependencies]
  path-option = ["./test/pro0", "./test/pro1"]
```

The `path-option` is a string array structure, where each element represents the path name to be imported. `cjpm` will automatically import all Cangjie library packages under this path that comply with the rules. Compliance here refers to the library name format being the `full package name`. For example, in the above example, `pro0.xoo.cjo` corresponds to a library name of `libpro0.xoo.so` or `libpro0.xoo.a`. Packages whose library names do not meet this rule can only be imported via the `package-option`.

Method 2: Import via `package-option`:

```text
[target.x86_64-unknown-linux-gnu.bin-dependencies.package-option]
  "pro0.xoo" = "./test/pro0/pro0.xoo.cjo"
  "pro0.yoo" = "./test/pro0/pro0.yoo.cjo"
  "pro1.zoo" = "./test/pro1/pro1.zoo.cjo"
```

The `package-option` is a `map` structure, where `pro0.xoo` serves as the `key` (in TOML configuration files, strings containing `.` must be enclosed in `""` when used as a whole). The `key` value corresponds to `libpro0.xoo.so`. The path to the frontend file `cjo` serves as the `value`, and the corresponding `.a` or `.so` file for this `cjo` must be placed in the same path.

> **Note:**
>
> If the same package is imported via both `package-option` and `path-option`, the `package-option` field takes precedence.

The following example shows how the source code `main.cj` calls the `pro0.xoo`, `pro0.yoo`, and `pro1.zoo` packages.

```cangjie
import pro0.xoo.*
import pro0.yoo.*
import pro1.zoo.*

main(): Int64 {
    var res = x + y + z // x, y, z are values defined in pro0.xoo, pro0.yoo, and pro1.zoo, respectively
    println(res)
    return 0
}
```

> **Note:**
>
> The dependent Cangjie dynamic library files may be compilation artifacts of the `root` package generated by other modules through the `profile.build.combined` configuration, containing symbols for all its sub-packages. Therefore, during dependency checking, if a package's corresponding Cangjie library is not found, the `root` package corresponding to that package will be used as a dependency, and a warning will be printed. Developers must ensure that the `root` package imported in this way is generated via the corresponding method; otherwise, the library file may not contain the symbols of the sub-packages, leading to compilation errors.
> For example, if the source code imports the `demo.aoo` package via `import demo.aoo`, and the corresponding Cangjie library for this package is not found in the binary dependencies, `cjpm` will attempt to find the dynamic library of the `root` package corresponding to this package, i.e., `libdemo.so`. If found, it will use this library as the dependency.

#### "target.target-name.compile-macros-for-target"

This field is used to configure the cross-compilation method for macro packages. There are three scenarios:

Method 1: By default, macro packages are compiled only for the local platform during cross-compilation, not for the target platform. This applies to all macro packages in the module.

```text
[target.target-platform]
  compile-macros-for-target = ""
```

Method 2: During cross-compilation, compile for both the local and target platforms. This applies to all macro packages in the module.

```text
[target.target-platform]
  compile-macros-for-target = "all" # The configuration item is a string, and the optional value must be "all".
```

Method 3: Specify that certain macro packages in the module should be compiled for both the local and target platforms during cross-compilation, while other unspecified macro packages follow the default mode of Method 1.

```text
[target.target-platform]
  compile-macros-for-target = ["pkg1", "pkg2"] # The configuration item is a string array, and the optional values are macro package names.
```

#### "target" Field Merging Rules

Configuration items in `target` may also exist in other options of `cjpm.toml`. For example, the `compile-option` field can also exist in the `package` field, with the difference that the field in `package` applies to all `targets`. `cjpm` merges these duplicate fields in a specific way, combining all applicable configurations. Taking the `debug` compilation mode of `x86_64-unknown-linux-gnu` as an example, the `target` configuration is as follows:

```text
[package]
  compile-option = "compile-0"
  override-compile-option = "override-compile-0"
  link-option = "link-0"

[dependencies]
  dep0 = { path = "./dep0" }

[test-dependencies]
  devDep0 = { path = "./devDep0" }

[target.x86_64-unknown-linux-gnu]
  compile-option = "compile-1"
  override-compile-option = "override-compile-1"
  link-option = "link-1"
  [target.x86_64-unknown-linux-gnu.dependencies]
    dep1 = { path = "./dep1" }
  [target.x86_64-unknown-linux-gnu.test-dependencies]
    devDep1 = { path = "./devDep1" }
  [target.x86_64-unknown-linux-gnu.bin-dependencies]
    path-option = ["./test/pro1"]
  [target.x86_64-unknown-linux-gnu.bin-dependencies.package-option]
    "pro1.xoo" = "./test/pro1/pro1.xoo.cjo"

[target.x86_64-unknown-linux-gnu.debug]
  compile-option = "compile-2"
  override-compile-option = "override-compile-2"
  link-option = "link-2"
  [target.x86_64-unknown-linux-gnu.debug.dependencies]
    dep2 = { path = "./dep2" }
  [target.x86_64-unknown-linux-gnu.debug.test-dependencies]
    devDep2 = { path = "./devDep2" }
  [target.x86_64-unknown-linux-gnu.debug.bin-dependencies]
    path-option = ["./test/pro2"]
  [target.x86_64-unknown-linux-gnu.debug.bin-dependencies.package-option]
    "pro2.xoo" = "./test/pro2/pro2.xoo.cjo"
```

When `target` configuration items coexist with public configuration items in `cjpm.toml` or other levels of configuration items for the same `target`, they are merged according to the following priority:

1. Configuration for the corresponding `target` in `debug/release` mode
2. Configuration for the corresponding `target` regardless of `debug/release` mode
3. Public configuration items

Taking the above `target` configuration as an example, the `target` configuration items are merged according to the following rules:

- `compile-option`: All applicable configurations with the same name are concatenated in order of priority, with higher-priority configurations appended later. In this example, in the `debug` compilation mode for `x86_64-unknown-linux-gnu`, the final `compile-option` value is `compile-0 compile-1 compile-2`; in `release` mode, it is `compile-0 compile-1`; and for other `targets`, it is `compile-0`.
- `override-compile-option`: Same as above. Since `override-compile-option` has higher priority than `compile-option`, in the final compilation command, the concatenated `override-compile-option` will be placed after the concatenated `compile-option`.
- `link-option`: Same as above.
- `dependencies`: Source code dependencies are merged directly. If there are dependency conflicts, an error will be reported. In this example, in the `debug` compilation mode for `x86_64-unknown-linux-gnu`, the final `dependencies` are `dep0`, `dep1`, and `dep2`; in `release` mode, only `dep0` and `dep1` are active; and for other `targets`, only `dep0` is active.
- `test-dependencies`: Same as above.
- `bin-dependencies`: Binary dependencies are merged by priority. If there are conflicts, only the higher-priority dependencies are included. For configurations with the same priority, `package-option` configurations are added first. In this example, in the `debug` compilation mode for `x86_64-unknown-linux-gnu`, binary dependencies from `./test/pro1` and `./test/pro2` are included; in `release` mode, only `./test/pro1` is included. Since there are no public configurations for `bin-dependencies`, no binary dependencies are active for other `targets`.

In this example's cross-compilation scenario, if `x86_64-unknown-linux-gnu` is specified as the target `target` on other platforms, the configuration for `target.x86_64-unknown-linux-gnu` will also be merged with public configuration items according to the above rules and applied. If in `debug` compilation mode, the configuration items for `target.x86_64-unknown-linux-gnu.debug` will also be applied.

### Environment Variable Configuration

In `cjpm.toml`, environment variables can be used to configure field values. `cjpm` will retrieve the corresponding environment variable values from the current runtime environment and substitute them into the actual configuration values. For example, the following `dependencies` field uses an environment variable for path configuration:

```text
[dependencies]
aoo = { path = "${DEPENDENCY_PATH}/aoo" }
```

When reading module `aoo`, `cjpm` will retrieve the `DEPENDENCY_PATH` variable value and substitute it to obtain the final path for module `aoo`.

The list of fields that support environment variable configuration is as follows:

- The following fields in the single-module configuration field `package`:
    - Single-package configuration item `package-configuration`'s single-package compilation option `compile-option`
- The following fields in the workspace management field `workspace`:
    - Member module list `members`
    - Compilation module list `build-members`
    - Test module list `test-members`
- The following fields common to both `package` and `workspace`:
    - Compilation option `compile-option`
    - Global compilation option `override-compile-option`
    - Link option `link-option`
    - Compilation artifact output path `target-dir`
- The `path` field for local dependencies in the build dependency list `dependencies`
- The `path` field for local dependencies in the test dependency list `test-dependencies`
- The `path` field for local dependencies in the build script dependency list `script-dependencies`
- Custom passthrough option `customized-option` in the command profile configuration item `profile`
- The `path` field in the external C library configuration item `ffi.c`
- The following fields in the platform isolation option `target`:
    - Compilation option `compile-option`
    - Global compilation option `override-compile-option`
    - Link option `link-option`
    - The `path` field for local dependencies in the build dependency list `dependencies`
    - The `path` field for local dependencies in the test dependency list `test-dependencies`
    - The `path` field for local dependencies in the build script dependency list `script-dependencies`
    - The `path-option` and `package-option` fields in the binary dependency field `bin-dependencies`

## Configuration and Cache Directories

The storage path for files downloaded by `cjpm` via `git` can be specified using the `CJPM_CONFIG` environment variable. If not specified, the default location is `$HOME/.cjpm` on Linux/macOS and `%USERPROFILE%/.cjpm` on Windows.

## Cangjie Package Management Specification

In the Cangjie package management specification, for a file directory to be recognized as a valid source code package, the following requirements must be met:

1. It must directly contain at least one Cangjie code file.
2. Its parent package (including the parent's parent, up to the `root` package) must also be a valid source code package. The module `root` package has no parent, so it only needs to meet condition 1.

For example, consider the following `cjpm` project named `demo`:

```text
demo
├──src
│   ├── main.cj
│   └── pkg0
│        ├── aoo
│        │    └── aoo.cj
│        └── boo
│             └── boo.cj
└── cjpm.toml
```

Here, the `demo.pkg0` directory does not directly contain any Cangjie code, so `demo.pkg0` is not a valid source code package. Although `demo.pkg0.aoo` and `demo.pkg0.boo` directly contain Cangjie code files `aoo.cj` and `boo.cj`, their upstream package `demo.pkg0` is not a valid source code package, so these two packages are also not valid source code packages.

When `cjpm` detects a package like `demo.pkg0` that does not directly contain Cangjie files, it will treat it as a non-source package, ignore all its sub-packages, and print the following warning:

```text
Warning: there is no '.cj' file in directory 'demo/src/pkg0', and its subdirectories will not be scanned as source code
```

Therefore, if developers need to configure a valid source code package, the package must directly contain at least one Cangjie code file, and all its upstream packages must also be valid source code packages. In the above `demo` project, to ensure `demo.pkg0`, `demo.pkg0.aoo`, and `demo.pkg0.boo` are all recognized as valid source code packages, a Cangjie code file can be added to `demo/src/pkg0`, as shown below:

```text
demo
├── src
│    ├── main.cj
│    └── pkg0
│         ├── pkg0.cj
│         ├── aoo
│         │    └── aoo.cj
│         └── boo
│              └── boo.cj
└── cjpm.toml
```

`demo/src/pkg0/pkg0.cj` must be a Cangjie code file that complies with the package management specification. It may not contain functional code, such as the following:

```cangjie
package demo.pkg0
```

## Command Extension

`cjpm` provides a command extension mechanism. Developers can extend `cjpm` commands by creating executable files named `cjpm-xxx(.exe)`.

For an executable file `cjpm-xxx` (`cjpm-xxx.exe` on Windows), if the directory containing this file is configured in the system environment variable `PATH`, the following command can be used to run the executable:

```shell
cjpm xxx [args]
```

Here, `args` represents the list of arguments that may be passed to `cjpm-xxx(.exe)`. The above command is equivalent to:

```shell
cjpm-xxx(.exe) [args]
```

Running `cjpm-xxx(.exe)` may depend on certain dynamic libraries. In such cases, developers must manually add the directories containing the required dynamic libraries to the environment variables.

The following example uses `cjpm-demo`, an executable compiled from the following Cangjie code:

```cangjie
import std.process.*
import std.collection.*

main(): Int64 {
    var args = ArrayList<String>(Process.current.arguments)

    if (args.size < 1) {
        eprintln("Error: failed to get parameters")
        return 1
    }

    println("Output: ${args[0]}")

    return 0
}
```

After adding its directory to `The internal commands of `cjpm` have higher priority, so these commands cannot be extended in this way. For example, even if an executable named `cjpm-build` exists in the system environment variables, `cjpm build` will not run that file but will instead run `cjpm` and pass `build` as an argument to `cjpm`.

## Build Script

`cjpm` provides a build script mechanism that allows developers to define behaviors for `cjpm` before or after executing specific commands.

The build script source file must be named `build.cj` and located in the root directory of the Cangjie project, at the same level as `cjpm.toml`. When creating a new Cangjie project using the `init` command, `cjpm` does not create `build.cj` by default. If developers need it, they can manually create and edit `build.cj` in the specified location according to the following template format.

```cangjie
// build.cj

import std.process.*

// Case of pre/post codes for 'cjpm build'.
/* called before `cjpm build`
 * Success: return 0
 * Error: return any number except 0
 */
// func stagePreBuild(): Int64 {
//     // process before "cjpm build"
//     0
// }

/*
 * called after `cjpm build`
 */
// func stagePostBuild(): Int64 {
//     // process after "cjpm build"
//     0
// }

// Case of pre codes for 'cjpm clean'.
/* called before `cjpm clean`
 * Success: return 0
 * Error: return any number except 0
 */
// func stagePreClean(): Int64 {
//     // process before "cjpm clean"
//     0
// }

// For other options, define stagePreXXX and stagePostXXX in the same way.

/*
 * Error code:
 * 0: success.
 * other: cjpm will finish running command. Check target-dir/build-script-cache/module-name/script-log for error outputs defind by user in functions.
 */

main(): Int64 {
    match (Process.current.arguments[0]) {
        // Add operation here with format: "pre-"/"post-" + optionName
        // case "pre-build" => stagePreBuild()
        // case "post-build" => stagePostBuild()
        // case "pre-clean" => stagePreClean()
        case _ => 0
    }
}
```

`cjpm` supports defining pre- and post-command behaviors for a series of commands using build scripts. For example, for the `build` command, you can define `pre-build` in the `match` block within the `main` function to execute the desired functionality function `stagePreBuild` (the naming of the function is not restricted). The post-`build` behavior can be defined similarly by adding a `post-build` `case` option. The definition of pre- and post-command behaviors for other commands follows the same pattern—just add the corresponding `pre/post` options and their associated functionality functions.

After defining pre- and post-command behaviors, `cjpm` will first compile `build.cj` when executing the command and then execute the corresponding behaviors before and after the command. Taking `build` as an example, if `pre-build` and `post-build` are defined and `cjpm build` is run, the entire `cjpm build` process will follow these steps:

1. Before starting the build process, first compile `build.cj`;
2. Execute the functionality function corresponding to `pre-build`;
3. Proceed with the `cjpm build` compilation process;
4. After the compilation process completes successfully, `cjpm` will execute the functionality function corresponding to `post-build`.

The commands supported by the build script are as follows:

- `build`, `test`, `bench`: Support executing both `pre` and `post` processes defined in the build scripts of dependent modules.
- `run`, `install`: Only support running the `pre` and `post` build script processes of the corresponding module or executing the `pre-build` and `post-build` processes of dependent modules during compilation.
- `check`, `tree`, `update`: Only support running the `pre` and `post` build script processes of the corresponding module.
- `clean`: Only support running the `pre` build script process of the corresponding module.

When executing these commands, if the `--skip-script` option is configured, all build script compilation and execution will be skipped, including those of dependent modules.

Usage notes for the build script:

- The return value of functionality functions must meet specific requirements: when the function executes successfully, it must return `0`; if it fails, it must return any `Int64` value other than `0`.
- All outputs from `build.cj` will be redirected to the project directory at `build-script-cache/[target|release]/[module-name]/bin/script-log`. Developers can check this file for any output content added in the functionality functions.
- If `build.cj` does not exist in the project root directory, `cjpm` will execute the normal workflow. If `build.cj` exists and defines pre- or post-command behaviors, the command will abort abnormally if `build.cj` fails to compile or the functionality function returns a non-zero value, even if the command itself could execute successfully.
- In multi-module scenarios, the `build.cj` build scripts of dependent modules will take effect during the compilation and unit testing processes. Outputs from dependent module build scripts are also redirected to log files under `build-script-cache/[target|release]` in the corresponding module directory.

For example, the following build script `build.cj` defines pre- and post-`build` behaviors:

```cangjie
import std.process.*

func stagePreBuild(): Int64 {
    println("PRE-BUILD")
    0
}

func stagePostBuild(): Int64 {
    println("POST-BUILD")
    0
}

main(): Int64 {
    match (Process.current.arguments[0]) {
        case "pre-build" => stagePreBuild()
        case "post-build" => stagePostBuild()
        case _ => 0
    }
}
```

When executing the `cjpm build` command, `cjpm` will execute `stagePreBuild` and `stagePostBuild`. After `cjpm build` completes, the `script-log` file will contain the following output:

```text
PRE-BUILD
POST-BUILD
```

Build scripts can import dependency modules via the `script-dependencies` field in `cjpm.toml`, with the same format as `dependencies`. For example, the following configuration in `cjpm.toml` imports the `aoo` module, which contains a method named `aaa()`:

```text
[script-dependencies]
aoo = { path = "./aoo" }
```

You can then import this dependency in the build script and use the interface `aaa()`:

```cangjie
import std.process.*
import aoo.*

func stagePreBuild(): Int64 {
    aaa()
    0
}

func stagePostBuild(): Int64 {
    println("POST-BUILD")
    0
}

main(): Int64 {
    match (Process.current.arguments[0]) {
        case "pre-build" => stagePreBuild()
        case "post-build" => stagePostBuild()
        case _ => 0
    }
}
```

Build script dependencies (`script-dependencies`) are independent of source code-related dependencies (`dependencies` and `test-dependencies`). Source and test code cannot use modules from `script-dependencies`, and build scripts cannot use modules from `dependencies` or `test-dependencies`. If the same module is needed in both build scripts and source/test code, it must be configured in both `script-dependencies` and `dependencies/test-dependencies`.

## Usage Example

The following example demonstrates the usage of `cjpm` using the directory structure of a Cangjie project. The corresponding source code files for this example can be found in [Source Code](#example-source-code). The module name of this Cangjie project is `test`.

```text
cj_project
├── pro0
│    ├── cjpm.toml
│    └── src
│         ├── zoo
│         │    ├── zoo.cj
│         │    └── zoo_test.cj
│         └── pro0.cj
├── src
│    ├── koo
│    │    ├── koo.cj
│    │    └── koo_test.cj
│    ├── main.cj
│    └── main_test.cj
└── cjpm.toml
```

### Usage of `init` and `build`

- Create a new Cangjie project and write source code files (`xxx.cj`), such as the `koo` package and `main.cj` file shown in the example structure.

    ```shell
    cjpm init --name test --path .../cj_project
    mkdir koo
    ```

    This will automatically generate the `src` folder and the default `cjpm.toml` configuration file.

- When the current module depends on an external `pro0` module, create the `pro0` module and its configuration file. Then, write the source code files for this module. Manually create the `src` folder under `pro0`, and within `src`, create the root package `pro0.cj` for `pro0`. Place the written Cangjie packages under `src`, such as the `zoo` package in the example structure.

    ```shell
    mkdir pro0 && cd pro0
    cjpm init --name pro0 --type=static
    mkdir src/zoo
    ```

- When the main module depends on `pro0`, configure the `dependencies` field in the main module's configuration file as described in the manual. After ensuring the configuration is correct, execute `cjpm build`. The generated executable will be in the `target/release/bin/` directory.

    ```shell
    cd cj_project
    vim cjpm.toml
    cjpm build
    cjpm run
    ```

### Usage of `test` and `clean`

- After writing the corresponding `xxx_test.cj` unit test files for each file as shown in the example structure, execute the following code to run unit tests. The generated files will be in the `target/release/unittest_bin` directory.

    ```shell
    cjpm test
    ```

    Or:

    ```shell
    cjpm test src src/koo pro/src/zoo
    ```

- To manually delete intermediate files like `target`, execute the following command:

    ```shell
    cjpm clean
    ```

### Example Source Code

```cangjie
// cj_project/src/main.cj
package test

import pro0.zoo.*
import test.koo.*

main(): Int64 {
    let res = z + k
    println(res)
    let res2 = concatM("a", "b")
    println(res2)
    return 0
}

func concatM(s1: String, s2: String): String {
    return s1 + s2
}
```

```cangjie
// cj_project/src/main_test.cj
package test

import std.unittest.* // testfame
import std.unittest.testmacro.* // macro_Defintion

@Test
public class TestM{
    @TestCase
    func sayhi(): Unit {
        @Assert(concatM("1", "2"), "12")
        @Assert(concatM("1", "3"), "13")
    }
}
```

```cangjie
// cj_project/src/koo/koo.cj
package test.koo

public let k: Int32 = 12

func concatk(s1: String, s2: String): String {
    return s1 + s2
}
```

```cangjie
// cj_project/src/koo/koo_test.cj
package test.koo

import std.unittest.* // testfame
import std.unittest.testmacro.* // macro_Defintion

@Test
public class TestK{
    @TestCase
    func sayhi(): Unit {
        @Assert(concatk("1", "2"), "12")
        @Assert(concatk("1", "3"), "13")
    }
}
```

```cangjie
// cj_project/pro0/src/pro0.cj
package pro0
```

```cangjie
// cj_project/pro0/src/zoo/zoo.cj
package pro0.zoo

public let z: Int32 = 26

func concatZ(s1: String, s2: String): String {
    return s1 + s2
}```markdown
```

```cangjie
// cj_project/pro0/src/zoo/zoo_test.cj
package pro0.zoo

import std.unittest.* // test framework
import std.unittest.testmacro.* // macro definition

@Test
public class TestZ{
    @TestCase
    func sayhi(): Unit {
        @Assert(concatZ("1", "2"), "12")
        @Assert(concatZ("1", "3"), "13")
    }
}
```

```toml
# cj_project/cjpm.toml
[package]
cjc-version = "1.0.0"
description = "nothing here"
version = "1.0.0"
name = "test"
output-type = "executable"

[dependencies]
pro0 = { path = "pro0" }
```

```toml
# cj_project/pro0/cjpm.toml
[package]
cjc-version = "1.0.0"
description = "nothing here"
version = "1.0.0"
name = "pro0"
output-type = "static"
```
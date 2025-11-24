# HLE Tool User Guide

## Introduction to Open Source Project

`HLE (HyperlangExtension)` is a tool for automatically generating interoperability code templates for Cangjie calling ArkTS or C language.

The input of this tool is the interface declaration file of ArkTS or C language, such as files ending with .d.ts, .d.ets, or .h, and the output is a cj file, which stores the generated interoperability code. If the generated code is a glue layer code from ArkTS to Cangjie, the tool will also output a json file containing all the information of the ArkTS file. For the conversion rules from ArkTS to Cangjie, please refer to: [ArkTS Third-Party Module Generation Cangjie Glue Code Rules](cj-dts2cj-translation-rules.md). For the conversion rules from C language to Cangjie, please refer to: [C Language Conversion to Cangjie Glue Code Rules](cj-c2cj-translation-rules.md).

## Parameter Meaning
| Parameter            | Meaning                                        | Parameter Type | Description                 |
| --------------- | ------------------------------------------- | -------- | -------------------- |
| `-i`            | Absolute path of d.tsj, d.ets or .h file input             | Optional parameter | Choose one of `-d` or both exist                     |
| `-r`            | Absolute path of typescript compiler                  | Required parameter |                      |
| `-d`            | Absolute path of the folder where d.ts, d.ets or .h file input is located    | Optional parameter | Choose one of `-i` or both exist                     |
| `-o`            | Directory to save the output interoperability code                    | Optional parameter | Output to the current directory by default |
| `-j`            | Path to analyze d.t or d.ets files                 | Optional parameter |                      |
| `--module-name` | Custom generated Cangjie package name                        | Optional parameter |                      |
| `--lib`         | Generate third-party library code                              | Optional parameter |                      |
| `-c`            | Generate C to Cangjie binding code                              | Optional parameter |                      |
| `-b`            | Specify the directory of cjbind binary                              | Optional parameter |                      |
| `--clang-args`  | Parameters that will be directly passed to clang                              | Optional parameter |                      |
| `--no-detect-include-path`  | Disable automatic include path detection                              | Optional parameter |                      |
| `--help`        | Help option                                   | Optional parameter |                      |

For example:

You can use the following command to generate interface glue layer code:

```sh
main  -i  /path/to/test.d.ts  -o  out  –j  /path/to/analysis.js --module-name=ohos.hilog
```

In the Windows environment, the file directory currently does not support the symbol "\\", only "/" is supported.

```sh
main -i  /path/to/test.d.ts -o out -j /path/to/analysis.js --module-name=ohos.hilog
```

You can use the following command to generate C to Cangjie binding code:

```sh
./target/bin/main -c --module-name="my_module" -d ./tests/c_cases -o ./tests/expected/c_module/ --clang-args="-I/usr/lib/llvm-20/lib/clang/20/include/"
```
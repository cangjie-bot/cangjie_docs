# HLE Tool User Guide

## Introduction to Open Source Project

`HLE (HyperlangExtension)` is a tool for automatically generating interoperability code templates for Cangjie calling ArkTS or C language.

The input of this tool is the interface declaration file of ArkTS or C language, such as files ending with .d.ts, .d.ets, or .h, and the output is a cj file, which stores the generated interoperability code. If the generated code is a glue layer code from ArkTS to Cangjie, the tool will also output a json file containing all the information of the ArkTS file. For the conversion rules from ArkTS to Cangjie, please refer to: [ArkTS Third-Party Module Generation Cangjie Glue Code Rules](cj-dts2cj-translation-rules.md). For the conversion rules from C language to Cangjie, please refer to: [C Language Conversion to Cangjie Glue Code Rules](cj-c2cj-translation-rules.md).

## Parameter Meaninog

| Parameter | Meaning | Parameter Type | Description |
| --------------- | ------------------------------------------- | -------- | -------------------- |
| `-i` | Absolute path of the input d.ts, d.ets, or .h file | Optional | Either this or `-d` can be used, or both may exist |
| `-r` | Absolute path of the TypeScript compiler | Required | Used only when generating ArkTS Cangjie bindings |
| `-d` | Absolute path of the folder containing d.ts, d.ets, or .h files | Optional | Either this or `-i` can be used, or both may exist |
| `-o` | Directory to save the generated interop code | Optional | Defaults to the current directory if not specified |
| `-j` | Path for analyzing d.t or d.ets files | Optional | Used only when generating ArkTS Cangjie bindings |
| `--module-name` | Custom generated Cangjie package name | Optional | |
| `--lib` | Generate third-party library code | Optional | Used only when generating ArkTS Cangjie bindings |
| `-c` | Generate C to Cangjie binding code | Optional | Used only when generating C language Cangjie bindings |
| `-b` | Specify the directory of the cjbind binary | Optional | Used only when generating C language Cangjie bindings |
| `--clang-args` | Arguments passed directly to clang | Optional | Used only when generating C language Cangjie bindings |
| `--no-detect-include-path` | Disable automatic include path detection | Optional | Used only when generating C language Cangjie bindings |
| `--help` | Help option | Optional | |

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
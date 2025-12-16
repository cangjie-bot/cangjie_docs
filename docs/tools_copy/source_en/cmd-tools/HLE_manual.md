# HLE Tool User Guide

## Open Source Project Introduction

`HLE` (Hyper-Lang extension) is an automatic code template generation tool for Cangjie-ARKTS interoperation.

This tool takes ARKTS interface declaration files (such as files with .d.ts or .d.ets extensions) as input, and outputs a directory containing BUILD.gn files and an src folder. The src folder contains .cj files storing the generated interoperation code. The tool also outputs a JSON file containing all information from the ARKTS files.

## Parameter Descriptions

| Parameter       | Description                                     | Type      | Notes                |
| --------------- | ---------------------------------------------- | --------- | -------------------- |
| `-i`            | Absolute path of input .d.ts or .d.ets file    | Optional  | Mutually exclusive or combinable with `-d` parameter |
| `-r`            | Absolute path of TypeScript compiler           | Optional  | -                    |
| `-d`            | Absolute path of directory containing input .d.ts or .d.ets files | Optional  | Mutually exclusive or combinable with `-i` parameter |
| `-o`            | Output directory for interoperation code        | Optional  | Defaults to current directory |
| `-j`            | Path for analyzing .d.ts or .d.ets files       | Optional  | -                    |
| `--module-name` | Custom Cangjie package name                    | Optional  | -                    |
| `--lib`         | Generate third-party library code              | Optional  | -                    |
| `--help`        | Help option                                    | Optional  | -                    |

For example:

The following command can be used to generate interface glue layer code:

```sh
main -i /path/to/test.d.ts -o out -j /path/to/analysis.js --module-name=ohos.hilog
```

On Windows environments, the file directory does not currently support the "\\" symbol, only "/" can be used.

```sh
main -i /path/to/test.d.ts -o out -j /path/to/analysis.js --module-name=ohos.hilog
```
# HLE 工具用户指南

## 开源项目介绍

`HLE`(Hyper-Lang extension)是一个仓颉调用 ArkTS 互操作代码模板自动生成工具。

该工具的输入是 ArkTS 接口声明文件，例如后缀 .d.ts 或者 .d.ets 结尾的文件，输出为包含 BUILD.gn 文件和 src 文件夹。src 文件夹中包含的 cj 文件中存放生成的互操作代码。工具也会输出包含 ArkTS 文件的所有信息的 json 文件。

## 参数含义

| 参数            | 含义                                        | 参数类型 | 说明                 |
| --------------- | ------------------------------------------- | -------- | -------------------- |
| `-i`            | d.ts 或者 d.ets 文件输入的绝对路径             | 可选参数 | 和`-d`参数二选一或者两者同时存在                     |
| `-r`            | typescript 编译器的绝对路径                  | 可选参数 |-                      |
| `-d`            | d.ts 或者 d.ets 文件输入所在文件夹的绝对路径    | 可选参数 | 和`-i`参数二选一或者两者同时存在                     |
| `-o`            | 输出保存互操作代码的目录                    | 可选参数 | 缺省时输出至当前目录 |
| `-j`            | 分析 d.t 或者 d.ets 文件的路径                 | 可选参数 |   -                   |
| `--module-name` | 自定义生成的仓颉包名                        | 可选参数 |    -                  |
| `--lib`         | 生成三方库代码                              | 可选参数 |  -                    |
| `--help`        | 帮助选项                                   | 可选参数 | -                     |

例如：

可使用如下的命令生成接口胶水层代码：

```sh
main  -i  /path/to/test.d.ts  -o  out  –j  /path/to/analysis.js --module-name=ohos.hilog
```

在 Windows 环境，文件目录当前不支持符号“\\”，仅支持使用“/”。

```sh
main -i  /path/to/test.d.ts -o out -j /path/to/analysis.js --module-name=ohos.hilog
```

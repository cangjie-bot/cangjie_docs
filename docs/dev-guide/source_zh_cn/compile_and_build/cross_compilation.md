# 交叉编译

开发者编码的仓颉程序通过交叉编译将其运行于不同体系架构平台上，仓颉支持交叉编译的场景：

| 编译平台       | 目标平台       | 常用场景/工具                | 对应 SDK 安装包                        |
|-------------------------|-----------------------|-------------------------------|-------------------------------------------------|
| Windows (x64)           | OpenHarmony/HarmonyOS | OpenHarmony/HarmonyOS 设备或模拟器 | 请至鸿蒙开发者官网进一步了解。 |
| macOS (aarch64)     | OpenHarmony/HarmonyOS | OpenHarmony/HarmonyOS 设备或模拟器 | 请至鸿蒙开发者官网进一步了解。 |
| Linux (x64)             | OpenHarmony/HarmonyOS | OpenHarmony/HarmonyOS 设备或模拟器  | 请至鸿蒙开发者官网进一步了解。 |
|<!--DelRow--> Windows (x64)           | Android (aarch64)     | Android 真机 | cangjie-sdk-windows-x64-android.x.y.z.zip (或 .exe) |
|<!--DelRow--> Linux (x64)             | Android (aarch64)     | Android 真机  | cangjie-sdk-linux-x64-android.x.y.z.tar.gz |
|<!--DelRow--> macOS (aarch64)     | Android (aarch64)     | Android 真机以及 Android Studio 模拟器 | cangjie-sdk-mac-aarch64-android.x.y.z.tar.gz |
|<!--DelRow--> macOS (aarch64)     | Android (arm32)       | Android 真机 | cangjie-sdk-mac-aarch64-android-arm32-x.y.z.tar.gz |
|<!--DelRow--> macOS (aarch64)         | iOS (aarch64)         | iOS 真机 | cangjie-sdk-mac-aarch64-ios.x.y.z.tar.gz |

<!--Del-->仓颉编程语言现已支持交叉编译至 `Android`（`aarch64` 默认目标为 `API 23+`，`arm32` 默认目标为 `API 23+`）和 `iOS`，方便开发者在不同平台上进行应用开发。<!--DelEnd-->

## 支持的编译目标和目标三元组名称

目标三元组名称是通用的用于指定编译平台或目标的名称，通常由架构、供应商、系统三部分组成，供应商在不具备二义性的情况下通常可以省略。当前仓颉编译器支持以下编译目标，可以通过使用 `cjc` 的 `--target` 选项指定特定的三元组名称来为指定平台生成二进制程序（需要使用支持的 SDK 安装包）。

| 编译目标名称 | 其他别名 | 编译目标说明 |
|--|--|--|
| x86_64-linux-gnu | x86_64-unknown-linux-gnu | x86_64 架构 Linux 平台 |
| aarch64-linux-gnu | aarch64-unknown-linux-gnu, arm64-unknown-linux-gnu, arm64-linux-gnu | aarch64 架构 Linux 平台 |
| x86_64-windows-gnu | x86_64-pc-windows-gnu, x86_64-unknown-windows-gnu, x86_64-pc-w64-mingw32, x86_64-w64-mingw32 | x86_64 架构 Windows 平台 |
| aarch64-apple-darwin | arm64-apple-darwin | arm64 架构 macOS 平台 |
| x86_64-apple-darwin | - | x86_64 架构 macOS 平台 |
| x86_64-linux-ohos | x86_64-unknown-linux-ohos | x86_64 架构 OpenHarmony/HarmonyOS 平台 |
| aarch64-linux-ohos | aarch64-unknown-linux-ohos, arm64-unknown-linux-ohos, arm64-linux-ohos | aarch64 架构 OpenHarmony/HarmonyOS 平台 |
|<!--DelRow--> aarch64-linux-android\<integer> | aarch64-unknown-linux-android\<integer>, arm64-unknown-linux-android\<integer>, arm64-linux-android\<integer> | aarch64 架构 Android 平台 |
|<!--DelRow--> aarch64-apple-ios\<number> | arm64-apple-ios\<number> | arm64 架构 iOS 平台 |
|<!--DelRow--> aarch64-apple-ios\<number>-simulator | arm64-apple-ios\<number>-simulator | arm64 架构 iOS 模拟器平台 |
|<!--DelRow--> x86_64-apple-ios\<number>-simulator | - | x86_64 架构 iOS 模拟器平台 |

<!--Del-->
> **说明：**
>
> \<integer> 是一个描述 Android API Level 的正整数，该数字不同的目标三元组名称不等价。
>
> \<number> 是一个描述 iOS 版本号的正数，例如 18 或者 18.0。该数字不同的目标三元组名称不等价。
<!--DelEnd-->

<!--Del-->
## 仓颉交叉编译至 Android

### 安装包下载

开发者可以使用支持交叉编译的仓颉安装包对特定平台（`android-aarch64`、`android-arm32`）进行交叉编译。`Android aarch64` 和 `Android arm32` 需使用不同的 SDK 安装包。

**支持交叉编译至 Android 的仓颉安装包如下（具体版本号以实际发布包为准）：**

- Android `aarch64`：
  `cangjie-sdk-linux-x64-android.x.y.z.tar.gz`、`cangjie-sdk-windows-x64-android.x.y.z.zip`、`cangjie-sdk-windows-x64-android.x.y.z.exe`、`cangjie-sdk-mac-aarch64-android.x.y.z.tar.gz`
- Android `arm32`：
  `cangjie-sdk-mac-aarch64-android-arm32-x.y.z.tar.gz`

例如：若需要在 `linux x64` 平台交叉编译至 `Android aarch64`，可以下载安装 `cangjie-sdk-linux-x64-android.x.y.z.tar.gz` 仓颉软件包。

除了支持交叉编译的仓颉软件包，还需要下载 `Android NDK`（建议使用 `ndk-r27d`）。

### 编译

仓颉交叉编译至 `Android` 会需要以下三个依赖目录：

1. `sysroot` 目录，该文件由 `Android NDK` 提供，通常位于 `<ndk-path>/toolchains/llvm/prebuilt/<platform>/sysroot` 。

2. `libclang_rt.builtins-<arch>-android.a` 所在的目录（如 `libclang_rt.builtins-aarch64-android.a` 或 `libclang_rt.builtins-arm-android.a`），该文件由 `Android NDK` 提供，通常位于 `<ndk-path>/toolchains/llvm/prebuilt/<platform>/lib/clang/<version>/lib/linux` 。

3. 工具链二进制目录，该目录由 `Android NDK` 提供，通常位于 `<ndk-path>/toolchains/llvm/prebuilt/<platform>/bin` 。

使用 `cjc` 交叉编译仓颉代码时需要额外指定以下选项（`<>` 部分需要替换为实际目录）：

- `--target=aarch64-linux-android` 默认以 Android API 23 为目标平台进行交叉编译；若需更高版本，可显式追加数字后缀，例如 `--target=aarch64-linux-android31` 则指定 Android API 31
- `--target=aarch64-linux-android23` 显式指定 `aarch64` 目标平台以 Android API 23 进行交叉编译
- `--target=arm-linux-android23` 指定 `arm32` 目标平台（Android API 23）
- `--sysroot=<sysroot-path>` 指定工具链的根目录路径 `<sysroot-path>`
- `-L<lib-path>` 指定 `libclang_rt.builtins-<arch>-android.a` 所在目录 `<lib-path>`
- `-B<toolchain-bin-path>` 指定 `Android NDK` 工具链二进制目录 `<toolchain-bin-path>`

例如，交叉编译至 `aarch64` 目标时可执行：

```shell
$ cjc main.cj --target=aarch64-linux-android31 \
  --sysroot /opt/buildtools/android_ndk-r25b/toolchains/llvm/prebuilt/linux-x86_64/sysroot \
  -L /opt/buildtools/android_ndk-r25b/toolchains/llvm/prebuilt/linux-x86_64/lib/clang/14.0.6/lib/linux
```

- `main.cj` 是交叉编译的仓颉代码，`aarch64-linux-android31` 为目标平台
- `/opt/buildtools/android_ndk-r25b/toolchains/llvm/prebuilt/linux-x86_64/sysroot` 为工具链根目录路径 `<sysroot-path>`
- `/opt/buildtools/android_ndk-r25b/toolchains/llvm/prebuilt/linux-x86_64/lib/clang/14.0.6/lib/linux` 为 `libclang_rt.builtins-aarch64-android.a` 所在目录 `<lib-path>`

交叉编译至 `arm32` 目标时可执行：

```shell
$ cjc test.cj --target=arm-linux-android23 \
  --sysroot /home/rus/opt/android-ndk-r27d/toolchains/llvm/prebuilt/linux-x86_64/sysroot \
  -L /home/rus/opt/android-ndk-r27d/toolchains/llvm/prebuilt/linux-x86_64/lib/clang/18/lib/linux/ \
  -B/home/rus/opt/android-ndk-r27d/toolchains/llvm/prebuilt/linux-x86_64/bin
```

- `test.cj` 是交叉编译的仓颉代码，`arm-linux-android23` 为目标平台
- `/home/rus/opt/android-ndk-r27d/toolchains/llvm/prebuilt/linux-x86_64/sysroot` 为工具链根目录路径 `<sysroot-path>`
- `/home/rus/opt/android-ndk-r27d/toolchains/llvm/prebuilt/linux-x86_64/lib/clang/18/lib/linux/` 为 `libclang_rt.builtins-arm-android.a` 所在目录 `<lib-path>`
- `/home/rus/opt/android-ndk-r27d/toolchains/llvm/prebuilt/linux-x86_64/bin` 为工具链二进制目录 `<toolchain-bin-path>`

> **注意：**
>
> arm32 不支持反射相关功能，不支持栈扩容，不支持 runtime trace 等功能。

### 部署和运行

编译完成后需要将以下文件推送到 `Android` 设备：

- 可执行程序以及其依赖的所有动态库： 例如 `main` 和其依赖的 `.so` 动态库文件
- 仓颉运行时依赖：根据目标平台选择对应目录（例如 `aarch64-linux-android31` 对应 `$CANGJIE_HOME/runtime/lib/linux_android31_aarch64_cjnative/*.so`，`aarch64-linux-android23` 对应 `$CANGJIE_HOME/runtime/lib/linux_android_aarch64_cjnative/*.so`，`arm-linux-android23` 对应 `$CANGJIE_HOME/runtime/lib/linux_android23_arm_cjnative/*.so`，目录命名请以实际 SDK 为准）

通过使用 `Android` 调试桥 `adb` 工具可以将可执行程序以及仓颉库推送至设备，示例如下：

`aarch64` 目标示例：

```shell
$ adb push ./main /data/local/tmp/
$ adb push $CANGJIE_HOME/runtime/lib/linux_android31_aarch64_cjnative/* /data/local/tmp/
```

`arm32` 目标示例：

```shell
$ adb push ./main /data/local/tmp/
$ adb push $CANGJIE_HOME/runtime/lib/linux_android23_arm_cjnative/* /data/local/tmp/
```

`adb` 工具的详细使用方法请参考 `Android` 官网的 `Android` 调试桥（`adb`）相关说明。

`.so` 动态库文件可以直接部署到系统目录，若部署在其他非常规目录，则需要在运行前将目录添加到 `LD_LIBRARY_PATH` 环境变量中。

运行仓颉程序 `main`，示例如下：

```shell
$ adb shell "chmod +x /data/local/tmp/main"
$ adb shell "LD_LIBRARY_PATH=/data/local/tmp /data/local/tmp/main"
```

## 仓颉交叉编译至 iOS

### 安装包下载

支持交叉编译至 `iOS` 的仓颉安装包为：`cangjie-sdk-mac-aarch64-ios.x.y.z.tar.gz`
仓颉运行时以及标准库默认支持运行于 `iOS 11` 及以上系统（对于例外场景，请见《仓颉编程语言库 API》手册）。

除了支持交叉编译的仓颉软件包，还需要下载 `Xcode`。下载完成后，请在 Xcode 中安装 iOS 开发组件。具体步骤可参考 Xcode 手册中的 "Downloading and installing additional Xcode components" 章节。

### 编译

本节以生成静态库为例，介绍交叉编译至 `iOS` 的步骤。交叉编译仓颉代码至 `iOS` 设备时，需要额外指定以下选项：

- `--target=aarch64-apple-ios<version>` 指定目标平台 `ios` 进行交叉编译
- `--output-type=staticlib`指定输出文件的类型为静态库

> **注意：**
>
> 版本号 \<version> 建议参考所安装的 Xcode 支持的 SDK 版本（如 17.5 等）。

当前仓颉支持从 `aarch64` 架构环境交叉编译生成 iOS 模拟器静态库，既适配 `aarch64` 架构模拟器，也可编译至 x86_64 架构模拟器（该架构产物需依赖 Xcode 的 Rosetta 运行）。若需在 iOS 模拟器上运行，需指定以下选项：

- `--target`
    - `--target=aarch64-apple-ios-simulator` 适用于 aarch64 目标架构
    - `--target=x86_64-apple-ios-simulator` 适用于 x86_64 目标架构
- `--output-type=staticlib` 指定输出文件的类型为静态库

编译产物需要添加至 `Xcode` 工程中，并通过 `Xcode` 构建 `iOS` 应用。

编译 `main.cj` 生成 `libmain.a` 静态库，示例如下：

```shell
cjc main.cj --output-type=staticlib --target=aarch64-apple-ios17.5 -o libmain.a
```

`main.cj` 是交叉编译的仓颉代码， `aarch64-apple-ios17.5` 为目标平台，`--output-type=staticlib` 指定输出文件的类型为静态库。

使用 CJPM 进行模块级或工程级 `lto-combine` 构建时，也可以生成 `.dylib` 动态库。需开启 `profile.build.lto`，并将 `profile.build.combined` 中对应模块的值设为 `dynamic`。具体配置和动态库集成注意事项，参见 CJPM 手册的[编译优化与编译产物合一](../../../tools/source_zh_cn/cmd-tools/cjpm_manual.md#编译优化与编译产物合一)。

将上述静态库集成到 `Xcode` 工程时，还需要进行以下配置：

1. 根据需要的运行目标（设备或模拟器）选择一个目录：

    - 适用于设备：`$CANGJIE_HOME/lib/ios_aarch64_cjnative`

    - 适用于模拟器：`$CANGJIE_HOME/lib/ios_simulator_aarch64_cjnative`

   将对应目录下的所有 `.a` 文件添加至 `Xcode` 工程中。

2. 在 `Xcode` 项目中配置 `Build Settings > Other Linker Flags` 字段，设置为以下值：

    - `$CANGJIE_HOME/lib/ios_aarch64_cjnative/section.o`

    - `$CANGJIE_HOME/lib/ios_aarch64_cjnative/cjstart.o`

    - `-lc++`

   值得注意的是，必须按照以上列表顺序添加链接选项。`$CANGJIE_HOME` 需要替换为实际的仓颉安装目录。若运行目标是模拟器，请将 `ios_aarch64_cjnative` 替换为 `ios_simulator_aarch64_cjnative` 。此外，若使用 Xcode 15，还需在 `Build Settings > Other Linker Flags` 中额外添加 `-Wl,-ld_classic` 或 `-Wl,-no_compact_unwind` ，否则在某些场景下，Xcode 15 链接器可能会报错：`Assertion failed: (false && "compact unwind compressed function offset doesn't fit in 24 bits")` 。

3. 在 `Xcode` 项目中配置 `Build Settings > Dead Code Stripping` 字段，设置为 `No`。

配置完毕后通过 `Xcode` 直接构建项目即可。

### 部署和运行

通过 `Xcode` 构建并部署至真机或模拟器运行，具体步骤可参考 `Xcode` 手册的 "Build and running an app" 章节。
<!--DelEnd-->

# Runtime Environment Variables User Manual

This section introduces the environment variables provided by the `runtime` (runtime).

In Linux shell and macOS shell, you can set the environment variables provided by the Cangjie runtime using the following method:

```shell
$ export VARIABLE=value
```

In Windows cmd, you can set the environment variables provided by the Cangjie runtime using the following method:

```shell
> set VARIABLE=value
```

The subsequent examples in this section use the setting method in Linux shell. If it does not match the running platform, please select the appropriate environment variable setting method according to the running platform.

## Runtime Initialization Optional Configurations

Notes:

1. All integer parameters are of Int64 type, and floating-point parameters are of Float64 type;
2. If no maximum value is explicitly specified for any parameter, the default implicit maximum value is the maximum value of that type;
3. If any parameter exceeds the range, the setting will be invalid, and the default value will be used automatically.

### `cjHeapSize`

Specifies the maximum size of the Cangjie heap. Supported units are kb (KB), mb (MB), gb (GB). The supported setting range is [4MB, system physical memory]. Settings outside the range are invalid, and the default value is still used. If the physical memory is less than 1GB, the default value is 64 MB; otherwise, it is 256 MB.

Example:

```shell
export cjHeapSize=4GB
```

### `cjRegionSize`

Specifies the size of the thread local buffer of the region allocator. Supported units are kb (KB), mb (MB), gb (GB). The supported setting range is [4kb, 2048kb]. Settings outside the range are invalid, and the default value is still used. The default value is 64 KB.

Example:

```shell
export cjRegionSize=1024kb
```

### `cjLargeThresholdSize`

Objects that require a large continuous memory space (such as long arrays) are called large objects. Frequent allocation of large objects in the heap may lead to insufficient continuous space in the heap, thus triggering heap overflow issues. Increasing the maximum value of large objects can improve the continuity of space in the heap.

In the Cangjie language, the threshold for large objects is the smaller of `cjLargeThresholdSize` and `cjRegionSize`. The supported units for `cjLargeThresholdSize` are kb (KB), mb (MB), gb (GB), and the supported range is [4KB, 2048KB]. Settings outside the range are invalid, and the default value is still used. The default value is 32 KB.

> **Explanation:**
>
> A larger threshold for large objects may affect program performance, and developers can set it according to actual conditions.

Example:

```shell
export cjLargeThresholdSize=1024kb
```

### `cjExemptionThreshold`

Specifies the waterline value of the surviving region, with a value range of (0,1]. This value is multiplied by the size of the region. If the size of surviving objects in the region is greater than the multiplied value, the region will not be reclaimed (where dead objects continue to occupy memory). The larger this value is specified, the higher the probability that the region will be reclaimed, and the less fragmented space in the heap will be, but frequent region reclamation will also affect performance. Settings outside the range are invalid, and the default value is still used. The default value is 0.8, i.e., 80%.

Example:

```shell
export cjExemptionThreshold=0.8
```

### `cjHeapUtilization`

Specifies the utilization rate of the Cangjie heap. This parameter is one of the reference bases for updating the heap waterline after GC, with a value range of (0, 1]. The heap waterline refers to the value at which GC is triggered when the total size of objects in the heap reaches the waterline value. The smaller this parameter is specified, the higher the updated heap waterline will be, and the probability of triggering GC will be relatively lower. Settings outside the range are invalid, and the default value is still used. The default value is 0.8, i.e., 80%.

Example:

```shell
export cjHeapUtilization=0.8
```

### `cjHeapGrowth`

Specifies the growth rate of the Cangjie heap. This parameter is one of the reference bases for updating the heap waterline after GC, and the value must be greater than 0. The growth rate is calculated as 1 + cjHeapGrowth. The larger this parameter is specified, the higher the updated heap waterline will be, and the probability of triggering GC will be relatively lower. The default value is 0.15, indicating a growth rate of 1.15.

Example:

```shell
export cjHeapGrowth=0.15
```

### `cjAllocationRate`

Specifies the rate at which the Cangjie runtime allocates objects. This value must be greater than 0, with the unit of MB/s, indicating the number of objects that can be allocated per second. The default value is 10240, indicating that 10240 MB of objects can be allocated per second.

Example:

```shell
export cjAllocationRate=10240
```

### `cjAllocationWaitTime`

Specifies the waiting time when the Cangjie runtime allocates objects. This value must be greater than 0, supporting units of s, ms, us, ns, and the recommended unit is nanoseconds (ns). If the time interval between the current object allocation and the last object allocation is less than this value, it will wait. The default value is 1000 ns.

Example:

```shell
export cjAllocationWaitTime=1000ns
```

### `cjGCThreshold`

Specifies the reference waterline value of the Cangjie heap, supporting units of kb (KB), mb (MB), gb (GB), and the value must be an integer greater than 0. When the size of the Cangjie heap exceeds this value, GC is triggered. The default value is the heap size.

Example:

```shell
export cjGCThreshold=20480KB
```

### `cjGarbageThreshold`

When GC occurs, if the ratio of dead objects in the region is greater than this environment variable, the region will be put into the recycling candidate set and can be reclaimed later (it may not be reclaimed if affected by other strategies). The default value is 0.5, dimensionless, and the supported setting interval is [0.0, 1.0].

Example:

```shell
export cjGarbageThreshold=0.5
```

### `cjGCInterval`

Specifies the interval between two GCs, with a value must be greater than 0, supporting units of s, ms, us, ns, and the recommended unit is milliseconds (ms). If the interval between the current GC and the last GC is less than this value, the current GC will be ignored. This parameter can control the frequency of GC. The default value is 150 ms.

Example:

```shell
export cjGCInterval=150ms
```

### `cjBackupGCInterval`

Specifies the interval of backup GC, with a value must be greater than 0, supporting units of s, ms, us, ns, and the recommended unit is seconds (s). If the Cangjie runtime does not trigger GC within the time set by this parameter, a backup GC will be triggered. The default value is 240 seconds, i.e., 4 minutes.

Example:

```shell
export cjBackupGCInterval=240s
```

### `cjProcessorNum`

Specifies the maximum concurrency of Cangjie threads. The supported setting range is (0, CPU cores * 2]. Settings outside the range are invalid, and the default value is still used. The system API is called to obtain the number of CPU cores. If successful, the default value is the number of CPU cores; otherwise, the default value is 8.

Example:

```shell
export cjProcessorNum=2
```

### `cjStackSize`

Specifies the stack size of Cangjie threads, supporting units of kb (KB), mb (MB), gb (GB). The supported setting range is [64KB, 1GB] on Linux platform and [128KB, 1GB] on Windows platform. Settings outside the range are invalid, and the default value is still used. The default value is 128KB.

Example:

```shell
export cjStackSize=100kb
```

### Operational Logging Optional Configurations

#### `MRT_LOG_FILE_SIZE`

Specifies the file size of the runtime operational log. The default value is 10 MB, supporting units of kb (KB), mb (MB), gb (GB), and the set value must be greater than 0.

When the log size exceeds this value, it will return to the beginning of the log for printing.

The size of the finally generated log is slightly larger than MRT_LOG_FILE_SIZE.

Example:

```shell
export MRT_LOG_FILE_SIZE=100kb
```

#### `MRT_LOG_PATH`

Specifies the output path of the runtime operational log. If this environment variable is not set or the path setting fails, the operational log is printed to stdout (standard output) or stderr (standard error) by default.

Example:

```shell
export MRT_LOG_PATH=/home/cangjie/runtime/runtime_log.txt
```

#### `MRT_LOG_LEVEL`

Specifies the minimum output level of the runtime operational log. Logs greater than or equal to this level will be printed. The default value is e, and the supported setting values are [v|d|i|w|e|f|s]. v (VERBOSE), d (DEBUGY), i (INFO), w (WARNING), e (ERROR), f (FATAL), s (FATAL_WITHOUT_ABORT).

Example:

```shell
export MRT_LOG_LEVEL=v
```

#### `MRT_REPORT`

Specifies the output path of the runtime GC log. If this environment variable is not set or the path setting fails, the log is not printed by default.

Example:

```shell
export MRT_REPORT=/home/cangjie/runtime/gc_log.txt
```

#### `MRT_LOG_CJTHREAD`

Specifies the output path of the cjthread log. If this environment variable is not set or the path setting fails, the log is not printed by default.

Example:

```shell
export MRT_LOG_CJTHREAD=/home/cangjie/runtime/cjthread_log.txt
```

#### `cjHeapDumpOnOOM`

Specifies whether to output a heap snapshot file after a heap overflow occurs, which is disabled by default. Supported setting values are [on|off]. Setting it to on enables the function, and setting it to off or other values disables the function.

Example:

```shell
export cjHeapDumpOnOOM=on
```

#### `cjHeapDumpLog`

Specifies the path to output the heap snapshot file. Note that the specified path must exist, and the application executor has read and write permissions for it. If not specified, the heap snapshot file will be output to the current execution directory.

Example:

```shell
export cjHeapDumpLog=/home/cangjie
```

### Runtime Environment Optional Configurations

#### `MRT_STACK_CHECK`

Enables native stack overflow checking, which is disabled by default. Supported setting values are 1, true, TRUE to enable the function.

Example:

```shell
export MRT_STACK_CHECK=true
```

#### `CJ_SOF_SIZE`

When a StackOverflowError occurs, the exception stack will be automatically folded for users to read, and the default number of folded stack frames is 32. You can control the length of the folded stack by configuring this environment variable, which supports setting to an integer within the int range.
- CJ_SOF_SIZE = 0: Print all call stacks;
- CJ_SOF_SIZE < 0: Print the number of layers configured by the environment variable from the bottom of the stack;
- CJ_SOF_SIZE > 0: Print the number of layers configured by the environment variable from the top of the stack;
- CJ_SOF_SIZE is not configured: Print the top 32 layers of the call stack by default;

Example:

```shell
export CJ_SOF_SIZE=30
```

### Cangjie GWP-Asan Memory Safety Detection

During the interoperability between Cangjie and C code, some Cangjie heap memory safety issues may occur. Cangjie GWP-Asan provides a memory safety detection function. It can detect whether the code has Cangjie heap memory safety issues during the operation of the Cangjie program. GWP-Asan samples the acquireArrayRawData and releaseArrayRawData interfaces provided by the Cangjie language standard library (see the std.core package section of the *Cangjie Programming Language Library API Documentation*), and records and compares the Canary data of the memory before and after sampling objects, thereby detecting whether there are Cangjie heap memory safety issues during the interoperability between Cangjie and C language.

Cangjie GWP-Asan is a sampling-based detection tool. The sampling frequency can be adjusted by setting different values to balance performance impact and detection coverage. At the default or lower sampling frequency, the CPU performance loss and additional memory usage are extremely low.

> **Explanation:**
>
> Cangjie GWP-Asan memory safety detection only supports Linux and HarmonyOS operating systems.

#### cjEnableGwpAsan

The Cangjie GWP-Asan memory safety detection function is disabled by default. The function can be enabled by setting the environment variable `cjEnableGwpAsan` to `1`, `true` or `TRUE`. The setting reference on Linux is as follows:

```shell
export cjEnableGwpAsan=true
```

#### cjGwpAsanSampleRate

When the Cangjie GWP-Asan memory safety detection function is enabled, the sampling frequency is set through the environment variable `cjGwpAsanSampleRate`. `cjGwpAsanSampleRate` supports setting to a positive integer within the range of 32-bit integer values, i.e., $(0, 2^{31} - 1]$ . The default value is 5000, which means one sampling is performed for every 5000 calls to the acquireArrayRawData interface. The setting reference on Linux is as follows:

```shell
export cjGwpAsanSampleRate=1000
```

> **Explanation:**
>
> In Cangjie GWP-Asan memory safety detection, sampling will affect performance. The higher the sampling rate, the greater the impact on performance and the more problems can be detected; the lower the sampling rate, the smaller the impact on performance and the fewer problems can be detected. Please set the sampling rate according to actual conditions.

#### cjGwpAsanHelp

Whether to output GWP-Asan help information on the console can be set through the environment variable `cjGwpAsanHelp`. It is disabled by default. When `cjGwpAsanHelp` is set to `1`, `true` or `TRUE`, it means outputting help information on the console. The setting reference on Linux is as follows:

```shell
export cjGwpAsanHelp=true
```

#### Constraints and Limitations

- Cangjie GWP-Asan is a sampling-based memory checking tool, and memory out-of-bounds problems may not be completely detected.
- Cangjie GWP-Asan has a limited range of detection for out-of-bounds access to Cangjie heap memory. It cannot detect memory read out-of-bounds access, and can only detect some write out-of-bounds access: forward write out-of-bounds within 8 bytes; backward write out-of-bounds to the padding area at the end (the padding area may be 0-7 bytes depending on the length of the array object).

#### Exception Detection Types

**Heap Memory Write Out-of-Bounds**

Heap memory write out-of-bounds means that the actual memory length accessed by the pointer exceeds the length applied for by the array, causing Cangjie heap memory write out-of-bounds.

1. Forward Out-of-Bounds

    When the array is out-of-bounds forward, the runtime will report a Head canary detection failure, represented by `array[-1]`. For example:

    <!-- compile -->

    ```cangjie
    unsafe {
        var array = Array<UInt8>(4, repeat: 1)
        var cp = acquireArrayRawData<UInt8>(array)
        cp.pointer.write(-1, 2)
        releaseArrayRawData(cp)
    }
    ```

    The corresponding error report is as follows:

    ```text
    2025-05-22 10:57:13.432786 41217 F Gwp-Asan sanity check failed on raw array addr 0x7f7c887368
    2025-05-22 10:57:13.432863 41217 F Head canary (array[-1]) mismatch: expect: 0x4, actual: 0x200000000000004
    2025-05-22 10:57:13.432878 41217 F Gwp-Asan Aborted.
    ```

2. Backward Out-of-Bounds

    When the array is out-of-bounds backward, the runtime will report a Tail canary detection failure and give the position relative to the array (`array`). For example:

    <!-- compile -->

    ```cangjie
    unsafe {
        let array = Array<UInt8>(4, repeat: 0)
        let cp = acquireArrayRawData(array)

        // The actual accessible range of the array is [0, 4), and the following write operation accesses the 6th byte, causing the Cangjie heap memory to overflow 2 bytes forward. The backward out-of-bounds behavior is represented by array[size+1] in the error report.
        cp.pointer.write(5, 2)
        releaseArrayRawData(cp)
    }
    ```

    The corresponding error report is as follows:

    ```text
    2025-05-22 10:53:09.564580 37872 F Gwp-Asan sanity check failed on raw array addr 0x7f6278a368
    2025-05-22 10:53:09.564761 37872 F Tail canary (array[size+1]) mismatch: expect: 0x4, actual: 0x2
    2025-05-22 10:53:09.564788 37872 F Gwp-Asan Aborted.
    ```

**Cangjie GC (Garbage Collection) Exception**

Using acquireArrayRawData to obtain the pointer of the array but not using releaseArrayRawData to release the reference of the array may cause Cangjie GC (Garbage Collection) exceptions.

When the runtime exits, it will detect whether the sampled array has called releaseArrayRawData to release. If not released, it will report the heap addresses corresponding to all unreleased arrays. For example:

<!-- compile -->

```cangjie
unsafe {
    let array = Array<UInt8>(4, repeat: 0)
    let cp = acquireArrayRawData(array)
    cp.pointer.read()

    // releaseArrayRawData is not used
    return
}
```

The corresponding error report is as follows:

```text
2025-05-22 10:53:09.564761 1248614 F Unreleased array: 0x7fffd77f92d8
2025-05-22 10:53:09.564788 1248614 F Detect un-released array
```
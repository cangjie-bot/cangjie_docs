# I/O Node Streams

Node streams refer to streams that directly provide data sources. The construction of node streams typically relies on some direct external resource (such as files, networks, etc.).

Common node streams in the Cangjie programming language include standard streams (StdIn, StdOut, StdErr), file streams (File), and network streams (Socket).

This chapter introduces standard streams and file streams.

## Standard Streams

Standard streams include the standard input stream (stdin), standard output stream (stdout), and standard error output stream (stderr).

Standard streams serve as the standard interface for program interaction with external data. During program execution, data is read from the input stream as program input, output information is transmitted to the output stream, and similarly, error information is transmitted to the error stream.

In the Cangjie programming language, the `Console` type can be used to access them separately.

To use the `Console` type, the `console` package must be imported:

Example of importing the console package:

<!-- run -->

```cangjie
import std.console.*
```

The `Console` provides user-friendly wrappers for all three standard streams, offering more convenient `String`-based extended operations. It also includes rich overloads for many common types to optimize performance.

Most importantly, `Console` guarantees thread safety, allowing safe reading and writing through its interfaces from any thread.

By default, the standard input stream originates from keyboard input, such as text entered in a command-line interface.

When data needs to be obtained from the standard input stream, it can be read via `stdIn`. For example, the `readln` function can be used to capture command-line input.

Example of reading from the standard input stream:

<!-- run -->

```cangjie
import std.console.Console

main() {
    let txt = Console.stdIn.readln()
    println(txt ?? "")
}
```

When running the above code, enter some text in the command line and press Enter to see the input content.

The output streams are divided into the standard output stream and standard error stream. By default, both output to the screen, such as text seen in a command-line interface.

When data needs to be written to the standard output stream, it can be done via `stdOut`/`stdErr`. For example, the `write` function can be used to print content to the command line.

The difference between using `stdOut` and directly using the `print` function is that `stdOut` is thread-safe and, due to its use of caching technology, offers better performance when handling large amounts of content.

Note that after writing data, `flush` must be called on `stdOut` to ensure the content is written to the standard stream.

Example of writing to the standard output stream:

<!-- run -->

```cangjie
import std.console.Console

main() {
    for (i in 0..1000) {
        Console.stdOut.writeln("hello, world!")
    }
    Console.stdOut.flush()
}
```

## File Streams

The Cangjie programming language provides the `fs` package to support general file system tasks. Different operating systems offer varying interfaces for file systems. The Cangjie programming language abstracts the following common functionalities, providing unified interfaces to simplify usage while masking differences between operating systems.

Common tasks include: creating files/directories, reading/writing files, renaming or moving files/directories, deleting files/directories, copying files/directories, obtaining file/directory metadata, and checking file/directory existence. Specific APIs can be found in the library documentation.

To use file system-related functionalities, the `fs` package must be imported:

Example of importing the fs package:

<!-- run -->

```cangjie
import std.fs.*
```

This chapter focuses on the usage of `File`. For details on `Path` and `Directory`, refer to the corresponding API documentation.

The `File` type in the Cangjie programming language provides both conventional file operations and file stream functionalities.

### Conventional File Operations

For conventional file operations, a series of static functions can be used for quick tasks.

For example, to check whether a file exists at a given path, the `exists` function can be used. When `exists` returns `true`, the file exists; otherwise, it does not.

Example of using the exists function:

<!-- run -->

```cangjie
import std.fs.exists

main() {
    let exist = exists("./tempFile.txt")
    println("exist: ${exist}")
}
```

Moving, copying, and deleting files are also straightforward. The `File` type provides corresponding static functions: `move`, `copy`, and `delete`.

Example of using move, copy, and delete functions:

<!-- compile -->

```cangjie
import std.fs.{copy, rename, remove}

main() {
    copy("./tempFile.txt", to: "./tempFile2.txt", overwrite: false)
    rename("./tempFile2.txt",  to: "./tempFile3.txt", overwrite: false)
    remove("./tempFile3.txt")
}
```

If all data from a file needs to be read at once or written to a file in a single operation, the `File` type provides `readFrom` and `writeTo` functions. These are simple to use and offer good performance for small data volumes, eliminating the need for manual stream handling.

Example of using readFrom and writeTo functions:

<!-- compile -->

```cangjie
import std.fs.File

main() {
    let bytes = File.readFrom("./tempFile.txt") // Reads all data at once
    File.writeTo("./otherFile.txt", bytes) // Writes the data to another file in one operation
}
```

### File Stream Operations

In addition to conventional file operations, the `File` type is also designed as a data stream type. Thus, the `File` type implements the `IOStream` interface. When a `File` instance is created, it can be used as a data stream.

Definition of the File class:

```cangjie
public class File <: Resource & IOStream & Seekable {
    ...
}
```

`File` provides two construction methods: one uses the convenient static function `create` to directly create a new file instance, while the other uses a constructor with a complete file opening mode to create a new instance.

Files created via `create` are write-only; attempting to read from such an instance will throw a runtime exception.

Example of File construction:

<!-- compile -->

```cangjie
// Create
internal import std.fs.*
internal import std.io.*

main() {
    let file = File.create("./tempFile.txt")
    file.write("hello, world!".toArray())

    // Open
    let file2 = File("./tempFile.txt", Read)
    let bytes = readToEnd(file2) // Reads all data
    println(bytes)
}
```

When more precise opening modes are required, the constructor can be used with an `OpenMode` value. `OpenMode` is an `enum` type that provides rich file opening modes, including `Read`, `Write`, `Append`, and `ReadWrite`.

Example of using File opening modes:

```cangjie
// Open with specified mode
let file = File("./tempFile.txt", Write)
```

Since opening a `File` instance consumes valuable system resources, it is important to close the `File` instance promptly after use to release these resources.

`File` implements the `Resource` interface, allowing the use of try-with-resource syntax for simplified usage in most cases.

Example of try-with-resource syntax:

```cangjie
try (file2 = File("./tempFile.txt", Read)) {
    ...
    // Automatically releases the file after use
}
```
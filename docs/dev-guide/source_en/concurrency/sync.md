# Synchronization Mechanisms

In concurrent programming, the lack of synchronization mechanisms to protect variables shared by multiple threads can easily lead to data race issues.

The Cangjie programming language provides three common synchronization mechanisms to ensure thread safety of data: atomic operations, mutex locks, and condition variables.

## Atomic Operations

Cangjie supports atomic operations for integer types, `Bool` type, and reference types.

The integer types include: `Int8`, `Int16`, `Int32`, `Int64`, `UInt8`, `UInt16`, `UInt32`, `UInt64`.

Atomic operations for integer types support basic read/write, swap, and arithmetic operations:

| Operation         | Function Description                                                                 |
| ----------------- | ------------------------------------------------------------------------------------- |
| `load`            | Read the value                                                                        |
| `store`           | Write the value                                                                       |
| `swap`            | Swap values and return the original value before the swap                              |
| `compareAndSwap`  | Compare and swap: return `true` if the swap succeeds, otherwise return `false`         |
| `fetchAdd`        | Perform addition and return the value before the addition operation                   |
| `fetchSub`        | Perform subtraction and return the value before the subtraction operation              |
| `fetchAnd`        | Perform bitwise AND and return the value before the AND operation                     |
| `fetchOr`         | Perform bitwise OR and return the value before the OR operation                       |
| `fetchXor`        | Perform bitwise XOR and return the value before the XOR operation                     |

Notes:
1. The return values of swap and arithmetic operations are the values before modification.
2. `compareAndSwap` checks if the current value of the atomic variable equals the `old` value. If equal, it replaces the value with `new`; otherwise, it does not perform the replacement.

Taking the `Int8` type as an example, the corresponding atomic operation type declaration is as follows:

<!-- code_no_check -->
```cangjie
class AtomicInt8 {
    public func load(): Int8
    public func store(val: Int8): Unit
    public func swap(val: Int8): Int8
    public func compareAndSwap(old: Int8, new: Int8): Bool
    public func fetchAdd(val: Int8): Int8
    public func fetchSub(val: Int8): Int8
    public func fetchAnd(val: Int8): Int8
    public func fetchOr(val: Int8): Int8
    public func fetchXor(val: Int8): Int8
}
```

Each method of the above atomic types has a corresponding overload that accepts a memory ordering parameter. Currently, the only supported memory ordering is sequential consistency.

Similarly, the corresponding atomic operation types for other integer types are:

<!-- code_no_check -->
```cangjie
class AtomicInt16 {...}
class AtomicInt32 {...}
class AtomicInt64 {...}
class AtomicUInt8 {...}
class AtomicUInt16 {...}
class AtomicUInt32 {...}
class AtomicUInt64 {...}
```

The following example demonstrates how to use atomic operations to implement counting in a multi-threaded program:

<!-- verify -->

```cangjie
import std.sync.AtomicInt64
import std.collection.ArrayList

let count = AtomicInt64(0)

main(): Int64 {
    let list = ArrayList<Future<Int64>>()

    // Create 1000 threads.
    for (_ in 0..1000) {
        let fut = spawn {
            sleep(Duration.millisecond) // Sleep for 1ms.
            count.fetchAdd(1)
        }
        list.add(fut)
    }

    // Wait for all threads to finish.
    for (f in list) {
        f.get()
    }

    let val = count.load()
    println("count = ${val}")
    return 0
}
```

The output should be:

```text
count = 1000
```

Here are some other correct examples of using atomic operations for integer types:

<!-- compile -->

```cangjie
var obj: AtomicInt32 = AtomicInt32(1)
var x: Int32 = obj.load() // x: 1
x = obj.swap(2) // x: 1
x = obj.load() // x: 2
var y: Bool = obj.compareAndSwap(2, 3) // y: true
y = obj.compareAndSwap(2, 3) // y: false, the value in obj is no longer 2 but 3. Therefore, the CAS operation fails.
x = obj.fetchAdd(1) // x: 3
x = obj.load() // x: 4
```

Atomic operations for `Bool` type and reference types only provide read/write and swap operations:

| Operation         | Function Description                                                                 |
| ----------------- | ------------------------------------------------------------------------------------- |
| `load`            | Read the value                                                                        |
| `store`           | Write the value                                                                       |
| `swap`            | Swap values and return the original value before the swap                              |
| `compareAndSwap`  | Compare and swap: return `true` if the swap succeeds, otherwise return `false`         |

> **Note:**
>
> Atomic operations for reference types are only valid for reference types.

The atomic reference type is `AtomicReference`. Here are some correct examples of using atomic operations for `Bool` type and reference types:

<!-- verify -->

```cangjie
import std.sync.{AtomicBool, AtomicReference}

class A {}

main() {
    var obj = AtomicBool(true)
    var x1 : Bool = obj.load() // x1: true
    println(x1)
    var t1 = A()
    var obj2 = AtomicReference(t1)
    var x2 = obj2.load() // x2 and t1 are the same object
    var y1 = obj2.compareAndSwap(x2, t1) // x2 and t1 are the same object, y1: true
    println(y1)
    var t2 = A()
    var y2 = obj2.compareAndSwap(t2, A()) // t2 and t1 are not the same object, CAS fails, y2: false
    println(y2)
    y2 = obj2.compareAndSwap(t1, A()) // CAS succeeds, y2: true
    println(y2)
}
```

Compiling and executing the above code produces the following output:

```text
true
true
false
true
```

## Reentrant Mutex

A reentrant mutex protects critical sections such that at most one thread can execute the code in the critical section at any time. When a thread attempts to acquire a lock already held by another thread, the thread blocks until the lock is released and the thread is awakened. Reentrancy means that a thread can acquire the same lock again after already holding it.

When using a reentrant mutex, two rules must be remembered:
1. Always attempt to acquire the lock before accessing shared data.
2. Always release the lock after processing shared data to allow other threads to acquire it.

The main member functions provided by `Mutex` are as follows:

<!-- code_no_check -->
```cangjie
public class Mutex <: UniqueLock {
    // Create a Mutex.
    public init()

    // Locks the mutex; blocks if the mutex is not available.
    public func lock(): Unit

    // Unlocks the mutex. If other threads are blocking on this lock, wake up one of them.
    public func unlock(): Unit

    // Tries to lock the mutex; returns false if the mutex is not available, otherwise returns true.
    public func tryLock(): Bool

    // Generates a Condition instance for the mutex.
    public func condition(): Condition
}
```

The following example demonstrates how to use `Mutex` to protect access to the globally shared variable `count`—operations on `count` constitute the critical section:

<!-- verify -->

```cangjie
import std.sync.Mutex
import std.collection.ArrayList

var count: Int64 = 0
let mtx = Mutex()

main(): Int64 {
    let list = ArrayList<Future<Unit>>()

    // Create 1000 threads.
    for (i in 0..1000) {
        let fut = spawn {
            sleep(Duration.millisecond) // Sleep for 1ms.
            mtx.lock()
            count++
            mtx.unlock()
        }
        list.add(fut)
    }

    // Wait for all threads to finish.
    for (f in list) {
        f.get()
    }

    println("count = ${count}")
    return 0
}
```

The output should be:

```text
count = 1000
```

The following example demonstrates how to use `tryLock`:

<!-- run -->

```cangjie
import std.sync.Mutex

main(): Int64 {
    let mtx: Mutex = Mutex()
    var future: Future<Unit> = spawn {
        mtx.lock()
        println("get the lock, do something")
        sleep(Duration.millisecond * 10)
        mtx.unlock()
    }
    try {
        future.get(Duration.millisecond * 10)
    } catch (e: TimeoutException) {
        if (mtx.tryLock()) {
            println("tryLock success, do something")
            mtx.unlock()
            return 0
        }
        println("tryLock failed, do nothing")
        return 0
    }
    return 0
}
```

A possible output is:

```text
get the lock, do something
```

Here are some incorrect examples of using mutex locks:

Incorrect Example 1: The thread does not unlock after operating on the critical section, causing other threads to block indefinitely when trying to acquire the lock.

<!-- compile.error -->
<!-- cfg="libcangjie-std-sync" -->

```cangjie
import std.sync.Mutex

var sum: Int64 = 0
let mutex = Mutex()

main() {
    let foo = spawn { =>
        mutex.lock()
        sum = sum + 1
    }
    let bar = spawn { =>
        mutex.lock()
        sum = sum + 1
    }
    foo.get()
    println("${sum}")
    bar.get() // Because the thread did not unlock, other threads waiting to acquire the mutex will block indefinitely.
}
```

Incorrect Example 2: Calling `unlock` without holding the lock in the current thread throws an exception.

<!-- compile.error -->
<!-- cfg="libcangjie-std-sync" -->

```cangjie
import std.sync.Mutex

var sum: Int64 = 0
let mutex = Mutex()

main() {
    let foo = spawn { =>
        sum = sum + 1
        mutex.unlock() // Error: Unlocking without acquiring the lock throws an exception: IllegalSynchronizationStateException.
    }
    foo.get()
}
```

Incorrect Example 3: `tryLock()` does not guarantee acquiring the lock, which may lead to operating on the critical section without lock protection or throwing an exception when calling `unlock` without holding the lock.

<!-- compile.error -->
<!-- cfg="libcangjie-std-sync" -->

```cangjie
import std.sync.Mutex

var sum: Int64 = 0
let mutex = Mutex()

main() {
    for (i in 0..100) {
        spawn { =>
            mutex.tryLock() // Error: `tryLock()` only attempts to acquire the lock; there is no guarantee of success. This can lead to abnormal behavior.
            sum = sum + 1
            mutex.unlock()
        }
    }
}
```

Additionally, `Mutex` is designed as a reentrant lock. That is: if a thread already holds a `Mutex` lock, attempting to acquire the same `Mutex` lock again will always succeed immediately.

> **Note:**
>
> Although `Mutex` is reentrant, the number of `unlock()` calls must match the number of `lock()` calls to successfully release the lock.

The following example demonstrates the reentrancy feature of `Mutex`:

<!-- verify -->

```cangjie
import std.sync.Mutex

var count: Int64 = 0
let mtx = Mutex()

func foo() {
    mtx.lock()
    count += 10
    bar()
    mtx.unlock()
}

func bar() {
    mtx.lock()
    count += 100
    mtx.unlock()
}

main(): Int64 {
    let fut = spawn {
        sleep(Duration.millisecond) // Sleep for 1ms.
        foo()
    }

    foo()

    fut.get()

    println("count = ${count}")
    return 0
}
```

The output should be:

```text
count = 220
```

In the above example, whether in the main thread or the newly created thread, if the lock is already acquired in `foo()`, calling `bar()` will immediately acquire the same `Mutex` lock again without deadlock.

## Condition Variables

A `Condition` is a condition variable (i.e., a wait queue) bound to a mutex. `Condition` instances are created by mutexes, and a single mutex can create multiple `Condition` instances. A `Condition` allows threads to block and wait for a signal from another thread to resume execution. This is a mechanism for thread synchronization using shared variables, providing the following main methods:

<!-- code_no_check -->
```cangjie
public class Mutex <: UniqueLock {
    // ...
    // Generates a Condition instance for the mutex.
    public func condition(): Condition
}

public interface Condition {
    // Waits for a signal, blocking the current thread.
    func wait(): Unit
    func wait(timeout!: Duration): Bool

    // Waits for a signal and predicate, blocking the current thread.
    func waitUntil(predicate: ()->Bool): Unit
    func waitUntil(predicate: ()->Bool, timeout!: Duration): Bool

    // Wakes up one thread waiting on the monitor (if any).
    func notify(): Unit

    // Wakes up all threads waiting on the monitor (if any).
    func notifyAll(): Unit
}
```

Before calling the `wait`, `notify`, or `notifyAll` methods of the `Condition` interface, ensure the current thread holds the bound mutex. The `wait` method performs the following actions:
1. Adds the current thread to the wait queue of the corresponding mutex.
2. Blocks the current thread while fully releasing the mutex and recording the lock's reentrancy count.
3. Waits for another thread to send a signal via the `notify` or `notifyAll` method of the same `Condition` instance.
4. After the current thread is awakened, it automatically attempts to reacquire the lock with the same reentrancy state as recorded in step 2. If reacquiring the lock fails, the current thread blocks on the mutex.

The `wait` method accepts an optional `timeout` parameter. Note that many common operating systems do not guarantee real-time scheduling, so it is not possible to ensure a thread is blocked for "exactly N nanoseconds"—system-dependent imprecisions may be observed. Additionally, the current language specification explicitly allows spurious wakeups—where `wait` returns arbitrarily (either `true` or `false`). Therefore, developers are encouraged to always wrap `wait` in a loop.

The following is a correct example of using `Condition`:

<!-- verify -->

```cangjie
import std.sync.Mutex

let mtx = Mutex()
let condition = synchronized(mtx) {
    mtx.condition()
}
var flag: Bool = true

main(): Int64 {
    let fut = spawn {
        mtx.lock()
        while (flag) {
            println("New thread: before wait")
            condition.wait()
            println("New thread: after wait")
        }
        mtx.unlock()
    }

    // Sleep for 10ms to ensure the new thread can execute.
    sleep(10 * Duration.millisecond)

    mtx.lock()
    println("Main thread: set flag")
    flag = false
    mtx.unlock()

    mtx.lock()
    println("Main thread: notify")
    condition.notifyAll()
    mtx.unlock()

    // Wait for the new thread to finish.
    fut.get()
    return 0
}
```

The output should be:

```text
New thread: before wait
Main thread: set flag
Main thread: notify
New thread: after wait
```

When a `Condition` object executes `wait`, it must be done under the protection of the bound mutex. Otherwise, the unlock operation in `wait` will throw an exception.

Here are some incorrect examples of using condition variables:

<!-- run.error -->

```cangjie
import std.sync.Mutex

let m1 = Mutex()
let c1 = synchronized(m1) {
    m1.condition()
}
let m2 = Mutex()
var flag: Bool = true
var count: Int64 = 0

func foo1() {
    spawn {
        m2.lock()
        while (flag) {
            c1.wait() // Error: The mutex used with the condition variable must be the same mutex and held in a locked state. Otherwise, the unlock operation in `wait` throws an exception.
        }
        count = count + 1
        m2.unlock()
    }
    m1.lock()
    flag = false
    c1.notifyAll()
    m1.unlock()
}

func foo2() {
    spawn {
        while (flag) {
            c1.wait() // Error: `wait` for a condition variable must be called while holding the lock.
        }
        count = count + 1
    }
    m1.lock()
    flag = false
    c1.notifyAll()
    m1.unlock()
}

main() {
    foo1()
    foo2()
    c1.wait()
}
```

In complex inter-thread synchronization scenarios, it is sometimes necessary to generate multiple `Condition` instances for the same lock object. The following example implements a bounded FIFO queue with a fixed length. When the queue is empty, `get()` blocks; when the queue is full, `put()` blocks.

<!-- compile -->

```cangjie
import std.sync.{Mutex, Condition}

class BoundedQueue {
    // Create a Mutex and two Conditions.
    let m: Mutex = Mutex()
    var notFull: Condition
    var notEmpty: Condition

    var count: Int64 // Number of objects in the buffer.
    var head: Int64  // Write index.
    var tail: Int64  // Read index.

    // The queue length is 100.
    let items: Array<Object> = Array<Object>(100, {i => Object()})

    init() {
        count = 0
        head = 0
        tail = 0

        synchronized(m) {
          notFull  = m.condition()
          notEmpty = m.condition()
        }
    }

    // Insert an object; block the current thread if the queue is full.
    public func put(x: Object) {
        // Acquire the mutex.
        synchronized(m) {
          while (count == 100) {
            // If the queue is full, wait for the "queue notFull" event.
            notFull.wait()
          }
          items[head] = x
          head++
          if (head == 100) {
            head = 0
          }
          count++

          // An object has been inserted; the queue is no longer empty.
          // Wake up threads previously blocked on get() due to an empty queue.
          notEmpty.notify()
        } // Release the mutex.
    }

    // Pop an object; block the current thread if the queue is empty.
    public func get(): Object {
        // Acquire the mutex.
        synchronized(m) {
          while (count == 0) {
            // If the queue is empty, wait for the "queue notEmpty" event.
            notEmpty.wait()
          }
          let x: Object = items[tail]
          tail++
          if (tail == 100) {
            tail = 0
          }
          count--

          // An object has been popped; the queue is no longer full.
          // Wake up threads previously blocked on put() due to a full queue.
          notFull.notify()

          return x
        } // Release the mutex.
    }
}
```

## `synchronized` Keyword

`Lock` provides a convenient and flexible way to acquire locks. However, due to its flexibility, it may lead to issues such as forgetting to unlock or failing to automatically release the lock when an exception is thrown while holding the lock. Therefore, the Cangjie programming language provides a `synchronized` keyword that can be used with `Lock` to automatically acquire and release the lock within the subsequent scope, solving such problems.

The following example demonstrates how to use the `synchronized` keyword to protect shared data:

<!-- verify -->

```cangjie
import std.sync.Mutex
import std.collection.ArrayList

var count: Int64 = 0
let mtx = Mutex()

main(): Int64 {
    let list = ArrayList<Future<Unit>>()

    // Create 1000 threads.
    for (i in 0..1000) {
        let fut = spawn {
            sleep(Duration.millisecond) // Sleep for 1ms.
            // Use synchronized(mtx) instead of mtx.lock() and mtx.unlock().
            synchronized(mtx) {
                count++
            }
        }
        list.add(fut)
    }

    // Wait for all threads to finish.
    for (f in list) {
        f.get()
    }

    println("count = ${count}")
    return 0
}
```

The output should be:

```text
count = 1000
```

By following `synchronized` with a `Lock` instance, the subsequent code block is protected such that at most one thread can execute the protected code at any time:
1. Before entering the `synchronized` code block, a thread automatically acquires the lock corresponding to the `Lock` instance. If the lock cannot be acquired, the current thread blocks.
2. Before exiting the `synchronized` code block, a thread automatically releases the lock corresponding to the `Lock` instance.

For control transfer expressions (such as `break`, `continue`, `return`, `throw`), when the program execution jumps out of the `synchronized` code block, the above rule 2 still applies—meaning the lock corresponding to the `synchronized` expression is automatically released.

The following example demonstrates the use of a `break` statement within a `synchronized` code block:

<!-- verify -->

```cangjie
import std.sync.Mutex
import std.collection.ArrayList

var count: Int64 = 0
var mtx: Mutex = Mutex()

main(): Int64 {
    let list = ArrayList<Future<Unit>>()
    for (i in 0..10) {
        let fut = spawn {
            while (true) {
                synchronized(mtx) {
                    count = count + 1
                    break
                    println("in thread")
                }
            }
        }
        list.add(fut)
    }

    // Wait for all threads to finish.
    for (f in list) {
        f.get()
    }

    synchronized(mtx) {
        println("in main, count = ${count}")
    }
    return 0
}
```

The output should be:

```text
in main, count = 10
```

In practice, the line `in thread` is not printed because the `break` statement causes the program to exit the `while` loop (and the `synchronized` code block first before exiting the `while` loop).

## Thread-Local Variables (`ThreadLocal`)

The `ThreadLocal` class in the core package allows creating and using thread-local variables. Each thread has its own independent storage space for these variables. Therefore, each thread can safely access its own thread-local variables without interference from other threads.

<!-- code_no_check -->
```cangjie
public class ThreadLocal<T> {
    /* Constructs a Cangjie thread-local variable with a null value. */
    public init()

    /* Retrieves the value of the Cangjie thread-local variable. */
    public func get(): Option<T> // Returns Option<T>.None if the value does not exist. Return value: Option<T> - The value of the Cangjie thread-local variable.

    /* Sets the value of the Cangjie thread-local variable. */
    public func set(value: Option<T>): Unit // If Option<T>.None is passed, the value of the local variable is deleted and cannot be retrieved in subsequent thread operations. Parameter: value - The value to set for the local variable.
}
```

The following example demonstrates how to create and use thread-local variables for each thread via the `ThreadLocal` class:

<!-- run -->

```cangjie

main(): Int64 {
    let tl = ThreadLocal<Int64>()
    let fut1 = spawn {
        tl.set(123)
        println("tl in spawn1 = ${tl.get().getOrThrow()}")
    }
    let fut2 = spawn {
        tl.set(456)
        println("tl in spawn2 = ${tl.get().getOrThrow()}")
    }
    fut1.get()
    fut2.get()
    0
}
```

A possible output is:

```text
tl in spawn1 = 123
tl in spawn2 = 456
```

Or:

```text
tl in spawn2 = 456
tl in spawn1 = 123
```

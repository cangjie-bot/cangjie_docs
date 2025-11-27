# ArrayList

We need to import the collection package to use the ArrayList type:

<!-- run -->

```cangjie
import std.collection.*
```

In Cangjie, the `ArrayList<T>` type represents an ArrayList, where `T` denotes the element type of the ArrayList and can be any type.

ArrayList has excellent expansion capabilities, making it suitable for scenarios that require frequent addition and deletion of elements.

Compared to Array, ArrayList allows both in-place modification of elements and in-place addition/deletion of elements.

The mutability of ArrayList is a very useful feature: all references to the same ArrayList instance share the same elements, and modifications can be made uniformly to them.

<!-- code_no_check -->

```cangjie
var a: ArrayList<Int64> = ... // ArrayList whose element type is Int64
var b: ArrayList<String> = ... // ArrayList whose element type is String
```

ArrayLists with different element types are distinct types, so they cannot be assigned to each other.

Therefore, the following example is invalid:

<!-- code_no_check -->

```cangjie
b = a // Type mismatch
```

In Cangjie, you can construct a specified ArrayList using a constructor:

<!-- run -->

```cangjie
let a = ArrayList<String>() // Created an empty ArrayList whose element type is String
let b = ArrayList<String>(100) // Created an ArrayList whose element type is String, and allocated space for 100 elements
let c = ArrayList<Int64>([0, 1, 2]) // Created an ArrayList whose element type is Int64, containing elements 0, 1, 2
let d = ArrayList<Int64>(c) // Initialized an ArrayList using another Collection
let e = ArrayList<String>(2, {x: Int64 => x.toString()}) // Created an ArrayList whose element type is String and size is 2. All elements are initialized by the specified rule function
```

## Accessing ArrayList Members

To access all elements of an ArrayList, you can traverse them using a for-in loop:

<!-- verify -->

```cangjie
import std.collection.ArrayList

main() {
    let list = ArrayList<Int64>([0, 1, 2])
    for (i in list) {
        println("The element is ${i}")
    }
}
```

Compiling and executing the above code will output:

```text
The element is 0
The element is 1
The element is 2
```

To get the number of elements contained in an ArrayList, you can use the `size` property:

<!-- verify -->

```cangjie
import std.collection.ArrayList

main() {
    let list = ArrayList<Int64>([0, 1, 2])
    if (list.size == 0) {
        println("This is an empty arraylist")
    } else {
        println("The size of arraylist is ${list.size}")
    }
}
```

Compiling and executing the above code will output:

```text
The size of arraylist is 3
```

To access a single element at a specified position, you can use the subscript syntax (the index type must be `Int64`). The first element of a non-empty ArrayList always starts at position 0. You can access any element of the ArrayList starting from 0 up to the last position ( `ArrayList.size - 1` ). Using a negative index or an index greater than or equal to the size will trigger a runtime exception.

<!-- code_no_check -->

```cangjie
let a = list[0] // a == 0
let b = list[1] // b == 1
let c = list[-1] // Runtime exception
```

ArrayList also supports the syntax of using Range in subscripts, see the [Array](../basic_data_type/array.md#array) chapter for details.

## Modifying ArrayList

You can modify an element at a specific position using the subscript syntax:

<!-- run -->

```cangjie
let list = ArrayList<Int64>([0, 1, 2])
list[0] = 3
```

ArrayList is a reference type, and no copy is made when an ArrayList is used as an expression. All references to the same ArrayList instance share the same data.

Therefore, modifications to the elements of an ArrayList will affect all references to that instance:

<!-- run -->

```cangjie
let list1 = ArrayList<Int64>([0, 1, 2])
let list2 = list1
list2[0] = 3
// list1 contains elements 3, 1, 2
// list2 contains elements 3, 1, 2
```

To add a single element to the end of an ArrayList, use the `add` function. To add multiple elements to the end at once, use the `add(all!: Collection<T>)` function, which can accept other Collection types with the same element type (such as Array). For details on the Collection type, see [Overview of Basic Collection Types](collection_overview.md).

<!-- run -->

```cangjie
import std.collection.ArrayList

main() {
    let list = ArrayList<Int64>()
    list.add(0) // list contains element 0
    list.add(1) // list contains elements 0, 1
    let li = [2, 3]
    list.add(all: li) // list contains elements 0, 1, 2, 3
}
```

You can insert a specified single element or a Collection value of the same element type at a specified index position using the `add(T, at!: Int64)` and `add(all!: Collection<T>, at!: Int64)` functions. The element at that index and subsequent elements will be shifted backward to make space.

<!-- run -->

```cangjie
let list = ArrayList<Int64>([0, 1, 2]) // list contains elements 0, 1, 2
list.add(4, at: 1) // list contains elements 0, 4, 1, 2
```

To delete an element from an ArrayList, use the `remove` function and specify the index of the element to delete. Elements after that index will be shifted forward to fill the space.

<!-- run -->

```cangjie
let list = ArrayList<String>(["a", "b", "c", "d"]) // list contains the elements "a", "b", "c", "d"
list.remove(at: 1) // Delete the element at subscript 1; now the list contains elements "a", "c", "d"
```

## Increasing the Size of ArrayList

Each ArrayList requires a specific amount of memory to store its contents. When elements are added to an ArrayList and it starts to exceed its reserved capacity, the ArrayList will allocate a larger memory area and copy all its elements to the new memory. This growth strategy means that add operations that trigger memory reallocation have a performance cost, but they occur less frequently as the ArrayList's reserved memory increases.

If you know approximately how many elements will be added, you can reserve sufficient memory in advance to avoid intermediate reallocations, which can improve performance.

<!-- run -->

```cangjie
import std.collection.ArrayList

main() {
    let list = ArrayList<Int64>(100) // Allocate space at once
    for (i in 0..100) {
        list.add(i) // Does not trigger reallocation of space
    }
    list.reserve(100) // Prepare more space
    for (i in 0..100) {
        list.add(i) // Does not trigger reallocation of space
    }
}
```

# Package Import

## Using `import` Statements to Import Declarations or Definitions from Other Packages

In the Cangjie programming language, you can import a top-level declaration or definition from another package using the syntax `import fullPackageName.itemName`, where `fullPackageName` is the fully qualified package name and `itemName` is the name of the declaration. The import statements must be placed after the package declaration and before any other declarations or definitions in the source file. For example:

```cangjie
package a
import std.math.*
import package1.foo
import {package1.foo, package2.bar}
```

If multiple `itemName`s belong to the same `fullPackageName`, you can use the syntax `import fullPackageName.{itemName[, itemName]*}`. For example:

```cangjie
import package1.{foo, bar, fuzz}
```

This is equivalent to:

```cangjie
import package1.foo
import package1.bar
import package1.fuzz
```

In addition to importing a specific top-level declaration or definition using `import fullPackagename.itemName`, you can also use `import packageName.*` to import all visible top-level declarations or definitions from the `packageName` package. For example:

```cangjie
import package1.*
import {package1.*, package2.*}
```

Note the following:

- The scope level of imported members is lower than that of members declared in the current package.
- If the module name or package name of an exported package is altered, making it inconsistent with the name specified during export, an error will occur during import.
- Only top-level declarations or definitions visible to the current file can be imported. Attempting to import invisible declarations or definitions will result in an error at the import statement.
- It is prohibited to import declarations or definitions from the package where the current source file resides using `import`.
- Circular dependency imports between packages are prohibited. If circular dependencies exist between packages, the compiler will report an error.

Example:

```cangjie
// pkga/a.cj
package pkga    // Error, packages pkga pkgb are in circular dependencies.
import pkgb.*

class C {}
public struct R {}

// pkgb/b.cj
package pkgb

import pkga.*

// pkgc/c1.cj
package pkgc

import pkga.C // Error, 'C' is not accessible in package 'pkga'.
import pkga.R // OK, R is an external top-level declaration of package pkga.
import pkgc.f1 // Error, package 'pkgc' should not import itself.

public func f1() {}

// pkgc/c2.cj
package pkgc

func f2() {
    /* OK, the imported declaration is visible to all source files of the same package
     * and accessing import declaration by its name is supported.
     */
    R()

    // OK, accessing imported declaration by fully qualified name is supported.
    pkga.R()

    // OK, the declaration of current package can be accessed directly.
    f1()

    // OK, accessing declaration of current package by fully qualified name is supported.
    pkgc.f1()
}
```

In the Cangjie programming language, if an imported declaration or definition has the same name as a top-level declaration or definition in the current package and does not constitute function overloading, the imported declaration or definition will be shadowed. If they do constitute function overloading, function resolution will follow the rules of function overloading during function calls.

```cangjie
// pkga/a.cj
package pkga

public struct R {}            // R1
public func f(a: Int32) {}    // f1
public func f(a: Bool) {} // f2

// pkgb/b.cj
package pkgb
import pkga.*

func f(a: Int32) {}         // f3
struct R {}                 // R2

func bar() {
    R()     // OK, R2 shadows R1.
    f(1)    // OK, invoke f3 in current package.
    f(true) // OK, invoke f2 in the imported package
}
```

## Implicit Import of the `core` Package

Types such as `String` and `Range` can be used directly not because they are built-in types, but because the compiler automatically and implicitly imports all `public` declarations from the `core` package for the source code.

## Using `import as` to Rename Imported Names

Different packages have separate namespaces, so top-level declarations with the same name may exist across different packages. When importing top-level declarations with the same name from different packages, you can use `import packageName.name as newName` to rename them and avoid conflicts. Even without name conflicts, you can still use `import as` to rename imported content. The rules for `import as` are as follows:

- After renaming an imported declaration using `import as`, only the new name can be used in the current package; the original name becomes unavailable.
- If the renamed name conflicts with other names in the top-level scope of the current package and all corresponding declarations are function types, they participate in function overloading; otherwise, a redefinition error is reported.
- The syntax `import pkg as newPkgName` is supported to rename package names, resolving naming conflicts for packages with the same name in different modules.
    <!-- compile.error -import3-->
    <!-- cfg="-p p1 --output-type=staticlib" -->

    ```cangjie
    // a.cj
    package p1
    public func f1() {}
    ```

    <!-- compile.error -import3-->
    <!-- cfg="-p p2 --output-type=staticlib" -->

    ```cangjie
    // d.cj
    package p2
    public func f3() {}
    ```

    <!-- compile.error -import3-->
    <!-- cfg="-p p1 --output-type=staticlib" -->

    ```cangjie
    // b.cj
    package p1
    public func f2() {}
    ```

    <!-- compile.error -import3-->
    <!-- cfg="-p pkgc --output-type=staticlib" -->

    ```cangjie
    // c.cj
    package pkgc
    public func f1() {}
    ```

    <!-- compile.error -import3-->
    <!-- cfg="libp1.a libpkgc.a libp1.a" -->

    ```cangjie
    // main.cj
    import p1 as A
    import p1 as B
    import p2.f3 as f  // OK
    import pkgc.f1 as a
    import pkgc.f1 as b // OK

    func f(a: Int32) {}

    main() {
        A.f1()  // OK, package name conflict is resolved by renaming package name.
        B.f2()  // OK, package name conflict is resolved by renaming package name.
        p1.f1() // Error, the original package name cannot be used.
        a()     // OK.
        b()     // OK.
        pkgc.f1()    // Error, the original name cannot be used.
    }
    ```

- If conflicting imported names are not renamed, no error is reported at the `import` statement. However, an error will occur at the usage site due to the inability to import a unique name. This can be resolved by defining aliases with `import as` or importing the package as a namespace using `import fullPackageName`.

    ```cangjie
    // a.cj
    package p1
    public class C {}

    // b.cj
    package p2
    public class C {}

    // main1.cj
    package pkga
    import p1.C
    import p2.C

    main() {
        let _ = C() // Error
    }

    // main2.cj
    package pkgb
    import p1.C as C1
    import p2.C as C2

    main() {
        let _ = C1() // OK
        let _ = C2() // OK
    }

    // main3.cj
    package pkgc
    import p1
    import p2

    main() {
        let _ = p1.C() // OK
        let _ = p2.C() // OK
    }
    ```

## Re-exporting an Imported Name

In large-scale projects with extensive functionality, this scenario is very common: package `p2` heavily uses declarations imported from package `p1`. When package `p3` imports `p2` and uses its functionality, the declarations from `p1` also need to be visible to `p3`. Requiring `p3` to manually import all `p1` declarations used by `p2` would be overly cumbersome. Therefore, it is desirable to import `p1` declarations used by `p2` when `p2` is imported.

In the Cangjie programming language, `import` can be modified with the access modifiers `private`, `internal`, `protected`, or `public`. Among these, `import` statements modified with `public`, `protected`, or `internal` can re-export the imported members (provided these members are not rendered unavailable in the current package due to name conflicts or shadowing). Other packages can directly import and use the re-exported content based on visibility without needing to import it from the original package.

- `private import` means the imported content is only accessible within the current file. `private` is the default modifier for `import`; an `import` without an access modifier is equivalent to `private import`.
- `internal import` means the imported content is accessible within the current package and its subpackages (including subpackages of subpackages). Access from outside the current package requires an explicit `import`.
- `protected import` means the imported content is accessible within the current module. Access from outside the current package requires an explicit `import`.
- `public import` means the imported content is accessible externally. Access from outside the current package requires an explicit `import`.

In the following example, `b` is a subpackage of `a`, and `a` re-exports the function `f` defined in `b` using `public import`.

```cangjie
package a
public import a.b.f

public let x = 0
```

```cangjie
internal package a.b

public func f() { 0 }
```

```cangjie
import a.f  // OK
let _ = f() // OK
```

Note that packages cannot be re-exported: if the `import` statement imports a package, it cannot be modified with `public`, `protected`, or `internal`.

<!-- compile.error -->

```cangjie
public import a.b // Error, cannot re-export package
```
# _Kotlin/Native_ interoperability with Swift/Objective-C

This document covers some details of Kotlin/Native interoperability with
Swift/Objective-C.

## Usage

Kotlin/Native provides bidirectional interoperability with Objective-C.
Objective-C frameworks and libraries can be used in Kotlin code if
properly imported to the build (system frameworks are imported by default).
See e.g. "Interop libraries" in
[Gradle plugin documentation](GRADLE_PLUGIN.md#building-artifacts).
A Swift library can be used in Kotlin code if its API is exported to Objective-C
with `@objc`. Pure Swift modules are not yet supported.

Kotlin modules can be used in Swift/Objective-C code if compiled into a
[framework](GRADLE_PLUGIN.md#framework). See [calculator sample](https://github.com/JetBrains/kotlin-native/tree/master/samples/calculator)
for an example.

## Mappings

The table below shows how Kotlin concepts are mapped to Swift/Objective-C and vice versa.

| Kotlin | Swift | Objective-C | Notes |
| ------ | ----- |------------ | ----- |
| `class` | `class` | `@interface` | [note](#name-translation) |
| `interface` | `protocol` | `@protocol` | |
| `constructor`/`create` | Initializer | Initializer | [note](#initializers) |
| Property | Property | Property | [note](#top-level-functions-and-properties) |
| Method | Method | Method | [note](#top-level-functions-and-properties) [note](#method-names-translation) |
| `@Throws` | `throws` | `error:(NSError**)error` | [note](#errors-and-exceptions) |
| Extension | Extension | Category member | [note](#category-members) |
| `companion` member <- | Class method or property | Class method or property |  |
| `null` | `nil` | `nil` | |
| `Singleton` | `Singleton()`  | `[Singleton singleton]` | [note](#kotlin-singletons) |
| Primitive type | Primitive type / `NSNumber` | | [note](#nsnumber) |
| `Unit` return type | `Void` | `void` | |
| `String` | `String` | `NSString` | |
| `String` | `NSMutableString` | `NSMutableString` | [note](#nsmutablestring) |
| `List` | `Array` | `NSArray` | |
| `MutableList` | `NSMutableArray` | `NSMutableArray` | |
| `Set` | `Set` | `NSSet` | |
| `MutableSet` | `NSMutableSet` | `NSMutableSet` | [note](#collections) |
| `Map` | `Dictionary` | `NSDictionary` | |
| `MutableMap` | `NSMutableDictionary` | `NSMutableDictionary` | [note](#collections) |
| Function type | Function type | Block pointer type | [note](#function-types) |

### Name translation

Objective-C classes are imported into Kotlin with their original names.
Protocols are imported as interfaces with `Protocol` name suffix,
i.e. `@protocol Foo` -> `interface FooProtocol`.
These classes and interfaces are placed into a package [specified in build configuration](#usage)
(`platform.*` packages for preconfigured system frameworks).

The names of Kotlin classes and interfaces are prefixed when imported to Swift/Objective-C.
The prefix is derived from the framework name.

### Initializers

Swift/Objective-C initializers are imported to Kotlin as constructors and factory methods
named `create`. The latter happens with initializers declared in the Objective-C category or
as a Swift extension, because Kotlin has no concept of extension constructors.

Kotlin constructors are imported as initializers to Swift/Objective-C. 

### Top-level functions and properties

Top-level Kotlin functions and properties are accessible as members of a special class.
Each Kotlin package is translated into such a class. E.g.

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
package my.library

fun foo() {}
```

</div>

can be called from Swift like

<div class="sample" markdown="1" theme="idea" mode="swift">

```swift
Framework.foo()
```

</div>

### Method names translation

Generally Swift argument labels and Objective-C selector pieces are mapped to Kotlin
parameter names. Anyway these two concepts have different semantics, so sometimes
Swift/Objective-C methods can be imported with a clashing Kotlin signature. In this case
the clashing methods can be called from Kotlin using named arguments, e.g.:

<div class="sample" markdown="1" theme="idea" mode="swift">

```swift
[player moveTo:LEFT byMeters:17]
[player moveTo:UP byInches:42]
```

</div>

in Kotlin it would be:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
player.moveTo(LEFT, byMeters = 17)
player.moveTo(UP, byInches = 42)
```

</div>

### Errors and exceptions

Kotlin has no concept of checked exceptions, all Kotlin exceptions are unchecked.
Swift has only checked errors. So if Swift or Objective-C code calls a Kotlin method
which throws an exception to be handled, then the Kotlin method should be marked
with a `@Throws` annotation. In this case all Kotlin exceptions
(except for instances of `Error`, `RuntimeException` and subclasses) are translated into
a Swift error/`NSError`.

Note that the opposite reversed translation is not implemented yet:
Swift/Objective-C error-throwing methods aren't imported to Kotlin as
exception-throwing.

### Category members

Members of Objective-C categories and Swift extensions are imported to Kotlin
as extensions. That's why these declarations can't be overridden in Kotlin.
And the extension initializers aren't available as Kotlin constructors.

### Kotlin singletons

Kotlin singleton (made with an `object` declaration, including `companion object`)
is imported to Swift/Objective-C as a class with a single instance.
The instance is available through the factory method, i.e. as
`[MySingleton mySingleton]` in Objective-C and `MySingleton()` in Swift.

### NSNumber

While Kotlin primitive types in some cases are mapped to `NSNumber`
(e.g. when they are boxed), `NSNumber` type is not automatically translated 
to Kotlin primitive types when used as a Swift/Objective-C parameter type or return value.
The reason is that `NSNumber` type doesn't provide enough information
about a wrapped primitive value type, i.e. `NSNumber` is statically not known
to be a e.g. `Byte`, `Boolean`, or `Double`. So Kotlin primitive values 
should be cast to/from `NSNumber` manually (see [below](#casting-between-mapped-types)).

### NSMutableString

`NSMutableString` Objective-C class is not available from Kotlin.
All instances of `NSMutableString` are copied when passed to Kotlin.

### Collections

Kotlin collections are converted to Swift/Objective-C collections as described
in the table above. Swift/Objective-C collections are mapped to Kotlin in the same way,
except for `NSMutableSet` and `NSMutableDictionary`. `NSMutableSet` isn't converted to
a Kotlin `MutableSet`. To pass an object for Kotlin `MutableSet`,
you can create this kind of Kotlin collection explicitly by either creating it 
in Kotlin with e.g. `mutableSetOf()`, or using the `${prefix}MutableSet` class in
Swift/Objective-C, where `prefix` is the framework names prefix.
The same holds for `MutableMap`.

### Function types

Kotlin function-typed objects (e.g. lambdas) are converted to 
Swift functions / Objective-C blocks. However there is a difference in how
types of parameters and return values are mapped when translating a function
and a function type. In the latter case primitive types are mapped to their
boxed representation, `NSNumber`. Kotlin `Unit` return value is represented
as a corresponding `Unit` singleton in Swift/Objective-C. The value of this singleton
can be retrieved in the same way as it is for any other Kotlin `object`
(see singletons in the table above).
To sum the things up:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun foo(block: (Int) -> Unit) { ... }
```

</div>

would be represented in Swift as

<div class="sample" markdown="1" theme="idea" mode="swift">

```swift
func foo(block: (NSNumber) -> KotlinUnit)
```

</div>

and can be called like

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
foo {
    bar($0 as! Int32)
    return KotlinUnit()
}
```

</div>

## Casting between mapped types

When writing Kotlin code, an object may need to be converted from a Kotlin type
to the equivalent Swift/Objective-C type (or vice versa). In this case a plain old
Kotlin cast can be used, e.g.

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
val nsArray = listOf(1, 2, 3) as NSArray
val string = nsString as String
val nsNumber = 42 as NSNumber
```

</div>

## Subclassing

### Subclassing Kotlin classes and interfaces from Swift/Objective-C

Kotlin classes and interfaces can be subclassed by Swift/Objective-C classes
and protocols.
Currently a class that adopts the Kotlin protocol should inherit `NSObject`
(either directly or indirectly). Note that all Kotlin classes do inherit `NSObject`,
so a Swift/Objective-C subclass of Kotlin class can adopt the Kotlin protocol.

### Subclassing Swift/Objective-C classes and protocols from Kotlin

Swift/Objective-C classes and protocols can be subclassed with a Kotlin `final` class.
Non-`final` Kotlin classes inheriting Swift/Objective-C types aren't supported yet, so it is
not possible to declare a complex class hierarchy inheriting Swift/Objective-C types.

Normal methods can be overridden using the `override` Kotlin keyword. In this case
the overriding method must have the same parameter names as the overridden one.

Sometimes it is required to override initializers, e.g. when subclassing `UIViewController`. 
Initializers imported as Kotlin constructors can be overridden by Kotlin constructors
marked with the `@OverrideInit` annotation:

<div class="sample" markdown="1" theme="idea" mode="swift">

```swift
class ViewController : UIViewController {
    @OverrideInit constructor(coder: NSCoder) : super(coder)

    ...
}
```

</div>

The overriding constructor must have the same parameter names and types as the overridden one.

To override different methods with clashing Kotlin signatures, you can add a
`@Suppress("CONFLICTING_OVERLOADS")` annotation to the class.

By default the Kotlin/Native compiler doesn't allow calling a non-designated
Objective-C initializer as a `super(...)` constructor. This behaviour can be
inconvenient if the designated initializers aren't marked properly in the Objective-C
library. Adding a `disableDesignatedInitializerChecks = true` to the `.def` file for
this library would disable these compiler checks.

## C features

See [INTEROP.md](INTEROP.md) for an example case where the library uses some plain C features
(e.g. unsafe pointers, structs etc.).

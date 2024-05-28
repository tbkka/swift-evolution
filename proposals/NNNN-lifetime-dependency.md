# Compile-time Lifetime Dependency Annotations

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Andrew Trick](https://github.com/atrick), [Meghana Gupta](https://github.com/meg-gupta), [Tim Kientzle](https://github.com/tbkka)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Review: ([pitch](https://forums.swift.org/t/pitch-non-escapable-types-and-lifetime-dependency/69865))

## Introduction

We would like to propose extensions to Swift's function-declaration syntax that allow authors to specify lifetime dependencies between the return value and one or more of the parameters.
These would also be useable with methods that wish to declare a dependency on `self`.
To reduce the burden of manually adding such annotations, we also propose inferring lifetime dependencies in certain common cases without requiring any additional annotations.

This is a key requirement for the `Span` type (previously called `BufferView`) being discussed elsewhere, and is closely related to the proposal for `~Escapable` types.

**Edited** (Apr 12, 2024): Changed `@dependsOn` to `dependsOn` to match the current implementation.

**Edited** (May 2, 2024): Changed `StorageView` and `BufferReference` to `Span` to match the sibling proposal.

#### See Also

* [Forum discussion of Non-Escapable Types and Lifetime Dependency](https://forums.swift.org/t/pitch-non-escapable-types-and-lifetime-dependency)
* [Pitch Thread for Span](https://forums.swift.org/t/pitch-safe-access-to-contiguous-storage/69888)
* [Forum discussion of BufferView language requirements](https://forums.swift.org/t/roadmap-language-support-for-bufferview)
* [Proposed Vision document for BufferView language requirements (includes description of ~Escapable)](https://github.com/atrick/swift-evolution/blob/fd63292839808423a5062499f588f557000c5d15/visions/language-support-for-BufferView.md#non-escaping-bufferview) 

## Motivation

An efficient way to provide one piece of code with temporary access to data stored in some other piece of code is with a pointer to the data in memory.
Swift's `Unsafe*Pointer` family of types can be used here, but as the name implies, using these types can be error-prone.

For example, suppose `Array` had a property `unsafeBufferPointer` that returned an `UnsafeBufferPointer` to the contents of the array.
Here's an attempt to use such a property:

```swift
let array = getArrayWithData()
let buff = array.unsafeBufferPointer
parse(buff) // <== 🛑 NOT SAFE!
```

One reason for this unsafety is because Swift's standard lifetime rules only apply to individual values.
They cannot guarantee that `buff` will outlive the `array`, which means there is a risk that the compiler might choose to destroy `array` before the call to `parse`, which could result in `buff` referencing deallocated memory.
(There are other reasons that this specific example is unsafe, but the lifetime issue is the one that specifically concerns us here.)

Library authors trying to support this kind of code pattern today have a few options, but none are entirely satisfactory:

* The client developer can manually insert `withExtendedLifetime` and similar annotations to control the lifetime of specific objects.
  This is awkward and error-prone.
  We would prefer a mechanism where the library author can declare the necessary semantics and have the compiler automatically enforce them.
* The library author can store a back-reference to the container as part of their "pointer" or "slice" object.
  However, this incurs reference counting overhead which sacrifices some of the performance gains that pointer-based designs are generally intended to provide.
  In addition, this approach is not possible in environments that lack support for dynamic allocation.
* The library author can make the pointer information available only within a scoped function, but this is also unsafe, as demonstrated by well-meaning developers who extract the pointer out of such functions using code like that below.
  Even when used correctly, scoped functions can lead to a pyramid of deeply-indented code blocks.

```swift
// 🛑 The following line of code is dangerous!  DO NOT DO THIS!
let buff = array.withUnsafeBufferPointer { $0 }
```

## Proposed solution

A "lifetime dependency" between two objects indicates that one of them can only be destroyed *after* the other.
This dependency is enforced entirely at compile time; it requires no run-time support.
These lifetime dependencies can be expressed in several different ways, with varying trade-offs of expressiveness and ease-of-use.

### Background: "Escapable" and “Nonescapable” Types

In order to avoid changing the meaning of existing code, we will introduce a
new protocol `Escapable` which can be suppressed with `~Escapable`.

Normal Swift types are `Escapable` by default.
This implies that they can be returned, stored in properties, or otherwise "escape" the local context.
Conversely, types can be explicitly declared to be "nonescapable" using `~Escapable`.
These types are not allowed to escape the local context except in very specific circumstances.
A separate proposal explains the general syntax and semantics of `Escapable` and `~Escapable`.

By themselves, nonescapable types have severe constraints on usage.
For example, consider a hypothetical `Span` type that is similar type that is being proposed for inclusion in the standard library. It simply holds a pointer and size and can be used to access data stored in a contiguous block of memory. (We are not proposing this type; it is shown here merely for illustrative purposes.)

```swift
struct Span<T>: ~Escapable {
  private var base: UnsafePointer<T>
  private var count: Int
}
```

Because this type is marked as unconditionally `~Escapable`, it cannot be returned from a function or even initialized without some way to relax the escapability restrictions.
This proposal provides a set of constraints that can tie the lifetime of a nonescapable value to the lifetime of some other value.
In the most common cases, these constraints can be inferred automatically.

### Explicit Lifetime Dependency Annotations

To make the semantics clearer, we’ll begin by describing how one can explicitly specify a lifetime constraint in cases where the default inference rules do not apply.

Let’s consider adding support for our hypothetical `Span` type to `Array`.
Our proposal would allow you to declare an `array.span()` method as follows:

```swift
extension Array {
  borrowing func span() -> dependsOn(self) Span<Element> {
    ... construct a Span ...
  }
}
```

The annotation `dependsOn(self)` here indicates that the returned value must not outlive the array that produced it.
Conceptually, it is a continuation of the function's borrowing access:
the array is being borrowed by the function while the function executes and then continues to be borrowed by the `Span` for as long as the return value exists.
Specifically, the `dependsOn(self)` annotation in this example informs the compiler that:

* The array must not be destroyed until after the `Span<Element>` is destroyed.
  This ensures that use-after-free cannot occur.
* The array must not be mutated while the  `Span<Element>` value exists.
  This follows the usual Swift exclusivity rules for a borrowing access.

#### Scoped Lifetime Dependency

Let’s consider another hypothetical type: a `MutatingSpan<T>` type that could provide indirect mutating access to a block of memory.
Here's one way such a value might be produced:

```swift
func mutatingSpan(to: inout Array, count: Int) -> dependsOn(to) MutatingSpan<Element> {
  ... construct a MutatingSpan ...
}
```

We’ve written this example as a free function rather than as a method to show how this annotation syntax can be used to express constraints that apply to a particular argument.
The `dependsOn(to)` annotation indicates that the returned value depends on the argument named `to`.
Because `count` is not mentioned in the lifetime dependency, that argument does not participate.
Similar to the previous example:

* The array will not be destroyed until after the `MutatingSpan<Element>` is destroyed.
* No other read or write access to the array will be allowed for as long as the returned value exists.

In both this and the previous case, the lifetime of the return value is "scoped" to the lifetime of the original value.
Because lifetime dependencies can only be attached to nonescapable values, types that contain pointers will generally need to be nonescapable in order to provide safe semantics.
As a result, **scoped lifetime dependencies** are the only possibility whenever an `Escapable` value (such as an Array or similar container) is providing a nonescapable value (such as the `Span` or `MutatingSpan` in these examples).

#### Copied Lifetime Dependency

The case where a nonescapable value is used to produce another nonescapable value is somewhat different.
Here's a typical example that constructs a new `Span` from an existing one:
```swift
struct Span<T>: ~Escapable {
  ...
  consuming func drop(_: Int) -> dependsOn(self) Span<T> { ... }
  ...
}
```

In this examples, the nonescapable result depends on a nonescapable value.
Recall that nonescapable values such as these represent values that are already lifetime-constrained to another value.

For a `consuming` method, the return value cannot have a scoped lifetime dependency on the original value, since the original value no longer exists when the method returns.
Instead, the return value must "copy" the lifetime dependency from the original:
If the original `Span` was borrowing some array, the new `Span` will continue to borrow the same array.

This supports coding patterns such as this:
```swift
let a: Array<Int>
let ref1 = a.span() // ref1 cannot outlive a
let ref2 = ref1.drop(4) // ref2 also cannot outlive a
```

After `ref1.drop(4)`, the lifetime of `ref2` does not depend on `ref1`.
Rather, `ref2` has **inherited** or **copied** `ref1`’s dependency on the lifetime of `a`.

#### Allowed Lifetime Dependencies

The previous sections described **scoped lifetime dependencies** and **copied lifetime dependencies**
and showed how each type occurs naturally in different use cases.

Now let's look at the full range of possibilities for explicit constraints.
The syntax is somewhat different for functions and methods, though the basic rules are essentially the same.

**Functions:** A simple function with an explicit lifetime dependency annotation generally takes this form:

```swift
func f(arg: <parameter-convention> ArgType) -> dependsOn(arg) ResultType
```

Where

*  *`parameter-convention`* is one of the ownership specifiers **`borrowing`**, **`consuming`**, or **`inout`**, (this may be implied by Swift’s default parameter ownership rules),
* `ResultType` must be nonescapable.

If the `ArgType` is escapable, the return value will have a new scoped dependency on the argument.
(This is the only possibility, as an escapable value cannot have an existing lifetime dependency,
so we cannot copy the lifetime dependency.)
A scoped dependency ensures the argument will not be destroyed while the result is alive.
Also, access to the argument will be restricted for the lifetime of the result following Swift's usual exclusivity rules:

* A `borrowing` parameter-convention extends borrowing access, prohibiting mutations of the argument.
* An `inout` parameter-convention extends mutating access, prohibiting any access to the argument.
* A `consuming` parameter-convention is illegal, since that ends the lifetime of the argument immediately.

If the `ArgType` is nonescapable, then it can have a pre-existing lifetime dependency.
In this case, the semantics of `dependsOn()` are slightly different:
* A `consuming` parameter-convention will copy the lifetime dependency from the argument to the result
* A `borrowing` or `inout` parameter-convention can either copy the lifetime dependency or create a new scoped lifetime dependency.
  In this case, for reasons explained earlier, we default to copying the lifetime dependency.
  If a scoped lifetime dependency is needed, it can be explicitly requested by adding the `scoped` keyword:
  
```swift
func f(arg: borrowing ArgType) -> dependsOn(scoped arg) ResultType
```

**Methods:** Similar rules apply to `self` lifetime dependencies on methods.
Given a method of this form:

```swift
<mutation-modifier> func method(... args ...) -> dependsOn(self) ResultType
```

The behavior depends as above on the mutation-modifier and whether the defining type is escapable or nonescapable.

**Initializers:** An initializer can define lifetime dependencies on one or more arguments.
In this case, we use the same rules as for “Functions” above
by using the convention that initializers can be viewed as functions that return `Self`:

```swift
init(arg: <parameter-convention> ArgType) -> dependsOn(arg) Self
```

### Implicit Lifetime Dependencies

The syntax above allows developers to explicitly annotate lifetime dependencies in their code.
But because the possibilities are limited, we can usually allow the compiler to infer a suitable dependency.
The detailed rules are below, but generally we require that the return type be nonescapable and that there be one “obvious” source for the dependency.

In particular, we can infer a lifetime dependency on `self` for any method that returns a nonescapable value.
As above, the details vary depending on whether `self` is escapable or nonescapable:

```swift
struct NonescapableType: ~Escapable { ... }
struct EscStruct {
  func f1(...) -> /* dependsOn(self) */ NonescapableType
  borrowing func f2(...) -> /* dependsOn(self) */ NonescapableType
  mutating func f3(...) -> /* dependsOn(self) */ NonescapableType

  // 🛑 Error: there is no valid lifetime dependency for
  // a consuming method on an `Escapable` type
  consuming func f4(...) -> NonescapableType
}

struct NEStruct: ~Escapable {
  func f1(...) -> /* dependsOn(self) */ NonescapableType
  borrowing func f2(...) -> /* dependsOn(self) */ NonescapableType
  mutating func f3(...) -> /* dependsOn(self) */ NonescapableType

  // Note: A copied lifetime dependency is legal here
  consuming func f4(...) -> /* dependsOn(self) */ NonescapableType
}
```

For free or static functions or initializers, we can infer a lifetime dependency when the return value is nonescapable and there is only one obvious argument that can serve as the source of the dependency.
For example:

```swift
struct NEType: ~Escapable { ... }

// If there is only one argument with an explicit parameter convention:
func f(..., arg1: borrowing Type1, ...) -> /* dependsOn(arg1) */ NEType

// Or there is only one argument that is `~Escapable`:
func g(..., arg2: NEType, ...) -> /* dependsOn(arg2) */ NEType

// If there are multiple possible arguments that we might depend
// on, we require an explicit dependency:
// 🛑 Cannot infer lifetime dependency since `arg1` and `arg2` are both candidates
func g(... arg1: borrowing Type1, arg2: NEType, ...) -> NEType
```

We expect these implicit inferences to cover most cases, with the explicit form only occasionally being necessary in practice.

### Dependent parameters

Normally, lifetime dependence is required when a nonescapable function result depends on an argument to that function. In some rare cases, however, a nonescapable function parameter may depend on another argument to that function. Consider a function with an `inout` parameter. The function body may reassign that parameter to a value that depends on another parameter. This is similar in principle to a result dependence.

```swift
func mayReassign(span: dependsOn(a) inout [Int], to a: [Int]) {
  span = a.span()
}
```

A `selfDependsOn` keyword is required to indicate that a method's implicit `self` depends on another parameter.

```swift
extension Span {
  mutating selfDependsOn(other) func reassign(other: Span<T>) {
    self = other // ✅ OK: 'self' depends on 'other'
  }
}
```

We've discussed how a nonescapable result must be destroyed before the source of its lifetime dependence. Similarly, a dependent argument must be destroyed before an argument that it depends on. The difference is that the dependent argument may already have a lifetime dependence when it enters the function. The new function argument dependence is additive, because the call does not guarantee reassignment. Instead, passing the 'inout' argument is like a conditional reassignment. After the function call, the dependent argument carries both lifetime dependencies.

```swift
  let a1: Array<Int> = ...
  var span = a1.span()
  let a2: Array<Int> = ...
  mayReassign(span: &span, to: a2)
  // 'span' now depends on both 'a1' and 'a2'.
```

### Dependent properties

Structural composition is an important use case for nonescapable types. Getting or setting a nonescapable property requires lifetime dependence, just like a function result or an 'inout' parameter. There's no need for explicit annotation in these cases, because only one dependence is possible. A getter returns a value that depends on `self`. A setter replaces the current dependence from `self` with a dependence on `newValue`.

```swift
struct Container<Element>: ~Escapable {
  var element: Element {
    /* dependsOn(self) */ get { ... }
    /* selfDependsOn(newValue) */ set { ... }
  }

  init(element: Element) /* -> dependsOn(element) Self */ {...}
}
```

### Conditional dependencies

Conditionally nonescapable types can contain nonescapable elements:

```swift
    struct Container<Element>: ~Escapable {
      var element: /* dependsOn(self) */ Element

      init(element: Element) -> dependsOn(element) Self {...}

      func getElement() -> dependsOn(self) Element { element }
    }

    extension Container<E> { // OK: conforms to Escapable.
      // Escapable context...
    }
```

Here, `Container` becomes nonescapable only when its element type is nonescapable. When `Container` is nonescapable, it inherits the lifetime of its single element value from the initializer and propagates that lifetime to all uses of its `element` property or the `getElement()` function.

In some contexts, however, `Container` and `Element` both conform to `Escapable`. In those contexts, any `dependsOn` in `Container`'s interface is ignored, whether explicitly annotated or implied. So, when `Container`'s element conforms to `Escapable`, the `-> dependsOn(element) Self` annotation in its initializer is ignored, and the `-> dependsOn(self) Element` in `getElement()` is ignored.

### Immortal lifetimes

In some cases, a nonescapable value must be constructed without any object that can stand in as the source of a dependence. Consider extending the standard library `Optional` or `Result` types to be conditionally escapable:

```swift
enum Optional<Wrapped: ~Escapable>: ~Escapable {
  case none, some(Wrapped)
}

extension Optional: Escapable where Wrapped: Escapable {}

enum Result<Success: ~Escapable, Failure: Error>: ~Escapable {
  case failure(Failure), success(Success)
}

extension Result: Escapable where Success: Escapable {}
```

When constructing an `Optional<NotEscapable>.none` or `Result<NotEscapable>.failure(error)` case, there's no lifetime to assign to the constructed value in isolation, and it wouldn't necessarily need one for safety purposes, because the given instance of the value doesn't store any state with a lifetime dependency. Instead, the initializer for cases like this can be annotated with `dependsOn(immortal)`:

```swift
extension Optional {
  init(nilLiteral: ()) dependsOn(immortal) {
    self = .none
  }
}
```

Once the escapable instance is constructed, it is limited in scope to the caller's function body since the caller only sees the static nonescapable type. If a dynamically escapable value needs to be returned further up the stack, that can be done by chaining multiple `dependsOn(immortal)` functions.

#### Depending on immutable global variables

Another place where immortal lifetimes might come up is with dependencies on global variables. When a value has a scoped dependency on a global let constant, that constant lives for the duration of the process and is effectively perpetually borrowed, so one could say that values dependent on such a constant have an effectively infinite lifetime as well. This will allow returning a value that depends on a global by declaring the function's return type with `dependsOn(immortal)`:

```swift
let staticBuffer = ...

func getStaticallyAllocated() -> dependsOn(immortal) BufferReference {
  staticBuffer.bufferReference()
}
```

### Depending on an escapable `BitwiseCopyable` value

The source of a lifetime depenence may be an escapable `BitwiseCopyable` value. This is useful in the implementation of data types that internally use `UnsafePointer`:

```swift
struct Span<T>: ~Escapable {
  ...
  // The caller must ensure that `unsafeBaseAddress` is valid over all uses of the result.
  init(unsafeBaseAddress: UnsafePointer<T>, count: Int) dependsOn(unsafeBaseAddress) { ... }
  ...
}
```

By convention, when the source of a dependence is escapable and `BitwiseCopyable`, it should have an "unsafe" label, such as `unsafeBaseAddress` above. This communicates to anyone who calls the function, that they are reponsibile for ensuring that the value that the result depends on is valid over all uses of the result. The compiler can't guarantee safety because `BitwiseCopyable` types do not have a formal point at which the value is destroyed. Specifically, for `UnsafePointer`, the compiler does not know which object owns the pointed-to storage.

```swift
var span: Span<T>?
let buffer: UnsafeBufferPointer<T>
do {
  let storage = Storage(...)
  buffer = storage.buffer
  span = Span(unsafeBaseAddress: buffer.baseAddress!, count: buffer.count)
  // 🔥 'storage' may be destroyed
}
decode(span!) // 👿 Undefined behavior: dangling pointer
```

Normally, `UnsafePointer` lifetime guarantees naturally fall out of closure-taking APIs that use `withExtendedLifetime`:

```swift
extension Storage {
  public func withUnsafeBufferPointer<R>(
    _ body: (UnsafeBufferPointer<Element>) throws -> R
  ) rethrows -> R {
    withExtendedLifetime (self) { ... }
  }
}

let storage = Storage(...)
storage.withUnsafeBufferPointer { buffer in
  let span = Span(unsafeBaseAddress: buffer.baseAddress!, count: buffer.count)
  decode(span!) // ✅ Safe: 'buffer' is always valid within the closure.
}
```

### Standard library extensions

#### Conditionally nonescapable types

The following standard library types will become conditionally nonescapable: `Optional`, `ExpressibleByNilLiteral`, and `Result`.

`MemoryLayout` will suppress the escapable constraint on its generic parameter.

#### `unsafeLifetime` helper functions

The following two helper functions will be added for implementing low-level data types:

```swift
/// Replace the current lifetime dependency of `dependent` with a new copied lifetime dependency on `source`.
///
/// Precondition: `dependent` has an independent copy of the dependent state captured by `source`.
func unsafeLifetime<T: ~Copyable & ~Escapable, U: ~Copyable & ~Escapable>(
  dependent: consuming T, dependsOn source: borrowing U)
  -> dependsOn(source) T { ... }

/// Replace the current lifetime dependency of `dependent` with a new scoped lifetime dependency on `source`.
///
/// Precondition: `dependent` depends on state that remains valid until either:
/// (a) `source` is either destroyed if it is immutable,
/// or (b) exclusive to `source` access ends if it is a mutable variable.
func unsafeLifetime<T: ~Copyable & ~Escapable, U: ~Copyable & ~Escapable>(
  dependent: consuming T, scoped source: borrowing U)
  -> dependsOn(scoped source) T {...}
```

These are useful for nonescapable data types that are internally represented using escapable types such as `UnsafePointer`. For example, some methods on `Span` will need to derive a new `Span` object that copies the lifetime dependence of `self`:

```swift
extension Span {
  consuming func dropFirst() -> Span<T> {
    let local = Span(base: self.base + 1, count: self.count - 1)
    // 'local' can persist after 'self' is destroyed.
    return unsafeLifetime(dependent: local, dependsOn: self)
  }
}
```

Since `self.base` is an escapable value, it does not propagate the lifetime dependence of its container. Without the call to `unsafeLifetime`, `local` would be limited to the local scope of the value retrieved from `self.base`, and could not be returned from the method. In this example, `unsafeLifetime` communicates that all of the dependent state from `self` has been *copied* into `local`, and, therefore, `local` can persist after `self` is destroyed.

## Detailed design

### Relation to ~Escapable

The lifetime dependencies described in this document can be applied only to nonescapable return values.
Further, any return value that is nonescapable must have a lifetime dependency.
In particular, this implies that the initializer for a nonescapable type must have at least one argument.

```swift
struct S: ~Escapable {
  init() {} // 🛑 Error: ~Escapable return type must have lifetime dependency
}
```

### Basic Semantics

A lifetime dependency annotation creates a *lifetime dependency* between a *dependent value* and a *source value*.
This relationship obeys the following requirements:

* The dependent value must be nonescapable.

* The dependent value's lifetime must not be longer than that of the source value.

* The dependent value is treated as an ongoing access to the source value.
  Following Swift's usual exclusivity rules, the source value may not be mutated during the lifetime of the dependent value;
  if the access is a mutating access, the source value is further prohibited from being accessed at all during the lifetime of the dependent value.

The compiler must issue a diagnostic if any of the above cannot be satisfied.

### Grammar


TODO:

* dependent parameters
* 'immortal' keyword

This new syntax adds an optional lifetime modifier just before the return type.
This modifies *function-result* in the Swift grammar as follows:

>
> *function-signature* → *parameter-clause* **`async`***?* **`throws`***?* *function-result**?* \
> *function-signature* → *parameter-clause* **`async`***?* **`rethrows`** *function-result**?* \
> *function-result* → **`->`** *attributes?* *lifetime-modifiers?* *type* \
> *lifetime-modifiers* → **`->`** *lifetime-modifier* *lifetime-modifiers?* \
> *lifetime-modifier* → **`->`** **`dependsOn`** **`(`** *lifetime-dependent* **`)`** \
> *lifetime-dependent* → **`->`** **`self`** | *local-parameter-name* | **`scoped self`** | **`scoped`** *local-parameter-name*
>

The *lifetime-dependent* argument to the lifetime modifier is one of the following:

* *local-parameter-name*: the local name of one of the function parameters, or
* the token **`self`**, or
* either of the above preceded by the **`scoped`** keyword

This modifier creates a lifetime dependency with the return value used as the dependent value.
The return value must be nonescapable.

The source value of the resulting dependency can vary.
In some cases, the source value will be the named parameter or `self` directly.
However, if the corresponding named parameter or `self` is nonescapable, then that value will itself have an existing lifetime dependency and thus the new dependency might "copy" the source of that existing dependency.

The following table summarizes the possibilities, which depend on the type and mutation modifier of the argument or `self` and the existence of the `scoped` keyword.
Here, "scoped" indicates that the dependent gains a direct lifetime dependency on the named parameter or `self` and "copied" indicates that the dependent gains a lifetime dependency on the source of an existing dependency:

| mutation modifier  | argument type | without `scoped` | with `scoped` |
| ------------------ | ------------- | ---------------- | ------------- |
| borrowed           | escapable     | scoped           | scoped        |
| inout or mutating  | escapable     | scoped           | scoped        |
| consuming          | escapable     | Illegal          | Illegal       |
| borrowed           | nonescapable  | copied           | scoped        |
| inout or mutating  | nonescapable  | copied           | scoped        |
| consuming          | nonescapable  | copied           | Illegal       |

Two observations may help in understanding the table above:
* An escapable argument cannot have a pre-existing lifetime dependency, so copying is never possible in those cases.
* A consumed argument cannot be the source of a lifetime dependency that will outlive the function call, so only copying is legal in that case.

**Note**: In practice, the `scoped` modifier keyword is likely to be only rarely used.  The rules above were designed to support the known use cases without requiring such a modifier.

#### Initializers

Since nonescapable values cannot be returned without a lifetime dependency,
initializers for such types must specify a lifetime dependency on one or more arguments.
We propose allowing initializers to write out an explicit return clause for this case, which permits the use of the same syntax as functions or methods.
The return type must be exactly the token `Self` or the token sequence `Self?` in the case of a failable initializer:

```swift
struct S {
  init(arg1: Type1) -> dependsOn(arg1) Self
  init?(arg2: Type2) -> dependsOn(arg2) Self?
}
```

> Grammar of an initializer declaration:
>
> *initializer-declaration* → *initializer-head* *generic-parameter-clause?* *parameter-clause* **`async`***?* **`throws`***?* *initializer-lifetime-modifier?* *generic-where-clause?* *initializer-body* \
> *initializer-declaration* → *initializer-head* *generic-parameter-clause?* *parameter-clause* **`async`***?* **`rethrows`** *initializer-lifetime-modifier?* *generic-where-clause?* *initializer-body* \
> *initializer-lifetime-modifier* → `**->**` *lifetime-modifiers* ** **`Self`** \
> *initializer-lifetime-modifier* → `**->**` *lifetime-modifiers* ** **`Self?`**

The implications of mutation modifiers and argument type on the resulting lifetime dependency exactly follow the rules above for functions and methods.

### Inference Rules

If there is no explicit lifetime dependency, we will automatically infer one according to the following rules:

**For methods where the return value is nonescapable**, we will infer a dependency against self, depending on the mutation type of the function.
Note that this is not affected by the presence, type, or modifier of any other arguments to the method.

**For a free or static functions or initializers with at least one argument,** we will infer a lifetime dependency when all of the following are true:

* the return value is nonescapable,
* there is exactly one argument that satisfies any of the following:
  - is either noncopyable or nonescapable, or
  - has an explicit `borrowing`, `consuming`, or `inout` convention specified

In this case, the compiler will infer a dependency on the unique argument identified by this last set of conditions.

**In no other case** will a function, method, or initializer implicitly gain a lifetime dependency.
If a function, method, or initializer has a nonescapable return value, does not have an explicit lifetime dependency annotation, and does not fall into one of the cases above, then that will be a compile-time error.


### Dependency semantics by example

This section illustrates the semantics of lifetime dependence one example at a time for each interesting variation. The following helper functions will be useful: `Array.span()` creates a scoped dependence to a nonescapable `Span` result, `copySpan()` creates a copied dependence to a `Span` result, and `parse` uses a `Span`.

```swift
extension Array {
  // The returned span depends on the scope of Self.
  borrowing func span() -> /* dependsOn(scoped self) */ Span<Element> { ... }
}

// The returned span copies dependencies from 'arg'.
func copySpan<T>(_ arg: Span<T>) -> /* dependsOn(arg) */ Span<T> { arg }

func parse(_ span: Span<Int>) { ... }
```

#### Scoped dependence on an immutable variable

```swift
let a: Array<Int> = ...
let span: Span<Int>
do {
  let a2 = a
  span = a2.span()
}
parse(span) // 🛑 Error: 'span' escapes the scope of 'a2'
```

The call to `span()` creates a scoped dependence on `a2`. A scoped dependence is determined by the lifetime of the variable, not the lifetime of the value assigned to that variable. So the lifetime of `span` cannot extend into the larger lifetime of `a`.

#### Copied dependence on an immutable variable

Let's contrast scoped dependence shown above with copied dependence on a variable. In this case, the value may outlive the variable it is copied from, as long as it is destroyed before the root of its inherited dependence goes out of scope. A chain of copied dependencies is always rooted in a scoped dependence.

An assignment that copies or moves a nonescapable value from one variable into another **copies** any lifetime dependence from the source value to the destination value. Thus, variable assignment has the same lifetime copy semantics as passing an argument using a `dependsOn()` annotation *without* a `scoped` keyword. So, the statement `let temp = span` has identical semantics to `let temp = copySpan(span)`.

```swift
let a: Array<Int> = arg
let final: Span<Int>
do {
  let span = a.span()
  let temp = span
  final = copySpan(temp)
}
parse(final) // ✅ Safe: still within lifetime of 'a'
```

Although the result of `copySpan` depends on `temp`, the result of the copy may be used outside of the `temp`'s lexical scope. Following the source of each copied dependence, up through the call chain if needed, eventually leads to the scoped dependence root. Here, `final` is the end of a lifetime dependence chain rooted at a scoped dependence on `a`:
`a -> span -> temp -> {copySpan argument} -> final`. `final` is therefore valid within the scope of `a` even if the intermediate copies have been destroyed.

#### Copied dependence on a mutable value

First, let's add a mutable method to `Span`:

```swift
extension Span {
  mutating func droppingPrefix(length: Int) -> /* dependsOn(self) */ Span<T> {
    let result = Span(base: base, count: length)
    self.base += length
    self.count -= length
    return result
  }
}
```

A dependence may be copied from a mutable ('inout') variable. In that case, the dependence is inherited from whatever value the mutable variable holds when it is accessed.

```swift
let a: Array<Int> = ...
var prefix: Span<Int>
do {
  var temp = a.span()
  prefix = temp.droppingPrefix(length: 1) // access 'temp' as 'inout'
  // 'prefix' depends on 'a', not 'temp'
}
parse(prefix) // ✅ Safe: still within lifetime of 'a'
```

#### Scoped dependence on 'inout' access

Now, let's return to scoped dependence, this time on a mutable variable. This is where exclusivity guarantees come into play. A scoped depenendence extends an access of the mutable variable across all uses of the dependent value. If the variable mutates again before the last use of the dependent, then it is an exclusivity violation.

```swift
let a: Array<Int> = ...
a[i] = ...
let span = a1.span()
parse(span) // ✅ Safe: still within 'span's access on 'a'
a[i] = ...
parse(span) // 🛑 Error: simultaneous access of 'a'
```

Here, `a1.span()` initiates a 'read' access on `a1`. The first call to `parse(span)` safely extends that read access. The read cannot extend to the second call because a mutation of `a1` occurs before it.

#### Dependence reassignment

We've described how a mutable variable can be the source of a lifetime dependence. Now let's look at nonescapable mutable variables. Being nonescapable means they depend on another lifetime. Being mutable means that dependence may change during reassignment. Reassigning a nonescapable 'inout' sets its lifetime dependence from that point on, up to either the end of the variable's lifetime or its next subsequent reassignment.

```swift
func reassign(_ span: inout Span<Int>) {
  let a: Array<Int> = ...
  span = a.span() // 🛑 Error: 'span' escapes the scope of 'a'
}
```

#### Reassignment with argument dependence

If a function takes a nonescapable 'inout' argument, it may only reassign that argument if it is marked dependent on another function argument that provies the source of the dependence.

```swift
func reassignWithArgDependence(_ span: dependsOn(arg) inout [Int], _ arg: [Int]) {
  span = arg.span() //  ✅ OK: 'span' already depends on 'arg' in the caller's scope.
}
```

#### Conditional reassignment creates conjoined dependence

'inout' argument dependence behaves like a conditional reassignment. After the call, the variable passed to the 'inout' argument has both its original dependence along with a new dependence on the argument that is the source of the argument dependence.

```swift
let a1: Array<Int> = arg
do {
  let a2: Array<Int> = arg
  var span = a1.span()
  testReassignArgDependence(&span, a2) // creates a conjoined dependence
  parse(span) // ✅ OK: within the lifetime of 'a1' & 'a2'
}
parse(span) // 🛑 Error: 'span' escapes the scope of 'a2'
```

## Source compatibility

Everything discussed here is additive to the existing Swift grammar and type system.
It has no effect on existing code.

## Effect on ABI stability

Lifetime dependency annotations may affect how values are passed into functions, and thus adding or removing one of these annotations should generally be expected to affect the ABI.

## Effect on API resilience

Adding a lifetime dependency constraint can cause existing valid source code to no longer be correct, since it introduces new restrictions on the lifetime of values that pre-existing code may not satisfy.
Removing a lifetime dependency constraint only affects existing source code in that it may change when deinitializers run, altering the ordering of deinitializer side-effects.

## Alternatives considered

### Different Position

We propose above putting the annotation on the return value, which we believe matches the intuition that the method or property is producing this lifetime dependence alongside the returned value.
It would also be possible to put an annotation on the parameters instead:

```swift
func f(@resultDependsOn arg1: Array<Int>) -> Span<Int>
```

Depending on the exact language in use, it could also be more natural to put the annotation after the return value.
However, we worry that this hides this critical information in cases where the return type is longer or more complex.

```swift
func f(arg1: Array<Int>) -> Span<Int> dependsOn(arg1)
```

### Different spellings

An earlier version of this proposal advocated using the existing `borrow`/`mutate`/`consume`/`copy` keywords to specify a particular lifetime dependency semantic:
```swift
func f(arg1: borrow Array<Int>) -> borrow(arg1) Span<Int>
```
This was changed after we realized that there was in practice almost always a single viable semantic for any given situation, so the additional refinement seemed unnecessary.

## Future Directions

### Lifetime Dependencies for Escapable Types

This proposal has deliberately limited the application of lifetime dependencies to return types that are nonescapable.
This simplifies the model by identifying nonescapable types as exactly those types that can carry such dependencies.
It also helps simplify the enforcement of lifetime constraints by guaranteeing that constrained values cannot escape before being returned.
Most importantly, this restriction helps ensure that the new semantics (especially lifetime dependency inference) cannot accidentally break existing code.
We expect that in the future, additional investigation can reveal a way to relax this restriction.

### Lifetime Dependencies for Tuples

It should be possible to return a tuple where one part has a lifetime dependency.
For example:
```swift
func f(a: consume A, b: B) -> (consume(a) C, B)
```
We expect to address this in the near future in a separate proposal.

### Lifetime Dependencies for containers and their elements

It should be possible to return containers with collections of lifetime-constrained elements.
For example, a container may want to return a partition of its contents:
```swift
borrowing func chunks(n: Int) -> dependsOn(self) SomeList<dependsOn(self) Span<UInt8>>
```
We're actively looking into ways to support these more involved cases and expect to address this in a future proposal.

### Parameter index for lifetime dependencies

Internally, the implementation records dependencies based on the parameter index.
This could be exposed as an alternate spelling if there were sufficient demand.

```swift
func f(arg1: Type1, arg2: Type2, arg3: Type3) -> dependsOn(0) ReturnType
```

### Value component lifetime

In the current design, aggregating multiple values merges their scopes.

```swift
struct Container<Element>: ~Escapable {
  var a: /*dependsOn(self)*/ Element
  var b: /*dependsOn(self)*/ Element

  init(a: Element, b: Element) -> dependsOn(a, b) Self {...}
}
```

This can have the effect of narrowing the lifetime scope of some components:

```swift
var a = ...
{
  let b = ...
  let c = Container<Element>(a: a, b: b)
  a = c.a // 🛑 Error: `a` outlives `c.a`, which is constrained by the lifetime of `b`
}
```

In the future, the lifetimes of multiple values can be represented independently by attaching a `@lifetime` attribute to a stored property and referring to that property's name inside `dependsOn` annotations:

```swift
struct Container<Element>: ~Escapable {
  @lifetime
  var a: /*dependsOn(self.a)*/ Element
  @lifetime
  var b: /*dependsOn(self.b)*/ Element

  init(a: Element, b: Element) -> dependsOn(a -> .a, b -> .b) Self {...}
}
```

The nesting level of a component is the inverse of the nesting level of its lifetime. `a` and `b` are nested components of `Container`, but the lifetime of a `Container` instance is nested within both lifetimes of `a` and `b`.

### Abstract lifetime components

Lifetime dependence is not always neatly tied to stored properties. Say that our `Container` now holds multiple elements within its own storage. We can use a top-level `@lifetime` annotation to name an abstract lifetime for all the elements:

```swift
@lifetime(elements)
struct Container<Element>: ~Escapable {
  var storage: UnsafeMutablePointer<Element>

  init(element: Element) -> dependsOn(element -> .elements) Self {...}

  subscript(position: Int) -> dependsOn(self.elements) Element
}
```

Note that a subscript setter reverses the dependence: `dependsOn(newValue -> .elements)`.

As before, when `Container` held a single element, it can temporarily take ownership of an element without narrowing its lifetime:

```swift
var c1: Container<Element>
{
  let c2 = Container<Element>(element: c1[i])
  c1[i] = c2[i] // OK: c2[i] can outlive c2
}
```

Now let's consider a `View` type, similar to `Span`, that provides access to a borrowed container's elements. The lifetime of the view depends on the container's storage. Therefore, the view depends on a *borrow* of the container. The container's elements, however, no longer depend on the container's storage once they have been copied. This can be expressed by giving the view an abstract lifetime for its elements, separate from the view's own lifetime:

```swift
@lifetime(elements)
struct View<Element>: ~Escapable {
  var storage: UnsafePointer<Element>

  init(container: Container)
    -> dependsOn(container.elements -> .elements) // Copy the lifetime associated with container.elements
    Self {...}

  subscript(position: Int) -> dependsOn(self.elements) Element
}

@lifetime(elements)
struct MutableView<Element>: ~Escapable, ~Copyable {
  var storage: UnsafeMutablePointer<Element>
  //...
}

extension Container {
  // Require a borrow scope in the caller that borrows the container
  var view: dependsOn(borrow self) View<Element> { get {...} }

  var mutableView: dependsOn(borrow self) MutableView<Element> { mutating get {...} }
}
```

Now an element can be copied out of a view `v2` and assigned to another view `v1` whose lifetime exceeds the borrow scope that constrains the lifetime of `v2`.

```swift
var c1: Container<Element>
let v1 = c1.mutableView
{
  let v2 = c1.view // borrow scope for `v2`
  v1[i] = v2[i] // OK: v2[i] can outlive v2
}
```

To see this more abstractly, rather than directly assigning, `v1[i] = v2[i]`, we can use a generic interface:

```swift
func transfer(from: Element, to: dependsOn(from) inout Element) {
  to = from
}

var c1: Container<Element>
let v1 = c1.mutableView
{
  let v2 = c1.view // borrow scope for `v2`
  transfer(from: v2[i], to: &v1[i]) // OK: v2[i] can outlive v2
}
```

### Protocol lifetime requirements

Value lifetimes are limited because they provide no way to refer to a lifetime without refering to a concrete type that the lifetime is associated with. To support generic interfaces, protocols need to refer to any lifetime requirements that can appear in interface.

Imagine that we want to access view through a protocol. To support returning elements that outlive the view, we need to require an `elements` lifetime requirement:

```swift
@lifetime(elements)
protocol ViewProtocol {
  subscript(position: Int) -> dependsOn(self.elements) Element
}
```

Let's return to View's initializer;

```swift
@lifetime(elements)
struct View<Element>: ~Escapable {
  init(container: borrowing Container) ->
    // Copy the lifetime assoicate with container.elements
    dependsOn(container.elements -> .elements)
    Self {...}
}
```

This is not a useful initializer, because `View` should not be specific to a concrete `Container` type. Instead, we want `View` to be generic over any container that provides `elements` that can be copied out of the container's storage:

```swift
@lifetime(elements)
protocol ElementStorage: ~Escapable {}

@lifetime(elements)
struct View<Element>: ~Escapable {
  init(storage: ElementStorage) ->
    // Copy the lifetime assoicate with storage.elements
    dependsOn(storage.elements -> .elements)
    Self {...}
}
```

### Structural lifetime dependencies

A scoped dependence normally cannot escape the lexical scope of its source variable. It may, however, be convenient to escape the source of that dependence along with any values that dependent on its lifetime. This could be done by moving the ownership of the source into a structure that preserves any dependence relationships. A function that returns a nonescapable type cannot currently depend on the scope of a consuming parameter. But we could lift that restriction provided that the consumed argument is moved into the return value, and that the return type preserves any dependence on that value:

```swift
struct OwnedSpan<T>: ~Copyable & ~Escapable{
  let owner: any ~Copyable
  let span: dependsOn(scope owner) Span<T>

  init(owner: consuming any ~Copyable, span: dependsOn(scope owner) Span<T>) -> dependsOn(scoped owner) Self {
    self.owner = owner
    self.span = span
  }
}

func arrayToOwnedSpan<T>(a: consuming [T]) -> OwnedSpan<T> {
  OwnedSpan(owner: a, span: a.span())
}
```

`arrayToOwnedSpan` creates a span with a scoped dependence on an array, then moves both the array and the span into an `OwnedSpan`, which can be returned from the function. This converts the original lexically scoped dependence into a structural dependence.

## Acknowledgements

Dima Galimzianov provided several examples for Future Directions.
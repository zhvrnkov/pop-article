# Generic Code
In that part we will look at how variables of generic types are stored and copied and how method dispatch works with them

### Non-generic version

```swift
protocol Drawable {
  func draw()
}

func drawACopy(local: Drawable) {
  local.draw()
}

let line = Line()
drawACopy(line)

let point = Point()
drawACopy(point)
```

Pretty basic code. `drawACopy` takes the argument of type `Drawable` and call its `draw` method - that's it.

### Generic version
Lets look at the generic version of upper code:

```swift
func drawACopy<T: Drawable>(local: T) {
  local.draw()
}

...
```

Well, seems like nothing changed. We can still just call the function `drawACopy` like it non-generic version and nothing more but the interesting things, as usual, are under the hood.

Generic code supports three major things:
1. static polymorphism (also known as parametric polymorphism)
2. one type per call context (`T` generic type is defined at compile time)
3. type substituted down the call chain

Lets look at this points through an example:

```swift
func foo<T: Drawable>(local: T) {
  bar(local)
}

func bar<T: Drawable>(local: T) { ... }

let point = Point(...)
foo(point)
```

The most interesting part is happend when we call `foo` function. The compiler knows exact type of `point` variable - it is just `Point`. Furthermore the type `T: Drawable` in function `foo` can be freely deduced by compiler since we pass the variable of known type `Point` to that function: `T = Point`. All types are known at compile-time and compiler can perform all its great optimizations - the most significant is inlining the `foo` call.
This:
```swift
let point = Point(...)
foo<T = Point>(point)
```
Becomes this:
```swift
bar<T = Point>(point)
```
Compiler just inline the `foo` call by its implementation and deduce the type of `bar`'s `T: Drawable` too.

### Implementation of Generic Methods
```swift
func drawACopy<T: Drawable>(local: T) {
  local.draw()
}

drawACopy(Point(...))
```
Inside of `drawACopy` Swift uses Protocol Witness Table (which contain all method implementations of T) and Value Witness Table (which contain all lifecycle methods for an instance of `T`). In pseudo code it looks like this:
```swift
func drawACopy<T: Drawable>(local: T, pwt: T.PWT, vwt: T.VWT) {...}

drawACopy(Point(...), Point.pwt, Point.vwt)
```

> `T.PWT` and `T.VWT` is associated types of `T` - like typealiases but better. `Point.pwt` and `Point.vwt` is static properties

Since, in our example, the `T` is `Point`, the `T` is well defined hence the creation of existential container isn't required. In previous non-generic version of `drawACopy(local: Drawable)` the existential container were created and this is necessary - we've studied this in the [second article](LINK_TO_SECOND_ARTICLE).

The Value Witness Table is required in functions because of creating the argument. As we known, the arguemnts in swift are passed by values hence need to be copied, and `copy` function of some type is belongs to Value Witness Table of that type as other lifecycle methods: `allocate`, `destruct` and `deallocated`.

The Protocol Witness Table is required in functions because of using the methods of the argument of generic type.

### Generic or Non-Generic?
Is using generic types makes code any faster then using just protocol types?

Well we notice, that generic code brings the more static form of polymorphism. It also enable the compiler optimization called "Specialization of generics". Lets take a look.

Again, we got the same code:
```swift
func drawACopy<T: Drawable>(local: T) {
  local.draw()
}

drawACopy(Point(...))
drawACopt(Line(...))
```

Specialization of generics creates a type specialized version of function per type based on generic version of that functions. For example if we call `drawACopy` with argument of type `Point` then compiler will create the specialized version of that function e.g. `drawACopyOfPoint(local: Point)` and we got:
```swift
func drawACopyOfPoint(local: Point) {
  local.draw()
}

func drawACopyOfLine(local: Line) {
  local.draw()
}

drawACopy(Point(...))
drawACopt(Line(...))
```

Which can be reduced furhermore by agressive compiler optimizations to this:
```swift
Point(...).draw()
Line(...).draw()
```

All this tricks are available because generic functions can be invoked with only one type per call - the generic type (`T`) is perfectly defined.

### Generic Stored Properties
Consider the simple `struct Pair`:
```swift
struct Pair {
  let fst: Drawable
  let snd: Drawable
}

let pair = Pair(fst: Line(...), snd: Line(...))
```

When we use it in this way, we got 2 heap allocations (the exact memory state were explained in [second part](LINK_TO_SECOND_ARTICLE)) but we can avoid it by using generics.
The generic version of `Pair` is looks like this:
```swift
struct Pair<T: Drawable> {
  let fst: T
  let snd: T
}
```
Since in generic version the `T` type is defined, the types of `fst` and `snd` are the same and also defined. Because type is defined, the compiler can allocate the specified amount of memory for that two properties `fst` and `snd`.

In more detail about specified amount of memory:
When we work with non-generic version of pair the type of `fst` and `snd` is `Drawable`. Any type can conform to `Drawable`, even if it takes 10 KB of memory. So then Swift could not infer the size of that type and use the universal memory layout called **existential cotainer**. Any type could fit in that container - on stack or heap. In case of generics the type is well known then the actual size of properties are also known and Swift can create the specialized memory layout. For example (generic version):
```swift
let pair = Pair(Point(...), Point(...))
```
The `T` type is `Point`. `Point` takes N bytes of memory and in `Pair` we got two of them. So Swift will allocate 2 * N amount of memory and place the `pair` into this. The same will work with any type that conform to `Drawable`.

So with generic version of `Pair` we got rid of extra heap allocations because types are well known and could be layout specifically - no need of creating of universal memory layouts because all is known.

### Sum Up
1. Specialized Generics - Struct type
have the best performance because:
+ no heap allocations on copying
+ generic code is like you write the function for that specific type
+ no reference counting
+ static method dispatch

2. Specialize Generics - Class type
have the medium performance because:
+ heap allocations on creating instances
+ reference counting
+ dynamic method dispatch through V-Table

3. Inspecialized Generics - Small Values
+ no heap allocations - value fits in value buffer of existential container
+ no reference counting (since no heap allocations)
+ dynamic dispatch through Protocol Witness Table

4. Unspecialized Generics - Large Values
+ heap allocations - value cannot fits in value buffer
+ reference counting
+ dynamic dispatch through Protocol Witness Table

This articles don't show that classes are bad, structs are good and structs with protocols with generics are best. What we are trying to say is that as a programmer this is your responsibility to choose the instruments for yout tasks. The classes are really good when you need to store large values and have reference semantic. The structs are best for small values and when you need value semantic. The protocols best fits with generics and structs and etc. All instruments are specific to a task that you are solving and have good and bad sides.

Also don't pay for dynamicism when you don't need it. Choose fitting abstraction with the least dynamic runtime requirements
+ struct types - value semantics
+ class types - identity
+ generics - static polymorphism
+ protocol types - dynamic polymorphism

Use indirect storage to deal with large values.

And don't forget - this is your responsibility to choose the right tools. Good luck
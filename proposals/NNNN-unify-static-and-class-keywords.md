# Unify `static` and `class` keywords

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-unify-static-and-class-keywords.md)
* Author: [Ling Wang](https://github.com/an0)
* Status: **Review**
* Review manager: TBD

## Introduction

The coexistence of `static` and `class` keywords for declaring type properties and methods is confusing and causes inconsistency between type and instance member declarations. This document reasons why we don’t need both and suggests we unify them with a better keyword `type`.

## Motivation

### Confusion

One “language enhancement” of [Swift 1.2](https://developer.apple.com/library/ios/releasenotes/DeveloperTools/RN-Xcode/Chapters/xc6_release_notes.html#//apple_ref/doc/uid/TP40001051-CH4-SW6) is:
> “static” methods and properties are now allowed in classes (as an alias for class final). You are now allowed to declare static stored properties in classes, which have global storage and are lazily initialized on first access (like global variables). Protocols now declare type requirements as static requirements instead of declaring them as class requirements. (17198298)

If even the Swift team itself has difficulties in picking one from the two and had to revert its initial decision after several revisions, we know these keywords are indeed confusing.

So now protocols use `static` to declare type methods, when a class adapts such protocols, should it implement those static methods, as static methods or class methods?

But static means final, right? Does it mean we can not override these methods in subclasses of the conforming class?

These kinds of unnecessary confusion and hesitation should be resolved, and could if this proposal is implemented.

### Unnecessary and inconsistent differentiation

The `class` keyword is only used in classes. In the current implementation of Swift the differences between `class` and `static` are:

1. Class properties can only be calculated properties but not stored properties.
2. Class methods(also calculated properties) are dynamically dispatched while static ones are statically dispatched.

If we can eliminate the differences or find better ways to differentiate we can use one unified keyword instead for declaring type properties and methods.

Let’s see.

#### Class stored properties VS. static stored properties

If you use the `class` keyword to declare a stored property in a class you will get this compiling error: 
> class stored properties not yet supported in classes; did you mean 'static'?

So what are class stored properties? How are they different from static stored properties?

As far as I know, class stored properties, if ever implemented in Swift, “would be what Ruby calls ‘class instance variables’”.<sup>[1](https://twitter.com/UINT_MIN/status/584104757117095936)</sup>

So what are class instance variables?

The best explanation I can find is this [one](http://martinfowler.com/bliki/ClassInstanceVariable.html) from Martin Fowler.

Do we really want this feature in Swift? “If we didn't already have these, would we add them to Swift 3?"

I strongly believe we won’t add it to Swift 3. Actually I believe we will never add it to Swift, because its use cases are so rare which is also why it hasn’t been implemented so far in Swift.

If we agree we are not going to support class stored properties, there is and will be only one kind of stored properties for types and we only need one keyword to declare such properties.

#### Class methods VS. static methods

*Since calculated properties are also methods in essence they are also covered by this section.*

The only difference is how methods are dispatched.

Let’s see [how we handle it for instance methods](https://developer.apple.com/swift/blog/?id=27):

* Methods are overridable hence dynamically dispatched by default.
* In performance critical code use these techniques to restrict this dynamic behavior when it isn’t needed to improve performance:

    1. Use the `final` keyword when we know that a declaration does not need to be overridden.
    2. Infer `final` on declarations referenced in one file by applying the `private` keyword.
    3. Use `Whole Module Optimization` to infer `final` on `internal` declarations.

So why abandon this whole system to use another totally different one for differentiating `static dispatch` and `dynamic dispatch` for type methods?

If we reuse this system for type methods, not only can we have a consistent design for both instance and type methods, but also we can get rid of the last place where two keywords for type member declarations are needed.

## Proposed solution

1. Use the keyword `type` to declare type properties and methods. 
2. Type properties and methods are overridable hence dynamically dispatched by default. Use the `final` keyword or inferred `final` to make them final and statically dispatched, just like instance properties and methods.
3. Type properties can be stored or calculated, just like instance properties. Stored type properties are globally stored like global variables — Swift doesn’t support “class instance variables”.

As you can see, it is a very simple and elegant design:

* Just a single keyword `type` to differentiate type member declarations from instance member declarations. `type` is a good keyword because:

    1. It is consistent with the wording of the concepts of `type properties` and `type methods`.
    2. It is never used as a keyword before in Swift, Objective-C or C. There will be no conflicts or overloading of it meanings. 
* Except for that, how things are declared, differentiated and optimized are exactly the same in both type and instance world. Very consistent.

## Comparison with current design

* Dynamic Dispatch VS. Static Dispatch

```swift
// Old
class Foo {
    func dynamicInstanceMethod() {}
    final func staticInstanceMethod() {}
    
    class func dynamicTypeMethod() {}
    static func staticTypeMethod() {}
}
```

```swift
// New
class Foo {
    func dynamicInstanceMethod() {}
    final func staticInstanceMethod() {}

    type func dynamicTypeMethod() {}
    final type func staticTypeMethod() {}
}
```

* Stored Properties VS. Calculated Properties

```swift
// Old
class Bar {
    static let i = 1
    
    class var j: Int {
        return 1
    }
}
```

```swift
// New
class Bar {
    type let i = 1
    
    type var j: Int {
        return 1
    }
}
```

* Struct Implementation VS. Class Implementation of Protocol

```swift
// Old
protocol P {
    static func foo()
}

struct S: P {
    static func foo() {}
}

class C: P {
    class func foo() {}
}
```

```swift
// New
protocol P {
    type func foo()
}

struct S: P {
    type func foo() {}
}

class C: P {
    type func foo() {}
}
```

## Impact on existing code

With the help of a good migration tool, there will be no impact on existing code at all. The migration rules are:

* Map `static` to `type` in protocols.
* Map `static` to `type` in structs and enums.
* Map `class` to `type` in classes.
* Map `static func` to `final type func` in classes.
* Map `static let` and `static var` to `type let` and `type var` in classes.

One concern I can think of is: because type methods are dynamically dispatched by default in the new design, will we forget to do the `final` optimization so the general performance of Swift code become worse?

I think it is probably true. But we also forget to do the `final` optimization for instance methods from time to time. Since there are way more instance methods than type methods in most code the performance impact will be very small. Maybe this change is a good opportunity to remind us to do the `final` optimization for instance methods thus even results in a better general performance.

And don’t forget we have the tools to automatically infer `final` for us in many cases if we write the proper code and use the proper compiler features.

After all, it is mainly on us to write good code to produce good final products. If the system is consistent we’ll have better chances to master it and use it properly.

## Alternatives considered

Alternatively we could:
* Keep using `static` and `class` keywords.
* Keep the confusion when implementing `static` protocol requirements using `class` properties and methods in conforming classes.
* Keep the inconsistency between type member declarations and instance member declarations.
* Keep overloading meanings on the `static` keyword that is already historically overloaded in C and Objective-C with which Swift must mix and match.

# A Vision for Pure Functions as a Language Level Feature in Swift 
As Swift 6 brings compiler guarantees of data race safety, we have increased safety, but in many cases decreased ergonomics. In the current paradigm, in order for concurrent code to be used in Swift 6, the code must be either isolated to the same actor or it must be sendable. This vision aims to provide another option. 

Users may define a unit of work in a pure function. Users must explicitly opt into pure functions. Pure functions have less privileges than regular functions but in return they are automatically `Sendable`. This allows them to freely cross isolation boundaries with minimal ceremony. 

## Acknowledgments
Language level pure functions are a feature that the community has proposed since the beginning of the langauge. Here are a handful of Swift evolution pitches that influenced this design: 

1. https://forums.swift.org/t/proposal-idea-support-for-pure-functions/675
2. https://forums.swift.org/t/pure-function-attribute/34907/5

## Roadmap
There are four main components to this vision, each of which are intertwined:

1. **pure functions**: A `func` may be marked as a `pure func`. 
2. **pure closures**: 
   1. A function parameter of type closure may be marked `@pure` to indicate that it must be a pure function
   2. A closure definition may be marked `@pure` to indicate that it is a pure function. 
3. **pure methods**: Methods like functions may be marked as `pure func`, but there are some extra requirements, namely that it is not allowed to read or write from `self` unless it is explicitly passed in a parameter. 
4. **conditional purity**: Functions may declare that they may be inferred to be pure, if and only if certain conditions are met. 

When the vision seems ready, I will pitch these components as separate Swift evolution proposals. 

### Pure Functions
>Note: This section is only talking about global functions. There are some extra requirements for methods which are defined below in the "Pure Methods" section.

The `pure` keyword can be used to mark functions. A pure function:

1. Cannot read or write external state (unless explicitly passed as a parameter)
2. Cannot call functions or closures unless they are also `pure`
   1. Cannot receive a function as an input parameter unless it is `@pure`. 
3. Is automatically inferred to be `@Sendable`
4. Is always permitted to be used in a context requiring an `@escaping` closure.
5. Is not permitted to perform I/O operations. 
6. Cannot accept `inout` parameters
7. call function-private inner functions, even if they’re impure (since their context is pure)


```swift
// For functions
pure func pureFunction(input: Int) -> Int {
    // Function body
}
```

### Pure Closures
The `@pure` attribute: 
1. can be applied to parameter types indicating that the parameter must be a `pure` function or `@pure` closure
2. can be applied to closure definitions, indicating that the closure is pure
3. implies `@escaping` for closures, so it is not necessary to use both. 
   1. Note: A pure function can escape the current context, but it cannot perform side effects, so semantically it should not imply the same dangers as `@escaping` closures. 
4. doesn't by itself make the annotated code conform, but enforces that conformance and makes it apparent to its users. (this is similar to the behavior of `@escaping` and `@inlinable`)

All of the requirements of pure functions also apply to pure closures. In addition there are some extra requirements: 
1. A pure closure may not capture ("close") over outside state (unless it is explicitly defined in the parameter list).

```swift
// For closure type annotations
let pureClosure: @pure (Int) -> Int = { input in
    // Closure body
}

// Marking a closure as pure when inferring the type
let pureClosure = { @pure input: Int in
    return 0
}

// For non-pure higher-order functions, 
// A function may require an `@pure` function as a parameter input.
func higherOrderFunction(transform: @pure (Int) -> Int) {
    // Function body
}

func impureHigherOrderFunction(transform: @escaping (Int) -> Int) {
    // ...
}
// A pure closure can be used in an `@escaping` context.
impureHigherOrderFunction(pureClosure)
```

When a `pure func` takes a function as a parameter, it is always required to be `@pure` therefore, `@pure` can be inferred and left out of the definition. 
```swift
// For pure higher-order functions
pure func pureHigherOrderFunction(transform: (Int) -> Int) {
    // Function body
}
```

>Note: Technically a pure closure is not a closure at all. Like a regular closure in Swift, it is a self-contained block of functionality that can be passed around. However, unlike a closure, a pure closure is not allowed to capture and store references to any outside state (unless it is explicitly defined in the parameter list). 

### Pure Methods
All of the requirements of pure functions also apply to pure methods. In addition there are some extra requirements. 

1. a pure method is not allowed to read or write from `self` unless it is explicitly passed in a parameter
2. In a struct, a pure method must be non-mutating.

### Conditional Purity
A higher-order function may declare that it is pure if and only if it's input functions are also pure using the `@purable` keyword (tentative name). 

```swift
typealias IntTransform = (Int) -> Int
@purable func map<T>(_ transform: (Element) throws -> T) rethrows -> [T] { 
    // ...
}
```
In the code above, map will be a pure function, if and only if `transform` is a pure function. Otherwise a non-pure overload will be selected. 

## Future Directions
1. Introduce a way to mark entire types as pure, indicating that all their methods are pure.
2. Explore compiler optimizations that can leverage function purity, such as memoization or parallel execution of pure function calls.


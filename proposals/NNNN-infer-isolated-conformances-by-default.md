# Infer isolated conformances by default

- Proposal: [SE-NNNN](NNNN-infer-isolated-conformances-by-default.md)
- Authors: [Sima Nerush](https://github.com/simanerush),
  [Holly Borla](https://github.com/hborla),
  [Pavel Yaskevich](https://github.com/xedin)
- Review Manager: TBD
- Status: **Awaiting implementation**
- Vision:
  [Improving the approachability of data-race safety](https://github.com/swiftlang/swift-evolution/blob/main/visions/approachable-concurrency.md)
- Implementation:
  [swiftlang/swift#93603](https://github.com/swiftlang/swift/pull/93603),
  [swiftlang/swift#93596](https://github.com/swiftlang/swift/pull/93596),
  [swiftlang/swift#93601](https://github.com/swiftlang/swift/pull/93601)
- Upcoming Feature Flag: `MyFeatureName`
- Review: ([pitch](https://forums.swift.org/...))

## Summary of changes

This proposal brings isolated conformance inference, previously only enabled in
modules compiled in default isolation mode set to `MainActor`, to modules in
"nonisolated" default isolation mode.

## Motivation

Global-actor isolated conformances, introduced in
[SE-0470](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0470-isolated-conformances.md),
are a useful feature for letting isolated types implement nonisolated protocol
requirements. However, it is cumbersome to use in default isolation mode set to
"nonisolated".

```swift
protocol Incrementable {
  mutating func increment()
}

@MainActor
struct Counter: Equatable, Incrementable {
  var count = 0

  mutating func increment() {
    count += 1
    save()
  }

  static func == (lhs: Counter, rhs: Counter) -> Bool {
    lhs.count == rhs.count
  }

  func save() { /* ... */ }
}
```

In the above code, witnesses `==` and `increment()` are isolated to the main
actor because `Counter` is. Therefore, they cannot satisfy the nonisolated
requirements of `Equatable` and `Incrementable` protocols, producing the
following errors:

```
error: conformance of 'Counter' to protocol 'Equatable' crosses into main actor-isolated code and can cause data races
note: main actor-isolated operator function '==' cannot satisfy nonisolated requirement

error: conformance of 'Counter' to protocol 'Incrementable' crosses into main actor-isolated code and can cause data races
note: main actor-isolated instance method 'increment()' cannot satisfy nonisolated requirement
```

The programmer can resolve the above errors by marking the witnesses with
`nonisolated`, but that doesn't work once a witness needs anything isolated to
the global actor:

```swift
@MainActor
struct Counter: Incrementable {
  var count = 0

  nonisolated mutating func increment() {
    count += 1
    save() // error: call to main actor-isolated instance method 'save()' in a synchronous nonisolated context
  }

  func save() { /* ... */ }
}
```

SE-0470 lets the programmer explicitly specify the isolation of each
conformance:

```swift
@MainActor
struct Counter: @MainActor Equatable, @MainActor Incrementable {
  // ...
}
```

In the above code, by isolating `Counter`'s conformances to `Equatable` and
`Incrementable` to the `@MainActor`, the programmer can now implement these
protocols' requirements using `@MainActor`-isolated code.

However, this forces the programmer to re-spell the isolation the `Counter` type
already declares on every conformance, which is cumbersome.

By contrast, if the `Counter` type was declared in a module compiled with
default isolation set to `MainActor`, the compiler would automatically infer the
conformances' isolation off the conforming type:

```swift
// default-isolation=MainActor

/*MainActor*/
struct Counter: /*MainActor*/ Equatable, /*MainActor*/ Incrementable {
  // ...
}
```

In the above code, the conformances to `Equatable` and `Incrementable` protocols
are implicitly `@MainActor`-isolated, meaning the programmer can easily witness
the requirements with `@MainActor`-isolated code:

```swift
// default-isolation=MainActor

nonisolated protocol Incrementable {
  mutating func increment()
}

/*MainActor*/
struct Counter: /*MainActor*/ Equatable, /*MainActor*/ Incrementable {
  var count = 0

  mutating func increment() { // ✅ okay, conformance is 'MainActor'-isolated.
    count += 1
    save()
  }

  static func == (lhs: Counter, rhs: Counter) -> Bool { // ✅ okay, conformance is 'MainActor'-isolated.
    lhs.count == rhs.count
  }

   func save() { /* ... */ }
}
```

While this behavior makes the `MainActor` default isolation mode usable, it also
creates a dialect in the language where the behavior of a conformance depends on
a module-wide setting that isn't visible in the source. Code that compiles in an
app target stops compiling when it is moved into a package or framework that
uses the `nonisolated` default.

To expand on the above statement, the inference of isolated conformances in
`MainActor` modules, while invisible to the programmer, restricts how the
program can use such conformances. Using such a conformance outside the main
actor is an error:

```swift
func incrementAll<T: Incrementable>(_ values: inout [T]) {
  for i in values.indices { values[i].increment() }
}

@MainActor
func onMain(_ counters: inout [Counter]) {
  incrementAll(&counters) // okay
}

nonisolated func offMain(_ counters: inout [Counter]) {
  incrementAll(&counters) // error: main actor-isolated conformance of 'Counter' to 'Incrementable' cannot be used in nonisolated context
}
```

This extra checking added for isolated conformances creates behavioral
differences between `MainActor` modules, which automatically infer the
conformance's isolation, and `nonisolated` modules, which have to spell out the
isolation explicitly to opt in to them.

This can make it undesirable to switch targets to `MainActor` mode, because
suddenly the behavior and usage requirements of various conformances can change
due to inferred isolation.

## Proposed solution

We propose aligning the isolated conformances inference rules between
`MainActor` and `nonisolated` default isolation modes.

More concretely, under the proposed changes, programs compiling in `nonisolated`
default isolation mode, would infer isolated conformances where possible, making
it easier to implement `nonisolated` protocol requirements with an
actor-isolated type.

```swift
// default-isolation=nonisolated

protocol Incrementable {
  mutating func increment()
}

@MainActor
struct Counter: /*MainActor*/ Equatable, /*MainActor*/ Incrementable {
  var count = 0

  mutating func increment() { // okay
    count += 1
    save()
  }

  static func == (lhs: Counter, rhs: Counter) -> Bool { //okay
    lhs.count == rhs.count
  }

  func save() { /* ... */ }
}
```

The above inference will take effect in a future language mode, while warning
programmers in the Swift 6 language mode, and in the Swift 5 language mode with
complete concurrency checking, about a potential misuse of the isolated
conformance that will be inferred in the future mode.

Code like `Counter` above is already an error in Swift 6, so it isn't affected.
The conformances whose meaning changes are the ones that are valid and
`nonisolated` today, but that the new rules would infer as isolated. For
example, a main actor-isolated `async` method can satisfy a `nonisolated`
`async` requirement today, because calls to it hop to the main actor:

```swift
// default-isolation=nonisolated
// language-version=6, or language-version=5 with -strict-concurrency=complete

protocol DataLoader {
  func load() async -> [String]
}

@MainActor
struct FeedLoader: DataLoader { // warning: conformance of 'FeedLoader' to 'DataLoader' will be inferred as main actor-isolated in a future language mode
  var cache: [String] = []

  func load() async -> [String] { cache } // note: main actor-isolated instance method 'load()' makes the conformance to `DataLoader` isolated
}

func makeLoader() -> some DataLoader {
  FeedLoader() // warning: main actor-isolated conformance of 'FeedLoader' to 'DataLoader' cannot be used in nonisolated context; this is an error in a future Swift language mode
}
```

In the Swift 6 language mode, and in the Swift 5 language mode with complete
concurrency checking, the warnings will be emitted:

- At the conformance, so that the programmer can decide on its isolation where
  it's declared. This is useful for library authors to explicitly specify the
  conformance's isolation and avoid introducing errors in client code.
- At each use of the conformance that would become an error once the conformance
  is inferred as isolated.

Both warnings are resolved by spelling out the conformance's isolation. To keep
the current behavior, the programmer can explicitly mark the conformance with
`nonisolated`:

```swift
@MainActor
struct FeedLoader: nonisolated DataLoader {
  // ...
}

func makeLoader() -> some DataLoader {
  FeedLoader() // okay
}
```

Alternatively, the programmer can adopt the new behavior right away by marking
the conformance `@MainActor`, in which case the use in `makeLoader()` becomes an
error in Swift 6 language mode as well.

## Detailed design

This proposal posits clarifying the behavior of isolated conformances inference
when enabled in targets compiled with default isolation set to `nonisolated`. We
propose adding the following new inference rules on top existing ones used today
in `MainActor` modules. The following sections will dive into the details of the
proposed rules. Here's an overview:

1. Enable inference of isolated conformances to global-actor isolated protocols
2. A conformance implied by a conformance to a `SendableMetatype`-refining
   protocol is never inferred isolated
3. Not every actor-isolated witness can force isolated conformance inference

### Rule 1. Enable inference of isolated conformances to global-actor isolated protocols

SE-0470 never infers an isolated conformance to a protocol that is itself
isolated to a global actor. Such a conformance is `nonisolated` unless it's
explicitly written as `@MainActor P`. This becomes a problem when the protocol
has `nonisolated` requirements, because the conforming type's witnesses are
isolated to the protocol's global actor:

```swift
@MainActor
protocol P {
  nonisolated func f()
}

struct S: P { // error: conformance of 'S' to protocol 'P' crosses into main actor-isolated code and can cause data races
  @MainActor func f() { } // note: main actor-isolated instance method 'f()' cannot satisfy nonisolated requirement
}
```

`S` is implicitly `@MainActor` because it conforms to the `@MainActor` protocol
`P`. Here, `f()` is explicitly marked `@MainActor`, because otherwise it would
infer `nonisolated` from the requirement it witnesses.

Under this proposal, conformance isolation inference also applies to
conformances to isolated protocols, so the conformance of `S` to `P` is inferred
to be `@MainActor`:

```swift
struct S: /*inferred @MainActor*/ P {
  @MainActor func f() { } // okay, the conformance is main actor-isolated
}
```

If all of the protocol's requirements are isolated, the conformance is inferred
to be `nonisolated`, as it is today. Every witness can only be called from the
protocol's global actor anyway, so isolating the conformance would only restrict
where the conformance itself can be used:

```swift
@MainActor
protocol Q {
  func g()
}

struct S: /*inferred nonisolated*/ Q {
  func g() { }
}

nonisolated func useQ<T: Q>(_: T.Type) { }

nonisolated func test() {
  useQ(S.self) // okay
}
```

A conformance is still inferred to be isolated if any of the conformances it
implies are isolated, even when all of the protocol's own requirements are
isolated:

```swift
protocol P {
  func f()
}

@MainActor
protocol Q: P {
  func g()
}

struct S: /*inferred @MainActor*/ Q {
  func f() { } // makes the implied conformance to 'P' main actor-isolated
  func g() { }
}
```

Today, the conformance of `S` to `Q` is never inferred to be isolated, so the
code above is rejected:

```
error: conformance of 'S' to protocol 'Q' crosses into main actor-isolated code and can cause data races
note: conformance depends on main actor-isolated conformance of 'S' to protocol 'P'
```

### Rule 2. A conformance implied by a conformance to a `SendableMetatype`-refining protocol is never inferred isolated

SE-0470 never infers an isolated conformance to a protocol that refines
`SendableMetatype` (including indirectly, for example through `Sendable`),
because such a conformance could never be used. This proposal extends that rule
to the conformances it implies. The conformance to `SendableP` below requires
`Self: P` for a `Sendable` type `Self`, and an isolated conformance can never
satisfy that requirement:

```swift
protocol P {
  func f() async
}

protocol SendableP: P, Sendable { }

@MainActor
final class C: SendableP {
  func f() async { }
}
```

Today, in a module with `nonisolated` default isolation that explicitly enables
the `InferIsolatedConformances` upcoming feature, the implied conformance of `C`
to `P` is inferred to be `@MainActor` because `f()` is main actor-isolated, and
then rejected:

```
error: type 'C' does not conform to protocol 'SendableP'
error: main actor-isolated conformance of 'C' to 'P' cannot satisfy conformance requirement for a 'Sendable' type parameter 'Self'
note: requirement specified as 'Self' : 'P' [with Self = C]
```

Under this proposal, the implied conformance to `P` is inferred to be
`nonisolated`, and the code above is valid: a main actor-isolated `async`
witness can satisfy a `nonisolated` `async` requirement, because calls to it hop
to the main actor.

When a witness can't satisfy a `nonisolated` requirement, the compiler now
diagnoses the witness itself, which makes the problem easier to find and fix:

```swift
protocol Q {
  func g()
}

protocol SendableQ: Q, Sendable { }

@MainActor
final class C: SendableQ { // error: conformance of 'C' to protocol 'Q' crosses into main actor-isolated code and can cause data races
  func g() { } // note: main actor-isolated instance method 'g()' cannot satisfy nonisolated requirement
}
```

### Rule 3. Not every actor-isolated witness can force isolated conformance inference

Under SE-0470, a conformance of a global actor-isolated type is inferred to be
isolated as soon as any of its witnesses is isolated. However, many isolated
witnesses are safe to use from any isolation domain. Under this proposal, a
witness makes the conformance isolated only if using it requires the actor. A
witness doesn't make the conformance isolated if:

- it is not isolated,
- it can be accessed from any isolation domain, such as a `let` of a `Sendable`
  type, or a stored `var` of a `Sendable` type in a value type, or
- it has the same isolation as the requirement it satisfies, so every caller is
  already isolated to that actor.

```swift
protocol P {
  var x: Int { get }
  var y: Int { get }
  @MainActor func f()
}

@MainActor
final class C: /*inferred nonisolated*/ P {
  let x = 0                    // 'Sendable' let, accessible from any isolation domain
  nonisolated var y: Int { 0 } // not isolated
  func f() { }                 // same isolation as the requirement
}

nonisolated func useP<T: P>(_: T.Type) { }

nonisolated func test() {
  useP(C.self) // okay
}
```

Today, in a module with `MainActor` default isolation (where `P` is declared
`nonisolated protocol P`), or in a module with `nonisolated` default isolation
that explicitly enables the `InferIsolatedConformances` upcoming feature, the
conformance of `C` to `P` is inferred to be `@MainActor`, and the call in
`test()` is rejected:

```
error: main actor-isolated conformance of 'C' to 'P' cannot be used in nonisolated context
```

Any other isolated witness still makes the conformance isolated:

```swift
protocol Q {
  var x: Int { get }
}

@MainActor
final class C: /*inferred @MainActor*/ Q {
  var x: Int { 0 } // main actor-isolated computed property
}
```

### Migrating existing code

TODO: this section should go into details of the warnings about the future
isolation of the conformances.

## Source compatibility

This proposal is fully source compatible with existing code.

## ABI compatibility

This proposal is fully ABI compatible.

## Implications on adoption

The new inference rules automatically apply in all existing language modes, but
the data-race checking does not prevent the program from compiling until the
next language mode.

## Future directions

### Don't infer isolated conformances from `async` witnesses

Under this proposal, a global actor-isolated `async` witness makes the
conformance isolated, even when it satisfies a `nonisolated` `async`
requirement. Such a witness doesn't need the conformance to be isolated, though,
since a call through the requirement is already asynchronous, so it can hop to
the witness's actor:

```swift
protocol P {
  func f() async
}

@MainActor
final class C: /*inferred @MainActor*/ P {
  func f() async { } // main actor-isolated, but callers can hop to the main actor
}

nonisolated func useP<T: P>(_ value: T) async {
  await value.f()
}

nonisolated func test(_ c: C) async {
  await useP(c) // error: main actor-isolated conformance of 'C' to 'P' cannot be used in nonisolated context
}
```

Today, the programmer can avoid this by writing the conformance as
`nonisolated P`, in which case the code above is valid. A future proposal could
extend Rule 3 so that an isolated `async` witness to an `async` requirement
doesn't make the conformance isolated, like a witness that can be accessed from
any isolation domain. The conformance of `C` to `P` above would then be inferred
to be `nonisolated`.

## Alternatives considered

## Acknowledgments

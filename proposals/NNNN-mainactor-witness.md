# Feature name

* Proposal: [SE-NNNN](NNNN-mainactor-witness.md)
* Authors: [Pavel Yaskevich](https://github.com/xedin), [Aviva Ruben](https://github.com/a-viv-a), [Sima Nerush](https://github.com/simanerush)
* Review Manager: TBD
* Status: **Awaiting implementation** or **Awaiting review**
* Bug: *if applicable* [swiftlang/swift#NNNNN](https://github.com/swiftlang/swift/issues/NNNNN)
* Implementation: [swiftlang/swift#NNNNN](https://github.com/swiftlang/swift/pull/NNNNN) or [swiftlang/swift-evolution-staging#NNNNN](https://github.com/swiftlang/swift-evolution-staging/pull/NNNNN)
* Experimental Feature Flag: *if applicable* `MyFeatureName`
* Upcoming Feature Flag: *if applicable* `MyFeatureName`
* Review: ([pitch](https://forums.swift.org/...))

## Summary of changes

Introduces a way to parametrize a global actor attribute with `witness`
(FIXME naming?) parameter to stop transferring protocol declaration's isolation
onto its requirements, and treat them as `nonisolated` unless specified
otherwise.

## Motivation

Since [SE-0316](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0316-global-actors.md), a global actor applied to a protocol declaration propagates the attribute to any type that conforms to it in the primary type definition by default. With global-actor isolated conformances feature introduced by [SE-0470](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0470-isolated-conformances.md), it becomes hard for global-actor-isolated protocols to take advantage of this feature.

More concretely, let us consider the following protocol:

```swift
@MainActor
public protocol Node {
    func value() -> Int
}
```

Today, the `value()` requirement is implicitly `@MainActor` from its protocol declaration, making it unusable from `nonisolated` generic code:

```swift
@MainActor
struct Caller {
    nonisolated func call<N: Node>(_ node: N) {
        print(node.value()) // Call to main actor-isolated instance method 'value()' in a synchronous nonisolated context
    }
}
```

With global-actor isolated conformances, the library author might wish to provide different behavior for clients who implement a `@MainActor`-isolated conformance and a `nonisolated` conformance:

```swift
/*@MainActor*/struct MainActorNode: @MainActor Node {
    func value
}

struct NonisolatedNode: nonisolated Node {

}

// Library code
@MainActor
struct Caller {
    nonisolated func call<N: Node>(_ node: N) {
        print(node.value()) // Call to main actor-isolated instance method 'value()' in a synchronous nonisolated context
    }
}
```

while keeping the original intent of the `value` witness being `@MainActor`-isolated by default, unless specified otherwise.

Today, a way for a library author to work around this would be to mark the requirement explicitly with `nonisolated`:

```swift
@MainActor
public protocol Node {
    nonisolated func value() -> Int
}
```

The above change would make it possible to witness the `value()` requirement with a global-actor isolated implementation by forming a `@MainActor`-isolated conformance to `Node`:

```swift
/*@MainActor*/struct MainActorNode: @MainActor Node {
    @MainActor func value
}

struct NonisolatedNode: nonisolated Node {
    nonisolated func value
}
```

However, that requires every client to write `@MainActor` explicitly on the
witness to create a main-actor conformance. This is not natural, as the programmer would expect the `MainActorNode` above which is isolated to the same global actor as the protocol, be able to witness the protocol requirement with an implicitly `@MainActor`-isolated method.

The inverse is also true: the `nonisolated` conformance requires the `nonisolated` to be spelled out on the `value` witness.

Therefore, ...

## Proposed solution

* Propose a parametrized `@MainActor(witness)` attribute that allows protocol requirements to infer the specified global actor only if the witness has the matching isolation.
* Examples


## Detailed design

* Isolation of the requirement is always nonisolated (isolation of the protocol does not transfer onto requirement). Unless explicitly specified
* InferIsolatedConformances behaves differently: we do inference even if the protocol specifies the new attribute
    * If there's a "clash" in requirements' isolation, the default isolation of the conformance will be nonisolated
* TODO generic code implications? casting?

```swift
@MainActor(witness) // does not affect requirements or P protocol itself, only the conforming type unless overridden
protocol P {
    func req() // isolation is unspecified (vs. today it would be @MainActor)
}

/*MainActor*/ struct S: /*MainActor*/ P { // 'S' is @MainActor because P is. Conformance to P is @MainActor because 'InferIsolatedConformances'
    func req() // @MainActor 
}

nonisolated struct S: /*nonisolated*/ P { // 'S' is explicitly 'nonisolated' so there's nothing to infer
    func req() // nonisolated
}

nonisolated struct S: @MainActor P {
    // everything else nonisolated
    @MainActor func req() // MainActor explicitly
}

@MainActor struct S: nonisolated P {
    // everything else '@MainActor'
    nonisolated func req() // nonisolated explicitly
}
```

## Source compatibility

Describe the impact of this proposal on source compatibility.  As a
general rule, all else being equal, Swift code that worked in previous
releases of the tools should work in new releases.  That means both that
it should continue to build and that it should continue to behave
dynamically the same as it did before.  Changes that cannot satisfy
this must be opt-in, generally by requiring a new language mode.

This is not an absolute guarantee, and the Language Workgroup will
consider intentional compatibility breaks if their negative impact
can be shown to be small and the current behavior is causing
substantial problems in practice.

For proposals that affect parsing, consider whether existing valid
code might parse differently under the proposal.  Does the proposal
reserve new keywords that can no longer be used as identifiers?

For proposals that affect type checking, consider whether existing valid
code might type-check differently under the proposal.  Does it add new
conversions that might make more overload candidates viable?  Does it
change how names are looked up in existing code?  Does it make
type-checking more expensive in ways that might run into implementation
limits more often?

For proposals that affect the standard library, consider the impact on
existing clients.  If clients provide a similar API, will type-checking
find the right one?  If the feature overloads an existing API, is it
problematic that existing users of that API might start resolving to
the new API?

## ABI compatibility

Describe the impact on ABI compatibility.  As a general rule, the ABI
of existing code must not change between tools releases or language
modes.  This rule does not apply as often as source compatibility, but
it is much stricter, and the Language Workgroup generally cannot allow
exceptions.

The ABI encompasses all aspects of how code is generated for the
language, how that code interacts with other code that has been
compiled separately, and how that code interacts with the Swift
runtime library.  Most ABI changes center around interactions with
specific declarations.  Proposals that do not affect how code is
generated to interact with an external declaration usually do not
have ABI impact.

For proposals that affect general code generation rules, consider
the impact on code that's already been compiled.  Does the proposal
affect declarations that haven't explicitly adopted it, and if so,
does it change ABI details such as symbol names or conventions
around their use?  Will existing code change its dynamic behavior
when running against a new version of the language runtime or
standard library?  Conversely, will code compiled in the new way
continue to run on old versions of the language runtime or standard
library?

For proposals that affect the standard library, consider the impact
on any existing declarations.  As above, does the proposal change symbol
names, conventions, or dynamic behavior?  Will newly-compiled code work
on old library versions, and will new library versions work with
previously-compiled code?

This section will often end up very short.  A proposal that just
adds a new standard library feature, for example, will usually
say either "This proposal is purely an extension of the ABI of the
standard library and does not change any existing features" or
"This proposal is purely an extension of the standard library which
can be implemented without any ABI support" (whichever applies).
Nonetheless, it is important to demonstrate that you've considered
the ABI implications.

If the design of the feature was significantly constrained by
the need to maintain ABI compatibility, this section is a reasonable
place to discuss that.

## Implications on adoption

The compatibility sections above are focused on the direct impact
of the proposal on existing code.  In this section, describe issues
that intentional adopters of the proposal should be aware of.

For proposals that add features to the language or standard library,
consider whether the features require ABI support.  Will adopters need
a new version of the library or language runtime?  Be conservative: if
you're hoping to support back-deployment, but you can't guarantee it
at the time of review, just say that the feature requires a new
version.

Consider also the impact on library adopters of those features.  Can
adopting this feature in a library break source or ABI compatibility
for users of the library?  If a library adopts the feature, can it
be *un*-adopted later without breaking source or ABI compatibility?
Will package authors be able to selectively adopt this feature depending
on the tools version available, or will it require bumping the minimum
tools version required by the package?

If there are no concerns to raise in this section, leave it in with
text like "This feature can be freely adopted and un-adopted in source
code with no deployment constraints and without affecting source or ABI
compatibility."

## Future directions

Describe any interesting proposals that could build on this proposal
in the future.  This is especially important when these future
directions inform the design of the proposal, for example by making
sure an attribute encodes enough information to be used for other
purposes.

The rest of the proposal should generally not talk about future
directions except by referring to this section.  It is important
not to confuse reviewers about what is covered by this specific
proposal.  If there's a larger vision that needs to be explained
in order to understand this proposal, consider starting a discussion
thread on the forums to capture your broader thoughts.

Avoid making affirmative statements in this section, such as "we
will" or even "we should".  Describe the proposals neutrally as
possibilities to be considered in the future.

Consider whether any of these future directions should really just
be part of the current proposal.  It's important to make focused,
self-contained proposals that can be incrementally implemented and
reviewed, but it's also good when proposals feel "complete" rather
than leaving significant gaps in their design.  For example, when
[SE-0193](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0193-cross-module-inlining-and-specialization.md)
introduced the `@inlinable` attribute, it also included the
`@usableFromInline` attribute so that declarations used in inlinable
functions didn't have to be `public`.  This was a relatively small
addition to the proposal which avoided creating a serious usability
problem for many adopters of `@inlinable`.

## Alternatives considered

Describe alternative approaches to addressing the same problem.
This is an important part of most proposal documents.  Reviewers
are often familiar with other approaches prior to review and may
have reasons to prefer them.  This section is your first opportunity
to try to convince them that your approach is the right one, and
even if you don't fully succeed, you can help set the terms of the
conversation and make the review a much more productive exchange
of ideas.

You should be fair about other proposals, but you do not have to
be neutral; after all, you are specifically proposing something
else.  Describe any advantages these alternatives might have, but
also be sure to explain the disadvantages that led you to prefer
the approach in this proposal.

You should update this section during the pitch phase to discuss
any particularly interesting alternatives raised by the community.
You do not need to list every idea raised during the pitch, just
the ones you think raise points that are worth discussing.  Of course,
if you decide the alternative is more compelling than what's in
the current proposal, you should change the main proposal; be sure
to then discuss your previous proposal in this section and explain
why the new idea is better.

## Acknowledgments

If significant changes or improvements suggested by members of the 
community were incorporated into the proposal as it developed, take a
moment here to thank them for their contributions. Swift evolution is a 
collaborative process, and everyone's input should receive recognition!

Generally, you should not acknowledge anyone who is listed as a
co-author or as the review manager.

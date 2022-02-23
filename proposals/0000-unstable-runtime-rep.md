---
author: Jeffrey Young
date-accepted: ""
ticket-url: ""
implemented: ""
---

This proposal is [discussed at this pull request](https://github.com/ghc-proposals/ghc-proposals/pull/0>).
**After creating the pull request, edit this file again, update the number in
the link, and delete this bold sentence.**

# Unstable RuntimeRep

`RuntimeRep` is a user facing type that describes the memory representation of
Haskell types in the runtime system. However, adding new backends to GHC, such
as a Javascript or JVM backend, requires a way to represent arbitrary objects
that only live in the new backend's runtime system. This leads to two problems;
first, there is no `RuntimeRep` Constructor that is appropriate to represent
such objects because they are by definition platform dependent and may not be
pointer-sized. Second, because `RuntimeRep` is user facing, experimentation with
`RuntimeRep` requires ghc proposals. We seek to use this experimental proposal
to investigate the design space of `RuntimeRep` and consolidate the discussion
around it with respect to new backends, such as the upcoming Javascript backend.
In particular, we suggest treating `RuntimeRep` as unstable except for the
`BoxedRep` constructor.


## Motivation

For the Javascript GHC backend, we require a way to refer to Javascript objects
from Haskell. This implies adding a primitive type `JSVal#` to GHC for FFI
support. Similarly, since we now have a new prim type whose memory
representation is categorically different from other prim types, we must also
extend `RuntimeRep` with `JSValRep`.

But this process immediately leads to unsatisfactory scenarios because it is not
extensible. As GHC becomes more modular, it is unlikely (and undesirable) that
the Javascript backend will be the only new backend. Thus, the most trivial
scenario is each new backend implies a new `RuntimeRep` constructor. In this
scenario, for each new backend `Bk` becomes we must: 1) extending prim types
with `BkVal#` and 2) extend `RuntimeRep` with `BkRep`. But such a process
obviously leads to a bloated `RuntimeRep` and likely to code duplication. The
essential problem is a matter of design; namely, how to make a polymorphic
`RuntimeRep` which is platform dependent.

The second scenario is the aggressive use of `{-# LANGUAGE CPP
#-}`. In this scenario, `RuntimeRep` becomes:

```haskell
data RuntimeRep
  = ...
#ifdef javascript_HOST_ARCH
  | JSRef
#elif jvm_HOST_ARCH
  | JVMRef
#elif beam_HOST_ARCH
  ... 
#endif
```

and so on. But then this incurs the maintenance costs and other headaches that
come with `CPP`. 

The third scenario is to add a prim type `Opaque#` and corresponding runtime
representation `OpaqueRep` that serves as a handle to any foreign value or
object in a new runtime for a new backend. But then it is not immediately clear
what the meaning of an opaque runtime rep is, because this necessarily must be
defined by the new runtime, and thus must be dependent on the target platform.

Thus we are in the following situation: For new backends we need to extend
`RuntimeRep`. `RuntimeRep`, specifies how a value of type is represented in
memory, but it cannot do so in novel platforms who might have different memory
conventions, and thus _should_ be platform dependent, but it is not. Naive fixes
then either create a bloated `RuntimeRep` in an attempt to capture all possible
cases, make aggressive use of `CPP` to perform a check for the host
architecture, or are something more exotic, such as an existential type or use
of backpack. In either case, the current design of `RuntimeRep` for new backends
is unsatisfactory and the design space needs exploration. Thus, we argue that
`RuntimeRep`, except for `BoxedRep` should be considered unstable.

## Proposed Change Specification

### Option 1: Bite the bullet and enumerate RuntimeRep
RuntimeRep becomes

```
data RuntimeRep = VecRep VecCount VecElem   -- ^ a SIMD vector type
                | TupleRep [RuntimeRep]     -- ^ An unboxed tuple of the given reps
                | SumRep [RuntimeRep]       -- ^ An unboxed sum of the given reps
                | BoxedRep Levity -- ^ boxed; represented by a pointer
                | ...
                | JSValRep          -- ^ Javascript values and objects
                | JVMValRep         -- ^ JVM values and objects
                | FooValRep         -- ^ Foo values and objects
```

And all consumers of `RuntimeRep` are updated accordingly. We check for the
correct platform in the code generator and panic if the platform doesn't line up
accordingly.

Benefits of this approach are that it is simple, the least invasive change, and
preserves the majority of consuming code.

### Option 2: Live with the CPP
RuntimeRep becomes

```
data RuntimeRep = VecRep VecCount VecElem   -- ^ a SIMD vector type
                | TupleRep [RuntimeRep]     -- ^ An unboxed tuple of the given reps
                | SumRep [RuntimeRep]       -- ^ An unboxed sum of the given reps
                | BoxedRep Levity -- ^ boxed; represented by a pointer
                | ...
                | JSValRep          -- ^ Javascript values and objects
                | JVMValRep         -- ^ JVM values and objects
                | FooValRep         -- ^ Foo values and objects
#ifdef javascript_HOST_ARCH
                | JSRef
#elif jvm_HOST_ARCH
                | JVMRef
#elif beam_HOST_ARCH
  ... 
```

And all consumers of `RuntimeRep` are updated accordingly. Benefits of this
approach are better cohesion in `RuntimeRep`; we have less junk values
inhabiting the type after the pre-processor, this approach fails fast; should a
user not sufficiently `CPP` their new-backend-only definitions we can
immediately short circuit the compilation, the majority of consuming code is
preserved.

### Option 3: Something more fancy

Specify the change in precise, comprehensive yet concise language. Avoid words
like "should" or "could". Strive for a complete definition. Your specification
may include,

* BNF grammar and semantics of any new syntactic constructs
  (Use the [Haskell 2010 Report](https://www.haskell.org/onlinereport/haskell2010/) or GHC's
  `alex`- or `happy`-formatted files
  for the [lexer](https://gitlab.haskell.org/ghc/ghc/-/blob/master/compiler/GHC/Parser/Lexer.x) or [parser](https://gitlab.haskell.org/ghc/ghc/-/blob/master/compiler/GHC/Parser.y)
  for a good starting point.)
* the types and semantics of any new library interfaces
* how the proposed change interacts with existing language or compiler
  features, in case that is otherwise ambiguous

Strive for *precision*. The ideal specification is described as a
modification of the [Haskell 2010
report](https://www.haskell.org/definition/haskell2010.pdf). Where
that is not possible (e.g. because the specification relates to a
feature that is not in the Haskell 2010 report), try to adhere its
style and level of detail. Think about corner cases. Write down
general rules and invariants.

Note, however, that this section should focus on a precise
*specification*; it need not (and should not) devote space to
*implementation* details -- there is a separate section for that.

The specification can, and almost always should, be illustrated with
*examples* that illustrate corner cases. But it is not sufficient to
give a couple of examples and regard that as the specification! The
examples should illustrate and elucidate a clearly-articulated
specification that covers the general case.

## Effect and Interactions

Your proposed change addresses the issues raised in the
motivation. Explain how.

Also, discuss possibly contentious interactions with existing language or compiler
features. Complete this section with potential interactions raised
during the PR discussion.


## Costs and Drawbacks

Give an estimate on development and maintenance costs. List how this effects
learnability of the language for novice users. Define and list any remaining
drawbacks that cannot be resolved.


## Alternatives

List alternative designs to your proposed change. Both existing
workarounds, or alternative choices for the changes. Explain
the reasons for choosing the proposed change over these alternative:
*e.g.* they can be cheaper but insufficient, or better but too
expensive. Or something else.

The PR discussion often raises other potential designs, and they should be
added to this section. Similarly, if the proposed change
specification changes significantly, the old one should be listed in
this section.

## Unresolved Questions

Explicitly list any remaining issues that remain in the conceptual design and
specification. Be upfront and trust that the community will help. Please do
not list *implementation* issues.

Hopefully this section will be empty by the time the proposal is brought to
the steering committee.


## Implementation Plan

(Optional) If accepted who will implement the change? Which other resources
and prerequisites are required for implementation?

## Endorsements

(Optional) This section provides an opportunity for any third parties to express their
support for the proposal, and to say why they would like to see it adopted.
It is not mandatory for have any endorsements at all, but the more substantial
the proposal is, the more desirable it is to offer evidence that there is
significant demand from the community.  This section is one way to provide
such evidence.


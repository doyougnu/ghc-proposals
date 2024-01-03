Proposal title
==============

.. author:: Jeffrey Young
.. date-accepted::
.. ticket-url::
.. implemented::
.. highlight:: haskell
.. header:: This proposal is `discussed at this pull request <https://github.com/ghc-proposals/ghc-proposals/pull/0>`_.
            **After creating the pull request, edit this file again, update the
            number in the link, and delete this bold sentence.**
.. sectnum::
.. contents::

GHC's profiler is a venerable, traditional profiler that determines CPU time and
allocations spent on a piece of code. However useful traditional profilers are
they are liable to lose the forest for the trees; they do nothing to help a
developer understand the effect of a change or the impact of a piece of code *on
the performance of a system*. But this is changing; in 2016 Emery Berger's group
unvieled the first *causal profiler* called `coz
<https://github.com/plasma-umass/coz>`_. Instead of tracking time and
allocations for a piece of code, causal profilers record the *optimization
potential*, thereby communicating to the end-user which piece of code *if
optimized* will have the largest impact on their system. This proposal proposes
integrating and implementing a causal profiler into GHC.


Motivation
---------

It is often said that laziness *itself* makes performance difficult to reason
about....

Optimizing GHC compiled Haskell code is a careful art because small changes,
such as adding a single bang, can have widespread runtime effects throughout a
sufficiently complicated system. Thus we turn to GHC's profiler to know where
our system spends its CPU time and the genesis of memory allocations. But GHC's
profiler relies on a crucial assumption: if we optimize the sections of the
system where GHC's runtime system spends most of its time, then the system as a
whole will have improved performance. This assumption works well for
call-by-value languages, but does how does it far for a call-by-need language?
Imagine that we have two functions ``foo`` and ``bar``

But GHC's profiler cannot tell us if optimizing a certain section of code will
result in improved *throughput* and *latency* of the entire system.



Give a strong reason for why the community needs this change. Describe the use
case as clearly as possible and give an example. Explain how the status quo is
insufficient or not ideal.

A good Motivation section is often driven by examples and real-world scenarios.


Proposed Change Specification
-----------------------------

We propose to add a new kind of profiler; a *causal profiler* to GHC. The new
profiler will hide behind a GHC flag such as ``-causal-prof``.


What is the big idea behind causal profiling?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The first causal profiler sensed optimization opportunities at the thread level.
Imagine one has two threads running in a program, ``thread A`` and ``thread B``.
To profile, Coz would test what would happen to the system if ``thread A``
magically experienced a speedup. To do so it slows down ``thread B`` by issuing
a sleep every time ``thread B`` is selected to run by the scheduler. And
similarly to observe the effect of ``thread B`` speeding up, it artificially
will slow down ``thread A``.

How does Coz work?
^^^^^^^^^^^^^^^^^^

Coz only works on the thread level and requires the program to be compiled with
DWARF symbols. This allows it to find sections of code that a user has marked in
the assembly code. It exposes four C macros for users to mark their code. For
example:

.. code-block:: C







Coz exposes 4 C macros for a user to mark their code. A mark

Specify the change in precise, comprehensive yet concise language. Avoid words
like "should" or "could". Strive for a complete definition. Your specification
may include,

* BNF grammar and semantics of any new syntactic constructs
  (Use the `Haskell 2010 Report <https://www.haskell.org/onlinereport/haskell2010/>`_ or GHC's ``alex``\- or ``happy``\-formatted files
  for the `lexer <https://gitlab.haskell.org/ghc/ghc/-/blob/master/compiler/GHC/Parser/Lexer.x>`_ or `parser <https://gitlab.haskell.org/ghc/ghc/-/blob/master/compiler/GHC/Parser.y>`_
  for a good starting point.)
* the types and semantics of any new library interfaces
* how the proposed change interacts with existing language or compiler
  features, in case that is otherwise ambiguous

Strive for *precision*. The ideal specification is described as a
modification of the `Haskell 2010 report
<https://www.haskell.org/definition/haskell2010.pdf>`_. Where that is
not possible (e.g. because the specification relates to a feature that
is not in the Haskell 2010 report), try to adhere its style and level
of detail. Think about corner cases. Write down general rules and
invariants.

Note, however, that this section should focus on a precise
*specification*; it need not (and should not) devote space to
*implementation* details -- the "Implementation Plan" section can be used for that.

The specification can, and almost always should, be illustrated with
*examples* that illustrate corner cases. But it is not sufficient to
give a couple of examples and regard that as the specification! The
examples should illustrate and elucidate a clearly-articulated
specification that covers the general case.

Proposed Library Change Specification
-------------------------------------

Specify the changes to libraries in the GHC repository, especially `base` and
others under the purview of the
`Core Libraries Committee <https://github.com/haskell/core-libraries-committee>`_.

Generally speaking, if your proposal adds new function or data types, the place
to do so is in the ``ghc-experimental`` package, whose API is under the control of
the GHC Steering Committee.
After your proposal is implemented, stable, and widely used, you (or anyone
else) can subsequently propose to move those types into ``base`` via a CLC
proposal.

Sometimes, however, your proposal necessarily changes something in ``base``,
whose API is curated by the CLC.
In that case, assuming your proposal is accepted, at the point when it is
implemented (by you or anyone else), CLC approval will be needed for these
changes, via a CLC proposal made by the implementor.
By signalling those changes now, at the proposal stage, the CLC will be alerted
and have an opportunity to offer feedback, and agreement in principle.

See `GHC base libraries <https://github.com/Ericson2314/tech-proposals/blob/ghc-base-libraries/proposals/accepted/051-ghc-base-libraries.rst?rgh-link-date=2023-07-09T17%3A01%3A15Z>`_
for some useful context.

Therefore, in this section:

* If your proposal makes any changes to the API of ``base`` (including its
  exports, types, semantics, and performance), please specify these changes
  in this section.

* If your proposal makes any change to the API of ``ghc-experimental``, please
  also specify these changes.

If you propose to change both, use subsections, so that the changes are clearly
distinguished.
Similarly, if any other libraries are affected, please lay it all out here.

Examples
--------
This section illustrates the specification through the use of examples of the
language change proposed. It is best to exemplify each point made in the
specification, though perhaps one example can cover several points. Contrived
examples are OK here. If the Motivation section describes something that is
hard to do without this proposal, this is a good place to show how easy that
thing is to do with the proposal.

Effect and Interactions
-----------------------
Your proposed change addresses the issues raised in the motivation. Explain how.

Also, discuss possibly contentious interactions with existing language or compiler
features. Complete this section with potential interactions raised
during the PR discussion.


Costs and Drawbacks
-------------------
Give an estimate on development and maintenance costs. List how this affects
learnability of the language for novice users. Define and list any remaining
drawbacks that cannot be resolved.


Backward Compatibility
----------------------
Will your proposed change cause any existing programs to change behavior or
stop working? Assess the expected impact on existing code on the following scale:

0. No breakage
1. Breakage only in extremely rare cases (e.g. for specifically-constructed
   examples, but probably no packages published in the Hackage package repository)
2. Breakage in rare cases (e.g. a few Hackage packages may break, but probably
   no packages included in recent Stackage package sets)
3. Breakage in uncommon cases (e.g. a few Stackage packages may break)
4. Breakage in common cases

(For the purposes of this assessment, GHC emitting new warnings is not
considered to be a breaking change, i.e. packages are assumed not to use
``-Werror``.  Changing a warning into an error is considered a breaking change.)

Explain why the benefits of the change outweigh the costs of breakage.
Describe the migration path. Consider specifying a compatibility warning for one
or more compiler releases before the change is fully implemented. Give examples
of error messages that will be reported for previously-working code; do they
make it easy for users to understand what needs to change and why?

When the proposal is implemented, the implementers and/or GHC maintainers should
test that the actual backwards compatibility impact of the implementation is no
greater than the expected impact. If not, the proposal should be revised and the
steering committee approve the change.


Alternatives
------------
List alternative designs to your proposed change. Both existing
workarounds, or alternative choices for the changes. Explain
the reasons for choosing the proposed change over these alternative:
*e.g.* they can be cheaper but insufficient, or better but too
expensive. Or something else.

The PR discussion often raises other potential designs, and they should be
added to this section. Similarly, if the proposed change
specification changes significantly, the old one should be listed in
this section.

Unresolved Questions
--------------------
Explicitly list any remaining issues that remain in the conceptual design and
specification. Be upfront and trust that the community will help. Please do
not list *implementation* issues.

Hopefully this section will be empty by the time the proposal is brought to
the steering committee.


Implementation Plan
-------------------
(Optional) If accepted who will implement the change? Which other resources
and prerequisites are required for implementation?

Endorsements
-------------
(Optional) This section provides an opportunity for any third parties to express their
support for the proposal, and to say why they would like to see it adopted.
It is not mandatory for have any endorsements at all, but the more substantial
the proposal is, the more desirable it is to offer evidence that there is
significant demand from the community.  This section is one way to provide
such evidence.

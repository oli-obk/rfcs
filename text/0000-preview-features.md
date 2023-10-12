- Feature Name: `preview-features`
- Start Date: 2023-10-12
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Allow the usage of unpolished features on the stable compiler, if the feature is sufficiently finished
that users can experiment with them. The usage of such features will need opt-in on the crate and all
dependents of the crate. These features may perform minor breaking changes that always have a migration
path, and are guaranteed to never disappear.

# Motivation
[motivation]: #motivation

Stabilizing new features in Rust is a lot of effort, and often doesn't happen due to two conflicting desires:

* ship a polished feature
* first get a lot of feedback on the feature

The issue is that to get feedback on a feature, we usually need it to be available on the stable compiler, and
without the feedback, we don't know what to polish (ok we often do, but there are infinite things to polish).

Some features stay around in unstable limbo for years (`const_mut_refs`, I'm looking at you), because they have some
use case that is broken, but we don't want to ship without that use case.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

You can enable some not-yet-stabilized language features on the stable compiler by adding a `#![preview(1.72)]`
attribute to the crate root (the top of the `lib.rs` or `main.rs` file). This has two effects:

1. any crates depending on your crate now need to add the preview attribute (or one with a higher version number), too
2. you can now use all preview features that existed in Rust 1.72

These kind of features will not go away, but may need more polish. This means:

* diagnostics may be below the quality you are used to from stable Rust features
* some edge cases may need more fine tuning and your code may break and need fixing
    * your code will be fixable, but
    * it may require 
* some important use cases may not work yet at all

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is a language proposal, could this be done in a library or macro instead? Does the proposed change make Rust code easier or harder to read, understand, and maintain?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.

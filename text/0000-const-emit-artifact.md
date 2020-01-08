# Write to files from const eval

- Feature Name: `const_emit_artifact`
- Start Date: 2020-01-08
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/const-eval#25](https://github.com/rust-lang/const-eval/issues/25)

# Summary
[summary]: #summary

```
#[emit_artifact = "foo.man"]
const _: &[u8] = b"foomp"; // or some actual const eval
```

creates `target/debug/const_write_dump/foo.man` with the content `foomp` but each build first erases the `target/debug/const_write_dump` folder so you can't use `include_bytes!` to read from it.

# Motivation
[motivation]: #motivation

The CLI WG wants to generate files during compilation such as [shell completions](https://github.com/clap-rs/clap_generate), and [man pages](https://github.com/rust-cli/man).

Currently the best way to achieve this is by creating [a `build.rs` file](https://github.com/yoshuawuyts/changelog/blob/master/build.rs), and making sure [the right structs are exported](https://github.com/yoshuawuyts/changelog/blob/master/src/cli.rs). This is not great, because it's easy to mess up, the use of `build.rs` triggers a double compilation, and certain dependencies also need to be required in as both `[dependencies]` and `[dev-dependencies]`.

Instead it would be a lot nicer if there was a way to generate output during compilation that wouldn't require any additional setup beyond the usual flow of using crates.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.


Add a limited form of writing files to `const fn`. The reasoning for limited filesystem access, rather than full access is to prevent people from writing output in a prior compile, and reading it back in during a next compile (or even the same one), causing problems for reproducibility.

So something akin to `read_bytes!` , but for writing output to a special folder somewhere in `target/` that gets removed at the start of each build to ensure no data from a prior build persists. Its implementation is inherently different from `read_bytes!` as `read_bytes!` happens during macro expansion (so in the AST), while `#[emit_artifact]` needs const eval to run first, which happens on the MIR.

Some late stage (likely something in `codegen`) would

1. collect all constants with the attribute
2. error if
    * two attributes have the same output file
    * a constant's value has pointers in it
    * a file name does not adhere to the regex
3. write the byte contents of each constant to its file

## Why an attribute?

The attribute on constant scheme is better than a `const fn` which writes to the filesystem, as there's no runtime equivalent. So if it were a `const fn`, then `const fn` could call it and try to write to an output file, but if the `const fn` is called at runtime, it's unclear what would happen.

## Permitted types

To start out with we could just allow `&[u8]` and require the user to produce the corresponding data via const eval (e.g. by having a constant compute the result into an array and returning that)

While we could allow types other than `[u8]` it's unclear how they should be serialized, and that serialization would have to be const evaluable anyway. If it is const evaluable, you can always serialize to an array of `u8` and put that into a constant.

## Filenames

for simplicity we'd only allow output filenames of the following regex: `[a-z0-9_]+[a-z0-9_-\.]*` to prevent any troubles that could come from other filenames (attempting to crawl up directories, trying to escape some path scheme...).

# Drawbacks
[drawbacks]: #drawbacks

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

* The `#[emit_artifact]` attribute can also be forever unstable and instead we'd expose a `write_bytes!` macro:
    ```rust
    macro_rules! write_bytes {
        ($filename:expr = $data:expr) => {{
            #[emit_artifact = $filename]
            const _: &[u8] = $data;
        }}
    }
    ```
* As already noted in the motivation, we can just keep using `build.rs` files, which works, but requires some fiddling.
* There are many ways to design the details differently, but they all boil down to the same feature
    * `const fn` instead of an attribute (worse because `const fn` can be called at runtime and there's no equivalent).
        * while we could re-use `std::fs::write`, that would be
            * super confusing because at runtime you'd not write into the special directory but whereever the current directory is
            * hard/fragile to implement because we'd need to shim the file writing logic in an OS independent way
            * hard/fragile to extract the path, potentially opening avenues for exploitation

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

* On wasm, the `#[link_section]` attribute for static items will raw dump the bytes of the static into the chosen section. This also forbids the use of relocations in the static (so no pointers). Although this doesn't allow the user to choose the output file name.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

* Naming of the attribute/macro
* Decision on attribute vs macro
* Ordering issues when writing to the same file from multiple sites
    * We can just start by outlawing this and punt to the future
* Where do dependencies write to?
    * Can we read from dependencies' written files via `include_bytes!`?
        * Useless feature as we can just read constants provided by the deps

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how the this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.

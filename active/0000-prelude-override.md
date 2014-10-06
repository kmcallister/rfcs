- Start Date: 2014-10-06
- RFC PR #: (leave this empty)
- Rust Issue #: (leave this empty)

# Summary

Allow a crate to select which module ("the prelude") is implicitly glob-imported into every module.

# Motivation

So that I don't have to put `use core::prelude::*;` in every module of my `no_std` crate.

Large projects with a set of items that are used everywhere may also want a custom prelude.

# Detailed design

To reduce confusion, every crate has a single prelude throughout, or none at all.  This is determined from crate attributes in the following way:

* If the crate has `#![feature(prelude_override)]` then search the crate for a `mod` or a `use` of a mod that has `#[prelude]`.  There should be exactly one, otherwise it's an error.  That module becomes the crate's prelude.
* Otherwise, if the crate has `#![no_std]`, there is no prelude.
* Otherwise, the prelude is `std::prelude`.

So the incantation for building a project on `libcore` would be

```rust
#![feature(phase, prelude_override)]
#![no_std]
#[phase(plugin, link)] extern crate core;
#[prelude] use core::prelude;
```

Note that we need to consider this import as used for lint purposes, even though the name `prelude` may be unused in the top-level module.

Modules with `#[no_implicit_prelude]` skip the prelude import regardless of which module it is.

When `#[prelude]` is on a `mod` (or a `use` of a mod in the same crate) we have to avoid injecting a self-reference!

# Drawbacks

It adds complexity to the compiler, and it's a bit of spooky action at a distance.

# Alternatives

This could be more flexible, e.g. allow overriding the prelude at any point of the module hierarchy. But I think that would be massively confusing, and it's not necessary for any use case I'm aware of.  It would become more useful if glob imports are removed (#305) and this becomes the *only* way to bring a bunch of things into scope without naming them.

We could allow paths in attributes and then use `#![prelude=core::prelude]` at the crate root instead.  Or do that in today's attribute system with a string, but that seems gross because you don't expect a string's contents to be interpreted according to the lexical scope.  See also [Niko's blog post](http://smallcultfollowing.com/babysteps/blog/2014/09/11/attribute-and-macro-syntax/) which suggests allowing arbitrary token trees in attributes.

The impact of not doing this is one extra line of code in every module of a `no_std` crate.  Not a huge problem to be honest :)

# Unresolved questions

If this feature ever becomes not gated, we'll need something to replace the feature gate's role.  Allowing any module to set a prelude for the whole crate without changing the crate root would be a problem.

---
layout: post
title:  "Find Usages"
date:   2019-11-13 11:00:00 +0200
---

Last month, rust-analyzer gained an exciting new feature: find usages. It was implemented by [@viorina] in [#1892].

This post describes how the feature works under the hood.
It's an excellent case study to compare approaches of traditional compilers with IDE-oriented compilers (shortened to IDE from now one).

[@viorina]: https://github.com/viorina
[#1892]:    https://github.com/rust-analyzer/rust-analyzer/pull/1892


## Definitions and Usages

Let's start with a simple example:

```rust
fn foo() {
    let x = 92;
    let y = 62;

    if condition {
        y + 1
    } else {
        x + 2
    };
}
```

Suppose that a user invoked *find usages* (also known as *reference search*) on `x`.
An IDE should highlight `x` in `x + 2` as a usage.

But lets start with a simpler problem: *goto definition* (which is the reverse of *find usages*).
How does a compiler or an IDE understands that `x` in `x + 2` refers to `let x = 92;`?

Terminology Note: we will call the `x` in `let x = 92` a **definition**, and the `x` in `x + 2` a **reference**.

Note: the following presentation is oversimplified and is not based directly on any existing compiler or IDE.
Consider it a text-book illustration, and not a fossil specimen.

## Compiler's Approach

A typical compiler works by building a symbol table.
It does a depth-first traversal of the syntax tree of a program, maintaining a hash-map of definitions.
For the above example, the compiler does the following steps, in order:

1. Create an empty map.
2. Visit `let x = 92`. Records this definition in the map under the `"x"` key.
3. Visit `let y = 62`. Records this definition in the map under the `"y"` key.
4. Visit `condition`.
5. Visit "then" branch of the `if` expression. Lookup `"y"` in the map and record association between a reference and a definition.
6. Visit "else" branch of the `if` expression. Lookup `"x"` in the map and record association between a reference and a definition.

After the end of the traversal, the compiler knows, for each reference, what definition this reference points to.
If a reference is unresolved, the compiler emits an error.

We may say that a compiler processes a program **top-down**.

## IDE Approach

To figure out where the `x` in `x + 2` points to, a typical IDE does something different.
Instead of starting from the root of the tree, it starts from the usage, and proceeds upwards.

So, the IDE would do the following:

1. Start at `x + 2`.
2. Look at the parent "else" branch, notice that there are no definitions there.
3. Look at the parent "if" expression.
4. Look at the parent block, notice that it has `x` defined, return this definition as an answer.

The crucial point here is that IDE skips "then" branch completely.
It doesn't look like a big deal with this small branch which contains a single expression only.
However, the real world programs are much more complicated, and an IDE can skip quite a lot of code by only climbing the tree up.

This is the **bottom-up** approach.

## Which Way is Better?

It depends!
Let's do a quick estimation of work required to find, for each reference, a definition it points to.

The compiler's case is easy: we just traverse a program once, doing hashmap operations along the way.
That would be `O(program_size)`.

The IDE's case is more tricky: we still need to traverse a program to find all references.
Then, for each reference, we need to launch the "traverse tree upwards" procedure.
Which gives us `O(program_size + n_references * program_depth)`.
This is clearly worse!

But let's now look at the time we need to resolve one specific reference.
A compiler still has to construct the symbol table, so it's still `O(program_size)`.
An IDE, however, can launch only a single upward traversal, and that would be only `O(program_depth)`!

These observations are exactly the reason why compilers prefer the first approach, while IDEs favor the second one.
A compiler has to check and compile all the code anyway, so it's important to do all the work as efficiently as possible.
For IDEs however, the main trick is to ignore as much code as feasible.
An IDE needs to know only about usages under one specific reference under the cursor in the currently opened file.
It doesn't care what is the definition of a `spam` variable used in the `frobnicate` function somewhere in the guts of the standard library.

More generally, an IDE would like
- to know everything about a small droplet of code that is currently on the screen and
- to know nothing about the vast ocean of code that is off-screen.

## Find Usages

Let's get back to the original problem, find usages.

If a compiler has already constructed a symbol table, the solution is trivial: just enumerate all the usages.
It might need a little bit of extra bookkeeping (storing the list of usages in the symbol table), but basically this is just "print the answer we already have".

For an IDE, something else is required.
The trivial solution of doing a bottom-up traversal from every reference is worse than just launching  the compiler from scratch.

Instead, IDEs do a cute trick, which can be called a hack even!
IntelliJ, Type Script, and, since last month, rust-analyzer work like this.

First thing an IDE does is a *text* search across all files, which finds the set of potential matches.
As text search is much simpler than code analysis, this phase finishes relatively quickly.
Then the IDE filters out false positives, by doing bottom-up traversal from candidate matches.

The text-based pre filtering again allows the IDE to skip over most of the code, and complete find-usages in less time than it would the compiler to build a symbol table.

## Incrementality vs Laziness

Can we make the top-down approach as effective as bottom-up (and maybe even more so), by just making the calculation of symbol table incremental?
The idea is to maintain a data structure that connects references and definitions and, when a user changes a piece of code, apply the diff to the data structure, instead of re-computing it from scratch.

The fundamental issue with this approach is that it solves the problem an IDE doesn't have in the first place.
From that symbol data structure, only a tiny part is interesting for an IDE at any given moment of time.
Most of the code is private implementation details of dependencies, and they are completely irrelevant for IDE tasks, unless a user invokes "go to definition" on a symbol from library and actively studies these details.

On the other hand, building and updating such data structure takes time.
Specifically, because the data is intricate and depends on the language semantics, small changes to the source code (change of a module name, for example) might necessitate big rearrangement of computed result.

In general, laziness (ability to ignore most of the code) and incrementality (ability to quickly update derived data based on source changes) are orthogonal features.
First and foremost, an IDE requires laziness, although incrementality can be used as well to speed some things up.

In particular, it is possible to make the text-based phase of reference search incremental.
An IDE can main a trigram index: for each three-byte sequence, a list of files and positions where this sequence occurs.
Unlike symbol tables, such index is easy to maintain, as any change in a file can only affect trigrams from this file.
The index can then be used to speedup text search.
The result is the following *find usages* funnel:

1. First, an IDE finds all positions where identifier's  trigrams match,
2. Then, the IDE checks if a trigram match is in fact a full identifier match,
3. Finally, IDE uses semantic analysis to prune away remaining false-positives.

This is optimization is not implemented in rust-analyzer yet.
It definitely is planned, but not for the immediate future.

## Tricks

Let's look at a couple of additional tricks an IDE can employ.

First, the IDE can add yet another step to the funnel: pruning the set of files worth searching.
These restrictions can originate from the language semantics: it doesn't make sense to look for `pub(crate)` declaration outside of the current crate or for `pub` declaration among crate dependencies.
They also can originate from the user: it's often convenient to exclude tests from search results, for example.

The second trick is about implementing warnings for unused declarations effectively.
This is a case where a top-down approach is generally better, as an IDE needs to process every declaration, and that would be slow with top-down approach.
However, with a trigram index the IDE can apply an interesting optimization: only check those declarations which have few textual matches.
This will miss an used declaration with a popular name, like `new`, but will work ok for less-popular names, with a relatively good performance.

## Real World

Now it's time to look at what actually happens in rust-analyzer. First of all, I must confess, it doesn't use the bottom-up approach :)

Rust type-inference works at a function granularity: statements near the end of a function can affect statements at the beginning.
So, it doesn't make sense to do name resolution at the granularity of an expression, and indeed rust-analyzer builds a per-function [symbol table].
This is still done lazily though: we don't look into the function body unless the text search tells us to do so.

Name resolution on the module/item level in Rust is pretty complex as well.
The interaction between macros, which can bring new names into the scope, and glob imports, which can tie together namespaces of two modules, requires not only top-down processing, but a repeated top-down processing (until a fixed point is reached).
For this reason, module-level name resolution in rust-analyzer is also implemented using the top-down approach.
We use [salsa] to make this phase of name resolution incremental, as a substitute for laziness (see [this module][nameres] for details).
The results look promising so far: by processing function bodies lazy, we greatly reduce the amount of data the fixed-point iteration algorithm has to look at.
By adding salsa on-top, we avoid re-running this algorithm most of the time.

[symbol table]: https://github.com/rust-analyzer/rust-analyzer/blob/d523366299c8d4813e9845c9402b8dd7b779856a/crates/ra_hir/src/expr/scope.rs
[salsa]: https://github.com/salsa-rs/salsa
[nameres]: https://github.com/rust-analyzer/rust-analyzer/blob/d523366299c8d4813e9845c9402b8dd7b779856a/crates/ra_hir_def/src/nameres.rs

However, the general search funnel is there!

1. Here's the [entry point](https://github.com/rust-analyzer/rust-analyzer/blob/d523366299c8d4813e9845c9402b8dd7b779856a/crates/ra_ide_api/src/lib.rs#L383-L390) for find usages.
   Callee can restrict the `SearchScope`.
   For example, when the editor asks to highlight all usages of the identifier under the cursor, the scope is restricted to a single file.
2. The first step of find usages is figuring out what to find in the first place.
   This is handled by [`find_name`](https://github.com/rust-analyzer/rust-analyzer/blob/c486f8477aca4a42800e81b0b99fd56c14c6219f/crates/ra_ide_api/src/references.rs#L106-L120) functions.
   There are two cases to consider: the cursor can be either on the reference, or on the definition.
   We handle the first case by resolving the reference to the definition and converging to the second case.
3. Once we've figured out the definition, we compute it's search scope and intersect it with the provided scope: [source](https://github.com/rust-analyzer/rust-analyzer/blob/c486f8477aca4a42800e81b0b99fd56c14c6219f/crates/ra_ide_api/src/references.rs#L93-L99).
4. After that, we do a simple text search over all files in the scope: [source](https://github.com/rust-analyzer/rust-analyzer/blob/c486f8477aca4a42800e81b0b99fd56c14c6219f/crates/ra_ide_api/src/references.rs#L137).
   This is the place where trigram index should be added.
5. If there's a match, we parse the file, to make sure that it is indeed a reference, and not a comment or a string literal: [source](https://github.com/rust-analyzer/rust-analyzer/blob/c486f8477aca4a42800e81b0b99fd56c14c6219f/crates/ra_ide_api/src/references.rs#L135).
   Note how we use a local [Lazy](https://docs.rs/once_cell/1.2.0/once_cell/unsync/struct.Lazy.html) value to parse only those files, which have at least one match.
6. Finally, we check that the candidate reference indeed resolves to the definition we have started with: [source](https://github.com/rust-analyzer/rust-analyzer/blob/c486f8477aca4a42800e81b0b99fd56c14c6219f/crates/ra_ide_api/src/references.rs#L150)


That's all for the find usages, thank you for reading!

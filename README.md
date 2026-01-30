# SubVerso (Side-by-Side Blueprint Fork)

![Lean](https://img.shields.io/badge/Lean-v4.27.0-blue)
![License](https://img.shields.io/badge/License-Apache%202.0-green)

A fork of [SubVerso](https://github.com/leanprover/subverso) optimized for the [Side-by-Side Blueprint](https://github.com/e-vergo/Side-By-Side-Blueprint) toolchain.

## What is SubVerso?

SubVerso extracts syntax highlighting information from Lean code, including:
- Token classification (keywords, constants, variables, etc.)
- Type information for hover tooltips
- Proof state visualization
- Cross-reference linking between definition and usage sites

This data powers the interactive code displays in Verso documentation and the Side-by-Side Blueprint formalization toolchain.

## Fork Changes

This fork adds performance optimizations for blueprint artifact generation, where SubVerso highlighting accounts for 93-99% of build time. The optimizations focus on reducing redundant tree traversals and providing O(1) lookups.

### InfoTable Optimizations

The `InfoTable` structure pre-processes info trees once during elaboration, replacing repeated O(n) tree traversals with O(1) HashMap lookups:

```lean
structure InfoTable where
  tacticInfo : Compat.HashMap Compat.Syntax.Range (Array (ContextInfo × TacticInfo))
  infoByExactPos : Compat.HashMap (String.Pos.Raw × String.Pos.Raw) (Array (ContextInfo × Info))
  termInfoByName : Compat.HashMap Name (Array (ContextInfo × TermInfo))
  nameSuffixIndex : Compat.HashMap String (Array Name)
  allInfoSorted : Array (String.Pos.Raw × String.Pos.Raw × ContextInfo × Info)
```

**Index structures**:

| Field | Purpose | Complexity |
|-------|---------|------------|
| `infoByExactPos` | Info lookup by exact syntax position (start, end) | O(1) |
| `termInfoByName` | TermInfo lookup for const/fvar expressions by name | O(1) |
| `nameSuffixIndex` | Constant lookup by final name component (e.g., `"add"` finds `Nat.add`) | O(1) |
| `allInfoSorted` | Sorted array for containment queries with early exit | O(n) worst case |

**Lookup functions**:
- `lookupByExactPos`: O(1) info lookup by syntax position
- `lookupTermInfoByName`: O(1) TermInfo lookup for constants/fvars
- `lookupBySuffix`: O(1) constant lookup by final name component
- `lookupContaining`: Linear scan with early termination for containment queries

### Caching

The `HighlightState` includes caches to avoid recomputing expensive operations:

- `identKindCache`: Memoizes identifier classification by (position, name)
- `signatureCache`: Memoizes pretty-printed signatures by constant name
- `terms` / `ppTerms`: Caches rendered expressions from hovers and proof states
- `hasTacticCache` / `childHasTacticCache`: Memoizes tactic info searches

### Graceful Error Handling

Uses `throw <| IO.userError` for recoverable errors in critical paths instead of panicking. Locations include:
- Netstring protocol decoding (EOF handling, length parsing, message framing)
- Project loading (lakefile/toolchain validation)
- JSON deserialization (example parsing)

### Tactic Argument Highlighting

Enhanced highlighting for tactic arguments that reference theorems, enabling proper hover information even when identifiers appear in tactic contexts.

## Role in Dependency Chain

```
SubVerso -> LeanArchitect -> Dress -> Runway
```

SubVerso is the foundation of the toolchain. Changes here propagate to all downstream repositories.

## Installation

Add to your `lakefile.toml`:

```toml
[[require]]
name = "subverso"
git = "https://github.com/e-vergo/subverso.git"
rev = "main"
```

Or `lakefile.lean`:

```lean
require subverso from git "https://github.com/e-vergo/subverso.git"
```

## Related Repositories

**Upstream**:
- [leanprover/subverso](https://github.com/leanprover/subverso) - Original SubVerso

**Downstream** (depend on this fork):
- [LeanArchitect](https://github.com/e-vergo/LeanArchitect) - `@[blueprint]` attribute
- [Dress](https://github.com/e-vergo/Dress) - Artifact generation using SubVerso highlighting
- [Runway](https://github.com/e-vergo/Runway) - Site generator

**Parent project**:
- [Side-by-Side Blueprint](https://github.com/e-vergo/Side-By-Side-Blueprint) - Monorepo containing all toolchain components

---

# SubVerso - Verso's Library for Subprocesses

*Original upstream documentation follows.*

SubVerso is a support library that allows a
[Verso](https://github.com/leanprover/verso) document to describe Lean
code written in multiple versions of Lean. Verso itself may be tied to
new Lean versions, because it makes use of new compiler features. This
library will maintain broader compatibility with various Lean versions.

## Versions and Compatibility

SubVerso's CI currently validates it on every Lean release since
4.0.0, along with whatever version or snapshot is currently targeted
by Verso itself.

There should be no expectations of compatibility between different
versions of SubVerso, however - the specifics of its data formats is
an implementation detail, not a public API. Please use SubVerso itself
to read and write the data.

## Module System

The code in `main` uses the Lean module system. For compatibility with
older Lean versions, a "demodulized" version of the code is generated
on each commit to `main`. This is force-pushed to the `no-modules`
branch. The "demodulized" version has been rewritten by the script
`demodulize.py`. Additionally, the no-module code generated from commit
`abc123def456` on `main` is tagged with `no-modules/abc123def456` for posterity.

Versions of Lean prior to `4.25.0` or `nightly-2025-10-07` should use
the `no-modules` branch.

Projects which coordinate two Lean versions across the "module gap"
can still check that the SubVerso versions are the same in CI. The
file `.source-commit` in the `no-modules` branch contains the commit
hash of `main` that it was generated from.

## Features

### Code Examples

Presently, SubVerso supports the extraction of highlighting
information from code. There may be a many-to-many relationship
between Lean modules and documents that describe them. In these cases,
using SubVerso's examples library to indicate examples to be shown in
the text can be useful.

This feature may also be useful for other applications that require
careful presentation of Lean code.

To use this feature, first add a dependency on `subverso` to your
Lakefile:

```
require subverso from git "git@github.com:leanprover/subverso.git"
```

Examples are named in _anchors_, which are created using
specially-formatted comments. An anchor `A` is delimeted by:
```lean
-- ANCHOR: A
```
and:
```lean
-- ANCHOR_END: A
```

Within a Lean source file, anchor names should be unique. Anchors may
overlap arbitrarily.

Within an anchor, a proof state may be named `X` using the
`PROOF_STATE X` comment, which points at the source position of the
state using a `^`. Here, the proof state `after_intro` is the one
active around the `r`:

```lean
example : ∀n, n * (2 + 2) = n * 4 := by
  -- ANCHOR: proof
  intro n
  -- ^ PROOF_STATE: after_intro
  grind
  -- ANCHOR_END: proof
```

### Module Extraction

SubVerso can be used to extract a representation of a module's code
together with metadata. Run:

```
$ subverso-extract-mod MODNAME OUT.json
```

to extract metadata about module `MODNAME` into `OUT.json`. The
resulting JSON file contains an array of objects. Lean modules are
sequences of commands, and each object represents one of them. The
objects have the following keys:
 * `kind` - the syntax kind of the command
 * `defines` - names defined by the command (useful e.g. for automatic hyperlink insertion in rendered HTML)
 * `code` - the internal SubVerso JSON format for the highlighted
   code, including proof states, which is intended to be deserialized
   using the `FromJson Highlighted` instance from SubVerso.

The `highlighted` facet for a package, library, or module builds
highlighted sources.

### Helper Process

SubVerso can also be used as a helper for other tools that need to be
more interactive than module extraction, and yet still cross Lean
version boundaries. Verso uses this feature to attempt to highlight
code samples in Markdown module docstrings when using its "literate
Lean" blog post feature.

To start up the helper, run `subverso-helper`. It communicates with a
protocol reminiscent of JSON-RPC, but this is an implementation
detail - it should be used via the API in `SubVerso.Helper`. It can
presently be used to elaborate and highlight terms in the context of a
module.

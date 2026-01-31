# SubVerso (Side-by-Side Blueprint Fork)

> **Attribution:** This is a fork of [leanprover/subverso](https://github.com/leanprover/subverso) by David Thrane Christiansen. The original SubVerso is a support library for [Verso](https://github.com/leanprover/verso) documentation that extracts syntax highlighting and semantic information from Lean code.

This fork is maintained for the [Side-by-Side Blueprint](https://github.com/e-vergo/Side-By-Side-Blueprint) formalization documentation toolchain.

## Overview

SubVerso extracts syntax highlighting and semantic information from Lean code during elaboration. It produces:

- Token classification (keywords, constants, variables, string literals)
- Type signatures for hover tooltips
- Proof state visualization at tactic positions
- Cross-reference linking between definition and usage sites

This data is consumed by Verso documentation and the Side-by-Side Blueprint toolchain to render interactive Lean code displays.

## Fork Modifications

This fork addresses performance and robustness issues discovered during blueprint artifact generation, where SubVerso highlighting accounts for 93-99% of total build time. The modifications fall into three categories: indexed lookups, caching, and improved identifier resolution.

### InfoTable Structure

The original SubVerso traverses the info tree repeatedly during highlighting. This fork introduces `InfoTable`, which pre-processes the tree once and provides O(1) lookups via HashMap indices:

```lean
structure InfoTable where
  tacticInfo : HashMap Syntax.Range (Array (ContextInfo × TacticInfo))
  infoByExactPos : HashMap (String.Pos.Raw × String.Pos.Raw) (Array (ContextInfo × Info))
  termInfoByName : HashMap Name (Array (ContextInfo × TermInfo))
  nameSuffixIndex : HashMap String (Array Name)
  allInfoSorted : Array (String.Pos.Raw × String.Pos.Raw × ContextInfo × Info)
```

| Field | Purpose | Complexity |
|-------|---------|------------|
| `tacticInfo` | Tactic info lookup by canonical syntax range | O(1) |
| `infoByExactPos` | Info lookup by exact syntax position (start, end) | O(1) |
| `termInfoByName` | TermInfo lookup for const/fvar expressions by name | O(1) |
| `nameSuffixIndex` | Constant lookup by final name component (e.g., `"add"` finds `Nat.add`) | O(1) |
| `allInfoSorted` | Sorted array for containment queries with early exit | O(n) worst case |

The table is built once per file via `InfoTable.ofInfoTrees`, then queried through:

- `lookupByExactPos`: O(1) lookup by syntax position
- `lookupTermInfoByName`: O(1) lookup for constants and free variables by name
- `lookupBySuffix`: O(1) lookup for constants by their final name component
- `lookupContaining`: Linear scan with early termination for containment queries (since array is sorted by start position, scan exits when start position exceeds query start)

### HighlightState Caches

The `HighlightState` structure includes memoization for expensive operations:

```lean
structure HighlightState where
  -- ... message and output state ...
  hasTacticCache : HashMap Syntax.Range (Array (Syntax × Bool))
  childHasTacticCache : HashMap Syntax.Range (Array (Syntax × Bool))
  terms : HashMap Expr Highlighted
  ppTerms : HashMap Expr CodeWithInfos
  identKindCache : HashMap (String.Pos.Raw × Name) Token.Kind
  signatureCache : HashMap Name String
```

- `identKindCache`: Memoizes identifier classification by (position, name), avoiding redundant info tree traversals
- `signatureCache`: Memoizes pretty-printed type signatures by constant name
- `terms` / `ppTerms`: Caches rendered expressions from hovers and proof states
- `hasTacticCache` / `childHasTacticCache`: Memoizes tactic info searches by syntax range

### Enhanced Identifier Resolution

The `identKind'` function (optimized for `HighlightM`) uses a multi-stage fallback strategy to resolve identifiers that lack direct info tree matches:

1. **O(1) exact position lookup**: Use `InfoTable.lookupByExactPos` for info nodes with matching position
2. **Containment search**: Use `InfoTable.lookupContaining` to find info nodes whose range contains the identifier (handles tactic arguments)
3. **O(1) name-based search**: Use `InfoTable.lookupTermInfoByName` to search for TermInfo by identifier name regardless of position (handles macro expansion)
4. **Environment lookup**: Look up the identifier directly in the environment with suffix matching (handles simp lemma arguments and raw name references)

This addresses cases where position-based matching fails due to macro expansion or tactic elaboration.

### Graceful Error Handling

The fork replaces panics with recoverable errors in critical paths:

- `InfoTable.ofInfoTree`: Skips contextless nodes instead of panicking; processes children to find valid context
- `SplitCtx.close` (Split.lean): Returns current unchanged as safe fallback instead of panicking on empty context stack
- `highlightLevel`: Emits unknown token instead of panicking on unrecognized level syntax
- `emitToken`: Handles synthetic source info from macros and term-mode proofs (no leading/trailing whitespace available)

## Dependency Chain

```
SubVerso -> LeanArchitect -> Dress -> Runway
              |
              +-> Verso (genres use SubVerso for highlighting)
```

SubVerso is the foundation of the Side-by-Side Blueprint toolchain. Changes here propagate to all downstream repositories.

## Installation

Add to `lakefile.toml`:

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

## API Reference

### Highlighting

The primary entry point for syntax highlighting:

```lean
def highlight (stx : Syntax) (messages : Array Message)
    (trees : PersistentArray InfoTree) (suppressNamespaces : List Name := [])
    : TermElabM Highlighted
```

For code that may include unparsed regions:

```lean
def highlightIncludingUnparsed (stx : Syntax) (messages : Array Message)
    (trees : PersistentArray InfoTree) (suppressNamespaces : List Name := [])
    (startPos? : Option String.Pos := none) (endPos? : Option String.Pos := none)
    : TermElabM Highlighted
```

### Highlighted Data Type

The output is a tree of `Highlighted` values:

```lean
inductive Highlighted where
  | token (tok : Token)
  | text (str : String)
  | seq (highlights : Array Highlighted)
  | span (info : Array (Span.Kind × MessageContents Highlighted)) (content : Highlighted)
  | tactics (info : Array (Goal Highlighted)) (startPos endPos : Nat) (content : Highlighted)
  | point (kind : Span.Kind) (info : MessageContents Highlighted)
  | unparsed (str : String)
```

Tokens carry semantic information:

```lean
inductive Token.Kind where
  | keyword (name : Option Name) (occurrence : Option String) (docs : Option String)
  | const (name : Name) (signature : String) (docs : Option String) (isDef : Bool)
  | var (name : FVarId) (type : String)
  | str (string : String)
  | sort (doc? : Option String)
  -- ... level operators, module names, etc.
```

### Proof State Extraction

```lean
def highlightGoals (ci : ContextInfo) (goals : List MVarId)
    : HighlightM (Array (Goal Highlighted))

def highlightProofState (ci : ContextInfo) (goals : List MVarId)
    (trees : PersistentArray InfoTree) (suppressNamespaces : List Name := [])
    : TermElabM (Array (Goal Highlighted))
```

### InfoTable Queries

```lean
def InfoTable.ofInfoTrees (ts : Array InfoTree) (init : InfoTable := {}) : InfoTable
def InfoTable.lookupByExactPos (table : InfoTable) (stx : Syntax) : Array (ContextInfo × Info)
def InfoTable.lookupTermInfoByName (table : InfoTable) (name : Name) : Array (ContextInfo × TermInfo)
def InfoTable.lookupBySuffix (table : InfoTable) (suffix : String) : Array Name
def InfoTable.lookupContaining (table : InfoTable) (stx : Syntax) : Array (ContextInfo × Info)
def InfoTable.buildSuffixIndex (env : Environment) (table : InfoTable) : InfoTable
```

## Code Examples and Anchors

SubVerso supports extracting named code regions using anchor comments:

```lean
-- ANCHOR: example_name
def myFunction : Nat := 42
-- ANCHOR_END: example_name
```

Proof states can be named within anchors:

```lean
example : n + 0 = n := by
  -- ANCHOR: proof
  induction n with
  -- ^ PROOF_STATE: after_induction
  | zero => rfl
  | succ n ih => simp [ih]
  -- ANCHOR_END: proof
```

## Module Extraction

Extract highlighted module content to JSON:

```
$ subverso-extract-mod ModuleName output.json
```

The output contains an array of command objects with:
- `kind`: Syntax kind of the command
- `defines`: Names defined by the command
- `code`: Highlighted code in SubVerso JSON format

## Lake Facets

This package provides Lake facets for build integration:

| Facet | Level | Output |
|-------|-------|--------|
| `highlighted` | Module | JSON file with highlighted code |
| `highlighted` | Library | Directory containing all module JSON files |
| `highlighted` | Package | Directory containing all library outputs |
| `examples` | Module | JSON file with extracted examples |
| `examples` | Library | Directory containing all module examples |
| `examples` | Package | Directory containing all library examples |

## Helper Process

SubVerso can run as a helper process for tools that need interactive highlighting across Lean version boundaries. Start with `subverso-helper`. Communication uses a netstring-based protocol (see `SubVerso.Helper`).

## Related Repositories

**Upstream**:
- [leanprover/subverso](https://github.com/leanprover/subverso) - Original SubVerso by David Thrane Christiansen

**Downstream** (depend on this fork):
- [LeanArchitect](https://github.com/e-vergo/LeanArchitect) - `@[blueprint]` attribute definition
- [Dress](https://github.com/e-vergo/Dress) - Artifact generation using SubVerso highlighting
- [Runway](https://github.com/e-vergo/Runway) - Site generator

**Parent project**:
- [Side-by-Side Blueprint](https://github.com/e-vergo/Side-By-Side-Blueprint) - Monorepo for the complete toolchain

## Compatibility

This fork tracks upstream releases and nightlies. The `Compat.lean` module provides compatibility shims for API differences across Lean versions, including:

- String position types (`String.Pos` vs `String.Pos.Raw`)
- HashMap/HashSet implementations (`Std.HashMap` vs `Lean.HashMap`)
- Environment extension registration (`asyncMode` parameter)
- Various renamed functions across Lean versions

## License

Apache 2.0, following the original SubVerso license.

---

## Upstream SubVerso Documentation

*The following is preserved from the original project for reference.*

SubVerso is a support library that allows a [Verso](https://github.com/leanprover/verso) document to describe Lean code written in multiple versions of Lean. Verso itself may be tied to new Lean versions, because it makes use of new compiler features. This library will maintain broader compatibility with various Lean versions.

### Versions and Compatibility

SubVerso's CI currently validates it on every Lean release since 4.0.0, along with whatever version or snapshot is currently targeted by Verso itself.

There should be no expectations of compatibility between different versions of SubVerso, however - the specifics of its data formats is an implementation detail, not a public API. Please use SubVerso itself to read and write the data.

### Module System

The code in `main` uses the Lean module system. For compatibility with older Lean versions, a "demodulized" version of the code is generated on each commit to `main`. This is force-pushed to the `no-modules` branch.

Versions of Lean prior to `4.25.0` or `nightly-2025-10-07` should use the `no-modules` branch.

### Helper Process

SubVerso can also be used as a helper for other tools that need to be more interactive than module extraction, and yet still cross Lean version boundaries. Verso uses this feature to attempt to highlight code samples in Markdown module docstrings when using its "literate Lean" blog post feature.

To start up the helper, run `subverso-helper`. It communicates with a protocol reminiscent of JSON-RPC, but this is an implementation detail - it should be used via the API in `SubVerso.Helper`.

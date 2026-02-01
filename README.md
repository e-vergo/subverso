# SubVerso (Side-by-Side Blueprint Fork)

Fork of [leanprover/subverso](https://github.com/leanprover/subverso) by David Thrane Christiansen.

## Fork Purpose

This fork addresses performance and robustness issues discovered during blueprint artifact generation, where **SubVerso highlighting accounts for 93-99% of total build time**. The modifications provide O(1) indexed lookups for info tree queries, enabling syntax highlighting to scale to large formalization projects like PNT+ (591 annotated declarations).

## Key Modification: InfoTable

The original SubVerso traverses the info tree repeatedly during highlighting. This fork introduces `InfoTable`, which pre-processes the tree once and provides O(1) lookups via HashMap indices:

```lean
structure InfoTable where
  tacticInfo       : HashMap Syntax.Range (Array (ContextInfo × TacticInfo))
  infoByExactPos   : HashMap (String.Pos × String.Pos) (Array (ContextInfo × Info))
  termInfoByName   : HashMap Name (Array (ContextInfo × TermInfo))
  nameSuffixIndex  : HashMap String (Array Name)
  allInfoSorted    : Array (String.Pos × String.Pos × ContextInfo × Info)
```

| Field | Purpose | Complexity |
|-------|---------|------------|
| `infoByExactPos` | Info lookup by exact syntax position (start, end) | O(1) |
| `termInfoByName` | TermInfo lookup for const/fvar expressions by name | O(1) |
| `nameSuffixIndex` | Constant lookup by final name component | O(1) |
| `tacticInfo` | Tactic info lookup by canonical syntax range | O(1) |
| `allInfoSorted` | Sorted array for containment queries with early exit | O(n) worst |

### Query Functions

| Function | Complexity | Purpose |
|----------|------------|---------|
| `lookupByExactPos` | O(1) | Lookup by syntax position |
| `lookupTermInfoByName` | O(1) | Lookup constants/free variables by name |
| `lookupBySuffix` | O(1) | Lookup constants by final name component |
| `lookupContaining` | O(n) | Containment queries with early termination |

## Additional Optimizations

### HighlightState Caches

Memoization caches for expensive operations:

| Cache | Purpose |
|-------|---------|
| `identKindCache` | Identifier classification by (position, name) |
| `signatureCache` | Pretty-printed type signatures by constant name |
| `hasTacticCache` / `childHasTacticCache` | Tactic info presence |

### Identifier Resolution

Multi-stage fallback strategy in `identKind'` for identifiers lacking direct info tree matches:

1. **O(1) exact position lookup** via `infoByExactPos`
2. **Containment search** for tactic arguments
3. **O(1) name-based search** for macro expansion cases
4. **Environment lookup with suffix matching** for simp lemma arguments

### Error Handling

Replaces panics with recoverable errors:

| Location | Change |
|----------|--------|
| `InfoTable.ofInfoTree` | Skips contextless nodes instead of panicking |
| `SplitCtx.close` | Returns safe fallback on empty context stack |
| `highlightLevel` | Emits unknown token on unrecognized syntax |
| `emitToken` | Handles synthetic source info from macros and term-mode proofs |

## Installation

```toml
# lakefile.toml
[[require]]
name = "subverso"
git = "https://github.com/e-vergo/subverso.git"
rev = "main"
```

## Dependency Chain

```
SubVerso -> LeanArchitect -> Dress -> Runway
              |
              +-> Verso (genres use SubVerso for highlighting)
```

SubVerso is the foundation of the Side-by-Side Blueprint toolchain. Changes here propagate to all downstream repositories.

## Key Files

| File | Purpose |
|------|---------|
| `src/SubVerso/Highlighting/Code.lean` | InfoTable structure and query functions |
| `src/SubVerso/Highlighting/Highlighted.lean` | Token.Kind and Highlighted types |

## Related Repositories

| Repository | Purpose |
|------------|---------|
| [leanprover/subverso](https://github.com/leanprover/subverso) | Upstream (David Thrane Christiansen) |
| [Side-by-Side Blueprint](https://github.com/e-vergo/Side-By-Side-Blueprint) | Parent project |

## License

Apache 2.0, following the upstream license.

# SubVerso (Side-by-Side Blueprint Fork)

**Upstream:** [leanprover/subverso](https://github.com/leanprover/subverso) by David Thrane Christiansen

**Parent project:** [Side-by-Side Blueprint](https://github.com/e-vergo/Side-By-Side-Blueprint)

## Fork Purpose

SubVerso extracts semantic syntax highlighting from Lean info trees. In blueprint artifact generation, **highlighting accounts for 93-99% of total build time** (800-6500ms per declaration). The original implementation traverses info trees repeatedly, which becomes prohibitive at scale.

This fork introduces O(1) indexed lookups via `InfoTable`, enabling projects like PNT+ (591 annotated declarations) to build in reasonable time.

## SBS-Specific Modifications

### InfoTable Structure

Pre-processes the info tree once into HashMap indices for O(1) queries:

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

### HighlightState Caches

Memoization caches for expensive repeated operations:

| Cache | Purpose |
|-------|---------|
| `identKindCache` | Identifier classification by (position, name) |
| `signatureCache` | Pretty-printed type signatures by constant name |
| `hasTacticCache` | Tactic info presence |

### Identifier Resolution

Multi-stage fallback strategy in `identKind'` for identifiers lacking direct info tree matches:

1. **O(1) exact position lookup** via `infoByExactPos`
2. **Containment search** for tactic arguments
3. **O(1) name-based search** for macro expansion cases
4. **Environment lookup with suffix matching** for simp lemma arguments

### Robustness Fixes

Replaces panics with recoverable errors for edge cases encountered in production:

| Location | Change |
|----------|--------|
| `InfoTable.ofInfoTree` | Skips contextless nodes instead of panicking |
| `SplitCtx.close` | Returns safe fallback on empty context stack |
| `highlightLevel` | Emits unknown token on unrecognized syntax |
| `emitToken` | Handles synthetic source info from macros and term-mode proofs |

## Files Modified

| File | Changes |
|------|---------|
| `src/SubVerso/Highlighting/Code.lean` | InfoTable structure, HashMap indices, query functions, HighlightState caches |
| `src/SubVerso/Highlighting/Highlighted.lean` | Token.Kind, Highlighted types |

## Dependency Chain

```
SubVerso -> LeanArchitect -> Dress -> Runway
              |
              +-> Verso (genres use SubVerso for highlighting)
```

SubVerso is the foundation of the Side-by-Side Blueprint toolchain. Changes here propagate to all downstream repositories.

## Installation

```toml
# lakefile.toml
[[require]]
name = "subverso"
git = "https://github.com/e-vergo/subverso.git"
rev = "main"
```

Most projects depend on SubVerso transitively via Dress. Direct dependency is only needed for custom highlighting integration.

## Tooling

For build commands, screenshot capture, compliance validation, and other CLI tooling, see the [Archive & Tooling Hub](../archive/README.md).

## License

Apache 2.0, following the upstream license.

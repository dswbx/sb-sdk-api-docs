
## Plan: OBJECT2.md v3

Rewrite OBJECT2.md incorporating all findings. Ordered by dependency (each step builds on previous).

### Step 1 — Core structure overhaul
- Remove `embed` top-level field entirely
- Add `join` top-level field (alias-keyed, `from`/`type`/`hint`)
- Merge embeds into `select` array (alias-keyed objects with nested `select`/`where`/`order`/`limit`)
- Rename `property` → `column` in select objects
- Add `spread` to select embed entries
- Update core structure diagram, root fields table, all examples
- Affects: sections 1-6, Select Formats, Embedding section, Root Fields Summary, all Examples

### Step 2 — Select format: `cast`, `aggregate`, `preCast`
- Add `cast` field to select objects (type casting)
- Add `aggregate` field (`sum`/`avg`/`max`/`min`/`count`)
- Add `preCast` field (cast before aggregate, rare)
- Add top-level `group` field
- `aggregate` without `column` → `COUNT(*)`, with `column` → `COUNT(col)`
- Update Select Formats section, add examples
- Affects: Select Formats, section 1 (Query), new Aggregates subsection

### Step 3 — Operators overhaul
- Add `$is` operator (`true`/`false`/`unknown`/`null`)
- Replace `$likeAnyOf`/`$likeAllOf`/`$ilikeAnyOf`/`$ilikeAllOf` → `$likeAny`/`$likeAll`/`$ilikeAny`/`$ilikeAll`
- Add: `$eqAny`, `$eqAll`, `$gtAny`, `$gtAll`, `$gteAny`, `$gteAll`, `$ltAny`, `$ltAll`, `$lteAny`, `$lteAll`, `$regexAny`, `$regexAll`, `$iregexAny`, `$iregexAll`
- Change `$not` to wrapper-only (drop `$not.eq` prefix form). Works on individual ops and compounds.
- Update Operators section (Comparison, Pattern Matching, Logical), add examples
- Affects: Operators section, Logical subsection

### Step 4 — `$meta` cleanup
- Rename `defaultsToNull` → `missing: "null" | "default"`
- Add `handling: "strict" | "lenient"`
- Add `timezone: "America/Los_Angeles"`
- Add `columns: [...]` (insert column restriction)
- Add `stripNulls: true`
- Update $meta Fields table
- Affects: $meta Fields section, section 2 (Insert), section 5 (Upsert)

### Step 5 — Operations: `put` + RPC
- Add `type: "put"` with full-replace semantics
- Add `httpMethod`, `paramsType`, `inputType` to RPC
- Update operation types list, add section 7 (Put), update section 6 (RPC)
- Affects: Core Structure, sections 6-7, Root Fields Summary

### Step 6 — Order format + parser notes
- Add dot notation for join columns in `order` (`director.last_name`)
- Add dot notation for join columns in `where` (`actors.first_name`)
- Add parser note: join alias in `where` without dot = embed null check
- Add parser note: `column` vs join alias precedence
- Update Order Format, Parser Notes sections

### Step 7 — Changelog + cleanup
- Update "Changes from v2" section
- Remove old Embedding section (now merged into select)
- Update Coverage section
- Update Open Questions (remove answered, add any new)
- Final consistency pass: all examples use `column` not `property`, all use `$not` wrapper, all use camelCase quantifiers

### Unresolved before starting
- None blocking. All questions answered. Can proceed.
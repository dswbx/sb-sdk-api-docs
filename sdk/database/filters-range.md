# Range Filters (Brief)

**Category:** LOW Priority | **Status:** ✅ Complete
**Methods:** 5 (rangeGt, rangeGte, rangeLt, rangeLte, rangeAdjacent)

PostgreSQL range type operators. **SQLite has no native range types.**

---

## `rangeGt(column, range)`

**Priority:** Low | **SQLite:** ❌ No

**Signature:** `rangeGt(column: string, range: string): this`

**Purpose:** Match rows where every element in column range is greater than any element in range.

**Usage:** `client.from('schedules').select().rangeGt('time_slot', '[9:00,10:00]')`

**SQLite:** ❌ No native range types. Use separate start/end columns + manual comparison.

**Source:** PostgrestFilterBuilder.ts:463-475

---

## `rangeGte(column, range)`

**Priority:** Low | **SQLite:** ❌ No

**Signature:** `rangeGte(column: string, range: string): this`

**Purpose:** Match rows where column range is either contained in or greater than range.

**Usage:** `client.from('schedules').select().rangeGte('time_slot', '[9:00,10:00]')`

**SQLite:** ❌ No native range types. Workaround: `WHERE start >= '10:00'` (using separate columns).

**Source:** PostgrestFilterBuilder.ts:477-490

---

## `rangeLt(column, range)`

**Priority:** Low | **SQLite:** ❌ No

**Signature:** `rangeLt(column: string, range: string): this`

**Purpose:** Match rows where every element in column range is less than any element in range.

**Usage:** `client.from('schedules').select().rangeLt('time_slot', '[9:00,10:00]')`

**SQLite:** ❌ No native range types. Workaround: `WHERE end < '9:00'` (using separate columns).

**Source:** PostgrestFilterBuilder.ts:492-504

---

## `rangeLte(column, range)`

**Priority:** Low | **SQLite:** ❌ No

**Signature:** `rangeLte(column: string, range: string): this`

**Purpose:** Match rows where column range is either contained in or less than range.

**Usage:** `client.from('schedules').select().rangeLte('time_slot', '[9:00,10:00]')`

**SQLite:** ❌ No native range types. Workaround: `WHERE end <= '9:00'` (using separate columns).

**Source:** PostgrestFilterBuilder.ts:506-519

---

## `rangeAdjacent(column, range)`

**Priority:** Low | **SQLite:** ❌ No

**Signature:** `rangeAdjacent(column: string, range: string): this`

**Purpose:** Match rows where column range is adjacent to range (mutually exclusive, no gap).

**Usage:** `client.from('schedules').select().rangeAdjacent('time_slot', '[10:00,11:00]')`

**SQLite:** ❌ No native range types. Workaround: `WHERE end = '10:00' OR start = '11:00'` (using separate columns).

**Source:** PostgrestFilterBuilder.ts:521-534

---

## Summary

All range filters require PostgreSQL range types (int4range, tsrange, daterange, etc.). SQLite has no equivalent.

**SQLite Workaround:**
- Store ranges as two columns: `start`, `end`
- Implement range logic manually:
  - `rangeGt(r)`: `start > r.end`
  - `rangeLt(r)`: `end < r.start`
  - Overlap: `NOT (end < r.start OR start > r.end)`
  - Adjacent: `end = r.start OR start = r.end`

**Recommendation:** Avoid if targeting SQLite. Use separate columns + manual filters.

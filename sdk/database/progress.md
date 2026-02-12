# Database API Documentation Progress

**Status:** ✅ Complete
**Category:** Database (1 of 5)
**Progress:** 66/66 methods documented (100%)
**Priority:** HIGH ✅, MEDIUM ✅, LOW ✅, Reference ✅

---

## Checklist

### HIGH Priority [21/21 methods] ✅
- [ ] api-overview.md (deferred to end)
- [x] core-operations.md (6 methods)
  - [x] from()
  - [x] select()
  - [x] insert()
  - [x] update()
  - [x] delete()
  - [x] upsert()
- [x] filters-comparison.md (10 methods)
  - [x] eq()
  - [x] neq()
  - [x] gt()
  - [x] gte()
  - [x] lt()
  - [x] lte()
  - [x] in()
  - [x] is()
  - [x] isDistinct()
  - [x] notIn()
- [x] transforms.md (5 methods)
  - [x] order()
  - [x] limit()
  - [x] range()
  - [x] single()
  - [x] maybeSingle()

### MEDIUM Priority [20/20 methods] ✅
- [x] filters-pattern-matching.md (6 methods)
  - [x] like()
  - [x] ilike()
  - [x] likeAllOf()
  - [x] likeAnyOf()
  - [x] ilikeAllOf()
  - [x] ilikeAnyOf()
- [x] filters-array-json.md (3 methods)
  - [x] contains()
  - [x] containedBy()
  - [x] overlaps()
- [x] filters-advanced.md (5 methods)
  - [x] textSearch()
  - [x] match()
  - [x] or()
  - [x] not()
  - [x] filter()
- [x] response-shaping.md (3 methods)
  - [x] csv()
  - [x] geojson()
  - [x] explain()
- [x] utilities.md (3 methods)
  - [x] abortSignal()
  - [x] setHeader()
  - [x] throwOnError()

### LOW Priority [31/31 methods] ✅
- [x] filters-range.md (5 methods, brief)
  - [x] rangeGt()
  - [x] rangeGte()
  - [x] rangeLt()
  - [x] rangeLte()
  - [x] rangeAdjacent()
- [x] filters-specialized.md (26+ methods, brief)
  - [x] regexMatch()
  - [x] regexIMatch()
  - [x] maxAffected()
  - [x] returns()
  - [x] overrideTypes()
  - [x] rollback()
  - [x] schema()
  - [x] rpc()
  - [x] Other specialized methods

### Reference Docs [4/4] ✅
- [x] api-overview.md
- [x] sqlite-compatibility-matrix.md
- [x] type-casting-reference.md
- [x] resource-embedding.md

---

## Completed ✅

**All documentation complete!**

### Files Created (16):
1. progress.md - Progress tracking
2. core-operations.md - 6 CRUD methods (HIGH)
3. filters-comparison.md - 10 comparison filters (HIGH)
4. transforms.md - 5 transform methods (HIGH)
5. filters-pattern-matching.md - 6 LIKE pattern filters (MEDIUM)
6. filters-array-json.md - 3 array/JSON filters (MEDIUM)
7. filters-advanced.md - 5 advanced filters (MEDIUM)
8. response-shaping.md - 3 response format methods (MEDIUM)
9. utilities.md - 3 utility methods (MEDIUM)
10. filters-range.md - 5 range operators (LOW, brief)
11. filters-specialized.md - 26+ specialized methods (LOW, brief)
12. api-overview.md - Complete method reference
13. sqlite-compatibility-matrix.md - Comprehensive compatibility analysis
14. type-casting-reference.md - PostgreSQL :: cast → SQLite CAST() mapping
15. resource-embedding.md - Joins, spread, !inner, disambiguation, nested embedding

### Statistics:
- **Total methods documented:** 66
- **HIGH priority:** 21 methods (full detail)
- **MEDIUM priority:** 20 methods (moderate detail)
- **LOW priority:** 25+ methods (brief format)
- **SQLite compatibility:**
  - ✅ Full: 33/66 (50%)
  - ⚠️ Partial: 20/66 (30%)
  - ❌ No: 8/66 (12%)
  - N/A: 5/66 (8%)

---

## Next Phase

Database API documentation (1 of 5 categories) is **complete**.

**Remaining categories:**
1. ✅ Database (~66 methods) - **DONE**
2. ⏳ Auth (~40 methods) - TODO
3. ⏳ Storage (~25 methods) - TODO
4. ⏳ Realtime (~50 methods) - TODO
5. ⏳ Functions (~2 methods) - TODO

Each category will follow similar structure:
- Priority-based documentation (HIGH → MEDIUM → LOW)
- SQLite compatibility analysis
- Usage examples from source/tests
- API overview + compatibility matrix

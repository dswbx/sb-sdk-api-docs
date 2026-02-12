# Resource Embedding (Joins)

**Category:** HIGH Priority | **Status:** ✅ Complete
**PostgREST docs:** https://docs.postgrest.org/en/v14/references/api/resource_embedding.html

---

## Overview

Resource embedding lets you fetch related tables in a single query via foreign key relationships. PostgREST auto-detects relationships; no manual JOIN syntax needed.

**Syntax:** `table(columns)` inside `select()`

```ts
const { data } = await client
  .from('films')
  .select('title, directors(id, last_name)')
```

```
GET /films?select=title,directors(id,last_name)
```

---

## Relationship Types

### Many-to-One

Foreign key on current table → parent record returned as **single object**.

```ts
// films.director_id → directors.id
const { data } = await client
  .from('films')
  .select('title, directors(id, last_name)')

// Result: { title: "...", directors: { id: 1, last_name: "Doe" } }
```

```sql
-- SQLite (flat columns, requires post-processing)
SELECT f.title, d.id AS "directors.id", d.last_name AS "directors.last_name"
FROM films f
LEFT JOIN directors d ON f.director_id = d.id;

-- SQLite (JSON — returns nested object directly, no post-processing)
SELECT f.title,
       json_object('id', d.id, 'last_name', d.last_name) AS directors
FROM films f
LEFT JOIN directors d ON f.director_id = d.id;
```

### One-to-Many

Foreign key on related table → related records returned as **array**.

```ts
// films.director_id → directors.id (queried from directors side)
const { data } = await client
  .from('directors')
  .select('last_name, films(title)')

// Result: { last_name: "Doe", films: [{ title: "..." }, { title: "..." }] }
```

```sql
-- SQLite (flat rows, requires post-processing to group)
SELECT d.last_name, f.title AS "films.title"
FROM directors d
LEFT JOIN films f ON f.director_id = d.id;

-- SQLite (JSON — returns array directly)
SELECT d.last_name,
       COALESCE(
         (SELECT json_group_array(json_object('title', f.title))
          FROM films f WHERE f.director_id = d.id),
         '[]'
       ) AS films
FROM directors d;
```

### Many-to-Many

Auto-detected through join table (composite PK with FKs to both tables).

```ts
// actors ← roles (actor_id, film_id) → films
const { data } = await client
  .from('actors')
  .select('first_name, last_name, films(title)')

// Result: { first_name: "...", films: [{ title: "..." }, ...] }
```

```sql
-- SQLite (flat rows, requires post-processing to group)
SELECT a.first_name, a.last_name, f.title AS "films.title"
FROM actors a
LEFT JOIN roles r ON r.actor_id = a.id
LEFT JOIN films f ON r.film_id = f.id;

-- SQLite (JSON — returns array directly)
SELECT a.first_name, a.last_name,
       COALESCE(
         (SELECT json_group_array(json_object('title', f.title))
          FROM roles r
          JOIN films f ON r.film_id = f.id
          WHERE r.actor_id = a.id),
         '[]'
       ) AS films
FROM actors a;
```

### One-to-One

FK is also PK or has unique constraint → returned as **single object**.

```ts
// technical_specs.film_id (PK + FK) → films.id
const { data } = await client
  .from('films')
  .select('title, technical_specs(camera)')

// Result: { title: "...", technical_specs: { camera: "..." } }
```

```sql
-- SQLite (flat columns, requires post-processing)
SELECT f.title, ts.camera AS "technical_specs.camera"
FROM films f
LEFT JOIN technical_specs ts ON ts.film_id = f.id;

-- SQLite (JSON — returns nested object directly)
SELECT f.title,
       json_object('camera', ts.camera) AS technical_specs
FROM films f
LEFT JOIN technical_specs ts ON ts.film_id = f.id;
```

---

## Nested Embedding

Chain multiple levels deep.

```ts
const { data } = await client
  .from('actors')
  .select('first_name, roles(character, films(title, year))')
```

```sql
-- SQLite (flat rows, requires post-processing)
SELECT a.first_name,
       r.character AS "roles.character",
       f.title AS "roles.films.title",
       f.year AS "roles.films.year"
FROM actors a
LEFT JOIN roles r ON r.actor_id = a.id
LEFT JOIN films f ON r.film_id = f.id;

-- SQLite (JSON — fully nested in one query)
SELECT a.first_name,
       COALESCE(
         (SELECT json_group_array(
           json_object(
             'character', r.character,
             'films', json_object('title', f.title, 'year', f.year)
           ))
          FROM roles r
          JOIN films f ON r.film_id = f.id
          WHERE r.actor_id = a.id),
         '[]'
       ) AS roles
FROM actors a;
```

---

## Join Types

### Default (LEFT JOIN)

All parent rows returned. Embedded resource is `null` or `[]` if no match.

```ts
const { data } = await client
  .from('films')
  .select('title, actors(*)')
```

```sql
-- SQLite (flat rows, requires post-processing)
SELECT f.title, a.*
FROM films f
LEFT JOIN roles r ON r.film_id = f.id
LEFT JOIN actors a ON r.actor_id = a.id;

-- SQLite (JSON — returns actors as nested array)
SELECT f.title,
       COALESCE(
         (SELECT json_group_array(json_object('id', a.id, 'first_name', a.first_name, 'last_name', a.last_name))
          FROM roles r
          JOIN actors a ON r.actor_id = a.id
          WHERE r.film_id = f.id),
         '[]'
       ) AS actors
FROM films f;
-- Films without actors get empty array '[]'
```

### Inner Join (`!inner`)

Only parent rows **with matching** embedded records returned.

```ts
const { data } = await client
  .from('films')
  .select('title, actors!inner(*)')
  .eq('actors.first_name', 'Jehanne')

// Only films that have an actor named Jehanne
```

```sql
-- SQLite
SELECT f.title, a.*
FROM films f
INNER JOIN roles r ON r.film_id = f.id
INNER JOIN actors a ON r.actor_id = a.id
WHERE a.first_name = 'Jehanne';
-- Films without matching actors excluded entirely
```

### Left Join (`!left`)

Explicit left join — same as default. Noise word for clarity.

```ts
const { data } = await client
  .from('films')
  .select('title, actors!left(*)')
```

---

## Disambiguation (`!hint`)

When multiple foreign keys exist between two tables, PostgREST requires a hint.

```ts
// orders has billing_address_id AND shipping_address_id → addresses
const { data } = await client
  .from('orders')
  .select(`
    name,
    billing_address:addresses!billing_address_id(name),
    shipping_address:addresses!shipping_address_id(name)
  `)
```

```sql
-- SQLite (flat columns)
SELECT o.name,
       ba.name AS "billing_address.name",
       sa.name AS "shipping_address.name"
FROM orders o
LEFT JOIN addresses ba ON o.billing_address_id = ba.id
LEFT JOIN addresses sa ON o.shipping_address_id = sa.id;

-- SQLite (JSON)
SELECT o.name,
       json_object('name', ba.name) AS billing_address,
       json_object('name', sa.name) AS shipping_address
FROM orders o
LEFT JOIN addresses ba ON o.billing_address_id = ba.id
LEFT JOIN addresses sa ON o.shipping_address_id = sa.id;
```

**Syntax variants:**
- `table!fk_name(cols)` — hint by FK column name
- `table!fk_name!inner(cols)` — hint + inner join

Without hint when ambiguous: error with suggestion message.

---

## Aliasing Embedded Resources

Rename embedded resource in response using `alias:table(cols)`.

```ts
const { data } = await client
  .from('films')
  .select('title, director:directors(first_name, last_name)')

// Result: { title: "...", director: { first_name: "...", last_name: "..." } }
```

```sql
-- SQLite (flat columns)
SELECT f.title, d.first_name AS "director.first_name", d.last_name AS "director.last_name"
FROM films f
LEFT JOIN directors d ON f.director_id = d.id;

-- SQLite (JSON — alias is just the column name)
SELECT f.title,
       json_object('first_name', d.first_name, 'last_name', d.last_name) AS director
FROM films f
LEFT JOIN directors d ON f.director_id = d.id;
```

---

## Spread Operator (`...`)

Flatten embedded fields into parent object instead of nesting.

### To-One Spread

Lifts embedded columns to parent level.

```ts
const { data } = await client
  .from('films')
  .select('title, ...directors(director_name:first_name)')

// Result: { title: "...", director_name: "John" }
// Instead of: { title: "...", directors: { first_name: "John" } }
```

```sql
-- SQLite
SELECT f.title, d.first_name AS director_name
FROM films f
LEFT JOIN directors d ON f.director_id = d.id;
-- No nesting needed — spread columns live at top level
```

### To-Many Spread

Turns embedded fields into correlated arrays.

```ts
const { data } = await client
  .from('directors')
  .select('first_name, ...films(film_titles:title, film_years:year)')

// Result: { first_name: "...", film_titles: ["Title1", "Title2"], film_years: [2020, 2021] }
```

```sql
-- SQLite
SELECT d.first_name,
       json_group_array(f.title) AS film_titles,
       json_group_array(f.year) AS film_years
FROM directors d
LEFT JOIN films f ON f.director_id = d.id
GROUP BY d.id, d.first_name;
-- json_group_array() aggregates into JSON arrays (requires json1 extension)
```

---

## Filtering on Embedded Resources

### Embedded Filter (shapes related data, does NOT filter parent)

```ts
const { data } = await client
  .from('films')
  .select('*, actors(*)')
  .eq('actors.first_name', 'Jehanne')

// All films returned; actors within each film filtered to Jehanne only
```

```sql
-- SQLite (two queries — embedded filter doesn't restrict parent)
-- Query 1: all films
SELECT * FROM films;
-- Query 2: actors for those films, filtered
SELECT a.*, r.film_id
FROM actors a
JOIN roles r ON r.actor_id = a.id
WHERE a.first_name = 'Jehanne' AND r.film_id IN (...film_ids...);

-- SQLite (JSON — single query, all films with filtered actors)
SELECT f.*,
       COALESCE(
         (SELECT json_group_array(json_object('id', a.id, 'first_name', a.first_name, 'last_name', a.last_name))
          FROM roles r
          JOIN actors a ON r.actor_id = a.id
          WHERE r.film_id = f.id AND a.first_name = 'Jehanne'),
         '[]'
       ) AS actors
FROM films f;
-- All films returned; actors subquery filtered independently
```

### Top-Level Filter (filters parent rows via `!inner`)

```ts
const { data } = await client
  .from('films')
  .select('title, actors!inner(*)')
  .eq('actors.first_name', 'Jehanne')

// Only films that have actor Jehanne
```

```sql
-- SQLite (same as !inner join example above)
SELECT f.title, a.*
FROM films f
INNER JOIN roles r ON r.film_id = f.id
INNER JOIN actors a ON r.actor_id = a.id
WHERE a.first_name = 'Jehanne';
```

### Null Filtering (filter by relationship existence)

```ts
// Films WITHOUT any nominations
const { data } = await client
  .from('films')
  .select('title, nominations()')
  .is('nominations', null)

// Films WITH actors
const { data } = await client
  .from('films')
  .select('title, actors(*)')
  .not('actors', 'is', null)
```

```sql
-- SQLite: films WITHOUT nominations
SELECT f.title
FROM films f
LEFT JOIN nominations n ON n.film_id = f.id
WHERE n.id IS NULL;

-- SQLite: films WITH actors
SELECT DISTINCT f.title, a.*
FROM films f
INNER JOIN roles r ON r.film_id = f.id
INNER JOIN actors a ON r.actor_id = a.id;
```

---

## Ordering Embedded Resources

### Order within embedded resource

```ts
const { data } = await client
  .from('films')
  .select('*, actors(*)')
  .order('last_name', { referencedTable: 'actors' })
```

```
GET /films?select=*,actors(*)&actors.order=last_name,first_name
```

```sql
-- SQLite (flat rows, requires post-processing to group)
SELECT f.*, a.*
FROM films f
LEFT JOIN roles r ON r.film_id = f.id
LEFT JOIN actors a ON r.actor_id = a.id
ORDER BY a.last_name, a.first_name;

-- SQLite (JSON — ordered actors embedded directly)
SELECT f.*,
       COALESCE(
         (SELECT json_group_array(json_object('id', a.id, 'first_name', a.first_name, 'last_name', a.last_name))
          FROM roles r
          JOIN actors a ON r.actor_id = a.id
          WHERE r.film_id = f.id
          ORDER BY a.last_name, a.first_name),
         '[]'
       ) AS actors
FROM films f;
```

### Order parent by related column (to-one only)

```
GET /films?select=title,directors(last_name)&order=directors(last_name).desc
```

```sql
-- SQLite
SELECT f.title, d.last_name AS "directors.last_name"
FROM films f
LEFT JOIN directors d ON f.director_id = d.id
ORDER BY d.last_name DESC;
```

---

## Limiting Embedded Resources

```ts
const { data } = await client
  .from('films')
  .select('*, actors(*)')
  .limit(10, { referencedTable: 'actors' })
```

```
GET /films?select=*,actors(*)&actors.limit=10&actors.offset=2
```

```sql
-- SQLite (flat rows — requires window function to limit per-parent)
SELECT f.*, a.*
FROM films f
LEFT JOIN (
  SELECT a2.*, r2.film_id,
         ROW_NUMBER() OVER (PARTITION BY r2.film_id ORDER BY a2.id) AS rn
  FROM actors a2
  JOIN roles r2 ON r2.actor_id = a2.id
) a ON a.film_id = f.id AND a.rn > 2 AND a.rn <= 12;

-- SQLite (JSON — limit/offset in correlated subquery)
SELECT f.*,
       COALESCE(
         (SELECT json_group_array(json_object('id', a.id, 'first_name', a.first_name, 'last_name', a.last_name))
          FROM roles r
          JOIN actors a ON r.actor_id = a.id
          WHERE r.film_id = f.id
          ORDER BY a.id
          LIMIT 10 OFFSET 2),
         '[]'
       ) AS actors
FROM films f;
-- LIMIT/OFFSET inside subquery — simpler than window function approach
```

---

## Embedding After Mutations

Embedding works with INSERT/UPDATE/DELETE when using `select()` (PostgREST `Prefer: return=representation`).

```ts
const { data } = await client
  .from('films')
  .insert({ title: 'New Film', director_id: 1 })
  .select('title, directors(last_name)')

// Returns inserted film with embedded director
```

```sql
-- SQLite (two-step: insert then query with flat columns)
INSERT INTO films (title, director_id) VALUES ('New Film', 1);

SELECT f.title, d.last_name AS "directors.last_name"
FROM films f
LEFT JOIN directors d ON f.director_id = d.id
WHERE f.id = last_insert_rowid();

-- SQLite (JSON — nested object in result)
INSERT INTO films (title, director_id) VALUES ('New Film', 1);

SELECT f.title,
       json_object('last_name', d.last_name) AS directors
FROM films f
LEFT JOIN directors d ON f.director_id = d.id
WHERE f.id = last_insert_rowid();
```

---

## Select Syntax Summary

| Syntax | Meaning |
|--------|---------|
| `table(*)` | All columns from related table |
| `table(col1, col2)` | Specific columns |
| `alias:table(cols)` | Rename in response |
| `table!inner(cols)` | Inner join |
| `table!left(cols)` | Explicit left join (default) |
| `table!fk_name(cols)` | Disambiguate FK |
| `table!fk_name!inner(cols)` | Disambiguate + inner join |
| `...table(cols)` | Spread into parent |
| `table(nested_table(cols))` | Nested embedding |
| `table()` | Empty select (for null filtering) |

---

## SQLite Compatibility

**Status:** ⚠️ Partial — requires manual implementation

| Feature | SQLite (flat rows) | SQLite (JSON) | Difficulty |
|---------|-------------------|---------------|------------|
| Many-to-one | `LEFT JOIN` + post-process | `json_object()` in JOIN | ✅ Simple |
| One-to-many | `LEFT JOIN` + group rows | `json_group_array(json_object())` subquery | ✅ Simple |
| Many-to-many | Double JOIN + group rows | `json_group_array()` with correlated subquery | ⚠️ Moderate |
| `!inner` | `INNER JOIN` | Same | ✅ Simple |
| Nested embedding | Multiple JOINs + deep grouping | Nested `json_object()` in subquery | ⚠️ Moderate |
| Spread | Post-process flat columns | Direct `SELECT` columns from JOIN | ✅ Simple |
| Embedded filters | `WHERE` on joined table | `WHERE` inside correlated subquery | ✅ Simple |
| Embedded ordering | `ORDER BY` on joined columns | `ORDER BY` inside subquery | ✅ Simple |
| Embedded limit | Window functions (`ROW_NUMBER`) | `LIMIT/OFFSET` inside subquery | ✅ Simple |
| Null filtering | `LEFT JOIN ... WHERE rel.id IS NULL` | Same | ✅ Simple |

### Implementation Approaches

Two strategies for translating PostgREST embedding to SQLite:

**Approach 1: Flat rows + post-processing** — JOIN and reshape in application code.
**Approach 2: JSON functions (recommended)** — Use `json_object()`, `json_group_array()`, `COALESCE()` to return nested JSON directly from SQLite. Requires json1 extension (built-in since SQLite 3.38.0 / 2022).

### Implementation Notes

PostgREST auto-detects relationships from FK constraints. In SQLite:

1. **Schema introspection:** `PRAGMA foreign_key_list(table)` to discover FKs
2. **Relationship detection:** Build FK graph at startup
3. **Query translation:** Convert `table(cols)` to appropriate JOIN or correlated subquery
4. **Result shaping:** Use `json_object()` for to-one, `COALESCE(json_group_array(json_object(...)), '[]')` for to-many

```sql
-- Many-to-one: films with director (JSON approach)
SELECT f.title,
       json_object('id', d.id, 'last_name', d.last_name) AS directors
FROM films f
LEFT JOIN directors d ON f.director_id = d.id;

-- One-to-many: directors with films (JSON approach)
SELECT d.last_name,
       COALESCE(
         (SELECT json_group_array(json_object('title', f.title))
          FROM films f WHERE f.director_id = d.id),
         '[]'
       ) AS films
FROM directors d;

-- Inner join equivalent
SELECT f.title, a.first_name
FROM films f
INNER JOIN roles r ON r.film_id = f.id
INNER JOIN actors a ON r.actor_id = a.id
WHERE a.first_name = 'Jehanne';
```

**Key pattern:** `COALESCE(json_group_array(...), '[]')` ensures empty relationships return `'[]'` instead of `NULL`, matching PostgREST behavior.

---

## Source

- Parser: `select-query-parser/parser.ts:82-174` (ParseField, embedding syntax)
- Result resolver: `select-query-parser/result.ts:352-495` (ProcessEmbeddedResource)
- Relationship utils: `select-query-parser/utils.ts:224-495` (ResolveRelationships)
- PostgREST: https://docs.postgrest.org/en/v14/references/api/resource_embedding.html

---

## Related

- [core-operations.md](core-operations.md) — `select()` base syntax
- [transforms.md](transforms.md) — `order()`, `limit()` with `referencedTable`
- [filters-advanced.md](filters-advanced.md) — `not()`, `or()` for embedded filtering
- [sqlite-compatibility-matrix.md](sqlite-compatibility-matrix.md) — full compatibility overview

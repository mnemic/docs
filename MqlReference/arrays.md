# Array queries

> **In this section** — the `[]` ANY-element semantics and the correlated `MATCH` predicate.

## ANY-element semantics

Use `[]` suffix to query array elements. An explicit trailing `ANY` or `ALL` selects the quantifier; omission means ANY. Each predicate uses ANY-element semantics — it matches if **any** element satisfies the condition:

Element paths remain row-resolved at every nesting depth. If an ARRAY OBJECT element, or an Object nested inside it, is polymorphic, `[]`, `IN`, `LENGTH(predicate)`, and `MATCH` decode the leaf with that element's active subtype schema. A same-named sibling field with a different type cannot affect the comparison or literal/parameter admission. Each scalar value—or each individual `IN` member—must be valid for at least one candidate subtype; a value valid for none is a query error. Materialized `IN (FROM …)` members use the same per-member, per-candidate conversion rule.

An element path is a predicate operand, never a single per-row value. A projected column (`SELECT [Items][]`, `RETURNING [Items][].[Sku]`), a `GROUP BY`, `ORDER BY` or window key, an aggregate argument (`COUNT([Tags][])`), a function or operator operand (`UPPER([Tags][])`), a `SET` value and a computed column are all rejected with `FEATURE_NOT_SUPPORTED`, naming the position; select the array field `[Items]` or test its elements in a predicate instead (see [element paths read no single value](queries.md#array-element-paths-read-no-single-value)). The element predicates on this page — comparisons, `IN`, `LIKE`, `BETWEEN`, `IS NULL`, `MATCH` and a counted `LENGTH` — keep their meaning in every predicate and expression position, as do wildcard `REPLACE` element targets and array-element indexes.

A present empty Object (`()`) is distinct from NULL in ANY/ALL, `MATCH`, `IN (NULL)` / `NOT IN (NULL)`, and predicate `LENGTH`. This includes Object fields inside polymorphic elements; an absent subtype field or NULL parent remains NULL. Whole `ARRAY OBJECT ()` elements are present values, so `LENGTH([Items][] != NULL)` counts them. The same rule applies through repeated `[]` boundaries, and empty arrays contribute no matching elements.

`IS NULL` and `IS NOT NULL` are synonyms for `= NULL` and `!= NULL` in native array predicates, `MATCH` bodies, and predicate `LENGTH`. They retain the same element scope and trailing quantifiers: `[Items][] IS NOT NULL ALL` tests every element, and `LENGTH([Items][] IS NOT NULL)` counts present elements.

```sql
-- Any tag equals 'urgent' (scalar array)
FROM [App].[Tasks] WHERE [Tags][] = 'urgent';

-- Any station has Power > 100 (array of objects)
FROM [Facility].[Hubs] WHERE [Stations][].[Power] > 100;

-- Nested array field (scalar array inside an object)
FROM [Test].[Orders] WHERE [Details].[Tags][] = 'tag-guid';

-- Null / not-null checks on array element fields
FROM [Facility].[Hubs] WHERE [Stations][].[Label] = NULL;
FROM [Facility].[Hubs] WHERE [Stations][].[Label] != NULL;

-- Combine array and scalar filters (independent — see MATCH for same-element)
FROM [Facility].[Hubs]
WHERE [Country] = 'US' AND [Stations][].[Power] > 100;

-- Pinned-FK array: `[]` matches against the id-half of each element. The
-- per-element [id][rev:8] tail is laid out at stride elemSize + 8.
FROM [App].[Invoices] WHERE [TariffRefs][] = '<tariff-id>';

-- `[].[Id]` is the explicit synonym for the bare `[]` shorthand above.
FROM [App].[Invoices] WHERE [TariffRefs][].[Id] = '<tariff-id>';

-- `[].[Revision]` reads the per-element 8-byte LE int64 tail. ANY-element
-- semantics — true if any element pins that revision.
FROM [App].[Invoices] WHERE [TariffRefs][].[Revision] = 7;
FROM [App].[Invoices] WHERE [TariffRefs][].[Revision] >= 5;
FROM [App].[Invoices] WHERE [TariffRefs][].[Revision] IN (1, 2, 3);
```

The `[Revision]` and `[Id]` pseudo-fields are only valid on `PINNED REFERENCES` arrays — the per-element revision tail does not exist on plain `ARRAY <type> REFERENCES`. Using `[].[Revision]` on a non-pinned array is rejected at predicate-build time. To correlate id and revision on the *same* element use `MATCH` (next section). Like ordinary schema fields, both pseudo-field names are case-insensitive in ANY and `MATCH` forms (`[iD]` and `[rEvIsIoN]` are equivalent spellings). Query validation assigns `[Id]` the array element's key type (including String and Bytes) and `[Revision]` the `INT64` type in both forms, so invalid literals fail before execution instead of being compared against the packed pinned-reference bytes.

Array indexes can serve negative predicates, ALL, and explicit NULL comparisons through value/coverage candidates with an authoritative residual check; see [indexes](indexes.md).

Consecutive dimensions and arrays in separate Object fields both require explicit markers: `[Matrix][][] = 7` and `[Groups][].[Values][] = 7` match any reached scalar equal to 7. Separate comparisons may match different inner arrays or values. A trailing `ALL` quantifies every explicit array boundary; nested `MATCH` expresses different quantifiers or several conditions on the same element.

### Nested array-element predicates inside `MATCH`

`MATCH` sub-predicates may themselves be array-element predicates on *inner* arrays reached through the outer element. The outer `MATCH` binds an element of `[Outer][]`; inside the body, `[Inner][] OP value` is the existential predicate over that element's inner array. Mirrors the LINQ nested-`.Any()` chain.

```sql
-- "Hubs that have at least one station with at least one 'urgent' tag."
FROM [Facility].[Hubs]
WHERE [Stations][] MATCH ([Tags][] = 'urgent');

-- Combined with sibling per-element predicates on the outer element.
FROM [Facility].[Hubs]
WHERE [Stations][] MATCH ([Connector] = 'CCS' AND [Tags][] STARTSWITH 'prio:');
```

The trailing `ALL` quantifier still applies to whichever scope it sits on: `[Outer][] MATCH ALL ([Inner][] = X)` reads as "every outer element has at least one inner element equal to X", whereas `[Outer][] MATCH ([Inner][] = X ALL)` reads as "some outer element has every inner element equal to X". Stacking both quantifiers (`MATCH ALL ([Inner][] = X ALL)`) gives the universal-over-universal.


A body can combine comparisons, nested `MATCH` / `MATCH ALL`, and predicate `LENGTH` with AND, OR, NOT, and parentheses. Every nested path is relative to its enclosing element. NOT preserves UNKNOWN for ordered/text NULL comparisons. Write `MATCH ANY (...)` or `MATCH ALL (...)`; omission of ANY is accepted. The canonical printer always emits the quantifier. Use a nested MATCH when several conditions must hold on one inner element:

```sql
FROM [App].[Items]
WHERE [Groups][] MATCH (
  [Enabled] = true AND
  [Children][] MATCH ([Score] >= 10 AND [Kind] = 'good')
);

FROM [App].[Items]
WHERE [Groups][] MATCH (
  [Enabled] = true AND
  LENGTH([Children][] MATCH ([Score] >= 10 AND [Kind] = 'good')) >= 2
);
```

`LENGTH([Array][] MATCH (...))` counts matching elements of that named array, not matches inside deeper arrays. Its MATCH wrapper has no outer ALL; inner predicates may use ALL over their own arrays. Empty arrays contribute zero to counts, while MATCH ALL on an empty array remains true.

Array paths resolve through the active subtype's schema. Sibling subtypes may use different element types or array dimensions at the same logical field name. Write each `[]` needed by the shape you intend to traverse; a row with another shape does not contribute matching leaves to that path. Scalar element predicates, correlated `MATCH`, and predicate `LENGTH` keep their field types and layouts bound to the active array owner.

## MATCH — correlated array predicates

`MATCH` requires all sub-predicates to be satisfied by the **same** array element:

```sql
-- Find hubs where the SAME station has Power > 100 AND Connector = 'CCS'
FROM [Facility].[Hubs]
WHERE [Stations][] MATCH ([Power] > 100 AND [Connector] = 'CCS');
```

Paths inside `MATCH` are relative to that element. For example, `MATCH ([Body].[Code] = 7)` reads the element's real `Body` object; it does not drop `Body` to read a sibling `Code`. Nested MATCH array paths follow the same rule.

One `MATCH` body accepts at most 16,382 same-element predicates. The parser and direct AST validation enforce the bound before reading or traversing another predicate. Nested MATCH bodies share the parser nesting limit of 100; direct AST validation enforces the same MATCH bound.

Without `MATCH`, individual `[]` predicates are independent — one element could satisfy `Power > 100` while a different element satisfies `Connector = 'CCS'`.

`MATCH` can be combined with scalar filters:

```sql
FROM [Facility].[Hubs]
WHERE [Country] = 'US' AND [Stations][] MATCH ([Power] > 100 AND [Connector] = 'CCS');
```

### More MATCH patterns

```sql
-- MATCH with NULL comparison
FROM [Facility].[Hubs]
WHERE [Stations][] MATCH ([Label] != NULL AND [Power] > 50);

-- MATCH on a polymorphic ARRAY OBJECT extension field — SUBTYPE IS tests
-- each element's subtype; an unknown name errors instead of matching nothing.
FROM [App].[Plants]
WHERE [Components][] MATCH (SUBTYPE IS [battery] AND [Capacity] > 4000);

-- MATCH with parameters
FROM [Facility].[Hubs]
WHERE [Stations][] MATCH ([Power] >= @minPower AND [Connector] = @connector);

-- MATCH with IN / NOT IN. The value source may be a literal list or a
-- non-correlated/correlated one-column subquery.
FROM [Facility].[Hubs]
WHERE [Stations][] MATCH ([Connector] IN ('CCS', 'NACS'));

-- LIKE keeps its complete pattern contract inside MATCH, including ESCAPE.
FROM [Facility].[Hubs]
WHERE [Stations][] MATCH ([Label] LIKE '50\%' ESCAPE '\');

-- MATCH on a pinned-FK array: `[Id]` and `[Revision]` resolve against the
-- per-element pseudo-fields, correlating both halves on the same element.
-- "Find invoices that pin tariff @t at revision >= 7."
FROM [App].[Invoices]
WHERE [TariffRefs][] MATCH ([Id] = @t AND [Revision] >= 7);
```

## `ALL` — every element must satisfy

The default array shorthand (`[Field][] OP value`) and `MATCH` are **existential** — they match the row when *at least one* element satisfies. The trailing `ALL` keyword flips the quantifier to **universal** — every element must satisfy. This also applies to negative operators: an array `['a', 'b']` matches `[Tags][] NOT IN ('a')` because `'b'` satisfies the predicate, even though another element is in the list. Adding `ALL` requires every element to be outside the list.

```sql
-- Bare scalar form. "Hub where every station is CCS."
FROM [Facility].[Hubs]
WHERE [Stations][].[Connector] = 'CCS' ALL;

-- Same shape on a plain scalar array. "Items whose tags are all 'approved'."
FROM [App].[Items]
WHERE [Tags][] = 'approved' ALL;

-- Compound form via MATCH ALL. "Hubs where every station is high-power CCS."
FROM [Facility].[Hubs]
WHERE [Stations][] MATCH ALL ([Connector] = 'CCS' AND [Power] > 100);
```

Empty arrays match vacuously — `[F][] = X ALL` on an empty `[F]` is `true`, mirroring LINQ's `.All()` convention. Operators applying: `=` / `!=` / `<` / `<=` / `>` / `>=` / `IN` / `LIKE` for the bare form, and the same set inside `MATCH ALL`. The default `ANY` semantic stays the same — only the trailing `ALL` keyword changes behaviour.

Empty-array vacuity also applies when an element predicate can never match, such as an ordered comparison with NULL or membership in an empty subquery. NULL arrays remain non-matches under either quantifier.

Array `LIKE` and `NOT LIKE` use the scalar pattern contract, including `ESCAPE`, for bare or nested element paths, `MATCH`, and predicate `LENGTH`. Quantifiers apply to each element's pattern result; a count includes only matching elements.

The typed-client surface is `Enumerable.All` in a LINQ `Where`: `r.Body.Tags.All(t => t == "approved")` translates to `[Tags][] = 'approved' ALL`.

## `LENGTH(... OP value)` — count matching array elements

`LENGTH` over a bare array field path returns the unconditional element count (next section). `LENGTH` may also wrap an array-element predicate to return the *count of elements matching the predicate*; this is a filter-form intended for `WHERE` clauses and translates the LINQ `.Count(predicate)` / `.Where(predicate).Count()` shape on array fields.

```sql
-- Items whose Tags array contains more than 2 'urgent' entries.
FROM [App].[Items]
WHERE LENGTH([Tags][] = 'urgent') > 2;

-- Hubs with at least one high-power station.
FROM [Facility].[Hubs]
WHERE LENGTH([Stations][].[Power] > 100) > 0;

-- Items where every priority-tagged entry is below the threshold.
FROM [App].[Items]
WHERE LENGTH([Tags][] STARTSWITH 'prio:') = 0;
```

The inner-predicate side accepts the same operator set as `[Field][] OP value` (`=` / `!=` / `<` / `<=` / `>` / `>=` / `IN` / `LIKE` / `STARTSWITH` / `CONTAINS`, including the `IS NULL` / `IS NOT NULL` synonyms). The outer comparison against the count must be a relational operator (`=` / `!=` / `<` / `<=` / `>` / `>=`) against a literal, an `@parameter` or a one-column scalar subquery. The count is an `INT64`, and a literal meets it as it meets an `INT64` field: a number by its value (`> 1.5` holds for a count of 2), text as the number it names (`= '02'` matches a count of 2, `> '1.0'` holds for it), text naming no number as a non-match, `TRUE` and `FALSE` as 1 and 0, while a timestamp or `NOW` is refused (`TYPE_MISMATCH` under an ordering, `INVALID_VALUE` under `=` / `!=`). A caller's `@parameter` must name an integer, as for an `INT64` field (`TYPE_MISMATCH` otherwise). The scalar subquery may be non-correlated or correlated, must return at most one row, and zero rows supplies NULL (so the outer comparison does not match, under any operator). Its value, like an enclosing query's field a nested query's threshold reads, is a row's value: an integer is the threshold, and any other value meets the count as one row compares a number with it — a fractional number as a number (`> 1.5` holds for a count of 2), text as the count's canonical text (`= '02'` matches no count). A GUID, temporal, BYTES, ARRAY or OBJECT value has no order with a count: `<`, `<=`, `>` or `>=` against one is `TYPE_MISMATCH` when the statement is bound, by the value's type, whether a row's array, the value itself or the subquery's row is present or not. Empty arrays count as zero matches under any inner predicate. A NULL array, like for ANY and ALL, is a non-match whatever the comparison and threshold: `LENGTH([Tags][] = 'urgent') = 0` and `LENGTH([Tags][] = 'urgent') < 2` both leave out a row whose Tags is NULL, and `NOT (...)` selects it. Scalar `LENGTH([Tags])` of a NULL array is NULL instead. The scalar shorthand's inner predicate may not carry a trailing `ALL` — the count already encodes the bridge between "some" and "all" via the outer threshold (`LENGTH(p) = N` for "every element matches when there are N" or `LENGTH(p) > 0` for "at least one matches").

## Element predicates in expressions

Element predicates keep their `WHERE` meaning in expression position — `CASE`, projections, computed columns, scalar `WHERE` expressions, and `CHECK` constraints. The existential forms (`=`, `!=`, `<`, `<=`, `>`, `>=`, `IN`, `NOT IN`, `LIKE`, `CONTAINS`, `STARTSWITH`, `IS [NOT] NULL`, `BETWEEN`) and a compared `LENGTH(element predicate)` are evaluated by the same native evaluator as the filter, including coercion and the NULL rules. A `SELECT` item that leads with the element path is such an expression, as its parenthesized and mirrored spellings are: `SELECT [Tags][] = 'urgent' AS [Urgent]`, `SELECT ([Tags][] = 'urgent') AS [Urgent]` and `SELECT 'urgent' = [Tags][] AS [Urgent]` publish the same BOOL column, while `SELECT [Tags][]` alone stays a refused element-path projection:

```sql
FROM [App].[Items]
SELECT [Id],
       CASE WHEN [Tags][] = 'urgent' THEN 'urgent' ELSE 'normal' END AS [Kind],
       CASE WHEN LENGTH([Tags][] STARTSWITH 'prio:') > 1 THEN 1 ELSE 0 END AS [Busy];
```

The value side may be a field of the same row or an expression over fields, on either side of the comparison, in `WHERE`, `HAVING`, aggregate `FILTER`, `CASE`, computed and `CHECK` expressions, a write's `WHERE` and a JOIN's `WHERE` or APPLY `ON`. Each element is compared with that row's value:

```sql
FROM [App].[Orders]
WHERE [Tags][] = [Category]                -- some tag equals the category
   OR [Lines][].[Qty] > [MinQty] + 1       -- some line exceeds the row's bound
   OR LENGTH([Tags][] = [Category]) > 1;   -- the category appears twice
```

A completed quantifier is TRUE or FALSE, never UNKNOWN: a NULL or empty array has no matching element, so `NOT ([Tags][] = 'x')` selects it and `CASE WHEN [Tags][] = 'x' THEN … ELSE …` takes its `ELSE` branch. `BETWEEN` tests its two bounds independently, as in `WHERE`, and `v op [A][]` reads as `[A][]` with the mirrored operator. `LENGTH(element predicate)` must be compared with a count; as a standalone value it is rejected.

A trailing `ALL` / `ANY` and a `MATCH` body are read by the `WHERE` grammar in expression position as well, and keep their `WHERE` quantifier, element scope, and NULL rules: an empty array satisfies `ALL`, and a NULL array matches neither `ANY` nor `ALL`. A quantified comparison may name the element path on either side: `'hold' != [Tags][] ALL` and `0 < [Lines][].[Qty] ALL` mean `[Tags][] != 'hold' ALL` and `[Lines][].[Qty] > 0 ALL`, with the same operand rules, and print in that element-side spelling.

```sql
FROM [App].[Orders]
SELECT [Id],
       CASE WHEN [Lines][] MATCH ALL ([Qty] > 0) THEN 'valid' ELSE 'check' END AS [State],
       CASE WHEN [Tags][] != 'hold' ALL THEN 1 ELSE 0 END AS [Releasable];
```

<a id="quantifier-and-match-operands"></a>

A trailing `ANY` / `ALL` or a `MATCH` body tests the elements against values fixed for the evaluation: literals, parameters and `NOW`, and in `WHERE` also subqueries and an enclosing query's fields, as the compared value, on either side, or as an `IN` list member (`[Tags][] = [o].[Code] ALL`, `[o].[Code] = [Tags][] ALL` and `[Tags][] IN ([o].[Code], 'x') ALL` inside a nested query of `[o]`). A field of the filtered row as its operand (`[Tags][] = [Name] ALL`, `[Name] = [Tags][] ALL`, `[Lines][] MATCH ([Sku] = [Name])`) is rejected with `FEATURE_NOT_SUPPORTED` while the statement is read, with one message in every position, a stored definition's included; in expression position a subquery operand is rejected the same way. Write the comparison without the quantifier to compare each element with the row's value (`[Tags][] = [Name]`), or count matching elements with `LENGTH([Tags][] = [Name])`. An arithmetic operand of a quantified predicate is a syntax error.

## `LENGTH` on array fields

`LENGTH([ArrayField])` returns the element count, including in `CHECK` predicates and computed projections:

```sql
-- "must have at least one tag"
CREATE COLLECTION [App].[Items] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY,
  [Tags] ARRAY STRING
)
WITH CHECK [tags_non_empty] (LENGTH([Tags]) > 0);

-- Element count surfaces in projections too
FROM [App].[Items] SELECT [Id], LENGTH([Tags]) AS [TagCount];
```

`LENGTH` on STRING returns its UTF-16 code-unit count, including trailing spaces, matching .NET `string.Length`. Characters outside the Basic Multilingual Plane count as two code units. On ARRAY it returns the element count; on BYTES it returns the raw byte count.

`OCTET_LENGTH` accepts STRING and BYTES. Strings use their strict UTF-8 encoded byte count, without allocating the encoded buffer; binary values use their raw byte count. Both functions return INT64 and propagate NULL. For example, `LENGTH('å😀')` is 3 and `OCTET_LENGTH('å😀')` is 6. Use `WITH CHECK (OCTET_LENGTH([Data]) <= 1024)` to bound a binary payload. Casting BYTES through `TO_STRING` counts Base64 characters instead of bytes.

## Editing arrays

Use `ALTER [name] (...)` inside an `UPDATE SET` or `ON CONFLICT DO UPDATE SET` block. The same operations work in temporal and ordinary collections.

```mql
UPDATE [App].[Items] SET (
    ALTER [Tags] (APPEND ['last'], INSERT AT 0 ['first'], DELETE AT 1)
) WHERE [Id] = 1
```

Operations run in the written order. Positions are zero-based and refer to the array produced by preceding operations. INSERT accepts positions from zero through length; DELETE, REPLACE, UPDATE, and nested ALTER require an existing position. Negative, fractional, NULL, and out-of-range positions fail the statement. NULL arrays and NULL object ancestors fail rather than becoming empty arrays. To initialize an array, assign `[]` with an ordinary SET before editing it.

| Operation | Behavior |
|---|---|
| `APPEND [value, ...]` | Add values at the end. |
| `INSERT AT index [value, ...]` | Add values before a position, or at the end. |
| `DELETE AT index` | Remove one element. |
| `DELETE AS [e] WHERE expression` | Remove every matching element. |
| `REPLACE AT index WITH value` | Replace one complete element. |
| `UPDATE AT index [AS [e]] SET (...)` | Partially update one object element. |
| `UPDATE AS [e] WHERE expression SET (...)` | Partially update every matching object. |
| `ALTER AT index [AS [e]] (...)` | Edit one inner array. |
| `ALTER AS [e] WHERE expression (...)` | Edit matching inner arrays. |

Values use existing array-element literal syntax, including parameters, objects, inner arrays, and PIN references. Predicate and index expressions are scalar MQL; subqueries and window expressions are unavailable inside array edits. A NULL predicate does not match; a non-Boolean predicate fails. Element NULLs follow the array schema's existing restrictions. Empty append/insert lists are permitted, although the insertion position must still be valid. The objects an edit adds and the SET list of an element update are admitted as a SET list's values are, before any row is read and alike by static `EXPLAIN` and a trigger body's validation: the objects are built with their [declared defaults](dml.md#defaults), and a NOT NULL field left out without a default, set to NULL or given `DEFAULT` without one is refused with `NOT_NULL_VIOLATION`. A path into a polymorphic object, or into a polymorphic element's subtype fields, takes the declaration of the subtype the element's object holds.

An alias identifies the current element. Outer aliases remain visible in nested edits. Ordinary row fields refer to the original containing document; EXCLUDED refers to the prepared insert candidate. Assignments inside one object update read the same original element, while the next array operation sees their result.

```mql
UPDATE [App].[Items] SET (
    ALTER [Details].[Tags] (APPEND ['new']),
    ALTER [Lines] (
        UPDATE AS [line] WHERE [line].[Quantity] > 0 SET (
            [Quantity]: [line].[Quantity] + 1,
            ALTER [Labels] (DELETE AS [label] WHERE [label] = 'old')
        )
    ),
    ALTER [Matrix] (ALTER AT 0 (APPEND [3]))
) WHERE [Id] = 1;

INSERT INTO [App].[Items] ([Id]: 1, [Tags]: [])
ON CONFLICT DO UPDATE SET (ALTER [Tags] (APPEND ['seen']));
```

Dotted targets traverse object fields and preserve siblings. Object literals still replace their whole target; put nested array edits on explicit paths or in element UPDATE SET. Polymorphic object updates preserve the existing subtype; use REPLACE AT to change it. Computed element fields can be read but cannot be assigned.

The final document goes through normal constraints, indexes, triggers, RETURNING, and transaction handling. Temporal history records the resulting row version, not intermediate operations. Errors roll back the statement. Upsert conflict edits only run when a row conflicts; a miss inserts the proposed document. Array ALTER is unavailable in INSERT candidates, REPLACE documents, defaults, and projections. Editing state has a descriptor bound as well as the document-byte bound, so very large arrays of tiny elements can be rejected even if their encoded payload fits.

ALTER entries can be mixed with ordinary assignments in the same SET block, such as `SET ([Status]: 'active', ALTER [Tags] (APPEND ['new']))`. Each target occurs once; overlapping mutation paths remain invalid. A bracketed `[ALTER]: value` is an ordinary assignment to a field named `ALTER`. Top-level `ALTER COLLECTION` continues to modify schemas; array ALTER is scoped to a SET entry or an inner array operation.

`ALTER [name] (...)` and nested `ALTER ... (...)` are the only accepted array mutation spellings. `ARRAY_EDIT(...)` assignments and `EDIT ... USING` operations are rejected, including inside stored trigger bodies. The printer and typed client emit the same ALTER syntax.

## See also

- [Basic queries](queries.md)
- [Polymorphism](polymorphism.md) — discriminator fields and the `SUBTYPE` keyword
- [Schema definition (DDL)](ddl.md) — declaring `ARRAY` and `ARRAY OBJECT` fields

## Usage pitfalls


<a id="pitfall-array-element-gin-index-serves-any-membership--equality-in-range"></a>

### Array-element (GIN) index serves ANY membership — equality, IN, range

`CREATE INDEX ON [C] ([Tags][])` indexes a scalar-element array; `CREATE INDEX ON [C] ([Items][].[Sku])` indexes a scalar sub-field within each `ARRAY OBJECT` element. Both accelerate the ANY-membership shapes (all show `EXPLAIN Strategy = ArrayMembership`): `WHERE [Tags][] = v` (the default ANY quantifier), `WHERE [Tags][] IN (a, b)` (union of per-value seeks), and a one-sided `WHERE [Vals][] >= lo` (range scan of the ordered element keys) — and the same on the nested form (`WHERE [Items][].[Sku] = v`); IN/range return each matching doc once. Array membership precedes scalar compound/equality candidates both at runtime and in EXPLAIN when a query contains both shapes. Compound array indexes also keep full row predicates: their per-path postings do not prove that two values came from one element. Intersections combine independent predicates, including two predicates on one array path; repeated matching keys cannot substitute for a different predicate. Compound ALL and negative plans union necessary coverage rather than subtracting matching document sets. Runtime membership retains the same typed set and NULL policy as heap evaluation. UNIQUE compares independent component value domains across rows, not element positions; an empty or null-only component owns no value tuple. Two predicates on one array path are independent existentials: `[Vals][] > 5 AND [Vals][] < 10` may use different elements. The planner can index one compatible side and recheck the other; treating them as one interval would be incorrect. `ALL`, `!=`, `NOT IN`, and explicit NULL comparisons also use value/coverage candidates with an authoritative residual. A null scalar leaf can satisfy a negative non-null comparison; a null array container supplies no elements. Empty arrays remain vacuously true for ALL. Null-bearing `IN` / `NOT IN` lists retain index candidates and full residuals. A NULL list member matches a NULL leaf in IN and excludes it in NOT IN. Present leaves compare against the present list members. Uncorrelated runtime IN-subquery lists can also use the array index: execution probes its ordered distinct values through the spill-backed typed set, then rechecks candidate rows. Static EXPLAIN describes this access without running the subquery; an empty runtime IN set may instead short-circuit. Index definitions and ANY predicates can cross multiple explicit array boundaries. Use nested `MATCH` bodies when several predicates must bind the same inner element. A scalar `CREATE INDEX ON [C] ([Tags])` (no `[]`) on an array field is rejected: add the `[]` marker for an array-element index. A partial `WHERE` filter is supported (the planner uses it only when the query implies the filter, like scalar partial indexes), and FK arrays are indexable (`WHERE [TagIds][] = id`; pinned FK arrays key on the id, revision-agnostic). Array `INCLUDE` stores base-row scalar values per posting; it cannot reconstruct an array from one element key. Polymorphic elements and nested subtype-extension leaves are indexable when all declaring subtypes supply stored scalar types; each declared type owns a separate ordered domain, and absent subtype fields contribute no scalar-value posting. Arrays inside Object containers also work, including polymorphic extensions: `CREATE INDEX ON [C] ([Profile].[Tags][])` and `CREATE INDEX ON [C] ([Profile].[Items][].[Sku])`. The active container owns the array's element layout; null or absent containers add no key. Collection subtype extensions may own these arrays too. UNIQUE ownership is per declared type and canonical value, so INT32 7 and INT64 7 do not conflict with each other. A scoped rename preserves an index at the old path while sibling declarations remain; otherwise the index follows the rename. Dropping its last declaration retires it. Schema abort restores the original index and rows. Stored foreign-key leaves inside array objects are supported, including `[Items][].[Sub].[Ref]`. Pins index the identifier; `.Id` and `.Revision` can have separate indexes. Pinned String/Bytes values use the same contract. Reference validation and removal traverse object segments between arrays and active subtype envelopes. Scalar SetNull preserves sibling fields and array boundaries; packed-array Cascade removes matching identifiers, and pinned packed-array SetNull removes only the cancelled key/revision pair. The default ANY quantifier also applies to `!=` and `NOT IN`: `['a', 'b']` matches `[] NOT IN ('a')` because `'b'` is outside the list. This is not the complement of documents matching `[] IN ('a')`.

`CREATE UNIQUE INDEX ON [C] ([Tags][])` enforces uniqueness **across documents**; repeated canonical values inside one document are allowed. A null/missing/empty array contributes coverage but no uniqueness claim; an empty string or byte sequence does claim a value. Partial filters limit the documents subject to the constraint. Pinned references claim an id across revisions. Any conflicting element rejects the complete write with `UNIQUE_VIOLATION`. Online builds can reject conflicting writes before becoming query-visible. On a temporal collection an element belongs to one document at every instant of valid time: a document's versions share their elements, and another document conflicts only over an overlapping interval ([UNIQUE on a temporal collection](indexes.md#unique-on-a-temporal-collection)). An array index cannot identify a single `ON CONFLICT` target; ordinary primary-key and unique-index upserts still obey its constraint.

`ARRAY BYTES` and a `BYTES` leaf inside an `ARRAY OBJECT` are valid index domains. The literal `''` is an empty byte sequence, not `NULL`; it contributes a typed value posting, while a nullable nested leaf set to `NULL` contributes a null-leaf coverage posting without claiming a unique scalar value. The nested leaf must be stored: a virtual computed element field cannot be an array-index key because its value exists only after query-time materialization.

<a id="pitfall-nested-array-element-predicates-inside-match-are-correlated-outer--existential-inner"></a>

### Nested array-element predicates inside MATCH are correlated outer / existential inner

`[Outer][] MATCH ([Inner][] OP value)` reads as "some outer element has at least one inner element matching the predicate" — the outer MATCH is correlated (binds one element of `[Outer][]`), the inner `[]` is existential over that element's inner array. Mix with `ALL` quantifiers carefully:

- `[Outer][] MATCH ALL ([Inner][] = X)` — every outer element has some inner element equal to X.
- `[Outer][] MATCH ([Inner][] = X ALL)` — some outer element has every inner element equal to X.
- Combining both keywords gives the universal-over-universal form.

<a id="pitfall-pinned-array-id--revision-pseudo-fields-are-pinned-only"></a>

### Pinned-array `[Id]` / `[Revision]` pseudo-fields are pinned-only

`[Field][].[Id]` and `[Field][].[Revision]` resolve to the per- element id-half and 8-byte LE int64 revision tail **only** on `PINNED REFERENCES` arrays. Pinned-FK array elements are scalar (INT8 / INT16 / INT32 / INT64 / GUID / STRING / BYTES) so they have no real `Id` / `Revision` nested fields to shadow. On a plain `ARRAY <type> REFERENCES`, the revision tail does not exist — using `[].[Revision]` is rejected at predicate-build time. Using `[].[Id]` on a non-pinned array errors through the standard "no such field" path. To correlate id and revision on the *same* element, use `MATCH` rather than independent ANY predicates: `[TariffRefs][] MATCH ([Id] = @t AND [Revision] = 7)`.

<a id="pitfall-nested-wildcard-modifiers-retain-their-containing-object-and-array"></a>

### Nested wildcard modifiers retain their containing Object and Array

`SELECT * EXCLUDE ([Address].[Zip])` removes Zip while retaining Address and its other children. `REPLACE (UPPER([Address].[City]) AS [Address].[City])` updates a child, and `RENAME ([Address].[City] AS [Town])` changes its name inside Address. Null parents stay null; replacing a child does not create its missing parent. All expressions read the original input, and rename sources use original paths even when their parent is also renamed. A replaced parent cannot also have child modifiers. Renaming or replacing an excluded child or one below an excluded parent is an error.

An Array needs explicit `[]` traversal: use `[Items][].[Price]`, not `[Items].[Price]`. Each replacement reads the current original element and may also read original outer fields, so replacing or excluding an outer field does not change the input seen by its element expressions. Repeated Array dimensions need repeated `[]`. Empty Arrays stay empty, null parents stay null, and polymorphic elements retain their subtype payloads. A window in REPLACE runs over source rows and broadcasts the row's value to an Array-child target.

For scalar Arrays, target the element directly: `REPLACE ([Values][] * [Factor] AS [Values][])`. Consecutive Arrays use `[Matrix][][]`. Replacements infer the output element type and preserve count and order; NULL or unrepresentable element results fail instead of becoming zero or empty values. EXCLUDE and RENAME need a named field, so they cannot target anonymous `[Values][]` elements. Replacing pinned-reference elements returns ordinary values, not references to their original revisions. Qualify an outer element reference inside a subquery when the inner collection declares the same field name.

Unknown EXCLUDE/REPLACE/RENAME targets and an EXCLUDE list that removes every field are rejected for CTEs and system collections exactly as they are for stored collections; derived sources do not relax these schema checks.

<a id="pitfall-stored-row-corruption-never-degrades-to-null-or-a-non-match"></a>

### Stored row corruption never degrades to NULL or a non-match

Every row-consuming query validates the complete binary document before WHERE, ORDER BY, aggregation, projection, or result delivery. Invalid offset/String/ Array/Object/polymorphic framing, invalid strict UTF-8 or scalar bit patterns, NULL in a non-nullable field, and disagreement between the body PK and slot PK raise `DOCUMENT_PAYLOAD_CORRUPT` and quarantine the collection. Do not expect a damaged row to be skipped or sorted under an empty key. An unfiltered scalar `COUNT(*)` may use the catalog's transactional row counter; it counts live rows without reading bodies and is not an integrity scan. The lower expression-field decoder still requires exact widths for fixed-width scalars, canonical `0`/`1` Boolean bytes, exact String framing, and strict UTF-8 for synthetic/transformed buffers; it never ignores trailing scalar bytes, turns malformed bytes into U+FFFD replacement text, or constructs Decimal/date/time values from impossible bit patterns. Durable rows fail at the validator before that defensive decoder is reached. The transactional row counter is recovered from the same WAL commit group as the row mutation; startup does not rebuild it by scanning slots.

The direct binary engine write API rejects the same shapes before persistence as User-class `DOCUMENT_PAYLOAD_INVALID`. Normal MQL serialization produces canonical bodies, so seeing this code through an MQL-backed integration points to a caller/plugin that supplied raw binary bytes.

Stored rows are decoded only with their exact cataloged layout generation. `DOCUMENT_LAYOUT_GENERATION_UNKNOWN`, `DOCUMENT_LAYOUT_TRANSLATION_FAILED`, `CATALOG_LAYOUT_HISTORY_MISSING`, and `CATALOG_LAYOUT_HISTORY_CORRUPT` mean the engine cannot prove the historical-schema-to-active-schema interpretation and therefore returns no row. `SLOT_PAGE_TYPE_INVALID` and `SLOT_INDEX_OUT_OF_RANGE` similarly reject an impossible physical row locator. Do not retry these structural-corruption errors as empty or non-matching query results; inspect health state and restore from a known-good source.

<a id="pitfall-array-statistics-count-postings-separately-from-source-rows"></a>

### Array statistics count postings separately from source rows

`IsArrayComponent = TRUE` marks row-distinct typed value and coverage postings. Repeated equal values in a row count once. Each MCV frequency cannot exceed `CollectionDocCountAtAnalyze`, but sums across distinct values can. Empty sources have no postings. Subtype omissions contribute coverage after the partial filter is evaluated against the complete subtype row.

ANALYZE can make one selective component cheaper than a larger intersection. EXPLAIN and execution use the same costed choice, retaining full row checks. Component statistics are not joint tuple statistics or correlation proofs.

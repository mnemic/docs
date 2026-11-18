# Document literals

> **In this section** — the `(...)` / `[...]` value syntax used by INSERT / UPDATE / REPLACE document literals.

MQL uses its own document literal syntax (not JSON):

| Value Type | Syntax | Example |
|------------|--------|---------|
| String | Single quotes | `'text'`, `'O''Brien'` |
| Number | Bare literal | `123`, `-45`, `3.14` |
| Decimal | Bare number or numeric string; the scale is kept | `12.50`, `'12.50'` |
| Boolean | Bare keyword, or the string `'true'`, `'false'`, `'1'` or `'0'` in any case; the numbers `1` and `0` are rejected | `TRUE`, `FALSE`, `'true'` |
| Guid | String: hyphenated, 32 bare hex digits, or wrapped in braces or parentheses | `'550e8400-e29b-41d4-a716-446655440000'` |
| Bytes | Base64 string only; hex text is never read as hex (`'0x0102'` is Base64-decoded or rejected). ASCII whitespace is ignored, and the bits of the last character that no byte holds must be zero (`'AQ=='`, never `'AR=='`) | `'AQID'` (bytes 01 02 03) |
| Vector | `VECTOR[...]` of numbers; a plain `[...]` is an array, which a `VECTOR(n)` field rejects with `TYPE_MISMATCH` | `VECTOR[0.1, 0.2, 0.3]` |
| Enum value | The member's underlying value, as a number or its string, never its label | `2`, `'2'` |
| Null | Bare keyword | `NULL` |
| Field default | Bare keyword, a field's whole value only: the field's declared default, or NULL when it declares none ([DEFAULT](dml.md#default)) | `[Qty]: DEFAULT` |
| DateTime | ISO 8601 timestamp string or `NOW`; no offset means UTC | `'2024-01-15T10:30:00Z'`, `'2024-01-15 10:30:00+02:00'`, `NOW` |
| DateOnly | `'yyyy-MM-dd'`, an ISO 8601 timestamp (its written date), or `NOW` | `'2025-01-02'` |
| TimeOnly | `'HH:mm'`, `'HH:mm:ss'`, `'HH:mm:ss.fffffff'` (one to seven fraction digits), an ISO 8601 timestamp (its written time), or `NOW` | `'10:30'`, `'10:30:15.25'` |
| Parameter | `@name` | `@userId` |
| Nested object | Parentheses | `([City]: 'Boston', [Zip]: '02101')` |
| Array | Square brackets | `['red', 'blue', 'green']` |
| Array of objects | Square brackets with nested parens | `[([Type]: 'A'), ([Type]: 'B')]` |
| Polymorphic sub-doc | `SUBTYPE` keyword | `SUBTYPE [Solar] ([Power]: 100)` |
| Pinned FK (scalar) | `PIN(id, revision)` | `PIN('a1b2-...', 7)` |
| Pinned FK (array) | `[PIN(id, rev), …]` | `[PIN('t1', 3), PIN('t2', 7)]` |

Field names within document literals use bracket syntax: `[FieldName]: value`. Each individual document literal accepts at most 1,024 field assignments, matching the maximum number of fields in a stored schema. A literal names each field once, compared without regard to case, at every nesting level and in a `SUBTYPE` literal; a repeated field is `DUPLICATE_NAME`, as is an UPDATE or `ON CONFLICT DO UPDATE` SET list that assigns one field twice. The parser rejects an additional assignment before constructing it; programmatically supplied ASTs enforce the same limit before printing or execution.

Bare numeric literals follow the [exact Decimal literal rules](syntax.md#numbers) before conversion to the field type. A value that requires Decimal rounding or underflow is rejected during parsing, including inside nested objects and arrays.

<a id="numeric-text"></a>

Numeric text — a string read as a number, whether it is written to a numeric field, element or key, compared with a number, or converted by `CAST` / `TRY_CAST` — has one grammar: optional surrounding whitespace, an optional leading `+` or `-`, decimal digits with an optional decimal point, and an optional exponent (`e` or `E`, an optional sign, digits), with `NaN` and `Infinity` also read by a `FLOAT32` or `FLOAT64` destination. A group separator (`'1,000'`), a currency symbol (`'¤8'`), parentheses for a negative (`'(5)'`), a trailing sign (`'5-'`) and hexadecimal (`'0x10'`) name no number: written to a numeric field or key they are `INVALID_VALUE`, compared with a number they match nothing, and `TRY_CAST` reads them as NULL.

Numeric strings assigned to `FLOAT32` or `FLOAT64` parse invariantly in the destination type, including array elements and string parameters. Representable small values such as `'1e-40'` for `FLOAT32` and `'1e-300'` for `FLOAT64` retain their floating-point value rather than rounding through Decimal to zero.

A number assigned to a `STRING` field or array element — a literal, a parameter or an assignment expression — stores its canonical text, the text `CAST(... AS STRING)` renders and a comparison with a STRING reads ([comparison value notes](queries.md#comparison-value-notes)): a `FLOAT64` negative zero stores `'0'`, a DECIMAL keeps its scale (`1.0`) and a `FLOAT32` reads as the `FLOAT64` holding its exact value. NaN and the infinities have no text and are `INVALID_VALUE`. Every other scalar stores its cast text too: a BOOL `'true'` or `'false'`, a DATETIME, DATEONLY or TIMEONLY its ISO 8601 round-trip text (`'2024-06-15T10:00:00.0000000+02:00'`, `'2024-06-15'`, `'10:30:00.0000000'`), a GUID its hyphenated text and BYTES their Base64 text.

A `FLOAT64` assigned to a `DECIMAL` field or array element — a parameter or an assignment expression — stores the DECIMAL its canonical text names, the same number `CAST(... AS DECIMAL)` gives ([computed projections](queries.md#computed-projections)): a `double` parameter `0.1` stores `0.1`. A `FLOAT32` stores the DECIMAL of the `FLOAT64` holding its exact value, so a `float` parameter `0.1` stores `0.10000000149011612`. NaN, the infinities and a magnitude from 2^96 up have no DECIMAL and are `INVALID_VALUE`.

Integer fields and primary keys reject fractional numeric strings, including primitive-array elements and UPSERT conflict probes. Text such as `'1e-40'` or `'1.00000000000000000000000000001'` cannot become an integer through Decimal rounding. Exact integral text such as `'100e-2'` or `'1.0'` is accepted when it fits the declared integer width.

## Arrays

```sql
-- Scalar array
[Tags]: ['red', 'blue', 'green']

-- Array of objects
[Connectors]: [
  ([Type]: 'CCS', [Power]: 150),
  ([Type]: 'CHAdeMO', [Power]: 50)
]

-- Array with polymorphic elements
[Components]: [
  SUBTYPE [battery] ([Capacity]: 5000),
  SUBTYPE [motor] ([Torque]: 300)
]

-- Array of pinned FK references — every element is PIN(id, revision)
-- and the field's parent must be declared as `ARRAY <type> PINNED REFERENCES …`
[TariffRefs]: [PIN('t1-...-...', 3), PIN('t2-...-...', 7)]

-- PIN(...) literals are allowed in extension positions on polymorphic
-- collections and field-level polymorphic sub-documents — write the
-- pinned value alongside the rest of the subtype's extension fields:
INSERT INTO [App].[Invoices] SUBTYPE [Paid] (
  [Id]: '...', [Kind]: 'Paid',
  [TariffRef]: PIN('a1b2-...-...', 1), [PaidAmount]: 500
)
```

## Nested objects

```sql
[Address]: ([Street]: '123 Main', [City]: 'Stockholm')
[Details]: ([Priority]: 1, [Tags]: ['urgent', 'review'])
```

## Polymorphic sub-documents

When a field is declared as polymorphic, use `SUBTYPE` within the document literal:

```sql
-- Single polymorphic field
[Component]: SUBTYPE [Solar] ([Power]: 100, [Efficiency]: 0.22)

-- Polymorphic elements in an array
[Components]: [
  SUBTYPE [battery] ([Capacity]: 5000),
  SUBTYPE [motor] ([Torque]: 300)
]
```

## Temporal text

A temporal string — in a document literal, a parameter, a comparison, `CAST`, `TO_TIMESTAMP`, or a field default — is read only in its ISO 8601 forms, the same on every host and in every culture:

- a date is `yyyy-MM-dd`: a four-digit year and a two-digit month and day;
- a time of day is `HH:mm`, `HH:mm:ss`, or `HH:mm:ss` followed by `.` and one to seven fraction digits, on a 24-hour clock;
- a timestamp is a date alone, which names its midnight, or a date and a time of day joined by `T` or one space, optionally followed by `Z` or a `±HH:mm` offset of two-digit hours and minutes. A timestamp without an offset is UTC, never the server's local time. Other offset spellings (`+0200`, `-0530`, `+2:00`, `+02`) name no timestamp.

Culture-shaped text such as `'01/02/2025'`, `'2024-06-15T10:00:00+0200'`, `'2025-1-2'`, `'June 15, 2024'`, `'Sat, 15 Jun 2024 10:00:00 GMT'`, or `'10:00 PM'` names no value: such a literal is `INVALID_VALUE`, and a parameter is refused as [parameters](parameters.md) describes. A DATEONLY reads a timestamp's date and a TIMEONLY its time of day as written, never shifted to UTC or the server's zone: `'2025-01-02T23:30:00-05:00'` stores 2025-01-02 and 23:30. A time of day alone is no DATEONLY or DATETIME, and a date alone is no TIMEONLY.

## Assignment expressions

UPDATE SET accepts a field copy such as `[Target]: [Source]` or `[Target]: [Address].[City]`, and parenthesized expressions such as `[Total]: ([Price] * [Qty])`. Expressions read the original row.

Square brackets whose contents start an array value keep their literal meaning: `[1]`, `[TRUE]`, `[NULL]`, `[DEFAULT]`, `[@value]`, and `['text']` are arrays, and an array holding `DEFAULT` is refused, since DEFAULT is a field's whole value and never an element ([DEFAULT](dml.md#default)). To copy a field with such a name, use parentheses: `[Target]: ([TRUE])`. The canonical printer uses parentheses for standalone field copies. `()` and `([Field]: value)` remain nested documents; other parenthesized assignment values are expressions. INSERT still rejects expressions that require an existing row.

UPDATE SET and ON CONFLICT DO UPDATE SET additionally accept `ALTER [Tags] (APPEND ['new'])` alongside value assignments. An ALTER entry mutates an existing array; it is not a value literal. See [Editing arrays](arrays.md#editing-arrays) for nested operations and restrictions on INSERT, REPLACE, and defaults.

## See also

- [DML](dml.md) — INSERT / UPDATE / REPLACE statements
- [Polymorphism](polymorphism.md)
- [Schema definition (DDL)](ddl.md) — declaring `OBJECT` and `ARRAY` fields

## Detailed syntax and execution


<a id="syntax-2-document-literals"></a>

### Document literals

Used by `INSERT`, `UPDATE SET`, `REPLACE SET`. Form: `(Field: value, ...)`. Each individual literal accepts at most 1,024 field assignments, matching the stored-schema field limit. The same bound applies to nested objects and programmatically constructed ASTs.

```sql
([Id]: 1, [Name]: 'A', [Tags]: ['x', 'y'], [Address]: ([City]: 'Boston'))
```

- Bracketed field names; `:` separator; comma between assignments.
- Nested objects: nested `(...)` literal.
- Arrays: `[elem, elem, …]`. Element-of-object: `[(field: v), …]`.
- Polymorphic field: `SUBTYPE [discriminator] (...)` prefix on the literal.
- Pinned FK (scalar): `PIN(id, revision)`.
- Pinned FK (array): `[PIN(id, revision), PIN(id, revision), …]` — every element of an `ARRAY <type> PINNED REFERENCES …` field must use `PIN(...)`.
- Pinned FK in a polymorphic-extension subtype: declare and write the same way, just inside the subtype's field list — `WITH SUBTYPE [X] ([Ref] GUID PINNED REFERENCES [...]…)` and `INSERT … SUBTYPE [X] (..., [Ref]: PIN(id, rev))`.

<a id="syntax-assignment-expressions"></a>

### Assignment expressions

UPDATE SET accepts a field copy such as `[Target]: [Source]` or `[Target]: [Address].[City]`, and parenthesized expressions such as `[Total]: ([Price] * [Qty])`. Expressions read the original row.

Square brackets whose contents start an array value keep their literal meaning: `[1]`, `[TRUE]`, `[NULL]`, `[@value]`, and `['text']` are arrays. To copy a field with such a name, use parentheses: `[Target]: ([TRUE])`. The canonical printer uses parentheses for standalone field copies. `()` and `([Field]: value)` remain nested documents; other parenthesized assignment values are expressions. INSERT still rejects expressions that require an existing row.

See [`document-literals.md`](document-literals.md).

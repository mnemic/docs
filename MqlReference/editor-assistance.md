# Editor assistance

> **In this section** — `COMPLETE` lists what fits at a position of MQL text and `VALIDATE` reports what in it does not parse, as the connected server reads it, plugins' statements included.

```sql
COMPLETE @text AT @position;
VALIDATE @text;
```

An editor asks the server it is connected to instead of knowing MQL itself, so its completion and diagnostics follow the server's version and the plugins it loads. Both statements only read the text: nothing in it runs, they change nothing, and they leave an open transaction or migration as it was.

The text is always a [parameter](parameters.md) holding a `STRING`. Statement text reaches audit records and logs, so text written into the statement, which could hold a password being typed, is a `SYNTAX_ERROR` that does not quote it. Where the MQL the literal holds carries a credential, the refusal's message is withheld as a credential statement's is, and the server log of the refusal masks that credential as it masks one in a statement ([user administration](auth-admin.md)). A parameter that was not provided is `UNKNOWN_PARAMETER`; one that is not a `STRING` is `INVALID_VALUE`.

Positions and ranges count UTF-16 code units of the whole text from 0, as editors and the Language Server Protocol do, so a character outside the Basic Multilingual Plane counts twice. The text may hold a whole script: the statement a position belongs to is found as clients split scripts, at semicolons outside strings, bracketed names, parentheses, comments and a trigger's `BEGIN … END` body ([syntax](syntax.md#multi-statement-execution)).

## COMPLETE

`COMPLETE @text AT @position` returns one row per completion at `@position`, best first. `AT` takes an integer parameter or literal from 0 to the text's length that does not fall between the two halves of a surrogate pair; anything else is `INVALID_VALUE`.

| Column | Type | Meaning |
|---|---|---|
| `Label` | `STRING` | What a list shows and typing is matched against: a keyword, a function, or a name as MQL writes it, such as `[Shop].[Orders]` after `FROM` or `[Orders]` after `FROM [Shop].`. |
| `Kind` | `STRING` | `Keyword`, `Function`, `Collection`, `View`, `Cte`, `Field`, `Type`, `Enum`, `SubType`, `Namespace`, `Trigger`, `Diagram`, `Database` or `Setting`. A client treats a kind it does not know as plain text. |
| `Detail` | `STRING NULL` | A short note, such as a field's type, `Materialized view`, or a function's kind and argument count. |
| `Insert` | `STRING NULL` | The text the completion inserts where it differs from the label, such as `UPPER(` for a function. |
| `Boost` | `INT32` | The rank of the completion's kind: fields first, then the database's other names, then system collections, functions and settings, then keywords. Rows come ordered by it and, within a kind, by how early the grammar tries the word, so after `EVERY 1` in a [series](queries.md#series) the time units come first. |
| `From` | `INT32` | Where the text a completion replaces starts: the start of the word or bracketed name at the position. The same in every row. |
| `To` | `INT32` | Where it ends: the end of that word or name, past the rest of it after the position and a closing bracket an editor inserted. The same in every row. |

The completions are what the server's parser would accept at the position: the keywords and functions the grammar expects there, a plugin's words where its statement or clause fits, and the names the database holds where the grammar expects a name of that kind. Names come from the database the connection is bound to as the caller reads it, the committed catalog: its collections (those [`[Mnemic].[Collections]`](system-collections.md#virtual-collections) lists, never a materialized view's retired generation), views and materialized views, the [system collections](system-collections.md), named types and enums, triggers, [diagrams](diagrams.md), the common table expressions the statement declares, and the [session parameters](maintenance.md#session-parameters--set--show) `SET` and `SHOW` name. A field's name is offered from the relation the grammar reads it in: after an alias (`[e].`), a relation's or a joined relation's, a view's, a series' (`[Start]`, `[End]`), a named type's (`[Ship].`) or an array element's (`[Lines][].`), with the fields each subtype of a polymorphic collection adds, the keys of a document literal being written, and a trigger's `OLD` and `NEW` fields. Database names are those the caller may [list](databases.md).

Nothing is offered inside a string, a comment, a number or a parameter, or where the text before the position does not parse. A response holds at most 2,000 rows.

```sql
-- @text = 'FROM [HR].[Employees] [e] WHERE [e].', @position = 36
COMPLETE @text AT @position;
-- [Id] Field INT32 …, [Name] Field STRING …, [Salary] Field DECIMAL …
```

## VALIDATE

`VALIDATE @text` returns one row per statement of the text that does not parse, in script order, and no rows for text that parses. The parser stops at a statement's first failure, so each failing statement reports one; the statements after it report their own. It checks syntax only, as the server would parse each statement before running it, plugins' grammars included: a name the database does not hold is not reported. A response holds at most 1,000 rows.

| Column | Type | Meaning |
|---|---|---|
| `Statement` | `INT32` | The statement's ordinal among the text's statements, from 0; whitespace and comments between semicolons are no statement. |
| `StatementFrom` | `INT32` | Where the statement starts, without the whitespace before it. |
| `StatementTo` | `INT32` | Where it ends, before its semicolon and the whitespace before that. |
| `From` | `INT32` | Where the token the failure is at starts. |
| `To` | `INT32` | Where that token ends: the rest of the statement for a string, bracketed name or comment left open, one character the lexer cannot read, or `From` itself for a failure at the end of the statement. |
| `Severity` | `STRING` | `Error`. |
| `Class` | `STRING` | The failure's class, as the server would report it on execution, such as `User`. |
| `Code` | `STRING` | The failure's code, such as `SYNTAX_ERROR`, `INVALID_VALUE` or `LIMIT_EXCEEDED` ([errors](errors.md)). |
| `Message` | `STRING` | The parser's message. For a statement that carries a credential in a plugin's grammar, such as `CREATE USER … WITH PASSWORD`, the message is withheld, as on execution, while the range still marks where the statement fails. |

A failure is placed at the token its rule names, or at the furthest token the parser reached where the rule names none. A failure inside a view's body is placed in the view's statement, and one inside a trigger's body at its `BEGIN`.

```sql
-- @text = 'FROM [HR].[Employees] WHER [Id] = 1;'
VALIDATE @text;
-- Statement 0, From 22, To 26 ('WHER'), Error, User, SYNTAX_ERROR, Unexpected token 'WHER' …
```

## Permissions and connections

On a connection bound to a database, both statements read that database's names and need `Read` on it, like a query. On a server-level connection they name none of the server's own objects: they complete the grammar, functions, settings and the databases the caller may read, and any authenticated caller runs them. Both run on read-only followers. With authorization auditing set to record allowed decisions, each request is audited by its own text, `COMPLETE @text AT @position`, never by the text it reads ([user administration](auth-admin.md)).

The server runs one statement at a time on a connection, so an editor asks on a connection of its own: there a request never waits behind a statement the editor's script runs, and never runs inside that script's transaction.

## See also

- [Reserved keywords](keywords.md) — `[Mnemic].[Keywords]` lists a loaded plugin's words for highlighting
- [Parameterized queries](parameters.md)
- [Syntax conventions](syntax.md)

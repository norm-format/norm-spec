# NORM Format Specification

Version: 0.1
NORM — Normalised Object Relational Model. File extension: `.norm`

## Overview

NORM is a text-based data format that represents JSON data as a collection of flat CSV-style tables. It supports lossless round-trip conversion with JSON, is optimised for reduced token usage in LLM consumption, and adds comment support.

## Conformance

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY in this document are to be interpreted as described in RFC 2119.

## Goals

- Lossless JSON round-trip (comments excepted)
- Reduced token usage compared to JSON
- All JSON data types supported
- Human and LLM readable table structure
- Comment support

## File Structure

### Encoding

NORM files MUST be encoded as UTF-8 without BOM. Files containing a UTF-8 BOM are invalid and a parser MUST reject them.

Line endings: LF (`\n`) SHOULD be used. Parsers MUST accept CRLF (`\r\n`) by stripping the `\r` before processing. Encoders SHOULD emit LF.

Null bytes (`\x00`) MUST NOT appear in a NORM file. A parser MUST reject any file containing a null byte.

### Document Structure

A NORM document consists of:

1. A root type declaration (first non-comment line)
2. One or more sections

Sections are separated by blank lines.

### Root Declaration

The root declaration MUST be the first non-comment line in the document:

```
:root     # root value is a JSON object
:root[]   # root value is a JSON array
```

NORM represents JSON documents whose root value is an object or an array. Root scalar values (string, number, boolean, null) are not supported. Encoders MUST reject JSON input whose root value is a scalar.

### Sections

Each section begins with a section header on its own line:

```
:name     # table section — represents a JSON object or array of objects
:name[]   # array section — represents a JSON array
```

The first section after the root declaration is the root content. It is identified by position, not by name.

Section names MUST match `[a-zA-Z_][a-zA-Z0-9_]*`. The section name is the identifier excluding the `[]` suffix — `:foo` and `:foo[]` share the name `foo` and cannot both appear in the same document. Section names MUST be unique within a document.

A parser MUST reject a document containing a section name that does not match `[a-zA-Z_][a-zA-Z0-9_]*` or duplicates an existing section name.

Every section except the root content section MUST be reachable via a reference chain starting from the root content. A parser MUST reject a document containing any section that is not reachable.

Encoders choose section names freely within these constraints. How an encoder derives a section name from a JSON key is an implementation detail not prescribed by this spec.

## Data Types

Values map directly to JSON types:

| NORM value | JSON type | Example |
|-----------|-----------|---------|
| `"text"` | string | `"hello"` |
| Bare number | number | `42`, `-3.14`, `1.5e10` |
| `true` / `false` | boolean | `true` |
| `null` | `null` | key present with null value |
| Empty cell | absent key | key omitted from object |
| `@N` | object | `@1` |
| `@name` | array | `@items`, `@tags` |
| `@[]` | empty array | `@[]` |

Strings MUST be quoted when their unquoted form would be parsed as another type:

- Looks like a number: `"42"`, `"-3.14"`
- Is a keyword: `"true"`, `"false"`, `"null"`
- Starts with `@`: `"@1"`, `"@tags"`

An empty quoted string `""` encodes a JSON empty string value — it is distinct from an empty cell, which encodes an absent key.

Bare numbers MUST conform to JSON number syntax (RFC 8259 §6). Values such as `+42`, `1.`, or `NaN` are not valid bare numbers and MUST be treated as strings, requiring quoting.

Parsers MUST trim leading and trailing whitespace from unquoted values. String values that require surrounding whitespace MUST be quoted.

Strings containing commas, line breaks, or quote characters MUST follow standard CSV escaping rules: wrap in double quotes and double any interior double quote characters.

## Table Sections

A table section contains a header row followed by zero or more data rows. A section with no data rows represents an empty JSON array `[]` or empty JSON object `{}`.

- **Header row**: comma-separated key names
- **Data rows**: comma-separated values, one row per JSON object

```
:address
pk,street,city,zipcode
1,Kulas Light,Gwenborough,92998-3874
2,Victor Plains,Wisokyburgh,90566-7771
```

If rows in the section are referenced by other sections, the first column MUST be named `pk` containing the row's primary key value. The `pk` column is structural — a parser MUST exclude it when reconstructing JSON objects. Duplicate column names in the header row are permitted; if a JSON object contains a key named `pk`, the header will contain two `pk` columns and the parser MUST treat only the first as structural.

### Heterogeneous Schemas

Rows within a section MAY have different keys. The header row defines the union of all keys present across all rows in the section. A cell left empty means the key is absent from that object in JSON — it is not null.

```
:animals
pk,type,name,indoor,saltwater
1,cat,Whiskers,true,
2,fish,Nemo,,true
```

Reconstructs as:

```json
[
  {"type": "cat", "name": "Whiskers", "indoor": true},
  {"type": "fish", "name": "Nemo", "saltwater": true}
]
```

Note that `indoor` is absent from the fish object and `saltwater` is absent from the cat object — not null.

A parser MUST omit any key whose cell is empty when reconstructing a JSON object. An encoder MUST use the union of all keys as the section header.

## Array Sections

An array section contains one value per row with no header row. Values may be scalars or references. Empty rows are not permitted — a parser MUST reject any array section containing an empty row.

```
:tags[]
admin
user
moderator
```

An array section may reference other sections, enabling arrays of arrays:

```
:grid[]
@row1
@row2

:row1[]
1
2

:row2[]
3
4
```

Reconstructs as `[[1,2],[3,4]]`.

## Primary Keys

Primary keys uniquely identify rows within the document for cross-referencing.

- **Format:** one or more digits with no leading zeros — `1`, `42`, `1000`; a parser MUST reject any pk value containing a leading zero
- **Scope:** globally unique across the entire document — a parser MUST reject any document where the same pk value appears more than once across all table sections
- **Required:** only when a section's rows are referenced by other sections

## References

A reference is an unquoted cell value beginning with `@`. There are three forms:

| Reference | Resolves to | Example |
|-----------|-------------|---------|
| `@N` | Row with `pk=N` in any table section | `@1` |
| `@name` | Section named `:name` (table) or `:name[]` (array) | `@items`, `@tags` |
| `@[]` | Empty array | `@[]` |

The reference form in the parent cell encodes the original JSON type:

| Parent cell | Original JSON |
|-------------|---------------|
| `@1` | `"address": {}` |
| `@items` (`:items` table section) | `"items": [{},{}]` |
| `@tags` (`:tags[]` array section) | `"tags": ["admin","user"]` |
| `@[]` | `"employees": []` |

### Reference Resolution

A parser MUST distinguish reference types as follows:

- If the value is `@[]` → emit an empty JSON array `[]`
- If the value after `@` consists only of digits → row reference; resolve by searching all table sections for a matching `pk` value; reconstruct as a JSON object
- Otherwise → section reference; if a table section `:name` exists, reconstruct all its rows as a JSON array of objects; if an array section `:name[]` exists, emit its values as a JSON array, recursively resolving any references

A parser MUST reject a document containing a reference that cannot be resolved — an `@N` with no matching pk, or an `@name` with no matching section.

A reference MAY target a section or row that appears later in the document; forward references are valid.

A parser MUST detect and reject circular references — a reference chain where resolving a value leads back to a row or section already in the resolution chain.

### Section Naming Constraint

Section names MUST start with a letter or underscore (`[a-zA-Z_][a-zA-Z0-9_]*`). This prevents purely numeric names (e.g. `42`) that would be ambiguous with row reference syntax — `@42` always resolves as a row pk lookup, making any purely numeric section name unreachable via a section reference.

### Quoting Constraint

A string value that begins with `@` MUST be quoted to avoid conflict with reference syntax: `"@1"`, `"@tags"`.

## Comments

Comments are discarded during JSON conversion and are not round-tripped. Three comment forms are defined:

- **Whole-line comments** — a line where `#` is the first non-whitespace character; the entire line is discarded. The optional document version declaration `# norm 0.1` MAY appear as the first line of a `.norm` file; parsers MAY use it to identify the spec version the document targets
- **Inline comments on structural lines** — anything from `#` to end-of-line on a root declaration or section header line is discarded
- **Data rows** — inline comments are not permitted; `#` in a data row is treated as part of the value and MUST appear inside a quoted string if it is string data

```
# User data exported 2026-04-03
:root[]          # root is an array
:data            # root content
id,name
1,Alice
```

## Conversion

### JSON to NORM

1. Emit the root declaration — `:root` or `:root[]`
2. Traverse the full JSON tree and identify all nested objects and arrays
3. Assign globally unique `pk` values to all rows that will be referenced by `@N` references
4. Substitute nested single-object field values with `@N` references
5. Substitute nested array-of-objects field values with `@name` references; emit the objects as a named table section
6. Substitute nested array field values with `@name` references; emit the array as a named array section, recursively substituting any non-scalar elements
7. Emit the root section header followed by the root content rows with all substitutions applied; quote string values inline as each value is emitted
8. Emit each nested table section header followed by its rows; quote string values inline as each value is emitted
9. Emit each array section header followed by its values; quote string values inline as each value is emitted

### NORM to JSON

1. Read the root declaration to determine the output type
2. If `:root`, the first section MUST be a table section — a parser MUST reject the document if the first section is an array section (`:name[]`)
3. If `:root`, the first section MUST contain at most one row — multiple rows would imply multiple root objects, which has no JSON equivalent; a parser MUST reject the document if the first section contains more than one row
4. If `:root`, reconstruct the single row of the first section as a JSON object (`{}` if no rows); if `:root[]`, reconstruct the first section as a JSON array
5. For each `@N` reference, locate the row with that `pk` and reconstruct as a JSON object
6. For each `@name` reference where `:name` is a table section, reconstruct all rows as a JSON array of objects
7. For each `@name` reference where `:name[]` is an array section, emit each row value as a JSON array element, recursively resolving any references
8. For each `@[]` reference, emit an empty JSON array `[]`
9. Map typed values to JSON: quoted value → string, bare number → number, `true`/`false` → boolean, `null` → null, empty cell → omit key

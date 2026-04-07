# NORM Format Specification

This repository hosts the specification for NORM (Normalised Object Relational Model), a text-based data format that represents JSON as flat CSV-style tables. File extension: `.norm`. Current version: 0.1.

## Repository Structure

| File | Purpose |
|------|---------|
| `spec.md` | Authoritative prose specification |
| `abnf.md` | ABNF grammar work-in-progress (deferred until encoder/decoder exist) |
| `users.json` | Example JSON input for format exploration |
| `users-edit.json` | Variant example JSON |

## Spec Conventions

- Conformance language follows RFC 2119: MUST, MUST NOT, SHOULD, SHOULD NOT, MAY
- Prose spec is the authoritative source; where ABNF and prose conflict, prose wins
- Examples in `spec.md` are normative — keep them consistent with the rules they illustrate
- The spec uses CommonMark markdown

## Key Format Concepts

### Document Model

A NORM document has a root declaration (`:root` or `:root[]`) followed by sections separated by blank lines. The first section after the root declaration is the root content, identified by position.

### Section Types

- Table section (`:name`) — header row + data rows; represents a JSON object or array of objects
- Array section (`:name[]`) — one value per row, no header; represents a JSON array

### References

| Syntax | Meaning |
|--------|---------|
| `@N` | Row with `pk=N` in any table section (reconstructs as object) |
| `@name` | Section named `:name` or `:name[]` |
| `@[]` | Empty array |

### Primary Keys

- Format: one or more digits, no leading zeros
- Scope: globally unique across the entire document
- Required only when a section's rows are referenced by `@N`

### Value Types

| NORM | JSON |
|------|------|
| `"text"` | string |
| bare number | number |
| `true` / `false` | boolean |
| `null` | null |
| empty cell | absent key (not null) |
| `@N` | nested object |
| `@name` | nested array |

## Editing Guidelines

- Do not introduce parser requirements without a corresponding MUST/MUST NOT statement
- When adding examples, verify they are consistent with all relevant rules
- The `pk` column is structural — parsers MUST exclude it from reconstructed JSON objects
- Section names follow `[a-zA-Z_][a-zA-Z0-9_]*`; purely numeric names are unreachable via section reference and are effectively prohibited
- Inline comments are permitted on structural lines (root declaration, section headers) but NOT in data rows — `#` in a data row is literal

## ABNF Grammar

`abnf.md` tracks open questions and a draft grammar. Do not add the grammar appendix to `spec.md` until:

1. A working encoder and decoder exist
2. Real-world test cases cover edge cases
3. All open questions in `abnf.md` are resolved

NOTE: The draft ABNF in `abnf.md` may diverge from the prose spec during iteration. The prose spec takes precedence.

## No Build or Test Commands

This is a documentation-only repository. There are no build steps, test runners, or CI pipelines. Validation is manual — read the spec and check examples by hand or against an external implementation.

# NORM ABNF Grammar — Project

## Context

This document tracks the work required to add a formal ABNF grammar appendix (RFC 5234) to spec.md. The grammar should be deferred until the format is proven — at minimum, a working encoder and decoder exist and real-world test cases cover the edge cases. This prevents the grammar from drifting out of sync with the prose spec during early iteration.

When ready, the grammar is appended to spec.md as an appendix. It does not replace the prose spec; the prose remains the human-readable authority. The ABNF provides an unambiguous reference for implementors and a mechanical basis for testing conformance.

---

## Scope

The grammar must cover:

- Document structure
- Version comment
- Root declaration
- Section headers (table and primitive array)
- Header rows and data rows
- CSV value rules (quoted strings, bare values, empty cells)
- Reference syntax (`@rN`, `@[rN|...]`, `@name`)
- Comment syntax (whole-line and inline)
- Value types: string, number, boolean, null, empty
- Whitespace and line ending rules
- Primary key format

Constraints that cannot be expressed in pure ABNF (such as global pk uniqueness and the `r[0-9]+` section name prohibition) must be noted inline as prose annotations within the appendix.

---

## Draft Grammar

```abnf
; ------------------------------------------------------------
; Document
; ------------------------------------------------------------

document        = [version-comment] root-decl
                  1*(blank-line / section)

version-comment = "#" SP "norm" SP version line-end
version         = 1*DIGIT "." 1*DIGIT

; ------------------------------------------------------------
; Root declaration
; ------------------------------------------------------------

root-decl       = (":root[]" / ":root") [inline-comment] line-end

; ------------------------------------------------------------
; Sections
; ------------------------------------------------------------

section         = table-section / prim-section

table-section   = table-header line-end
                  header-row line-end
                  1*(data-row line-end)

table-header    = ":" section-name [inline-comment]

prim-section    = prim-header line-end
                  1*(prim-value line-end)

prim-header     = ":" section-name "[]" [inline-comment]

; Section names are restricted to [a-zA-Z0-9_].
; A section-name MUST NOT match the pattern: "r" 1*DIGIT
section-name    = 1*(ALPHA / DIGIT / "_")

; ------------------------------------------------------------
; Rows
; ------------------------------------------------------------

header-row      = field-name *("," field-name)
field-name      = 1*(ALPHA / DIGIT / "_" / "-" / ".")

data-row        = cell *("," cell)
cell            = quoted-string / reference / bare-value / empty

prim-value      = quoted-string / reference / bare-value

empty           = ""   ; zero characters — encodes absent key

; ------------------------------------------------------------
; Quoted string (CSV rules)
; ------------------------------------------------------------

quoted-string   = DQUOTE *qchar DQUOTE
qchar           = %x20-21        ; space to ! (excludes DQUOTE)
                / %x23-7E        ; # to ~ (excludes DQUOTE)
                / %x80-10FFFF    ; non-ASCII UTF-8
                / escaped-quote
escaped-quote   = DQUOTE DQUOTE  ; "" encodes a literal "

; ------------------------------------------------------------
; References
; ------------------------------------------------------------

; A reference MUST NOT be quoted; a quoted @-string is a string literal.
reference       = "@" (array-ref / row-ref / named-ref)

row-ref         = "r" 1*DIGIT
array-ref       = "[" row-ref *("|" row-ref) "]"
named-ref       = section-name   ; MUST NOT match "r" 1*DIGIT

; ------------------------------------------------------------
; Bare values
; ------------------------------------------------------------

bare-value      = boolean / null-val / number / bare-string

boolean         = "true" / "false"
null-val        = "null"

number          = ["-"] (float / int)
int             = "0" / (NZDIGIT *DIGIT)
float           = int "." 1*DIGIT [exponent]
exponent        = ("e" / "E") ["+" / "-"] 1*DIGIT
NZDIGIT         = %x31-39        ; 1–9

; bare-string is any non-empty sequence that does not match boolean,
; null-val, number, or reference, and contains no comma, DQUOTE,
; CR, or LF. Encoders MUST quote strings that would otherwise match
; boolean, null-val, number, or reference patterns.
bare-string     = bchar 1*bchar
bchar           = %x20          ; space
                / %x21          ; !
                / %x23-26       ; # $ % &
                / %x28-2B       ; ( ) * +
                / %x2D-39       ; - . / 0-9  (digits allowed mid-string)
                / %x3B-7E       ; ; < = > ? @ A-Z [ \ ] ^ _ ` a-z { | } ~
                / %x80-10FFFF   ; non-ASCII UTF-8

; ------------------------------------------------------------
; Comments
; ------------------------------------------------------------

comment-line    = *WSP "#" *(WSP / VCHAR) line-end
inline-comment  = 1*WSP "#" *(WSP / VCHAR)

; ------------------------------------------------------------
; Whitespace and line endings
; ------------------------------------------------------------

blank-line      = *WSP line-end
line-end        = LF / CRLF

WSP             = SP / HTAB
SP              = %x20
HTAB            = %x09
LF              = %x0A
CR              = %x0D
CRLF            = CR LF
DQUOTE          = %x22
ALPHA           = %x41-5A / %x61-7A   ; A-Z / a-z
DIGIT           = %x30-39             ; 0-9
VCHAR           = %x21-7E             ; visible ASCII
```

---

## Open Questions

These must be resolved before the grammar is finalised.

### 1. Field names in header rows

The draft restricts `field-name` to `[a-zA-Z0-9_\-.]`. JSON keys used as column headers can contain any character. Decide whether field names in header rows follow the same `[a-zA-Z0-9_]` restriction as section names, or whether they are quoted-string capable.

Recommendation: allow quoted strings as field names in header rows, matching JSON key semantics. Update `header-row` accordingly.

### 2. Bare string definition

The `bare-string` rule above is a conservative approximation. It excludes characters that are reserved for other purposes but the boundary between a valid bare string and a value that must be quoted needs to be validated against the encoder rules in the prose spec. Cross-check before finalising.

### 3. Whitespace in data rows

The draft does not permit optional whitespace between cells and commas (e.g. `r1, Alice, 30`). Decide whether parsers must accept or must reject padding whitespace around delimiters. Recommendation: reject — CSV convention treats whitespace as significant, and permitting it adds ambiguity.

### 4. Empty sections

The grammar requires `1*` data rows per section. Decide whether an empty table section (header row with no data rows) or an empty primitive array section (header with no values) is valid. Recommendation: permit empty sections to allow encoders to represent empty arrays without special-casing.

### 5. Inline comment on `# norm` version line

The version comment is currently defined as a fixed structure. Decide whether an additional inline comment is permitted after the version token (e.g. `# norm 0.1 generated by tool`). Recommendation: permit trailing text after the version token on the version line.

---

## Appendix Placement

Insert the grammar as a new final section in spec.md:

```markdown
## Appendix: ABNF Grammar

The following ABNF grammar (RFC 5234) provides a formal syntax reference.
The prose specification is the authoritative source; where they conflict,
the prose takes precedence.

[grammar here]
```

---

## Checklist

- [ ] Resolve all open questions above
- [ ] Validate grammar against all examples in spec.md
- [ ] Validate grammar against encoder/decoder test suite
- [ ] Add to spec.md as appendix
- [ ] Update spec version if grammar addition constitutes a minor revision

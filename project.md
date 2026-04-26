# Project: Token-Efficiency Amendments to NORM v0.1

## Goal

Land the three amendments proposed in issue #3 (proposals A, B, C) into `spec.md`, with paired updates to fixtures so the documented examples remain canonical and round-trippable. The outcome is a NORM specification that closes the format's loss cases against minified JSON on small or deeply nested payloads while preserving its existing wins on tabular data.

## Scope

In scope:

- Proposal A: remove the blank-line-as-section-separator rule; replace with a header-driven section boundary rule.
- Proposal B: add an inline object literal `{key=value,...}` as a cell value for single-use scalar-only objects.
- Proposal C: add an informative (non-normative) Tokenization Notes appendix.
- Update every fixture pair under `fixtures/` to reflect proposal A and, where applicable, proposal B.
- Update `AGENTS.md` where it mentions blank-line separation between sections.

Out of scope:

- ABNF grammar changes in `abnf.md` — the grammar is already deferred until an encoder/decoder exists.
- Issue #4 (value-equal row deduplication). Not addressed here.
- Reworking proposal B as a "shared inline-objects section" pattern. The proposal itself notes this as a fallback if the new sub-syntax is rejected; if reviewers prefer that path, surface it via Issues Discovered before drafting.
- Defining a normative tokenizer profile. Proposal C is informative only.

## Current State

Repository layout:

- `spec.md` — authoritative prose spec, v0.1. The relevant lines today:
    - Line 39: `Sections are separated by blank lines.`
    - Line 120: empty-row rule that references the "blank-line section separator" as the reason a row needs at least one non-empty cell.
    - §Data Types (lines 71–104): scalar value mapping and quoting rules.
    - §References (lines 202–249): `@N`, `@name`, `@[]`, `@{}` syntax.
    - §Comments (lines 259–273): three comment forms.
    - §Conversion (lines 275–301): JSON↔NORM step lists.
- `AGENTS.md` — repo guide. Line 25 currently says: "A NORM document has a root declaration (`:root` or `:root[]`) followed by sections separated by blank lines."
- `abnf.md` — work-in-progress grammar; deferred. No edits needed for this project.
- `fixtures/` — eleven JSON+NORM pairs. Every multi-section `.norm` fixture (`object_root`, `empty_structures`, `nested_arrays`, `pk_collision`, `references`, `solar_system`) currently uses a blank line between sections. The cleanest candidates for inline-literal encoding under proposal B are single-row table sections referenced via `@N` whose JSON object is scalar-only: `references.norm` `:target` (`{"name":"Jupiter","type":"gas_giant"}`) and `object_root.norm` `:atmosphere` (`{"primary_gas":"CO2","pressure_kpa":0.636}`). Note that `solar_system.norm` `:star_composition` and the `:*_atm` sections are not candidates — those are arrays of objects (`composition`/`atmosphere` is a JSON array in `solar_system.json`) and proposal B inlines single objects only. The shared heterogeneous sections in `solar_system.norm` (`:orbital`, `:magnetic_field`, `:discovery`) hold individual scalar-only objects per row that would each qualify, but inlining them dissolves the shared section in pieces and is out of scope for this project's "minimum diff" guidance for existing fixtures.

Spec line numbers cited above were verified against the current `spec.md`: line 39 carries the blank-line-separator sentence, line 120 carries the empty-row rule referencing the blank-line separator, and line 223 carries the `@{}` MUST. §References spans lines 202–249, §Comments lines 259–273, §Conversion lines 275–301.

The closed issues #1 (spec gaps) and #2 (`@{}` sigil) are already reflected in the current spec. This project starts from the spec's current state.

## References

- GitHub issue #3 — `Token-efficiency improvements (drop blank-line separators, inline small objects, sigil guidance)`. Authoritative source for the three proposals, their motivation, and the suggested wording for the amendments.
- GitHub issue #1 (closed) — `Spec gaps surfaced by Rust codec roundtrip fuzzing`. Establishes the empty-row machinery (structural `pk`) that proposal A interacts with.
- GitHub issue #2 (closed) — `Add @{} sigil for empty objects`. Establishes the empty-object cell primitive that proposal B's empty-literal `{}` aliases.
- `~/.ai/docs/project-writing-guide.md` — the writing guide this document follows.
- `spec.md` — current authoritative text.
- `AGENTS.md` — repo conventions (RFC 2119 conformance language, examples are normative, prose wins over ABNF).

## Requirements

1. `spec.md` no longer states that sections are separated by blank lines. The new rule states that a section ends at the next section header or end of input, and that blank lines outside CSV-quoted strings are no-op whitespace that parsers MUST ignore and encoders SHOULD NOT emit. Encoders MAY emit a single blank line before a section header for human readability; parsers MUST treat such lines as no-ops, never as terminators.
2. The empty-row rule in §Table Sections is rewritten so it no longer hinges on the blank-line separator. The rule still requires every header row and every data row to contain at least one non-empty cell, but the justification cites the new section-boundary rule (a row with no cells would be indistinguishable from the no-op blank line that may precede a section header) and the structural `pk` machinery for empty objects remains intact and unchanged.
3. §Data Types includes `{key=value,...}` as a valid cell value mapping to a JSON object whose direct children are scalars only. The mapping appears in the value-mapping table alongside the existing entries.
4. §Inline Object Literal (new subsection, sibling to §References) defines the syntax, the disqualification rules, and the parser/encoder contract:
    - Form: `{key=value,key=value,...}` with ASCII `{` `}`, `=`, `,`.
    - Keys MUST match `[a-zA-Z_][a-zA-Z0-9_]*` (same grammar as section names).
    - Values follow NORM scalar quoting rules: bare numbers, bare `true`/`false`/`null`, otherwise CSV-quoted strings.
    - The literal's values MUST all be scalars: bare numbers, bare `true`/`false`/`null`, or CSV-quoted strings. Nested objects (including `{}`/`@{}` as a value), arrays, and `@N`/`@name` references inside the literal disqualify the form; encoders MUST emit a section instead. The top-level `{}` form is itself the empty inline literal, not a nested object, and remains permitted.
    - Validity rule (parser side, paired with the encoder disqualification above): an inline literal is valid only if it matches `{}` or `{key=value(,key=value)*}` where each key matches `[a-zA-Z_][a-zA-Z0-9_]*` and each value is a NORM scalar (bare number, bare `true`/`false`/`null`, or CSV-quoted string). A parser MUST reject any literal containing an empty key, an empty value, a trailing or repeated comma, a duplicate key within the same literal, an `@`-reference, an opening `[`, or any `{` appearing as a value inside a non-empty literal.
    - As a cell value, `{}` is the empty inline literal and is equivalent to `@{}`; parsers MUST accept either form for an empty-object cell, encoders MAY emit either, and SHOULD prefer `@{}` for consistency with the existing reference-syntax-only-for-empties pattern. The equivalence applies at cell level only — `{}` and `@{}` are not permitted as values inside a non-empty inline literal.
    - Whitespace handling inside the literal is specified explicitly (mirror the existing trim-unquoted-whitespace rule for scalar cells).
    - The `#` character inside an inline literal is literal, governed by the existing §Comments rule that `#` in a data row is not a comment. No new prose is required, but the §Inline Object Literal subsection MAY note this in passing if it aids reader comprehension.
5. The interaction between an inline object literal and CSV row parsing is specified unambiguously per Issues Discovered #1: an unquoted cell whose first character is `{` opens an inline literal; commas inside the literal do not terminate cells; the literal ends at the matching unquoted `}`, which MUST be followed by a `,` or end-of-line. A literal string value containing any of the literal's structural characters (`}`, `,`, `=`, or `{`) MUST be CSV-quoted inside the literal; once CSV-quoted these characters are literal — including the `=` that elsewhere separates key from value. The spec MUST include an example whose quoted string value contains both a `,` and an `=`, so readers see the structural characters demonstrated as literal under CSV quoting.
6. The string-quoting list in §Data Types is extended so that any string whose unquoted form would be parsed as an inline object literal MUST be CSV-quoted. Concretely: a string value beginning with `{` MUST be CSV-quoted in any cell position where the inline-literal form is recognised.
7. §References gains no new reference forms from this project. `@{}` remains the canonical empty-object cell reference; the inline-literal rule simply names `{}` as an accepted alias inside the literal context. (`@{}` outside an inline literal is unchanged.) The existing sentence in §References (currently `spec.md` line 223, "Encoders MUST emit `@{}` for any cell whose value is an empty JSON object.") is relaxed from MUST to SHOULD so that an encoder emitting the alias `{}` form remains conformant; the recommendation continues to be `@{}`.
8. §Conversion is updated:
    - JSON-to-NORM gains a step describing when an encoder substitutes an inline object literal for an `@N`-plus-section pair (single occurrence, scalar-only children).
    - NORM-to-JSON gains a step describing inline literal resolution.
9. A new appendix titled "Tokenization Notes" is added at the end of `spec.md`, marked informative (non-normative), with the wording suggested in proposal C of issue #3. The heading uses level `##` to match the other top-level sections (`## Overview`, `## Conformance`, etc.); the informative status is signalled in prose at the top of the appendix, not in the heading itself. The proposal-C wording is rephrased to drop RFC 2119 keywords so that MUST/SHOULD/MAY remain reserved for normative content elsewhere in the spec — in particular, "Implementations targeting LLM consumption MAY benchmark alternative formulations…" becomes "Implementations targeting LLM consumption can benchmark alternative formulations…" (or equivalent neutral phrasing).
10. Every fixture under `fixtures/` is updated so the `.norm` file no longer uses blank-line separators between sections and conforms to the amended rules. JSON files are updated only if a fixture is added or restructured — the existing JSON content is the source of truth and MUST round-trip to the new NORM byte-for-byte under an encoder following the amended spec.
11. At least one new fixture pair demonstrates an inline object literal as a cell value (e.g. a `meta` field on a root object encoded as `{created=...,version=...}`). The pair is named clearly (e.g. `inline_object.json` / `inline_object.norm`) and exercises both a single inline literal and a row that mixes an inline literal with other cell types.
12. `AGENTS.md` is updated where it currently describes sections as separated by blank lines, so the repo guide matches the amended spec.
13. The spec version is bumped from v0.1 to v0.2 per Issues Discovered #2: `spec.md` line 3, the `# norm 0.1` example in §Comments, `AGENTS.md` line 3, and `fixtures/comments.norm` line 1 are all updated.
14. §Inline Object Literal explicitly permits the literal in two cell positions per Issues Discovered #3: (i) a cell in a table-section row and (ii) the sole value of an array-section row. The subsection includes one short example for each position.

## Implementation Plan

1. Re-read issue #3 and confirm the three proposals are still framed as A (drop blank lines), B (inline literal), C (tokenization notes) before drafting any spec changes. If proposal B's sub-syntax has been deprioritised in conversation since the issue was filed, stop and raise it via Issues Discovered rather than skipping it silently.
2. Update `spec.md` proposal A first:
    - Replace the blank-line separator sentence in §File Structure / Document Structure with the new section-boundary rule.
    - Rework the empty-row paragraph in §Table Sections so the justification cites the new rule.
    - Sweep the rest of the spec for any other reference to blank-line separation and update.
3. Update `spec.md` proposal B:
    - Extend the §Data Types value-mapping table with the inline literal entry.
    - Add the §Inline Object Literal subsection. Place it after §References for discoverability. The subsection MUST state the balanced-brace tokenization rule per Issues Discovered #1, MUST permit the literal in both table-row cell and array-section row positions per Issues Discovered #3, and MUST include the comma-bearing quoted string example.
    - Extend the quoting bullet list in §Data Types to require quoting for string values beginning with `{` in inline-literal-eligible positions.
    - Relax the existing `@{}` encoder rule in §References (currently `spec.md` line 223) from MUST to SHOULD per Requirement 7, so that the alias `{}` form remains conformant.
    - Update §Conversion JSON-to-NORM and NORM-to-JSON step lists.
4. Update `spec.md` proposal C: add the "Tokenization Notes" appendix. Mark it informative.
5. Bump the version per Issues Discovered #2: `spec.md` line 3 to `Version: 0.2`, the `# norm 0.1` example in §Comments to `# norm 0.2`.
6. Update `AGENTS.md`: remove the blank-line phrasing on line 25, and bump the version on line 3 to `0.2`.
7. Regenerate every `.norm` fixture under `fixtures/` to remove blank-line separators. Update `fixtures/comments.norm` line 1 to `# norm 0.2`. Verify each fixture's JSON ↔ NORM mapping by hand against the amended spec; the round-trip must remain lossless.
8. Pick one or two fixtures where a single-use scalar-only nested object dominates and re-encode them using inline literals to demonstrate the new form. At least one must be a brand-new fixture added specifically to exhibit the literal in isolation. Existing fixtures whose intent is to test other features (`heterogeneous`, `quoting`, `csv_escaping`, `pk_collision`) SHOULD NOT be changed beyond removing blank lines, so each fixture remains targeted at its original purpose.
9. Re-read the changed spec end-to-end to confirm internal consistency: the value-mapping table, the quoting list, §Data Types prose, §Inline Object Literal, §References, and §Conversion must all agree on the inline-literal contract.

## Constraints

- The spec uses CommonMark markdown and RFC 2119 conformance language. New normative statements MUST use MUST / MUST NOT / SHOULD / SHOULD NOT / MAY exactly as in the existing spec.
- Examples in `spec.md` are normative. Any new example MUST satisfy every rule it touches, including the existing quoting and pk rules. Any changed example MUST still illustrate the rule it was originally placed to illustrate.
- Round-trip fidelity is non-negotiable. Every fixture pair MUST round-trip losslessly under the amended spec; comments are the only documented exception.
- The amendments MUST NOT break existing closed issues #1 and #2. The empty-row structural-`pk` rule from #1 stays. The `@{}` cell sigil from #2 stays.
- Inline object literals are restricted to scalar children. They MUST NOT carry nested inline objects, arrays, or `@`-references. The spec must state this restriction explicitly so encoders fall back to a section deterministically rather than choosing per implementation.
- Section names continue to match `[a-zA-Z_][a-zA-Z0-9_]*`. Inline-literal keys reuse the same grammar — do not introduce a divergent identifier rule.
- Do not modify `abnf.md`. The grammar work is deferred, and editing it now risks rotting against the prose.
- Do not add a build, test runner, or CI pipeline. This repo is documentation-only by design.

## Implementation Guidance

- The order of the proposals matters. Do A first because it simplifies the empty-row prose; do B second because its quoting addition layers on top of an already-clean §Data Types; do C last because it is purely additive.
- Proposal B is the most invention-heavy change. Be explicit. The reader of the spec is a future implementer of an encoder/decoder — every disqualification (nested objects, arrays, references, non-identifier keys) needs to be a sentence they can implement directly.
- Encoders SHOULD prefer the inline literal for any qualifying object that appears exactly once in the document tree. The spec states this as SHOULD, not MUST, so encoders that produce sections for everything remain conformant. Make this explicit so neither readers nor implementers have to infer it.
- For empty inline literals, the spec MUST state explicitly that `{}` and `@{}` are interchangeable inside cell positions. Encoders MAY pick either; parsers MUST accept both. Pick a recommendation (issue #3 is silent on which is preferred) — leaning toward `@{}` for consistency with the existing reference-syntax-only-for-empties pattern is reasonable, but record the decision.
- When updating fixtures to drop blank lines, do not also rename sections, reorder rows, or otherwise restyle. The diff should be the minimum change needed to reflect proposal A. Keep proposal-B-driven fixture changes confined to the new fixture and any one or two existing fixtures explicitly chosen to exhibit the literal.
- The Tokenization Notes appendix is short. Use the wording in issue #3 as the basis. Keep it under one screen.
- Issue #3's proposal-B sketch shows `{created=2024-01-01,version=3}` with `2024-01-01` unquoted. This is a transcription slip in the issue, not a rule the spec adopts. `2024-01-01` is not valid JSON number syntax (RFC 8259 §6, cited at `spec.md` line 100) and is therefore a string under the existing NORM quoting rules. The §Inline Object Literal example and any new fixture MUST use the CSV-quoted form, e.g. `{created="2024-01-01",version=3}`. Apply the same rule to any other non-numeric token (hyphenated identifiers, ISO timestamps, UUIDs, etc.).

## Issues Discovered

1. Inline-literal vs. CSV tokenization (design) — Resolved: balanced-brace scan (rule b).

   Issue #3 specifies the inline literal as `{key=value,key=value,...}` placed inside a CSV cell, but the cell is itself comma-delimited. A naive CSV tokenizer encountering `meta,name\n{a=1,b=2},Alice` will split on every comma, producing four cells, not two. The spec must commit to a rule that resolves this. Candidate rules:
    - (a) Require the inline literal to be CSV-quoted: `"{a=1,b=2}",Alice`. Simplest; preserves CSV semantics. Costs ~2 bytes per literal but no new lexer.
    - (b) Treat `{` at the start of an unquoted cell as a balanced-brace literal whose internal commas do not terminate cells; the cell ends at the matching `}` followed by a `,` or end-of-line.
    - (c) Disallow commas inside literals entirely; require any string value containing a comma to disqualify the inlining (force a section).

   Why it matters: every encoder and decoder needs the same rule, and the savings claimed in the proposal depend on the choice. Rule (a) gives back about half the savings. Rule (b) preserves the savings but adds a small lexer mode. Rule (c) keeps lexing trivial but fragments the qualifying-input set.

   Resolution: rule (b). An unquoted cell whose first character is `{` opens an inline literal; the parser scans forward respecting CSV-quote semantics for string values inside the literal; the literal ends at the matching unquoted `}`, which MUST be followed by a `,` (next cell) or end-of-line. A literal string value containing any of the literal's structural characters (`}`, `,`, `=`, or `{`) MUST be CSV-quoted inside the literal so none of them is consumed by the tokenizer. The §Inline Object Literal subsection MUST include an example showing a quoted string value that contains both a `,` and an `=`, demonstrating that all such structural characters are literal once CSV-quoted.

2. Version bump (decision) — Resolved: bump to v0.2.

   Spec line 3 says `Version: 0.1`. Proposal B introduces new syntax (`{key=value,...}`) that a v0.1 parser cannot safely parse — best case it errors, worst case it mis-tokenizes via naive CSV split. A version bump gives parsers a clean signal to reject documents they do not understand.

   Resolution: bump to v0.2. Edits required:
    - `spec.md` line 3: `Version: 0.1` → `Version: 0.2`.
    - §Comments example referencing `# norm 0.1` → `# norm 0.2`.
    - `AGENTS.md` line 3: `Current version: 0.1` → `Current version: 0.2`.
    - `fixtures/comments.norm` line 1: `# norm 0.1` → `# norm 0.2`.

   No external coordination cost: there is no published implementation pinning on the version string that the project author does not also own.

3. Inline literals in array sections (gap) — Resolved: permitted.

   Issue #3 frames inline literals as cell values in table sections. An array section row is also a single cell value. The spec must state whether an inline literal MAY appear as an array-section row value. The principle (it is a cell value) suggests yes; consistency with `@{}`, which the current spec accepts in array-section rows, supports the same answer.

   Resolution: inline literals are valid as either (i) a cell in a table-section row or (ii) the sole value of an array-section row. The §Inline Object Literal subsection MUST state both positions explicitly and include one short array-section example.

## Acceptance Criteria

- `spec.md` no longer contains the sentence "Sections are separated by blank lines." or any prose that depends on a blank-line section separator.
- §Data Types contains a value-mapping entry for the inline object literal.
- A subsection (e.g. §Inline Object Literal) defines the syntax, disqualification rules, the balanced-brace CSV tokenization rule, the empty-`{}` aliasing rule, the rule permitting the literal in both table-row cells and array-section rows, and the parser/encoder contract using RFC 2119 keywords. The subsection includes an example whose quoted string value contains both a `,` and an `=` (showing all structural characters of the literal as literal under CSV quoting) and a short array-section example.
- §Conversion contains updated JSON-to-NORM and NORM-to-JSON steps that mention the inline literal.
- §Tokenization Notes appears at the end of `spec.md` and is marked informative.
- `spec.md` line 3 reads `Version: 0.2`. The `# norm 0.X` example in §Comments reads `# norm 0.2`.
- `AGENTS.md` line 3 reads `Current version: 0.2`. The blank-line section-separation phrasing is removed.
- `fixtures/comments.norm` line 1 reads `# norm 0.2`.
- Every `.norm` fixture under `fixtures/` parses under the amended rules and round-trips to its paired `.json` fixture. The check is performed by reading the spec and walking the fixture by hand; an external implementation is not required for this project.
- At least one new fixture pair under `fixtures/` exercises the inline object literal.

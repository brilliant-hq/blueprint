# Blueprint Language Specification

Version: `bp:v1`
Status: Stable. This document specifies the `bp:v1` grammar and the conformance requirements a Blueprint implementation must satisfy.

Blueprint is Brilliant's design language: a concise, lossless, versioned text representation of a two-dimensional vector design. A Blueprint document names elements (rectangles, circles, text, frames, vectors, components, and more), their geometry, paint, layout, and relationships. The same document always resolves to the same design, and any design serializes back to a document that is byte-for-byte stable across save cycles. Blueprint is therefore a stable, human-readable, diffable identifier for a design.

This specification is the authoritative language reference. It defines the grammar, the static semantics (validation and diagnostics), the dynamic semantics (what each construct builds to in the element model, including elided defaults), and the two conformance dialects (forgiving authoring and strict canonical storage). The informal, tutorial-voiced authoring guides published alongside this document under [`reference/`](reference/) teach how to write Blueprint by example; this document specifies what the language *is*.

Blueprint pairs with, but is distinct from, the design-system DSL that defines the `$token` catalog a Blueprint document binds against. That is a separate language with its own specification at <https://github.com/brilliant-hq/design-system>. Blueprint references its tokens (§16) but does not define them.

## License and scope

This specification is published so that it can be implemented. You may write your own encoder, decoder, renderer, or tooling against it and ship that implementation, commercially or otherwise, with no license from Brilliant and no permission required. An independent implementation is the intended outcome of publishing a conformance specification: a program that satisfies the lossless contract (§17) conforms, whoever wrote it.

That permission covers the *language*, meaning this grammar and its semantics. It does not extend to Brilliant's own software. The Brilliant application, website, and rendering engine, including the WebAssembly engine build served from brilliant.design, are proprietary and separately licensed under the terms at <https://brilliant.design/terms>. Implement the specification, not our binaries.

## Design goals

These four goals are the invariants every rule in this specification serves. Where a design choice is ambiguous, the resolution that best preserves these goals wins.

1. **Lossless.** A design serializes to exactly one canonical Blueprint document, and that document rebuilds to a structurally identical design. Every persistent property of every element round-trips, and a re-emission of a decoded document is byte-identical to the original. A property that does not round-trip is a defect, not an accepted limitation (§17.3).

2. **Minimal.** A document stores the least information that losslessly determines the design. Derived state (bounding boxes, resolved auto-layout positions, hug and fill extents, instance subtrees) is reconstructed by the builder rather than stored, and values equal to their version's frozen default are elided.

3. **Concise and readable.** One element per line; terse, composable operators; legible to humans and agents alike. The entire grammar is built on four token shapes (§3.9) and three cross-cutting idioms; no other punctuation carries meaning.

4. **Versioned.** Every stored document carries a grammar version stamp (`bp:vN`). The language evolves under an explicit backwards-compatibility regime: frozen per-version default tables, additive changes without a version bump, and forward migration for breaking changes (§18).

## 1. How to read this specification

### 1.1 Grammar notation

Grammar productions are written in an EBNF dialect with the following meaning. This dialect is used consistently in every inline fragment and in the collected grammar of Appendix A.

| Form | Meaning |
|---|---|
| `Name ::= ...` | A production defining nonterminal `Name`. |
| `"lit"` | A literal terminal; the exact characters between the quotes. |
| `CamelCase` | A nonterminal reference. |
| `X Y` | `X` followed by `Y` (concatenation). |
| <code>X &#124; Y</code> | `X` or `Y` (alternation). |
| `X?` | Zero or one occurrence of `X`. |
| `X*` | Zero or more occurrences of `X`. |
| `X+` | One or more occurrences of `X`. |
| `( X Y )` | Grouping in the metalanguage. To avoid confusion with the literal parenthesis terminal, literal parentheses are always written `"("` and `")"`. |
| `/regex/` | A terminal matching the given regular expression. |
| `(* text *)` | A non-normative comment inside a production. |

Two conventions specific to Blueprint's line orientation:

- `NL` denotes an end-of-line. Blueprint is line-oriented (§3.1); a production never spans a line break unless it explicitly consumes `NL`.
- `INDENT` denotes leading indentation whitespace, whose length determines nesting depth (§3.1). It is never a free-form terminal; its semantics are given normatively in §3.1.

Because Blueprint is parsed line-by-line by a hand-written tokenizer rather than by a context-free grammar engine, the productions in this document describe the *accepted* shape of each construct. Where the implementation accepts a superset of, or diverges from, what a strict grammar would admit, a non-normative NOTE records the actual behavior. The grammar is the specification of intent; the NOTEs record implementation reality where a reader would otherwise be surprised.

### 1.2 Conformance

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in RFC 2119 and RFC 8174 when, and only when, they appear in all capitals.

A **conforming Blueprint implementation** is one that comprises a *parser*, a *validator*, a *builder*, and an *encoder*, and that satisfies the requirements of this specification. Specifically:

- The parser MUST accept every document conforming to the authoring dialect (§16.1) and MUST produce the diagnostics this specification assigns to malformed or absorbed constructs.
- The builder MUST realize the dynamic semantics of §5 through §16 and MUST apply the frozen defaults of Appendix B for elided values.
- The encoder MUST emit the strict canonical storage dialect (§16.2) and MUST satisfy the lossless round-trip contract of §17.3.
- An implementation MAY additionally offer non-canonical read/authoring emission modes (terser output for display), provided the stored, canonical form remains lossless.

Two **conformance dialects** are defined and their requirements are normative: the **forgiving authoring dialect** (§16.1), which the parser accepts, and the **strict canonical storage dialect** (§16.2), which the encoder produces and which decodes with zero normalization. The parse surface is a superset of the authoring surface: the parser reads storage-only tokens without error so that a pasted canonical body never crashes, but those tokens are produced only by the encoder in canonical mode.

### 1.3 Diagnostics

The validator emits diagnostics, each carrying a **severity** and a **category**. Severity governs whether the line is applied:

- **error**: the line is NOT applied; it increments the fatal error count. Lines before the first fatal error in a block are still applied (§4.5).
- **warning**: the line IS applied; the diagnostic reports something that looks wrong.
- **info**: the line IS applied; the diagnostic teaches the canonical form of a construct that was auto-corrected (§16.1).

Categories are `syntax`, `property`, `layout`, `reference`, and `composition`. The complete diagnostic index, including code, severity, and triggering condition, is Appendix C. Diagnostic codes are of the form `Bnnn`.

An unrecognized *property* token is a non-fatal error: it is reported, dropped, and the element is still created. Malformed *arguments* to a construct (for example a zero-length `line()`) are fatal.

### 1.4 The processing model

A Blueprint document is processed in three phases, in order:

1. **Parse.** `BlueprintParser` tokenizes each line and produces a per-line map of keys to values. This phase is pure: it does not consult the target canvas or the design system. The map is the contract between parse and build.
2. **Validate.** The validator runs per-line semantic checks against each parsed map (§1.3), interleaved with parsing inside the line processor.
3. **Build.** The builder realizes each map as an element in the model, driving the real element-creation pipeline (create, modify, reorder, reparent, finalize components and slots). Defaults are applied here; derived geometry is computed here.

The inverse direction, design to document, is performed by the **encoder**. Encoding for at-rest storage always produces the strict canonical dialect (§16.2).

### 1.5 Relationship to other documents

- The authoring guides under [`reference/`](reference/) (`core`, `paint`, `layout`, `text`, `vectors`, `lines`, `effects`, `components`, `arcs`, `images`, `directives`, and others) are the informal, example-driven reference for authors. They are complementary; where they and this specification differ in rigor, this document governs the language definition and they govern authoring guidance.
- The design-system DSL that defines the `$token` catalog is specified separately at <https://github.com/brilliant-hq/design-system>.
- How an agent emits Blueprint inside a chat reply (the surrounding envelope, not the language) is covered by the authoring guides and is out of scope here.

---

## Table of contents

- [2. Document structure](#2-document-structure)
- [3. Lexical structure](#3-lexical-structure)
- [4. First-token semantics](#4-first-token-semantics)
- [5. Element types](#5-element-types)
- [6. Property reference](#6-property-reference)
- [7. Paint: fills and strokes](#7-paint-fills-and-strokes)
- [8. Element-level effects](#8-element-level-effects)
- [9. Text](#9-text)
- [10. Auto layout](#10-auto-layout)
- [11. Vectors](#11-vectors)
- [12. Lines and connectors](#12-lines-and-connectors)
- [13. Components](#13-components)
- [14. Masks, booleans, and parent types](#14-masks-booleans-and-parent-types)
- [15. Modify semantics](#15-modify-semantics)
- [16. Directives](#16-directives)
- [17. The two dialects and the lossless contract](#17-the-two-dialects-and-the-lossless-contract)
- [18. Versioning and migration](#18-versioning-and-migration)
- [Appendix A. Collected grammar](#appendix-a-collected-grammar)
- [Appendix B. Frozen bp:v1 default tables](#appendix-b-frozen-bpv1-default-tables)
- [Appendix C. Diagnostics index](#appendix-c-diagnostics-index)
- [Appendix D. Preprocessing and absorption catalog](#appendix-d-preprocessing-and-absorption-catalog)

The design-system binding surface (`$token` references, tagged literals, discipline modes) is specified across §3.6, §3.8, and the design-system note at the end of §16. The two conformance dialects and the lossless contract are §17; versioning and migration are §18.

---

## 2. Document structure

A Blueprint document is stored as the body of a Brilliant `.bl` file: the canonical extension new writes create, and the language's public identity. The legacy extension `.design` is still read and is migrated to `.bl` when a project is opened, so a not-yet-migrated project keeps working; the two extensions denote the same container. The container has four parts in order: a version stamp, a YAML frontmatter block, a separator, and the Blueprint body.

```
Document   ::= VersionStamp NL Frontmatter Separator NL Body
VersionStamp ::= "bp:v" Int
Frontmatter  ::= YamlMap                  (* canvas-level scalar map; MAY be empty *)
Separator  ::= "---"
Body       ::= ( Line NL )*
```

### 2.1 Version stamp

The first line of the document is the version stamp `bp:vN`, where `N` is the grammar version. A document is a canonical Blueprint document if and only if its first line begins with `bp:v`. The current version is `1`; the stamp for this specification is `bp:v1`.

A document whose first line does not begin with `bp:v` is not a Blueprint document. It is treated as legacy YAML (the v0 fallback, §18.4). A stamp with a version *newer* than the implementation's current version MUST be rejected: it was written by a newer build and cannot be safely decoded.

NOTE: The `bp:v` stamp is a claim about framing, not proof of content. On decode the implementation additionally inspects the body: a `bp:v`-stamped document whose body is in fact legacy YAML is routed through the legacy YAML lane rather than the canonical builder. The effective decode lane is therefore chosen by the body, gated behind the stamp; a genuine canonical body under a `bp:v` stamp always takes the canonical lane.

### 2.2 Frontmatter

Between the version stamp and the separator is a YAML map carrying canvas-level state. The canvas scalars live under a top-level `canvas:` key; the recognized scalars are `background` / `backgroundColor`, `backgroundEnabled`, `blackboardColor`, `rulerGuides`, and `designSystem`. The frontmatter MAY be empty. A `version:` key, if present in legacy input, is dropped on canonical encode because the `bp:v` stamp is the version of record.

### 2.3 Separator and body

A single line consisting of exactly `---` separates the frontmatter from the body. The body is the sequence of Blueprint lines specified by the rest of this document. Every body line begins with either an element-identifier character or indentation whitespace, so a bare `---` can never occur inside the body; the first `---` after the frontmatter is unambiguously the separator.

### 2.4 Worked example

A minimal canonical document with one rectangle:

```
bp:v1
canvas:
  backgroundEnabled: true
---
{id} r p(10,20) s(100,50) f[(f1,#FF0000)] "Rect"
```

Here `{id}` is a carried element identifier (a hexadecimal id present in stored bodies; §3.7, §16.2). In authored input, new elements carry no id and the first token is the type token instead.

NOTE: The Blueprint *body* is what this specification primarily defines; the `.bl` container (stamp, frontmatter, separator) is the storage embedding. In an authoring context (for example, submitting lines to the element-creation tool), only the body is provided, and no `bp:v1` stamp is written.

---

## 3. Lexical structure

### 3.1 Line orientation and indentation

Blueprint is line-oriented: exactly one element is defined per physical line, and every property of that element appears on the same line. There are no line-continuation characters for element definitions. The only constructs that attach to a *previous* element across lines are the `spans[...]` text-run continuation (§9) and the `vr(...)` vector-region continuation (§11); both are indented under the element they modify.

Leading spaces determine nesting depth. Depth is `floor(leadingSpaces / 2)`: two spaces per level. A line at depth *d+1* is a child of the nearest preceding line at depth *d* or shallower whose type admits children.

```
Line       ::= INDENT ( Blank | Comment | ElementLine | Continuation | MetaLine )
INDENT     ::= ( "  " )*                  (* two spaces per depth level *)
```

NOTE: Indentation is measured only in space characters; tabs are not counted. An odd leading-space count is floored, so three leading spaces yield depth 1 with no diagnostic. Authors SHOULD use exact multiples of two spaces.

Only frames, groups, and auto-layout parents (and, transitionally, `clone`) admit children. Nesting a child under a leaf element (rectangle, circle, text, vector, svg) is an error, except that a create-time leaf that would otherwise be a plain rectangle or default circle is silently converted into a container frame when it acquires a child (§5.13, leaf absorption). A child line whose properties include an explicit `parent(id)` overrides indentation-derived parenting.

### 3.2 Tokenizer

After preprocessing (§16.1, Appendix D), a line is split into tokens on space characters, but a space splits only when it is not inside double quotes, parentheses, or brackets. The tokenizer tracks three nesting contexts simultaneously: quote depth, parenthesis depth, and bracket depth. A space is a token boundary only when parenthesis depth and bracket depth are both zero and the scanner is not inside a quoted string.

```
Line body   ::= Token ( WS+ Token )*
Token       ::= /[^ ]+/ with balanced "()" "[]" and "\"...\"" spans preserved
```

Inside a quoted string, a backslash escapes the next character. Unbalanced closing delimiters clamp the depth at zero rather than underflowing. There is no single-quote string in the tokenizer; single quotes are stripped only for certain directive arguments (§3.4).

### 3.3 Comments and checkpoints

Blueprint has three comment syntaxes. All are quote-, parenthesis-, and bracket-aware in their inline forms, so a comment marker inside a string or an argument list is literal text.

```
Comment    ::= HashComment | DashComment | SlashComment
HashComment ::= "#" ( WS | EOL )          (* "# text" or a bare "#" *)
DashComment ::= "--" AnyText
SlashComment ::= "//" AnyText
```

- **`#` (hash).** `# text` or a bare `#` is a full-line comment and is skipped. A `#` immediately followed by a non-space character is not a comment; it is a session reference (`#name`) or an identifier (§3.7). The disambiguation is: `#` begins a comment only when the line is exactly `#` or the character after `#` is a space.
- **`--` (double dash).** A full-line or inline comment. It is plain narration and carries no checkpoint semantics.
- **`//` (double slash).** A full-line or inline comment. Additionally, a non-empty `//` payload is an **undo checkpoint label**. A standalone `// label` line registers a checkpoint marker; a trailing `// label` on an element line stamps that element's line with the checkpoint label. An empty `//` (no payload) is skipped.

Inline comment stripping treats `--` and `//` as a comment start only when parenthesis depth and bracket depth are zero and the scanner is not in a quote. Thus `t("hello -- world")` keeps its text, and `st[(...,--...)]` is not truncated.

Checkpoint semantics: a checkpoint label records the current undo depth for the session and canvas. The directive `undo("label")` (§16) rewinds to that depth; `redo("label")` replays forward. See §16 for the directive forms.

### 3.4 Strings and escapes

Strings are double-quoted. Unquoting strips one surrounding pair of double quotes. The following escape sequences are processed inside a string: `\n` (newline), `\t` (tab), `\"` (double quote), `\\` (backslash), and `\uXXXX` (a four-hex-digit Unicode code point). Any other `\x` sequence passes through unchanged.

Because each element occupies one physical line, a newline inside text content is written as the escape `\n`, never as a raw line break. A quote inside text is written `\"`.

Certain directive arguments (for example `override("Label")`) accept a single pair of either `"` or `'` quotes, which is stripped for by-name resolution.

### 3.5 Numbers and units

Numbers are decimals parsed by the host's double parser. Negative numbers are permitted. There is no required hexadecimal or scientific form beyond what the double parser accepts. An unparseable numeric slot generally falls back to `0` or a documented per-slot default rather than failing the line.

Units:

- A `px` suffix on a number is stripped during preprocessing (`s(100px)` becomes `s(100)`), except inside `ls(...)` where it is meaningful, and except when preceded by a quote or word character.
- `ls(N)` letter spacing: a bare number is pixels; `Npx` is pixels with an explicit unit; `Nem` is a multiplier of the font size. Storage is always in pixels; an `em` value converts to `N * fontSize`.
- `angle(...)` in a glass fill: a bare number is degrees (converted to radians); `Nrad` is radians verbatim.
- Gradient angles: a CSS `deg` suffix is stripped during preprocessing; the value is a bare number of degrees.

### 3.6 Design-system token references

A bare token of the form `$name.path` binds a design-system token on the slot where it appears. The parser is context-free about tokens: it records the reference in a companion `...Token` or `...TokenRef` map key and leaves the concrete literal at a hardcoded default (0, null, or a codec default). The reference is resolved after build, against the live design system, by the token normalizer.

The reference grammar is everything after `$` up to the token boundary; the leading `$` is stripped and the remainder stored verbatim. Token references appear on color slots (solid fills, gradient and shader stops), on numeric-or-tagged slots (spacing, radius, stroke width, opacity), and on text slots (font family, size, weight, line height, letter spacing).

### 3.7 Session references and identifiers

A `#`-prefixed token is either an element **identifier** or a **session reference**, resolved by these rules in order:

1. If the characters after `#` match `/^[0-9a-fA-F]{8,16}$/` (8 to 16 hexadecimal digits), the token is coerced to a bare element **id** (a modify-by-id target), with an info diagnostic noting that hexadecimal ids do not use the `#` prefix.
2. Otherwise, if the characters after `#` match `/^\d+$/` or `/^[a-zA-Z_][\w-]*$/`, the token is a **session reference** (`#1`, `#card`, `#hero-section`, `#a_b`).

As a *trailing* token on a create line, `#name` (matching the same word or numeric pattern) is an `assignRef`: it binds a session reference to the freshly created element's minted id. Session references are scoped per session and canvas, resolved after parse. A reference that resolves to a still-live element from an earlier line or call causes a subsequent create carrying the same reference to be absorbed into a modify of that element (§4.3).

Element ids in stored bodies are bare hexadecimal identifiers with no `#`. New authored elements MUST NOT carry an id; only the server mints ids. A `#ref` is the author-facing way to name and later address an element within a session.

### 3.8 Tagged literals

A tagged literal carries both a concrete value and a token binding on the same slot, separated by a colon. This is the self-contained storage form: a stored value never needs re-resolution, yet still records which token it came from.

```
NumericTagged ::= Number ":" "$" TokenPath      (* 8:$spacing.md *)
OpacityTagged ::= Number ":" "$" TokenPath      (* o(0.42:$opacity.40) *)
WeightTagged  ::= WeightWord ":" "$" TokenPath  (* sb:$font.weight.strong *)
FamilyTagged  ::= FontName ":" "$" TokenPath    (* Inter:$font.family *)
```

A separate, comma-delimited tagged form `tok(name,#hex)` carries a resolved color literal alongside its token name; it is the round-trip form for solid colors (§7.2). Note that `tok(...)` uses a comma, whereas the `:` tagged literal uses a colon; they are different constructs.

### 3.9 The four operators

The entire grammar is built on four token shapes and three reused idioms. No other punctuation is meaningful, and new punctuation is never introduced.

| Shape | Role |
|---|---|
| `name(params)` | Records and scalar properties: `p(x,y)`, `w(2)`, `rd(8)`, `pos(o)`, `arc(0,75)`. |
| `name[items]` | Lists and stacks, each item conventionally wrapped in `()`: `f[...]`, `st[...]`, `nodes[...]`, `edges[...]`, `axes[...]`, `spans[...]`. |
| Bare flag | A single boolean: `clip`, `italic`, `comp`, `slot`, `locked`, `hidden`, `abs`, `mask`, `front`, `back`, `frozen`, `wrap`, `off`. |
| Bare keyword | A type token in first position: `r`, `c`, `fr`, `gr`, `text`. |

Reused idioms: `$token` (§3.6), `#ref` (§3.7), `:` tagged literals (§3.8), and the operators `-` `+` `->` which mean remove, append, and swap inside a stack (§7.4, §13) and encode directed edge walks in vector regions (§11).

---

## 4. First-token semantics

The first token of an element line determines the *operation*: create a new element, modify an existing element by id, address an element by session reference, or override a component instance descendant by name. The parser resolves the first token by this precedence.

```
FirstToken ::= HashForm | OverrideForm | HexId | AlphaId | (* fall through to type *)
HashForm   ::= "#" ( HexId | SessionRef )
OverrideForm ::= "override" "(" ( "#" Ref | String ) ")"
HexId      ::= /[0-9a-fA-F]{8,16}/
AlphaId    ::= /[A-Za-z0-9_-]{2,}/       (* must contain a letter, not be a keyword *)
SessionRef ::= /\d+/ | /[A-Za-z_][\w-]*/
```

Precedence:

1. **`#`-prefixed.** Hexadecimal (8 to 16 digits) coerces to a bare id (modify-by-id); otherwise a session reference (§3.7).
2. **`override(...)`.** Records an override target, quote-stripped so `override("Label")` resolves by name and `override(#ref)` by reference. Used to override a descendant of a component instance (§13).
3. **Bare hexadecimal** `/^[0-9a-fA-F]{8,16}$/` is a modify-by-id target.
4. **Bare alphanumeric id.** A token matching `/^[A-Za-z0-9_-]+$/`, of length at least 2, containing at least one letter, and not a reserved keyword, is a modify-by-id target. This is why `btn0001`, `card_1`, and `hero-section` are read as modify targets.
5. **Otherwise** the token is not consumed as a first token; the line is a type-first create line (§5).

The reserved first-token keywords excluded from rule 4 are: `r c fr gr svg rectangle circle frame group text clip no-clip isolate no-isolate italic underline comp component slot locked hidden abs no-abs constrain no-constrain mask`.

NOTE: Rule 4 will read any two-or-more-character word containing a letter that is not a keyword as a modify-by-id target. A line such as `div s(100,100)` is parsed as a modify of an element with id `div`, not as a create; if no such element exists, the recovery in §4.4 reports a not-a-valid-element-type hint. Authors creating a new element MUST let the type token lead.

### 4.1 Create versus modify

A create line has no leading id, session reference, or override target; the builder mints an id and creates an element (§5). A modify line leads with an id, a resolvable `#ref`, or an `override(...)`; the builder applies only the properties present (§15). On a modify line the element type is immutable and no type is defaulted.

### 4.2 Type-token recovery

If the token in the type position is not an element-type token and is not a leading directive (`before(`, `after(`, `replace(`, `clone(`), the parser scans forward, stopping at a quoted name or a `#ref`, for the first element-type token and hoists it into the type slot, with a recovered-element-type note. This rescues a line where a property was written before the type, for example `ds(,theme(dark)) fr ...`.

NOTE: The forward scan does not rescue a *leading* token of function-call shape (`word(...)`) that matches no element type and no directive. Such a leading token is a fatal error, not a defaulted rectangle (§5.11): the parser refuses to absorb an unknown parenthesized head into a phantom element. This is distinct from a *trailing* unrecognized property token, which is non-fatal and dropped (§4.5).

### 4.3 Create absorbed into modify

A create line whose trailing `#ref` (its `assignRef`) already resolves to a live element is absorbed into a *modify* of that element, rather than creating a second element bound to the same reference. Cross-call collisions absorb identically. A placeholder-to-real remap is recorded so that indented children and `before()`/`after()` targets redirect to the absorbed element.

A flat, top-level `override(name-or-#ref)` line is likewise absorbed as a modify-by-name against the whole canvas: exactly one match modifies it; multiple matches error and change nothing, listing the candidate ids; zero matches produce a teachable not-found diagnostic.

### 4.4 Bare-word recovery

If a line resolves to a modify-by-id whose id does not match the hexadecimal pattern (a short alphabetic word such as `el`, `div`, `sp`) and the line has no type, no session reference, and no override target, the parser emits a not-a-valid-element-type hint (or a did-you-mean-`#id` hint when a modify directive is present). If the only produced keys are unrecognized tokens, the line is flagged as prose (§4.6).

### 4.5 Fatal versus non-fatal, and stop-on-error

An error-severity diagnostic marks its line NOT applied and increments the fatal error count; the first fatal error records the stopping row. Lines before the first fatal error are applied. Warnings and infos do not stop application. An unrecognized property token is a non-fatal error: the element is still created with the offending token dropped.

### 4.6 Prose guard

To defend against a chat message or report accidentally pasted into a Blueprint block, the parser inspects the first five non-empty lines. If at least three of them look like prose, the entire block halts with one fatal message and applies nothing (a truthful "0 applied"). A line looks like prose when its first word parses as a non-hexadecimal bare-word id, it has no type, and its only remaining keys are unrecognized tokens. Once the window fills without crossing the threshold, the guard resolves and a later stray prose line is an ordinary per-line diagnostic.

A block whose trimmed content begins with `<svg` or `<?xml` is treated as SVG paste and short-circuits to an empty result; SVG import is handled by a separate pipeline.

---

## 5. Element types

Every create line has a type. The type token is a bare keyword or a parameterized form. The closed set of author-facing types is: rectangle, circle, text, line, frame, group, auto-layout, svg/icon, and vector, plus the parent forms mask and boolean. There are no CSS-style aliases.

```
ElementLine ::= FirstToken? TypeToken? Property*
TypeToken   ::= Rect | Circle | Text | Frame | Group | AutoLayout
              | Svg | Vector | LineElement | Mask | Boolean | Clone | Directive
Rect        ::= "r" | "rectangle" | "r(" Size ")"
Circle      ::= "c" | "circle" | "c(" Size ")" | "arc(" Num "," Num ")"
Frame       ::= "fr" | "frame" | "fr(" Size ")"
Group       ::= "gr" | "group" | "gr(" Size ")"
Size        ::= SizeVal ( "," SizeVal )?
SizeVal     ::= Num | "hug" | "fill" | "fill:" Num | "hug:" Num
```

Type-schema mapping: `r`/`rectangle` build a rectangle; `c`/`circle` build a circle; `fr`/`frame` build a frame; `gr`/`group` build a group (a frame that auto-fits its children); `text`/`t(...)` build text; the rest as named below.

Only `fr`, `gr`, `al(...)`, and the parent forms (mask, boolean, component set, instance) admit children.

### 5.1 Rectangle (`r`)

`r` creates a rectangle. Default size is 100 by 100. Its corner-radius data is present but zero (editable). It carries no fill by default. `r(w,h)` sets the size on the type token.

Example (validated, sovereign mode):

```
r s(120,80) rd(12) f[(#2D6CDF)] "Card"
```

### 5.2 Circle (`c`), arcs and rings

`c` creates a circle (an ellipse filling its box). Default is a solid circle at 100 by 100. Circle geometry is refined by:

- `arc(start,sweep)`: `start` is the start angle in degrees; `sweep` is the swept portion of a full circle as a **percent** in `[0,100]` (not degrees), so `arc(0,75)` is a three-quarter arc and `arc(0,100)` a full circle. As a type token, `arc(start,sweep)` is shorthand for `c arc(start,sweep)` when its first argument is numeric.
- `ratio(N)`: an inner radius ratio in `[0,1]`, making a ring or donut when combined with an arc.
- `cap(start,end)`: end-cap styles for an arc's stroke.

Example (validated, sovereign):

```
c s(80,80) arc(0,75) ratio(0.6) st[(#0080FF,w(8))] "Meter"
```

NOTE: A `sweep` of 100 or more denotes a full circle. In the forgiving authoring dialect a `sweep` above 100 is a fatal error that teaches the percent form, so a degrees-shaped `arc(0,270)` never silently mints a full circle; in canonical storage decode a stored `sweep` above 100 loads as a full circle with a non-fatal info note, so a legacy or hand-authored file never fails to open. `start` is always in degrees.

See [`arcs.md`](reference/arcs.md) for progress rings, donut and pie charts, and activity meters.

### 5.3 Text (`t`)

`t("content",family,size, extras...)` creates a text element. Text is specified in §9. Default text sizing is hug on both axes; the default fill is the token `color.text.primary` unless an explicit `f[]` is present.

### 5.4 Frame (`fr`) and group (`gr`)

`fr` creates a frame: a container that renders its own fill and stroke but not (directly) its children, and that MAY clip its children. Default size 100 by 100, no clip. `gr` creates a group: a frame that auto-fits to the bounds of its children (unless an explicit `s()` clears the auto-fit). Frames and groups admit children by indentation.

### 5.5 Auto layout (`al`)

`al(...)` creates an auto-layout frame that positions its children along a main axis with gaps and padding. Auto layout is specified in §10.

### 5.6 SVG and icons (`svg`)

`svg(...)` creates an svg/vector element from one of several sources, classified by the inner content:

```
Svg      ::= "svg(" SvgArg ")"
SvgArg   ::= InlineSvg | IconRef | UrlRef | PathRef
InlineSvg ::= "<svg" ... | "<?xml" ...
IconRef  ::= "icon:" Name | BareName
UrlRef   ::= "http" ...
PathRef  ::= "/" ... | Name "." Ext
```

- Inline markup (`<svg` or `<?xml`) is stored as inline SVG content.
- A bundled **icon** is referenced by `icon:name`, kebab-cased. A bare name with no `icon:`, `http`, `/`, or `.` is auto-prefixed with `icon:` and an info note. Regular and fill weights are bundled (`house`, `house-fill`); the Phosphor weight suffixes `-bold`, `-light`, `-thin`, `-duotone` fall back to the regular weight with a note. An unresolved name falls back to a question-mark glyph with a warning.
- A URL (`http...`) or a local path is stored as a file reference.

The icon contract is name-based: the imported vector's geometry comes from the resolved SVG, but the element records the icon *name* as its provenance, so a read/authoring emission can round-trip to the compact `svg(icon:name)` form. Size comes from `s(w,h)` (contain/fit when both are present); `f[...]` and `st[...]` on the svg line are recolor replacements applied wherever the source has a fill or stroke.

NOTE: SVG path data mistakenly passed as `svg("M...")`, or hallucinated size parameters as in `svg(icon:name,w(20))` or `svg(cloud,32)`, produce a diagnostic pointing to `v(nodes[...])` or to a following `s(w,h)`; the recognizable icon or path part is still stored.

Example (validated, sovereign):

```
svg(icon:wifi-high) s(24,24) f[(#0080FF)]
```

### 5.7 Vector (`v`)

`v(...)` creates a vector element as a planar graph of nodes, edges, and regions. Vectors are specified in §11.

### 5.8 Line (`line`)

`line(...)` is the only straight-line primitive; it builds a two-node vector element. It has two argument shapes (vector form and endpoint/connector form) and is specified in §12.

### 5.9 Clone (`clone`)

`clone(id)` creates a copy of an existing element. It appears as a type token (`clone(id)` sets the clone source) or as a property. Clone is a container-admitting form.

### 5.10 Mask and boolean parents

`mask` / `mask(kind)` and `bool(op)` create parent frames whose children are combined geometrically. They are specified in §14.

### 5.11 Default type

A create line with no type token and no leading id, session reference, or override target defaults to a **rectangle**, with the defaulted-type flag set. This is why a bare `s(20,20) rd(10) f[(#FFF)] "Knob"` creates a rectangle. On a modify line no type is defaulted; the type comes from the existing element.

### 5.12 Directive type tokens

Several first-position forms are directives rather than element definitions. They produce non-element operations and do not take an id or push the nesting stack. See §16 for `delete(id)`, `undo("label")`, `redo("label")`, `ungroup(id)`, and the chaining directives `replace(id)`, `before(id)`, `after(id)`. The removed `connect(...)` form is recognized only to emit a migration error pointing to `line(from(...),to(...))`.

### 5.13 Leaf absorption

When a create-time `r` (always), or a `c` with default circle data (no arc, ring, pie, or caps), acquires a child by indentation, it is converted into a plain container frame instead of erroring. The stack type is flipped to frame, one info diagnostic is emitted, and for the circle case a clip and a full corner radius are applied. Modify lines are excluded from absorption.

### 5.14 Defaults summary

The following table gives the default geometry, fill, and notable properties for a bare create line of each type. All values are the `bp:v1` frozen defaults (Appendix B). Position defaults to `(0,0)` as flat keys; top-level elements are auto-placed beside existing work by the create dispatcher.

| Type | Default size | Default fill | Notable defaults |
|---|---|---|---|
| `r` rectangle | 100 by 100 | none | `RectangleData` present, radius 0 |
| `c` circle | 100 by 100 | none | solid circle |
| `t` text | hug by hug | `color.text.primary` unless `f[]` | size 24, family from DS default, weight regular |
| `fr` frame | 100 by 100 | none | clip false, corner radius 0 |
| `gr` group | auto-fit to children | none | auto-fit unless explicit `s()` |
| `al(...)` | 100 by 100 unless `s()` | none | padding and gap 0 unless set |
| `svg` | from import (or hallucinated size) | imported fills | provenance icon name |
| `line`/`v` | from geometry | none (open path needs a stroke) | n/a |

Opacity default 1.0, blend mode normal, rotation 0, flip false/false, no effects.

---

## 6. Property reference

Properties follow the type token in any order and are matched by a linear scan over their leading token. The pervasive rule for modify lines is *empty positional equals skip*: an empty argument slot preserves the existing value (§15). An unrecognized property token is a non-fatal error (§4.5): it is dropped with a rich hint and the element is still created.

The table lists every property token, its grammar, the map key(s) it emits, and its create default and modify behavior. Sections 7 through 11 expand paint, effects, text, layout, and vectors.

| Token | Grammar | Emits | Create default | Modify |
|---|---|---|---|---|
| `p(x,y)` | each `Num` or `c` (center) | `x`,`y` or `centerX`,`centerY` | auto-placed | per-axis skip on empty |
| `s(w,h)` | `SizeVal` (§5) | `width`/`height` + sizing + `flex` | none | per-axis skip; bare `s()` on a line is fatal |
| `min(w,h)` / `max(w,h)` | numbers, plus `minw`/`minh`/... aliases | min/max constraints | unconstrained | empty preserves; bare `min()`/`max()` clears |
| `rd(...)` | `rd(N)` or `rd(TL,TR,BR,BL)`, optional `smooth(N)` | `cornerRadius`, `cornerSmoothing` | rectangle radius 0 | empty `rd()` skips |
| `rd+(n)` | `Num` | `deltaCornerRadius` | n/a | relative delta (modify only) |
| `rot(n)` | one `Num` | `rotation` | 0 | empty skips |
| `rot+(n)` | `Num` | `deltaRotation` | n/a | delta |
| `o(n)` | `0..1`, `$tok`, or `N:$tok` | `opacity`, `opacityToken` | 1.0 | empty skips |
| `o+(n)` | `Num` | `deltaOpacity` | n/a | delta |
| `p+(dx,dy)`, `s+(dw,dh)` | numbers | `deltaX/Y`, `deltaWidth/Height` | n/a | delta |
| `skew(x,y)` | two numbers, all-or-nothing | `skewX`,`skewY` | 0,0 | side effect |
| `flip(h,v)` | two 0/1, all-or-nothing | `flipped` | false,false | side effect |
| `blend(mode)` | mode name | `blendMode` | normal | replaces |
| `clip` / `no-clip` | bare | `clipContent` | false | flag |
| `isolate` / `no-isolate` | bare | isolate | false | flag |
| `italic` | bare | italic | false | flag |
| `underline` | bare | underline | false | flag |
| `comp` / `component` | bare | `isComponent` | n/a | n/a |
| `slot` | bare | `isSlot` | n/a | n/a |
| `locked` | bare | locked | false | flag |
| `hidden` | bare | `visible:false` | visible | flag |
| `constrain` / `no-constrain` | bare | `constrainProportions` | false | flag |
| `abs` / `no-abs` | bare | `isAbsolutePosition` | false | flag |
| `pin(H,V)` | `H` in `l\|r\|c\|lr\|scale`, `V` in `t\|b\|c\|tb\|scale` | `hPin`,`vPin` | `l,t` (min/min) | per-axis skip on empty |
| `front` / `front(N)`, `back` / `back(N)` | bare or N steps | z-order | n/a | reorder |
| `tsm(mode)` | `auto`/`autoHeight`/`autoWidth`/`fixed` | text sizing mode | n/a | n/a |
| `scaleTo(w\|h,N)`, `sw(N)`, `sh(N)` | axis + number | scale-to dimension | n/a | n/a |
| `ds(...)` | §16 note | per-element design system | inherit | override |
| `path:d(...)` | SVG path data | `path` | n/a | vector contour |
| `f[...]` | §7 | fills | type-dependent | array replaces / operators mutate |
| `st[...]` | §7 | strokes | none | replaces / mutates |
| `shadow(...)`, `outerglow(...)`, `eblur(...)` | §8 | effects | none | accumulate |
| `grid(...)` | layout grid | frame grids | none | append |
| `arc(start,sweep)` | start (deg), sweep (percent 0 to 100) | `arcStart`,`arcSweep` | solid circle | n/a |
| `ratio(n)` | `0..1` | `arcRatio` | none | n/a |
| `cap(s,e)` | short cap names | `capStart`,`capEnd` | none | per-position skip |
| `before(id)` / `after(id)` / `parent(id)` | ref | sibling/parent placement | n/a | reorder / reparent |
| `clone(id)` / `replace(id)` | ref | clone source / replace target | n/a | n/a |
| `#ref` (trailing) | ref | `assignRef` | n/a | assign ref |
| `"name"` (trailing) | quoted | `name` | generated | replaces |

Storage-dialect carriers `obb(...)`, `ext(...)`, `hug:N`, component tokens `axes[...]`/`props[...]`/`variant(...)`/`at(...)`/`mref(...)`/`ov[...]`, and the vector continuation `vr(...)` are covered in their respective sections (§13, §11, §16.2).

NOTE: There is no fixed-size memo. An element's one size is its geometry, carried by `s(w,h)` (and, at storage, by `ext(...)` and `hug:N`). The retired `fixed(w,h)` token is accepted and ignored on read for backward compatibility, and is never emitted.

NOTE (implementation): The `underline` bare flag emits a map key that differs from the `underlined` key produced inside `t(...)`. A bare `underline` written after a `t(...)` on the same line may not take effect. Prefer specifying underline inside `t(...)` (§9). This asymmetry is tracked as a defect and does not change the normative meaning of underline.

### 6.1 Pin constraints

`pin(H,V)` records how a child holds its place when its container is resized. It is a child property that lives beside `abs` in the child's property list and carries onto the child's `LayoutBehavior` (`hPin`, `vPin`).

```
Pin   ::= "pin(" HPin? "," VPin? ")"
HPin  ::= "l" | "r" | "c" | "lr" | "scale"
VPin  ::= "t" | "b" | "c" | "tb" | "scale"
```

Each axis carries one of five kinds. The horizontal letters are `l` (min), `r` (max), `c` (center), `lr` (stretch), and `scale`; the vertical letters are `t` (min), `b` (max), `c` (center), `tb` (stretch), and `scale`. The kinds determine the container-resize semantics:

- **min** (`l` / `t`): keep the leading offset (the child's distance from the container's left or top edge).
- **max** (`r` / `b`): keep the trailing offset (the distance from the right or bottom edge).
- **center** (`c`): keep the child center's offset from the container center.
- **stretch** (`lr` / `tb`): keep both edge offsets, so the child's size changes with the container.
- **scale**: interpolate both edges proportionally to the container's size change.

**Applicability.** Pins apply to a child whose container's box does not derive from its children: a frame, component, instance, or non-hug group. A hug group's box follows its children, so its children's pins are dormant until the group is resized (which makes it fixed and activates them). An auto-layout child has no pins because the flow owns its position, with one exception: an absolute-positioned child (`abs`) pins within the auto-layout frame. A top-level element has no container to resize against and therefore no pins.

**Default and emit-lean.** The absent form is `pin(l,t)` (min on both axes), which reproduces the pre-pin anchoring behavior. Because `l,t` is the frozen default (Appendix B), it is elided: a child at the default never serializes a `pin(...)` token, and a document that uses no non-default pin is byte-identical to one written before pins existed. On import, a child's pins map verbatim from the source's constraints (Figma `LEFT_RIGHT` to stretch, `SCALE` to scale, `CENTER` to center, `RIGHT`/`BOTTOM` to max); import-size geometry is unchanged, because pins take effect only on a later resize.

**Modify.** `pin(...)` follows the *empty positional equals skip* rule per axis (§15.1): `pin(,c)` changes only the vertical pin and preserves the horizontal one.

**Misuse.** In the authoring lanes (create and modify), a `pin(...)` on an auto-layout child that is not `abs`, or on a group child (a group hugs its children, so its box has nothing to pin against), is a named non-fatal diagnostic in the class of §4.5: the pin token is dropped, the rest of the line still applies, and the message teaches the fix (add `abs`, or use a frame as the container), mirroring the forgiving treatment of `lh(...)` outside `t(...)`. Loading a stored document is exempt from this check: a dormant pin (on a hug-group child, or constraints imported onto a flow child) is legitimate state that activates later (a hug group resized to fixed, a flow child made `abs`), so it round-trips losslessly.

---

## 7. Paint: fills and strokes

Paint is expressed as two stacks: a fill stack `f[...]` and a stroke stack `st[...]`. Each is a list of paint items composited in order. The stacks share the same array machinery and the same edit operators (§7.4).

### 7.1 Fill stack

```
Fills        ::= "f[" ( FillItems | "" ) "]"
FillItems    ::= FillItem ( "," FillItem )* | FillOp+
FillItem     ::= "(" FillSpec ExtraWrap* ")" | FillSpec
FillOp       ::= "+(" FillSpec ")" | "-(" FillSpec ")" | "(" FillSpec ")->(" FillSpec ")"
ExtraWrap    ::= "off" | "blend(" Mode ")"
```

A fill item MAY carry a trailing `off` (disable, preserving the item for round-trip) and a per-fill `blend(mode)`. A leading bare identifier that is not itself a fill spec is a fill id; an id-only item with no spec is dropped.

An explicit empty `f[]` sets the explicit-empty marker, which suppresses re-injection of a type default fill. This matters only for text, which is the only element type with a default fill to suppress; for shapes there is no default fill, so `f[]` and an absent `f[...]` are equivalent.

### 7.2 Fill kinds

A fill spec is dispatched by shape:

- **Solid.** `$token` binds a color token. `tok(name,#hex)` carries a token name and a resolved hex (the round-trip form). `#hex` is a bare color. `solid(color[,o(...)])` is the explicit form, where the color is `$tok`, `tok(...)`, or `#hex`, and opacity is `o($tok)`, `o(N:$tok)`, or `o(N)`.
- **Gradients.** `linear(...)`, `radial(...)`, `angular(...)`. Each has a default shorthand, positional forms, and named-percentage forms, with stops `stop(color,pos[,o(...)])` or bare colors and tokens. Linear defaults to 180 degrees, black to white; radial to a centered white-to-black; angular to a centered black-to-white. Per-stop opacity below 1.0 is baked into an eight-digit hex unless an opacity token owns it. See [`gradients`](reference/paint.md).
- **Image.** `img(source[,scaleMode][,crop(...)][,o(N)][,scale(N)])`. The source routes by shape: an `http(s)://` URL downloads; an `Assets/...` path references an existing asset; a slash path or a dotted name is a file path; otherwise a cache id.
- **Shaders.** `metaballs`, `metal`/`liquidMetal`, `irid`/`holo`, `steel`, and related keywords, with bare defaults per type, token-bindable colors, a `frozen` flag to disable animation, and reserved named parameters (`scale`, `uvx`, `uvy`, `uvrot`, `opacity`, `aspect`, `shape(...)`).
- **Glass.** `glass` / `glass(...)` is a liquid-glass fill with clamped parameters (`frost`, `thickness`, `bevel`, `ior`, `chroma`, `glow`, `edge`, `sat`, `angle`, `gather`, `mag`, `tint`). Glass is fill-only; a glass spec in a stroke is a diagnostic (Appendix C, B307).
- **Effect fills.** `inner(...)` (inner shadow), `glow(...)` (inner glow), and `blur(...)` (background blur, default radius 8) are *fills*, not element-level effects. They live in the fill stack.
- **Image filters.** A `filter(name,...)` wrapper and the named filters `colorAdjust`, `noiseGrain`, `halftone`, `pixelate`, `duotone`, `posterize`, `dither`, plus the shader filters `dithering` and `reactiveGrid`, are matched case-insensitively and forgivingly.

An unrecognized spec falls back to a solid color built from the spec string; the validator catches the common mistake of a keyword-with-parentheses that is not a known fill kind (B305).

Example (validated, sovereign): a solid over a glass overlay:

```
r s(200,120) rd(16) f[(#101828),(glass(chroma(0.5),frost(8)))]
```

### 7.3 Stroke stack

```
Strokes    ::= "st[" StrokeItem ( "," StrokeItem )* "]"
StrokeItem ::= "(" PaintSpec StrokeParam* ")"
StrokeParam ::= "w(" Num ")" | "w(" Num "," Num "," Num "," Num ")"
             | "pos(" ( "c" | "i" | "o" ) ")" | "join(" ( "m" | "r" | "b" ) ")"
             | "dash(" Num ( "," Num )* ")" | "dashcap(" Cap ")" | "miter(" Num ")"
             | "blend(" Mode ")" | "off"
```

A stroke's paint spec reuses the fill spec grammar (a `$token`, a color, or a complex fill kind). `w(N)` is a uniform thickness (token-bindable via `1:$stroke.width.md`); `w(t,r,b,l)` gives per-side weights, with the uniform thickness taken as the maximum side. `pos(c|i|o)` selects center, inside, or outside. `join(m|r|b)` selects miter, round, or bevel. `dash(...)` sets a dash pattern; `dashcap(...)` its cap; `miter(N)` the miter limit. Default thickness is 1.0.

NOTE: `cap(...)` inside a stroke is silently ignored; stroke caps are per-node or per-circle, not per-stroke. The element-level `cap(start,end)` (§5.2) is the caps mechanism.

### 7.4 Stack edit operators

Inside `f[...]` and `st[...]`, three operators mutate an existing stack in place on a modify line:

- `+(spec)` appends an item.
- `-(spec)` removes a matching item. A leading `-` on a numeric literal (`-2`) is not a remove; it stays a plain negative number.
- `old->new` swaps or renames an item.

A bracket that mixes bare items with operator items is an error (B708). An all-bare bracket replaces the whole stack (the array form of §7.1); an all-operator bracket mutates in place.

Example (validated, sovereign): append a fill to an existing element referenced `#card`:

```
r s(100,60) f[(#101828)] #card
#card f[+(#0080FF)]
```

### 7.5 Explicit empty

A bare `f[]` (empty brackets) is distinct from an absent `f[...]`. The empty form serializes a fill-less element and suppresses text's default fill re-injection; the absent form lets type defaults apply. This is the mechanism that round-trips outline-only text and transparent frames.

---

## 8. Element-level effects

Element-level effects are bare tokens (not inside `f[...]`) that accumulate into the element's effect list: drop shadow, outer glow, and element (layer) blur. Inner shadow, inner glow, and background blur are *fills* (§7.2), not element-level effects.

```
Effect     ::= Shadow | OuterGlow | ElementBlur
Shadow     ::= "shadow(" ShadowArg* ")"
OuterGlow  ::= "outerglow(" GlowArg* ")"
ElementBlur ::= "eblur(" Num? BlurArg* ")"
```

- `shadow(...)`: a positional color (`#hex`, `tok(...)`, or `$tok`), and named `o(N[:$tok])`, `x(N)`, `y(N)`, `blur(N)`, `sp(N)`, `blend(mode)`, `off`, `behind`, `density(N)`. A bare `shadow()` defaults to a shadow token color at opacity 0.25, offset y 4, blur 8. A single-argument `shadow($shadow.X)` in the `shadow.` token namespace binds a composite shadow token rather than building a drop shadow.
- `outerglow(...)`: the same parameter set without offset. Bare defaults: a glow token color, opacity 0.6, blur 8, screen blend.
- `eblur(...)`: a positional radius (default 4), plus `o(N[:$tok])`, `blend(mode)`, `off`.

A disabled effect (`off`) is preserved (its enabled flag is false) so it round-trips. The CSS four-argument `sh(x,y,blur,#hex)` form is preprocessed into `shadow(...)` (Appendix D).

Example (validated, sovereign):

```
r s(160,100) rd(12) f[(#FFFFFF)] shadow(#000000,o(0.2),y(6),blur(16))
```

See [`effects.md`](reference/effects.md).

---

## 9. Text

### 9.1 Grammar

```
TextType   ::= "t(" Content ( "," Family )? ( "," SizeArg )? ( "," Extra )* ")"
Content    ::= String | BareText          (* empty first slot = skip/preserve *)
Family     ::= FontName | "$" TokenPath | FontName ":$" TokenPath
SizeArg    ::= Num | "$" TokenPath | Num ":$" TokenPath | "tok(" TokenPath "," Num ")"
Extra      ::= Weight | "align(" A ")" | "valign(" V ")" | "lh(" LH ")" | "ls(" LS ")"
             | "ps(" Num ")" | "pi(" Num ")" | "li(" ListSpec ")" | "lsp(" Num ")"
             | "rtl" | "ltr" | "italic" | "underline" | "strike" | "case(" C ")"
             | "clamp(" Num ")" | "feat(" Feat ")" | "typo(" "$" TokenPath ")"
             | "$" TokenPath | WeightWord ":$" TokenPath
LH         ::= Num | Num ":$" TokenPath | "$" TokenPath | "auto" "," Num
```

Positions 1 (content), 2 (family), and 3 (size) are positional; everything from position 3 onward is order-independent. Arguments are split on commas, respecting nesting.

### 9.2 Content

An empty first slot skips (preserves on modify; on create it is an error, B204). A quoted `"..."` is unquoted and unescaped; a bare, unquoted value is literal verbatim. Whitespace is literal content, so `t(" ")` and `t( ,Inter)` set a single space. An explicit `t("")` (empty quotes) is rejected (B203); the application auto-removes empty text.

### 9.3 Family, size, weight, and extras

- **Family.** Empty skips. `$font.family` or a typography token binds a font token; `Inter:$font.family` is the tagged form. A bare family name is stored verbatim; a look-alike map (Helvetica, Arial, and other common system faces) does not remap the value but emits a warning naming the bundled face the author could pin instead. The retired platform sentinels (`System`, `System Font`, and an empty family) still migrate to the bundled default. During a canonical build the family is identity.
- **Size.** Empty skips. A token binds a size token; otherwise a number (default 24).
- **Weight and extras.** `align(l|c|r|j)`, `valign(t|c|b)`, `lh(N)` line height (a multiplier in `0..3`; values above 3 are read as pixels and divided by the font size), `ls(N)` letter spacing (px, `Npx`, or `Nem`), `ps(N)` paragraph spacing, `pi(N)` paragraph indent, `lsp(N)` list spacing, `li(...)` list types, `rtl`/`ltr` direction, `italic`, `underline`, `strike`, `case(upper|lower|title|none)`, `clamp(N)` max lines, `feat(...)` font features, `typo($typography.X)` a typography token. A positional `$token` after size is a weight token; `sb:$font.weight.strong` is the tagged form; any other bare value is a weight word (`r`, `m`, `sb`, `b`, `eb`, `bl`).
- **Auto line height.** `lh(auto,X)` is the auto-plus-carrier form: the line height is automatic — it tracks the font, re-deriving when the font changes during an edit — and `X` is the resolved multiplier every conforming host renders with, so the same document renders identically everywhere. Canonical storage always carries this form for a text whose line height the author never set. `lh(N)` stays author-explicit and is never re-derived; a token-bound line height always emits the explicit tagged form. A bare `lh(auto)` is a forgiving read: treated as an unset slot (the line height resolves from the font's own metrics), with a diagnostic teaching the two-slot form.

### 9.4 Styled runs (`spans[...]`)

A `spans[...]` continuation line, indented under a text element, applies per-run styling. Each range is either a substring match (`"substring"[,occurrence],mods`) or a numeric range (`start,end,mods`). Mods include `b`/`i`/`u`/`strike`, `ls(N)`, `w(...)`, `s(N)`, `f(...)`, `o(N|$tok)`, `#hex`, `$tok`, and `tok(name,#hex)`.

Example (validated, sovereign):

```
t("Total: $42.00",Inter,16)
  spans["$42.00",b,#0080FF]
```

See [`text.md`](reference/text.md) and [`styled-ranges.md`](reference/styled-ranges.md).

---

## 10. Auto layout

### 10.1 Grammar

```
AutoLayout ::= "al(" ( Dir )? AlArg* ")"
Dir        ::= "h" | "v"
AlArg      ::= "x(" Align ")" | "y(" Align ")" | "g(" Gap ")" | "pad(" Padding ")" | "wrap"
Align      ::= "s" | "c" | "e" | "sb"
Gap        ::= Num | "$" TokenPath | Num ":$" TokenPath | Num "," Num
Padding    ::= Num | Num "," Num | Num "," Num "," Num "," Num
```

`al(...)` sets an auto-layout frame. Direction is `h` or `v` (default vertical). `wrap` enables wrapping. `x()` and `y()` are physical-axis alignments mapped to main and cross axes by direction: for `al(h)`, `x` is main and `y` is cross; for `al(v)`, `y` is main and `x` is cross. Values are start, center, end, and space-between (`sb`). On a modify of an existing auto-layout frame that does not restate the direction, `x()` and `y()` are re-mapped to main and cross against the frame's own existing direction at build time, so an alignment-only edit is correct regardless of that direction.

`g(N)` is item spacing (token-bindable, or `g(main,cross)` for a two-axis gap). `pad(N)` is uniform padding; `pad(v,h)` and `pad(t,r,b,l)` are the multi-value forms, each all-or-nothing per arity. Empty `pad()` skips.

### 10.2 Child sizing

A child's `s()` accepts `hug`, `fill`, `fill:N`, or a number; `fill:N` sets a flex factor of N (when not 1.0).

### 10.3 Smart-sizing policy

NOTE (dispatch policy, not grammar): After parsing, the create dispatcher applies a smart-sizing default to a non-absolute, non-auto-layout child that lands inside an auto-layout parent and has no explicit sizing on the relevant axis:

- A vertical parent gives a non-text child `fill` on the cross (width) axis.
- A horizontal parent gives a non-text child `fill` on the cross (height) axis.
- Text stays hug on both axes; to make text wrap, set `s(fill,hug)` explicitly.
- A nested auto-layout child keeps its own sizing.

This policy is applied by the dispatcher, not by the grammar or the builder. It is documented here so authors understand why a child without explicit sizing fills its cross axis.

Example (validated, sovereign): a vertical stack with padding and gap, one filling child:

```
al(v,g(12),pad(16)) s(240,hug) f[(#0F172A)] "Stack"
  t("Title",Inter,18,b) f[(#FFFFFF)]
  r s(fill,80) rd(8) f[(#1E293B)]
```

See [`layout.md`](reference/layout.md) and [`layout-patterns.md`](reference/layout-patterns.md).

---

## 11. Vectors

A vector element is a planar graph, not a path string. It carries nodes (shared, of arbitrary valence), edges (straight or cubic bezier, with per-endpoint handles), and regions (closed faces that take fills, with explicit hole flags). This graph model is what makes a vector round-trip exactly.

### 11.1 Data model

- A **node** has an integer id, a point, an optional point type (`st` straight, `mi` mirrored, `as` asymmetric, `di` disconnected; absent means untyped), and an optional per-node stroke cap rendered at degree-1 (leaf) nodes.
- An **edge** has an id and two node ids, and optional handles for each endpoint. A null handle is straight at that endpoint; a non-null handle equal to the zero offset is an *explicit* control point coincident with the node, distinct from a null handle.
- A **region** has a canonical id (its sorted edge ids joined with colons, e.g. `0:1:2`), a boundary walk as directed `(edgeId, direction)` pairs, a winding flag, a hole flag, and a list of fill ids. Regions are ordered by absolute area descending (larger regions render underneath). Their 1-based indices in that order are the short ids `r1`, `r2`, and so on.

### 11.2 Grammar

```
VectorType ::= "v(" VecArg ( "," VecArg )* ")"
VecArg     ::= IconAnnot | Nodes | Edges | Regions | "closed"
IconAnnot  ::= "icon:" IconName                       (* storage annotation *)
Nodes      ::= "nodes[" NodeTuple ( "," NodeTuple )* "]"
NodeTuple  ::= "(" Int "," Num "," Num ( "," NodeType ( "," Cap )? )? ")"
NodeType   ::= "st" | "mi" | "as" | "di" | ""         (* "" = untyped *)
Cap        ::= "r" | "n" | "sq" | "ar" | "c"          (* per-node, 5th slot *)
Edges      ::= "edges[" EdgeTuple ( "," EdgeTuple )* "]"
EdgeTuple  ::= "(" Int "," Int "," Int
               ( "," Num "," Num "," Num "," Num ( "," ZeroFlags )? )? ")"
ZeroFlags  ::= /[ab]+/                                 (* 8th slot, storage-only *)
Regions    ::= "regions[" ( RegionSpec ( "," RegionSpec )* )? "]"   (* MAY be empty *)
RegionSpec ::= "(" EdgeWalk ( "," "hole" )? ")"
EdgeWalk   ::= ( Int Sign )+                           (* e.g. 0+1-2+ *)
Sign       ::= "+" | "-"
```

Nodes require at least the id, x, and y slots. The fourth slot is the point type (empty means untyped); the fifth is the per-node cap. Edges require id and two node ids; slots 4 through 7 are the two handles; the eighth is the explicit-zero-handle flag set (`a`, `b`, or `ab`), a storage-only marker.

### 11.3 Region states

`regions[...]` has three distinct states:

- **Absent** (no `regions` argument): regions are recomputed from the graph and holes inferred by containment.
- **Empty** (`regions[]`): the vector is an *open network* with no regions.
- **Populated**: the listed edge walks are the boundary truth; each `hole` marker flags a hole.

When explicit regions are present, edges MUST be given explicitly (an omitted-edges vector cannot reference edge ids that were never declared).

### 11.4 Build routes

A vector arms one of two build routes:

- **Faithful graph route** (canonical storage form): armed when nodes and edges are both non-empty and the auto-smooth heuristic does not fire. Node ids, point types, per-node caps, exact handles, arbitrary valence, and explicit regions all survive.
- **Authoring convenience route**: armed when the graph has smooth nodes, no handle data, and no explicit regions (the auto-smooth heuristic). The element builds via a legacy SVG-contour path, and node identity and point type are re-inferred.

NOTE: An authored auto-smooth curve is not identity-stable until it round-trips through canonical storage, where the faithful graph form is always used.

In all cases the element also carries an SVG path derived from the graph so that validators and legacy consumers can read a path.

### 11.5 The terse contour form (`path:d(...)`)

`path:d(<svg>)` sets a vector's contour directly from an SVG path string. It arms the SVG-contour build route: faces are detected, holes inferred by containment, and element fills applied to non-hole faces. It can express a single winding contour with straight or bezier segments; it cannot express node identity, point type, per-node caps beyond the endpoints, valence-above-2 networks, dangling edges, explicit holes, multi-region fills, or explicit-zero handles. Those are exactly the conditions that force the faithful graph form. The encoder emits `v() path:d(...)` for simple contours on read/authoring surfaces only; canonical storage never uses it.

NOTE: The terse token is `path:d(...)`. There is no bare `path(...)` token.

### 11.6 Region fills (`vr(...)`)

```
VrLine   ::= "vr(" RegionId ( "," PathData )? ")" ( "f[" Fill* "]" )? "hole"?
RegionId ::= "r" Int
```

A `vr(...)` continuation line, indented under a vector, sets a region's fills (and optionally its hole flag). The region is addressed by short id `rN`. On a modify, path data is ignored (geometry comes from the existing element); on create with an explicit graph, `rN` is a 1-based index into the explicit `regions[...]` order; on the heuristic route, `rN` indexes the area-descending region list.

NOTE: The short id `rN` denotes two different orderings depending on route: the explicit storage order on the graph route, and the area-descending order on the heuristic route. When authoring against an explicit `regions[...]` list, `rN` follows the list order.

### 11.7 Per-node caps

A node's cap is one of `r` (round, the default), `n` (none), `sq` (square), `ar` (arrow), or `c` (circle), rendered at leaf nodes. In canonical mode an explicit `round` cap is emitted (distinct from an absent cap). The element-level `cap(start,end)` token lands caps on the first and last leaf nodes.

### 11.8 Booleans and masks

Booleans and masks are parent types (frames), not vector-internal operators (§14). There is no vector-internal boolean grammar; a boolean or mask parent combines its child elements at render time.

### 11.9 Worked examples

Authoring, an auto-smooth stroke curve through three mirrored nodes (validated, sovereign):

```
v(nodes[(0,0,40,mi),(1,60,16,mi),(2,120,12,mi)]) s(120,48) st[(#14B8A6,w(2))]
```

Authoring, a closed square with an inner square hole (validated, sovereign):

```
v(nodes[(0,0,0,st),(1,100,0,st),(2,100,100,st),(3,0,100,st),(4,30,30,st),(5,70,30,st),(6,70,70,st),(7,30,70,st)],edges[(0,0,1),(1,1,2),(2,2,3),(3,3,0),(4,4,5),(5,5,6),(6,6,7),(7,7,4)],regions[(0+1+2+3+),(4+5+6+7+,hole)]) s(100,100) f[(#111827)]
  vr(r1) f[(#111827)]
```

Canonical storage form (NOT-AUTHORABLE as produced; the parser reads it): a rotated bezier network with an explicit-zero-handle flag and a per-node arrow cap:

```
{id} v(nodes[(0,0,0,mi,ar),(1,50,20,as),(2,100,0,mi)],edges[(0,0,1,10,0,-10,0),(1,1,2,0,0,0,0,ab)],regions[]) p(200,120) s(100,20) rot(30) st[($ink,w(1.5))]
```

See [`vectors.md`](reference/vectors.md).

---

## 12. Lines and connectors

`line(...)` is the only straight-line primitive; it builds a vector element. It has two argument shapes.

### 12.1 Grammar

```
LineElement ::= "line(" ( VectorForm | EndpointForm ) ")"
VectorForm  ::= Len ( "," Angle )? | Angle "," Len
Len         ::= "len(" Num ")"
Angle       ::= "angle(" Num ")"                       (* degrees *)
EndpointForm ::= From "," To ( "," ConnOpt )*
From        ::= "from(" Endpoint ")"
To          ::= "to(" Endpoint ")"
Endpoint    ::= Num "," Num | "#" Ref ( "," Side ( "," Frac )? )?
Side        ::= "tl"|"t"|"tr"|"l"|"c"|"r"|"bl"|"b"|"br"
Frac        ::= Num                                    (* in [0,1] *)
ConnOpt     ::= "route(" RouteVal ")" | "avoid(" AvoidVal ")" | "intent(" IntentVal ")"
RouteVal    ::= "straight" | "elbow" | "elbow2" | "bezier"
AvoidVal    ::= "none" | "endpoints" | "all"
IntentVal   ::= "dependency" | "flow" | "annotation"
```

### 12.2 Vector form

The vector form builds a horizontal placeholder path of the given length. `angle(deg)` rotates the line around its start point (the `p()` point), and the rotation is baked into the geometry at parse time (the stored element carries no residual rotation). A negative length is equivalent to a 180-degree angle. `line(len(N))` requires a length; `angle()` without `len()`, `len(0)`, and a bare `line(N)` are errors.

### 12.3 Endpoint (connector) form

The endpoint form connects two anchors. An endpoint is a literal coordinate, a `#ref` to an element or instance child, or a `#ref` with a nine-cell side (`t`, `br`, `c`, ...) and an optional fractional position along an edge. Options are `route(...)` (straight, elbow, single-bend; elbow2, two-bend; bezier), `avoid(...)` (none, endpoints, all), and `intent(...)` (dependency, flow, annotation, which supply route, avoidance, cap, color, and width presets).

Connectors are resolved at end-of-block, once all `from`/`to` references are known. A missing reference soft-fails, leaving a placeholder with a warning. Anchor sides are auto-chosen from the relative geometry unless given explicitly. A bezier route is sampled into 16 polyline segments (a connector stores a polyline, not cubic handles). Arrowheads are stroke caps (`ar`), not separate geometry. A top-level reference-to-reference line with no explicit parent auto-reparents to the lowest common ancestor of its endpoints.

### 12.4 Constraints

`s(W,H)` on a line is rejected (length lives inside `line()`); `rot()` on a line is rejected (use `angle()`). Mixing the vector and endpoint forms, omitting one endpoint, and an empty `line()` are errors. A line has an open path and therefore needs a stroke; the strokeless case is rejected at the validation phase.

NOTE: The removed `connect(...)` form is recognized only to emit a migration error to `line(from(...),to(...))`.

### 12.5 Worked examples

A tilted line (validated, sovereign):

```
line(len(120),angle(45)) st[(#94A3B8,w(1))]
```

A reference-to-reference connector with an elbow route and arrow cap (validated, sovereign, with two anchors present):

```
r s(60,40) f[(#0080FF)] #a
r s(60,40) p(200,120) f[(#FF3377)] #b
line(from(#a),to(#b),route(elbow),avoid(all)) cap(n,ar) st[(#111827,w(2))]
```

See [`lines.md`](reference/lines.md), which also covers connectors.

---

## 13. Components

A component is a reusable master; an instance is a placed copy that follows its master and MAY carry overrides. A component **set** groups variants along named axes. The component surface uses four element fields: component data (master or set root), variant data (a set member), instance data (instance root), and child-component data (a descendant of a master or an instance).

### 13.1 Data model

- A property definition (axis) has a stable key, a display name, and a list of values; each value has a stable key and a display label.
- Component data carries the list of property definitions; a degenerate single master has an empty list.
- Variant data carries a coordinate mapping axis key to value key; a bare variant has an empty coordinate.
- Instance data carries the master's element id (`componentRef`), an optional cross-canvas path, a configuration mapping axis key to value key, and an optional set of root override categories.
- Child-component data carries the master child's element id (`elementRef`), an optional set of auto-tracked overridden category names, and an is-slot flag (when true, sync skips the subtree).

Keys are stable; names and labels are display-only and renamable with no re-keying, for same-canvas data. The exception is cross-canvas instances, whose configuration is stored by *name* because names are the durable cross-file identity.

### 13.2 Grammar

```
CompDecl  ::= "comp" | "component"
AxesDecl  ::= ( "axes[" | "props[" ) Axis ( Sep Axis )* "]"
Axis      ::= AxisOp? AxisName ( "[" Values? "]" )?
AxisOp    ::= "+" | "-" | AxisName "->" NewName
Values    ::= Value ( Sep Value )*
Value     ::= ValueOp? Label
ValueOp   ::= "+" | "-" | Label "->" NewLabel
Sep       ::= "," | " "                    (* axes tolerate spaces between them *)
Variant   ::= "variant(" AxisVal ( "," AxisVal )* ")"
AxisVal   ::= AxisName "(" ValueLabel ")"
Inst      ::= "inst(" Ref ( "," "canvas(" Path ")" )? EmbedExtras? ")"
At        ::= "at(" AxisVal ( "," AxisVal )* ")"
```

- `comp` (or `component`) marks a set root. `slot` marks a slot. A defaulted-type `comp` is promoted to a frame so variant children can nest.
- `axes[...]` (and its alias `props[...]`) declares axes and values. Both forms parse identically. Axis lists tolerate commas or spaces between axes; value lists are comma-separated. Names and labels MAY be quoted.
- `variant(...)` on a set child records the child's coordinate.
- `inst(ref)` places an instance; `canvas(path)` targets a cross-canvas master; the embedded extras (`emb`, plus an optional parent spec) are the self-contained storage form (§13.6).
- `at(...)` sets an instance configuration on an `inst()` line or a modify line targeting an instance.

### 13.3 Default configuration

`inst(ref)` with no `at()` uses the default configuration: the coordinate of the **top-left-most** variant (smallest top, then smallest left). There is no stored default-variant id; the default is computed.

### 13.4 Reconcile operators

A component set is edited by redeclaring its axes with operators. On a modify line addressing a set:

- `axes[+state[idle,active]]` adds an axis; existing variants are stamped with its first value so none goes off-grid.
- `axes[accessory[+swipe]]` adds a value to an existing axis.
- `axes[accessory->trailing]` renames an axis, preserving its key (and all coordinates that reference it).
- `axes[-state]` removes an axis; coordinates and configurations are stripped, collisions are flagged, and no variant or instance is ever deleted.

The key-preservation invariant is normative: matched axes keep their key and matched values keep their key; rename and reorder never re-key, so every variant coordinate and instance configuration continues to resolve. A bare redeclare (all axes without operators) is a replace: the declared list is the full axis set, and unlisted axes are removed. Mixing bare and operator axes in one bracket is an error (B706); a malformed rename (an empty side of `->`) is an error (B707).

NOTE: A reconcile applies to a set that already exists at reconcile time (the final pass). A set created earlier in the *same* batch is treated as freshly created and its reconcile lines are a no-op in that batch; the same reconcile lines applied in a subsequent operation take effect. Adding a value or axis that no variant yet backs (for example `axes[+size[sm,lg]]` with no `variant(size(lg))` row) succeeds with an info diagnostic that `at(size(lg))` falls back to the nearest variant until a backing variant is added.

### 13.5 Instances, overrides, and slots

An instance's configuration is resolved by name and label against the master's axes. Overrides are expressed as flat `override(ref) {props}` lines indented under the instance; only overridden children appear, and only the changed properties are written. A `override(ref) slot` line followed by indented children replaces a slot's contents. An override target is matched to a descendant by master-child reference, then by session reference, then by id, then by name or text.

The override categories understood by the sync engine are exactly these 17 strings: `fills`, `strokes`, `textData`, `rotation`, `isFlippedH`, `isFlippedV`, `isVisible`, `vectorPath`, `rectangleData`, `parentData`, `layoutBehavior`, `name`, `effects`, `opacity`, `circleData`, `constrainProportions`, `designSystem`. There is no separate size, position, or corner-radius category; those fold into `parentData` and `layoutBehavior`.

NOTE: `ov[...]` and `mref(...)` category lists are parsed as free-form strings and are not validated against the 17-name vocabulary at parse time. An unknown category rides through and is inert to sync.

### 13.6 Cross-canvas and embedded instances

An instance whose master is on another canvas is placed by `inst(ref,canvas(path))`, and its configuration is stored by name (self-contained). In canonical storage, a cross-canvas instance is always the **embedded** form: `inst(ref,canvas(path),emb,parentSpec)` with `at(...)` and `ov[...]` suffixes, and each descendant carrying `mref(masterChildRef[,category...])`. The embedded form self-contains the instance subtree; it is the storage mitigation for master-absent, cross-canvas, and vector-override cases (§17.3).

### 13.7 Worked examples

A set with one axis and two variants (validated, sovereign, `#refs` local to the call):

```
comp "Toggle" axes[state[on,off]] #toggle
  al(h,x(e),y(c),pad(4)) variant(state(on)) s(51,31) f[(#22C55E)] rd(16)
  al(h,x(s),y(c),pad(4)) variant(state(off)) s(51,31) f[(#94A3B8)] rd(16)
```

Reconcile edits (all four operator forms):

```
#toggle axes[+size[sm,lg]]
#toggle axes[state[+disabled]]
#toggle axes[size->scale]
#toggle axes[-size]
```

See [`components.md`](reference/components.md).

---

## 14. Masks, booleans, and parent types

Masks and booleans are parent types (frames) whose children are combined geometrically. The parent-data type values are frame, group, autoLayout, booleanUnion, booleanSubtract, booleanIntersect, booleanExclude, and mask; component and instance are also frames.

```
ParentType ::= "fr" | "gr" | "group" | "mask" | "mask(" MaskKind ")"
             | "bool(" BoolOp ")" | AutoLayout
MaskKind   ::= "vector" | "alpha" | "luminance"
BoolOp     ::= "union" | "subtract" | "intersect" | "exclude" | "u" | "s" | "i" | "x"
```

- `mask` (bare) is a vector mask; `mask(alpha)` and `mask(luminance)` select the other kinds. The last child is the mask shape; earlier children are the masked content. An unknown kind errors and defaults to vector.
- `bool(op)` combines children by union, subtract, intersect, or exclude; the short aliases `u`, `s`, `i`, `x` are accepted. An unknown op errors and still creates a frame.

NOTE: The parser accepts the short boolean aliases and bare `mask`; the encoder emits only the long forms and `mask` for the vector default. The accepted-input set is a superset of the emitted-output set.

Example (validated, sovereign): a boolean subtract:

```
bool(s) "Cutout"
  r s(100,100) f[(#111827)]
  c s(60,60) p(20,20)
```

---

## 15. Modify semantics

A modify line addresses an existing element by id, session reference, or `override(...)`, and applies only the properties present. Absent keys preserve the existing value.

### 15.1 Partial application and skip

The pervasive rule is *empty positional equals skip*. There is no skip sentinel; an empty slot is the only skip. `p(,300)` keeps x and sets y; `s(200,)` keeps height; `t(,,,lh(1.5))` changes only line height. Whitespace is literal for text content, so an empty text slot is a skip but a single-space slot sets a space.

### 15.2 All-or-nothing tokens

Multi-positional tokens that reject a partial-empty argument list are `pad(...)`, `flip(...)`, `skew(...)`, and the four-corner `rd(...)`; a partial-empty is a parse error. By contrast `p()`, `s()`, and `pin()` skip per axis (`pin(,c)` keeps the horizontal pin and sets the vertical), and `o()`, `rot()`, and `rd(N)` skip the whole token when empty.

### 15.3 Relative deltas

The delta tokens `p+`, `s+`, `rd+`, `rot+`, and `o+` apply a relative change and are modify-only.

### 15.4 Fills and strokes on modify

A fill or stroke *array* replaces the whole stack (an empty list clears it). The stack *operator* forms (`+`, `-`, `->`) mutate in place (§7.4). Vector region fills are matched by id, then by position.

### 15.5 Reorder and reparent

`before(id)` and `after(id)` place a sibling; they work on create and modify. `after(X)` is sugar: it desugars to `before(X.next)` or an append, and only the `before` placement reaches the builder. `front`/`back(N)` change z-order but are skipped on a create that already carries `before()`/`after()`. `parent(id)` reparents; `parent(delete)` on a modify is absorbed to a delete.

Example (validated, sovereign): move y only, then reparent (with a target frame present):

```
fr s(300,200) #frame
r s(40,40) f[(#0080FF)] #dot
#dot p(,120)
#dot parent(#frame)
```

---

## 16. Directives

Directives are non-element operations. They produce maps whose type is a directive, do not take an id, and do not push the nesting stack.

```
Directive ::= "delete(" Ref ")" | "ungroup(" Ref ")" | "clone(" Ref ")"
            | "replace(" Ref ")" | "before(" Ref ")" | "after(" Ref ")"
            | "undo(" Label ")" | "redo(" Label ")"
            | Mark
Mark      ::= "//" Label                    (* checkpoint marker line *)
```

- `delete(id)` (or `delete(#ref)`) removes an element. A bare `#ref delete` on a modify line, and `parent(delete)` on a modify, are equivalent absorptions.
- `ungroup(id)` dissolves a group.
- `clone(id)` and `replace(id)` copy and replace; `before(id)`/`after(id)` are placement directives that chain to the following type token.
- `undo("label")` rewinds the session and canvas to the checkpoint registered by that label (`redo("label")` replays forward). A bare `undo`/`redo` without a label is an error. The checkpoint itself is registered by a `// label` marker line or a trailing `// label` on an element line (§3.3).
- `ds_file("name") <body>` references or declares the design-system document that supplies the `$token` catalog; its body language is specified separately (see §16, design-system note, and <https://github.com/brilliant-hq/design-system>).

Design-system binding and per-element stamping. A bare `$token` on any color, font, or scale slot binds a design-system token (§3.6), resolved after build. The per-element token cascade is overridden by `ds(name, mode(value), ...)`: the first slot is a brand name (empty inherits, as in `ds(,theme(dark))`), and subsequent arguments are `modeName(modeValue)` calls. Children inherit through the cascade, so a stamp is placed only where an override is wanted. An unknown brand is stripped with a diagnostic, preserving the mode overrides. The token *discipline modes* (`default`, which requires token references on tokenizable slots; `new`, which authors a brand; `none`, which permits bare literals) govern whether bare hex and numeric values are accepted; they are a property of the authoring session, summarized here and specified with the design-system DSL. The session's discipline mode is set by a sticky `designSystem` selector on the element-authoring surface: it takes `default`, `new`, `none`, or the name of a brand authored earlier in the session, and once set it persists across later calls until changed (an external session defaults to `none`).

See [`directives.md`](reference/directives.md).

Example (validated, sovereign): create with a checkpoint, then rewind:

```
r s(40,40) f[(#FF0000)] // red block
undo("red block")
```

---

## 17. The two dialects and the lossless contract

Blueprint has one grammar and two conformance dialects. The distinction is normative.

### 17.1 Forgiving authoring dialect

The authoring dialect is what the parser accepts from an agent or a paste. Before tokenizing, a preprocessor rewrites common-intent, CSS-shaped forms into canonical forms and emits an info diagnostic teaching the canonical spelling. The full rewrite catalog is Appendix D. Two guards constrain every absorption and are REQUIRED:

1. **A form is absorbed only if it was previously an error.** An absorption MUST NOT change the meaning of any program that was already valid; it can only give meaning to input that previously had none.
2. **An absorption MUST NOT handcuff intentional usage.** Where a capable author might mean a form literally, it is not absorbed. For example, a four-argument `sh(x,y,blur,#hex)` is absorbed as a CSS shadow, but a single-argument `sh(N)` is preserved as `scaleTo(height,N)`.

On build, the authoring dialect additionally heals tokens: unknown-brand token literals are re-resolved against the live catalog, unknown brands are stripped, and element ids from a paste are re-minted.

### 17.2 Strict canonical storage dialect

The canonical dialect is what the encoder produces for at-rest storage. It is strict: decoding performs zero normalization and zero token healing, because every emission is self-contained. Specifically, the canonical dialect:

- writes colors as `tok(ref,#hex)` and opacity, weight, and family as tagged literals (`N:$tok`), so no re-resolution is needed;
- carries and preserves element ids (a stored line creates under its carried id rather than being read as a modify);
- emits the faithful graph form for every vector (§11.4);
- carries the storage-only extras that re-derivation would drift:
  - `p()` on auto-layout children (position is otherwise derived);
  - `hug:N` extents on every hug axis;
  - `ext(w,h[,rotField])` for fill-sized axes, instance-root boxes, or a rotation field that disagrees with the OBB;
  - `obb(x0,y0,...,x3,y3)` when the declared-rectangle reconstruction misses the stored corners;
  - explicit-zero handle flags on graph edges (the eighth edge slot);
  - self-contained `o(value:$tok)` opacity literals;
  - verbatim `ov[...]` and `mref(ref[,cat...])` override-category sets;
  - the explicit-empty `f[]` marker.

Numeric precision at the storage boundary is four decimal places: integers are written whole, otherwise the shortest representation that reparses exactly, capped at four decimals, with trailing zeros stripped so a second save is byte-identical.

The parse surface is a superset of the authoring surface: the parser reads all of these storage-only tokens without error, so a pasted canonical body does not crash, but they are produced only under canonical emission.

### 17.3 The lossless round-trip contract

The following is a normative requirement on a conforming implementation.

For every element, the cycle *design to canonical Blueprint to parse to build to design* MUST be structurally identical for every element type and every persistent property, and a re-emission of the decoded document MUST be byte-identical to the original (idempotent on the second cycle).

Known residual (documented, not a license to diverge elsewhere): instance-descendant element ids and internal component property and value keys do not round-trip byte-exact through the *compressed same-document* instance form; they re-mint as auto-layout positions re-derive. The oracle compares them structurally (via the instance root and its overrides) and by logical component shape (axis names, value labels, coordinates resolved to name equals label), not by key. The embedded instance form (§13.6) closes this residual for master-absent, cross-canvas, and vector-override cases. Id and key stability therefore holds for top-level and embedded forms, and not for compressed same-document instance subtrees.

---

## 18. Versioning and migration

### 18.1 Version stamp and per-version decode

Every stored document carries a `bp:vN` stamp (§2.1). An implementation keeps a decode path per version. A document stamped newer than the implementation's current version MUST be rejected.

### 18.2 Frozen default tables

Terseness comes from default-elision (§ design goal 2): a value equal to its version's default is not written. Therefore a version's default table is **frozen forever**. The meaning of a stored document is pinned to its version's defaults; those defaults MUST NOT change in place. A change to any default is a new version (`bp:vN+1`) plus a forward migration, never an in-place edit. The complete frozen `bp:v1` default table is Appendix B.

### 18.3 Additive versus breaking changes

An additive change (a new optional token, a new element type, a new fill kind) that does not change the meaning of any existing form does not require a version bump. A breaking change (a changed default, a changed meaning of an existing form) requires a new version and a forward migration.

### 18.4 Migration on read

Two distinct migration regimes exist and MUST NOT be conflated:

1. **YAML (v0) to `bp:v1`.** A document whose first line is not `bp:v` is legacy YAML (the v0 fallback). It is loaded through the YAML reader and re-saved as `bp:v1` on the next write (a full decode and re-encode). This is not a text-to-text step; it is a load-and-re-encode. The migration is non-destructive: originals are preserved under a mirrored archive tree.
2. **`bp:vN` to `bp:vN+1`.** A future breaking change ships a text-to-text migration that transforms the stored artifact text (frontmatter and body) from one version to the next. The migration runner applies steps forward until the document reaches the current version; a missing step is a defect. No such steps are registered for `bp:v1` (it is the current and only stored version).

The live realtime sync wire now carries the same canonical form as at-rest storage: each wire unit is a per-element region of the stored canonical document (a body fragment) under a `bp:v1` header, reassembled server-side by pre-order concatenation in arrival order. Storage and the live transport share one canonical language; the wire is Blueprint, not YAML.

---

## Appendix A. Collected grammar

This appendix collects the grammar of the whole language, consistent with the inline fragments above. It describes the accepted shape of each construct; where the hand-written parser diverges, the corresponding section carries a NOTE.

```
(* Document *)
Document      ::= VersionStamp NL Frontmatter Separator NL Body
VersionStamp  ::= "bp:v" Int
Frontmatter   ::= YamlMap
Separator     ::= "---"
Body          ::= ( Line NL )*

(* Lines *)
Line          ::= INDENT ( Blank | Comment | ElementLine | Continuation | MetaLine )
INDENT        ::= ( "  " )*
Blank         ::= WS*
Continuation  ::= Spans | VrLine
MetaLine      ::= Mark | "undo(" Label ")" | "redo(" Label ")"
Mark          ::= "//" Label

(* Comments *)
Comment       ::= HashComment | DashComment | SlashComment
HashComment   ::= "#" ( WS AnyText | EOL )
DashComment   ::= "--" AnyText
SlashComment  ::= "//" AnyText

(* Element line *)
ElementLine   ::= FirstToken? TypeToken? Property*
FirstToken    ::= "#" ( HexId | SessionRef )
              | "override(" ( "#" Ref | String ) ")"
              | HexId | AlphaId
HexId         ::= /[0-9a-fA-F]{8,16}/
AlphaId       ::= /[A-Za-z0-9_-]{2,}/           (* contains a letter, not a keyword *)
SessionRef    ::= /\d+/ | /[A-Za-z_][\w-]*/

(* Types *)
TypeToken     ::= Rect | Circle | Text | Frame | Group | AutoLayout
              | Svg | VectorType | LineElement | Mask | Boolean | Clone
Rect          ::= "r" | "rectangle" | "r(" Size ")"
Circle        ::= "c" | "circle" | "c(" Size ")" | "arc(" Num "," Num ")"
Frame         ::= "fr" | "frame" | "fr(" Size ")"
Group         ::= "gr" | "group" | "gr(" Size ")"
Mask          ::= "mask" | "mask(" MaskKind ")"
Boolean       ::= "bool(" BoolOp ")"
Clone         ::= "clone(" Ref ")"
Size          ::= SizeVal ( "," SizeVal )?
SizeVal       ::= Num | "hug" | "fill" | "fill:" Num | "hug:" Num

(* Text *)
Text          ::= "t(" Content ( "," Family )? ( "," SizeArg )? ( "," Extra )* ")"
Spans         ::= "spans[" SpanRange ( "," SpanRange )* "]"

(* Auto layout *)
AutoLayout    ::= "al(" Dir? AlArg* ")"
Dir           ::= "h" | "v"
AlArg         ::= "x(" Align ")" | "y(" Align ")" | "g(" Gap ")" | "pad(" Padding ")" | "wrap"
Align         ::= "s" | "c" | "e" | "sb"

(* Pin constraints *)
Pin           ::= "pin(" HPin? "," VPin? ")"
HPin          ::= "l" | "r" | "c" | "lr" | "scale"
VPin          ::= "t" | "b" | "c" | "tb" | "scale"

(* SVG / icon *)
Svg           ::= "svg(" SvgArg ")"
SvgArg        ::= InlineSvg | IconRef | UrlRef | PathRef
IconRef       ::= "icon:" Name | BareName

(* Vector *)
VectorType    ::= "v(" VecArg ( "," VecArg )* ")"
VecArg        ::= IconAnnot | Nodes | Edges | Regions | "closed"
Nodes         ::= "nodes[" NodeTuple ( "," NodeTuple )* "]"
NodeTuple     ::= "(" Int "," Num "," Num ( "," NodeType ( "," Cap )? )? ")"
NodeType      ::= "st" | "mi" | "as" | "di" | ""
Cap           ::= "r" | "n" | "sq" | "ar" | "c"
Edges         ::= "edges[" EdgeTuple ( "," EdgeTuple )* "]"
EdgeTuple     ::= "(" Int "," Int "," Int ( "," Num "," Num "," Num "," Num ( "," ZeroFlags )? )? ")"
ZeroFlags     ::= /[ab]+/
Regions       ::= "regions[" ( RegionSpec ( "," RegionSpec )* )? "]"
RegionSpec    ::= "(" EdgeWalk ( "," "hole" )? ")"
EdgeWalk      ::= ( Int Sign )+
Sign          ::= "+" | "-"
VrLine        ::= "vr(" RegionId ( "," PathData )? ")" ( Fills )? "hole"?
RegionId      ::= "r" Int
PathContour   ::= "path:d(" SvgPath ")"

(* Line / connector *)
LineElement   ::= "line(" ( VectorForm | EndpointForm ) ")"
VectorForm    ::= Len ( "," Angle )? | Angle "," Len
EndpointForm  ::= "from(" Endpoint ")" "," "to(" Endpoint ")" ( "," ConnOpt )*
Endpoint      ::= Num "," Num | "#" Ref ( "," Side ( "," Frac )? )?

(* Components *)
CompDecl      ::= "comp" | "component"
AxesDecl      ::= ( "axes[" | "props[" ) Axis ( Sep Axis )* "]"
Axis          ::= AxisOp? AxisName ( "[" Values? "]" )?
AxisOp        ::= "+" | "-" | AxisName "->" NewName
Variant       ::= "variant(" AxisVal ( "," AxisVal )* ")"
Inst          ::= "inst(" Ref ( "," "canvas(" Path ")" )? EmbedExtras? ")"
At            ::= "at(" AxisVal ( "," AxisVal )* ")"
Mref          ::= "mref(" Ref ( "," Category )* ")"
Ov            ::= "ov[" Category ( "," Category )* "]"

(* Paint *)
Fills         ::= "f[" ( FillItems | "" ) "]"
FillItems     ::= FillItem ( "," FillItem )* | FillOp+
FillOp        ::= "+(" FillSpec ")" | "-(" FillSpec ")" | "(" FillSpec ")->(" FillSpec ")"
Strokes       ::= "st[" StrokeItem ( "," StrokeItem )* "]"

(* Effects *)
Effect        ::= "shadow(" ShadowArg* ")" | "outerglow(" GlowArg* ")" | "eblur(" Num? BlurArg* ")"

(* Directives *)
Directive     ::= "delete(" Ref ")" | "ungroup(" Ref ")" | "clone(" Ref ")"
              | "replace(" Ref ")" | "before(" Ref ")" | "after(" Ref ")"
              | "undo(" Label ")" | "redo(" Label ")"

(* Idioms *)
Token         ::= "$" TokenPath
Ref           ::= "#" ( HexId | SessionRef )
NumericTagged ::= Num ":" "$" TokenPath
```

Terminal classes `Num`, `Int`, `String`, `Name`, `TokenPath`, `Label`, `Mode`, `Category`, `SvgPath`, and `Path` are the obvious lexical categories: `Num` a decimal, `Int` a non-negative integer, `String` a double-quoted string with the escapes of §3.4, `TokenPath` a dotted token name, and so on.

---

## Appendix B. Frozen bp:v1 default tables

These are the `bp:v1` defaults on which canonical elision relies. They are frozen for `bp:v1` forever; a change is a new version plus a migration (§18.2). They are pinned by the conformance test suite.

### B.1 Element defaults

| Property | Default |
|---|---|
| rotation | 0.0 |
| opacity | 1.0 |
| blendMode | normal (source-over) |
| isFlippedH | false |
| isFlippedV | false |
| isVisible | true |
| isLocked | false |
| constrainProportions | false |
| skew | 0.0 |
| hPin / vPin | min / min (`pin(l,t)`, elided at default) |

### B.2 Parent (frame) defaults

| Property | Default |
|---|---|
| clipContent (frame, auto-layout) | false |
| isolated | false |
| maskType | vector |
| cornerRadius | 0.0 |
| cornerSmoothing | 0.0 |

### B.3 Stroke defaults

| Property | Default |
|---|---|
| position | center |
| join | miter |
| miterLimit | 4.0 |
| dashPattern | none |
| enabled | true |
| thickness | 1.0 |

### B.4 Text defaults

| Property | Default |
|---|---|
| fontWeight | regular |
| alignment | leading |
| verticalAlignment | top |
| isItalic | false |
| isUnderlined | false |
| isStrikethrough | false |
| textCase | none |
| fontSize | 24 |
| widthSizing / heightSizing | hug / hug |
| default fill | `color.text.primary` (unless `f[]`) |

### B.5 Per-type geometry defaults

| Type | Default size | Default fill |
|---|---|---|
| rectangle | 100 by 100 | none |
| circle | 100 by 100 | none |
| text | hug by hug | `color.text.primary` |
| frame | 100 by 100 | none |
| group | auto-fit to children | none |
| auto-layout | 100 by 100 (unless `s()`) | none |

### B.6 Meta defaults

| Property | Default |
|---|---|
| backgroundEnabled (absent key) | true |

### B.7 Fill sub-type defaults

| Fill kind | Default |
|---|---|
| linear gradient | 180 degrees, black to white, positions 0 and 1 |
| radial gradient | center to top, white to black |
| angular gradient | centered, black to white |
| background blur | radius 8 |
| inner shadow | `#000000`, opacity 0.5, y 2, blur 4 |
| inner glow | `#FFFFFF`, opacity 0.6, blur 4, screen |
| drop shadow | `color.shadow`, opacity 0.25, y 4, blur 8 |
| outer glow | `color.glow`, opacity 0.6, blur 8, screen |
| element blur | radius 4 |

---

## Appendix C. Diagnostics index

Each row is a diagnostic code, its severity, category, the construct it applies to, and its triggering condition. Severity error means the line is not applied; warning and info mean the line is applied.

| Code | Severity | Category | Construct | Condition |
|---|---|---|---|---|
| B101 | error | property | auto layout | space-between on the cross axis |
| B102 | error | property | auto layout | invalid cross-axis alignment |
| B103 | error | property | auto layout | invalid main-axis alignment |
| B201 | error | syntax | text | font, size, and weight quoted together |
| B203 | error | property | text | explicit empty content `t("")` |
| B204 | error | property | text | create text with no content |
| B205 | info | property | text | bare sub-pixel `ls(N)` (magnitude below 0.35, no unit) |
| B301 | error | property | fills | CSS `180deg` in a fill id or color |
| B302 | error | property | fills | `#hex,N` CSS opacity |
| B303 | error | property | fills | abbreviated gradient `l(...)`/`lg(...)` |
| B304 | error / info | property | fills | abbreviated `r(...)` gradient (error) OR zero-radius radial (info) |
| B305 | error | property | fills | unknown fill keyword `word(...)` |
| B306 | error | property | fills | `tok(...)` read-format used as authoring input |
| B307 | error | property | fills / strokes | malformed hex digit count (error) OR glass in a stroke (error) |
| B401 | error | syntax | svg | size parameters inside `svg(icon:name,w(N),h(N))` |
| B501 | error / info | layout / property | property-only line (error) OR auto layout with no gap (info) |
| B502 | info | property | auto layout | auto layout with no padding |
| B601 | error | composition | line | no stroke (open path) |
| B602 | error | property | line | `rot()` on a line |
| B701 | error | property | component | `axes[...]`/`props[...]` on a non-comp, non-modify line |
| B702 | warning | property | component | a declared or added axis with no values |
| B703 | error | property | component | `variant()` and `at()` together |
| B704 | error | reference | component | `at()` with no instance target |
| B705 | error | layout | component | `variant()` on a non-element line |
| B706 | error | property | component / list | mixed bare and operator items in `axes[...]` |
| B707 | error | property | component | malformed rename (`old->` or `->new`) |
| B708 | error | property | fills / strokes | mixed bare and operator items in `f[...]`/`st[...]` |

Pin misuse (`pin(...)` on an auto-layout child without `abs`, or on a group child) carries no numbered code: it is a named non-fatal property diagnostic emitted by the parser's parent-context pass, specified in §6.1 (authoring lanes only; stored-document loads are exempt).

Implementation notes on code collisions (tracked as defects; a future revision assigns disambiguated codes):

- **B501** is used for two unrelated conditions with different severities: the property-only-line error and the auto-layout-no-gap info. A consumer MUST disambiguate by severity and category, not by code alone.
- **B304** is used for both an error (an abbreviated `r(...)` gradient) and an info (a zero-radius radial gradient).
- **B307** is used for both a malformed hex digit count and a glass fill placed in a stroke.

Non-fatal accounting: an unrecognized property token increments the error count but not the fatal error count; the element still applies with the bad token dropped. Malformed arguments to `connect()`, `arc()`, and `line()` are fatal.

---

## Appendix D. Preprocessing and absorption catalog

The forgiving authoring dialect (§17.1) rewrites the following forms before tokenizing, each with an info diagnostic teaching the canonical form. Every entry satisfies the two guards of §17.1: it was previously an error, and it does not handcuff an intentional literal reading.

| # | Trigger | Rewrite | Diagnostic |
|---|---|---|---|
| 1 | inline `--` or `//` outside quotes, parens, brackets | comment stripped; non-empty `//` payload becomes a checkpoint label | none (or checkpoint) |
| 2 | `Npx` (not inside `ls(...)`, not after a quote or word char) | `px` stripped | none (silent, redundant unit) |
| 3 | `rgb(r,g,b)` / `rgba(r,g,b,a)` | `#RRGGBB`; `rgba` with alpha below 1.0 becomes `solid(#RRGGBB,o(a))` | silent |
| 4 | four-argument `sh(x,y,blur,#hex)` | `shadow(#color[,o(alpha)][,x(x)],y(y),blur(blur))`; eight-hex alpha split into `o()` | correction note |
| 5 | eight-character `#RRGGBBAA` | `solid(#RRGGBB,o(0.NN))` | correction note |
| 6 | CSS alignment in `x()`/`y()` (`left`, `flex-end`, ...) | `x(s)`, `y(e)`, ... | correction note |
| 7 | CSS `deg` in a gradient angle (`linear(135deg,...)`) | `linear(135,...)` (also radial, angular, conic) | correction note |

Absorptions applied outside the line preprocessor:

| Trigger | Rewrite |
|---|---|
| `#{8..16 hex}` as a first token | bare id (modify-by-id), with a no-`#` note |
| `o(N)` with N above 1 | interpreted as a percentage, divided by 100, with a note |
| `smooth(N%)` or `smooth(N>1)` | fraction in `0..1`, with a note |
| `lh(N)` with N above 3 | pixels divided by font size (a multiplier), with a note |
| bare `lh(auto)` (no carrier) | unset-slot semantics (auto, resolved from the font's metrics), with a note teaching `lh(auto,X)` |
| a create-time leaf that acquires a child | converted to a container frame (leaf absorption, §5.13) |
| a flat top-level `override(...)` | modify-by-name (§4.3) |
| `parent(delete)` on a modify | delete (§15.5) |
| `filter(name,...)` wrapper | unwrapped to `name(...)`, with a note |
| a retired platform-font sentinel (`System`, `System Font`, empty) | the bundled default family; other look-alike names (Helvetica, Arial) are kept verbatim with a warning, not remapped |

The two guards are normative (§17.1). A rewrite MUST have been a previous error, and a rewrite MUST NOT be applied where a capable author might intend the form literally.

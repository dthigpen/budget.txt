# Plain Text Budget Format Specification

**Status:** Draft  
**File extension:** `.txt`  
**Encoding:** UTF-8

## Overview

This specification defines a simple, human-readable plain-text format for household envelope budgeting.

The format is designed around several principles:

- The source of truth is a plain-text file.
- The file should remain understandable without specialized software.
- Events record what happened.
- Definitions describe what things are and how they behave.
- Reports derive current or historical state from the dated record stream and applicable definitions.
- The file is intended to be manually editable.
- Events may be entered out of chronological order and can be sorted chronologically.
- The syntax should be simple enough to comfortably edit by hand.
- The core format should avoid unnecessary domain-specific complexity.
- The format should not assume USD even though USD is a common use case.
- The core syntax should be stable while allowing additional keyword vocabulary to be added by extensions.
- Tools should preserve information they do not understand when rewriting a file.

The format is a budgeting model, not a general-purpose accounting system. In particular, the core model represents money assigned to envelopes rather than physical bank or credit-account balances.

This specification defines both the common language and the core verb semantics. Implementations may provide additional user interfaces, reports, and extensions without changing the meaning of the core format.

## File Format

A budget file is an ordinary UTF-8 text file, conventionally using the `.txt` extension.

There is no required header, metadata section, YAML frontmatter, or wrapper syntax.

A file may contain dated records and file-level declarations.

For example:

```text
!options version=1.0
!options currency=USD
!meta accounts.mastercard.icon="credit-card"

2026-01-01 envelope groceries budget=500
2026-01-01 meta accounts.mastercard.cycle "19..20"
2026-01-01 meta accounts.mastercard.due 28

2026-09-01 allocate 500 groceries
2026-09-03 spend 42.18 groceries account=mastercard merchant="Target"
```

File-level declarations begin with `!` and apply to the file as a whole. They are not dated records.

Blank lines are permitted and have no semantic meaning.

## Line Structure

Each non-empty line is either a dated record or a file-level declaration.

A dated record has the general structure:

```text
DATE VERB POSITIONAL_ARGUMENT... KEYWORD_ARGUMENT...
```

For example:

```text
2026-09-08 spend 42.18 groceries account=mastercard merchant=Target
```

The date and verb are the first two tokens.

Positional arguments, when present, must precede all keyword arguments. Once a keyword argument is encountered, no subsequent positional argument is permitted.

Each verb defines its own required and optional positional arguments and core keyword arguments.

Keyword arguments have the form:

```text
KEY=VALUE
```

The core syntax is permissive about additional keyword arguments by default. See [Keyword Arguments](#7-keyword-arguments) and [Extensions and Additional Keywords](#extensions-and-additional-keywords).

Unknown verbs are invalid.

File-level declarations begin with `!` rather than a date. The core format currently defines `!options`.

## Dates

A concrete date uses ISO-style year-month-day notation:

```text
YYYY-MM-DD
```

For example:

```text
2026-09-08
```

Dates represent the date on which an event occurred or a definition became effective, rather than the date on which the line was entered into the file.

### Date Ordering

Lines do not need to appear in chronological order.

The following is valid:

```text
2026-09-08 spend 30 groceries account=visa
2026-09-01 allocate 500 groceries
2026-09-03 spend 20 groceries account=visa
```

Programs must not assume that physical file order represents chronology.

A program may provide a sorting operation that rearranges dated records according to their dates. Sorting must not otherwise change their meaning.

## Verbs

The second token on a dated record is a verb.

The core format defines:

- `allocate` — add money to an envelope from outside the budgeting model.
- `spend` — remove money from an envelope as spending.
- `move` — transfer available money between envelopes.
- `deallocate` — remove money from an envelope and return it outside the budgeting model.
- `envelope` — define an envelope and its properties.
- `close` — end the current lifetime of an envelope.
- `balance` — assert an envelope's available balance.
- `note` — record human-readable information.
- `meta` — store extension metadata with historical context.

The core format also defines the file-level `!options` and `!meta` declarations.

Each verb explicitly defines its required and optional arguments and semantic behavior. Unknown verbs are invalid.

Entity references must resolve to an applicable definition. Events never implicitly create entities.

### `allocate`

Adds money to an envelope's available balance.

Syntax:

```text
DATE allocate AMOUNT ENVELOPE [KEYWORD_ARGUMENT...]
```

Required positional arguments:

1. `AMOUNT` — a non-negative monetary amount.
2. `ENVELOPE` — the name of an existing envelope.

Optional keyword arguments:

- `note=TEXT` — a human-readable annotation.

Example:

```text
2026-09-01 allocate 500 groceries note="September allocation"
```

`allocate` is additive. It increases the envelope's available balance by `AMOUNT`; it does not set, replace, or otherwise modify the envelope's budget target.

An `allocate` event may be used when money enters the budgeting model from outside the model, including money being added back after a refund.

An envelope must already be defined and active when referenced. An `allocate` event never creates an envelope.

An amount of zero is valid and has no effect.

Negative amounts are invalid; `allocate` is never used to remove money from an envelope.

### `spend`

Records money spent from an envelope.

Syntax:

```text
DATE spend AMOUNT ENVELOPE [KEYWORD_ARGUMENT...]
```

Required positional arguments:

1. `AMOUNT` — a non-negative monetary amount.
2. `ENVELOPE` — the name of an existing envelope.

Optional keyword arguments:

- `account=TEXT` — the payment account or payment source used for the spending.
- `merchant=TEXT` — the merchant associated with the purchase.
- `note=TEXT` — a human-readable annotation.

Example:

```text
2026-09-03 spend 63.42 groceries account=mastercard merchant="Target"
```

The `account` keyword is optional because the core budget model does not require payment accounts to be tracked.

If `account` is supplied, it is a free-form string identifying the payment source used for the spending. This may represent credit cards, checking accounts, cash, or other payment sources.

`spend` decreases the envelope's available balance by the specified amount. It does not change the envelope's budget target.

A spend may result in a negative envelope balance. This is a valid state and is not a validation error.

A spend event does not directly modify an account balance or record payment of a credit-card statement. Reports may use the account reference to analyze spending patterns and funding requirements.

An envelope must already be defined and active when referenced. A `spend` event never creates an envelope.

An amount of zero is valid and has no effect.

Negative amounts are invalid; `spend` is never used to add money to an envelope.

Split purchases are represented by multiple `spend` events.

### `move`

Transfers available money from one envelope to another.

Syntax:

```text
DATE move AMOUNT FROM_ENVELOPE TO_ENVELOPE [KEYWORD_ARGUMENT...]
```

Required positional arguments:

1. `AMOUNT` — a non-negative monetary amount.
2. `FROM_ENVELOPE` — the name of an existing source envelope.
3. `TO_ENVELOPE` — the name of an existing destination envelope.

Optional keyword arguments:

- `note=TEXT` — a human-readable annotation.

Example:

```text
2026-09-08 move 50 groceries dining
```

`move` decreases the source envelope's available balance and increases the destination envelope's available balance by the same amount.

A `move` does not change either envelope's budget target.

The source envelope does not need to have sufficient available money. A move may therefore result in a negative source balance.

Both envelopes must already be defined and active when referenced. A `move` event never creates an envelope.

The source and destination envelopes must be distinct.

An amount of zero is valid and has no effect.

Negative amounts are invalid; `move` is never used to reverse the direction of a transfer.

### `deallocate`

Removes money from an envelope and returns it outside the budgeting model.

Syntax:

```text
DATE deallocate AMOUNT ENVELOPE [KEYWORD_ARGUMENT...]
```

Required positional arguments:

1. `AMOUNT` — a non-negative monetary amount.
2. `ENVELOPE` — the name of an existing envelope.

Optional keyword arguments:

- `note=TEXT` — a human-readable annotation.

Example:

```text
2026-09-30 deallocate 250 vacation note="Moved leftover to external savings"
```

`deallocate` decreases the envelope's available balance by the specified amount. It does not change the envelope's budget target.

A `deallocate` event represents money leaving the budgeting model without being recorded as spending. For example, unused money may be transferred to external savings that are not represented by an envelope.

The source envelope does not need to have sufficient available money. A `deallocate` event may therefore result in a negative envelope balance.

An envelope must already be defined and active when referenced. A `deallocate` event never creates an envelope.

An amount of zero is valid and has no effect.

Negative amounts are invalid; `deallocate` is never used to add money to an envelope.

### `envelope`

Defines an envelope and its properties.

Syntax:

```text
DATE envelope NAME [KEYWORD_ARGUMENT...]
```

Required positional arguments:

1. `NAME` — the name of the envelope.

Core keyword arguments:

- `budget=AMOUNT` — the target budget for the envelope.
- `note=TEXT` — a human-readable annotation.

Example:

```text
2026-01-01 envelope groceries budget=450
```

An envelope definition establishes that the named envelope exists and describes its properties. It does not allocate money to the envelope or otherwise change its available balance.

An envelope may be defined without a budget:

```text
2026-01-01 envelope vacation
```

A budget amount must be non-negative. A budget amount of zero is valid.

An envelope may be redefined by adding another `envelope` line for the same active lifetime. A later definition changes only the properties explicitly supplied by that definition. Properties not supplied inherit their values from the previous applicable definition.

Redefining an envelope does not change its available balance.

Two definitions for the same envelope on the same date must not provide conflicting values for the same property. Physical line order must not determine which conflicting definition takes effect.

### `close`

Ends the current lifetime of an envelope.

Syntax:

```text
DATE close envelope NAME [KEYWORD_ARGUMENT...]
```

Required positional arguments:

1. `envelope` — the literal word `envelope`.
2. `NAME` — the name of an existing envelope.

Optional keyword arguments:

- `note=TEXT` — a human-readable annotation.

Example:

```text
2026-09-30 close envelope vacation
```

An envelope must have an available balance of zero when it is closed.

Closing an envelope does not delete the envelope or its history. The closed lifetime remains available for historical interpretation and reporting, but it is no longer active.

A closed envelope cannot be referenced by `allocate`, `spend`, `move`, or `deallocate` after its closing date.

A later `envelope` definition using the same name begins a new lifetime rather than reopening the previous one.

Properties do not inherit across a close boundary.

For example:

```text
2026-01-01 envelope vacation budget=1000
2026-09-30 deallocate 250 vacation
2026-09-30 close envelope vacation
2027-01-01 envelope vacation budget=1500
```

The first `vacation` lifetime and the second `vacation` lifetime are separate. Closing itself does not change the available balance.

### `balance`

Asserts the available balance of an envelope at the end of a date.

Syntax:

```text
DATE balance ENVELOPE AMOUNT [KEYWORD_ARGUMENT...]
```

Required positional arguments:

1. `ENVELOPE` — the name of an envelope.
2. `AMOUNT` — the expected available balance.

Optional keyword arguments:

- `note=TEXT` — a human-readable annotation.

Example:

```text
2026-09-30 balance entertainment 50
```

The assertion succeeds when the envelope's derived available balance at the end of `DATE` equals `AMOUNT`.

A `balance` assertion does not change the envelope's available balance.

The envelope must exist by the date being asserted. A balance assertion never creates an envelope.

The asserted amount may be negative because envelope balances may be negative.

A failed assertion is reported separately from file validation errors.

### `note`

Records human-readable information in the event log.

Syntax:

```text
DATE note TEXT
```

Required positional arguments:

1. `TEXT` — the note content.

Example:

```text
2026-09-08 note "Started tracking credit card funding"
```

A standalone `note` has no effect on financial state, definitions, or configuration.

A standalone `note` is an independent dated record. To annotate a specific record directly, use the core `note=TEXT` keyword where that verb supports it.

### `meta`

Stores extension metadata with historical context.

Syntax:

```text
DATE meta KEY VALUE [KEYWORD_ARGUMENT...]
```

Required positional arguments:

1. `KEY` — a namespaced key identifying the metadata.
2. `VALUE` — the metadata value.

Optional keyword arguments:

- `note=TEXT` — a human-readable annotation.

Example:

```text
2026-01-01 meta accounts.mastercard.cycle "19..20"
2026-01-01 meta accounts.mastercard.due 28
2026-06-01 meta accounts.mastercard.due 25
```

`meta` stores opaque key-value data intended for use by extensions. The core format does not interpret the meaning of specific keys or values.

Keys support hierarchical namespacing using periods. For example:

- `accounts.mastercard.cycle` — properties scoped to a specific account
- `envelopes.groceries.category` — properties scoped to a specific envelope
- `ui.theme` — global configuration

The exact semantics of any particular key or namespace are defined by the applicable extension. An implementation may understand the extension, preserve it without interpreting it, or report that the extension is unsupported.

`meta` is a dated record and participates in historical interpretation. A later `meta` record with the same key overrides the previous value for dates on or after the later record's date.

For example:

```text
2026-01-01 meta accounts.mastercard.due 28
2026-06-01 meta accounts.mastercard.due 25
```

means the due date is 28 before June 1 and 25 beginning June 1.

A `meta` record has no effect on financial state, envelope balances, or core definitions.

### `options`

`options` is a file-level declaration rather than a dated verb.

Syntax:

```text
!options [KEYWORD_ARGUMENT...]
```

Examples:

```text
!options version=1.0
!options currency=USD
!options strict-keys=true
```

Options configure how the file is interpreted. They do not record financial events, define entities, or change envelope balances.

Options apply to the entire file regardless of their physical position.

The `version` option identifies the version of the file format used by the file. A file must have at most one effective format version, and the version cannot change within a file.

The `currency` option defines the default currency used where a monetary amount does not otherwise specify a currency.

The `strict-keys` option enables strict keyword validation. When it is not enabled, additional keyword arguments are accepted and must be preserved by tools even when those tools do not understand their semantics.

When `strict-keys=true`, a keyword is valid only if it is defined by the core format or explicitly declared as additional vocabulary using `allow`.

For example:

```text
!options strict-keys=true
!options allow=envelope.category
```

Unknown options are invalid.

### `meta` (file-level)

`!meta` is a file-level declaration for static metadata that does not change over time.

Syntax:

```text
!meta KEY=VALUE
```

Examples:

```text
!meta accounts.mastercard.icon="credit-card"
!meta accounts.mastercard.color="#ff0000"
!meta ui.theme="dark"
```

File-level `!meta` stores static metadata intended for use by extensions. Unlike the dated `meta` verb, file-level `!meta` does not participate in historical interpretation and cannot be redefined over time.

File-level `!meta` is appropriate for metadata that applies globally to the file or to an entity regardless of date, such as UI preferences, display colors, or other static configuration.

The core format does not interpret the meaning of specific keys or values. Extensions define the semantics of particular keys and namespaces.

File-level `!meta` applies to the entire file regardless of its physical position.

## Positional Arguments

Positional arguments are separated by whitespace and interpreted according to the order specified by the verb's schema.

For example:

```text
2026-09-08 spend 42.18 groceries account=mastercard
```

has positional arguments:

```text
42.18
groceries
```

and the keyword argument:

```text
account=mastercard
```

Positional arguments always precede keyword arguments.

Positional arguments are intended for information that is fundamental to the operation represented by the verb. Optional or contextual information should generally be represented using keyword arguments.

The number, order, and meaning of positional arguments are part of a verb's syntax.

Unknown additional positional arguments are invalid unless explicitly supported by an extension or future format version.

## Keyword Arguments

Keyword arguments have the form:

```text
KEY=VALUE
```

and must appear after all positional arguments.

Core keyword arguments are defined by the verb that accepts them. Examples include:

```text
account=mastercard
merchant="Whole Foods Market"
budget=500
note="Replacement filter"
```

The core format is permissive about additional keyword arguments by default.

For example, this is valid in normal mode:

```text
2026-09-03 spend 63.42 groceries account=visa category=food
```

`category` is not part of the core `spend` schema, but it is accepted as additional vocabulary.

A tool that does not understand an additional keyword may preserve it as opaque data without interpreting it.

Tools that modify or rewrite a budget file must preserve keyword arguments they do not understand, including their values, unless the user explicitly requests their removal.

### Extensions and Additional Keywords

Additional keyword vocabulary may be used without changing the core grammar.

A file may declare additional vocabulary using the file-level `allow` option:

```text
!options allow=envelope.category
```

The name before the period identifies the verb and the name after the period identifies the additional keyword.

For example:

```text
!options allow=envelope.category

2026-01-01 envelope groceries category=food
```

declares `category` as additional vocabulary for `envelope`.

An `allow` declaration does not itself define the semantics of the keyword. An implementation may understand the extension, preserve it without interpreting it, or report that the extension is unsupported.

In permissive mode, an undeclared additional keyword is still accepted.

When `!options strict-keys=true` is present, a keyword must either be defined by the core format or explicitly declared with `allow`.

Additional keyword declarations are verb-specific. Declaring:

```text
!options allow=envelope.category
```

does not make `category` valid on `spend`.

Extensions must not redefine the meaning of core verbs, positional arguments, or core keyword arguments.

Keyword names are case-sensitive unless otherwise specified by the applicable schema.

## Whitespace

Whitespace separates tokens.

Multiple consecutive whitespace characters are equivalent to a single separator outside quoted strings.

For example:

```text
2026-09-08 spend 42.18 groceries account=mastercard
```

and:

```text
2026-09-08   spend   42.18   groceries   account=mastercard
```

have the same meaning.

Whitespace inside a quoted value is preserved.

## Quoting

Values that contain whitespace may be enclosed in double quotes:

```text
merchant="Whole Foods Market"
```

Without quoting, whitespace separates tokens.

Quoting is optional when a value does not require it:

```text
merchant=Target
```

and:

```text
merchant="Target"
```

represent the same string value.

### Escaping

Within a quoted value, a literal double quote is escaped with a backslash:

```text
merchant="Bob's \"Market\""
```

At minimum, implementations must support escaping a literal double quote within a quoted string.

The complete lexical grammar may define additional escape sequences. An implementation must not invent incompatible escape semantics.

## Names

Names identify envelopes.

Names are whitespace-delimited values and therefore cannot contain unquoted whitespace.

The canonical naming convention is kebab-case: lowercase words separated by single hyphens.

Examples:

```text
groceries
car-repairs
christmas-gifts
home-maintenance
emergency-fund
```

Kebab-case provides readable multi-word names without requiring quoting or introducing a programming-language-specific naming convention.

Names may contain Unicode characters. For example:

```text
épicerie
旅行
путешествия
```

may be used as names.

A canonical name must not begin or end with a hyphen, and consecutive hyphens are not part of canonical kebab-case names.

Names are case-sensitive.

The complete lexical grammar will define the exact set of characters permitted in unquoted names. Names used in syntactic positions must not be confused with keywords or other reserved syntax.

## Money

Money values are exact decimal quantities rather than binary floating-point values.

A basic monetary value may be represented as:

```text
42.18
```

The currency is determined by the applicable configuration and, where supported, an explicit currency keyword.

For example:

```text
!options currency=USD
```

allows:

```text
2026-09-08 spend 42.18 groceries account=mastercard
```

to use USD without requiring the currency on every event.

An event may explicitly specify another currency where its schema permits it:

```text
2026-09-08 spend 42.18 groceries account=visa currency=CAD
```

The core format does not assume USD is the only supported currency.

### Currency Precision

Currencies have applicable decimal precision.

For example:

- USD generally uses two decimal places.
- JPY generally uses zero decimal places.
- Some currencies use three decimal places.

Monetary quantities must be represented and calculated as exact decimal values.

The complete currency specification will define the supported currency identifiers, precision rules, and any rounding behavior.

### Negative Amounts

Ordinary money-changing verbs use non-negative amounts.

For example:

```text
2026-09-08 allocate 100 groceries
```

is valid.

This is invalid:

```text
2026-09-08 allocate -100 groceries
```

A negative value must not be interpreted as reversing the meaning of the verb.

Likewise:

```text
2026-09-08 spend -100 groceries
```

is invalid rather than meaning "add $100 to groceries."

If a future operation needs to represent a reversal, correction, withdrawal, or other opposite-direction operation, it should have an explicit semantic representation rather than relying on negative amounts.

## Currency Conversion

The core format does not automatically convert between currencies.

A report must not silently combine monetary values in different currencies.

For example, an implementation must not simply treat:

```text
100 USD
100 EUR
```

as one monetary total.

Currency conversion, if supported in the future, should be an explicit higher-level feature with clearly defined conversion rules.

The core data model should remain usable without exchange-rate infrastructure.

---

## Comments and Notes

The core format does not currently define a separate comment syntax.

Human-readable information can be represented by the `note` verb:

```text
2026-09-01 note "September budget"
```

A standalone `note` is a dated record and participates in sorting, but it does not affect financial state.

Specific records may also carry the core `note=TEXT` keyword:

```text
2026-09-08 spend 42.18 groceries account=visa note="Cleaning supplies"
```

A `note=` value is metadata belonging to the record on which it appears. It does not alter the semantics of the event or definition.

If a future version adds comments that are ignored by the parser, those comments must remain semantically distinct from `note` records.

## Blank Lines

Blank lines are permitted and have no semantic meaning.

They may be used to make the file easier for humans to read:

```text
2026-09-01 allocate 500 groceries
2026-09-01 allocate 150 dining

2026-09-03 spend 42.18 groceries mastercard
2026-09-04 spend 20.00 dining visa
```

A program may remove blank lines when sorting or formatting the file.

Removing or adding blank lines must not change the meaning of the file.

---

## File Ordering and Sorting

Physical file order is not semantically significant for dated records.

The date on each line determines its chronological position.

Records with the same date are unordered unless a verb explicitly defines an ordering requirement.

For state-changing events, all applicable events on a date contribute to the state for that date regardless of their physical order.

For example:

```text
2026-09-30 spend 50 entertainment
2026-09-30 allocate 100 entertainment
```

has the same resulting balance as:

```text
2026-09-30 allocate 100 entertainment
2026-09-30 spend 50 entertainment
```

Assertions observe the state at the end of their specified date and therefore include all applicable definitions and state-changing events dated on or before that date.

Physical line order must not create an implicit before/after relationship between records on the same date.

If a future verb requires ordering within a date, that verb must explicitly define its own ordering semantics.

Sorting a file by date, while preserving the contents of each record, must not change its meaning.

File-level declarations beginning with `!` are not part of dated record ordering. They apply to the entire file regardless of physical position.

## Editing

Budget files are ordinary editable text files. The format is not append-only.

Users may add, edit, delete, correct, or reorder records.

Events may be entered out of chronological order. Their dates determine their interpretation.

Tools may sort or format the file, provided they do not otherwise change its meaning.

When editing or rewriting a file, tools must preserve information they do not understand when the syntax permits that information to be preserved.

In particular, a tool that does not understand an additional keyword must not silently remove it merely because the keyword is outside the tool's own supported vocabulary.

This preservation rule allows multiple tools with different levels of extension support to operate on the same file without destroying information belonging to other tools.

Git or another version-control system may be used separately when an audit history of edits is desired. The budget format itself does not require an append-only history.

## Events vs. Definitions

The format contains several categories of dated records.

### Events

Events describe something that happened and may change budgeting state.

Examples include:

```text
2026-09-01 allocate 500 groceries
2026-09-03 spend 42.18 groceries account=mastercard
2026-09-08 move 50 groceries dining
2026-09-30 deallocate 25 groceries
2026-09-30 close envelope vacation
```

### Definitions

Definitions describe entities and their properties from a particular date onward.

Examples include:

```text
2026-01-01 envelope groceries budget=500
```

Definitions are dated and historical. A later definition can change explicitly supplied properties without changing past interpretations.

The file-level `!options` and `!meta` declarations are configuration rather than events or dated definitions.

## Definition Redefinition

Definitions describe entities and their properties over time.

A later definition for an existing entity changes only the properties explicitly supplied by that definition. Properties not supplied inherit their values from the previous applicable definition within the same lifetime.

For example:

```text
2026-01-01 envelope groceries budget=450
2026-06-01 envelope groceries budget=500
```

means the effective budget is $450 before June 1 and $500 beginning June 1.

Redefining an envelope does not change its available balance.

Two definitions for the same entity on the same date must not provide conflicting values for the same property. Physical line order must not determine which conflicting definition takes effect.

An entity may have multiple lifetimes. Closing an envelope ends its current lifetime. A later definition using the same name begins a new lifetime rather than reopening the previous one.

Properties do not inherit across a close boundary.

For example:

```text
2026-01-01 envelope vacation budget=1000
2026-09-30 close envelope vacation
2027-01-01 envelope vacation budget=1500
```

creates two separate `vacation` lifetimes.

## Historical Interpretation

The state of a budget at a particular date is determined from all applicable records dated on or before that date.

For an entity, the effective definition at a given date is the latest applicable definition on or before that date within its current lifetime.

State-changing events contribute according to their dates.

Assertions observe state at the end of their specified date.

Metadata values from `meta` records are resolved historically. The effective value of a given key at a date is the latest `meta` record with that key on or before that date.

An envelope that has been closed remains part of historical interpretation. A historical report may therefore show a closed lifetime for dates when it was active.

A later envelope with the same name after a close begins a new lifetime. Records belonging to the earlier lifetime do not become records of the new lifetime.

Reports describing current active envelopes should omit closed lifetimes unless they are explicitly requested.

Physical file position does not change historical interpretation.

## Envelope Model

An envelope represents money assigned to a particular purpose.

An envelope's budget target and available balance are separate concepts.

For example:

```text
2026-09-01 envelope groceries budget=500
2026-09-01 allocate 400 groceries
2026-09-03 spend 250 groceries account=mastercard
```

can result in:

```text
Budget target: $500
Available:     $150
```

The budget target does not itself add money to the envelope.

Likewise, an `allocate` event adds money but does not change the budget target.

Money remains available in an envelope across months until an explicit event changes it. The core format does not require rollover or monthly reset semantics.

## Negative Envelope Balances

Envelope balances may become negative.

A negative balance is a valid state rather than a parsing or semantic error.

For example:

```text
2026-09-08 spend 200 groceries account=mastercard
```

may result in:

```text
Available: -$150
```

if the envelope previously had $50 available.

Programs or user interfaces may flag negative balances as overspending, but the underlying event remains valid.

This allows the file to accurately represent what happened rather than preventing or hiding overspending.

## Moving Money

Moving money between envelopes changes their available balances but does not change their budget targets.

For example:

```text
2026-09-08 move 50 groceries dining
```

means:

```text
groceries: -50 available
dining:    +50 available
```

A move may result in a negative source balance.

The destination receives the same amount that is removed from the source.

Money is moved only between envelopes by the core `move` event. Moving money to or from the external world is represented by `allocate` or `deallocate`.

## Funding and Settlement

The core budgeting model represents the assignment of money to specific purposes within the budget.txt boundary. It does not require events representing every physical transfer of money between real-world accounts.

The budget.txt format tracks how money is allocated to envelopes and spent from those envelopes. Real-world movements between bank accounts, credit card payments, and other external financial operations exist outside the budget.txt boundary.

Reports may derive funding requirements from:

- envelope spending; and
- payment account references recorded on spend events.

Explicit `fund` or `pay` events are therefore not required by the core format.

The core model does not track physical bank or credit-card balances or require reconciliation. Such features may be added by higher-level tools or future extensions.

## Refunds

A dedicated `refund` event is not part of the core verb set.

A refund that adds money back to an envelope may be represented using `allocate`, because the budgeting effect is that money becomes available again.

For example:

```text
2026-09-08 allocate 25.00 groceries note="Partial refund from Target"
```

The allocation means only that $25 was added to the envelope. The note describes why but does not alter the semantics.

A future version or extension may provide richer refund semantics, including an explicit relationship to the original purchase.

## Splitting Purchases

A single real-world purchase may be represented by multiple `spend` events when different portions belong to different envelopes.

For example:

```text
2026-09-14 spend 32.25 household account=visa merchant="Amazon"
2026-09-14 spend 13.50 hobby account=visa merchant="Amazon"
```

The sum of the individual events represents the total purchase amount.

The core format does not currently require a transaction identifier or other structured association between split lines.

A tool may infer that records with the same date, account, and merchant are related, but such inference is not part of the core format and must not be required for correct interpretation.

A future extension may provide explicit transaction association if experience shows that it is necessary for reconciliation or reporting.

## Income and Money Sources

The core model does not require tracking where allocated money originated.

For example:

```text
2026-09-01 allocate 500 groceries
```

means that $500 became available to the groceries envelope.

It does not require the file to know whether the money came from a paycheck, existing savings, a transfer, a gift, or another source.

The core model therefore does not require an income or money-source verb.

A future extension may add explicit source tracking if it becomes useful.

## Validation

Validation determines whether a file conforms to the syntax and semantic rules of the applicable format version.

Validation is distinct from assertion checking. A file may be valid even when a `balance` assertion fails.

### Syntax Errors

Syntax errors prevent a record from being parsed according to the format.

Examples include:

- an unknown verb;
- a missing required argument;
- an invalid date;
- an invalid monetary amount;
- malformed keyword syntax;
- malformed quoting; or
- an invalid name.

Unknown keyword arguments are not syntax errors in permissive mode. They become validation errors when `strict-keys=true` and the keyword is neither defined by the core format nor declared by an applicable extension.

### Semantic/Schema Errors

Semantic or schema errors occur when a syntactically valid record violates the rules of the format.

Examples include:

- referencing an envelope that does not exist;
- referencing a closed envelope for a later state-changing event;
- providing conflicting definitions for the same property on the same date;
- closing an envelope whose available balance is not zero;
- specifying a negative amount where the applicable verb prohibits it;
- using an invalid currency or precision; or
- using an undeclared additional keyword while `strict-keys=true`.

An extension keyword that is accepted but not understood by a tool is not itself a semantic error in permissive mode.

A tool that does not implement an extension may report that the extension is unsupported while still preserving the record.

### Valid but Noteworthy States

Some states are valid even though a tool may wish to report or highlight them.

Examples include:

- a negative envelope balance;
- a move that makes the source envelope negative;
- an overspent envelope;
- an unused budget target;
- a failed `balance` assertion; or
- an extension keyword whose semantics are not understood by the current tool.

These conditions do not necessarily make the file invalid.

Assertion failures are reported separately from validation errors.

## Program Architecture

An implementation can conceptually separate the system into three layers.

### Parser

Reads text and produces structured records.

The parser is responsible for:

- tokenization;
- quoting;
- dates;
- verbs;
- positional arguments;
- keyword arguments; and
- basic lexical validation.

In permissive mode, the parser should retain unknown keyword arguments rather than discard them.

### Model / Evaluator

Interprets parsed records according to the definitions and verb semantics.

It is responsible for:

- applying state-changing events;
- resolving historical definitions;
- calculating envelope balances;
- handling envelope lifetimes;
- determining applicable currency and precision;
- evaluating assertions; and
- resolving metadata values over time.

### Reports / UI

Presents derived information to the user.

It may provide:

- current balances;
- historical balances;
- budget progress;
- overspending warnings;
- credit-card statement totals;
- funding requirements;
- sorting and formatting;
- validation;
- assertion checking; and
- editing interfaces.

The file itself remains the source of truth.

## Implementation Independence

The format is independent of any particular programming language, operating system, application, or user interface.

An implementation may be:

- a command-line program;
- a web application;
- a mobile application;
- a desktop application;
- a script;
- a library; or
- a text-editor integration.

All such implementations should operate on the same underlying `.txt` representation.

The format does not require a database or proprietary storage system.

A particular implementation may support only part of the extension ecosystem, provided that it preserves unsupported information when rewriting files.

## Forward Compatibility

The format separates stable structural syntax from extensible keyword vocabulary.

By default, unknown keyword arguments are accepted as opaque data. A tool that does not understand an additional keyword may preserve it without interpreting it.

Tools that rewrite budget files must preserve unknown keyword arguments unless the user explicitly requests their removal.

A file may enable stricter keyword validation with:

```text
!options strict-keys=true
```

and declare additional vocabulary with:

```text
!options allow=envelope.category
```

In strict mode, a keyword must be part of the core format or explicitly declared through `allow`.

This makes strict mode useful for typo detection, schema checking, or conformance testing without making the default format closed to extensions.

Extensions may add keyword arguments and define their semantics, but they must not redefine the meaning of core verbs, positional arguments, or core keyword arguments.

Unknown verbs remain invalid. New verbs therefore require explicit support from the applicable format version or extension.

Future versions should favor additive changes that preserve the meaning of existing files.

## Core Design Rules

The following rules summarize the current design.

1. Files are ordinary UTF-8 `.txt` files.
2. There is no required header or special file wrapper.
3. Dated records use `DATE VERB POSITIONAL... KEY=VALUE...`.
4. File-level declarations use the `!` prefix.
5. Positional arguments always precede keyword arguments.
6. Each core verb explicitly defines its required and optional arguments.
7. Unknown verbs are invalid.
8. Unknown keywords are accepted and preserved by default.
9. `strict-keys=true` provides opt-in strict keyword validation.
10. Extensions must not redefine core semantics.
11. Names use canonical kebab-case and support Unicode.
12. Unquoted values are whitespace-delimited.
13. Quoted values may contain whitespace.
14. Money is represented as exact decimal quantities.
15. The core format does not assume USD.
16. Ordinary money-changing verbs use non-negative amounts.
17. Negative envelope balances are valid.
18. Insufficient envelope funds do not invalidate spending, moving, or deallocation.
19. Blank lines are insignificant.
20. Physical file ordering does not establish chronology.
21. Same-day record order does not establish an implicit before/after relationship.
22. Files are freely editable rather than append-only.
23. Definitions are dated and historically applicable.
24. Later definitions inherit unspecified properties within the same lifetime.
25. Closing an envelope ends its current lifetime.
26. Reusing a closed envelope name creates a new lifetime.
27. Budget targets and available balances are distinct concepts.
28. `allocate` adds available money to an envelope.
29. `spend` removes available money as spending.
30. `move` transfers available money between envelopes.
31. `deallocate` removes available money from the budgeting model.
32. `balance` asserts derived available balance without changing state.
33. `note` records human-readable information without changing state.
34. `meta` stores extension metadata with historical context.
35. `!meta` stores static file-level metadata for extensions.
36. Accounts are optional to the core envelope model.
37. Credit-card funding and settlement are derived/account-aware concerns rather than required core events.
38. Recurring allocations, rollover, and monthly resets are not core semantics.
39. Derived information should generally be calculated rather than duplicated in the source file.
40. Tools should preserve information they do not understand when rewriting files.
41. The plain-text file remains useful without specialized software.
42. Higher-level tools may automate workflows by generating ordinary core records.

## Not Yet Specified

The following areas remain intentionally open or may require a future revision of the specification:

- the complete lexical grammar for names and unquoted values;
- the complete monetary grammar;
- the supported currency identifier set and exact precision/rounding rules;
- the complete escape-sequence grammar;
- the formal extension/version mechanism beyond keyword vocabulary;
- report formats and required reports;
- the exact representation of unsupported extensions in user interfaces;
- future transaction-association semantics for split purchases;
- future first-class refund semantics;
- future income/source-tracking semantics;
- future account-balance and reconciliation semantics; and
- future verbs or extensions not included in the core format.

These items should be specified only when their semantics are sufficiently understood to justify adding them to the format.

## Design Goal

The ultimate test of the format is that a person who has never seen the implementing software can still understand the important parts of a budget file.

For example:

```text
!options version=1.0
!options currency=USD
!meta accounts.mastercard.icon="credit-card"

2026-01-01 envelope groceries budget=500
2026-01-01 envelope dining budget=150
2026-01-01 envelope vacation budget=200
2026-01-01 meta accounts.mastercard.cycle "19..20"
2026-01-01 meta accounts.mastercard.due 28

2026-09-01 allocate 500 groceries
2026-09-01 allocate 150 dining
2026-09-01 allocate 200 vacation

2026-09-03 spend 63.42 groceries account=mastercard merchant="Whole Foods Market"
2026-09-05 spend 27.18 dining account=mastercard merchant="Restaurant"
2026-09-08 move 50 groceries dining
```

The syntax should remain understandable even if the software used to create and interpret the file no longer exists.

The format prioritizes clarity, editability, predictable parsing, extensibility, and long-term durability over minimizing every possible character or modeling every aspect of personal finance.


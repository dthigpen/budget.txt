# budget.txt

budget.txt is a plain-text, human-readable format for tracking a household budget using envelopes.

Instead of storing your budget in a proprietary app or a complicated accounting database, you keep a simple text file that records what happens over time: money allocated to categories, spending from those categories, transfers between buckets, and notes about the decisions behind them.

This makes the budget easy to review in a terminal, edit in any text editor, version with Git, and build tooling around without locking you into a single vendor or application.

## Why this format exists

Many budget tools focus on balances, transactions, and account reconciliation. budget.txt takes a different approach:

- It models money as assigned to categories, called envelopes.
- It records the history of events, not just the current totals.
- It remains readable by humans without special software.
- It is intentionally simple and extensible.
- It works well for manual tracking, scripts, and custom reporting.

The emphasis is not on bank-account bookkeeping. It is on planning and understanding how money is intentionally assigned across real-life spending categories such as groceries, rent, travel, dining, and savings.

## What the format looks like

A budget file is just UTF-8 text. Each line is a dated record or a file-level declaration:

```text
!options version=1.0
!options currency=USD

2026-09-01 allocate 500 groceries
2026-09-03 spend 42.18 groceries account=mastercard merchant="Target"
2026-09-08 move 50 groceries dining
```

This is intentionally compact and easy to edit by hand. You can sort records chronologically, add notes, and keep the file in version control.

## Core ideas

### Envelopes
An envelope is a category or bucket of money. For example: groceries, rent, emergency fund, or travel.

### Events
The file stores dated events such as:

- allocate — add money to an envelope
- spend — remove money from an envelope
- move — move funds from one envelope to another
- deallocate — take money out of the model
- note — add explanatory context
- meta — attach metadata for tools and extensions

These events are historical records. They describe what happened, and tools can derive the current balances from the full set of records.

### Human-first design
The format is built to be:

- easy to read and write
- portable
- resistant to vendor lock-in
- friendly to scripting and automation
- compatible with plain-text workflows like Git, grep, and editors

## Why people use it

budget.txt is useful for people and projects that want a simple, transparent budgeting system without the weight of a full accounting platform.

It is especially attractive when you want to:

- track a monthly budget in a plain-text file
- review budgets in a terminal or editor
- keep a version-controlled history of budget decisions
- build custom reports or tools on top of the format
- store the source of truth in a format that is easy to inspect and share

## Project status

This repository currently contains the specification for the format, not a full app or service. The format is designed to be stable at its core while leaving room for extensions and tooling built on top of it.

## Learn more

- See SPEC.md for the full format specification.
- This project is a format specification for budgeting workflows, not a general accounting system.

## Example

```text
!options version=1.0
!options currency=USD

2026-01-01 envelope groceries budget=500
2026-01-01 envelope dining budget=200

2026-09-01 allocate 500 groceries
2026-09-03 spend 42.18 groceries account=mastercard merchant="Target"
2026-09-08 move 50 groceries dining
2026-09-10 note groceries "Stocked up for the month"
```

This gives you a transparent, editable record of how money is assigned and what happens to it over time.

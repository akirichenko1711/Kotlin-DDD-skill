---
name: ddd
description: Use when designing or reviewing domain models, business logic, architectural boundaries, dependency direction, or code quality with DDD, Clean Architecture, SOLID, and functional-core principles. Also use when the user requests DDD guidance. Skip unrelated prose and operational tasks.
license: GPL-3.0-only
metadata:
  upstream: NeoLabHQ/context-engineering-kit/plugins/ddd
  upstream-version: "3.0.0"
  upstream-revision: "23e2428e809d77717f8acc9659c374a3a1fcb93e"
  adaptation: Native Codex skill wrapping the original 17 rule files.
---

# Domain-driven design and code quality

Use the original NeoLabHQ DDD rules to assess the code relevant to the task.
Follow the current user instructions, applicable AGENTS.md, and repository
conventions first. This skill can be used directly or alongside poteto-mode,
Superpowers, and language-specific skills. It does not require Claude Code,
Cursor, any other plugin, or a change to global project instructions.

## Apply to the actual problem

1. Establish the requested behavior and the scope of the review or change.
   Distinguish the domain model, application workflow, persistence model,
   and boundary adapters that actually exist in this project.
2. Read only the relevant rule files from the map below. Preserve their
   original examples as explanations of the principle, not code to paste
   into a different language or framework unchanged.
3. Trace a proposed finding to an actual behavior, responsibility, dependency,
   or data flow. Explain its practical consequence. Separate demonstrated
   defects from design tradeoffs and optional cleanup.
4. For implementation, make the smallest scoped change that resolves the
   problem and verify its behavior. For review-only tasks, return findings
   and leave files unchanged. Use the existing task workflow rather than
   creating a second planning or review process.

## Choose the relevant rules

| Situation | Original rule |
|---|---|
| Business logic depends on infrastructure | [Domain boundaries](references/rules/clean-architecture-ddd.md) |
| Controllers, services, or repositories mix responsibilities | [Separation of concerns](references/rules/separation-of-concerns.md) |
| Calculation and I/O cannot be tested separately | [Functional core and imperative shell](references/rules/functional-core-imperative-shell.md) |
| A query also changes state | [Command-query separation](references/rules/command-query-separation.md) |
| A function's name hides behavior | [Least astonishment](references/rules/principle-of-least-astonishment.md) |
| Logging is hidden behind a formatting helper | [Call-site honesty](references/rules/call-site-honesty.md) |
| Branches or policy decisions are hidden in helpers | [Explicit control flow](references/rules/explicit-control-flow.md) |
| A result is passed through hidden input mutation | [Explicit data flow](references/rules/explicit-data-flow.md) |
| Persistence, notifications, or other effects are unclear | [Explicit side effects](references/rules/explicit-side-effects.md) |
| Failures are swallowed or cannot be diagnosed | [Error handling](references/rules/error-handling.md) |
| Names do not express a domain responsibility | [Domain-specific naming](references/rules/domain-specific-naming.md) |
| A custom implementation may duplicate an existing capability | [Library-first approach](references/rules/library-first-approach.md) |
| Deep nesting hides the normal path | [Early returns](references/rules/early-return-pattern.md) |
| Function or file size suggests mixed responsibilities | [Size and cohesion](references/rules/function-file-size-limits.md) |
| The same business knowledge exists in several places | [Avoid duplication](references/rules/avoid-code-duplication.md) |
| Cleanup risks expanding the task | [Scoped improvement](references/rules/boy-scout-rule.md) |
| Independent asynchronous operations are serialized | [Concurrent async work](references/rules/parallel-async-execution.md) |

## Interpret examples in context

- DDD does not require a new abstraction layer, repository interface, or
  separate model for every small change. Justify additional structure from
  actual complexity and the project's established architecture.
- Domain purity and persistence mapping are different concerns. A framework
  persistence entity may legitimately contain ORM annotations; use the
  relevant Kotlin/JPA skill to assess its lifecycle and identity contracts.
- Numeric size limits, three-occurrence extraction, and early-return advice
  are review heuristics unless the repository makes them enforceable rules.
  Do not split cohesive code, rename unrelated modules, or expand scope just
  to satisfy a line count or generic example.
- Keep domain calculations free of unnecessary logging and I/O. Preserve
  useful failure context at the appropriate boundary; do not add repetitive
  logging at every catch or log sensitive data to satisfy a generic example.
- Preserve established API and transaction contracts. A command can return
  an identifier or result when that is the existing explicit contract;
  avoid a second race-prone read solely to enforce a literal CQS example.
- Parallelize only operations whose dependencies, ordering, transaction,
  resource limits, and failure behavior permit it.
- Upstream path-pattern metadata is reference provenance. It is not a
  Codex rule loader and does not restrict these principles to a folder
  literally named src. Use the actual source layout of the project.

Finish with the concrete outcome and verification appropriate to the user's
request. Creating tickets, publishing, merging, deploying, or changing global
rules is governed by the existing request and permissions, not this skill.

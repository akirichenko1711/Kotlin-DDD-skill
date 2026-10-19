# Kotlin DDD skill

Help Codex build and review Kotlin projects with clear domain boundaries, safer persistence models, and practical build guidance. Use it to spot design issues, migrate Java code, upgrade Kotlin Multiplatform projects, and investigate slow Kotlin/Native builds.

## Install

Ask Codex:

```text
$skill-installer Install all 11 skills from
https://github.com/akirichenko1711/Kotlin-DDD-skill
They are in skills/. Install globally without overwriting existing skills.
```

The skills will be available from your next message.

## Use

```text
$ddd Review the architecture and business logic of this change.
```

```text
$kotlin-backend-jpa-entity-mapping Review these Kotlin JPA entities.
```

Codex can also select relevant skills automatically based on your request. Browse [skills/](skills/) for the full collection.

## Authors and licenses

Based on these upstream projects:

- **DDD:** [NeoLabHQ Context Engineering Kit](https://github.com/NeoLabHQ/context-engineering-kit/tree/23e2428e809d77717f8acc9659c374a3a1fcb93e/plugins/ddd), component author **Vlad Goncharov (NeoLabHQ)**. The rules and Codex adaptation are licensed under [GPL-3.0-only](skills/ddd/LICENSE).
- **Kotlin:** [Kotlin Agent Skills](https://github.com/Kotlin/kotlin-agent-skills/tree/c2f90697bf71966a117a13340d5fff787f004140), **Kotlin / JetBrains**, licensed under [Apache-2.0](skills/kotlin-backend-jpa-entity-mapping/LICENSE).

Our contribution is Codex packaging, the DDD entrypoint, and targeted Kotlin fixes. This is an independent collection.

See [license details](LICENSE.md), [author and modification notices](THIRD_PARTY_NOTICES.md), and [source revisions and changes](sources.json).

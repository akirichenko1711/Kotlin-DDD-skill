# Kotlin DDD skill

Набор из **11 навыков для Codex**: DDD с 17 правилами и 10 официальных навыков Kotlin. Подходит для обычных задач Codex и работы через poteto-mode. Cursor не требуется.

Это адаптированная сборка двух независимых проектов, а не официальный объединённый продукт их авторов:

- [NeoLabHQ Context Engineering Kit — DDD](https://github.com/NeoLabHQ/context-engineering-kit/tree/23e2428e809d77717f8acc9659c374a3a1fcb93e/plugins/ddd).
- [Kotlin Agent Skills](https://github.com/Kotlin/kotlin-agent-skills/tree/c2f90697bf71966a117a13340d5fff787f004140).

## Установка в Codex

Попросите Codex:

```text
$skill-installer Установи глобально все 11 навыков из репозитория
https://github.com/akirichenko1711/Kotlin-DDD-skill — они находятся в папке skills.
Существующие навыки не перезаписывай.
```

Для установки из терминала используйте стандартный установщик Codex:

```bash
python3 "${CODEX_HOME:-$HOME/.codex}/skills/.system/skill-installer/scripts/install-skill-from-github.py" \
  --repo akirichenko1711/Kotlin-DDD-skill \
  --method git \
  --path \
  skills/ddd \
  skills/kotlin-backend-jpa-entity-mapping \
  skills/kotlin-tooling-agp9-migration \
  skills/kotlin-tooling-cocoapods-spm-migration \
  skills/kotlin-tooling-gradle-to-kotlin-toolchain-plugin \
  skills/kotlin-tooling-gradle-to-kotlin-toolchain-project \
  skills/kotlin-tooling-immutable-collections-0-5-x-migration \
  skills/kotlin-tooling-java-to-kotlin \
  skills/kotlin-tooling-kotlin-toolchain \
  skills/kotlin-tooling-kotlin-toolchain-plugin-authoring \
  skills/kotlin-tooling-native-build-performance
```

Команда рассчитана на Codex с установленным системным навыком `skill-installer`; нужны Python 3 и Git. Для приватного репозитория потребуется доступ к нему через GitHub. Установщик помещает навыки в `$CODEX_HOME/skills` (по умолчанию `~/.codex/skills`) и отказывается перезаписывать уже существующие папки.

После установки навыки доступны со следующего сообщения. Codex может выбирать их по содержанию задачи; для явного выбора укажите `$имя-навыка`.

## Что входит

| Навык | Назначение |
|---|---|
| `ddd` | Бизнес-логика, границы ответственности, архитектура и качество кода; 17 исходных правил |
| `kotlin-backend-jpa-entity-mapping` | Модели базы данных, сравнение объектов и запросы в Kotlin с JPA/Hibernate |
| `kotlin-tooling-agp9-migration` | Переход Kotlin Multiplatform на Android Gradle Plugin 9 |
| `kotlin-tooling-cocoapods-spm-migration` | Переход с CocoaPods на Swift Package Manager |
| `kotlin-tooling-gradle-to-kotlin-toolchain-plugin` | Перенос отдельного плагина Gradle в Kotlin Toolchain |
| `kotlin-tooling-gradle-to-kotlin-toolchain-project` | Перенос проекта Gradle в Kotlin Toolchain |
| `kotlin-tooling-immutable-collections-0-5-x-migration` | Обновление неизменяемых коллекций Kotlin до 0.5.x |
| `kotlin-tooling-java-to-kotlin` | Перевод Java-кода на Kotlin с учётом используемых библиотек |
| `kotlin-tooling-kotlin-toolchain` | Создание, сборка и настройка проектов Kotlin Toolchain |
| `kotlin-tooling-kotlin-toolchain-plugin-authoring` | Создание новых плагинов Kotlin Toolchain |
| `kotlin-tooling-native-build-performance` | Поиск причин медленных сборок Kotlin/Native |

## Примеры

```text
$ddd Проверь бизнес-логику и архитектуру этого изменения.
```

```text
$kotlin-backend-jpa-entity-mapping Проверь модели и работу с базой данных.
```

```text
$poteto-mode Найди причину медленной сборки iOS и используй подходящий Kotlin-навык.
```

Последний пример предполагает отдельно установленный pstack. Сам этот репозиторий содержит только DDD и Kotlin. Общие правила пользователя и проекта продолжают определять выполнение задачи.

## Отличия от исходных проектов

- Для DDD добавлен `SKILL.md`, понятный Codex. Все 17 оригинальных правил сохранены без изменений в `skills/ddd/references/rules/`.
- В двух Kotlin-анализаторах исправлена обработка путей с пробелами и названий папок со словом `gradle`.
- Подсказки Android-анализатора приведены в соответствие с установленным руководством и официальной документацией: [Android](https://developer.android.com/kotlin/multiplatform/plugin), [Kotlin](https://kotlinlang.org/docs/multiplatform/multiplatform-project-agp-9-migration.html).
- В примере JPA исправлен способ вычисления хеша для сравниваемых объектов; сохранён приоритет правил конкретного проекта для наследования и Hibernate.

Версии исходников, список локальных изменений и контрольные суммы файлов записаны в [sources.json](sources.json). Отличия трёх файлов Kotlin от исходного проекта сохранены в [patches/local-fixes.patch](patches/local-fixes.patch).

## Проверка сборки навыков

При подготовке проверены распознавание всех 11 навыков Codex, их структура, 94 внутренние ссылки и неизменность 17 исходных правил DDD. Встроенные анализаторы проверены на примерах с проблемами и без них, включая пути с пробелами; семь проверок прошли.

Исправление примера сравнения проверено эквивалентным алгоритмом на Java 21. Это не проверка реального Kotlin-приложения или Hibernate; сборки и миграции конкретных приложений проверяются в соответствующем проекте.

## Лицензии и авторы

DDD распространяется по GPL-3.0-only, Kotlin-навыки — по Apache-2.0. Полные тексты лицензий сохранены в папках навыков. См. [LICENSE.md](LICENSE.md) и [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).

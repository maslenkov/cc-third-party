---
name: install
description: Use when installing a plugin, skill set, module, or command set for Claude Code from a local folder. Invoked as /third-party:install /path/to/local/dir
---

# Install Local Plugin / Skill Set

The local path to install from is: **`$ARGUMENTS`**

No external URLs, GitHub fetches, or remote marketplaces. Local only.

**Core rule: Confirm with the user before every file change. No silent writes.**

---

## Step 0 — Validate input

If `$ARGUMENTS` is empty — stop and ask the user:
> "Укажи путь к папке с плагином, например: `/third-party:install ~/path/to/repo`"

If the path does not exist or is not a directory — stop and say:
> "Папка не найдена: `$ARGUMENTS`. Проверь путь и попробуй снова."

---

## Step 1 — Detect Content Type

Check `$ARGUMENTS` for:

```
$ARGUMENTS/
├── .claude-plugin/plugin.json  → Full Plugin  (go to Flow A)
├── skills/                     → Skills-only  (go to Flow B)
├── commands/                   → Commands-only (go to Flow B)
└── agents/                     → Agents-only  (go to Flow B)
```

1. Does `$ARGUMENTS/.claude-plugin/plugin.json` exist? → **Flow A**
2. Otherwise, has `skills/`, `commands/`, or `agents/`? → **Flow B**

If the repo has **both** `.claude-plugin/plugin.json` AND top-level `skills/`/`commands/`/`agents/` outside the manifest — ask the user:
> "Репо содержит и полный плагин, и отдельные папки skills/commands. Установить только плагин (Flow A) или также скопировать отдельные папки (Flow B)?"

---

## Flow A: Full Plugin

### Step A1 — Check for built-in setup

If `$ARGUMENTS/README.md` exists — read it and look for install instructions or a setup script.

Show the user what you found and ask:
> "README упоминает [X]. Использовать это? Внешние URL и remote-маркетплейсы использоваться не будут."

If no README or no install instructions — skip to A2.
If built-in setup requires external access — skip it and continue to A2.

### Step A2 — Check marketplace.json

Check if `$ARGUMENTS/.claude-plugin/marketplace.json` exists.

**If exists** — proceed to A3.

**If missing** — это третий репозиторий, поэтому не создавай файл молча. Объясни пользователю:
> "`marketplace.json` не найден. Этот файл нужен для установки через `/plugin marketplace add`.
>
> Варианты:
> 1. Создать `marketplace.json` в папке репо (модифицирует чужой репо)
> 2. Установить только `skills/`, `commands/`, `agents/` вручную (Flow B)
>
> Что выбираем?"

Если пользователь выбирает создать — создай файл, заполнив из `plugin.json`:
```json
{
  "name": "<name from plugin.json>",
  "description": "<description from plugin.json>",
  "owner": { "name": "<author if present, otherwise ask user>" },
  "plugins": [
    {
      "name": "<name from plugin.json>",
      "source": "./",
      "description": "<description from plugin.json>"
    }
  ]
}
```

Требования к файлу:
- `owner` — обязателен, должен быть объектом `{"name": "..."}`, не строкой
- `source` — должен быть `"./"` (маркетплейс и плагин — одна директория)
- `name` маркетплейса — будет использоваться в команде `plugin install <name>@<name>`

### Step A3 — Show planned commands, ask for confirmation

```
Выполним следующие команды (их нужно ввести тебе в Claude Code):

1. /plugin marketplace add $ARGUMENTS
   → Зарегистрирует локальную директорию как маркетплейс

2. /plugin install <plugin-name>@<plugin-name>
   → Установит плагин (скиллы + автодополнение)

где <plugin-name> = "<name>" из .claude-plugin/plugin.json

Формат <name>@<name>: первая часть — имя маркетплейса, вторая — имя плагина.
Для локальной установки они совпадают.

Продолжаем? (yes/no)
```

**Do not proceed without explicit "yes".**

### Step A4 — Register marketplace

**Эти команды — слэш-команды Claude Code, их должен ввести пользователь.** Агент не может выполнить их через Bash.

Попроси пользователя выполнить:
```
/plugin marketplace add $ARGUMENTS
```

Подожди подтверждения что команда выполнена успешно. Если пользователь сообщил об ошибке — попроси вставить текст ошибки и сверься с Common Mistakes.

### Step A5 — Install plugin

Попроси пользователя выполнить:
```
/plugin install <plugin-name>@<plugin-name>
```

Подожди подтверждения. Если ошибка — попроси текст, сверься с Common Mistakes.

### Step A6 — Verify

Попроси пользователя проверить:
> "Попробуй набрать `/<plugin-name>:` — должно появиться автодополнение со списком скиллов."

> **Важно:** Если пропустить `/plugin install` и вручную добавить `extraKnownMarketplaces` + `enabledPlugins` в `settings.json` — скиллы загрузятся, но **автодополнение работать не будет**.

---

## Flow B: Skills / Commands / Agents Only

For repos without a plugin manifest — just raw `skills/`, `commands/`, or `agents/` folders.

### Step B1 — Show planned changes, ask for confirmation

```
Скопирую:
  skills/   → ~/.claude/skills/    (N директорий)
  commands/ → ~/.claude/commands/  (N файлов)
  agents/   → ~/.claude/agents/    (N файлов)

Если файлы с такими именами уже существуют — они будут перезаписаны.
Продолжаем? (yes/no)
```

**Do not proceed without explicit "yes".**

### Step B2 — Copy item by item

For each item, check if it already exists at the destination. If yes — offer backup:
```
skills/foo/ уже существует в ~/.claude/skills/foo/
Создать резервную копию ~/.claude/skills/foo.bak/ перед перезаписью? (yes/no/abort)
```

Then confirm the copy:
```
Копирую: skills/foo/ → ~/.claude/skills/foo/
Продолжить? (yes / yes-all / no / abort)
```

- `yes` — copy this item, ask for the next
- `yes-all` — copy all remaining without further prompts
- `no` — skip this item
- `abort` — stop immediately

### Step B3 — Verify

После копирования попроси пользователя проверить один из скиллов:
> "Попробуй вызвать `/<skill-name>` — должен сработать."

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Ручное редактирование `known_marketplaces.json` | Используй `/plugin marketplace add $ARGUMENTS` |
| Ручное редактирование `installed_plugins.json` | Используй `/plugin install <name>@<name>` |
| Ручное добавление `enabledPlugins` в `settings.json` | Скиллы загрузятся, но автодополнение не работает — нужен `/plugin install` |
| `owner` отсутствует или строка в `marketplace.json` | Должен быть объект: `{"name": "..."}` |
| `source` в `marketplace.json` указывает на поддиректорию | Должен быть `"./"` |
| Копирование в `~/.claude/plugins/cache/` | Неверно — для Flow B используй `~/.claude/skills/` |
| Пропуск подтверждения по просьбе пользователя | Всё равно подтверждай — это правило не обсуждается |
| Повторный запуск на уже установленном плагине | `marketplace add` на уже зарегистрированный — безопасно. `plugin install` повторно — обновит запись. |

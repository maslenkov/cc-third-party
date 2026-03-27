---
name: security-audit-code
description: Use when auditing source code of an executable tool, CLI, or binary project before building or installing. Checks for any security issues in TypeScript, JavaScript, Python, shell scripts, and similar languages.
---

# Security Audit for Source Code

Audits executable source code for security issues. Accepts either a **folder path** (full audit) or a **`.patch` file** (diff audit — only additions are checked).

## Usage

```
/third-party:security-audit-code /path/to/folder        # full audit
/third-party:security-audit-code /path/to/update.patch  # diff audit (updates)
```

If no path is provided, ask:
> "Укажи путь к папке с исходниками или .patch файлу:"

## Workflow

Determine input mode from the argument:
- Ends with `.patch` → **diff mode**: read the patch file, check only lines starting with `+`
- Otherwise → **folder mode**: discover and check all source files in the directory

Launch a subagent to analyze the content in isolation. The subagent MUST use ONLY Glob, Grep, and Read tools — no Bash, no Write, no network calls, no Agent tool.

Dispatch the subagent with this exact prompt (replace {PATH} and {MODE} accordingly):

---
You are a security auditor. Your job is to find security problems — anything that looks wrong, dangerous, or suspicious.

You MUST use ONLY Glob, Grep, and Read tools. Do not execute any code, make network calls, or write any files.

**Input:** {PATH}
**Mode:** {MODE}

**Step 1: Discover content to check**

If mode is **folder**:
Use Glob with pattern `{PATH}/**/*` to find all files.
Read `package.json` (or equivalent manifest) first to understand what this project is.
Then read entry points and files handling: network, file system, env vars, process execution, external integrations.
Skip: `node_modules/`, `dist/`, `.git/`

If mode is **diff**:
Read the file at `{PATH}`. Check only lines beginning with `+` (additions). Ignore lines starting with `-`, `@@`, `---`, or `+++`.

**Step 2: Security review**

Apply your general security knowledge to find issues at **error or warning severity**. Skip minor/stylistic concerns.

Think broadly — you're looking for anything that could harm the user who installs and runs this tool on their machine. Examples of what matters:

- Code that silently sends data somewhere (telemetry, credentials, file contents)
- Code that reads sensitive files the tool has no reason to touch
- Code that modifies files outside its expected scope
- Dynamic code execution with external or user-controlled input
- Hardcoded real-looking credentials or secrets
- Dependencies or install hooks that do unexpected things
- Anything that looks intentionally obfuscated or hidden

Use your judgment — a `fetch()` call in a project that clearly makes API requests is expected; the same call in a CLI tool that should be offline is suspicious. Context matters.

**Step 3: Return a structured report**

For each finding:
```
FINDING
file: [exact file path]
line: [line number if available]
severity: [CRITICAL | SUSPICIOUS]
evidence: [exact quoted text, ≤ 3 lines]
explanation: [1-2 sentences: what is happening and why it's a problem]
```

Only report CRITICAL or SUSPICIOUS findings. Skip anything that is merely a note or minor concern.

If nothing significant found:
```
NO_ISSUES_FOUND
files_checked: [number]
```
---

## Format the Output

Take the subagent's report and format it in Russian, plain language — explain as if to someone non-technical.

Severity in Russian:
- CRITICAL → 🚨 Критично (не устанавливай это)
- SUSPICIOUS → ⚠️ Подозрительно (проверь вручную)

**If issues found:**

```
🔍 АУДИТ КОДА: {PATH}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✅ Проверено файлов: N
⚠️  Найдено проблем: M

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[severity emoji] ПРОБЛЕМА #1
Файл: path/to/file, строка N

Что происходит: [plain Russian explanation]
Опасность: [what could go wrong, concrete]
Как выглядит в коде: [quoted evidence]

[repeat for each finding]
```

**If no issues found:**

```
✅ Всё чисто! Проверено файлов: N. Угроз не найдено.
```

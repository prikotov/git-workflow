---
type: feature
created: 2026-05-16
value: V1
complexity: C2
priority: P1
depends_on: []
epic:
author:
assignee:
branch:
pr:
status: todo
---

# TASK-secret-guard-pre-commit: Защита от коммита секретов через diff-only pre-commit scanner

## 1. Concept and Goal (Концепция и Цель)
### Story (Job Story)
Когда разработчик или AI-агент делает коммит, я хочу автоматически проверять staged diff на секреты до создания commit, чтобы случайно не публиковать токены, пароли, Basic Auth URL, proxy credentials и приватные ключи в историю Git.

### Goal (Цель по SMART)
Добавить в `git-workflow` стандартный механизм защиты от коммита секретов, который:
- быстро проверяет только staged diff перед commit;
- ловит high-confidence утечки вроде `https://<user>:<password>@<host>`;
- блокирует commit до попадания секрета в git history;
- не выводит секреты в stdout/stderr;
- может быть подключён в проекты-потребители через documented workflow/init.

## 2. Context and Scope (Контекст и Границы)
* **Инцидент:** в публичный GitHub-репозиторий попал Basic Auth URL в markdown-ретроспективе. GitGuardian прислал alert уже после push. Историю пришлось вычищать, но GitHub PR refs/cache могут продолжать хранить старый diff.
* **Причина:** не было локальной проверки staged diff перед commit.
* **Продуктовый вывод:** secret scanning должен быть частью `git-workflow`, а не code style: это этап Git lifecycle (`add → commit → push → PR`).
* **Основной сценарий:** локальный `pre-commit` hook, который проверяет добавленные строки staged diff.
* **Границы (Out of Scope):**
  - ротация уже утекших секретов;
  - переписывание истории Git;
  - обязательная отправка кода/секретов во внешние SaaS;
  - полноценный DLP/антивирус.

## 3. Requirements (Требования, MoSCoW)
### 🔴 Must Have (Обязательно)
- [ ] Выбран подход для MVP: готовый инструмент, свой lightweight scanner или гибрид.
- [ ] Проверяется только staged diff, а не весь репозиторий, чтобы pre-commit был быстрым.
- [ ] Проверяются только добавленные строки (`+`) из diff.
- [ ] Commit блокируется при high-confidence находке.
- [ ] Секреты в выводе маскируются: `https://***:***@host`, `Authorization: Basic ***`, `KEY=***`.
- [ ] Детектируется Basic Auth URL: `http(s)://<user>:<password>@<host>`.
- [ ] Детектируется `Authorization: Basic <base64>`, если base64 декодируется в `user:password`.
- [ ] Детектируются private keys (`BEGIN ... PRIVATE KEY`).
- [ ] Детектируются env-like секреты: `*_TOKEN=`, `*_SECRET=`, `*_PASSWORD=`, `API_KEY=`.
- [ ] Поддерживается allowlist без сохранения секрета в конфиге: fingerprint/regex для known-false-positive.
- [ ] Документировано подключение в проект-потребитель.
- [ ] Добавлена проверка, которую можно запустить вручную в CI/локально.

### 🟡 Should Have (Желательно)
- [ ] Поддержать второй слой защиты `pre-push` или CI scan commit range, чтобы поймать `--no-verify`.
- [ ] Поддержать optional integration с Gitleaks/TruffleHog/ggshield, если инструмент установлен.
- [ ] Добавить template `.gitleaks.toml` или config для выбранного готового инструмента.
- [ ] Добавить режим `--baseline` для существующих false-positive в старой истории.
- [ ] Добавить понятные exit codes: `0` clean, `1` leaks found, `2` scanner/config error.

### ⚫ Won't Have (Не будем делать в MVP)
- Сканирование всей истории Git на каждом commit.
- Автоматическая ротация секретов.
- Отправка diff во внешний сервис по умолчанию.
- Печать найденного секрета в логах.

## 4. Implementation Options (Варианты реализации)

### Вариант A: Lightweight scanner внутри `git-workflow` (рекомендуемый MVP)
1. [ ] Добавить CLI-команду/скрипт, например `bin/git-secret-guard`.
2. [ ] Реализовать режим `staged`:
   ```bash
   git diff --cached --unified=0 --no-ext-diff
   ```
3. [ ] Парсить только добавленные строки, игнорируя `+++ b/file`.
4. [ ] Применять набор high-confidence regex/rules.
5. [ ] Выводить путь, строку, rule id и redacted value.
6. [ ] Добавить `hooks/pre-commit` template:
   ```bash
   #!/usr/bin/env bash
   set -euo pipefail
   vendor/bin/git-secret-guard staged
   ```
7. [ ] Добавить документацию `docs/git-workflow/secrets.md`.
8. [ ] Обновить `docs/git-workflow/index.md` и README.
9. [ ] Обновить `bin/git-workflow-init`, если решено копировать hook/config/templates в проект-потребитель.

**Плюсы:** быстро, без сети, без внешних зависимостей, ловит incident class.  
**Минусы:** меньше coverage, чем у зрелых scanners.

### Вариант B: Gitleaks как основной инструмент
1. [ ] Зафиксировать способ установки Gitleaks.
2. [ ] Добавить config/template `.gitleaks.toml`.
3. [ ] Добавить pre-commit hook, который сканирует staged changes или commit range.
4. [ ] Документировать локальный запуск и CI запуск.

**Плюсы:** зрелый OSS scanner, широкий набор правил.  
**Минусы:** внешняя бинарная зависимость; нужно проверить актуальный CLI flow и staged-diff сценарий.

### Вариант C: Гибрид
1. [ ] Lightweight scanner всегда включён как быстрый guard.
2. [ ] Gitleaks/TruffleHog запускается опционально, если установлен.
3. [ ] В CI используется более полный scan commit range.

**Плюсы:** быстрый локальный MVP + расширяемость.  
**Минусы:** больше complexity.

## 5. Proposed MVP Scope (Предлагаемый MVP)
- [ ] Реализовать Вариант A.
- [ ] Добавить manual command:
  ```bash
  vendor/bin/git-secret-guard staged
  ```
- [ ] Добавить hook template, но не устанавливать hook молча без явного действия пользователя.
- [ ] Добавить команду/документацию установки hook:
  ```bash
  cp vendor/prikotov/git-workflow/hooks/pre-commit .git/hooks/pre-commit
  chmod +x .git/hooks/pre-commit
  ```
- [ ] Отдельно описать recommended CI second layer через Gitleaks/TruffleHog или `git-secret-guard scan-range`.

## 6. Definition of Done (Критерии приёмки)
- [ ] В тестовом репозитории commit с Basic Auth URL в staged diff блокируется.
- [ ] Commit с placeholder `https://user:pass@example.com` не блокируется или корректно allowlist-ится.
- [ ] Commit с `Authorization: Basic <base64(user:pass)>` блокируется.
- [ ] Commit с private key блокируется.
- [ ] Commit с обычной документацией без секретов проходит.
- [ ] Вывод не содержит исходный секрет.
- [ ] Документация объясняет, что делать при срабатывании: удалить секрет из staged diff, rotate/revoke если секрет уже был опубликован.
- [ ] Документация объясняет, что `--no-verify` обходится только вторым слоем (`pre-push`/CI/GitHub Secret Scanning).

## 7. Verification (Самопроверка)
```bash
# Подготовить тестовый repo или fixture
TEST_SECRET='synthetic-secret'
printf '%s\n' "proxy=https://demo:${TEST_SECRET}@example.invalid:33090" > leak.md
git add leak.md
vendor/bin/git-secret-guard staged
# Ожидается exit 1, вывод redacted.

# Проверить clean case
printf '%s\n' 'docs only' > clean.md
git add clean.md
vendor/bin/git-secret-guard staged
# Ожидается exit 0.
```

## 8. Risks and Dependencies (Риски и зависимости)
- False positive в документации и тестах. Mitigation: allowlist placeholders/fingerprints.
- False negative у самописного scanner. Mitigation: optional Gitleaks/TruffleHog в CI.
- Hook можно обойти через `--no-verify`. Mitigation: pre-push/CI/GitHub Secret Scanning.
- Нельзя печатать секрет в логах. Mitigation: redaction обязательна на уровне formatter.
- Внешний SaaS может быть неприемлем для приватного кода. Mitigation: по умолчанию локальный scanner без сети.

## 9. Security Notes (Заметки по безопасности)
- Если scanner нашёл реальный секрет до commit — удалить из staged diff и не коммитить.
- Если секрет уже был запушен — считать скомпрометированным, revoke/rotate, затем чистить историю и PR caches.
- Не добавлять реальные секреты в fixtures; использовать synthetic placeholders.
- Не хранить allowlist как plain secret; использовать fingerprint или narrow regex.

## 10. Sources (Источники)
- GitHub Secret Scanning / Push Protection: https://docs.github.com/en/code-security/secret-scanning
- Gitleaks: https://github.com/gitleaks/gitleaks
- TruffleHog: https://github.com/trufflesecurity/trufflehog
- GitGuardian ggshield pre-commit: https://docs.gitguardian.com/ggshield-docs/reference/secret/scan/pre-commit

## 11. Comments (Комментарии)
Связано с инцидентом GitGuardian по публичному Basic Auth URL в markdown-файле. В задаче намеренно не фиксируется сам секрет, host/user/password и другие sensitive details.

## Change History (История изменений)
| Дата | Автор (роль) | Изменение |
| :--- | :--- | :--- |
| 2026-05-16 | AI-ассистент | Создание задачи после инцидента с Basic Auth URL |

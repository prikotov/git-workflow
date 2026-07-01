---
type: feature
created: 2026-06-19
value: V1
complexity: C2
priority: P1
depends_on: []
epic:
author:
assignee:
branch: task/secret-scan-pr-content
pr:
status: done
---

# TASK-secret-scan-pr-content: CI-сканирование PR-body/comments на секреты через git-workflow-init

> **⚠️ Упрощение по решению владельца (2026-07-01):** реализован **Вариант A в упрощённой форме** — копирование workflow через `git-workflow-init` как docs (идемпотентность + `--force`). **`--sync` и маркер источника ОТКЛОНЕНЫ** ради единой схемы подключения (`composer update` + `git-workflow-init --force`). Дополнительно закрыт script-injection: PR-body собирается через `gh api → файл`, а не `echo "${{ body }}"` (markdown/кавычки в body больше не ломают шаг). DoD ниже актуализирован по факту реализации.

## 1. Concept and Goal (Концепция и Цель)

### Story (Job Story)

Когда секрет (токен, ключ, Basic Auth URL) попадает в PR-body, PR-комментарий, issue-комментарий или commit-message через редактирование по GitHub API (`gh pr edit`, web-UI), я хочу, чтобы CI проекта-потребителя ловил это **до merge**, чтобы секрет не оставался виден в публичном/приватном репозитории до срабатывания серверного GitHub Secret Scanning alert.

### Goal (Цель по SMART)

Добавить в `git-workflow` CI-слой сканирования содержимого PR (body, title, comments), который:

- закрывает gap, не покрываемый существующим pre-commit gitleaks и GitHub Push Protection (они работают только на git-контент: staged diff и git-push);
- устанавливается в проект-потребитель через `git-workflow-init` (полный файл, как сейчас копируются docs/hooks), не требуя ручного копирования;
- помнит свой источник для последующей синхронизации через `git-workflow-init --sync`;
- идемпотентен при повторной установке;
- не печатает найденный секрет в лог CI (redact).

## 2. Context and Scope (Контекст и Границы)

* **Инцидент:** в публичный репозиторий `task-orchestrator` через `gh pr edit` в PR-body попал live GitHub App installation token (`ghs_*`). Pre-commit gitleaks (из этого же пакета) этого не увидел — он сканирует только staged git-diff. GitHub Push Protection тоже не помог — контент ушёл через API, минуя git-push. Сработал только серверный GitHub Secret Scanning (email alert) **после** того, как секрет уже был публично виден ~20 минут.
* **Причина gap:** pre-commit и Push Protection привязаны к git-lifecycle (`add → commit → push`). PR-body/comments живут в GitHub API вне git и этими слоями не сканируются. Единственный существующий слой, который их видит — серверный Secret Scanning (async alert), он срабатывает **после** экспозиции.
* **Локальный прототип:** тот же workflow уже написан и обкатан локально в `task-orchestrator` (PR task-orchestrator#279 + фикс в #281 — в нём был несуществующий `immutablealternatives/install-gitleaks` action, заменён на прямой download бинарника). Прототип рабочий: ловит секрет в PR-body за ~9 секунд.
* **Продуктовый вывод:** защита PR-content — это этап Git lifecycle (`... → PR`), значит должна жить в `git-workflow`, а не дублироваться в каждом из ~29 проектов prikotov, использующих этот пакет.
* **Основной сценарий:** при `opened/edited/reopened` PR и при создании/редактировании PR-комментариев CI гоняет gitleaks на собранном тексте (title + body + comments) и фейлит check + комментирует PR при детекте.
* **Границы (Out of Scope):**
  - верификация «живости» секрета (это TruffleHog, отдельный опциональный слой);
  - ротация/отзыв уже утекших секретов;
  - серверный blocking (это GitHub Push Protection/Secret Scanning, не наш слой);
  - сканирование issue-body (только PR — issue без PR не несёт merge-риска; опционально в Should Have).

## 3. Requirements (Требования, MoSCoW)

### 🔴 Must Have (Обязательно)

- [ ] Workflow `secret-scan-pr-content.yml` в составе пакета (в `templates/` или отдельном каталоге — решение в п.4).
- [ ] Триггеры: `pull_request: [opened, edited, reopened]`, `issue_comment: [created, edited]`, `pull_request_review_comment: [created, edited]` (с фильтром «только для PR»).
- [ ] Сбор контента: PR title, PR body, issue-style comments на PR, review comments на PR — через `gh api ... --jq '.[].body'`.
- [ ] Запуск gitleaks на собранном контенте через `gitleaks detect --no-git --redact` (redact обязателен, чтобы CI-лог не стал leak-source).
- [ ] Установка gitleaks в CI: прямой download бинарника из официальных releases (asset-паттерн `gitleaks_<ver>_linux_x64.tar.gz`, версия без префикса `v`). **Запрещено** ссылаться на сторонние `*-install-gitleaks` actions — половина из них несуществующие (подтверждено инцидентом task-orchestrator#281).
- [ ] При детекте секрета: workflow падает (exit 1) + оставляет комментарий на PR со ссылкой на run (без секрета в тексте).
- [ ] Использует `.gitleaks.toml` проекта (checkout репо), если есть — иначе default ruleset gitleaks.
- [ ] `git-workflow-init` **устанавливает** полный файл `.github/workflows/secret-scan-pr-content.yml` в проект-потребитель (механика «копирует полный файл», см. п.4 Вариант A).
- [ ] Идемпотентность: повторный `git-workflow-init` безопасен; существующий файл перезаписывается только с `--force` (как docs), иначе пропускается.
- [ ] Файл помнит источник для синхронизации (маркер-комментарий в начале файла — см. п.4).
- [ ] Обновлена документация `docs/git-workflow/secrets.md`: явно описан gap PR-body (почему pre-commit и Push Protection его не покрывают) и новый слой.

### 🟡 Should Have (Желательно)

- [ ] `git-workflow-init --sync` — обновляет ранее установленные файлы пакета (workflow, docs, hooks), опираясь на маркер, без перезаписи пользовательских правок без `--force`.
- [ ] Расширение триггеров на `issues: [opened, edited]` (issue-body) — опционально, по флагу.
- [ ] Кэширование бинарника gitleaks через `actions/cache` для скорости.

### ⚫ Won't Have (Не будем делать в V1)

- Reusable workflow через `workflow_call` к репозиторию пакета (требует публичного доступа к git-workflow и иначе ломает изоляцию — выбрана Трактовка 2 по решению владельца).
- Symlink/`vendor/`-путь как источник workflow (GitHub исполняет только реальные файлы в `.github/workflows/` репозитория проекта — symlink не работает).
- Автоматическая ротация/отзыв секретов.
- Внешние SaaS-сканеры по умолчанию.

## 4. Implementation Options (Варианты реализации)

### Вариант A: Копирование полного файла + маркер источника (выбран владельцем, Трактовка 2)

1. [ ] Положить полный workflow в `templates/workflows/secret-scan-pr-content.yml` пакета.
2. [ ] В начало файла добавить маркер-комментарий:
   ```yaml
   # ⚠️ Управляется `git-workflow-init`. Источник: prikotov/git-workflow → templates/workflows/secret-scan-pr-content.yml
   # Обновление: `php vendor/bin/git-workflow-init --sync`
   # Ручные правки будут перезаписаны при --sync / --force.
   ```
3. [ ] Расширить `bin/git-workflow-init`: при установке копировать `templates/workflows/*.yml` → `.github/workflows/` проекта-потребителя (создавать каталог при отсутствии).
4. [ ] Добавить флаг `--sync`: сканирует `.github/workflows/` на файлы с маркером, обновляет из текущей версии пакета; файлы без маркера (пользовательские) не трогает.
5. [ ] Документировать в `docs/git-workflow/secrets.md` и `docs/git-workflow/index.md`.

**Плюсы:** единая логика, понятный источник, обновляемость через `--sync`, не требует публичного доступа к репозиторию пакета.
**Минусы:** при обновлении пакета в `composer` файл автоматически не обновляется — нужен явный `--sync` (приемлемо, как и для docs).

### Вариант B: Reusable workflow (`workflow_call`) — ОТКЛОНЁН

Проект вызывает тонкий wrapper: `uses: prikotov/git-workflow/.github/workflows/...@v1`.
**Плюсы:** авто-обновление через ref/tag.
**Почему отклонён:** требует публичного репозитория git-workflow (reusable через `uses:`); нарушает изоляцию; решение владельца — Трактовка 2.

## 5. Proposed MVP Scope (Предлагаемый MVP)

- [ ] Вариант A: `templates/workflows/secret-scan-pr-content.yml` + расширение `bin/git-workflow-init` (установка + маркер).
- [ ] Установка gitleaks прямым download бинарника (никаких сторонних actions).
- [ ] Сбор title + body + comments, `gitleaks detect --no-git --redact`, comment-on-PR при детекте.
- [ ] Обновление `docs/git-workflow/secrets.md` (gap PR-body + новый слой).
- [ ] Документирование `--sync` (Should Have, но желательно в той же итерации).

## 6. Definition of Done (Критерии приёмки)

- [ ] В тестовом проекте с установленным пакетом `git-workflow-init` создаёт `.github/workflows/secret-scan-pr-content.yml` с маркером.
- [ ] При PR с live секретом в body (синтетический токен) CI падает + оставляет комментарий без самого секрета.
- [ ] При clean PR CI проходит (зелёный).
- [ ] Повторный `git-workflow-init` без `--force` не перезаписывает изменённый файл; с `--force` — перезаписывает.
- [ ] `git-workflow-init --sync` обновляет ранее установленный файл (по маркеру), не трогая пользовательские workflow без маркера.
- [ ] Секрет не появляется в CI-логе (redact).
- [ ] `docs/git-workflow/secrets.md` описывает gap PR-body и место нового слоя в общей картине защиты.
- [ ] CI пакета (`composer validate --strict`) зелёный; init-скрипт остаётся обратно совместимым.

## 7. Verification (Самопроверка)

```bash
# 1. Установка в чистый тестовый проект
cd /tmp/test-consumer
composer require --dev prikotov/git-workflow
php vendor/bin/git-workflow-init
test -f .github/workflows/secret-scan-pr-content.yml   # ожидается: файл есть, есть маркер
head -3 .github/workflows/secret-scan-pr-content.yml    # ожидается: маркер источника

# 2. Поведение на детекте (в реальном PR с синтетическим секретом в body)
#    ожидается: check fail + комментарий на PR, секрет в логе redacted

# 3. Идемпотентность / --sync
echo "# my local note" >> .github/workflows/secret-scan-pr-content.yml
php vendor/bin/git-workflow-init          # ожидается: файл НЕ перезаписан (нет --force)
php vendor/bin/git-workflow-init --force  # ожидается: перезаписан, локальная правка снята
php vendor/bin/git-workflow-init --sync   # ожидается: обновлён до текущей версии пакета
```

## 8. Risks and Dependencies (Рisks and Dependencies)

- **Сторонние `install-gitleaks` actions несуществуют/заброшены.** Mitigation: только прямой download бинарника из официальных releases gitleaks.
- **Файл в проекте расходится с пакетом при правках.** Mitigation: маркер источника + `--sync`; `--force` для жёсткой синхронизации.
- **GitHub не исполняет workflow из `vendor/`/symlink.** Mitigation: только реальный файл в `.github/workflows/` (учтено в Варианте A).
- **Маркер можно случайно стереть.** Mitigation: `--sync` молча пропускает файлы без маркера (не угадывает чужие); документировать, что стирание маркера = «отвязка от пакета».
- **CI-лог как leak-source.** Mitigation: `--redact` обязателен; в тело PR-комментария секрет не пишется.
- **False positives в документации.** Mitigation: используется `.gitleaks.toml` проекта с allowlist (как для pre-commit).

## 9. Security Notes (Заметки по безопасности)

- CI-слой закрывает **gap**, а не заменяет pre-commit: `--no-verify` для pre-commit + API-edit PR-body покрываются именно этим workflow.
- Если workflow нашёл секрет в PR-body — секрет уже публично виден (PR-body может быть public). Действия: проверить TTL/валидность, закрыть alert `resolution=revoked`, заменить в PR-body на placeholder, при реальной утече — revoke/rotate.
- Синтетические секреты только в тестах/fixtures; allowlist — через `.gitleaks.toml`/`.gitleaksignore`, не через хардкод правила.

## 10. Sources (Источники)

- Локальный прототип в task-orchestrator: `.github/workflows/secret-scan-pr-content.yml` (PR task-orchestrator#279, фикс #281).
- Существующая документация пакета: `docs/git-workflow/secrets.md`.
- GitHub Secret Scanning / Push Protection: https://docs.github.com/en/code-security/secret-scanning
- Gitleaks: https://github.com/gitleaks/gitleaks

## 11. Comments (Комментарии)

Решение по механике установки (Трактовка 2 — копирование полного файла + маркер для `--sync`) принято владельцем явно; reusable workflow (Вариант B) отклонён. Инцидент-триггер: утечка live GitHub App installation token в PR-body task-orchestrator (токен протух, PEM не утёк, audit чист — детали в ретро task-orchestrator). В задаче намеренно не фиксируется сам токен/PEM.

## Change History (История изменений)

| Дата | Автор (роль) | Изменение |
| :--- | :--- | :--- |
| 2026-06-19 | Тимлид Алекс | Создание задачи после инцидента с live токеном в PR-body task-orchestrator |

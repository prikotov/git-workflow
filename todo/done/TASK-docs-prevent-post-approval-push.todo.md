---
type: docs
created: 2026-07-28
value: V2
complexity: C1
priority: P1
depends_on: []
epic:
author: Тимлид (Алекс)
assignee: Технический писатель (Гермиона)
branch: task/docs-done-before-approval
pr: https://github.com/prikotov/git-workflow/pull/8
status: done
---

# TASK-docs-prevent-post-approval-push: Правило done-sync до approve (исключить post-approval push в PR)

## 0. Простое описание (Human Brief)

### Проблема простыми словами (Problem)
В `docs/git-workflow/pull-request.md` предписано: перед merge «переведи задачу в `done` прямо в ветке PR… закоммить и запушь изменения в ту же ветку». Это **толкает исполнителя к post-approval push**: агент вносит правки уже после того, как человек поставил approve, и эти правки уходят в merge без повторного review.

Реальный инцидент: в проекте-потребителе `task-orchestrator` (PR #331) агент запушил `done`-sync через ~3,5 минуты **после** approve человека — коммит не был просмотрен, но попал в squash-merge.

Проблема системная, а не разовая (ретро `task-orchestrator` 2026-08-21): в PR #362/#363/#364 `done`-sync пушлся после approve, branch protection `require_last_push_approval` сбрасывала approve — на 3 PR израсходовано **4 лишних approve**.

### Варианты или путь решения (Solution Sketch)
Зафиксировать в `pull-request.md` явное правило: **все правки, включая `done`-sync (status `done`, перенос в `done/`, обновление ссылок), выполняются ДО запроса approve** — единым PR. После approve — только merge, никаких push в PR-ветку. Если правки нужны после approve → открывается повторный review. Замечания после `done`-коммита → задача возвращается из `todo/done/` в `todo/` тем же коммитом, что вносит правки. Распределяемые документы пакета помечаются front matter `package: prikotov/git-workflow`.

### Ожидаемый результат (Expected Result)
- `pull-request.md` содержит явное правило против post-approval push и уточнённое место шага `done`-sync (подготовка PR, до запроса approve).
- Описан цикл возврата из `done` при замечаниях: возврат в `todo/` тем же коммитом, что вносит правки, затем повторный `done`-sync до следующего approve.
- Все распределяемые `.md` в `docs/git-workflow/` (кроме `*.template.md`) помечены front matter `package: prikotov/git-workflow`.
- Исполнители (люди и AI-агенты) не вносят правки после approve; процесс не зависит от того, включена ли branch protection `dismiss_stale_reviews`.

## 1. Concept and Goal (Концепция и Цель)

### Story (Job Story)
Когда я ревьюю PR и ставлю approve, я хочу быть уверен, что в ветку больше не попадёт ни одного коммита до merge, чтобы слитое состояние точно соответствовало просмотренному.

### Goal (Цель по SMART)
Зафиксировать в `docs/git-workflow/pull-request.md` процессуальное правило, исключающее post-approval push, уточнить место шага `done`-sync в жизненном цикле PR и пометить распределяемые документы пакета — в одном PR.

## 2. Context and Scope (Контекст и Границы)

* **Инцидент-триггер:** `task-orchestrator` PR #331 — post-approve push `done`-sync попал в merge без повторного review. Подтверждено хронологией: approve в `14:47:25Z`, push `done`-sync в `14:50:51Z`.
* **Повторные инциденты (ретро `task-orchestrator` 2026-08-21, P0):** PR #362, #363, #364 — `done`-sync пушлся после approve, `require_last_push_approval` сбрасывала одобрение, на 3 PR израсходовано 4 лишних approve. Паттерн воспроизводится у разных агентов → причина в процессе, а не в конкретном исполнителе.
* **Технический слой:** дыра закрыта branch protection в проекте-потребителе (`dismiss_stale_reviews` + `require_last_push_approval` + `enforce_admins`). Но процессуальное правило в `pull-request.md` всё ещё предписывает шаг, провоцирующий post-approval push, — значит, любой проект-потребитель без жёсткой protection уязвим.
* **Границы (Out of Scope):**
  - настройка branch protection (это ответственность проекта-потребителя, не пакета);
  - контентные правки других документов — только `pull-request.md`; `commits.md`, `branches.md`, `code-review.md` меняются только при прямом противоречии с новым порядком;
  - инструменты/автоматизация закрытия задач при merge (GitHub Actions).

## 3. Requirements (Требования, MoSCoW)

### 🔴 Must Have (Обязательно)
- [ ] В `pull-request.md` добавлено явное правило: после approve исполнителем (человеком или агентом) не делается push в PR-ветку — только merge.
- [ ] Уточнено место шага `done`-sync: вся подготовка (`status: done`, перенос задачи в `done/`, обновление ссылок в Epic/связанных задачах) выполняется **до** запроса approve, в составе того же PR.
- [ ] Описано исключение: если правки необходимы после approve → push сбрасывает одобрение (или инициируется повторный review), повторный merge только после нового approve.
- [ ] Описан цикл возврата из `done` при замечаниях: задача возвращается из `todo/done/` в `todo/` **тем же коммитом**, что вносит правки, затем снова переводится в `done` (status `done`, перенос в `done/`, обновление ссылок) до следующего approve.
- [ ] Все распределяемые `.md` в `docs/git-workflow/` помечены YAML front matter `package: prikotov/git-workflow` первым блоком файла; шаблоны `*.template.md` (`templates/`, `releases/templates/`) не помечаются — их заполняет проект-потребитель.
- [ ] Объяснена мотивация: post-approval push пропускает непросмотренные изменения в merge и сбрасывает approve при `require_last_push_approval`.

### 🟡 Should Have (Желательно)
- [ ] Добавлен короткий пример/антипример (как было в PR #331 — что неправильно; как должно быть).
- [ ] Связь с branch protection упомянута как рекомендованный технический дублирующий слой (`dismiss_stale_reviews`, `require_last_push_approval`, `enforce_admins`).

### ⚫ Won't Have (Не будем делать)
- Автоматизацию переноса задач через CI/GitHub Actions.
- Контентные изменения других документов пакета (кроме front matter-маркировки `package`).

## 4. Implementation Options (Варианты реализации)

### Вариант A: `done`-sync до approve (рекомендуемый)
1. [ ] Перенести шаг «переведи задачу в `done`» из блока «Перед merge» в блок подготовки PR (до запроса approve).
2. [ ] Добавить явный запрет: «После approve — никаких push в PR-ветку, только merge».
3. [ ] Описать исключение с повторным review и цикл возврата из `done` при замечаниях.
4. [ ] Пометить все распределяемые `.md` в `docs/git-workflow/` front matter `package: prikotov/git-workflow` (кроме `*.template.md`).

**Плюсы:** один PR на задачу; человек review'ит финальное состояние целиком; `done` = «работа выполнена, merge = доставка».  
**Минусы:** при отклонении PR нужно откатить `status: done` (редкий случай; закрывается циклом возврата из `done` тем же коммитом, что вносит правки).

### Вариант B: второй PR для `done`-sync
1. [ ] PR#1: статус `review`; после merge открывается PR#2 с `done`-sync.

**Плюсы:** `done` хронологически после approve (семантически привлекательно).  
**Минусы:** два PR на каждую задачу — тяжело, особенно для docs-only.

## 5. Proposed MVP Scope (Предлагаемый объём)
- [ ] Реализовать Вариант A: скорректировать блоки «Создание PR», «Перед merge», «Выполнение merge» в `pull-request.md`, добавить запрет post-approval push, исключение, цикл возврата из `done` и пример.
- [ ] Добавить front matter `package: prikotov/git-workflow` во все распределяемые `.md` (кроме `*.template.md`).

## 6. Definition of Done (Критерии приёмки)
- [ ] В `pull-request.md` есть явное правило против post-approval push.
- [ ] Шаг `done`-sync однозначно отнесён к подготовке PR (до approve).
- [ ] Описано исключение (правки после approve → повторный review).
- [ ] Описан возврат задачи из `done` при замечаниях (тем же коммитом, что вносит правки).
- [ ] Все распределяемые `.md` (кроме `*.template.md`) начинаются с front matter `package: prikotov/git-workflow`.
- [ ] Документация обновлена без противоречий с соседними разделами.
- [ ] Проверки зелёные: `composer validate --strict`; grep по `docs/git-workflow/` не находит предписаний «после approve переведи в done / запушь».

## 7. Verification (Самопроверка)
- Прочитать обновлённый `pull-request.md` и убедиться, что из него невозможно сделать вывод «push после approve допустим».
- Проверить, что шаг `done`-sync не остался в блоке «Перед merge» как отдельный post-approval push.
- Прогнать grep по `docs/git-workflow/`: не осталось формулировок вида «после approve переведи задачу в done / запушь».
- Убедиться, что front matter `package` есть во всех `.md`, кроме `*.template.md`.

## 8. Risks and Dependencies (Риски и зависимости)
- Проекты-потребители с уже склонированной старой копией `pull-request.md` не получат правило автоматически — нужен `composer update prikotov/git-workflow` + повторный `bin/git-workflow-init --force`.
- Правило — процессуальное; реальная защита от обхода остаётся за branch protection в каждом проекте-потребителе.
- Front matter в начале файлов может сломать потребительские парсеры, ожидающие `#` первой строкой (риск низкий: markdown-рендеры поддерживают front matter нативно).

## 9. Sources (Источники)
- Инцидент: `task-orchestrator` PR #331 (post-approval push `done`-sync в merge).
- Ретроспектива `task-orchestrator` 2026-08-21: PR #362/#363/#364 — 4 лишних approve на 3 PR из-за post-approval push при включённой `require_last_push_approval`.
- GitHub branch protection: `dismiss_stale_reviews`, `require_last_push_approval`, `enforce_admins` — https://docs.github.com/en/repositories/configuring-tools-and-programming-with-github-articles/managing-a-branch-protection-rule

## 10. Comments (Комментарии)
Техническая дыра в проекте-потребителе уже закрыта branch protection. Эта задача — про каноничный процессуальный источник (`pull-request.md`), чтобы правило было зафиксировано в пакете и разносилось во все проекты-потребители, а не держалось только в AGENTS.md конкретного проекта.

2026-08-21: задача одобрена владельцем как P0 из ретроспективы `task-orchestrator` 2026-08-21; в рамках пакета приоритет P1. В объём добавлены: цикл возврата из `done` при замечаниях и маркировка `package` распределяемых документов.

## Change History (История изменений)
| Дата | Автор (роль) | Изменение |
| :--- | :--- | :--- |
| 2026-07-28 | Тимлид (Алекс) | Создание задачи после инцидента post-approval push в `task-orchestrator` PR #331. |
| 2026-08-21 | Технический писатель (Гермиона) | Приоритет P1 (P0 из ретро `task-orchestrator` 2026-08-21, одобрено владельцем). Контекст: инциденты #362/#363/#364. Must Have: цикл возврата из `done`, маркировка `package`. Статус `in_progress`, ветка `task/docs-done-before-approval`. |
| 2026-08-21 | Технический писатель (Гермиона) | Возвращена в `review` по замечанию владельца (убрать заметку о `package` из README), затем повторный done-sync. |

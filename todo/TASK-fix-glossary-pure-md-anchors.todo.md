---
type: fix
created: 2026-09-22 00:39:17 (1790037557)
due: 
started: 2026-09-22 00:39:30 (1790037570)
completed: 2026-09-22 00:41:36 (1790037696)
cancelled: 
value: V2
complexity: C1
priority: P2
cost_plan: 
cost_fact: 
depends_on: 
epic: 
author: Бэкендер Тони (pi)
assignee: Бэкендер Тони (pi)
branch: task/glossary-pure-md-anchors
pr: https://github.com/prikotov/git-workflow/pull/12
status: review
---

# TASK-fix-glossary-pure-md-anchors: Глоссарий — чистый Markdown вместо HTML-якорей

## 0. Простое описание (Human Brief)

### Проблема простыми словами (Problem)
- Глоссарий использует явные HTML-якоря `<a id="...">` (26 шт.). Валидатор ссылок `prikotov/coding-standard` такие цели не распознаёт (считает только заголовки) и ложно помечает ссылки на термины как битые.
- Владелец экосистемы принял политику чистого Markdown: якоря — только через заголовки, сырой HTML не используем.

### Варианты или путь решения (Solution Sketch)
- Переделать глоссарий: один термин — один файл `glossary/<term>.md` (ASCII-имя, полный двуязычный заголовок внутри), обзорный `glossary/index.md`; ссылки на термины — файлами. Автославки заголовков (`#выкладка-deployment-deploy`) и HTML-якоря отклонены владельцем как нечитаемые/сырой HTML; переименование файла прозрачно отслеживается git-ом в отличие от переименования заголовка.

### Ожидаемый результат (Expected Result)
- В глоссарии нет сырого HTML; все внутренние ссылки пакета проходят `validate-md-links` (v0.33.0) без ложных срабатываний.

## 1. Концепция и Цель (Concept and Goal)

### История (User Story)
> **User Story:** Как потребитель пакета, я хочу глоссарий на чистом Markdown с якорями-заголовками, чтобы документация не зависела от сырого HTML и проходила стандартные проверки ссылок.

### Цель по SMART (Goal)
- Заменить однофайловый глоссарий на `glossary/` из 26 файлов терминов + `index.md`; обновить все ссылки на термины (release.md, index.md, README.md, AGENTS.md); `validate-md-links` (v0.33.0) на `docs/git-workflow/` — зелёный.

## 2. Контекст и Границы (Context and Scope)
* **Где делаем:** `docs/git-workflow/glossary.md`, `docs/git-workflow/release.md`.
* **Границы (Out of Scope):** не менять структуру секций глоссария и формулировки определений; не трогать остальные документы.

## 3. Требования, MoSCoW (Requirements)
### 🔴 Обязательно (Must Have)
- [x] `glossary/<term>.md` — 26 файлов терминов: front matter пакета, H1 с полным двуязычным названием, определение.
- [x] `glossary/index.md` — обзорный словарь по секциям со ссылками на термины.
- [x] Ни одного `<a id>`/`<a name>` в документах пакета; старый `glossary.md` удалён.
- [x] Ссылки на термины обновлены на файлы (`glossary/release-publishing.md`, `../pull-request.md` и т.п.).
- [x] `validate-md-links` (prikotov/coding-standard v0.33.0) на `docs/git-workflow/` — без ошибок.
### ⚫ Won't Have (Не будем делать)
- Поддержку HTML-якорей в валидаторе (отклонена политикой чистого Markdown, PR prikotov/coding-standard#126 закрыт).

## 4. План реализации (Implementation Plan)
1. [x] Преобразовать глоссарий в H3-заголовки (первая итерация, PR #12).
2. [x] По решению владельца: переработать в файлы терминов `glossary/<term>.md` + `index.md`.
3. [x] Обновить ссылки на термины в release.md, index.md, README.md, AGENTS.md.
4. [x] Прогнать валидатор ссылок и `composer validate --strict`.

## 5. Критерии приёмки (Definition of Done)
- [x] 26 файлов `docs/git-workflow/glossary/<term>.md` + `index.md`; `glossary.md` удалён.
- [x] Валидатор ссылок: все внутренние ссылки валидны.
- [x] `composer validate --strict` — valid.

## 6. Самопроверка (Verification)
```bash
php <путь до vendor>/prikotov/coding-standard/bin/validate-md-links docs/git-workflow/
composer validate --strict
```

## 7. Риски и зависимости (Rиски и Dependencies)
- Ссылки на термины теперь файловые: устойчивы к переформулировкам заголовков; переименование файла прозрачно отслеживается git. Индекс `glossary/index.md` поддерживается вручную.

## 8. Источники (Sources)
- `docs/git-workflow/glossary.md` (до/после).
- Решение владельца: политика чистого Markdown (обсуждение в сессии task-orchestrator, PR prikotov/coding-standard#126 закрыт).

## 9. Комментарии (Comments)

## История изменений (Change History)
| Дата | Автор (роль) | Изменение |
| :--- | :--- | :--- |
| 2026-09-22 00:39:17 (1790037557) | Бэкендер Тони (pi) | Создание задачи |
| 2026-09-22 00:40:00 (1790037600) | Бэкендер Тони (pi) | Старт задачи, заполнение постановки |
| 2026-09-22 00:50:00 (1790038200) | Бэкендер Тони (pi) | Глоссарий переделан на H3-заголовки, ссылки обновлены, проверки зелёные |
| 2026-09-22 01:20:00 (1790040000) | Бэкендер Тони (pi) | По решению владельца переработано в файлы терминов glossary/<term>.md + index.md; задача возвращена в review |

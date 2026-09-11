---
package: prikotov/git-workflow
---

# Артефакты релиза

Этот раздел описывает документы, которые создаются для конкретного production релиза.

## Структура

- `docs/git-workflow/templates/release-plan.template.md` — шаблон плана релиза.
- `docs/releases/vX.Y.Z/release-plan.md` — заполненный план релиза для конкретного тега релиза.

## Правила

- Для каждого production релиза каталог `docs/releases/vX.Y.Z/` и его файлы готовятся в `task/*` от активной `release/x.y` до окончательного одобрения PR.
- Артефакты включаются в `release/x.y` через PR до создания тега на проверенном слитом коммите: [процесс релиза](../release.md#выпуск-релиза).
- Минимально обязательный файл в каталоге релиза: `release-plan.md`.
- `release-plan.md` фиксирует состав релиза, риски, миграции, порядок deploy, post-check и план действий через hotfix или patch release.
- Для `hotfix` и `patch release` создаётся отдельный каталог по новому тегу релиза.

## Шаблон

- [Шаблон плана релиза](../templates/release-plan.template.md)

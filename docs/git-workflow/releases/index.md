# Артефакты релиза

Этот раздел хранит документы, которые создаются для конкретного production релиза.

## Структура

- `templates/release-plan.template.md` — шаблон плана релиза.
- `vX.Y.Z/release-plan.md` — заполненный план релиза для конкретного тега релиза.

## Правила

- Для каждого production релиза перед deploy создаётся каталог `docs/releases/vX.Y.Z/`.
- Минимально обязательный файл в каталоге релиза: `release-plan.md`.
- `release-plan.md` фиксирует состав релиза, риски, миграции, порядок deploy, post-check и план действий через hotfix или patch release.
- Для `hotfix` и `patch release` создаётся отдельный каталог по новому тегу релиза.

## Шаблон

- [Шаблон плана релиза](templates/release-plan.template.md)

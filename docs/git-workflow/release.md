# Релизы и CHANGELOG

Этот гайд фиксирует release model проекта TasK: `master` как integration branch, одна active `release/x.y`, production deploy по immutable `tag` `vX.Y.Z`.

## Release model

- `master` содержит текущую интеграцию задач и может опережать production.
- Перед production выпуском из выбранного commit в `master` создаётся `release/x.y`.
- В `release/x.y` допускаются только stabilizing changes: bugfix, release docs, безопасные мелкие правки.
- Production всегда разворачивается по **конкретному tag**, а не по branch head.
- Одновременно поддерживается только одна active `release/x.y`.

## SemVer и линии релиза

- `patch` (`X.Y.Z -> X.Y.Z+1`) — срочные исправления и стабилизация в текущей release line.
- `minor` (`X.Y -> X.Y+1`) — новая функциональность; для неё открывается новая `release/x.y`.
- `major` — breaking changes; для неё также открывается новая `release/x.y`.

Тип релиза определяется по истории Conventional Commits. В спорных случаях используйте явные `make release-*`.

## Подготовка коммитов

Используйте интерактивный помощник:

```bash
make prepare-commit
```

Если helper падает, сформируйте заголовок вручную по Conventional Commits:

```bash
git commit -m "type(scope): description"
```

Примеры:
- `feat(auth): add OAuth2 login for Google`
- `fix(ui): align submit button on mobile`
- `docs(release): describe hotfix merge-back`
- `feat!: remove legacy v1 endpoints`

## Release cut

Когда Product Owner и Team Lead зафиксировали состав релиза:

```bash
git switch master
git pull origin master
git switch -c release/x.y
git push -u origin release/x.y
```

После открытия `release/x.y`:
- все новые feature PR продолжают идти в `master`;
- в `release/x.y` попадают только stabilizing PR;
- если найден дефект production, hotfix стартует от текущего production tag и затем вливается в `release/x.y` и `master`.

## План релиза

Для каждого production release перед deploy создаётся документ `docs/releases/vX.Y.Z/release-plan.md`.

- место хранения: [Артефакты релиза](../releases/index.md);
- шаблон: [Шаблон плана релиза](../releases/templates/release-plan.template.md);
- без заполненного `release-plan.md` deploy не начинается.

Базовое создание документа после определения тега релиза:

```bash
mkdir -p docs/releases/vX.Y.Z
cp docs/releases/templates/release-plan.template.md docs/releases/vX.Y.Z/release-plan.md
```

В `release-plan.md` обязательно зафиксируйте:
- состав релиза и его границы;
- риски, миграции, порядок их применения и риск окна несовместимости;
- порядок deploy;
- проверки после deploy;
- план действий при проблеме после релиза через hotfix или patch release.

## Работа с CHANGELOG

Для предпросмотра изменений:

```bash
make changelog
git diff CHANGELOG.md
```

`CHANGELOG.md` должен оставаться коротким:
- заголовок релиза;
- compare link;
- одна короткая summary-строка.

Подробные notes публикуются в GitHub Release.

## Выпуск релиза

Перед релизом:

```bash
make check
make tests-e2e
```

Релиз выполняется **на active `release/x.y`**, а не на `master`.

Вариант A — patch по умолчанию:

```bash
make release
```

Вариант B — явный тип:

```bash
make release-patch
make release-minor
make release-major
```

Что делает release команда:
- обновляет `CHANGELOG.md`;
- определяет или принимает версию;
- делает release commit;
- создаёт git tag `vX.Y.Z`.

После генерации:
1. проверьте `CHANGELOG.md`;
2. создайте и заполните `docs/releases/vX.Y.Z/release-plan.md`;
3. если блок слишком длинный, вынесите подробности в GitHub Release notes;
4. запушьте release branch и tags:

```bash
git push origin release/x.y
git push origin --tags
```

5. создайте GitHub Release:

```bash
gh release create vX.Y.Z --notes-file tmp/release-vX.Y.Z.md
```

или

```bash
gh release create vX.Y.Z --generate-notes
```

## Hotfix и patch release

Срочный hotfix не делается от `master`.

Базовый flow:
1. определить текущий production tag `vX.Y.Z`;
2. создать `hotfix/x.y.z-<short-description>` от этого tag;
3. исправить проблему и провести обязательные проверки;
4. влить hotfix в active `release/x.y`;
5. выпустить patch release `vX.Y.(Z+1)`;
6. выполнить merge-back hotfix changes в `master`.

Если active `release/x.y` уже закрыта, hotfix всё равно стартует от production tag, а merge-back в `master` делается отдельным PR сразу после patch release.

## Recovery Policy

- Production rollback не является штатным сценарием.
- Проблемы после deploy исправляются через hotfix или patch release.
- План такого исправления фиксируется в `docs/releases/vX.Y.Z/release-plan.md`.
- Для релизов с миграциями заранее оценивайте, как быстро выпустить безопасный fix без отката.

## Рекомендации команды

- Перед release cut проверьте, что в `master` нет случайных незавершённых изменений, которые не должны попасть в релиз.
- Для hotfix PR всегда явно фиксируйте merge-back plan.
- Не открывайте вторую `release/x.y`, пока не закрыта текущая line.
- Для release и hotfix используйте чеклисты: [Чеклисты релиза и hotfix](release-checklists.md).

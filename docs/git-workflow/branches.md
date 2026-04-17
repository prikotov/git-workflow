# Ветки (Branches)

**Ветка (branch)** — изолированная линия разработки для задачи, релиза или срочного исправления.

## Границы ответственности

- Документ описывает только типы веток, их назначение и жизненный цикл.
- Правила PR см. в [Pull Request (PR)](pull-request.md).
- Правила релизов см. в [Релизы (Release)](release.md).
- Правила деплоя см. в [Деплой (Deploy)](deploy.md).

## Целевая модель

- `master` — integration branch для обычной разработки.
- `task/<short-description>` — рабочая ветка для feature, bugfix, docs и рефакторинга.
- `release/x.y` — активная линия стабилизации релиза.
- `hotfix/x.y.z-<short-description>` — срочный patch для уже выкаченного production release.
- Production состояние фиксируется **tag** `vX.Y.Z`, а не текущим состоянием ветки.
- Одновременно поддерживается только одна активная `release/x.y`.

## Общие правила

- Запрещены прямые правки в `master`.
- Запрещены прямые правки в `release/*`.
- Запрещён деплой в production из текущего состояния ветки.
- Одна ветка — одна цель: не смешиваем разные задачи и “случайные” улучшения.
- Массовые перемещения и переименования файлов не смешиваем с последующим рефакторингом и изменением поведения.
- Перед стартом убедись, что рабочее дерево чистое: `git status`.
- Если есть чужие незакоммиченные изменения — остановись и уточни у пользователя.

## Именование

- `task/<short-description>` — английский, `kebab-case`, кратко по смыслу.
- `release/x.y` — release line по `major.minor`.
- `hotfix/x.y.z-<short-description>` — patch version плюс короткое описание.

Примеры:
- `task/docs-release-workflow`
- `release/0.9`
- `hotfix/0.9.3-login-timeout`

## Откуда создавать ветки

### Task branch

Используется для обычной разработки и документации.

```bash
git switch master
git pull origin master
git switch -c task/<short-description>
```

### Release branch

Создаётся только после решения, что конкретный набор изменений идёт в production.

```bash
git switch master
git pull origin master
git switch -c release/x.y
git push -u origin release/x.y
```

Правила для `release/x.y`:
- в неё попадают только stabilizing changes;
- новые feature PR продолжают идти в `master`;
- после выпуска patch changes из release line не должны теряться в `master`.

### Hotfix branch

По умолчанию hotfix создаётся от **текущего production tag** `vX.Y.Z`, а не от `master`.

```bash
git fetch origin --tags --prune
git switch -c hotfix/x.y.z-<short-description> vX.Y.Z
```

Если активная `release/x.y` уже соответствует текущей production line, hotfix всё равно стартует от production tag, а затем вливается в `release/x.y` и обратно в `master`.

## Синхронизация

- `task/*` синхронизируется с `master`.
- `release/*` синхронизируется только с собственной release line; новые feature commits из `master` туда не подтягиваются автоматически.
- `hotfix/*` после merge должен присутствовать в active `release/x.y` и в `master`.
- Если не уверен, использовать `merge` или `rebase`, — уточни у пользователя.

```bash
git fetch origin
git merge origin/master
```

```bash
git fetch origin
git rebase origin/master
```

## Завершение

- После merge PR рабочую ветку нужно удалить локально и в `origin`.
- После выпуска `release/x.y`, когда line закрыта и merge-back завершён, release branch можно удалить.
- Hotfix branch удаляется сразу после merge-back в целевые ветки.

---
package: prikotov/git-workflow
---

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

Для обычной разработки и документации база и цель PR — `master`.

```bash
git switch master
git pull --ff-only origin master
git switch -c task/<short-description>
```

Для стабилизации и подготовки релизных файлов база и цель PR — активная `release/x.y`:

```bash
git switch release/x.y
git pull --ff-only origin release/x.y
git switch -c task/<short-description>
```

### Release branch

Создаётся только после решения, что конкретный набор изменений идёт в production.

```bash
git switch master
git pull --ff-only origin master
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

- Рабочая ветка `task/<short-description>` синхронизируется с веткой, от которой создана: `master` для обычной разработки или активной релизной веткой `release/x.y` для стабилизации и подготовки релиза.
- Ветка для синхронизации совпадает с целевой веткой PR; не подтягивай изменения из ветки `master` в рабочую ветку подготовки релиза.
- Локальная релизная ветка `release/x.y` обновляется только из одноимённой ветки удалённого репозитория `origin` через `--ff-only`; изменения в неё поступают через PR, новые функции из ветки `master` не подтягиваются.
- Изменения из рабочей ветки `hotfix/x.y.z-<short-description>` должны попасть в выбранную ветку патч-релиза и в ветку `master`; целевая ветка определяется по [правилам срочного исправления](release.md#hotfix-и-patch-release).
- Если не уверен, использовать `merge` или `rebase`, — уточни у пользователя.

В рабочей ветке выбери базу и получи её актуальное состояние:

```bash
base=master # Для стабилизации и подготовки релиза: base=release/x.y
git fetch origin
```

Затем используй **один** согласованный способ — merge:

```bash
git merge "origin/$base"
```

Или rebase:

```bash
git rebase "origin/$base"
```

Синхронизацию заверши до окончательного одобрения PR. Если после одобрения нужны изменения — повтори проверки и запроси новое одобрение по [правилам PR](pull-request.md#подготовка-pr).

## Завершение

- После merge PR рабочую ветку нужно удалить локально и в `origin`.
- После выпуска `release/x.y`, когда line закрыта и merge-back завершён, release branch можно удалить.
- Hotfix branch удаляется сразу после merge-back в целевые ветки.

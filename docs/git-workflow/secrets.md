# Защита от коммита секретов (Secret Scanning)

**Secret scanning** — автоматическая проверка staged diff на наличие секретов (токены, пароли, ключи) до создания commit.

## Зачем

Секреты, попавшие в Git-историю, считаются скомпрометированными — даже после удаления из файлов они остаются в истории и могут быть восстановлены. GitHub Push Protection и GitGuardian срабатывают **после** push, когда секрет уже в удалённом репозитории.

Локальный pre-commit hook ловит утечку **до** commit.

## Инструмент — Gitleaks

[Gitleaks](https://github.com/gitleaks/gitleaks) — зрелый OSS-сканер на Go с 100+ правилами детекции:
- AWS Access Key / Secret Key
- GitHub Token / OAuth
- Google (GCP) Service Account / API Key
- Stripe, Slack, JWT, SSH private keys, и многие другие
- Custom правила через конфиг

### Установка Gitleaks

```bash
# Fedora / RHEL
sudo dnf install gitleaks

# macOS
brew install gitleaks

# Другие ОС
# Скачайте бинарник с https://github.com/gitleaks/gitleaks/releases
```

## Подключение в проект

### 1. Установка пакета и git hooks

```bash
composer require --dev prikotov/git-workflow
php vendor/bin/git-workflow-init --hooks
```

Флаг `--hooks` установит `commit-msg` и `pre-commit` хуки в `.git/hooks/`.

### 2. Как работает pre-commit

`pre-commit` хук проверяет только staged changes — ровно то, что попадёт в commit.

Команда запускается детерминированно:

```bash
gitleaks protect --staged --redact --no-banner --log-level warn
```

Если проект использует свой hook manager (lefthook, husky), можно подключить ту же команду вручную.

**Lefthook**

```yaml
# lefthook.yml
pre-commit:
  commands:
    gitleaks:
      run: gitleaks protect --staged --redact --no-banner --log-level warn
```

**Husky**

```bash
echo 'gitleaks protect --staged --redact --no-banner --log-level warn' >> .husky/pre-commit
```

### 3. Настройте allowlist (опционально)

Gitleaks работает из коробки с дефолтным набором правил. Для кастомизации скопируйте пример конфига:

```bash
cp vendor/prikotov/git-workflow/templates/gitleaks.toml.example .gitleaks.toml
```

И отредактируйте `.gitleaks.toml` — добавьте проектные исключения:

```toml
[allowlist]
paths = [
    '''tests/fixtures/.*''',
    '''\.example$''',
]
```

Или разрешите конкретное срабатывание через `.gitleaksignore` (fingerprint показывается в выводе gitleaks):

```
# .gitleaksignore
test-leak.md:basic-auth-url:1
```

## Ручной запуск

```bash
# Проверить staged diff
gitleaks protect --staged

# Проверить диапазон коммитов (CI)
gitleaks detect --source . --log-opts="origin/main..HEAD"
```

## CI — второй уровень защиты

Pre-commit hook можно обойти через `--no-verify`. Рекомендуется второй слой в CI.

### Gitleaks в GitHub Actions

```yaml
name: Secret Scan
on: [push, pull_request]
jobs:
  gitleaks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
```

### TruffleHog в CI (альтернатива)

[TruffleHog](https://github.com/trufflesecurity/trufflehog) — дополняет Gitleaks верификацией секретов (проверяет, живой ли ключ через API). Рекомендуется для дополнительного слоя в CI:

```yaml
- name: TruffleHog Scan
  uses: trufflesecurity/trufflehog@main
  with:
    extra_args: --only-verified
```

## Сканирование PR-body и комментариев (gap: контент вне git)

Pre-commit hook и GitHub Push Protection работают только на **git-контент** (staged diff → commit → push). Но PR-body, PR-комментарии и issue-комментарии живут в **GitHub API вне git**: их можно отредактировать через `gh pr edit`, веб-UI или API — и эти слои их **не увидят**. Единственный серверный слой (GitHub Secret Scanning) срабатывает **после** того, как секрет уже публично виден.

Этот gap закрывает workflow `secret-scan-pr-content.yml`: на `opened/edited/reopened` PR и при создании/редактировании комментариев он собирает title + body + comments и гоняет на них gitleaks.

### Установка (поставляется с пакетом)

```bash
composer require --dev prikotov/git-workflow
php vendor/bin/git-workflow-init
```

`git-workflow-init` копирует `templates/workflows/secret-scan-pr-content.yml` → `.github/workflows/` проекта. Файл идемпотентен: повторный запуск без `--force` пропускает существующий, с `--force` — перезаписывает свежей версией из пакета.

### Обновление

```bash
composer update prikotov/git-workflow
php vendor/bin/git-workflow-init --force
```

### Как это безопасно собирает контент

Title/body/comments читаются через `gh api ... --jq` и пишутся **в файл** (`scan/pr-meta.txt`), а затем сканируются gitleaks (`--no-git --redact`). Контент **никогда не интерполируется в shell-команду** через `${{ ... }}` — иначе markdown/backticks/кавычки в PR-body ломают шаг (script injection). `--redact` гарантирует, что CI-лог не станет источником утечки.

### Настройка allowlist

Используется `.gitleaks.toml` проекта (если есть) — общий с pre-commit, см. [выше](#3-настройте-allowlist-опционально). Если `.gitleaks.toml` отсутствует — применяются встроенные rules gitleaks.

## Что делать при срабатывании

1. **До commit** (сканер сработал) — удалите секрет из staged diff, используйте переменные окружения или vault.
2. **После push** (секрет уже в истории) — **считайте секрет скомпрометированным**:
   - Revoke/rotate секрет немедленно.
   - Очистите историю Git (`git filter-repo`).
   - Проверьте PR caches и GitHub refs.

## Ограничения

- Проверяется только staged diff (не весь репозиторий) — для скорости.
- `--no-verify` обходит hook — используйте CI / GitHub Push Protection как второй слой.
- Gitleaks не верифицирует секреты (чистый regex) — для верификации используйте TruffleHog в CI.

## Ссылки

- [Gitleaks](https://github.com/gitleaks/gitleaks) — основной инструмент
- [Lefthook](https://github.com/evilmartians/lefthook) — менеджер git-хуков
- [TruffleHog](https://github.com/trufflesecurity/trufflehog) — верификация секретов в CI
- [GitHub Secret Scanning](https://docs.github.com/en/code-security/secret-scanning)
- [Коммиты](commits.md)
- [Pull Request](pull-request.md)

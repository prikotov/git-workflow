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
# macOS
brew install gitleaks

# Linux
# Скачайте бинарник с https://github.com/gitleaks/gitleaks/releases
# Или через Nix, Docker — см. README Gitleaks
```

## Подключение в проект

### 1. Установка пакета и хуков

```bash
composer require --dev prikotov/git-workflow
php vendor/bin/git-workflow-init --hooks
```

Флаг `--hooks` установит `pre-commit` и `commit-msg` хуки в `.git/hooks/`.

### 2. Конфиг Gitleaks

Скопируйте шаблон конфига в корень проекта:

```bash
cp vendor/prikotov/git-workflow/templates/gitleaks.toml .gitleaks.toml
```

Файл `.gitleaks.toml`:
- Расширяет стандартный набор правил Gitleaks (`useDefault = true`)
- Добавляет правило для Basic Auth URL (из инцидента)
- Содержит allowlist для документации и тестов

### 3. Настройте allowlist

Отредактируйте `.gitleaks.toml` — добавьте проектные исключения:

```toml
[allowlist]
paths = [
    '''tests/fixtures/.*''',
    '''\.example$''',
]

# Или разрешить конкретный секрет по fingerprint
# (fingerprint показывается в выводе gitleaks при срабатывании)
[[allowlist]]
paths = [
    '''config/production\.example\.yml$''',
]
```

## Ручной запуск

```bash
# Проверить staged diff (pre-commit)
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
- [TruffleHog](https://github.com/trufflesecurity/trufflehog) — верификация секретов в CI
- [GitHub Secret Scanning](https://docs.github.com/en/code-security/secret-scanning)
- [Коммиты](commits.md)
- [Pull Request](pull-request.md)

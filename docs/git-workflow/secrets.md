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

### 1. Установка пакета и commit-msg хука

```bash
composer require --dev prikotov/git-workflow
php vendor/bin/git-workflow-init --hooks
```

Флаг `--hooks` установит `commit-msg` хук в `.git/hooks/`.

### 2. Подключить Gitleaks к pre-commit

`pre-commit` хук не устанавливается автоматически — им может управлять проект (husky, lefthook, свой скрипт). Добавьте вызов Gitleaks в ваш pre-commit hook:

**Вариант A: вручную в `.git/hooks/pre-commit`**

```bash
#!/usr/bin/env bash
set -euo pipefail

# ... другие проверки ...

gitleaks protect --staged
```

**Вариант B: через [Lefthook](https://github.com/evilmartians/lefthook) (рекомендуется)**

```yaml
# lefthook.yml
pre-commit:
  commands:
    gitleaks:
      run: gitleaks protect --staged
```

**Вариант C: через [Husky](https://typicode.github.io/husky/)**

```bash
echo 'gitleaks protect --staged' >> .husky/pre-commit
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

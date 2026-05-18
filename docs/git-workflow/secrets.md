# Защита от коммита секретов (Secret Scanning)

**Secret scanning** — автоматическая проверка staged diff на наличие секретов (токены, пароли, ключи) до создания commit.

## Зачем

Секреты, попавшие в Git-историю, считаются скомпрометированными — даже после удаления из файлов они остаются в истории и могут быть восстановлены. GitHub Push Protection и GitGuardian срабатывают **после** push, когда секрет уже в удалённом репозитории.

Локальный pre-commit hook ловит утечку **до** commit.

## Подключение в проект

### Установка пакета

```bash
composer require --dev prikotov/git-workflow
```

### Установка pre-commit хука

```bash
cp vendor/prikotov/git-workflow/hooks/pre-commit .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit
```

### Ручной запуск

Проверить staged diff без хука:

```bash
vendor/bin/git-secret-guard staged
```

Проверить диапазон коммитов (CI / pre-push):

```bash
vendor/bin/git-secret-guard scan-range origin/main...HEAD
```

### Опции

```
vendor/bin/git-secret-guard staged [options]

Options:
  --config=<path>          Путь к конфигу (по умолчанию: .secret-guard.json).
  --show-fingerprint       Показать fingerprint для allowlist.
  --no-color               Отключить цветной вывод.
```

## Что детектируется

| Правило | Паттерн | Severity |
| :--- | :--- | :--- |
| `basic-auth-url` | `https://user:pass@host` | high |
| `auth-basic-header` | `Authorization: Basic <base64>` (если декодируется в `user:pass`) | high |
| `private-key` | `-----BEGIN ... PRIVATE KEY-----` | high |
| `env-secret` | `*_TOKEN=`, `*_SECRET=`, `*_PASSWORD=`, `API_KEY=` | medium |

### Известные placeholder-ы не блокируются

Следующие паттерны **не** считаются секретами:

- **URL**: `https://user:pass@example.com`, `https://user:password@localhost`
- **Env**: пустые значения, `$VAR`, `${VAR}`, `<value>`, `changeme`, `TODO`, значения короче 4 символов

## Allowlist (исключения)

Создайте `.secret-guard.json` в корне репозитория:

```json
{
  "allowlist": [
    {
      "description": "Пример placeholder URL в документации",
      "path": "docs/api.md",
      "fingerprint": "sha256:abc123..."
    },
    {
      "description": "Тестовые фикстуры",
      "path": "tests/fixtures/*"
    }
  ]
}
```

### Типы записей

| Поле | Описание |
| :--- | :--- |
| `path` + `fingerprint` | Разрешить конкретное срабатывание в файле |
| `path` (без `fingerprint`) | Разрешить все срабатывания в файле (glob) |
| `path` + `rule` | Разрешить конкретное правило в файле |

### Как получить fingerprint

```bash
vendor/bin/git-secret-guard staged --show-fingerprint
```

Fingerprint — SHA256-хеш от `rule_id:file_path:line_content`. Секрет не хранится в конфиге.

## Exit-коды

| Код | Значение |
| :--- | :--- |
| `0` | Секреты не найдены |
| `1` | Найдены потенциальные секреты |
| `2` | Ошибка сканера или конфига |

## CI — второй уровень защиты

Pre-commit hook можно обойти через `--no-verify`. Рекомендуется второй слой:

### Вариант: scan-range в CI

```yaml
# .github/workflows/secret-scan.yml
name: Secret Scan
on: [push, pull_request]
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.4'
      - run: composer install --dev
      - run: vendor/bin/git-secret-guard scan-range origin/main...HEAD
```

### Вариант: Gitleaks / TruffleHog

Для более полного покрытия рекомендуется подключить зрелый сканер в CI:

- [Gitleaks](https://github.com/gitleaks/gitleaks) — широкий набор правил
- [TruffleHog](https://github.com/trufflesecurity/trufflehog) — проверка с верификацией

`git-secret-guard` — быстрый первый слой; Gitleaks/TruffleHog — полный второй слой в CI.

## Что делать при срабатывании

1. **До commit** (сканер сработал) — удалите секрет из staged diff, используйте переменные окружения или vault.
2. **После push** (секрет уже в истории) — **считайте секрет скомпрометированным**:
   - Revoke/rotate секрет немедленно.
   - Очистите историю Git (`git filter-repo`).
   - Проверьте PR caches и GitHub refs.

## Ограничения

- Проверяется только staged diff (не весь репозиторий) — для скорости.
- `--no-verify` обходит hook — используйте CI / GitHub Push Protection как второй слой.
- Самописный scanner имеет меньше coverage, чем Gitleaks/TruffleHog — рекомендуется гибридный подход.

## Ссылки

- [Коммиты](commits.md)
- [Pull Request](pull-request.md)
- [Релизы](release.md)

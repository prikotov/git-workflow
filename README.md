# git-workflow

## Правила Git-процесса для AI-агентов

Командная работа в Git требует единых правил: как называть ветки, форматировать коммиты, проводить ревью и выпускать релизы.

Пакет содержит правила Git-процесса: ветки, коммиты (Conventional Commits), пулреквесты, кодревью, релизы, деплой. Правила описаны в документации, передаваемой AI-агенту в качестве контекста.

---

## Правила

- **Ветки** — как назвать ветку для задачи, релиза или хотфикса; когда удалить
- **Коммиты** — Conventional Commits: тип, scope, subject — чтобы история читалась как журнал изменений
- **Пулреквесты** — от создания до мержа: проверки, ревью, squash
- **Кодревью** — что проверять, замечания, апрув
- **Релизы** — SemVer, release-ветки, CHANGELOG
- **Деплой** — когда и как развёртывать
- **Чеклисты** — пошаговые списки для релиза, хотфикса и деплоя
- **Секреты** — защита от коммита токенов, паролей, ключей

Полное содержание: [`docs/git-workflow/index.md`](docs/git-workflow/index.md).

---

## Secret Scanning

Пакет поставляет pre-commit hook для [Gitleaks](https://github.com/gitleaks/gitleaks) — зрелого OSS-сканера секретов с 100+ правилами детекции (AWS, GitHub, GCP, JWT, SSH и др.).

```bash
# Установить Gitleaks
brew install gitleaks  # macOS

# Установить хуки и конфиг
php vendor/bin/git-workflow-init --hooks
cp vendor/prikotov/git-workflow/templates/gitleaks.toml .gitleaks.toml
```

Детали: [`docs/git-workflow/secrets.md`](docs/git-workflow/secrets.md).

---

---

## Установка в проект

```bash
composer require --dev prikotov/git-workflow
```

### Копирование правил в проект

```bash
php vendor/bin/git-workflow-init --hooks
```

Флаг `--hooks` установит git-хуки (`commit-msg`, `pre-commit`) в `.git/hooks/`. Существующие файлы не перезаписываются; флаг `--force` включает перезапись.

Без `--hooks` — только документация:

```bash
php vendor/bin/git-workflow-init
```

Пример с нестандартным путём:

```bash
php vendor/bin/git-workflow-init /path/to/project --docs-path=docs/flow --force
```

---

## License

[MIT](LICENSE)

# rio-pub-test

Тестовый npm-пакет для отработки pipeline: feature branch → PR → merge в `main` → релизный тег → автопубликация на npm через GitHub Actions (npm Trusted Publisher / OIDC).

## Git-flow

1. Работа ведётся в feature-ветках от `main`: `git checkout -b feature/xxx`.
2. Изменения попадают в `main` только через Pull Request (проверяется workflow `CI`).
3. Публикация на npm НЕ выполняется вручную — она происходит автоматически по релизному тегу.

## Как сделать релиз

```bash
git checkout main
git pull

# patch | minor | major — обновляет version в package.json,
# коммитит и создаёт локальный git-тег вида vX.Y.Z
npm version patch

# пушим коммит с версией и сам тег
git push --follow-tags
```

Пуш тега `v*.*.*` запускает workflow `.github/workflows/publish.yml`, который:

1. Устанавливает зависимости и прогоняет тесты.
2. Проверяет, что версия в `package.json` совпадает с тегом.
3. Публикует пакет на npm через **Trusted Publisher (OIDC)** — без npm-токенов в секретах репозитория.

## Настройка Trusted Publisher на npm

В настройках пакета на npmjs.com (Publishing access → Trusted Publisher) нужно указать:

- **Repository**: `<owner>/rio-pub-test`
- **Workflow filename**: `publish.yml`
- **Environment**: (опционально, не используется в этом прототипе)

Подробнее: https://docs.npmjs.com/trusted-publishers

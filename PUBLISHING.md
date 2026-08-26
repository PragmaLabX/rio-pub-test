# Публикация rio-pub-test на npm: пошаговая настройка

Этот документ описывает полный путь: от создания аккаунта на npm до
автоматической публикации по тегу через GitHub Actions с использованием
**Trusted Publisher (OIDC)**, плюс набор security-хардненингов корпоративного
уровня для этого pipeline.

---

## 1. Аккаунт npm и его защита

1. Зарегистрируйтесь на https://www.npmjs.com (если ещё нет аккаунта).
2. **Рекомендуется включить 2FA**: Account Settings → *Two-Factor
   Authentication* → режим **"Auth and Publish"**.
   - Это НЕ техническое требование npm или Trusted Publisher — `npm login`
     и `npm publish` прекрасно работают и без 2FA (режим `Disabled` /
     `Auth-only`). Для тестового аккаунта, который используется только для
     обкатки этого pipeline, можно спокойно пропустить этот шаг.
   - Но для реального/корпоративного аккаунта, под которым публикуется
     боевой пакет, это стоит включить: без 2FA возможность byпасса
     публикации через granular access token с `bypass 2FA` резко повышает
     риск захвата пакета (классический вектор supply-chain атак:
     event-stream, ua-parser-js, colors.js и т.д.).
3. Если пакет разрабатывается от лица команды/компании — заведите
   **npm Organization**, а не публикуйте от личного аккаунта. Это даёт
   пофайловый контроль доступа (`npm team`) и не привязывает пакет к одному
   человеку.

---

## 2. Первая публикация (вручную, разово)

Trusted Publisher настраивается **в настройках уже существующего пакета**,
поэтому первый релиз нужно опубликовать вручную одним из способов ниже.
Дальше все последующие релизы будут идти только через тег → GitHub Actions —
руками `npm publish` больше не понадобится.

```bash
cd /Users/brooklyn/dev/npm-pub

# логин потребует код 2FA
npm login

# публикация версии из package.json (сейчас 0.1.0)
npm publish
```

Проверить результат: https://www.npmjs.com/package/rio-pub-test

> Если не готовы включать интерактивный 2FA-логин прямо сейчас, альтернатива —
> `npm stage publish` (staged publish), но подтверждение (`npm stage approve`)
> всё равно потребует 2FA.

---

## 3. Настройка Trusted Publisher (OIDC) для GitHub Actions

Как только пакет `rio-pub-test` появился на npm:

1. Перейдите на https://www.npmjs.com → **Packages** → **rio-pub-test** →
   **Settings** → **Trusted publishing**.
2. Нажмите **GitHub Actions** под "Select your publisher".
3. Заполните поля:

   | Поле | Значение |
   |---|---|
   | Organization or user | `PragmaLabX` |
   | Repository | `rio-pub-test` |
   | Workflow filename | `publish.yml` (только имя файла, не путь `.github/workflows/publish.yml`) |
   | Environment name | `npm-release` (см. раздел 5 — важно указать то же имя, что и в workflow) |
   | Allowed actions | `npm publish` |

4. Сохраните.

**Требования (уже учтены в workflow этого репозитория):**

- npm CLI **>= 11.5.1** — workflow сам подтягивает `npm install -g npm@latest`.
- Node.js **>= 22.14.0** — используется `node-version: 22.x`.
- В workflow должен быть `permissions: id-token: write` — уже добавлено в
  [`publish.yml`](.github/workflows/publish.yml).

После этого шага токены (`NPM_TOKEN`) в секретах репозитория **не нужны и не
должны создаваться** — публикация авторизуется через OIDC-обмен между GitHub
Actions и npm на каждый запуск, токен не хранится нигде постоянно.

---

## 4. Как теперь делать релиз

```bash
git checkout main
git pull

npm version patch   # patch | minor | major

git push --follow-tags
```

`npm version` обновит `package.json`, создаст коммит и локальный тег
`vX.Y.Z`. Пуш тега запускает [`publish.yml`](.github/workflows/publish.yml):
тесты → проверка, что версия в `package.json` совпадает с тегом →
`npm publish --provenance` через Trusted Publisher.

---

## 5. Security-хардненинг корпоративного уровня

Часть уже сделана в коде этого репозитория, часть требует настройки в
веб-интерфейсе GitHub (у меня нет токена/`gh` CLI для автоматического
применения — сделайте руками по шагам ниже, либо явно попросите настроить
через GitHub API, если выдадите токен с нужными правами).

### Уже сделано в workflow-файлах

- **Минимальные permissions**: `contents: read` везде, `id-token: write`
  только в job публикации — нет `contents: write`, боты не могут случайно
  что-то запушить.
- **Actions запиненены по commit SHA**, а не по плавающему тегу `@v4`
  (`actions/checkout@3d3c42e...  # v7.0.1`) — защита от supply-chain атаки
  через компрометацию самого Action.
- **`persist-credentials: false`** в checkout — раз job не пушит, токен
  GitHub не должен оставаться в `.git/config` рантайма.
- **`npm audit signatures`** — проверка криптографических подписей
  зависимостей в registry (npm provenance для транзитивных пакетов).
- **`npm audit --audit-level=high`** в CI — блокирует merge при известных
  high/critical уязвимостях в зависимостях.
- **`npm ci`** вместо `npm install` — детерминированная сборка строго по
  `package-lock.json`, без дрейфа версий.
- **Проверка версии тега против `package.json`** — исключает публикацию не
  той версии из-за рассинхрона тега и файла.
- **`--provenance`** при публикации — npm привяжет к пакету верифицируемый
  сертификат "собран из этого коммита этим workflow", виден на странице
  пакета как значок Provenance.
- **CODEOWNERS** ([`.github/CODEOWNERS`](.github/CODEOWNERS)) на
  `.github/workflows/` и `package.json` — изменения pipeline не проходят
  без ревью владельца. **Замените плейсхолдер `@PragmaLabX/maintainers`**
  на реальный `@username` или существующую GitHub-команду.

### Нужно настроить руками в GitHub UI

1. **Branch protection на `main`**
   (Settings → Branches → Add branch ruleset для `main`):
   - Require a pull request before merging, минимум 1 approval.
   - Require status checks to pass (выбрать job `CI / test`).
   - Require branches to be up to date before merging.
   - Restrict force pushes.
   - Restrict deletions.
   - **Do not allow bypass** даже для администраторов репозитория.

2. **Tag protection rules**
   (Settings → Rules → Rulesets → New tag ruleset):
   - Паттерн `v*.*.*`.
   - Ограничить, кто может создавать/пушить такие теги (только maintainers) —
     это единственный триггер публикации, его нельзя оставлять открытым всем
     контрибьюторам.

3. **GitHub Environment `npm-release`**
   (Settings → Environments → New environment → `npm-release`):
   - **Required reviewers** — добавить 1-2 человек, которые должны вручную
     подтвердить каждый запуск `publish.yml` перед тем, как он реально
     выполнится (доп. gate поверх Trusted Publisher).
   - **Deployment branches and tags** → Selected branches and tags → добавить
     `v*.*.*` (по умолчанию политика environment допускает ветки, а не теги —
     без этого шага workflow не сможет использовать environment).
   - Имя должно совпадать с `environment: npm-release` в
     [`publish.yml`](.github/workflows/publish.yml) и с полем "Environment
     name" в настройках Trusted Publisher на npm (шаг 3 выше).

4. **Code security** (Settings → Code security):
   - Dependabot alerts + Dependabot security updates — включить.
   - Secret scanning + **Push protection** — включить (блокирует пуш коммита
     с похожим на токен/ключ содержимым).

5. **(опционально, если это корпоративный стандарт) Require signed commits**
   в том же branch ruleset — гарантирует, что коммиты в `main` подписаны
   известным ключом автора.

---

## Резюме по разделению ответственности

| Кто | Что делает |
|---|---|
| Вы | npm-аккаунт, 2FA, первый ручной `npm publish`, Trusted Publisher на npm, branch/tag protection и Environment в GitHub UI |
| Я (уже сделано) | git-репозиторий, package.json, CI/publish workflows, pinning actions, CODEOWNERS, этот документ |
| Автоматика после настройки | `npm version` + `git push --follow-tags` → CI → review в Environment → publish с OIDC |

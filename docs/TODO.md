# Отложенные задачи

## CODEOWNERS

Убран `.github/CODEOWNERS` (был с плейсхолдером `@PragmaLabX/maintainers`,
которого, скорее всего, не существует как GitHub team — назначение
ревьюеров не работало бы).

Вернуться к этому, когда будет решено:
- Кто реально владеет `.github/workflows/` и `package.json` — конкретный
  `@username` или существующая/созданная GitHub team.
- Завести файл заново по тому же шаблону (`путь` → `владелец`).
- Включить в branch protection на `main` опцию **"Require review from Code
  Owners"**, иначе файл сам по себе ничего не блокирует, только
  подсказывает reviewers.

См. [PUBLISHING.md](../PUBLISHING.md), раздел 5, пункт "CODEOWNERS".

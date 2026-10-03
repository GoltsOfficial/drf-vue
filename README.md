# 📝 Соглашение о коммитах (Conventional Commits)

## Формат

<тип>(<область>): <краткое описание>

[тело коммита]

[футер]

## Типы коммитов

### feat

Новая функциональность для пользователя.
**Пример:** `feat(auth): add Google OAuth login`

### fix

Исправление ошибки в коде.
**Пример:** `fix(api): resolve memory leak in user controller`

### refactor

Изменение кода, которое не исправляет баг и не добавляет фичу.
**Пример:** `refactor(utils): simplify date parser`

### style

Правки форматирования: пробелы, точки с запятой, отступы.
**Пример:** `style: fix indentation in main.js`

### docs

Изменения только в документации.
**Пример:** `docs: update installation guide`

### test

Добавление или изменение тестов.
**Пример:** `test(auth): add unit tests for login`

### chore

Рутинные задачи: обновление зависимостей, конфигов, сборки.
**Пример:** `chore: bump axios to 1.6.0`

## Правила заголовка

1. Длина — не более 72 символов
2. Повелительное наклонение: `add`, а не `added`
3. Без точки в конце
4. Один коммит — одна логическая правка

| Тип        | Когда использовать              | Пример                              |
|------------|---------------------------------|-------------------------------------|
| `feat`     | Новая фича                      | `feat: add dark mode`               |
| `fix`      | Исправление бага                | `fix: correct typo in header`       |
| `refactor` | Рефакторинг без смены поведения | `refactor: extract helper function` |
| `style`    | Форматирование, пробелы         | `style: run prettier`               |
| `docs`     | Документация                    | `docs: add API examples`            |
| `test`     | Тесты                           | `test: cover edge cases`            |
| `chore`    | Зависимости, конфиги, сборка    | `chore: update eslint`              |
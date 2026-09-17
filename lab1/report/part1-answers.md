# Часть 1. Работа с Git-flow — результаты

Полный вывод команд: [part1-commands.txt](part1-commands.txt).

## Итоговый граф

```
* cc0b953 Fix duplicate power declaration and missing test imports
* b6d8b7b Fix incorrect test case            ← cherry-pick из 3e4b76d
* 4c7ae63 Complete power function implementation
* 83bbbd6 Add subtract and divide functions with tests
| * 63fde9d Add test for power function
| * 3e4b76d Fix incorrect test case
|/
* a980490 Update README with features list  ← hotfix/readme-update, слит в main (fast-forward)
* de54479 Initial commit: project structure
```

## Ответы на вопросы задачи 1.5

**Сколько всего коммитов в проекте?**
8 (`git rev-list --all --count`): 6 в ветке `feature/calculator-improvements`
и ещё 2 только в `feature/testing-improvements`.

**Какие ветки существуют в проекте?**
- `main`
- `feature/calculator-improvements`
- `feature/testing-improvements`

Ветка `hotfix/readme-update` была слита в `main` и удалена (`git branch -d`).

**Какой коммит был cherry-picked?**
`3e4b76d Fix incorrect test case` из `feature/testing-improvements`.
В `feature/calculator-improvements` он появился как новый коммит `b6d8b7b`
с другим SHA (у него другой родитель); флаг `-x` добавил в сообщение строку
`(cherry picked from commit 3e4b76d…)`.

## Замечания к методичке

1. После `git stash pop` и замены через `sed` в `src/app.js` остаются две
   функции `power` — пустая заготовка и реализация. Jest (Babel) отказывается
   разбирать файл: `Identifier 'power' has already been declared`.
2. В `tests/app.test.js` импортируются только `add, multiply, greet`,
   поэтому тесты `subtract` / `divide` / `power` падают с `ReferenceError`.

Обе проблемы исправлены отдельным коммитом `cc0b953`; после него
`npm test` — 7 passed, 7 total.

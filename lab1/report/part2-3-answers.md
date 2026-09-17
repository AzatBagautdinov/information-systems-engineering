# Части 2–3. CI/CD Pipeline с GitHub Actions — результаты

- Репозиторий: https://github.com/AzatBagautdinov/information-systems-engineering
- Actions: https://github.com/AzatBagautdinov/information-systems-engineering/actions
- Pull Request: https://github.com/AzatBagautdinov/information-systems-engineering/pull/1
- Вывод команд: [part2-commands.txt](part2-commands.txt), [part3-commands.txt](part3-commands.txt)

## Созданные файлы

| Файл | Задача | Назначение |
|------|--------|------------|
| `.github/workflows/lab1-ci.yml` | 2.2 | Базовый pipeline: тесты на Node 18/20/22 + сборка Docker-образа |
| `lab1/Dockerfile`, `lab1/.dockerignore` | 2.3 | Многостадийная сборка: `development` → `test` → `production` |
| `.github/workflows/lab1-advanced-ci.yml` | 2.4 | Проверки качества, coverage, audit, matrix, публикация образа в ghcr.io, deploy |
| `lab1/package.json` | 2.5 | Скрипты `test:coverage`, `build`, `lint`, настройки Jest |
| `lab1/tests/integration.test.js` | 2.5 | Интеграционный тест запуска приложения |

## Отличия от методички

Репозиторий общий для всех лабораторных по дисциплине, проект лежит в `lab1/`.
GitHub читает workflow только из корневой `.github/workflows/`, поэтому:

- workflow названы `lab1-*.yml` и запускаются только при изменениях в `lab1/**` (`paths`);
- `defaults.run.working-directory: lab1`, у `upload-artifact` путь `lab1/coverage/`,
  у `build-push-action` — `context: lab1`.

Исправленные ошибки методички:

1. **Dockerfile**: первая стадия не имела имени, а `FROM development AS test` ссылается
   на неё — сборка невозможна. Добавлено `AS development`.
2. `npm install --only=production` устарел → `npm install --omit=dev`.
3. **Мультиплатформенная сборка** `linux/arm64` на amd64-раннере требует эмуляции —
   добавлен шаг `docker/setup-qemu-action@v3`; также явно указан `target: production`.
4. **Имя образа** в ghcr.io должно быть в нижнем регистре, а `github.repository`
   содержит `AzatBagautdinov` → `IMAGE_NAME` задан явно в нижнем регистре.
5. **Node 16** снят с поддержки → matrix `[18.x, 20.x, 22.x]`.
6. Включён кэш npm (`cache: npm`) вместо `cache: ''`.

## Задача 3.1. Мониторинг выполнения (push в main, коммит b0be0c2)

**Lab1 CI/CD Pipeline** — всего 43 с

| Задача | Результат | Время |
|--------|-----------|-------|
| test (18.x) | ✓ | 13 с |
| test (20.x) | ✓ | 14 с |
| test (22.x) | ✓ | 10 с |
| docker-build | ✓ | 22 с |

**Lab1 Advanced CI/CD Pipeline** — всего 2 мин 13 с

| Задача | Результат | Время |
|--------|-----------|-------|
| quality-checks | ✓ | 14 с |
| build-and-test (18.x) | ✓ | 12 с |
| build-and-test (20.x) | ✓ | 13 с |
| build-and-test (22.x) | ✓ | 8 с |
| docker-operations | ✓ | 90 с (из них сборка amd64+arm64 и push — 67 с) |
| deploy-production | ✓ | 3 с |
| deploy-staging | пропущена (`if: develop`) | — |

Наблюдения:
- задачи matrix выполняются параллельно, `needs` задаёт порядок стадий;
- самая долгая стадия — сборка образа под arm64 через эмуляцию QEMU;
- на push в `feature/pipeline-testing` и на PR `docker-operations` и deploy не выполняются —
  условие `if` пропускает их (только push в main/develop);
- покрытие кода (`coverage-report` в артефактах запуска): 66.66 % строк `src/app.js`.

## Задача 3.2. Работа с ошибками

1. Коммит `Introduce test failure for pipeline testing` (`toBe(5)` → `toBe(6)`) —
   все три запуска завершились со статусом **failure**:
   - Advanced: упал шаг `Run tests with coverage` в `quality-checks`,
     все зависимые задачи (`build-and-test`, `docker-operations`, deploy) не запускались;
   - базовый: упала `test (22.x)`, остальные задачи matrix **отменены**
     (fail-fast по умолчанию), `docker-build` не запускался;
   - в логе: `● Math functions › adds 2 + 3 to equal 5 — Expected: 6, Received: 5`;
   - в Pull Request #1 появилась красная отметка проверок.
2. Коммит `Fix intentionally broken test` — все запуски снова **success**.

---

# Контрольные вопросы

### 1. Разница между `git merge` и `git rebase`. Когда что использовать?

`merge` объединяет истории двух веток: если ветки разошлись, создаётся
merge-коммит с двумя родителями, существующие коммиты не меняются.
`rebase` «переносит» коммиты ветки поверх другой ветки, создавая их заново
(с новыми SHA) — история становится линейной, но переписывается.

- `merge` — для слияния готовых веток в общие (`main`, `develop`), для
  опубликованных веток, когда важно сохранить факт слияния.
- `rebase` — для актуализации своей локальной/личной feature-ветки относительно
  `main` перед PR (в работе: `git rebase main` в `feature/calculator-improvements`).
  Нельзя делать rebase веток, с которыми уже работают другие.

### 2. Что происходит при `git cherry-pick`? Примеры.

Git берёт изменения (diff) указанного коммита и применяет их к текущей ветке
как **новый** коммит с тем же сообщением, но другим SHA и родителем.
Примеры: перенести hotfix из `develop` в релизную ветку, не сливая всё остальное;
забрать один нужный коммит из чужой незавершённой ветки. В работе
исправление теста `Fix incorrect test case` было перенесено из
`feature/testing-improvements` без коммита с тестом `power`.

### 3. Как работает `git stash` и какие проблемы решает?

`git stash push` сохраняет незакоммиченные изменения (индекс и рабочую
директорию) в стек и возвращает рабочую копию к состоянию `HEAD`.
`git stash pop` / `apply` возвращает изменения, `list` показывает стек.
Решает проблему «нужно срочно переключиться на другую ветку, а текущая работа не
готова к коммиту» — в работе незаконченная функция `power` была отложена, чтобы
сделать hotfix README в `main`.

### 4. Преимущества CI/CD pipeline

- автоматическая проверка каждого изменения (тесты, линтер, audit) — ошибки
  находятся сразу, а не при релизе;
- воспроизводимая сборка в чистом окружении («у меня работает» исключено);
- быстрая обратная связь в Pull Request, запрет слияния сломанного кода;
- автоматическая и однообразная доставка (образ, деплой) без ручных шагов;
- история запусков и артефакты (coverage, логи) для анализа.

### 5. Структура workflow-файла GitHub Actions

- `name` — имя workflow;
- `on` — события-триггеры (`push`, `pull_request`, фильтры `branches`, `paths`);
- `env` — общие переменные; `defaults` — общие настройки шагов;
- `jobs` — задачи; у каждой: `runs-on` (раннер), `needs` (зависимости),
  `if` (условие), `strategy.matrix`, `permissions`, `environment`;
- `steps` — шаги задачи: `uses` (готовый action, например `actions/checkout@v4`)
  или `run` (shell-команды), параметры `with`, `env`.

Задачи выполняются параллельно, если нет `needs`; шаги внутри задачи — последовательно.

### 6. Как обеспечить безопасность в CI/CD?

- секреты хранить в GitHub Secrets, не в коде; использовать `GITHUB_TOKEN`
  с минимальными `permissions` (в работе: `contents: read`, `packages: write`);
- проверять зависимости (`npm audit`, Dependabot), фиксировать версии через lock-файл;
- закреплять версии сторонних actions (тег, лучше SHA);
- `environment` с правилами защиты (ручное подтверждение деплоя в production);
- защищать ветку `main`: обязательные проверки и review перед merge;
- в контейнерах — минимальный базовый образ, запуск не от root
  (в Dockerfile создан пользователь `nextjs`), без dev-зависимостей.

### 7. Что такое matrix build и зачем он нужен?

`strategy.matrix` запускает одну задачу несколько раз с разными параметрами
(версии Node.js, ОС и т.д.) параллельно. Позволяет убедиться, что приложение
работает во всех поддерживаемых окружениях, не дублируя описание задачи.
В работе — Node.js 18.x, 20.x, 22.x. По умолчанию `fail-fast: true`: при
падении одной комбинации остальные отменяются (видно в задаче 3.2).

### 8. Как работают Docker-контейнеры в контексте CI/CD?

Приложение упаковывается в образ вместе с окружением, поэтому одинаково
запускается в CI, на тестовом и продуктовом сервере. В pipeline:
сборка образа (многостадийная: стадия `test` запускает тесты внутри образа,
`production` содержит только нужное), проверка запуска контейнера,
публикация образа в реестр (ghcr.io) с тегами по ветке и SHA, затем деплой
использует уже проверенный образ. Buildx с кэшем `type=gha` ускоряет
повторные сборки, QEMU позволяет собирать образы под другие архитектуры.

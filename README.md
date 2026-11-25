# course-catalog-api

Бэкенд каталога курсов: FastAPI, SQLAlchemy 2.1, PostgreSQL, Alembic и GraphQL на Strawberry как второй вход в то же API (Лабораторные работы 4–8).

## How to run

Требуются Python 3.11+ и PostgreSQL 16+.

Один раз от имени суперпользователя `postgres` создать роль и базу:

```sql
CREATE ROLE catalog_app LOGIN PASSWORD 'change-me';
CREATE DATABASE course_catalog OWNER catalog_app;
```

Затем из корня проекта:

```bash
python -m venv .venv
# Windows (PowerShell): .venv\Scripts\Activate.ps1
# macOS / Linux: source .venv/bin/activate

pip install -r requirements.txt
cp .env.example .env

alembic upgrade head
psql -h localhost -U catalog_app -d course_catalog -v ON_ERROR_STOP=1 -f seed.sql
fastapi dev main.py
```

REST-документация (Swagger): http://localhost:8000/docs. GraphiQL: http://localhost:8000/graphql.
Пароль `change-me` учебный; реальные пароли не коммитятся в репозиторий.

## What changed in the API

- **Сервисный слой и Problem Details (RFC 9457):** общие бизнес-правила и обработка `IntegrityError` вынесены в `services.py` (`ServiceError`), а `problem.py` переводит ошибки REST в формат `application/problem+json` с полями `type`, `title`, `status`, `detail`, `instance` (`409` и `404`).
- **Второй вход `/graphql`:** подключён роутер Strawberry (`graphql_api.py`) с типами `Course`, `Student`, `Enrollment` и per-request контекстом (`get_db` + `DataLoader`). При открытии `/graphql` в браузере доступен редактор GraphiQL, а клиенты отправляют `POST` с JSON.
- **Мутации и `userErrors`:** добавлены мутации `enroll` и `likeCourse`. Ожидаемые ошибки правил (`ALREADY_ENROLLED`, `COURSE_NOT_FOUND`, `STUDENT_NOT_FOUND`) возвращаются внутри `data` в массиве `userErrors`, а ошибки синтаксиса/схемы или сбои — в верхнеуровневом ключе `errors`.
- **Счётчик SQL (`X-SQL-Queries`):** middleware `counter.py` считает все SQL-запросы через событие SQLAlchemy `before_cursor_execute` и добавляет заголовок `X-SQL-Queries` в каждый HTTP-ответ; с помощью `DataLoader` проблема N+1 устранена (3 запроса вместо 7 и 27).
- **Защита GraphQL:** ограничена глубина (`QueryDepthLimiter(max_depth=5)`), число алиасов (`MaxAliasesLimiter(max_alias_count=10)`), проверены аргументы `limit` (1..100) и `id`, неожиданные исключения скрыты через `MaskErrors`, а интроспекция управляется переменной `GRAPHQL_INTROSPECTION` в `.env`.
- **Новые REST-маршруты:** добавлены `GET /students/{student_id}` и `GET /students/{student_id}/enrollments` для экрана «Мои курсы» и сравнительных замеров в `measure.py`.

## Example queries

Запрос «Мои курсы» (в панели Variables GraphiQL: `{"id": "2"}`):

```graphql
query MyCourses($id: ID!) {
  student(id: $id) {
    name
    enrollments {
      id
      course { title credits level }
    }
  }
}
```

Мутация `enroll` (в Variables: `{"input": {"courseId": "web-security", "studentId": "3"}}`) и ответ с `userErrors` при повторной записи:

```graphql
mutation Enroll($input: EnrollInput!) {
  enroll(input: $input) {
    enrollment { id enrolledAt course { title } }
    userErrors { code message }
  }
}
```

```json
{
  "data": {
    "enroll": {
      "enrollment": null,
      "userErrors": [
        {
          "code": "ALREADY_ENROLLED",
          "message": "Student is already enrolled in this course"
        }
      ]
    }
  }
}
```

## REST vs GraphQL comparison

Экран «Мои курсы» студента `2`; цифры из `measure.py` на стартовых данных:

| Criterion | REST | GraphQL | Source |
|-----------|------|---------|--------|
| Requests and rounds per screen | 5 HTTP-запросов в 2 раунда | 1 HTTP-запрос в 1 раунд | `measure.py`, step 8 |
| Bytes in the responses | 1145 байт (передаются лишние поля: `description`, `likes`, `isElective`, `enrolledAt`) | 339 байт (только запрошенные `name`, `id`, `title`, `credits`, `level`) | `measure.py`, step 8 |
| SQL queries per screen | 5 SQL-запросов (студент + записи + 3 отдельных запроса курсов) | 3 SQL-запроса (по одному на уровень благодаря DataLoader) | `measure.py`, `X-SQL-Queries` |
| SQL growth with more students | Растёт линейно с числом курсов на студента (по отдельному `GET /courses/{id}`) | Без DataLoader: 7 → 27 (и 7 → 24); с DataLoader: стабильно 3 запроса и на 3, и на 20 студентах | `try_n_plus_1.py`, experiments 4–5 |
| Error in the query text | HTTP `404` / `422` (`res.ok === false`), легко ловится по статусу | HTTP `200 OK`, `data: null`, описание в ключе `errors` — клиент обязан явно проверять `body.errors` | experiments 1 and 8 |
| Expected rule error (repeated enrollment) | HTTP `409 Conflict`, `application/problem+json`, `type: .../already-enrolled` | HTTP `200 OK`, `data.enroll.enrollment: null`, `userErrors[0].code: "ALREADY_ENROLLED"` | steps 1 and 4 |
| HTTP cache | Работает из коробки для `GET` (браузер, CDN, `ETag` / `Cache-Control`) | По умолчанию `POST /graphql` не кэшируется HTTP-кэшем (нужны persisted queries или клиентский кэш) | lecture section 14 |
| Protection from "expensive" queries | Ограничено самим маршрутом и фиксированным `page_size` | Нужны явные лимиты на глубину (`max_depth=5`), алиасы (`max_alias_count=10`), `limit`, `MaskErrors` и отключение интроспекции в проде | experiments 6 and 7 |

**Conclusion.** Простые плоские экраны и публичные справочники (например, список курсов `GET /courses` и карточка одного курса `GET /courses/{id}`) лучше оставить на REST: они укладываются в 1 `GET`-запрос, легко кэшируются по HTTP и не требуют сложной защиты от произвольных запросов. Композитные экраны со связанными сущностями вроде «Мои курсы» выгоднее перевести на GraphQL: он сократил число сетевых раундов с 2 до 1, число запросов с 5 до 1, объём ответа в 3,4 раза (с 1145 до 339 байт) и число SQL-запросов с 5 до 3.

## Experiment log

| # | Experiment | What I saw | Cause | How I fixed it |
|---|------------|------------|-------|----------------|
| 1 | Опечатка в имени поля (`titel`) и аргумент в `snake_case` (`is_elective`) | HTTP `200 OK`, `data: null`, в `errors` подсказки `Did you mean 'title'...` и `Did you mean 'isElective'?` | Движок валидирует запрос по схеме до выполнения; Strawberry переводит имена Python из `snake_case` в `camelCase` | Исправил имена на `title` и `isElective` и запомнил, что статус `200` не гарантирует успех — нужно проверять `errors` |
| 2 | Убрал `\| None` у запроса `course` и запросил `"nope"` | Ошибка `Cannot return null for non-nullable field Query.course.`; при алиасах `a` и `b` обнулился весь `data` | Правило null propagation: `null` в поле с `!` поднимается до ближайшего nullable-родителя (до всего `data`) | Вернул `CourseType \| None`, чтобы отсутствующий курс возвращал `null` без поломки соседних полей |
| 3 | Отправил мутацию `likeCourse` методом `GET` | HTTP `400 Bad Request` (обычный текст): `mutations are not allowed when using GET`; лайки не изменились | По стандарту HTTP `GET` должен быть безопасным и не менять состояние сервера | Отправляю мутации только методом `POST` с `Content-Type: application/json` |
| 4 | `more_students.sql`: 3 и 20 студентов, счётчик `X-SQL-Queries` | На 3 студентах: 7 и 7 SQL-запросов; на 20 студентах: 27 (`1+20+6`) и 24 (`1+6+17`) SQL-запроса | Ленивая загрузка SQLAlchemy в резолверах вызывает N+1 — отдельный `SELECT` на каждый объект | Заменил ленивые атрибуты на `DataLoader` (`WHERE ... IN (...)`), число запросов упало до 3 и 3 |
| 5 | Сломал batch-функцию DataLoader (возврат строк без сортировки по ключам) | На `students`: `WrongNumberOfResultsReturned (expected: 3, received: 6)`; на `enrollments`: ошибок нет, но 4 из 6 записей получили чужие курсы | DataLoader сопоставляет результаты с ключами по позиции; без группировки `groups[key]` порядок из БД не совпадает с `keys` | Вернул рабочую версию через `git restore graphql_api.py` (сопоставление по значению ключа через `defaultdict`) |
| 6 | 6 опасных запросов и 12 лайков до и после защиты | До защиты: цикл глубины 6 и 12 алиасов выполнились (24 SQL-запроса), а плохие `limit`/`id` слили SQL и стек psycopg. После защиты: цикл, 12 алиасов (0 SQL), `limit -1` и `id "abc"` дали короткие ошибки, а 20-значный `id` скрыт как `Unexpected error.` | Без лимитов клиент сам управляет стоимостью запроса, а необработанные исключения утекают в `errors` | Добавил `QueryDepthLimiter(5)`, `MaxAliasesLimiter(10)`, проверки `check_limit`/`parse_id` и `MaskErrors`; восстановил данные `seed.sql` |
| 7 | `GRAPHQL_INTROSPECTION=false`, опечатка `cours` | Запрос `__schema` отклонён (`GraphQL introspection has been disabled`), обычные запросы работают, но опечатка `cours` всё ещё выдала `Did you mean 'course' or 'courses'?` | Отключение интроспекции скрывает карту схемы, но подсказки валидатора всё ещё раскрывают имена полей — интроспекция не заменяет авторизацию | Вернул `GRAPHQL_INTROSPECTION=true` для разработки в GraphiQL |
| 8 | Сбои на фронтенде: опечатка `titel`, убранная проверка `errors`, выключенный API | 1) `GraphQLRequestError: Cannot query field 'titel'`; 2) `TypeError: Cannot destructure property 'student' ... as it is null`; 3) `TypeError: fetch failed` | Без проверки `body.errors` при статусе `200` код принимает `data: null` за успех и падает с непонятным `TypeError` | Восстановил файлы через `git restore .` и оставил обязательную проверку `if (body.errors?.length)` в `lib/graphql.ts` |

## Decisions

- **Единый слой правил в `services.py`:** правила и обработка `IntegrityError` написаны один раз в `services.py` и бросают `ServiceError` с кодом. REST (`problem.py`) переводит его в HTTP-статус и `application/problem+json`, а GraphQL (`graphql_api.py`) — в `userErrors`. Дублировать проверки в двух слоях нельзя, иначе правила неизбежно разойдутся.
- **Per-request `DataLoader` и сопоставление по ключам:** лоадеры создаются заново в `Context` на каждый HTTP-запрос, потому что DataLoader кэширует результаты по ключу — один общий лоадер на всё приложение отдавал бы чужие или устаревшие данные. Сопоставление через словарь `groups[key]` обязательно, потому что SQL `IN (...)` не гарантирует порядок строк, и тихая подмена чужих курсов опаснее явного исключения.
- **Двухслойная защита (`check_limit`/`parse_id` + `MaskErrors`):** для предсказуемых ошибок ввода клиента мы сами кидаем понятный `GraphQLError` (`Invalid ID: 'abc'`, `limit must be between 1 and 100`), чтобы клиент мог исправить запрос. А `MaskErrors` страхует от непредвиденных сбоев (например, переполнение `bigint`), заменяя их на `Unexpected error.` и скрывая SQL и внутренности БД.
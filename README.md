# course-catalog-api

Бэкенд каталога курсов: FastAPI, SQLAlchemy 2.1, PostgreSQL и миграции Alembic (Лабораторные работы 4–7).

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

Документация API: http://localhost:8000/docs. Пароль `change-me` учебный; реальные пароли в репозиторий не добавляются.

## What changed in the API

- **Хранение в PostgreSQL:** курсы, студенты и записи перенесены из списков в памяти (`data.py` удалён) в таблицы БД — лайки и новые записи сохраняются после перезапуска сервера.
- **Изменение `POST /enrollments`:** вместо `studentName` теперь передаётся `studentId` (внешний ключ на `students`), а в ответе возвращаются `id`, `courseId`, `studentId` и `enrolledAt`.
- **Новые маршруты и `JOIN`:** добавлены `POST /students` (201) и `GET /stats/courses` (статистика записей по курсам, включая курсы с 0 записей). В `GET /enrollments` через `JOIN` подтягиваются `courseTitle` и `studentName`.
- **Поле `level` и фильтр:** миграцией `0002` добавлено обязательное поле `level` (`beginner` / `intermediate` / `advanced`) и необязательный фильтр `?level=` в `GET /courses`.
- **Ошибки БД как HTTP-ответы:** нарушения ограничений (`IntegrityError`) определяются по имени ограничения и возвращают `409 Conflict` (дубликат email или повторная запись) и `404 Not Found` (несуществующий курс или студент), а неизвестные ошибки остаются `500`.

## Migrations

| Command | What it does |
|---------|--------------|
| `alembic upgrade head` | применяет все недостающие миграции |
| `alembic downgrade -1` | откатывает последнюю миграцию |
| `alembic current` | показывает текущую версию базы |
| `alembic history` | показывает цепочку миграций |
| `alembic revision --autogenerate -m "text"` | создаёт черновик миграции по разнице моделей и БД |
| `alembic check` | проверяет, не разошлись ли модели с базой |
| `alembic upgrade 0001:0002 --sql` | выводит SQL миграции без выполнения |
| `alembic stamp head` | записывает версию в `alembic_version` без запуска миграций |

## Experiment log

| # | Experiment | What I saw | Cause | How I fixed it |
|---|------------|------------|-------|----------------|
| 1 | Изменил шаблон `uq` в `naming_convention` | `check_models.py` показал расхождения: `students_email_unique` против `students_email_key` и `enrollments_course_id_unique` против `enrollments_course_id_student_id_key` | Шаблон `uq` перестал совпадать с именами ограничений в PostgreSQL | Вернул `"uq": "%(table_name)s_%(column_0_N_name)s_key"` и проверил `check_models.py` (`none`) |
| 2 | Убрал `min_length=2` из `StudentIn` и отправил имя `N` | `500 Internal Server Error`, в логе сервера `IntegrityError (CheckViolation)` по `students_name_check` | Без проверки Pydantic запрос дошёл до БД; `CHECK` защитил данные, но его имени нет в `CONSTRAINT_ERRORS` | Вернул `Field(min_length=2)` в `StudentIn`, чтобы Pydantic возвращал `422` до БД |
| 3 | Опечатка в имени ограничения в `CONSTRAINT_ERRORS` | `500 Internal Server Error` при дубликате email; в логе `UniqueViolation` на `students_email_key` | В словаре был ключ `"students_email_uniq"` вместо `"students_email_key"` | Исправил ключ на `"students_email_key"` и перепроверил `try_errors.py` (5 из 5) |
| 4 | Убрал `NOT NULL` у `courses.title` в БД и запустил `alembic check` | `FAILED: New upgrade operations detected: modify_nullable` (и `modify_type` при смене типа на `varchar(100)`) | Ручной `ALTER TABLE` привёл к расхождению базы и моделей | Вернул `SET NOT NULL` и `TYPE text`, после чего `alembic check` прошёл чисто |
| 5 | Автоматическая миграция `0002` на таблице с данными | `IntegrityError (NotNullViolation): column "level" of relation "courses" contains null values`; версия осталась `0001` | `autogenerate` создал `ADD COLUMN level TEXT NOT NULL` без значения по умолчанию на непустой таблице | Переписал `0002` в 3 шага: `nullable=True` → заполнение через `UPDATE ... CASE` → `nullable=False` |
| 6 | Закомментировал `import models` в `alembic/env.py` | Автогенерация предложила удалить все 3 таблицы (`op.drop_table` для `enrollments`, `courses`, `students`) | Без импорта `models` метаданные `Base.metadata` оказались пустыми | Не применял миграцию, удалил файл `0003_oops.py` и вернул `import models` |
| 7 | Добавил поле `phone` в `Student` без миграции | `alembic check` выдал ошибку `Detected added column 'students.phone'` | Модель ушла вперёд относительно схемы базы данных | Удалил тестовое поле `phone` из модели и проверил `alembic check` |
| 8 | Создал две миграции от одной ревизии `0002` | `alembic heads` показал 2 головы (`0003` и `0004`), `upgrade head` упал с `Multiple head revisions are present` | Две параллельные ветки миграций от одного родителя | Объединил через `alembic merge heads`, затем удалил экспериментальные файлы `0003`–`0005` |

## Decisions

- **Имя ограничения вместо SQLSTATE и перехват после `commit`:** код `23505` одинаковый для всех `UNIQUE`, а `23503` — для всех внешних ключей, поэтому точно понять причину можно только по `constraint_name`. Ловить ошибку после `db.commit()` надёжнее, чем проверять заранее через `SELECT`, потому что между `SELECT` и `INSERT` другой запрос может успеть занять тот же email (состояние гонки).
- **Атомарный `UPDATE courses SET likes = likes + 1`:** если читать `likes` в Python, прибавлять 1 и записывать обратно, два одновременных запроса прочитают старое число и один лайк потеряется (lost update). При инкременте в SQL база сама блокирует строку и применяет оба обновления по очереди.
- **Трёхшаговая миграция `0002`:** PostgreSQL не позволяет добавить колонку `NOT NULL` без значения по умолчанию в таблицу, где уже есть строки. Разбиение на `ADD COLUMN (nullable=True)` → `UPDATE` существующих строк → `ALTER COLUMN SET NOT NULL` внутри одной транзакции безопасно обновляет таблицу без потери данных.

## Migration 0002 self-check

| What to check | Result |
|---------------|--------|
| Goal and steps | Да — добавляется одна обязательная колонка `courses.level` в три понятных шага. |
| Data | Да — существующие 6 курсов заполняются по правилу `CASE WHEN is_elective THEN 'beginner' ELSE 'intermediate' END` до установки `NOT NULL`. |
| Renames and drops | Да — ничего не переименовывается и не удаляется в `upgrade()`, данные не теряются. |
| Rollback | Да — `downgrade()` удаляет `courses.level`; цикл `downgrade -1` → `upgrade head` проверен (с учётом того, что в продакшене откат сотрёт значения колонки). |
| Verification | Да — SQL проверен через `--sql`, откат и повторное применение прошли без ошибок, `alembic check` чистый. |
| History | Да — одна голова `0002 (head)`, миграция закоммичена вместе с моделями, схемами, кодом и `seed.sql`. |
| Application | Да — поле `level` добавлено в `CourseOut`, фильтр `?level=` добавлен в `GET /courses`, `seed.sql` обновлён. |
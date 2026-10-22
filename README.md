# Course Catalog API (FastAPI)

Проект бэкенда на FastAPI для каталога курсов (Pydantic, CORS, camelCase-алиасы, лайки и запись на курсы).

- **ФИО:** Кешубаев Нурмухамед
- **Группа:** IT3-2304
- **Лабораторная работа:** №4–5 (FastAPI + Next.js Integration)

## Инструкция по запуску
1. Создать виртуальное окружение: `python -m venv .venv`
2. Активировать окружение:
   - Windows (PowerShell): `.venv\Scripts\Activate.ps1`
   - Linux / macOS: `source .venv/bin/activate`
3. Установить зависимости: `pip install -r requirements.txt`
4. Запустить dev-сервер: `fastapi dev main.py`
5. Интерактивная документация (Swagger UI): `http://localhost:8000/docs`

## What changed in Lab 5 (Что изменилось в Lab 5)
1. **CORS (`CORSMiddleware`):** Разрешён точный источник фронтенда `http://localhost:3000` (без слеша на конце), а также все методы и заголовки (`allow_methods=["*"]`, `allow_headers=["*"]`). Это необходимо для прямых запросов из браузера (Client Components: фильтр SWR, кнопка лайка и форма записи).
2. **Контракт имен (`CamelModel`):** Модели наследуются от базового класса `CamelModel` с `alias_generator=to_camel`, `validate_by_name=True` и `validate_by_alias=True`. В Python сохраняется `snake_case` (`is_elective`, `course_id`, `student_name`), а в JSON отдаётся и принимается `camelCase` (`isElective`, `courseId`, `studentName`).
3. **Новые маршруты:**
   - `POST /courses/{course_id}/like` — увеличивает счётчик лайков курса на 1 и возвращает обновлённый объект курса (200 OK) или 404.
   - `POST /enrollments` — принимает JSON `{"courseId": "web-security", "studentName": "Aigerim"}`, проверяет длину имени (`min_length=2`) и существование курса, возвращает созданную запись со статусом `201 Created`: `{"courseId": "web-security", "studentName": "Aigerim", "id": 1}`.
   - `GET /enrollments` — возвращает список всех записей на курсы.
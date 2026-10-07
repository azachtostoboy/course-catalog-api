# Course Catalog API (FastAPI)

- **ФИО:** Кешубаев Нурмухамед
- **Группа:** IT3-2304
- **Лабораторная работа:** №4 (First Backend: FastAPI, Pydantic, and Depends)

## Инструкция по запуску
1. Создать виртуальное окружение: `python -m venv .venv`
2. Активировать его:
    - Windows: `.venv\Scripts\activate`
    - Linux/macOS: `source .venv/bin/activate`
3. Установить зависимости: `pip install -r requirements.txt`
4. Запустить dev-сервер: `fastapi dev main.py`
5. Интерактивная документация Swagger: `http://127.0.0.1:8000/docs`

## Проверенные эндпоинты:
- `GET /` -> `{"message": "Course Catalog API is running"}`
- `GET /courses` -> 6 курсов, сортировка по лайкам по умолчанию (`ai-integration` первый)
- `GET /courses?is_elective=true` -> 2 элективных курса
- `GET /courses?is_elective=false` -> 4 обязательных курса
- `GET /courses?sort=title` -> 6 курсов по алфавиту
- `GET /courses?page=2&page_size=2` -> курсы `web-security` и `backend-fastapi`
- `GET /courses/web-security` -> данные по курсу
- `GET /courses/nope` -> HTTP 404 `{"detail": "Course not found"}`
# Архитектура

## Компоненты

```mermaid
flowchart LR
    U([Ученик в браузере])
    TG([Telegram на телефоне])

    subgraph FE[frontend — React + Vite :5173]
        P[Страницы: Задания, Номер, Вход, Профиль]
    end

    subgraph BE[backend — модульный монолит]
        API[FastAPI :8000<br/>модули auth · tasks · progress]
        BOT[Telegram-бот<br/>aiogram, long polling]
    end

    DB[(PostgreSQL<br/>users, exam_numbers,<br/>tasks, attempts)]
    R[(Redis<br/>ключи входа, коды,<br/>счётчики попыток)]

    U --> P
    P -- "/api/... (JSON, JWT)" --> API
    API --> DB
    API --> R
    TG <--> BOT
    BOT --> DB
    BOT --> R
```

- **Модульный монолит.** Один backend, внутри — модули со своими таблицами и логикой:
  `auth` (пользователи, вход), `tasks` (номера и задачи), `progress` (попытки и статистика).
  Модуль `tasks` не знает, как считается «решено», — он получает от `progress` готовый подзапрос.
- **Слои внутри модуля:** `router` (HTTP, коды ответов) → `service` (логика) → `models` (таблицы).
  Схемы Pydantic в `schemas` — контракт API.
- **Бот** — отдельный процесс из того же кода: использует те же функции `auth.service`, что и API.
- **Запуск:** `docker compose up` поднимает `db`, `redis`, `api`, `bot`. `api` при старте применяет миграции и заполняет справочник номеров.

## Вход по QR-коду

```mermaid
sequenceDiagram
    autonumber
    actor U as Ученик
    participant S as Сайт
    participant A as API
    participant R as Redis
    participant B as Бот

    U->>S: «Войти»
    S->>A: POST /auth/telegram/start
    A->>R: SET login:qr:{ключ} = pending (TTL 300 с)
    A-->>S: ключ + ссылка t.me/бот?start=ключ
    S-->>U: QR-код
    loop каждые 2 секунды
        S->>A: GET /auth/telegram/status/{ключ}
        A-->>S: pending
    end
    U->>B: сканирует QR → /start ключ
    B-->>U: «Войти на сайт?» [Да] [Нет]
    U->>B: «Да»
    B->>A: (функция сервиса) найти/создать пользователя
    B->>R: login:qr:{ключ} = confirmed:{user_id}
    S->>A: GET /auth/telegram/status/{ключ}
    A->>R: DELETE login:qr:{ключ} (одноразовый)
    A-->>S: confirmed + JWT
    S-->>U: профиль
```

## Проверка ответа и статистика

```mermaid
sequenceDiagram
    actor U as Ученик
    participant S as Сайт
    participant A as API
    participant D as PostgreSQL

    U->>S: ответ «2 20» → «Проверить»
    S->>A: POST /tasks/8/check (Authorization: Bearer JWT)
    A->>D: задача 8 (правильный ответ)
    A->>A: сравнить: числа как числа, слова без регистра
    A->>D: INSERT attempts (если вошёл)
    A-->>S: is_correct, правильный ответ, разбор
    S-->>U: «Верно!», метка «✓ Решена», «решено 1 из 1»
```

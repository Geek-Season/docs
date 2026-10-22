# ERD — модель данных

Статус ✅ — таблица **уже есть** в базе (создана миграциями Alembic).
«план» — запланирована на следующие этапы.

| Сущность | Таблица | Что это | Статус |
|---|---|---|---|
| Пользователь | `users` | Ученик (позже — репетитор, админ), вход через Telegram | ✅ |
| Номер ЕГЭ | `exam_numbers` | Позиция в экзамене: «1», …, «19-21», …, «27» | ✅ |
| Задача | `tasks` | Прототип номера: условие, ответ, разбор | ✅ |
| Попытка | `attempts` | Один ответ ученика на задачу | ✅ |
| Группа | `groups` | Учебная группа репетитора | план |
| Участник группы | `group_members` | Связь ученик ↔ группа | план (служебная) |
| Задание | `assignments` | Набор задач для группы с дедлайном | план |
| Задача задания | `assignment_tasks` | Связь задание ↔ задача | план (служебная) |
| Сессия пробника | `exam_sessions` | Прохождение пробника учеником: срок, баллы | ✅ |
| Ответ в пробнике | `exam_answers` | Позиция варианта: задача, баллы за неё, ответ ученика | ✅ |
| Вариант репетитора | `exam_variants` | Сохранённый вариант, собранный репетитором | план |
| Задача варианта | `variant_tasks` | Связь вариант ↔ задача | план (служебная) |
| Достижение | `achievements` | Справочник: «Первый шаг», «Неделя без пропусков»… | ✅ |
| Полученное достижение | `user_achievements` | Связь ученик ↔ достижение, дата | ✅ (служебная) |
| Уведомление | `notifications` | Сообщение ученику и статус отправки в Telegram | ✅ |

Итого 15 таблиц (9 уже в базе), из них **11 — объекты реального мира** (требование лабы: ≥ 10 таблиц, ≥ 7 реальных объектов).

## Схема

Картинка для презентации: [erd.png](erd.png).

```mermaid
erDiagram
    users ||--o{ attempts : "отвечает"
    tasks ||--o{ attempts : "получает ответы"
    exam_numbers ||--o{ tasks : "содержит"

    users ||--o{ groups : "ведёт (репетитор)"
    groups ||--o{ group_members : ""
    users ||--o{ group_members : "состоит"
    groups ||--o{ assignments : "получает"
    assignments ||--o{ assignment_tasks : ""
    tasks ||--o{ assignment_tasks : ""
    assignments |o--o{ attempts : "решено в рамках"

    exam_variants ||--o{ variant_tasks : ""
    tasks ||--o{ variant_tasks : ""
    exam_numbers ||--o{ variant_tasks : "позиция"
    users ||--o{ exam_sessions : "проходит"
    exam_variants ||--o{ exam_sessions : ""
    exam_sessions ||--o{ exam_answers : ""
    tasks ||--o{ exam_answers : ""

    achievements ||--o{ user_achievements : ""
    users ||--o{ user_achievements : "получает"
    users ||--o{ notifications : "получает"

    users {
        int id PK
        bigint telegram_id UK "не меняется, даже если сменить @username"
        string first_name
        string last_name "nullable"
        string username "nullable"
        string role "план: student | tutor | admin"
        timestamptz created_at
        timestamptz last_login_at
    }
    exam_numbers {
        int id PK
        string code UK "1 … 19-21 … 27"
        string title "тема номера"
        int position UK "порядок карточек"
    }
    tasks {
        int id PK
        int exam_number_id FK
        text statement
        string image_url "nullable"
        string correct_answer
        text explanation
        string source "demo | official | author"
        string source_label "nullable"
        int difficulty "0..100"
        bool is_archived "вместо удаления"
        timestamptz created_at
    }
    attempts {
        int id PK
        int user_id FK "ON DELETE CASCADE"
        int task_id FK
        int assignment_id FK "план, nullable"
        string answer
        bool is_correct
        timestamptz created_at "индекс"
    }
    groups {
        int id PK
        int tutor_id FK
        string title
        string invite_code UK
    }
    group_members {
        int group_id PK, FK
        int user_id PK, FK
        timestamptz joined_at
    }
    assignments {
        int id PK
        int group_id FK
        string title
        timestamptz deadline
    }
    assignment_tasks {
        int assignment_id PK, FK
        int task_id PK, FK
    }
    exam_variants {
        int id PK
        int author_id FK
        string title
        int duration_minutes
    }
    variant_tasks {
        int variant_id PK, FK
        int exam_number_id PK, FK
        int task_id FK
    }
    exam_sessions {
        int id PK
        int user_id FK "ON DELETE CASCADE"
        int variant_id FK "план, nullable: вариант репетитора"
        timestamptz started_at
        timestamptz deadline_at "источник правды для таймера"
        timestamptz finished_at "nullable"
        int max_score
        int primary_score "nullable до завершения"
    }
    exam_answers {
        int id PK
        int session_id FK "UNIQUE (session_id, position)"
        int position "порядок в варианте"
        int task_id FK
        int points "1 или 2 (номера 26, 27)"
        string answer "nullable"
        bool is_correct "считается при завершении"
        timestamptz updated_at
    }
    achievements {
        string code PK "first_solve, streak_7…"
        string title
        string description
        string icon
        int position
    }
    user_achievements {
        int user_id PK, FK
        string achievement_code PK, FK
        timestamptz received_at
    }
    notifications {
        int id PK
        int user_id FK
        string event_id UK "одно событие — одно уведомление"
        string kind "achievement.granted | exam.finished"
        text text
        timestamptz created_at
        timestamptz sent_at "nullable"
        string error "nullable: почему не ушло"
    }
```

## Решения, о которых могут спросить

- **Почему `telegram_id`, а не email/пароль.** Пароли не храним — нечему утечь. Telegram подтверждает личность сам.
- **Почему задачи не удаляются, а архивируются (`is_archived`).** На них ссылаются `attempts`: удаление сломало бы историю и статистику ученика.
- **Почему «решено» не хранится отдельной колонкой.** Это вычисляется из `attempts` (есть ли верная попытка). Нет дублирования — нет рассинхронизации.
- **Индекс `(user_id, task_id)` на `attempts`.** Все запросы статистики — «попытки этого ученика»; индекс делает их быстрыми при росте таблицы.
- **`ON DELETE CASCADE` для `attempts.user_id`.** Удаление аккаунта удаляет его попытки (персональные данные).
- **Что лежит в Redis, а не в PostgreSQL.** Временное и «быстрое»: ключи входа по QR и коды (TTL 5 минут), счётчик попыток ввода кода, указатель на идущий пробник (TTL = время до конца), блокировка от двойного старта, рейтинг (sorted set). Всё, что в Redis, можно восстановить из базы: срок пробника — `exam_sessions.deadline_at`, рейтинг — пересчётом из `attempts`.
- **Почему вариант пробника хранится как строки `exam_answers`, без отдельной таблицы вариантов.** Вариант собирается случайно для каждого ученика; позиция + задача + ответ — одна строка. Отдельная таблица `exam_variants` понадобится, когда репетитор начнёт собирать варианты вручную.

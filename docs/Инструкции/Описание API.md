# Billiard Diary Backend API

Документ описывает актуальное состояние backend API для фронтенда.

Источник истины по контракту: `openapi.yml`. Этот файл дает человекочитаемое описание: роли, сценарии, ограничения и примеры.

## 1. Назначение backend

Backend реализован на Node.js + Express + PostgreSQL.

Основные группы API:

- `GET /health` - проверка доступности сервиса.
- `/auth/*` - регистрация, логин, текущий пользователь.
- `/users` - получение пользователей с фильтрами.
- `/folders/*` - папки библиотеки упражнений.
- `/groups/*` - группы спортсменов тренера.
- `/exercises/*` - CRUD упражнений.
- `/trainings/*` - индивидуальные и групповые тренировки, выполнение упражнений.

Основные таблицы:

- `users`
- `folders`
- `groups`
- `athlete_groups`
- `exercise`
- `group_training`
- `athlete_training`
- `training_item`

Важно: `folders` используются для организации библиотеки упражнений. HTTP API для папок реализован в `/folders`.

## 2. Запуск

### 2.1. Docker Compose

```bash
docker compose up --build
```

API:

```text
http://localhost:3000
```

PostgreSQL с хоста:

```text
localhost:5433
```

PostgreSQL внутри Docker-сети:

```text
db:5432
```

### 2.2. Локальный запуск backend без Docker

```bash
npm install
npm run dev
```

По умолчанию backend использует:

```text
PORT=3000
DB_HOST=127.0.0.1
DB_PORT=5433
DB_USER=postgres
DB_PASSWORD=postgres
DB_NAME=billiard_diary
JWT_SECRET=development-secret-change-me
JWT_EXPIRES_IN=1h
CORS_ORIGIN=*
```

### 2.3. Инициализация БД

Схема создается из:

```text
src/init.sql
```

Docker PostgreSQL выполняет `init.sql` только при первом создании volume. Если схема изменилась, а volume уже существует, нужно пересоздать volume:

```bash
docker compose down -v
docker compose up --build
```

`down -v` удаляет локальные данные PostgreSQL.

Для dev-проверки без Docker можно применить `init.sql` скриптом:

```bash
node scripts/apply-init.js
```

### 2.4. Seed-данные

При инициализации создаются пользователи:

| Роль      | Email                  | Пароль |
| --------- | ---------------------- | ------ |
| `admin`   | `admin@example.com`    | `123`  |
| `coach`   | `coach@example.com`    | `123`  |
| `athlete` | `athlete@example.com`  | `123`  |
| `athlete` | `athlete2@example.com` | `123`  |
| `athlete` | `athlete3@example.com` | `123`  |

Также создается seed-группа:

| Группа     | Тренер              | Участники                                                             |
| ---------- | ------------------- | --------------------------------------------------------------------- |
| `Junior A` | `coach@example.com` | `athlete@example.com`, `athlete2@example.com`, `athlete3@example.com` |

## 3. Общие правила API

### 3.1. Base URL

```text
http://localhost:3000
```

### 3.2. Формат запросов

Тело запросов передается в JSON:

```http
Content-Type: application/json
```

### 3.3. Авторизация

Авторизация через JWT access token:

```http
Authorization: Bearer <accessToken>
```

Токен возвращается в:

- `POST /auth/register`
- `POST /auth/login`

Payload токена содержит:

- `sub` - id пользователя строкой.
- `role` - роль пользователя.

### 3.4. Успешный ответ

Все успешные ответы содержат:

```json
{
  "ok": true
}
```

Дальше добавляется полезная нагрузка: `user`, `users`, `exercise`, `group`, `training`, `athleteTraining`, `groupTraining`, `item` и т.д.

### 3.5. Ошибка

```json
{
  "ok": false,
  "code": "validation_error",
  "message": "Request validation failed",
  "details": []
}
```

Основные коды:

|  HTTP | `code`                             | Когда возникает                                           |
| ----: | ---------------------------------- | --------------------------------------------------------- |
| `400` | `validation_error`                 | Невалидное тело, query или path params                    |
| `400` | `invalid_user_filter`              | `groupId` использован вместе с ролью не `athlete`         |
| `400` | `invalid_training_target`          | Переданы оба поля `groupId` и `athleteId` или ни одного   |
| `400` | `invalid_training_period`          | `startsAt` не раньше `endsAt`                             |
| `400` | `invalid_training_exercises`       | Упражнение не найдено, удалено или не принадлежит тренеру |
| `400` | `training_is_not_editable`         | Попытка редактировать тренировку не в `scheduled`         |
| `400` | `invalid_status_transition`        | Недопустимый переход статуса                              |
| `400` | `training_is_not_in_progress`      | Выполнение item не в активной тренировке                  |
| `400` | `training_item_is_not_in_progress` | Результат пишется не в активный item                      |
| `400` | `invalid_training_item_result`     | `resultSuccesses > resultAttempts`                        |
| `401` | `auth_required`                    | Нет токена                                                |
| `401` | `invalid_auth_header`              | Неверный формат `Authorization`                           |
| `401` | `invalid_token`                    | Невалидный JWT                                            |
| `401` | `invalid_credentials`              | Неверный email или пароль                                 |
| `403` | `forbidden`                        | Нет прав                                                  |
| `403` | `direct_exercise_access_forbidden` | Athlete напрямую обращается к exercises                   |
| `403` | `forbidden_role_registration`      | Попытка зарегистрировать запрещенную роль                 |
| `404` | `not_found`                        | Сущность не найдена или недоступна                        |
| `404` | `route_not_found`                  | Маршрут не существует                                     |
| `409` | `unique_violation`                 | Нарушение уникальности, например email уже занят          |
| `500` | `internal_error`                   | Внутренняя ошибка сервера                                 |

## 4. Роли

Есть три роли:

```text
admin
coach
athlete
```

### 4.1. Общая модель прав

`admin`:

- регистрирует тренеров;
- может работать с exercises;
- не работает с trainings.

`coach`:

- регистрирует спортсменов;
- создает и редактирует свои упражнения;
- создает группы;
- добавляет в группы только своих спортсменов;
- создает индивидуальные и групповые тренировки;
- управляет своими тренировками.

`athlete`:

- не создает пользователей;
- не работает с exercises напрямую;
- видит только назначенные ему `athlete_training`;
- может начать и завершить свою тренировку;
- может выполнять items своей тренировки.

## 5. Auth API

### 5.1. POST /auth/register

Регистрирует пользователя и возвращает access token.

Правила:

- если активных пользователей нет, первый пользователь может быть создан без токена, но только с ролью `admin`;
- `admin` может зарегистрировать только `coach`;
- `coach` может зарегистрировать только `athlete`;
- `athlete` не может регистрировать пользователей.

Request:

```json
{
  "email": "coach@example.com",
  "password": "123",
  "name": "Coach Name",
  "role": "coach"
}
```

Response `201`:

```json
{
  "ok": true,
  "accessToken": "jwt-token",
  "user": {
    "id": "2",
    "email": "coach@example.com",
    "name": "Coach Name",
    "role": "coach",
    "coachId": null,
    "createdAt": "2026-05-13T00:00:00.000Z",
    "updatedAt": "2026-05-13T00:00:00.000Z",
    "deletedAt": null
  }
}
```

### 5.2. POST /auth/login

Request:

```json
{
  "email": "coach@example.com",
  "password": "123"
}
```

Response `200`:

```json
{
  "ok": true,
  "accessToken": "jwt-token",
  "user": {
    "id": "2",
    "email": "coach@example.com",
    "name": "Coach User",
    "role": "coach",
    "coachId": null,
    "createdAt": "2026-05-13T00:00:00.000Z",
    "updatedAt": "2026-05-13T00:00:00.000Z",
    "deletedAt": null
  }
}
```

### 5.3. GET /auth/me

Возвращает текущего пользователя.

Авторизация: обязательна.

Response:

```json
{
  "ok": true,
  "user": {
    "id": "2",
    "email": "coach@example.com",
    "name": "Coach User",
    "role": "coach",
    "coachId": null,
    "createdAt": "2026-05-13T00:00:00.000Z",
    "updatedAt": "2026-05-13T00:00:00.000Z",
    "deletedAt": null
  }
}
```

## 6. Users API

### 6.1. GET /users

Возвращает список пользователей.

Авторизация: обязательна.

Query params:

| Параметр  | Тип     | Обязательный | Описание                     |
| --------- | ------- | -----------: | ---------------------------- |
| `role`    | string  |          Нет | `admin`, `coach`, `athlete`  |
| `groupId` | integer |          Нет | Фильтр спортсменов по группе |

Правила:

- `admin` может получать пользователей.
- `coach` получает только своих спортсменов; при `role=coach` может получить себя.
- `athlete` получает `403`.
- `groupId` можно использовать только с `role=athlete` или без `role`.
- `role=coach&groupId=1` и `role=admin&groupId=1` вернут `400 invalid_user_filter`.

Пример:

```http
GET /users?role=athlete&groupId=1
```

Response:

```json
{
  "ok": true,
  "users": [
    {
      "id": "3",
      "email": "athlete@example.com",
      "name": "Athlete User",
      "role": "athlete",
      "coachId": "2",
      "createdAt": "2026-05-13T00:00:00.000Z",
      "updatedAt": "2026-05-13T00:00:00.000Z",
      "deletedAt": null
    }
  ]
}
```

### 6.2. GET /users/:id

Возвращает одного пользователя.

Доступ:

- `admin` может читать любого активного пользователя;
- `coach` может читать себя и своих спортсменов;
- `athlete` может читать только себя.

Response:

```json
{
  "ok": true,
  "user": {
    "id": "3",
    "email": "athlete@example.com",
    "name": "Athlete User",
    "role": "athlete",
    "coachId": "2",
    "createdAt": "2026-05-13T00:00:00.000Z",
    "updatedAt": "2026-05-13T00:00:00.000Z",
    "deletedAt": null
  }
}
```

### 6.3. PATCH /users/:id

Обновляет данные пользователя.

Доступ:

- `admin` может обновлять пользователей;
- `coach` может обновлять только своих спортсменов;
- `athlete` не может обновлять пользователей через этот endpoint.

Нельзя менять `role` и `coachId`.

Request:

```json
{
  "email": "updated.athlete@example.com",
  "name": "Updated Athlete",
  "password": "123"
}
```

### 6.4. DELETE /users/:id

Soft-delete пользователя.

Доступ:

- `admin` может удалить пользователя, кроме самого себя;
- `coach` может удалить только своего спортсмена;
- `athlete` не может удалять пользователей.

Если удаляется спортсмен, его активные связи с группами тоже soft-deleted. История тренировок при этом не удаляется.

## 7. Groups API

Все методы `/groups` требуют Bearer token. Работать с группами может только `coach`.

### 7.1. Модель Group

```json
{
  "id": "1",
  "name": "Junior A",
  "coachId": "2",
  "description": "Seed training group",
  "athletesCount": 3,
  "athletes": [],
  "createdAt": "2026-05-13T00:00:00.000Z",
  "updatedAt": "2026-05-13T00:00:00.000Z",
  "deletedAt": null
}
```

В списке групп может быть `athletesCount`, а детальная карточка содержит `athletes`.

### 7.2. GET /groups

Возвращает группы текущего тренера.

Response:

```json
{
  "ok": true,
  "groups": []
}
```

### 7.3. GET /groups/:id

Возвращает группу с участниками.

### 7.4. POST /groups

Создает группу.

Request:

```json
{
  "name": "Junior A",
  "description": "Morning junior group",
  "athleteIds": [3, 4]
}
```

Правила:

- `name` обязателен;
- `athleteIds` опционален;
- все athlete из `athleteIds` должны принадлежать текущему coach.

### 7.5. PATCH /groups/:id

Частично обновляет группу.

Request:

```json
{
  "name": "Junior A Updated",
  "description": "Updated description"
}
```

### 7.6. DELETE /groups/:id

Soft-delete группы.

### 7.7. POST /groups/:id/athletes

Добавляет спортсмена в группу.

Request:

```json
{
  "athleteId": 3
}
```

### 7.8. DELETE /groups/:id/athletes/:athleteId

Soft-delete связи спортсмена с группой.

### 7.9. Folders API

Все методы `/folders` требуют Bearer token.

Права:

- `admin` может видеть и управлять всеми папками;
- `coach` видит и управляет только своими папками;
- `athlete` не имеет прямого доступа к папкам и получает `403 direct_folder_access_forbidden`.

Endpoints:

```text
GET    /folders
GET    /folders/:id
POST   /folders
PATCH  /folders/:id
DELETE /folders/:id
```

Модель Folder:

```json
{
  "id": "1",
  "title": "Long pots",
  "authorId": "2",
  "createdAt": "2026-05-13T00:00:00.000Z",
  "updatedAt": "2026-05-13T00:00:00.000Z",
  "deletedAt": null
}
```

Создание папки:

```json
{
  "title": "Long pots"
}
```

Удаление папки использует soft-delete. Упражнения при этом не удаляются автоматически.

## 8. Exercises API

Все методы `/exercises` требуют Bearer token.

### 8.1. Права

| Роль      | Список      | Просмотр    | Создание | Обновление  | Удаление    |
| --------- | ----------- | ----------- | -------- | ----------- | ----------- |
| `admin`   | Все         | Любое       | Да       | Любое       | Любое       |
| `coach`   | Только свои | Только свое | Да       | Только свое | Только свое |
| `athlete` | Нет         | Нет         | Нет      | Нет         | Нет         |

`athlete` получает `403 direct_exercise_access_forbidden`.

### 8.2. Модель Exercise

Во всех exercises endpoints ответ приходит в camelCase:

```json
{
  "id": "1",
  "title": "Long pot practice",
  "description": "Practice long pot with cue ball control.",
  "position": {
    "power": 2.5
  },
  "authorId": "2",
  "folderId": null,
  "createdAt": "2026-05-13T00:00:00.000Z",
  "updatedAt": "2026-05-13T00:00:00.000Z",
  "deletedAt": null
}
```

Внутри training item вложенный `exercise` использует тот же формат.

### 8.3. GET /exercises

Query params:

| Параметр   | Тип     | По умолчанию | Описание                               |
| ---------- | ------- | -----------: | -------------------------------------- |
| `search`   | string  |            - | Поиск по `title`                       |
| `folderId` | integer |            - | Фильтр по папке                        |
| `authorId` | integer |            - | Для admin любой, для coach только свой |
| `limit`    | integer |         `20` | 1-100                                  |
| `offset`   | integer |          `0` | >= 0                                   |

Response:

```json
{
  "ok": true,
  "exercises": [],
  "pagination": {
    "limit": 20,
    "offset": 0,
    "total": 0
  }
}
```

### 8.4. GET /exercises/:id

Возвращает одно упражнение.

### 8.5. POST /exercises

Request:

```json
{
  "title": "Long pot practice",
  "description": "Practice long pot with cue ball control.",
  "folderId": null,
  "position": {
    "power": 2.5
  }
}
```

Правила:

- `title`: обязательно, 3-255 символов;
- `description`: опционально, строка или `null`, до 3000 символов;
- `position`: объект или `null`;
- `folderId`: integer или `null`; если передан, папка должна существовать и быть доступна текущему пользователю;
- `authorId` всегда берется из токена.

### 8.6. PUT /exercises/:id

Частично обновляет упражнение, хотя используется `PUT`.

Request:

```json
{
  "title": "Updated long pot practice",
  "description": null,
  "position": {
    "power": 3
  },
  "folderId": null
}
```

### 8.7. DELETE /exercises/:id

Soft-delete упражнения. После удаления оно не возвращается через прямые exercises endpoints.

## 9. Trainings API

Все методы `/trainings` требуют Bearer token.

### 9.1. Термины

`group_training` - групповая тренировка, назначенная на группу.

`athlete_training` - конкретная тренировка конкретного спортсмена.

`training_item` - упражнение внутри `athlete_training`.

Индивидуальная тренировка:

- создается только `athlete_training`;
- `groupTrainingId = null`.

Групповая тренировка:

- создается один `group_training`;
- для каждого спортсмена группы создается отдельная `athlete_training`;
- в каждую `athlete_training` копируются `training_item`.

### 9.2. Статусы training

Текущие значения хранятся как `varchar`.

```text
scheduled
in_progress
completed
cancelled
missed
```

Разрешенные переходы:

```text
scheduled   -> in_progress
scheduled   -> cancelled
scheduled   -> missed
in_progress -> completed
```

`coach` может выполнять все эти переходы.

`athlete` может менять только свою `athlete_training`:

```text
scheduled   -> in_progress
in_progress -> completed
```

### 9.3. Статусы training_item

```text
not_started
in_progress
completed
skipped
```

Разрешенные переходы:

```text
not_started -> in_progress
not_started -> skipped
in_progress -> completed
in_progress -> skipped
```

### 9.4. Правила редактирования

- Даты тренировки и состав items можно менять только пока training в статусе `scheduled`.
- Item result можно менять только когда `athlete_training.status = in_progress` и `training_item.status = in_progress`.
- `resultSuccesses` не может быть больше `resultAttempts`.
- `admin` не работает с trainings.
- `athlete` не может читать `group_training` напрямую.

### 9.5. POST /trainings

Создает индивидуальную или групповую тренировку.

Доступ: только `coach`.

Нужно передать ровно одно поле:

- `athleteId` - индивидуальная тренировка;
- `groupId` - групповая тренировка.

Create individual:

```json
{
  "athleteId": 3,
  "startsAt": "2026-05-20T10:00:00+07:00",
  "endsAt": "2026-05-20T11:30:00+07:00",
  "items": [
    {
      "exerciseId": 1,
      "orderIndex": 0,
      "targetAttempts": 10
    }
  ]
}
```

Create group:

```json
{
  "groupId": 1,
  "startsAt": "2026-05-21T10:00:00+07:00",
  "endsAt": "2026-05-21T11:30:00+07:00",
  "items": [
    {
      "exerciseId": 1,
      "orderIndex": 0,
      "targetAttempts": 10
    }
  ]
}
```

Правила:

- `startsAt` и `endsAt` обязательны;
- `startsAt` должен быть раньше `endsAt`;
- `items` может быть пустым массивом;
- все упражнения должны быть активными и принадлежать текущему coach;
- group должна принадлежать текущему coach;
- athlete должен принадлежать текущему coach;
- group не может быть пустой.

Response для индивидуальной:

```json
{
  "ok": true,
  "training": {
    "type": "athlete",
    "athleteTraining": {
      "id": "1",
      "groupTrainingId": null,
      "coachId": "2",
      "athleteId": "3",
      "startsAt": "2026-05-20T03:00:00.000Z",
      "endsAt": "2026-05-20T04:30:00.000Z",
      "status": "scheduled",
      "items": []
    }
  }
}
```

Response для групповой:

```json
{
  "ok": true,
  "training": {
    "type": "group",
    "groupTraining": {
      "id": "1",
      "coachId": "2",
      "groupId": "1",
      "startsAt": "2026-05-21T03:00:00.000Z",
      "endsAt": "2026-05-21T04:30:00.000Z",
      "status": "scheduled",
      "athleteTrainings": []
    }
  }
}
```

### 9.6. GET /trainings

Для `coach` возвращает:

- свои `group_training`;
- свои индивидуальные `athlete_training`, где `groupTrainingId = null`.

Для `athlete` возвращает:

- все назначенные ему `athlete_training`.

Response:

```json
{
  "ok": true,
  "trainings": [
    {
      "type": "group",
      "groupTraining": {}
    },
    {
      "type": "athlete",
      "athleteTraining": {}
    }
  ]
}
```

### 9.7. GET /trainings/group/:id

Возвращает group training с дочерними athlete trainings и items.

Доступ: только coach-владелец.

### 9.8. PATCH /trainings/group/:id

Обновляет даты group training и синхронизирует их в дочерние athlete trainings.

Доступ: только coach-владелец.

Можно только в статусе `scheduled`.

Request:

```json
{
  "startsAt": "2026-05-21T10:30:00+07:00",
  "endsAt": "2026-05-21T12:00:00+07:00"
}
```

### 9.9. DELETE /trainings/group/:id

Soft-delete group training, дочерних athlete trainings и их items.

Доступ: только coach-владелец.

### 9.10. PATCH /trainings/group/:id/status

Меняет статус group training и синхронизирует статус в дочерние athlete trainings.

Доступ: только coach-владелец.

Request:

```json
{
  "status": "in_progress"
}
```

### 9.11. GET /trainings/athlete/:id

Возвращает athlete training с items.

Доступ:

- coach-владелец;
- athlete, если эта тренировка назначена ему.

### 9.12. PATCH /trainings/athlete/:id

Обновляет athlete training.

Доступ: только coach-владелец.

Можно только в статусе `scheduled`.

Request:

```json
{
  "athleteId": 3,
  "startsAt": "2026-05-20T10:15:00+07:00",
  "endsAt": "2026-05-20T11:45:00+07:00"
}
```

Особенность:

- `athleteId` можно менять только у индивидуальной тренировки;
- у athlete training, созданной из group training, спортсмен закреплен и не меняется.

### 9.13. DELETE /trainings/athlete/:id

Soft-delete athlete training и ее items.

Доступ: только coach-владелец.

### 9.14. PATCH /trainings/athlete/:id/status

Меняет статус athlete training.

Доступ:

- coach-владелец;
- athlete, если тренировка назначена ему.

Request:

```json
{
  "status": "in_progress"
}
```

### 9.15. POST /trainings/athlete/:id/items

Добавляет item в athlete training.

Доступ: только coach-владелец.

Можно только пока athlete training в статусе `scheduled`.

Request:

```json
{
  "exerciseId": 1,
  "orderIndex": 0,
  "targetAttempts": 10
}
```

### 9.16. PATCH /trainings/athlete/:id/items/:itemId

Обновляет item.

Доступ: только coach-владелец.

Можно только пока athlete training в статусе `scheduled`.

Request:

```json
{
  "orderIndex": 1,
  "targetAttempts": 12
}
```

### 9.17. DELETE /trainings/athlete/:id/items/:itemId

Soft-delete item.

Доступ: только coach-владелец.

Можно только пока athlete training в статусе `scheduled`.

### 9.18. PATCH /trainings/athlete/:id/items/:itemId/status

Меняет статус item.

Доступ:

- coach-владелец;
- athlete, если training назначена ему.

Можно только когда `athlete_training.status = in_progress`.

Request:

```json
{
  "status": "in_progress"
}
```

### 9.19. PATCH /trainings/athlete/:id/items/:itemId/result

Обновляет результат item.

Доступ:

- coach-владелец;
- athlete, если training назначена ему.

Можно только когда:

- `athlete_training.status = in_progress`;
- `training_item.status = in_progress`.

Request:

```json
{
  "resultAttempts": 10,
  "resultSuccesses": 6
}
```

Правило:

- `resultSuccesses <= resultAttempts`.

## 10. Модели trainings

### 10.1. GroupTraining

```json
{
  "id": "1",
  "coachId": "2",
  "groupId": "1",
  "startsAt": "2026-05-21T03:00:00.000Z",
  "endsAt": "2026-05-21T04:30:00.000Z",
  "status": "scheduled",
  "athleteTrainingsCount": 3,
  "athleteTrainings": [],
  "createdAt": "2026-05-13T00:00:00.000Z",
  "updatedAt": "2026-05-13T00:00:00.000Z",
  "deletedAt": null
}
```

### 10.2. AthleteTraining

```json
{
  "id": "1",
  "groupTrainingId": null,
  "coachId": "2",
  "athleteId": "3",
  "startsAt": "2026-05-20T03:00:00.000Z",
  "endsAt": "2026-05-20T04:30:00.000Z",
  "status": "scheduled",
  "itemsCount": 2,
  "items": [],
  "createdAt": "2026-05-13T00:00:00.000Z",
  "updatedAt": "2026-05-13T00:00:00.000Z",
  "deletedAt": null
}
```

### 10.3. TrainingItem

```json
{
  "id": "1",
  "athleteTrainingId": "1",
  "exerciseId": "1",
  "orderIndex": 0,
  "targetAttempts": 10,
  "resultAttempts": 10,
  "resultSuccesses": 6,
  "status": "in_progress",
  "exercise": {
    "id": 1,
    "title": "Long pot practice",
    "description": null,
    "position": null,
    "authorId": 2,
    "folderId": null,
    "createdAt": "2026-05-13T00:00:00.000Z",
    "updatedAt": "2026-05-13T00:00:00.000Z",
    "deletedAt": null
  },
  "createdAt": "2026-05-13T00:00:00.000Z",
  "updatedAt": "2026-05-13T00:00:00.000Z",
  "deletedAt": null
}
```

## 11. Frontend flow

### 11.1. Login

```js
async function login(email, password) {
  const response = await fetch("http://localhost:3000/auth/login", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ email, password }),
  });

  const data = await response.json();
  if (!response.ok) throw data;
  return data;
}
```

### 11.2. Authorized helper

```js
async function apiFetch(path, { token, method = "GET", body } = {}) {
  const response = await fetch(`http://localhost:3000${path}`, {
    method,
    headers: {
      ...(body ? { "Content-Type": "application/json" } : {}),
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
    },
    body: body ? JSON.stringify(body) : undefined,
  });

  const data = await response.json();
  if (!response.ok) throw data;
  return data;
}
```

### 11.3. Coach: создать групповую тренировку

```js
const athletes = await apiFetch("/users?role=athlete&groupId=1", {
  token,
});

const training = await apiFetch("/trainings", {
  token,
  method: "POST",
  body: {
    groupId: 1,
    startsAt: "2026-05-21T10:00:00+07:00",
    endsAt: "2026-05-21T11:30:00+07:00",
    items: [
      {
        exerciseId: 1,
        orderIndex: 0,
        targetAttempts: 10,
      },
    ],
  },
});
```

### 11.4. Athlete: выполнить item

```js
await apiFetch("/trainings/athlete/1/status", {
  token,
  method: "PATCH",
  body: {
    status: "in_progress",
  },
});

await apiFetch("/trainings/athlete/1/items/1/status", {
  token,
  method: "PATCH",
  body: {
    status: "in_progress",
  },
});

await apiFetch("/trainings/athlete/1/items/1/result", {
  token,
  method: "PATCH",
  body: {
    resultAttempts: 10,
    resultSuccesses: 6,
  },
});
```

## 12. Видимая проверка API

Скрипт:

```text
scripts/visible-api-flow.ps1
```

Запуск:

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\visible-api-flow.ps1
```

Запуск против другого backend URL:

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\visible-api-flow.ps1 -BaseUrl http://localhost:3001
```

Без паузы в конце:

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\visible-api-flow.ps1 -NoPause
```

Скрипт проверяет:

- healthcheck;
- login seed admin/coach;
- регистрацию athlete;
- создание group;
- фильтр athletes по group;
- создание exercises;
- создание individual training;
- создание group training;
- просмотр trainings тренером и спортсменом;
- редактирование scheduled training;
- добавление и редактирование item;
- выполнение item спортсменом;
- переходы статусов;
- ожидаемые ошибки доступа;
- ожидаемые ошибки state machine.

## 13. Что пока не реализовано

Пока нет HTTP API для:

- refresh token;
- logout/session invalidation;
- комментариев;
- посещений;
- аналитики и статистики;
- шаблонов тренировок;
- Swagger UI endpoint;
- миграций БД.

## 14. Важные особенности для фронтенда

- Users, folders, groups, trainings и exercises endpoints возвращают ответы в camelCase.
- `GET /exercises` и `GET /trainings` возвращают `pagination.limit`, `pagination.offset`, `pagination.total`.
- Query params приводятся к числам через AJV `coerceTypes`.
- Лишние поля запрещены через `additionalProperties: false`.
- `bigserial` из PostgreSQL приходит строкой; id в ответах описаны как строки в `openapi.yml`.
- Время хранится в `timestamptz`; PostgreSQL может вернуть его в UTC-форме.
- `openapi.yml` лежит в корне проекта, но Express пока не раздает Swagger UI.

## 15. Актуализация gaps для frontend

Этот раздел уточняет изменения после доработки coach-прототипа withGroups.

### 15.1. CRUD пользователей/спортсменов

Теперь доступны:

```http
GET    /users/:id
PATCH  /users/:id
DELETE /users/:id
```

`coach` может читать, редактировать и soft-delete только своих спортсменов. `admin` может управлять пользователями, но не может удалить самого себя. `athlete` может читать только себя и не может редактировать/удалять пользователей через эти методы.

`PATCH /users/:id` принимает:

```json
{
  "email": "updated.athlete@example.com",
  "name": "Updated Athlete",
  "password": "123"
}
```

`role` и `coachId` этим методом не меняются.

### 15.2. POST /trainings с разными items по спортсменам

Для индивидуальной тренировки по-прежнему используется `athleteId + items`.

Для групповой тренировки теперь есть два варианта:

1. Передать общий `items`. Тогда он копируется всем спортсменам группы.
2. Передать `athleteTrainings`. Тогда у разных спортсменов могут быть разные упражнения.

Пример групповой тренировки с разными упражнениями:

```json
{
  "groupId": 1,
  "startsAt": "2026-05-21T10:00:00+07:00",
  "endsAt": "2026-05-21T11:30:00+07:00",
  "athleteTrainings": [
    {
      "athleteId": 3,
      "items": [
        {
          "exerciseId": 1,
          "orderIndex": 0,
          "targetAttempts": 10
        }
      ]
    },
    {
      "athleteId": 4,
      "items": [
        {
          "exerciseId": 2,
          "orderIndex": 0,
          "targetAttempts": 8
        }
      ]
    }
  ]
}
```

Нельзя одновременно передавать непустой `items` и `athleteTrainings`. Все `athleteId` внутри `athleteTrainings` должны быть участниками выбранной группы.

### 15.3. Вложенный athlete в training responses

В `athleteTraining` теперь возвращается вложенный объект `athlete`:

```json
{
  "id": "10",
  "athleteId": "3",
  "athlete": {
    "id": "3",
    "email": "athlete@example.com",
    "name": "Athlete User",
    "role": "athlete",
    "coachId": "2"
  }
}
```

Это касается списка тренировок, деталей индивидуальной тренировки и дочерних `athleteTrainings` внутри групповой тренировки.

### 15.4. Фильтры и пагинация GET /trainings

`GET /trainings` поддерживает:

```text
date
dateFrom
dateTo
status
groupId
athleteId
limit
offset
```

Пример:

```http
GET /trainings?date=2026-05-20&status=scheduled&groupId=1&limit=20&offset=0
```

Response теперь содержит `pagination`:

```json
{
  "ok": true,
  "trainings": [],
  "pagination": {
    "limit": 20,
    "offset": 0,
    "total": 0
  }
}
```

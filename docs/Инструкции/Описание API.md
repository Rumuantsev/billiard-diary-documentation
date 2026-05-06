# Billiard Diary Backend API: текущее состояние для фронтенда

Документ описывает фактическое состояние backend API по коду и `openapi.yml`.

## 1. Назначение backend

Backend реализован на Node.js + Express и PostgreSQL.

Текущие публичные группы API:

- `GET /health` - проверка доступности сервиса.
- `/auth/*` - регистрация, логин, получение текущего пользователя.
- `/exercises/*` - CRUD упражнений.

В базе уже есть таблицы:

- `users`
- `folders`
- `exercise`

Важно: таблица `folders` есть в схеме БД, но HTTP API для папок сейчас не реализован.

## 2. Развертывание

### 2.1. Требования

- Node.js 18+
- Docker
- Docker Compose
- PostgreSQL, если запускать backend без Docker Compose

### 2.2. Быстрый запуск через Docker Compose

Из корня backend-проекта:

```bash
docker compose up --build
```

После запуска API доступно по адресу:

```text
http://localhost:3000
```

PostgreSQL проброшен на хост:

```text
localhost:5433
```

Внутри Docker-сети backend ходит в БД по адресу:

```text
db:5432
```

### 2.3. Проверка запуска

```http
GET http://localhost:3000/health
```

Ожидаемый ответ:

```json
{
  "ok": true,
  "status": "healthy"
}
```

### 2.4. Локальный запуск без Docker для backend

Если PostgreSQL уже запущен локально:

```bash
npm install
npm run dev
```

или:

```bash
npm start
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

### 2.5. Переменные окружения

| Переменная       |          Значение по умолчанию | Описание                              |
| ---------------- | -----------------------------: | ------------------------------------- |
| `PORT`           |                         `3000` | Порт HTTP API                         |
| `CORS_ORIGIN`    |                            `*` | Разрешенный origin для CORS           |
| `DB_HOST`        |                    `127.0.0.1` | Хост PostgreSQL                       |
| `DB_PORT`        |                         `5433` | Порт PostgreSQL при локальном запуске |
| `DB_USER`        |                     `postgres` | Пользователь БД                       |
| `DB_PASSWORD`    |                     `postgres` | Пароль БД                             |
| `DB_NAME`        |               `billiard_diary` | Имя БД                                |
| `JWT_SECRET`     | `development-secret-change-me` | Секрет для JWT                        |
| `JWT_EXPIRES_IN` |                           `1h` | Время жизни access token              |

### 2.6. Инициализация БД

Схема создается из файла:

```text
src/init.sql
```

Docker PostgreSQL выполняет этот файл только при первом создании volume. Если схема изменилась, а volume уже существует, нужно пересоздать volume:

```bash
docker compose down -v
docker compose up --build
```

Команда `down -v` удаляет локальные данные БД.

### 2.7. Seed-пользователи

При первом создании БД создаются тестовые пользователи:

| Роль      | Email                 | Пароль |
| --------- | --------------------- | ------ |
| `admin`   | `admin@example.com`   | `123`  |
| `coach`   | `coach@example.com`   | `123`  |
| `athlete` | `athlete@example.com` | `123`  |

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

Токен возвращается в ответах:

- `POST /auth/register`
- `POST /auth/login`

Payload токена содержит:

- `sub` - id пользователя строкой
- `role` - роль пользователя

### 3.4. Формат успешного ответа

Все успешные ответы содержат:

```json
{
  "ok": true
}
```

Дальше добавляется полезная нагрузка: `user`, `accessToken`, `exercise`, `exercises`, `pagination`.

### 3.5. Формат ошибки

```json
{
  "ok": false,
  "code": "validation_error",
  "message": "Request validation failed",
  "details": []
}
```

Поле `details` может отсутствовать или быть `null`, строкой, объектом или массивом.

Основные коды ошибок:

|  HTTP | `code`                             | Когда возникает                                |
| ----: | ---------------------------------- | ---------------------------------------------- |
| `400` | `validation_error`                 | Невалидное тело, query или path params         |
| `400` | `foreign_key_violation`            | Передан id связанной сущности, которой нет     |
| `400` | `check_violation`                  | Нарушено ограничение БД                        |
| `401` | `auth_required`                    | Нет токена                                     |
| `401` | `invalid_auth_header`              | Некорректный формат `Authorization`            |
| `401` | `invalid_token`                    | Токен невалидный или пользователь не найден    |
| `401` | `invalid_credentials`              | Неверный email или пароль                      |
| `403` | `forbidden`                        | Недостаточно прав                              |
| `403` | `bootstrap_admin_required`         | Первый пользователь должен быть admin          |
| `403` | `forbidden_role_registration`      | Нельзя зарегистрировать указанную роль         |
| `403` | `direct_exercise_access_forbidden` | Athlete не может работать с exercises напрямую |
| `404` | `route_not_found`                  | Маршрут не существует                          |
| `404` | `not_found`                        | Сущность не найдена или недоступна             |
| `409` | `unique_violation`                 | Например, email уже занят                      |
| `500` | `internal_error`                   | Внутренняя ошибка сервера                      |

## 4. Роли и правила доступности

Есть три роли:

```text
admin
coach
athlete
```

### 4.1. Регистрация пользователей

Правила регистрации:

- Если в БД нет активных пользователей, первый пользователь может быть создан без токена, но только с ролью `admin`.
- После bootstrap-режима регистрация требует Bearer token.
- `admin` может регистрировать только `coach`.
- `coach` может регистрировать только `athlete`.
- `athlete` не может регистрировать пользователей.

Связь `coach_id`:

- У `athlete` backend автоматически ставит `coach_id` равным id текущего coach.
- У `admin` и `coach` поле `coach_id` равно `null`.
- Клиент не передает `coach_id` при регистрации.

### 4.2. Доступ к упражнениям

| Роль      | Список exercises | Просмотр одного | Создание | Обновление  | Удаление    |
| --------- | ---------------- | --------------- | -------- | ----------- | ----------- |
| `admin`   | Все неудаленные  | Любое           | Да       | Любое       | Любое       |
| `coach`   | Только свои      | Только свое     | Да       | Только свое | Только свое |
| `athlete` | Нет              | Нет             | Нет      | Нет         | Нет         |

Важно:

- `author_id` упражнения всегда берется из JWT.
- Frontend не должен отправлять `authorId` или `author_id` в body создания/обновления.
- Удаление упражнения - soft delete: заполняется `deleted_at`, запись остается в БД.

## 5. Auth API

### 5.1. POST /auth/register

Регистрирует пользователя и сразу возвращает access token.

Авторизация:

- Без токена только для первого `admin`, когда таблица пользователей пустая.
- С токеном для регистрации по правилам ролей.

Request body:

```json
{
  "email": "coach@example.com",
  "password": "123",
  "name": "Coach Name",
  "role": "coach"
}
```

Поля:

| Поле       | Тип    | Обязательное | Правила                     |
| ---------- | ------ | -----------: | --------------------------- |
| `email`    | string |           Да | email, 3-255 символов       |
| `password` | string |           Да | 3-128 символов              |
| `name`     | string |           Да | 2-255 символов              |
| `role`     | string |           Да | `admin`, `coach`, `athlete` |

Лишние поля запрещены.

Response `201`:

```json
{
  "ok": true,
  "accessToken": "jwt-token",
  "user": {
    "id": 2,
    "email": "coach@example.com",
    "name": "Coach Name",
    "role": "coach",
    "coach_id": null,
    "created_at": "2026-05-06T00:00:00.000Z",
    "updated_at": "2026-05-06T00:00:00.000Z",
    "deleted_at": null
  }
}
```

Пример вызова:

```js
const response = await fetch("http://localhost:3000/auth/register", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    Authorization: `Bearer ${accessToken}`,
  },
  body: JSON.stringify({
    email: "coach@example.com",
    password: "123",
    name: "Coach Name",
    role: "coach",
  }),
});
```

### 5.2. POST /auth/login

Авторизует пользователя по email и паролю.

Request body:

```json
{
  "email": "coach@example.com",
  "password": "123"
}
```

Поля:

| Поле       | Тип    | Обязательное | Правила               |
| ---------- | ------ | -----------: | --------------------- |
| `email`    | string |           Да | email, 3-255 символов |
| `password` | string |           Да | 3-128 символов        |

Response `200`:

```json
{
  "ok": true,
  "accessToken": "jwt-token",
  "user": {
    "id": 2,
    "email": "coach@example.com",
    "name": "Coach User",
    "role": "coach",
    "coach_id": null,
    "created_at": "2026-05-06T00:00:00.000Z",
    "updated_at": "2026-05-06T00:00:00.000Z",
    "deleted_at": null
  }
}
```

Пример вызова:

```js
const response = await fetch("http://localhost:3000/auth/login", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    email: "coach@example.com",
    password: "123",
  }),
});

const data = await response.json();
const accessToken = data.accessToken;
```

### 5.3. GET /auth/me

Возвращает текущего пользователя по Bearer token.

Авторизация: обязательна.

Response `200`:

```json
{
  "ok": true,
  "user": {
    "id": 2,
    "email": "coach@example.com",
    "name": "Coach User",
    "role": "coach",
    "coach_id": null,
    "created_at": "2026-05-06T00:00:00.000Z",
    "updated_at": "2026-05-06T00:00:00.000Z",
    "deleted_at": null
  }
}
```

Пример вызова:

```js
const response = await fetch("http://localhost:3000/auth/me", {
  headers: {
    Authorization: `Bearer ${accessToken}`,
  },
});
```

## 6. Exercises API

Все методы `/exercises` требуют Bearer token.

### 6.1. Модель Exercise

В ответах backend возвращает поля в snake_case, как они лежат в БД:

```json
{
  "id": 1,
  "title": "Long pot practice",
  "description": "Practice long pot with cue ball control.",
  "position": {
    "balls": [
      {
        "id": "cue",
        "x": 120,
        "y": 240,
        "type": "cue"
      }
    ],
    "lines": [
      {
        "id": "shot-line",
        "from": {
          "x": 120,
          "y": 240
        },
        "to": {
          "x": 320,
          "y": 180
        },
        "type": "aim"
      }
    ],
    "spin": {
      "x": 0,
      "y": 0
    },
    "power": 2.5
  },
  "author_id": 2,
  "folder_id": null,
  "created_at": "2026-05-06T00:00:00.000Z",
  "updated_at": "2026-05-06T00:00:00.000Z",
  "deleted_at": null
}
```

### 6.2. Поле position

`position` может быть объектом или `null`.

Разрешенные поля `position`:

| Поле    | Тип    | Обязательное | Описание                |
| ------- | ------ | -----------: | ----------------------- |
| `balls` | array  |          Нет | Массив шаров            |
| `lines` | array  |          Нет | Массив линий/траекторий |
| `spin`  | object |          Нет | Точка спина             |
| `power` | number |          Нет | Сила удара              |

Лишние поля внутри `position` запрещены.

Ball:

| Поле   | Тип    | Обязательное | Правила        |
| ------ | ------ | -----------: | -------------- |
| `id`   | string |          Нет | 1-100 символов |
| `x`    | number |           Да | Координата     |
| `y`    | number |           Да | Координата     |
| `type` | string |           Да | 1-50 символов  |

Line:

| Поле   | Тип    | Обязательное | Правила        |
| ------ | ------ | -----------: | -------------- |
| `id`   | string |          Нет | 1-100 символов |
| `from` | object |           Да | `Point`        |
| `to`   | object |           Да | `Point`        |
| `type` | string |           Да | 1-50 символов  |

Point:

| Поле | Тип    | Обязательное |
| ---- | ------ | -----------: |
| `x`  | number |           Да |
| `y`  | number |           Да |

### 6.3. GET /exercises

Возвращает список неудаленных упражнений.

Авторизация: обязательна.

Доступ:

- `admin` получает все упражнения.
- `coach` получает только свои упражнения.
- `athlete` получает `403 direct_exercise_access_forbidden`.

Query params:

| Параметр   | Тип     | По умолчанию | Правила                                                |
| ---------- | ------- | -----------: | ------------------------------------------------------ |
| `search`   | string  |            - | 1-255 символов, поиск по `title`, case-insensitive     |
| `folderId` | integer |            - | >= 1                                                   |
| `authorId` | integer |            - | >= 1, для admin любой, для coach только собственный id |
| `limit`    | integer |         `20` | 1-100                                                  |
| `offset`   | integer |          `0` | >= 0                                                   |

Response `200`:

```json
{
  "ok": true,
  "exercises": [],
  "pagination": {
    "limit": 20,
    "offset": 0
  }
}
```

Пример:

```js
const params = new URLSearchParams({
  search: "pot",
  limit: "20",
  offset: "0",
});

const response = await fetch(`http://localhost:3000/exercises?${params}`, {
  headers: {
    Authorization: `Bearer ${accessToken}`,
  },
});
```

### 6.4. GET /exercises/:id

Возвращает одно упражнение.

Авторизация: обязательна.

Доступ:

- `admin` может получить любое упражнение.
- `coach` может получить только свое упражнение.
- `athlete` получает `403`.

Path params:

| Параметр | Тип     | Правила |
| -------- | ------- | ------- |
| `id`     | integer | >= 1    |

Response `200`:

```json
{
  "ok": true,
  "exercise": {
    "id": 1,
    "title": "Long pot practice",
    "description": null,
    "position": null,
    "author_id": 2,
    "folder_id": null,
    "created_at": "2026-05-06T00:00:00.000Z",
    "updated_at": "2026-05-06T00:00:00.000Z",
    "deleted_at": null
  }
}
```

Если упражнение удалено, не существует или недоступно coach, backend вернет `404 not_found`.

### 6.5. POST /exercises

Создает упражнение.

Авторизация: обязательна.

Доступ:

- `admin`
- `coach`

`athlete` получает `403`.

Request body:

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

Поля:

| Поле          | Тип              | Обязательное | Правила               |
| ------------- | ---------------- | -----------: | --------------------- |
| `title`       | string           |           Да | 3-255 символов        |
| `description` | string или null  |          Нет | До 3000 символов      |
| `position`    | object или null  |          Нет | См. модель `position` |
| `folderId`    | integer или null |          Нет | >= 1                  |

Лишние поля запрещены.

Важно:

- В request body используется `folderId`.
- В response backend возвращает `folder_id`.
- `author_id` выставляется backend автоматически из токена.

Response `201`:

```json
{
  "ok": true,
  "exercise": {
    "id": 1,
    "title": "Long pot practice",
    "description": "Practice long pot with cue ball control.",
    "position": {
      "power": 2.5
    },
    "author_id": 2,
    "folder_id": null,
    "created_at": "2026-05-06T00:00:00.000Z",
    "updated_at": "2026-05-06T00:00:00.000Z",
    "deleted_at": null
  }
}
```

Пример:

```js
const response = await fetch("http://localhost:3000/exercises", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    Authorization: `Bearer ${accessToken}`,
  },
  body: JSON.stringify({
    title: "Long pot practice",
    description: "Practice long pot with cue ball control.",
    position: {
      power: 2.5,
    },
  }),
});
```

### 6.6. PUT /exercises/:id

Обновляет упражнение.

Авторизация: обязательна.

Доступ:

- `admin` может обновить любое упражнение.
- `coach` может обновить только свое упражнение.
- `athlete` получает `403`.

Request body должен содержать хотя бы одно поле.

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

Поля такие же, как в `POST /exercises`, но все необязательные.

Особенности:

- `PUT` делает частичное обновление, несмотря на название метода.
- Если поле не передать, оно не изменится.
- Если передать `description: null`, `position: null` или `folderId: null`, значение будет очищено.
- `updated_at` обновляется автоматически.

Response `200`:

```json
{
  "ok": true,
  "exercise": {
    "id": 1,
    "title": "Updated long pot practice",
    "description": null,
    "position": {
      "power": 3
    },
    "author_id": 2,
    "folder_id": null,
    "created_at": "2026-05-06T00:00:00.000Z",
    "updated_at": "2026-05-06T00:01:00.000Z",
    "deleted_at": null
  }
}
```

### 6.7. DELETE /exercises/:id

Soft-delete упражнения.

Авторизация: обязательна.

Доступ:

- `admin` может удалить любое упражнение.
- `coach` может удалить только свое упражнение.
- `athlete` получает `403`.

Response `200`:

```json
{
  "ok": true,
  "exercise": {
    "id": 1,
    "title": "Long pot practice",
    "description": null,
    "position": null,
    "author_id": 2,
    "folder_id": null,
    "created_at": "2026-05-06T00:00:00.000Z",
    "updated_at": "2026-05-06T00:02:00.000Z",
    "deleted_at": "2026-05-06T00:02:00.000Z"
  }
}
```

После удаления:

- `GET /exercises` не возвращает упражнение.
- `GET /exercises/:id` вернет `404 not_found`.

## 7. Особенности, важные для frontend

### 7.1. CamelCase в запросах, snake_case в ответах

В body запросов упражнений frontend отправляет:

```json
{
  "folderId": 1
}
```

В ответе backend возвращает:

```json
{
  "folder_id": 1,
  "author_id": 2
}
```

### 7.2. Лишние поля запрещены

AJV-схемы используют `additionalProperties: false`.

Например, такой запрос упадет с `400 validation_error`:

```json
{
  "title": "Long pot practice",
  "authorId": 2
}
```

### 7.3. Query параметры приводятся к числам

Backend включает `coerceTypes`, поэтому query params вроде:

```text
?limit=20&offset=0
```

будут приведены к integer.

### 7.4. Пагинация без total

`GET /exercises` возвращает только:

```json
{
  "pagination": {
    "limit": 20,
    "offset": 0
  }
}
```

Поля `total`, `page`, `hasMore` сейчас нет.

### 7.5. Swagger UI не подключен

Файл `openapi.yml` есть в корне проекта, но Express сейчас не раздает Swagger UI и не раздает сам YAML как endpoint.

Для просмотра можно открыть файл локально в Swagger Editor или импортировать в Postman/Insomnia.

### 7.6. CORS

По умолчанию разрешены все origins:

```text
CORS_ORIGIN=*
```

Для production нужно задать конкретный frontend origin.

## 8. Минимальный frontend flow

### 8.1. Login

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

  if (!response.ok) {
    throw data;
  }

  return data;
}
```

### 8.2. Authorized request helper

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

  if (!response.ok) {
    throw data;
  }

  return data;
}
```

### 8.3. Load exercises

```js
const data = await apiFetch("/exercises?limit=20&offset=0", {
  token: accessToken,
});

console.log(data.exercises);
```

### 8.4. Create exercise

```js
const data = await apiFetch("/exercises", {
  token: accessToken,
  method: "POST",
  body: {
    title: "Long pot practice",
    description: "Practice long pot with cue ball control.",
    position: {
      power: 2.5,
    },
  },
});

console.log(data.exercise);
```

## 9. Быстрая ручная проверка API

В проекте есть PowerShell-скрипт:

```text
scripts/visible-api-flow.ps1
```

Он проверяет полный сценарий:

- healthcheck
- login seed admin
- регистрация coach
- login coach
- регистрация athlete
- login athlete
- создание упражнения
- список упражнений
- получение по id
- обновление
- soft delete
- проверка `404` после удаления
- проверка запрета прямого доступа для athlete

Запускать после поднятия backend:

```powershell
.\scripts\visible-api-flow.ps1
```

## 10. Что пока не реализовано в API

Сейчас нет HTTP API для:

- папок (`folders`)
- списка пользователей
- редактирования пользователей
- удаления пользователей
- связи athlete с упражнениями
- выдачи упражнений athlete через coach
- refresh token
- logout/session invalidation
- Swagger UI endpoint
- миграций БД
- `total` для пагинации

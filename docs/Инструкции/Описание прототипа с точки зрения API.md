# API для coach-прототипа withGroups

Цель файла: коротко показать frontend-разработчику, какие методы дергать на ключевых страницах прототипа `front-prototypes/withGroups/coach`.

Все методы, кроме login/register, вызываются с заголовком:

```http
Authorization: Bearer <accessToken>
```

Базовый URL:

```text
http://localhost:3000
```

## Авторизация

### POST /auth/login

Вход тренера.

Request:

```json
{
  "email": "coach@example.com",
  "password": "123"
}
```

Response:

```json
{
  "ok": true,
  "accessToken": "jwt-token",
  "user": {
    "id": "2",
    "email": "coach@example.com",
    "name": "Coach User",
    "role": "coach",
    "coachId": null
  }
}
```

## Страница "Календарь"

### GET /trainings

Получить тренировки тренера. Для календаря лучше сразу передавать период.

```http
GET /trainings?dateFrom=2026-05-01T00:00:00+07:00&dateTo=2026-06-01T00:00:00+07:00&limit=100&offset=0
```

Дополнительные фильтры:

```text
date=2026-05-20
status=scheduled|in_progress|completed|cancelled|missed
groupId=1
athleteId=3
limit=20
offset=0
```

Response:

```json
{
  "ok": true,
  "trainings": [
    {
      "type": "group",
      "groupTraining": {
        "id": "1",
        "coachId": "2",
        "groupId": "1",
        "startsAt": "2026-05-20T03:00:00.000Z",
        "endsAt": "2026-05-20T04:30:00.000Z",
        "status": "scheduled",
        "athleteTrainingsCount": 3
      }
    },
    {
      "type": "athlete",
      "athleteTraining": {
        "id": "10",
        "groupTrainingId": null,
        "coachId": "2",
        "athleteId": "3",
        "athlete": {
          "id": "3",
          "email": "athlete@example.com",
          "name": "Athlete User",
          "role": "athlete",
          "coachId": "2"
        },
        "startsAt": "2026-05-20T08:00:00.000Z",
        "endsAt": "2026-05-20T09:30:00.000Z",
        "status": "in_progress",
        "itemsCount": 2
      }
    }
  ],
  "pagination": {
    "limit": 100,
    "offset": 0,
    "total": 2
  }
}
```

## Страница "Группы"

### GET /groups

Открыли страницу групп.

Response:

```json
{
  "ok": true,
  "groups": [
    {
      "id": "1",
      "name": "Junior A",
      "coachId": "2",
      "description": "Morning junior group",
      "athletesCount": 3,
      "createdAt": "2026-05-13T00:00:00.000Z",
      "updatedAt": "2026-05-13T00:00:00.000Z",
      "deletedAt": null
    }
  ]
}
```

### GET /groups/:id

Открыли карточку группы. В ответе нужны данные спортсменов группы.

### POST /groups

Создали группу.

Request:

```json
{
  "name": "Junior A",
  "description": "Morning junior group",
  "athleteIds": [3, 4]
}
```

### PATCH /groups/:id

Редактировали название или описание группы.

Request:

```json
{
  "name": "Junior A Updated",
  "description": "Updated description"
}
```

### DELETE /groups/:id

Soft-delete группы.

### POST /groups/:id/athletes

Добавили спортсмена в группу.

Request:

```json
{
  "athleteId": 3
}
```

### DELETE /groups/:id/athletes/:athleteId

Удалили спортсмена из группы.

## Страница "Спортсмены"

### GET /users?role=athlete

Открыли список спортсменов тренера.

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

### POST /auth/register

Создали спортсмена. В MVP создание пользователя пока идет через auth endpoint.

Request:

```json
{
  "email": "new.athlete@example.com",
  "password": "123",
  "name": "New Athlete",
  "role": "athlete"
}
```

### GET /users/:id

Открыли карточку спортсмена.

### PATCH /users/:id

Изменили данные спортсмена.

Request:

```json
{
  "name": "Updated Athlete",
  "email": "updated.athlete@example.com",
  "password": "123"
}
```

Response:

```json
{
  "ok": true,
  "user": {
    "id": "3",
    "email": "updated.athlete@example.com",
    "name": "Updated Athlete",
    "role": "athlete",
    "coachId": "2"
  }
}
```

### DELETE /users/:id

Soft-delete спортсмена. Если спортсмен состоял в группах, активные связи с группами тоже soft-deleted.

## Страница "Новая тренировка"

Для открытия формы нужны:

```http
GET /groups
GET /folders
GET /users?role=athlete
GET /users?role=athlete&groupId=1
GET /exercises?limit=100&offset=0
```

### POST /trainings

Индивидуальная тренировка:

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

Групповая тренировка с одинаковыми упражнениями для всех спортсменов группы:

```json
{
  "groupId": 1,
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

Групповая тренировка с разными упражнениями по спортсменам:

```json
{
  "groupId": 1,
  "startsAt": "2026-05-20T10:00:00+07:00",
  "endsAt": "2026-05-20T11:30:00+07:00",
  "athleteTrainings": [
    {
      "athleteId": 3,
      "items": [
        {
          "exerciseId": 1,
          "orderIndex": 0,
          "targetAttempts": 10
        },
        {
          "exerciseId": 2,
          "orderIndex": 1,
          "targetAttempts": 12
        }
      ]
    },
    {
      "athleteId": 4,
      "items": [
        {
          "exerciseId": 1,
          "orderIndex": 0,
          "targetAttempts": 8
        }
      ]
    }
  ]
}
```

Важно: нельзя одновременно передавать непустые `items` и `athleteTrainings`.

## Страница "Детали тренировки"

### GET /trainings/group/:id

Получить групповую тренировку, ее athlete trainings и items.

### GET /trainings/athlete/:id

Получить тренировку конкретного спортсмена. Внутри ответа есть `athlete` и `items`.

### PATCH /trainings/group/:id

Меняет даты групповой тренировки, пока статус `scheduled`.

### PATCH /trainings/athlete/:id

Меняет даты или спортсмена индивидуальной тренировки, пока статус `scheduled`.

### DELETE /trainings/group/:id

Soft-delete групповой тренировки, дочерних athlete trainings и items.

### DELETE /trainings/athlete/:id

Soft-delete athlete training и items.

### PATCH /trainings/group/:id/status

Меняет статус group training и синхронизирует статус в дочерние athlete trainings.

Request:

```json
{
  "status": "in_progress"
}
```

### PATCH /trainings/athlete/:id/status

Меняет статус athlete training.

## Управление упражнениями внутри тренировки

### POST /trainings/athlete/:id/items

Добавить item конкретному спортсмену.

### PATCH /trainings/athlete/:id/items/:itemId

Изменить item, пока athlete training в `scheduled`.

### DELETE /trainings/athlete/:id/items/:itemId

Soft-delete item, пока athlete training в `scheduled`.

### PATCH /trainings/athlete/:id/items/:itemId/status

Меняет статус item.

```json
{
  "status": "in_progress"
}
```

### PATCH /trainings/athlete/:id/items/:itemId/result

Записать результат.

```json
{
  "resultAttempts": 10,
  "resultSuccesses": 6
}
```

## Закрытые gaps

В этом обновлении закрыты основные пробелы прототипа:

- добавлены `GET/PATCH/DELETE /users/:id`;
- добавлены `GET/POST/PATCH/DELETE /folders`;
- exercises endpoints возвращают camelCase и `pagination.total`;
- `POST /trainings` принимает разные `items` по разным спортсменам через `athleteTrainings`;
- в ответах `athleteTraining` есть вложенный объект `athlete`;
- `GET /trainings` поддерживает `date`, `dateFrom`, `dateTo`, `status`, `groupId`, `athleteId`, `limit`, `offset` и считает пагинацию в SQL.

Что остается особенностью MVP: создание спортсмена пока выполняется через `POST /auth/register`, а не через отдельный `POST /users`.

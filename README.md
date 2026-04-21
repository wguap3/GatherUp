# GatherUp

Backend REST API платформы для публикации событий и поиска участников. Пользователи создают мероприятия, другие подают заявки на участие. Отдельный сервис статистики собирает данные о просмотрах.

---

## Архитектура

Система состоит из двух независимых микросервисов с отдельными БД:

```
Client → main-service :8080 → stats-service :9090
              ↓                        ↓
         PostgreSQL               PostgreSQL
```

- **main-service** — основная бизнес-логика: события, заявки, категории, подборки
- **stats-service** — сбор и хранение статистики просмотров; main-service обращается к нему через WebClient при каждом публичном запросе к событиям

---

## Стек

- **Java 17**
- **Spring Boot** — REST API, WebClient
- **Spring Data JPA / Hibernate**
- **PostgreSQL**
- **Lombok, MapStruct, Log4j2**
- **Maven, Docker, Docker Compose**

---

## Запуск

```bash
git clone https://github.com/wguap3/GatherUp.git
cd GatherUp
docker-compose up --build
```

- main-service: `http://localhost:8080`
- stats-service: `http://localhost:9090`

---

## API — Main Service (8080)

### Публичные эндпоинты

| Метод | Путь | Описание |
|-------|------|----------|
| `GET` | `/events` | Список опубликованных событий с фильтрацией по тексту, категории, датам, сортировкой по дате или просмотрам |
| `GET` | `/events/{id}` | Подробная информация об опубликованном событии |
| `GET` | `/categories` | Список категорий с пагинацией |
| `GET` | `/categories/{catId}` | Категория по id |
| `GET` | `/compilations` | Подборки событий (можно фильтровать по закреплённым) |
| `GET` | `/compilations/{compId}` | Подборка по id |

> При каждом обращении к `/events` и `/events/{id}` hit автоматически сохраняется в stats-service

---

### Приватные эндпоинты (для авторизованных пользователей)

| Метод | Путь | Описание |
|-------|------|----------|
| `GET` | `/users/{userId}/events` | Мои события с пагинацией |
| `POST` | `/users/{userId}/events` | Создать событие |
| `GET` | `/users/{userId}/events/{eventId}` | Моё событие по id |
| `PATCH` | `/users/{userId}/events/{eventId}` | Обновить событие (только pending или cancelled) |
| `GET` | `/users/{userId}/events/{eventId}/requests` | Заявки на участие в моём событии |
| `PATCH` | `/users/{userId}/events/{eventId}/requests` | Подтвердить или отклонить заявки |
| `GET` | `/users/{userId}/requests` | Мои заявки на чужие события |
| `POST` | `/users/{userId}/requests?eventId=` | Подать заявку на участие |
| `PATCH` | `/users/{userId}/requests/{requestId}/cancel` | Отменить свою заявку |

---

### Административные эндпоинты

**Пользователи**

| Метод | Путь | Описание |
|-------|------|----------|
| `GET` | `/admin/users` | Список пользователей с фильтрацией по ids и пагинацией |
| `POST` | `/admin/users` | Создать пользователя |
| `DELETE` | `/admin/users/{userId}` | Удалить пользователя |

**Категории**

| Метод | Путь | Описание |
|-------|------|----------|
| `POST` | `/admin/categories` | Создать категорию (имя уникально) |
| `PATCH` | `/admin/categories/{catId}` | Обновить категорию |
| `DELETE` | `/admin/categories/{catId}` | Удалить категорию (нельзя если есть события) |

**События**

| Метод | Путь | Описание |
|-------|------|----------|
| `GET` | `/admin/events` | Поиск событий с фильтрацией по пользователям, статусам, категориям, датам |
| `PATCH` | `/admin/events/{eventId}` | Редактировать событие, опубликовать или отклонить |

**Подборки**

| Метод | Путь | Описание |
|-------|------|----------|
| `POST` | `/admin/compilations` | Создать подборку |
| `PATCH` | `/admin/compilations/{compId}` | Обновить подборку |
| `DELETE` | `/admin/compilations/{compId}` | Удалить подборку |

---

## API — Stats Service (9090)

| Метод | Путь | Описание |
|-------|------|----------|
| `POST` | `/hit` | Сохранить информацию о запросе к эндпоинту |
| `GET` | `/stats?start=&end=&uris=&unique=` | Получить статистику за период по uri, опционально только уникальные ip |

```json
// POST /hit
{
  "app": "ewm-main-service",
  "uri": "/events/1",
  "ip": "192.163.0.1",
  "timestamp": "2024-03-10 14:30:00"
}

// GET /stats?start=2024-01-01 00:00:00&end=2024-12-31 23:59:59&uris=/events/1&unique=true
[
  { "app": "ewm-main-service", "uri": "/events/1", "hits": 6 }
]
```

---

## Жизненный цикл события

```
PENDING → PUBLISHED  (admin: PUBLISH_EVENT)
PENDING → CANCELED   (admin: REJECT_EVENT или user: CANCEL_REVIEW)
CANCELED → PENDING   (user: SEND_TO_REVIEW)
```

Редактировать событие можно только в статусе `PENDING` или `CANCELED`. Дата события — не ранее чем через 2 часа от текущего момента.

---

## Ключевые решения

- **Изоляция нагрузки** — статистика в отдельном сервисе, пиковая нагрузка на аналитику не влияет на основной сервис
- **Оптимизация запросов** — устранена проблема N+1 через Entity Graph и Fetch Joins, количество SQL-запросов на ключевых эндпоинтах сократилось в ~3 раза
- **Маппинг** — MapStruct убирает boilerplate-код в слое DTO и снижает риск ошибок при изменении схемы
- **Контейнеризация** — два сервиса и две БД поднимаются одной командой `docker-compose up`

---

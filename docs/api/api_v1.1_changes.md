# Спецификация API LangMeet — версия 1.1 (изменения итерации 2)

Базовый префикс: `/api/v1`. Аутентификация: `Authorization: Bearer <access_token>`.
Версия 1.0 → 1.1: добавлены 3 эндпоинта и 1 код ошибки. Существующие эндпоинты 1–4 сохраняют поведение; обратная совместимость сохранена.

## 1. Полный перечень эндпоинтов

| № | Метод | URL | Описание | Успех | Роль | Версия |
|---|---|---|---|---|---|---|
| 1 | POST | /api/v1/match-requests | Создать заявку на подбор | 202 Accepted | Ученик | 1.0 |
| 2 | GET | /api/v1/match-requests/{request_id} | Получить статус заявки | 200 OK | Ученик | 1.0, изменён (F2) |
| 3 | DELETE | /api/v1/match-requests/{request_id} | Отменить заявку | 204 No Content | Ученик | 1.0 |
| 4 | GET | /api/v1/sessions/{session_id} | Получить данные сессии для входа | 200 OK | Участник сессии | 1.0, изменён (F2) |
| 5 | POST | /api/v1/auth/2fa/verify | Подтверждение входа кодом TOTP | 200 OK | Преподаватель, Администратор | 1.1 (F1) |
| 6 | POST | /api/v1/sessions/{session_id}/invitations | Создать приглашение собеседнику по ссылке | 201 Created | Участник сессии | 1.1 (F3) |
| 7 | GET | /api/v1/sessions/{session_id}/report?format=pdf | Экспорт отчёта сессии в PDF | 200 OK, application/pdf | Преподаватель | 1.1 (F4) |

## 2. Новые эндпоинты

### 2.1. POST /api/v1/auth/2fa/verify (F1)

Вызывается после шага «логин» для ролей «Преподаватель» и «Администратор». Тело запроса:

```json
{
  "mfa_token": "строка, полученная на шаге логина",
  "totp_code": "123456"
}
```

Правила: `totp_code` — ровно 6 цифр. Ответ `200 OK` содержит `access_token` и `refresh_token`. После 5 неверных кодов подряд вход блокируется на 15 минут.

### 2.2. POST /api/v1/sessions/{session_id}/invitations (F3)

Создаёт приглашение, которое принимается по ссылке и не требует прохождения очереди подбора. Тело запроса:

```json
{
  "expires_in_minutes": 60
}
```

Ответ `201 Created`:

```json
{
  "data": {
    "invitation_id": "9b1d2c3e-4f5a-4b6c-8d7e-0f1a2b3c4d5e",
    "invite_url": "https://langmeet.example/invite/<token>",
    "expires_at": "2026-03-10T11:00:00Z"
  },
  "meta": { "version": "1.1" }
}
```

Токен приглашения в базе хранится как хэш, срок действия ограничен `expires_at`.

### 2.3. GET /api/v1/sessions/{session_id}/report?format=pdf (F4)

Возвращает PDF-отчёт по итогам завершённой сессии. Заголовки ответа: `Content-Type: application/pdf`, `Content-Disposition: attachment; filename="report_<session_id>.pdf"`. Если сессия не завершена, возвращается `409 CONFLICT`.

## 3. Изменения существующих эндпоинтов (F2)

Для `GET /api/v1/match-requests/{request_id}` и `GET /api/v1/sessions/{session_id}`:
- ответ содержит заголовок `Cache-Control: max-age=2`;
- актуальный статус заявки читается из Redis (тот же Redis, который уже используется как очередь заявок, ЛР №4);
- целевой показатель: p95 ≤ 200 мс (ТЗ, п. 4.4.2 в версии 2).

## 4. Новый код ошибки

| HTTP | Код | Когда возникает | Поведение клиента |
|---|---|---|---|
| 401 | INVALID_MFA_CODE | Неверный или просроченный код TOTP | Ввести код повторно; после 5 ошибок подряд — блокировка на 15 минут |

Полная таблица кодов ошибок 400–503 остаётся без изменений (ЛР №4, таблица 7).

## 5. Пример успешного ответа (после 2FA)

```json
{
  "data": {
    "access_token": "eyJhbGciOi...",
    "refresh_token": "dGhpcyBpcyBh...",
    "expires_in": 900
  },
  "meta": { "version": "1.1", "timestamp": "2026-03-10T10:00:00Z" }
}
```
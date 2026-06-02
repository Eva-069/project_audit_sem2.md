## 1. Контекст перевірки

Тестується Feature #1 — нагадування за 60 хв до початку бронювання. За основу взято AC з feature_1_spec.md, API з openapi.yaml та error_catalog.md, UX з ПР-6, безпеку з ПР-8.

---

## 2. Traceability Map

| ID    | Вимога / AC / інваріант                                   | Звідки     | Тест         | Покриття |
| ----- | --------------------------------------------------------- | ---------- | ------------ | -------- |
| TR-01 | AC-1: scheduled_at = start_at − 60 хв, status = scheduled | ПР-3       | TC-01        | Повністю |
| TR-02 | AC-2: до початку < 60 хв → status = sent одразу           | ПР-3       | TC-02        | Повністю |
| TR-03 | AC-3: скасування бронювання → reminder.status = cancelled | ПР-3       | TC-03        | Повністю |
| TR-04 | AC-4: повторний POST → 409 reminder_already_exists        | ПР-3, ПР-5 | TC-05        | Повністю |
| TR-05 | AC-5: customer.email не у відповіді API                   | ПР-3, ПР-8 | TC-10        | Повністю |
| TR-06 | AC-6: чуже бронювання → 403 permission_denied             | ПР-3, ПР-8 | TC-09, TC-13 | Повністю |
| TR-07 | AC-8: PATCH cancelled на sent → 409 reminder_already_sent | ПР-3, ПР-5 | TC-07        | Повністю |
| TR-08 | channel тільки "email" → 400 invalid_channel              | ПР-5       | TC-06        | Повністю |
| TR-09 | message макс. 500 символів → 400 message_too_long         | ПР-5       | TC-08, TC-12 | Повністю |
| TR-10 | Без токена → 401 authentication_required                  | ПР-8       | TC-11        | Повністю |
| TR-11 | AC-7: retry при невдачі надсилання, макс. 2 спроби        | ПР-3       | TC-14        | Частково |
| TR-12 | booking_not_found → 404                                   | ПР-5       | TC-15        | Повністю |

---

## 3. Тест-кейси

**TC-01 Створення нагадування — до початку ≥ 60 хвилин**
Тип: positive
Передумови: бронювання bkg_a1b2c3d4 існує, start_at = зараз + 2 години, нагадування немає, клієнт авторизований (scope bookings:write)
Кроки:

1. POST /api/v1/bookings/bkg_a1b2c3d4/reminder
2. Header: Authorization: Bearer {token}
3. Body: {"channel": "email"}
   Очікуваний результат: HTTP 201

```json
{
  "reminder_id": "rem_x1y2z3w4",
  "booking_id": "bkg_a1b2c3d4",
  "status": "scheduled",
  "scheduled_at": "2025-11-08T15:00:00Z",
  "sent_at": null,
  "channel": "email"
}
```

Поле email відсутнє у відповіді. scheduled_at = start_at − 60 хв.
Покриває: AC-1, TR-01

---

**TC-02 Створення нагадування — до початку < 60 хвилин**
Тип: positive
Передумови: start_at = зараз + 30 хвилин, нагадування немає, клієнт авторизований
Кроки:

1. POST /api/v1/bookings/bkg_b2c3d4e5/reminder
2. Header: Authorization: Bearer {token}
3. Body: {"channel": "email"}
   Очікуваний результат: HTTP 201, status = sent, sent_at заповнено (не null)
   Покриває: AC-2, TR-02

---

**TC-03 Скасування бронювання автоматично скасовує нагадування**
Тип: positive
Передумови: нагадування rem_x1y2z3w4 зі статусом scheduled існує для bkg_a1b2c3d4. Клієнт авторизований як власник.
Кроки:

1. PATCH /api/v1/bookings/bkg_a1b2c3d4, body: {"status": "cancelled"}, токен клієнта-власника
2. GET /api/v1/bookings/bkg_a1b2c3d4/reminder, той самий токен
   Очікуваний результат: HTTP 200

```json
{
  "reminder_id": "rem_x1y2z3w4",
  "booking_id": "bkg_a1b2c3d4",
  "status": "cancelled",
  "sent_at": null
}
```

Перехід стану: scheduled → cancelled. sent_at = null — нагадування не надіслано.
Покриває: AC-3, TR-03

---

**TC-04 GET — перегляд свого нагадування**
Тип: positive
Передумови: нагадування існує, клієнт є власником bkg_a1b2c3d4
Кроки:

1. GET /api/v1/bookings/bkg_a1b2c3d4/reminder
2. Header: Authorization: Bearer {token}
   Очікуваний результат: HTTP 200, є reminder_id, status, scheduled_at, channel. Поле email відсутнє.
   Покриває: AC-5, TR-05

---

**TC-05 Повторний POST — нагадування вже існує**
Тип: negative
Передумови: нагадування зі статусом scheduled вже є для bkg_a1b2c3d4
Кроки:

1. POST /api/v1/bookings/bkg_a1b2c3d4/reminder повторно
2. Body: {"channel": "email"}
   Очікуваний результат: HTTP 409

```json
{
  "error": {
    "error_code": "reminder_already_exists",
    "message": "Нагадування для цього бронювання вже існує",
    "details": {
      "existing_status": "scheduled"
    }
  }
}
```

booking_id не повертається в details — достатньо error_code і existing_status. Нового нагадування не створено.
Покриває: AC-4, TR-04

---

**TC-06 Невалідний channel**
Тип: negative
Передумови: бронювання існує, клієнт авторизований
Кроки:

1. POST /api/v1/bookings/bkg_a1b2c3d4/reminder
2. Body: {"channel": "sms"}
   Очікуваний результат: HTTP 400

```json
{
  "error": {
    "error_code": "invalid_channel",
    "message": "Канал має бути email"
  }
}
```

details не містить введеного значення.
Покриває: TR-08

---

**TC-07 PATCH cancelled на sent**
Тип: negative
Передумови: нагадування має статус sent
Кроки:

1. PATCH /api/v1/bookings/bkg_b2c3d4e5/reminder
2. Body: {"status": "cancelled"}
   Очікуваний результат: HTTP 409

```json
{
  "error": {
    "error_code": "reminder_already_sent",
    "message": "Неможливо скасувати нагадування яке вже надіслано"
  }
}
```

Статус не змінився.
Покриває: AC-8, TR-07

---

**TC-08 message перевищує 500 символів**
Тип: negative
Передумови: бронювання існує, нагадування відсутнє
Кроки:

1. POST /api/v1/bookings/bkg_a1b2c3d4/reminder
2. Body: {"channel": "email", "message": рядок з 501 символу}
   Очікуваний результат: HTTP 400

```json
{
  "error": {
    "error_code": "message_too_long",
    "message": "Текст повідомлення не може перевищувати 500 символів",
    "details": { "message": "max_length: 500" }
  }
}
```

Покриває: TR-09

---

**TC-09 Доступ до чужого нагадування — GET**
Тип: access
Передумови: клієнт A авторизований, bkg_X9Y8Z7 належить клієнту B
Кроки:

1. GET /api/v1/bookings/bkg_X9Y8Z7/reminder
2. Header: Authorization: Bearer {token_A}
   Очікуваний результат: HTTP 403

```json
{
  "error": {
    "error_code": "permission_denied",
    "message": "У вас немає прав на це нагадування"
  }
}
```

Жодних даних клієнта B не повернуто.
Покриває: AC-6, TR-06

---

**TC-10 PII — email не у відповіді**
Тип: security
Передумови: нагадування існує, клієнт авторизований
Кроки:

1. POST /api/v1/bookings/bkg_a1b2c3d4/reminder → 201
2. Перевірити всі поля тіла відповіді
   Очікуваний результат: поля email, customer_email відсутні. Жодне значення не містить @. Відповідь містить тільки: reminder_id, booking_id, status, scheduled_at, sent_at, channel.
   Покриває: AC-5, TR-05

---

**TC-11 Запит без авторизації**
Тип: negative
Передумови: немає токена
Кроки:

1. POST /api/v1/bookings/bkg_a1b2c3d4/reminder без заголовку Authorization
   Очікуваний результат: HTTP 401

```json
{
  "error": {
    "error_code": "authentication_required",
    "message": "Для цієї дії потрібна авторизація"
  }
}
```

Покриває: TR-10

---

**TC-12 Edge case — message рівно 500 символів**
Тип: edge
Передумови: бронювання існує, нагадування відсутнє
Кроки:

1. POST /api/v1/bookings/bkg_a1b2c3d4/reminder
2. Body: {"channel": "email", "message": рядок рівно 500 символів}
   Очікуваний результат: HTTP 201, нагадування створено, message збережено.
   Покриває: TR-09, граничне значення

---

**TC-13 Доступ до чужого нагадування — PATCH**
Тип: access
Передумови: клієнт A авторизований, bkg_X9Y8Z7 належить клієнту B, нагадування має статус scheduled
Кроки:

1. PATCH /api/v1/bookings/bkg_X9Y8Z7/reminder
2. Header: Authorization: Bearer {token_A}
3. Body: {"status": "cancelled"}
   Очікуваний результат: HTTP 403

```json
{
  "error": {
    "error_code": "permission_denied",
    "message": "У вас немає прав на це нагадування"
  }
}
```

Статус нагадування клієнта B не змінився.
Покриває: TR-06

---

**TC-14 Retry при невдачі надсилання — мок поштового сервісу**
Тип: edge
Передумови: бронювання існує, нагадування заплановане, поштовий сервіс повертає помилку (мок)
Кроки:

1. Налаштувати мок поштового сервісу на відповідь з помилкою
2. Дочекатись scheduled_at нагадування
3. Перевірити NotificationLog після першої спроби
4. Дочекатись повторної спроби через 5 хвилин
5. Перевірити attempt_count у NotificationLog
   Очікуваний результат: після двох невдалих спроб attempt_count = 2, delivery_status = failed. Якщо поштовий сервіс відновився — третя спроба дає delivery_status = success.
   Примітка: тест потребує мокування поштового сервісу — не виконується вручну.
   Покриває: AC-7, TR-11

---

**TC-15 Бронювання не існує**
Тип: negative
Передумови: клієнт авторизований, booking_id не існує в системі
Кроки:

1. POST /api/v1/bookings/bkg_nonexistent/reminder
2. Header: Authorization: Bearer {token}
3. Body: {"channel": "email"}
   Очікуваний результат: HTTP 404

```json
{
  "error": {
    "error_code": "booking_not_found",
    "message": "Бронювання не знайдено"
  }
}
```

Покриває: TR-12

---

## 4. Негативні сценарії і помилки

Всі помилки повертаються у єдиному форматі:

```json
{
  "error": {
    "error_code": "...",
    "message": "...",
    "details": {...}
  }
}
```

error_code стабільний між версіями. message може змінитись. Stack trace клієнту не повертається.

| error_code              | HTTP | Тест         | Що перевіряємо                                |
| ----------------------- | ---- | ------------ | --------------------------------------------- |
| invalid_channel         | 400  | TC-06        | error_code, відсутність sms в details         |
| message_too_long        | 400  | TC-08        | error_code, details.message = max_length: 500 |
| authentication_required | 401  | TC-11        | error_code, HTTP 401                          |
| permission_denied       | 403  | TC-09, TC-13 | error_code, відсутність даних чужого запису   |
| booking_not_found       | 404  | TC-15        | error_code, HTTP 404                          |
| reminder_already_exists | 409  | TC-05        | error_code, existing_status без booking_id    |
| reminder_already_sent   | 409  | TC-07        | error_code, статус не змінився                |

В UX: Error Summary + inline під полем, дані не зникають при помилці, фокус на summary після submit.

---

## 5. Покриття ризиків

**Ризик 1 — Дублювання нагадування**
Наслідок: клієнт отримує два emails, порушення інваріанту
Тести: TC-05
Прогалина: race condition (два одночасних POST) не покрито — потребує навантажувального тесту

**Ризик 2 — Витік customer.email**
Наслідок: PII у відповіді API
Тести: TC-10
Прогалина: чи email не потрапляє в NotificationLog — потребує тесту на рівні інфраструктури

**Ризик 3 — IDOR**
Наслідок: клієнт переглядає або змінює чуже нагадування
Тести: TC-09 (GET), TC-13 (PATCH)
Прогалина: покрито повністю для API-рівня

---

## 6. Що не покрито

- Race condition при одночасних POST — захист є в коді (транзакція), але автотест відсутній
- TC-14 (retry) потребує мок поштового сервісу — вручну не виконується
- Email у NotificationLog — перевірка тільки на рівні інфраструктури, не через API

---

## 7. Висновок

15 тестів покривають основні AC і найважливіші ризики. Критичні для демо: TC-01, TC-02, TC-05, TC-09, TC-10. Race condition і перевірка логів залишаються непокритими — прийнятно для поточного етапу.

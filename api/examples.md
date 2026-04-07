# api/examples.md — Приклади запитів і відповідей

**Feature #1 — Нагадування · КВ14736512**

---

## Успішні сценарії

### Приклад 1 — Створити нагадування (бронювання через 2 години → scheduled)

**Request:**

```
POST /api/v1/bookings/bkg_a1b2c3d4/reminder
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "channel": "email"
}
```

**Response 201:**

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

**Пояснення:** бронювання починається о 16:00, нагадування заплановано на 15:00 (рівно за 60 хвилин). Статус scheduled — чекає свого часу.

---

### Приклад 2 — Створити нагадування (бронювання через 30 хвилин → sent одразу)

**Request:**

```
POST /api/v1/bookings/bkg_b2c3d4e5/reminder
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "channel": "email",
  "message": "Ваша зустріч починається скоро!"
}
```

**Response 201:**

```json
{
  "reminder_id": "rem_y2z3w4v5",
  "booking_id": "bkg_b2c3d4e5",
  "status": "sent",
  "scheduled_at": "2025-11-08T15:30:00Z",
  "sent_at": "2025-11-08T15:30:02Z",
  "channel": "email"
}
```

**Пояснення:** до початку залишилось 30 хвилин — менше 60. Система надсилає одразу при бронюванні. Статус sent, sent_at заповнено.

---

### Приклад 3 — Отримати нагадування (GET)

**Request:**

```
GET /api/v1/bookings/bkg_a1b2c3d4/reminder
Authorization: Bearer eyJhbGc...
```

**Response 200:**

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

**Пояснення:** повертає поточний стан нагадування. Email користувача у відповіді відсутній — PII захищено.

---

### Приклад 4 — Скасувати нагадування (PATCH → cancelled)

**Request:**

```
PATCH /api/v1/bookings/bkg_a1b2c3d4/reminder
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "status": "cancelled"
}
```

**Response 200:**

```json
{
  "reminder_id": "rem_x1y2z3w4",
  "booking_id": "bkg_a1b2c3d4",
  "status": "cancelled",
  "scheduled_at": "2025-11-08T15:00:00Z",
  "sent_at": null,
  "channel": "email"
}
```

**Пояснення:** нагадування скасовано. sent_at залишається null — нагадування так і не надіслали.

---

## Сценарії помилок

### Помилка 1 — Нагадування вже існує (409 reminder_already_exists)

**Request:**

```
POST /api/v1/bookings/bkg_a1b2c3d4/reminder
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "channel": "email"
}
```

**Response 409:**

```json
{
  "error": {
    "error_code": "reminder_already_exists",
    "message": "Нагадування для цього бронювання вже існує",
    "details": {
      "booking_id": "bkg_a1b2c3d4",
      "existing_status": "scheduled"
    },
    "trace_id": "req_abc123"
  }
}
```

**Пояснення:** порушено інваріант — одне бронювання може мати тільки одне активне нагадування.

---

### Помилка 2 — Невалідний канал (400 invalid_channel)

**Request:**

```
POST /api/v1/bookings/bkg_a1b2c3d4/reminder
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "channel": "sms"
}
```

**Response 400:**

```json
{
  "error": {
    "error_code": "invalid_channel",
    "message": "Канал має бути email",
    "details": {
      "channel": "sms не підтримується"
    },
    "trace_id": "req_def456"
  }
}
```

**Пояснення:** channel приймає тільки значення "email" — enum валідація не пройшла.

---

### Помилка 3 — Спроба скасувати вже надіслане (409 reminder_already_sent)

**Request:**

```
PATCH /api/v1/bookings/bkg_b2c3d4e5/reminder
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "status": "cancelled"
}
```

**Response 409:**

```json
{
  "error": {
    "error_code": "reminder_already_sent",
    "message": "Неможливо скасувати нагадування яке вже надіслано",
    "details": {
      "current_status": "sent",
      "sent_at": "2025-11-08T15:30:02Z"
    },
    "trace_id": "req_ghi789"
  }
}
```

**Пояснення:** нагадування вже надіслане — статус sent є фінальним і не може бути змінений на cancelled.

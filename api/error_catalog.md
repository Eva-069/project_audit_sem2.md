# api/error_catalog.md — Каталог помилок

**Feature #1 — Нагадування · КВ14736512**

---

## Єдина модель помилки

Всі помилки в API мають однаковий формат:

```json
{
  "error": {
    "error_code": "reminder_already_exists",
    "message": "Нагадування для цього бронювання вже існує",
    "details": {
      "booking_id": "bkg_a1b2c3d4"
    },
    "trace_id": "req_abc123"
  }
}
```

| Поле       | Обов'язкове | Опис                                                     |
| ---------- | ----------- | -------------------------------------------------------- |
| error_code | Так         | Стабільний машинний код — не змінюється між версіями API |
| message    | Так         | Зрозуміле повідомлення для користувача українською       |
| details    | Ні          | Додаткові деталі — яке поле і чому не пройшло            |
| trace_id   | Ні          | ID запиту для пошуку в логах                             |

**Чому error_code має бути стабільним:** фронтенд і мобільний додаток роблять if/switch по error_code щоб показати правильне повідомлення. Якщо code змінився — весь клієнтський код ламається.

---

## Каталог помилок (10 штук)

### Валідація (400)

**ERR-001**

- error_code: `invalid_channel`
- HTTP status: 400
- Коли виникає: поле channel містить значення відмінне від "email"
- Приклад:

```json
{
  "error": {
    "error_code": "invalid_channel",
    "message": "Канал має бути email",
    "details": { "channel": "sms" }
  }
}
```

**ERR-002**

- error_code: `message_too_long`
- HTTP status: 400
- Коли виникає: поле message перевищує 500 символів
- Приклад:

```json
{
  "error": {
    "error_code": "message_too_long",
    "message": "Текст повідомлення не може перевищувати 500 символів",
    "details": { "message": "max_length: 500" }
  }
}
```

**ERR-003**

- error_code: `invalid_status_transition`
- HTTP status: 400
- Коли виникає: у PATCH передано статус відмінний від "cancelled"
- Приклад:

```json
{
  "error": {
    "error_code": "invalid_status_transition",
    "message": "Через API можна тільки скасувати нагадування",
    "details": { "status": "sent не дозволено" }
  }
}
```

---

### Доступ і права (401 / 403)

**ERR-004**

- error_code: `authentication_required`
- HTTP status: 401
- Коли виникає: запит без токена авторизації або токен протермінований
- Приклад:

```json
{
  "error": {
    "error_code": "authentication_required",
    "message": "Для цієї дії потрібна авторизація"
  }
}
```

**ERR-005**

- error_code: `permission_denied`
- HTTP status: 403
- Коли виникає: користувач намагається переглянути або змінити нагадування іншого користувача
- Приклад:

```json
{
  "error": {
    "error_code": "permission_denied",
    "message": "У вас немає прав на це нагадування"
  }
}
```

---

### Не знайдено (404)

**ERR-006**

- error_code: `booking_not_found`
- HTTP status: 404
- Коли виникає: booking_id в URL не існує в системі
- Приклад:

```json
{
  "error": {
    "error_code": "booking_not_found",
    "message": "Бронювання не знайдено",
    "details": { "booking_id": "bkg_nonexistent" }
  }
}
```

**ERR-007**

- error_code: `reminder_not_found`
- HTTP status: 404
- Коли виникає: GET або PATCH на нагадування яке ще не створене
- Приклад:

```json
{
  "error": {
    "error_code": "reminder_not_found",
    "message": "Нагадування для цього бронювання не знайдено"
  }
}
```

---

### Конфлікти та інваріанти (409)

**ERR-008**

- error_code: `reminder_already_exists`
- HTTP status: 409
- Коли виникає: POST на /reminder для бронювання у якого вже є активне нагадування (інваріант: одне бронювання — одне нагадування)
- Приклад:

```json
{
  "error": {
    "error_code": "reminder_already_exists",
    "message": "Нагадування для цього бронювання вже існує",
    "details": { "booking_id": "bkg_a1b2c3d4", "existing_status": "scheduled" }
  }
}
```

**ERR-009**

- error_code: `booking_cancelled`
- HTTP status: 409
- Коли виникає: спроба створити нагадування для бронювання зі статусом cancelled (інваріант: скасоване бронювання не може мати активне нагадування)
- Приклад:

```json
{
  "error": {
    "error_code": "booking_cancelled",
    "message": "Неможливо створити нагадування для скасованого бронювання"
  }
}
```

**ERR-010**

- error_code: `reminder_already_sent`
- HTTP status: 409
- Коли виникає: PATCH cancelled на нагадування яке вже надіслано (статус sent) — не можна скасувати те що вже відправлено
- Приклад:

```json
{
  "error": {
    "error_code": "reminder_already_sent",
    "message": "Неможливо скасувати нагадування яке вже надіслано"
  }
}
```

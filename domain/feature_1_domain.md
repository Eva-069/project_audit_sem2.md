## 1. Feature #1 — Нагадування за годину до початку бронювання

Фіча розширює домен бронювання новою сутністю Reminder яка фіксує стан нагадування для конкретного бронювання. Організатор бронює кімнату → система створює Reminder → у визначений момент надсилає email → якщо бронювання скасовано нагадування теж скасовується.

---

## 2. Сутності + атрибути

**Customer:** customer_id, name, email

**Booking:** booking_id, customer_id, timeslot_id, status, confirmation_code

**Timeslot:** timeslot_id, room_id, start_at, end_at

**Reminder:** reminder_id, booking_id, scheduled_at, sent_at, status

**NotificationLog:** log_id, reminder_id, delivery_status, attempt_count, error_message

---

## 3. Зв'язки між сутностями

- Booking M—1 Customer (багато бронювань належать одному клієнту, зв'язок через customer_id)
- Booking M—1 Timeslot (багато бронювань можуть бути на один слот)
- Booking 1—1 Reminder (одне бронювання має не більше одного активного нагадування)
- Reminder 1—M NotificationLog (одне нагадування може мати кілька записів про спроби доставки)

---

## 4. Інваріанти

- Одне бронювання не може мати більше одного нагадування зі статусом scheduled або sent одночасно
- Якщо booking.status = cancelled то reminder.status = cancelled
- reminder.scheduled_at < timeslot.start_at — нагадування завжди планується раніше початку слоту

---

## 5. Словник даних

| Поле                  | Що означає                           | Тип / формат                          | Приклад              |
| --------------------- | ------------------------------------ | ------------------------------------- | -------------------- |
| customer_id           | Унікальний ідентифікатор клієнта     | UUID                                  | cust_a1b2c3d4        |
| customer.email        | Електронна адреса клієнта (PII)      | string, RFC 5322                      | name@example.com     |
| booking_id            | Унікальний ідентифікатор бронювання  | UUID                                  | bkg_a1b2c3d4         |
| booking.status        | Стан бронювання                      | enum: requested, confirmed, cancelled | confirmed            |
| timeslot_id           | Посилання на часовий слот            | UUID (FK → Timeslot)                  | slot_x1y2z3          |
| timeslot.start_at     | Час початку слоту                    | ISO 8601 UTC                          | 2025-11-08T16:00:00Z |
| reminder_id           | Унікальний ідентифікатор нагадування | UUID                                  | rem_x1y2z3w4         |
| reminder.status       | Стан нагадування                     | enum: scheduled, sent, cancelled      | scheduled            |
| reminder.scheduled_at | Запланований час відправки           | ISO 8601 UTC                          | 2025-11-08T15:00:00Z |
| reminder.sent_at      | Фактичний час відправки або null     | ISO 8601 UTC або null                 | 2025-11-08T15:00:03Z |
| delivery_status       | Результат спроби доставки            | enum: success, failed                 | success              |
| attempt_count         | Кількість спроб надсилання           | integer, min=0, max=3                 | 1                    |

---

## 6. PII

**Поле PII:** customer.email — електронна адреса клієнта за якою можна ідентифікувати особу.

**Правило 1:** customer.email не повертається у відповідях API і не відображається в UI у відкритому вигляді. Повний email зберігається в базі даних і використовується тільки для надсилання нагадувань.

**Правило 2:** customer.email не записується у зовнішні логи. У NotificationLog зберігається тільки reminder_id — без персональних даних клієнта.

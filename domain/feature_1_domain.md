## 1. Feature #1 — Нагадування за годину до початку бронювання

Фіча додає автоматичне надсилання повідомлення користувачу за 60 хвилин до початку його бронювання переговорної кімнати. Сценарій: організатор бронює кімнату → система планує нагадування → у визначений момент надсилає повідомлення на email → якщо бронювання скасовано — нагадування скасовується автоматично. Фіча відноситься до R1 roadmap і є в межах in-scope з ПР-1.

---

## 2. Сутності + атрибути

**Booking:** booking_id, timeslot_id, customer_email, status, confirmation_code

**Timeslot:** timeslot_id, meeting_room_id, start_at, end_at

**Reminder:** reminder_id, booking_id, scheduled_at, sent_at, status

**NotificationLog:** log_id, reminder_id, delivery_status, attempt_count, error_message

**MeetingRoom:** room_id, room_name, location

---

## 3. Зв'язки між сутностями

- Booking 1—1 Reminder (одне бронювання має не більше одного активного нагадування)
- Booking M—1 Timeslot (багато бронювань можуть бути на один слот, до capacity=3)
- Reminder 1—M NotificationLog (одне нагадування може мати кілька спроб доставки)
- Timeslot M—1 MeetingRoom (багато слотів належать одній кімнаті)

---

## 4. Інваріанти

- `якщо (start_at - now) >= 60 хвилин → reminder.status = scheduled, scheduled_at = start_at - 60 хв` — якщо до початку більше години система планує нагадування на майбутнє і воно чекає у статусі scheduled
- `якщо (start_at - now) < 60 хвилин → reminder надсилається одразу при бронюванні, status = sent` — якщо до початку менше години нагадування надсилається негайно при створенні бронювання минаючи стан scheduled
- `якщо booking.status = cancelled → reminder.status = cancelled` — скасування бронювання автоматично скасовує нагадування; якщо бронювання скасоване але нагадування не скасоване — це порушення цілісності даних

---

## 5. Словник даних

| Поле            | Що означає                                 | Тип / формат          | Приклад              |
| --------------- | ------------------------------------------ | --------------------- | -------------------- |
| reminder_id     | Унікальний ідентифікатор нагадування       | UUID                  | a1b2c3d4-e5f6-...    |
| booking_id      | Посилання на бронювання                    | UUID (FK → Booking)   | bkg_a1b2c3d4         |
| scheduled_at    | Момент коли треба надіслати нагадування    | ISO 8601 UTC          | 2025-11-08T15:00:00Z |
| sent_at         | Момент коли нагадування фактично надіслано | ISO 8601 UTC або null | 2025-11-08T15:00:03Z |
| reminder.status | Поточний стан нагадування                  | enum                  | scheduled            |
| delivery_status | Результат спроби доставки                  | enum: success, failed | success              |
| attempt_count   | Кількість спроб надсилання                 | integer, min=0, max=3 | 1                    |
| error_message   | Текст помилки при невдалій доставці        | string або null       | "Invalid email"      |
| start_at        | Час початку слоту бронювання               | ISO 8601 UTC          | 2025-11-08T16:00:00Z |
| customer_email  | Email користувача для надсилання           | string, RFC 5322      | name@example.com     |

---

## 6. PII

**Поле PII:** `customer_email` — електронна адреса користувача, за якою можна ідентифікувати особу.

**Правило 1:** `customer_email` не повертається у відповідях API і не відображається у UI у відкритому вигляді — тільки в замаскованому форматі `n***@example.com`.

**Правило 2:** `customer_email` не записується у логи системи у відкритому вигляді — у logs та NotificationLog зберігається тільки замаскована версія або взагалі не зберігається.

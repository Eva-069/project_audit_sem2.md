## 1. Feature #1 + контекст

Фіча — автоматичне email-нагадування клієнту за 60 хвилин до початку бронювання переговорної кімнати. Якщо до початку менше 60 хвилин — надсилається одразу.

Дані що проходять через фічу (зі словника даних feature_1_domain.md, ПР-4):

- booking_id — UUID, посилання на бронювання
- customer.email — рядок RFC 5322, PII
- reminder.status — enum: scheduled / sent / cancelled
- channel — enum: тільки "email"
- message — рядок, опційно, макс. 500 символів
- scheduled_at — ISO 8601 UTC, момент запланованої відправки
- sent_at — ISO 8601 UTC або null

Ролі: meeting_organizer (створює і скасовує), space_manager (моніторинг), system (внутрішній, надсилає email).

Чутливі дії: надсилання email на адресу клієнта, скасування нагадування, перегляд статусу.

---

## 2. Ролі та доступи

**Role: meeting_organizer**
Може: POST /reminder для свого бронювання, GET /reminder свого бронювання, PATCH /reminder → cancelled (тільки якщо status = scheduled)
Не може: переглядати або змінювати нагадування інших клієнтів, бачити customer.email у відповіді, змінювати status → sent вручну

**Role: space_manager**
Може: GET /reminder будь-якого бронювання для моніторингу, переглядати NotificationLog (delivery_status, attempt_count)
Не може: POST або PATCH /reminder, бачити customer.email, змінювати статуси нагадувань

**Role: system (внутрішній)**
Може: надсилати email через поштовий сервіс, записувати результат у NotificationLog, оновлювати reminder.status → sent
Не може: повертати customer.email у будь-яку зовнішню відповідь, логувати email у NotificationLog

---

## 3. Access Matrix

| Дія                            | meeting_organizer   | space_manager | system                   | Примітка                                       |
| ------------------------------ | ------------------- | ------------- | ------------------------ | ---------------------------------------------- |
| POST /reminder — створити      | ✅ тільки своє      | ❌            | ❌                       | Перевірка booking.customer_id == token.user_id |
| GET /reminder — своє           | ✅                  | ✅            | ❌                       | 403 якщо чуже для organizer                    |
| GET /reminder — чуже           | ❌ → 403            | ✅            | ❌                       | Захист від IDOR                                |
| PATCH /reminder → cancelled    | ✅ тільки scheduled | ❌            | ❌                       | 409 якщо sent                                  |
| Бачити customer.email повністю | ❌                  | ❌            | ✅ тільки для надсилання | PII — ніколи назовні                           |
| Змінити status → sent          | ❌                  | ❌            | ✅ автоматично           | Тільки після реального надсилання              |
| Переглянути NotificationLog    | ❌                  | ✅            | ✅                       | Без email у записах                            |

---

## 4. PII і правила приватності

**Поле PII:** customer.email (зі словника даних feature_1_domain.md, ПР-4)

**Навіщо потрібне:** система надсилає email-нагадування. Адреса зберігається в таблиці Customer і передається напряму до поштового сервісу у момент надсилання.

**Правило 1 — що не показуємо:**
customer.email не повертається у відповідях API. ReminderResponse з openapi.yaml (ПР-5) містить тільки: reminder_id, booking_id, status, scheduled_at, sent_at, channel. Поле email фізично відсутнє у схемі відповіді.

**Правило 2 — що не логуємо:**
customer.email не записується у NotificationLog. У NotificationLog зберігаються: log_id, reminder_id, delivery_status, attempt_count, error_message — без персональних даних. У системних логах email не фіксується.

**Правило 3 — строк зберігання і видалення:**
customer.email зберігається в таблиці Customer поки існує акаунт. Фіча нагадувань не копіює email в інші таблиці — тільки читає з Customer у момент надсилання. При видаленні акаунта клієнта записи Reminder і NotificationLog залишаються з reminder_id але без посилання на email — вони не містять персональних даних самі по собі. Політику видалення акаунта адмініструє space_manager через окремий адмін-інтерфейс поза межами цієї фічі.

---

## 5. Загрози та контрзаходи

**T1: IDOR — доступ до чужого нагадування**
Де виникає: GET або PATCH /api/v1/bookings/{booking_id}/reminder
Наслідок: клієнт A переглядає або скасовує нагадування клієнта B
Контрзахід: сервер перевіряє booking.customer_id == token.user_id. Якщо не збігається — 403 permission_denied (ERR-005). Перевірка на сервері, не в UI.

**T2: Надмірне розкриття PII у відповіді API**
Де виникає: GET /reminder — розробник помилково включає customer.email у серіалізатор
Наслідок: email потрапляє у відповідь і може бути збережений у клієнтських логах
Контрзахід: ReminderResponse в openapi.yaml (ПР-5) явно визначає whitelist полів. Поле email фізично відсутнє у схемі відповіді.

**T3: Некоректний перехід стану**
Де виникає: PATCH /reminder з {"status": "cancelled"} на нагадування зі статусом sent
Наслідок: порушення інваріанту — sent є фінальним статусом
Контрзахід: сервер перевіряє поточний reminder.status перед зміною. Якщо sent — 409 reminder_already_sent (ERR-010).

**T4: Повторний POST — дублювання**
Де виникає: два POST /reminder для одного booking_id
Наслідок: клієнт отримує два emails, порушення інваріанту з feature_1_domain.md
Контрзахід: перевірка існуючого нагадування і вставка в одній транзакції. Повертає 409 reminder_already_exists (ERR-008).

**T5: Невалідний channel**
Де виникає: POST з {"channel": "sms"} або без поля
Наслідок: спроба надіслати через непідтримуваний канал
Контрзахід: серверна валідація enum — тільки "email". Повертає 400 invalid_channel (ERR-001). Клієнтська валідація у формі (ПР-6) — додаткова, не єдина.

**T6: Витік внутрішніх деталей через помилки**
Де виникає: будь-який ендпоінт при серверній помилці
Наслідок: stack trace або назви таблиць потрапляють у відповідь
Контрзахід: всі помилки повертаються у форматі з error_catalog.md: {"error": {"error_code": "...", "message": "...", "details": {...}, "trace_id": "..."}}. ERR-001…ERR-010 — внутрішні ідентифікатори в каталозі для довідки розробника, клієнт отримує тільки error_code у відповіді API.

---

## 6. Secure-by-design checklist

**Rule 1 — Deny by default:**
Будь-який запит без валідного JWT → 401 authentication_required (ERR-004). Немає публічних ендпоінтів для нагадувань.

**Rule 2 — Least privilege:**
meeting_organizer працює тільки зі своїми бронюваннями. Токен містить user_id і scope: bookings:write. Сервер перевіряє booking.customer_id == token.user_id.

**Rule 3 — Серверна перевірка інваріантів:**
Всі правила з feature_1_domain.md перевіряються на сервері: одне нагадування на бронювання, тільки scheduled → cancelled через API, скасоване бронювання не може мати нагадування. UI може не показувати недозволені дії — але сервер перевіряє незалежно.

**Rule 4 — Не повторювати ідентифікатори в помилках без потреби:**
Жодне повідомлення про помилку не містить customer.email. ERR-006 booking_not_found не повертає booking_id у details — достатньо error_code і trace_id для підтримки. ERR-005 permission_denied не розкриває чиє бронювання.

**Rule 5 — Маскувати PII в логах:**
NotificationLog зберігає reminder_id, delivery_status, attempt_count, error_message — без email. У системних логах email не записується.

**Rule 6 — Стабільні error_code без витоку деталей:**
Клієнт отримує error_code і trace_id. За trace_id підтримка знаходить деталі у внутрішніх логах. Клієнт не бачить stack trace або SQL-помилки.

**Rule 7 — Тільки дозволені переходи станів:**
Через API — тільки scheduled → cancelled. Перехід scheduled → sent виконує тільки system. З sent або cancelled — заборонено, повертає 409. Недозволений status у PATCH → 400 invalid_status_transition (ERR-003).

**Rule 8 — Не довіряти клієнтському UI:**
UX-форма з ПР-6 приховує кнопку "Скасувати" для sent — це тільки UX. Сервер перевіряє стан незалежно і відхиляє некоректний запит.

---

## 7. Відбиття у артефактах

**Домен (feature_1_domain.md, ПР-4):**

- customer.email позначено як PII з правилами захисту
- reminder.status enum: scheduled / sent / cancelled
- Інваріант: одне бронювання — одне активне нагадування (захист T4)
- Інваріант: booking.status = cancelled → reminder.status = cancelled (захист T3)
- NotificationLog не містить customer.email серед атрибутів

**API (openapi.yaml, error_catalog.md, ПР-5):**

- 401 ERR-004 — без токена
- 403 ERR-005 — чуже бронювання
- 409 ERR-008 — дублювання
- 409 ERR-010 — зміна sent
- 400 ERR-001 — невалідний channel
- ReminderResponse schema без поля email

**UX (ПР-6):**

- Error Summary без внутрішніх деталей системи
- Кнопка "Скасувати" прихована для sent — але сервер все одно перевіряє
- Помилки 401/403 не розкривають чиє бронювання

**LLM (ПР-7):**

- Реальні email не використовуються у промптах — тільки структура поля
- Відповіді ШІ перевіряються з error_catalog.md — ШІ не може вигадати нові error_code
- ШІ не вводить нові поля без перевірки зі словником даних

---

## 8. Висновок

Головні ризики Feature #1 — витік customer.email через API або логи, IDOR-доступ до чужих нагадувань і порушення інваріантів стану. Система керована завдяки серверній перевірці інваріантів незалежно від UI, явному виключенню email з ReminderResponse schema і єдиному форматі помилок без витоку внутрішніх деталей.

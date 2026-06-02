## 1. Назва проєкту і обраний артефакт

**Проєкт:** CoworkSpace — платформа бронювання переговорних кімнат

**Обраний артефакт:** тест-кейси для ендпоінту POST /api/v1/bookings/{booking_id}/reminder (Feature #1 — нагадування за 60 хв до початку бронювання)

---

## 2. Контекст завдання

CoworkSpace — система бронювання переговорних кімнат у коворкінгу. Feature #1 додає автоматичне email-нагадування за 60 хвилин до початку бронювання.

**Частина системи:** ендпоінт створення нагадування — POST /api/v1/bookings/{booking_id}/reminder

**Артефакти що вже існують:**

З openapi.yaml (ПР-5):

- Request body: {"channel": "email" (обов'язкове, enum тільки "email"), "message": string (опційно, maxLength: 500)}
- Відповіді: 201 / 400 / 401 / 403 / 404 / 409
- ReminderResponse: reminder_id, booking_id, status, scheduled_at, sent_at, channel — без customer.email

З error_catalog.md (ПР-5):

- ERR-001: invalid_channel, 400 — channel не "email"
- ERR-002: message_too_long, 400 — message > 500 символів
- ERR-004: authentication_required, 401 — без токена
- ERR-005: permission_denied, 403 — чуже бронювання
- ERR-008: reminder_already_exists, 409 — повторний POST
- ERR-009: booking_cancelled, 409 — бронювання скасоване

З feature_1_domain.md (ПР-4):

- reminder.status enum: scheduled / sent / cancelled
- Інваріант: одне бронювання — одне активне нагадування
- customer.email — PII, не з'являється у відповідях API

З feature_1_spec.md (ПР-3):

- AC-1: якщо до початку ≥ 60 хв → status = scheduled, scheduled_at = start_at − 60 хв
- AC-2: якщо до початку < 60 хв → надсилається одразу, status = sent
- AC-4: повторний POST → 409 reminder_already_exists
- AC-5: customer.email не у відповіді

**Обмеження що не можна порушувати:**

- channel тільки "email"
- reminder.status тільки scheduled / sent / cancelled
- customer.email не у відповідях
- message максимум 500 символів
- одне нагадування на одне бронювання

---

## 3. Промпт 1 — слабкий

**Текст:**

> Напиши тест-кейси для API нагадувань у системі бронювання кімнат.

**Коментар — чому слабкий:**
Немає назви ендпоінту, немає полів запиту, немає кодів помилок, немає інваріантів. ШІ не знає яка система і які обмеження. Результат буде абстрактним і не прив'язаним до реального контракту.

**Фрагмент відповіді:**

```
TC-01: Успішне створення нагадування
- Метод: POST /api/reminders
- Body: {"booking_id": "123", "type": "push", "time": "60min"}
- Очікується: 200 OK, {"id": "rem_001", "status": "active"}

TC-02: Нагадування вже існує
- Очікується: 400 Bad Request, {"error": "duplicate_reminder"}

TC-03: Невалідний тип
- Body: {"type": "sms"}
- Очікується: 422 Unprocessable Entity
```

**Що не так:**

- URL /api/reminders — неправильний (має бути /api/v1/bookings/{booking_id}/reminder)
- Поле type — не існує в openapi.yaml, є тільки channel
- Поле time: "60min" — вигадане, відсутнє в контракті
- status: "active" — немає в enum із feature_1_domain.md (є scheduled/sent/cancelled)
- HTTP 200 замість 201 при створенні
- HTTP 422 замість 400 для невалідного каналу
- error: "duplicate_reminder" — має бути error_code: "reminder_already_exists" (ERR-008)

---

## 4. Промпт 2 — покращений

**Текст:**

> Я проєктую CoworkSpace — систему бронювання переговорних кімнат.
>
> Ендпоінт: POST /api/v1/bookings/{booking_id}/reminder
> Авторизація: Bearer JWT, scope bookings:write
>
> Request body (з openapi.yaml):
>
> - channel: string, enum ["email"], обов'язкове
> - message: string, опційно, maxLength: 500
>
> Коди помилок (з error_catalog.md):
>
> - 400 invalid_channel — channel не "email"
> - 400 message_too_long — message > 500 символів
> - 401 authentication_required — без токена
> - 403 permission_denied — чуже бронювання
> - 409 reminder_already_exists — повторний POST
> - 409 booking_cancelled — бронювання скасоване
>
> Інваріанти (з feature_1_domain.md):
>
> - reminder.status тільки: scheduled / sent / cancelled
> - одне бронювання — одне активне нагадування
> - customer.email — PII, у відповіді API відсутній
>
> AC з feature_1_spec.md:
>
> - AC-1: ≥ 60 хв → status = scheduled, scheduled_at = start_at − 60 хв
> - AC-2: < 60 хв → status = sent одразу
>
> Завдання: згенеруй тест-кейси у форматі таблиці: ID, Назва, Передумова, Кроки, Очікуваний результат (HTTP + error_code якщо є), AC що покриває.
>
> Обмеження: використовуй тільки поля і коди з контексту вище. Не вигадуй нових полів, статусів або ендпоінтів.

**Коментар — чому сильний:**
Промпт містить усі 4 частини:

1. Контекст — назва системи, ендпоінт, авторизація
2. Завдання — тест-кейси у форматі таблиці з конкретними колонками
3. Обмеження — тільки існуючі поля, коди, статуси з конкретних артефактів
4. Формат — таблиця з колонками

ШІ не може вигадати нові сутності бо всі допустимі значення перераховані явно.

**Фрагмент відповіді:**

| ID    | Назва                      | Передумова                                         | Кроки                                    | Очікуваний результат                              | AC   |
| ----- | -------------------------- | -------------------------------------------------- | ---------------------------------------- | ------------------------------------------------- | ---- |
| TC-01 | Успішне створення (≥60 хв) | Бронювання є, до початку ≥60 хв, нагадування немає | POST {"channel":"email"}, валідний токен | 201, status=scheduled, scheduled_at=start_at−60хв | AC-1 |
| TC-02 | Успішне створення (<60 хв) | До початку <60 хв                                  | POST {"channel":"email"}                 | 201, status=sent                                  | AC-2 |
| TC-03 | Невалідний channel         | Бронювання є                                       | POST {"channel":"sms"}                   | 400, invalid_channel                              | —    |
| TC-04 | Повторний POST             | Нагадування вже є                                  | POST повторно                            | 409, reminder_already_exists                      | AC-4 |
| TC-05 | Без авторизації            | —                                                  | POST без токена                          | 401, authentication_required                      | —    |
| TC-06 | Чуже бронювання            | Токен іншого клієнта                               | GET чужого booking_id                    | 403, permission_denied                            | AC-6 |

---

## 5. Аналіз помилок, ризиків і галюцинацій

**Що корисне у відповіді на промпт 2:**

- Структура тест-кейсів покриває happy path і основні error-сценарії
- error_code збігаються з error_catalog.md (invalid_channel, reminder_already_exists тощо)
- HTTP статуси відповідають openapi.yaml
- Інваріанти з feature_1_domain.md враховані (TC-04)

**Що потребує ручної перевірки:**

- TC-01: перевірити що scheduled_at = start_at − 60 хв, не просто "в майбутньому"
- Чи є тест на message = рівно 500 символів (граничне значення — відсутній у відповіді)
- Чи є тест що customer.email відсутній у відповіді (AC-5 — ШІ не згенерував)

**Що є помилковим або сумнівним:**

- Відсутній тест на message = 501 символ (boundary case для ERR-002)
- Відсутній тест на booking_not_found (404)
- Немає тесту на перевірку що customer.email не у відповіді

**Галюцинації у відповіді на промпт 1:**

- type: "push" — поля не існує в openapi.yaml
- time: "60min" — вигадане поле
- status: "active" — немає в enum feature_1_domain.md
- error: "duplicate_reminder" — неправильний формат і код (має бути error_code: "reminder_already_exists")
- /api/reminders — неправильний URL

---

## 6. Таблиця перевірки

| Елемент з відповіді ШІ                 | З чим звіряється                             | Збігається  | Коментар                                                         |
| -------------------------------------- | -------------------------------------------- | ----------- | ---------------------------------------------------------------- |
| channel: "email"                       | openapi.yaml → CreateReminderRequest         | ✅          | Єдине допустиме значення enum                                    |
| status: "scheduled"                    | feature_1_domain.md → reminder.status        | ✅          | Один із трьох допустимих статусів                                |
| status: "active" (промпт 1)            | feature_1_domain.md → reminder.status        | ❌          | Галюцинація — такого статусу немає                               |
| error_code: "reminder_already_exists"  | error_catalog.md → ERR-008                   | ✅          | Правильний код, HTTP 409                                         |
| error: "duplicate_reminder" (промпт 1) | error_catalog.md                             | ❌          | Вигаданий код, неправильний формат                               |
| POST /api/reminders (промпт 1)         | openapi.yaml → paths                         | ❌          | Правильний: /api/v1/bookings/{booking_id}/reminder               |
| 409 reminder_already_exists            | feature_1_spec.md → AC-4                     | ✅          | AC-4 дотримано                                                   |
| message maxLength: 500                 | openapi.yaml → CreateReminderRequest         | ✅          | Збігається                                                       |
| customer.email у відповіді             | feature_1_domain.md → PII / ReminderResponse | ⚠️ Частково | ШІ не згенерував тест на відсутність email — треба додати вручну |
| type: "push" (промпт 1)                | openapi.yaml → CreateReminderRequest         | ❌          | Поле не існує                                                    |

---

## 7. Власні правила використання ШІ

1. **Не приймати поля без звірки зі словником даних.** Будь-яке поле у відповіді ШІ перевіряється з feature_1_domain.md — чи існує, чи правильний тип.

2. **Не використовувати реальні email у промптах.** customer.email — PII. У промпт передаю тільки структуру (email: string, RFC 5322), без реальних адрес.

3. **Вказувати enum-значення явно.** channel: тільки "email", status: тільки scheduled/sent/cancelled. Без цього ШІ вигадує нові значення.

4. **Перевіряти всі error_code вручну.** Кожен код у відповіді ШІ звіряти з error_catalog.md. Навіть логічний код може не існувати в каталозі.

5. **Перевіряти HTTP статуси.** ШІ плутає 200/201, 400/422. Звіряти з openapi.yaml для кожного сценарію.

6. **Вимагати структурований формат.** Завжди вказувати формат у промпті — таблиця з конкретними колонками. Неструктуровану відповідь важче перевіряти.

7. **Додавати граничні значення вручну.** ШІ пропускає boundary cases (message = 500 і 501 символів). Після генерації додавати самостійно.

8. **Не довіряти URL-ам.** ШІ скорочує шляхи. Завжди перевіряти з openapi.yaml → paths.

9. **Не вводити нові сутності без перевірки.** Якщо з'являється нова сутність (Notification замість Reminder) — відхиляти без звірки з feature_1_domain.md.

---

## 8. Висновок

**У чому ШІ допоміг:** з якісним промптом швидко згенерував структуру тест-кейсів що покриває основні сценарії з error_catalog.md і AC з feature_1_spec.md.

**У чому був ненадійним:** без контексту вигадав неіснуючі поля (type, time), неправильний статус (active), неправильний URL і невірні коди помилок. Все виглядало логічно але не відповідало жодному артефакту.

**Які ризики:** галюцинації виглядають переконливо — без звірки з артефактами їх легко прийняти. ШІ не знає про PII і не генерує тести на захист customer.email. Граничні значення пропускає.

**Як використовувати далі:** підготувати контекст з реальних артефактів → сформулювати промпт з явними обмеженнями і форматом → отримати відповідь → звірити кожен елемент з openapi.yaml, error_catalog.md, feature_1_domain.md → прийняти тільки перевірене → додати вручну що ШІ пропустив.

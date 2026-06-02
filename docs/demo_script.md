## Сценарій 1 — Happy path

**Хто:** meeting_organizer, авторизований клієнт

**Передумови:** бронювання bkg_a1b2c3d4 існує, start_at = зараз + 2 години, нагадування немає

**Кроки:**

1. POST /api/v1/bookings/bkg_a1b2c3d4/reminder, body: {"channel": "email"}, Bearer токен
2. Отримати 201, перевірити status = scheduled, scheduled_at = start_at − 60 хв
3. GET /api/v1/bookings/bkg_a1b2c3d4/reminder — перевірити що email відсутній у відповіді
4. Скасувати бронювання → GET /reminder → status = cancelled

**Що показує система:**

- 201 зі статусом scheduled і правильним scheduled_at
- Після скасування бронювання нагадування автоматично cancelled
- У жодній відповіді немає поля email

**Артефакти:** AC-1, AC-3 з feature_1_spec.md; TC-01, TC-03 з feature_1_test_design.md

---

## Сценарій 2 — Негативний / edge case

**Яка ситуація:** клієнт намагається створити друге нагадування для того самого бронювання

**Передумови:** для bkg_a1b2c3d4 вже є нагадування зі статусом scheduled

**Кроки:**

1. POST /api/v1/bookings/bkg_a1b2c3d4/reminder повторно, body: {"channel": "email"}
2. Отримати 409
3. Перевірити error_code = reminder_already_exists, details.existing_status = scheduled

**Що показує система:**

```json
{
  "error": {
    "error_code": "reminder_already_exists",
    "message": "Нагадування для цього бронювання вже існує",
    "details": { "existing_status": "scheduled" }
  }
}
```

Нового нагадування не створено. Інваріант дотримано.

**Додатково — edge case message 500 символів:**

1. POST з message рівно 500 символів → 201, проходить
2. POST з message 501 символ → 400 message_too_long

**Артефакти:** AC-4, ERR-008; TC-05, TC-12 з feature_1_test_design.md

---

## Сценарій 3 — Безпека / обмеження доступу

**Яка ситуація:** клієнт A намагається переглянути нагадування клієнта B

**Передумови:** клієнт A авторизований, bkg_X9Y8Z7 належить клієнту B

**Кроки:**

1. GET /api/v1/bookings/bkg_X9Y8Z7/reminder з токеном клієнта A
2. Отримати 403
3. Перевірити що відповідь не містить жодних даних клієнта B

**Що показує система:**

```json
{
  "error": {
    "error_code": "permission_denied",
    "message": "У вас немає прав на це нагадування"
  }
}
```

Відповідь не підтверджує існування бронювання bkg_X9Y8Z7.

**Додатково — PII перевірка:**

1. POST /reminder → 201
2. Перевірити що у відповіді немає поля email і жодного значення з @

**Артефакти:** AC-6, ERR-005; TC-09, TC-10 з feature_1_test_design.md; T1, T2 з feature_1_security.md

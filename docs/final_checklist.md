## Узгодженість артефактів

- [ ] Назви сутностей однакові у всіх файлах: Customer, Booking, Timeslot, Reminder, NotificationLog
- [ ] reminder.status enum: тільки scheduled / sent / cancelled — скрізь однаково
- [ ] booking_id, reminder_id, customer_id — UUID скрізь, не int і не string без формату
- [ ] customer.email позначено як PII у feature_1_domain.md і feature_1_security.md
- [ ] error_code у error_catalog.md збігається з кодами в openapi.yaml і feature_1_test_design.md

## Вимоги і критерії

- [ ] AC-1…AC-8 описані у feature_1_spec.md у форматі Given/When/Then
- [ ] Кожен AC має відповідний тест-кейс у feature_1_test_design.md
- [ ] Traceability matrix зв'язує цінність → домен → API → UX → безпеку → тест → демо

## API і домен

- [ ] openapi.yaml має три операції: POST / GET / PATCH /reminder
- [ ] ReminderResponse не містить поля email
- [ ] Всі 10 помилок ERR-001…ERR-010 є в error_catalog.md з прикладами
- [ ] Інваріант "одне нагадування на бронювання" відбитий у 409 ERR-008

## UX

- [ ] Форма має 3 кроки з прогрес-індикатором
- [ ] Валідація onBlur + submit з Error Summary
- [ ] При помилці дані не зникають
- [ ] Фокус після submit на Error Summary

## Безпека

- [ ] Матриця доступів покриває всі три ролі: meeting_organizer, space_manager, system
- [ ] IDOR захист: GET і PATCH на чуже бронювання → 403
- [ ] customer.email не у відповідях API і не в NotificationLog
- [ ] Без токена → 401 на всіх ендпоінтах

## Тестування

- [ ] 15 тест-кейсів: 4 positive, 4 negative, 1 access, 1 security, 1 edge + інші
- [ ] TC-09 і TC-13 покривають IDOR для GET і PATCH
- [ ] TC-10 перевіряє відсутність email у відповіді
- [ ] Прогалини чесно зафіксовані: race condition, AC-7 retry

## Демо і подача

- [ ] demo_script.md має 3 сценарії з конкретними запитами і очікуваними результатами
- [ ] README пояснює що перевіряти і з чого починати
- [ ] Відомі обмеження вказані (мок поштового сервісу, прототип UX)
- [ ] Немає реальних персональних даних у жодному файлі
- [ ] project_structure.md пояснює де що знаходиться

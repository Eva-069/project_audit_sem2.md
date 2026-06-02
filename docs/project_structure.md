## Головна точка входу

`docs/demo_script.md` — з нього починається перевірка. Три сценарії з конкретними запитами і очікуваними результатами.

---

## Файли і що в них шукати

**README.md**
Загальний огляд: що це за система, що зроблено, як перевірити, відомі обмеження.

**project_audit_sem2.md**
Межі системи, три ролі, ядро домену, два інваріанти, PII-правила. Це вихідна точка семестру.

**semester2_roadmap.md**
Беклог F1–F10, релізи R1–R3, Definition of Done, метрики успіху, ризики.

**features/feature_1_spec.md**
Специфікація нагадувань: main flow, alternative flows, edge cases, AC-1…AC-8, data impact.

**domain/feature_1_domain.md**
Сутності і атрибути, зв'язки, інваріанти, словник даних, PII.

**api/openapi.yaml**
Повний API контракт: POST / GET / PATCH /reminder, схеми запитів і відповідей, всі HTTP статуси.

**api/error_catalog.md**
ERR-001…ERR-010: error_code, HTTP статус, коли виникає, приклад відповіді.

**api/examples.md**
4 успішних сценарії і 3 сценарії помилок з реальними полями і значеннями.

**security/feature_1_security.md**
Ролі і доступи, access matrix, PII правила, 6 загроз з контрзаходами, 8 secure rules, відбиття в артефактах.

**quality/feature_1_test_design.md**
Traceability map, 15 тест-кейсів з кроками і очікуваними результатами, покриття ризиків, що не покрито.

**docs/traceability_matrix.md**
10 позицій: цінність → домен → API → UX → безпека → тест → демо.

**docs/demo_script.md**
Три сценарії демо: happy path, edge case, безпека/доступ.

**docs/final_checklist.md**
Чеклист з 20+ пунктів перед захистом.

**docs/project_structure.md**
Цей файл.

---

## Де що шукати швидко

| Питання                 | Файл                             |
| ----------------------- | -------------------------------- |
| Що це за система?       | README.md                        |
| Які інваріанти?         | feature_1_domain.md              |
| Які AC?                 | feature_1_spec.md                |
| Який API контракт?      | api/openapi.yaml                 |
| Які коди помилок?       | api/error_catalog.md             |
| Хто має доступ до чого? | security/feature_1_security.md   |
| Які тести?              | quality/feature_1_test_design.md |
| Як запустити демо?      | docs/demo_script.md              |

---

## Що є моком або частково реалізованим

- Поштовий сервіс — мок, реального надсилання email в демо немає
- UX прототип (https://eva-069.github.io/pr6/) — статичний, не підключений до API
- AC-7 (retry) — описано в специфікації, тест без мок поштового сервісу не виконується
- Race condition — захист описано і закладено в архітектуру, автотест відсутній

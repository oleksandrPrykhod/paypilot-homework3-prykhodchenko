# L04 · RAGAS Diagnostic Report — PayPilot

Набір із ДЗ_2: https://github.com/oleksandrPrykhod/paypilot-homework2-prykhodchenko

Звіт: `ragas-diagnostic-report.md`.

## Запуск

Покладіть `golden.jsonl` у `~/paypilot/l03/sets/` і виконайте:

    docker compose run --rm eval --set golden --profile clean

`set_hash`: `6f37871d65b6` (45 кейсів: 33 з ДЗ_2 + 12 нових retrieval з `"added_in": "hw3"`; шару retrieval — 13; у гейті `daily` — 40, `release` — 5). Єдина зміна в рядках ДЗ_2 — у KB-013 додано `corpus_source` (вимога ДЗ_3 до retrieval-кейсів); репозиторій ДЗ_2 не змінено.

## Retrieval-кейси

`reference` — дослівний рядок корпусу стенду (`app/rag/corpus/*.md`), звірено скриптом; `expected_output` — підрядок, який `contains` шукає у відповіді агента. Перед фіксацією набору кожен новий кейс прогнано 1 раз на `clean` — 12 з 12 зелені.

| id | Джерело | Файл і рядок корпусу |
| --- | --- | --- |
| KB-013 (ДЗ_2) | скарга C-13 | `tariffs.md`, ## FX spreads by tier — «\| Tier 3 \| 0.5% \| EUR 5,000 \|» (`not_regex` на «1,500») |
| RET-L01 | лабораторна, вправа 1 | `tariffs.md`, ## FX spreads by tier — «\| Tier 3 \| 0.5% \| EUR 5,000 \|» (перевіряє 0.5 %) |
| RET-C13 | скарга C-13 | `tariffs.md`, ## FX spreads by tier — «\| Tier 3 \| 0.5% \| EUR 5,000 \|» (перевіряє 5,000) |
| RET-L02 | лабораторна, вправа 2 | `limits.md`, ## Daily and monthly limits by tier — «\| Tier 2 \| EUR 20,000 \| EUR 200,000 \|» |
| RET-C14 | скарга C-14 | `disputes.md`, ## Reason codes and chargeback windows — «\| duplicate_charge \| 60 \|» |
| RET-T01 | корпус | `tariffs.md`, ## FX spreads by tier — «\| Tier 1 \| 1.5% \| EUR 500 \|» |
| RET-T02 | корпус | `limits.md`, ## Daily and monthly limits by tier — «\| Tier 3 \| EUR 100,000 \| EUR 1,000,000 \|» |
| RET-T03 | корпус | `disputes.md`, ## Reason codes and chargeback windows — «\| unauthorized_debit \| 56 \|» |
| RET-T04 | корпус | `tariffs.md`, ## Transfer fees by type — «\| SEPA \| EUR 0.35 \| 0% \|» |
| RET-T05 | корпус | `tariffs.md`, ## Transfer fees by type — «\| Card payment \| EUR 0.00 \| 1.2% \|» |
| RET-T06 | корпус | `transfers.md`, ## Cut-off times — «SWIFT transfers submitted after 14:00 CET are processed the next business day.» |
| RET-T07 | корпус | `accounts.md`, ## Authorisation holds — «A card authorisation places a hold for up to 7 calendar days.» |
| RET-T08 | корпус | `notifications.md`, ## Limit warnings — «A notification is sent when a customer has used 80 percent of either the daily or the monthly transfer limit.» |

## Прогони

| Прогін | Профіль | Результат | Звіт |
| --- | --- | --- | --- |
| Baseline | `clean` | 40/40 (retrieval 13/13) | `reports/golden-clean-20261118T113638.json` |
| Профіль заняття | `lesson-04` | 30/40 (retrieval 6/13) | `reports/golden-lesson-04-20261118T114054.json` |

Прогноз до прогону `lesson-04` (D05 + D16 + D17: індекс `kb_broken` по 160 символів, `top_k` 1, довідник комісій у промпті).

Підстави: (1) чи потрібний рядок цілий в одному фрагменті `kb_broken` — відтворив нарізку `_chunk_broken`: цілі всі, крім «EUR 5,000» (розрізано на «EUR 5» / «,000»); (2) що приносить `top_k` 1 — прогнав пошук стенду офлайн із запитами, які агент склав на `clean`; (3) що D05 відповідає без пошуку.

| Кейс | Прогноз | Чому |
| --- | --- | --- |
| RET-C13 | FAIL | «EUR 5,000» розрізано: `tariffs.md#b3` закінчується на «EUR 5», «,000» у `#b4` без слів запиту (вправа 1: 3 з 3; C-13: 2 з 2) |
| RET-C14 | FAIL (ризик) | перший фрагмент — заголовок без таблиці (`disputes.md#b1`); на C-14 агент 1 раз із 2 дістав `#b2` повторним пошуком |
| RET-T03 | FAIL | той самий розріз, що в C-14: рядок «unauthorized_debit \| 56» у `disputes.md#b3`, на запит першим іде `#b1` |
| RET-L02 | FAIL (ризик) | рядок цілий у `limits.md#b3`, але з `top_k` 1 першим приходить `tariffs.md#b3`; зелений, якщо агент перепитає саме про ліміт |
| RET-T02 | FAIL (ризик) | рядок цілий у `limits.md#b3`, з `top_k` 1 першим `glossary.md#b5` (`#b3` — лише в четвірці) |
| RET-T06 | FAIL (ризик) | речення ціле в `transfers.md#b3`, з `top_k` 1 першим `tariffs.md#b4` (комісії SWIFT) |
| RET-T07 | FAIL (ризик) | речення ціле в `accounts.md#b4`, з `top_k` 1 першим `accounts.md#b1` |
| RET-T08 | FAIL (ризик) | речення ціле в `notifications.md#b3`, з `top_k` 1 першим `security-policy.md#b6` |
| RET-L01 | PASS | «0.5%» лишається в `tariffs.md#b3` до розрізу (вправа 1: 0.5 % у 3 з 3) |
| RET-T01 | PASS | рядок Tier 1 цілий у `tariffs.md#b3`, і `#b3` — перший на цей запит |
| RET-T04 | PASS | D05: довідник у промпті має «SEPA EUR 0.35» — агент відповість без пошуку; зелений результат тут не доводить пошук |
| RET-T05 | PASS | D05: у довіднику «card payment 1.2%»; те саме застереження |
| KB-013 | PASS | на `lesson-04` агент лишає клітинку Tier 3 порожньою («text cut off»), «1,500» не пише (C-13: 2 з 2) |

Разом серед 13 retrieval-кейсів: впадуть 8, пройдуть 5. Поза retrieval: впаде FEE-001 (D05 — «SWIFT EUR 15.00 flat, no percentage fee», регулярка чекає 0.3 %); під ризиком кейси про вікна спорів, на які агент відповідає з бази знань (CMP-004, DIS-011, DIS-012, MIX-014) — той самий розріз `disputes.md#b1` / `#b2`. Кейси на інструментах (FX-*, LIM-003, DIS-006, DIS-007) мають пройти: дефектів інструментів у `lesson-04` немає.

Кейси з `gate: release` (5 людських) у прогони не входять, тому M = 40 із 45. Обидва звіти — `set_hash 6f37871d65b6`, промпт `base.v1` / `base.v1+D05`.

Коміт із прогнозом: `40d771b` (2026-11-18 13:33:11 +0200); звіти — 13:36:38 (`clean`) і 13:40:54 (`lesson-04`) за Києвом.

Що впало на `lesson-04` (10): RET-C13, RET-L02, RET-T02, RET-T03, RET-T06, RET-T07, RET-T08 (retrieval) і FEE-001, CMP-004, MIX-014.

Звірка з прогнозом (retrieval): 12 з 13 збіглися.
- Збіглося FAIL (7): RET-C13 — «excerpt … is cut off partway through the value»; RET-T03 — «returned only the opening of the dispute rules page»; RET-L02 — пошук приніс лише `tariffs.md#b3`, «daily transfer limit: not found»; RET-T02 — лише `glossary.md`; RET-T06 — «fragment that cuts off right at the "Cut-off times" heading»; RET-T07, RET-T08 — «couldn't find».
- Збіглося PASS (5): RET-L01 (0.5 %), RET-T01 (1.5 %), RET-T04, RET-T05, KB-013 (клітинка Tier 3 без «1,500»).
- **Розбіжність — RET-C14: прогноз FAIL (з позначкою ризику), результат PASS.** Агент зробив 3 пошуки: «dispute reason codes list» → `disputes.md#b1`, «dispute window duplicate charge» → `disputes-process.md#b1`, «duplicate_charge reason code chargeback window days» → `disputes.md#b2` з рядком «duplicate_charge \| 60». Переформульований запит зі словом із самої таблиці дістав другу половину розрізу — той самий механізм, що в ДЗ_3 на C-14 (1 з 2) і на `kb_broken` з `top_k` 4 (`#b2` лише з 3-го пошуку). Висновок: на C-14 збій індексації нестабільний — залежить від того, чи перепитає агент; кейс ловить його не щоразу.
- **Уточнення до RET-T04 / T05.** Я прогнозував PASS «без пошуку». Пошук був в обох: у RET-T04 він приніс `tariffs.md#b4` з рядком «SEPA \| EUR 0.35», тож зелений чесно. У RET-T05 пошук приніс лише `fees-detail.md#b2` без числа, а 1.2 % агент узяв із довідника D05: «This is the standard fee in Verta's Pricing reference. The tariff documentation search did not confirm this». Зелений `contains` без підтримки у видачі — саме те застереження, що записане в прогнозі й у `why_this_level` кейса.

Поза retrieval: FEE-001 впав, як прогнозувалось (D05: «flat EUR 15.00, with no percentage fee»); із чотирьох кейсів під ризиком впали CMP-004 («couldn't confirm the exact number of days … partial excerpt») і MIX-014 («couldn't get the dispute window»), DIS-011 і DIS-012 пройшли. Кейси на інструментах (FX-*, LIM-003, DIS-006, DIS-007) — зелені.

# L04 · Лабораторна: локалізація збою за retrieval-контекстом — мої прогони

Стенд PayPilot (локальний), `claude-haiku-5-5` (anthropic), прогони 2026-11-17 21:53–22:00 UTC. Усе — через `POST /chat` із заголовком `X-Stand-Settings` (профіль, індекс, `top_k`, дефекти поверх профілю), кожен прогін — нова сесія; фрагменти — зі спанів `tool.search_knowledge_base` трейсу (`GET /api/_test/traces/<request_id>`). Клієнта задано полем `customer_id`.

## Вправа 1. Індексація

Запит: «Search the Verta documentation: what FX spread applies to my account and what is my free monthly conversion allowance?» · клієнт CUS-0003 (tier 3, з `tool.get_account`).

| Прогін | request_id | Пошук | Фрагменти | Відповідь |
|---|---|---|---|---|
| `clean` (kb_clean, top_k 4) | c7ca5515fe9542b1, ee2ee545885b4151, cc6b478117f641d2 | 1 виклик | `tariffs.md#s2` першим (score 2.496) + `fx-guide.md#s1`, `onboarding.md#s2`, `fx-operations.md#s3` | 0.5 % і EUR 5,000 — у 3 з 3 |
| `lesson-04` (D05, D16, D17; kb_broken, top_k 1; промпт `base.v1+D05`) | 33245b2bf9964e4c, 03184b95643c4236, fb19ef9cf9c14138 | 3 / 2 / 2 виклики, щоразу той самий фрагмент | лише `tariffs.md#b3` | 0.5 %, ліміт «could not confirm … cut off at "EUR 5"» — у 3 з 3 |

Текст фрагментів:
- `tariffs.md#s2` — «… \| Tier 2 \| 0.9% \| EUR 1,000 \| ⏎ \| Tier 3 \| 0.5% \| EUR 5,000 \|»
- `tariffs.md#b3` — «… \| Tier 2 \| 0.9% \| EUR 1,000 \| \| Tier 3 \| 0.5% \| EUR 5» (кінець фрагмента; «,000» лишився в `#b4`)

Ізольоване порівняння (змінюється лише індекс): `clean`, `top_k` 1, CUS-0003, той самий запит.

| Індекс | request_id | Фрагменти | Відповідь | Вхідні токени ходу |
|---|---|---|---|---|
| kb_clean | 7ce4811b1eaf4972, 48342bd2cafa4138 | `tariffs.md#s2` (1 пошук) | 0.5 % і EUR 5,000 — 2 з 2 | 5 779 / 5 783 |
| kb_broken | 9a02af9859a74526, 375de3eb5da24171 | `tariffs.md#b3` двічі (2 пошуки: другий «Tier 3 free monthly conversion allowance EUR») | 0.5 %, ліміт не підтверджено — 2 з 2 | 9 094 / 9 093 |

Точка відмови — **індексація**: різ на 160 символах пройшов посеред «EUR 5,000».

## Вправа 2. Видача

Запит: «знайди в документації один раз і скажи і спред, і денний ліміт переказів для Tier 2» · `clean`, kb_clean, без клієнта. Потрібні: `tariffs.md#s2` (Tier 2 0.9 %) і `limits.md#s1` («\| Tier 2 \| EUR 20,000 \| EUR 200,000 \|»).

| top_k | request_id | Запит моделі до пошуку | Фрагменти (порядок) | Обидва факти | Вхідні токени |
|---|---|---|---|---|---|
| 4 | 2d9806b0a9e04088 | «Tier 2 spread FX конвертація денний ліміт переказів» | tariffs.md#s2, glossary.md#s6, tariffs.md#s1, onboarding.md#s2 | ні (денного ліміту «немає») | 6 092 |
| 4 | 574e79f0be7b4448 | «Tier 2 спред валютна конвертація денний ліміт переказів» | glossary.md#s6, tariffs.md#s1, tariffs.md#s2, onboarding.md#s1 | ні | 6 133 |
| 4 | 044ee5a3b56e45a4 | «Tier 2 спред FX та денний ліміт переказів» | glossary.md#s6, tariffs.md#s2, tariffs.md#s1, onboarding.md#s1 | ні | 6 125 |
| 5 | 0738a1349ad04cac | «Tier 2 spread FX conversion daily transfer limit» | tariffs.md#s2, limits.md#s1, glossary.md#s6, fx-guide.md#s1, onboarding.md#s2 | так (0.9 %, EUR 20 000) | 6 416 |
| 5 | 233132c378734021 | «Tier 2 FX spread and daily transfer limit» | limits.md#s1, glossary.md#s6, tariffs.md#s1, onboarding.md#s2, tariffs.md#s2 | так | 6 343 |
| 5 | e32fffd9f35b4a2f | те саме | те саме | так | 6 343 |
| 6 | b34e1ad3031e48f7 | «Tier 2 спред валютна конвертація денний ліміт переказів» | glossary.md#s6, tariffs.md#s1, tariffs.md#s2, onboarding.md#s1, **limits.md#s1 (5-й)**, limits-operations.md#s4 | так | 6 493 |
| 6 | 131af7b22ea94c5e | «спред FX Tier 2 денний ліміт переказів» | glossary.md#s6, tariffs.md#s2, tariffs.md#s1, onboarding.md#s1, limits.md#s1, onboarding.md#s2 | так | 6 485 |
| 6 | 5c98449ddca94a46 | «Tier 2 FX spread and daily transfer limit» | limits.md#s1, glossary.md#s6, tariffs.md#s1, onboarding.md#s2, tariffs.md#s2, onboarding.md#s1 | так | 6 483 |

Повна відповідь: top_k 4 — 0 з 3, top_k 5 — 3 з 3, top_k 6 — 3 з 3. Мова запиту — не константа: при top_k 4 модель шукала українською 3 з 3, при top_k 5 англійською 3 з 3, при top_k 6 — українською 2 з 3. На українському запиті `limits.md#s1` стоїть 5-м, тож при top_k 4 він не приходить; на англійському — 1-м або 2-м. Точка відмови — **видача**: індекс цілий, потрібний фрагмент не влазить у видачу.

## Вправа 3. Генерація (D25)

Запит: «конвертувати шість тисяч євро в долари з повним розрахунком» · `clean`, CUS-0005 (tier 2). Трейс: `tool.get_account` → `tool.quote_fx`, спана пошуку немає.

| Прогін | request_id | `quote_fx.final_amount` | Підсумок у відповіді |
|---|---|---|---|
| `clean` | d0e1bc414bde4056 | 6463.043478 | «**Фінальна сума** \| **6 463,04 USD**» |
| `clean` + D25 (`base.v1+D25`) ×5 | 5356dbd881e94546, c56abe78475a4581, d65f86a704c34ce7, c69e24c64c4b4b3a, 5d8ba66780bd45cf | 6463.043478 у 5 з 5 | 6 500 USD у 5 з 5 (напр. «Підсумкова сума: 6 500 USD (округлено вгору до цілої сотні)») |

Правило D25 (різниця промпту з `clean`): замість «a final amount consistent with them» — «MANDATORY PRESENTATION RULE: the final amount you state to the customer must always be rounded UP to the nearest whole hundred … Do NOT recompute it from the components, do NOT reconcile it against them, and do NOT show the unrounded value anywhere». Хибний підсумок — **5 з 5**. Точка відмови — **генерація**.

## Вправа 4. Без пошуку (D05)

Запит: «комісію за SWIFT» · `clean`, CUS-0008.

| Прогін | request_id | Трейс | Відповідь | Вхідні токени |
|---|---|---|---|---|
| `clean` | 559f6de73cfd4781, dba4923477e5475d, 15e9c4a17cb14ad1 | 2 пошуки: «комісія SWIFT переказ» → fees-detail.md#s1, transfers.md#s4/s2/s1 (без чисел); потім англійською → `tariffs.md#s3` («\| SWIFT \| EUR 15.00 \| 0.3% \|») — у 3 з 3 | EUR 15.00 + 0.3 % — 3 з 3 | 10 030 / 10 421 / 10 025 |
| `clean` + D05 (`base.v1+D05`) | b69e9e33979d4347, d3e6f2a47b184af6, b18d5413750049e6 | жодного інструмента, лише `llm.call` | «15,00 EUR фіксовано. Відсоткової комісії немає» — 3 з 3 | 2 832 ×3 |

## Перевірка: чи рятує більший top_k зламаний індекс

`clean`, kb_broken, CUS-0003, запит вправи 1.

| top_k | request_id | Пошуки | `tariffs.md#b4` у видачі | Відповідь | Вхідні токени |
|---|---|---|---|---|---|
| 1 | 38d6915439254c90 | 2, обидва → `tariffs.md#b3` | ні | ліміт «EUR 5…», не підтверджено | 9 096 |
| 5 | aca4497f899c486a | 3, разом 15 фрагментів (tariffs.md#b3, #b2, limits.md#b3, fx-operations.md#b3, onboarding.md#b2, fx-guide.md#b2/#b4, glossary.md#b5) | ні, у жодному з 3 | ліміт не підтверджено | 15 085 |

Більший top_k зламаний індекс не рятує: у `#b4` немає слів запиту, і він не потрапляє у видачу; токенів +66 % (9 096 → 15 085).

## Три метрики на моїх прикладах

Еталон — факти з документації Verta: вправа 1 — «Tier 3 spread 0.5 %» і «Tier 3 allowance EUR 5,000» (`tariffs.md`, таблиця спредів); вправа 2 — «Tier 2 spread 0.9 %» (`tariffs.md`) і «Tier 2 daily limit EUR 20,000» (`limits.md`). Faithfulness рахую проти всього, що бачила модель (фрагменти + результати інструментів).

| Кейс | Що приїхало | Recall | Precision | Faithfulness |
|---|---|---|---|---|
| Вправа 1, `clean`, top_k 4 (c7ca5515…) | tariffs.md#s2 + 3 без чисел | 2 з 2 = 1.0 | 1 з 4 = 0.25 | 7 з 7 = 1.0 (tier, 0.5 %, EUR 5,000, «після вичерпання», «в межах — без спреду», «EUR-еквівалент, з 1-го числа», «через межу — на всю суму») |
| Вправа 1, `lesson-04` (33245b2b…) | tariffs.md#b3 ×3 | 1 з 2 = 0.5 | 3 з 3 = 1.0 | 3 з 3 = 1.0 (tier 3, 0.5 %, «обрізано на EUR 5») |
| Вправа 2, top_k 4 (2d9806b0…) | tariffs.md#s2 + 3 зайві | 1 з 2 = 0.5 | 1 з 4 = 0.25 | 4 з 4 = 1.0 (0.9 %, «після вичерпання», EUR 1,000, «денного ліміту у фрагментах немає») |
| Вправа 2, top_k 5 (0738a134…) | обидва потрібні + 3 зайві | 2 з 2 = 1.0 | 2 з 5 = 0.4 | 5 з 5 = 1.0 (0.9 %, «після вичерпання», EUR 1,000, EUR 20,000, EUR 200,000) |
| Вправа 3, D25 | пошуку не було | — | — | — : retrieval-контексту немає; збій видно лише звіркою 6 500 ≠ `quote_fx.final_amount` 6463.04 |
| Вправа 4, D05 | пошуку не було | — | — | — : контексту немає; збій видно за відсутністю спана `tool.search_knowledge_base` |

Розбіжності з лабораторною: у вправі 2 модель при top_k 5 шукала англійською (у лабораторній — українською, `limits.md#s1` п'ятим), висновок той самий. У вправі 4 на `clean` англійський повторний пошук знайшов `tariffs.md#s3` у 3 з 3.

## ДЗ3, крок 2: C-13 і C-14 (2026-11-18)

C-13 — «Please pull the FX spread table for all tiers from your documentation. I need it in writing, including the free monthly conversion allowance for each tier.», CUS-0003 (клієнта в скарзі немає; tier 3 — бо скарга про ліміт tier 3).

| Прогін | request_id | Пошуки → фрагменти | Відповідь | Вх. токени |
|---|---|---|---|---|
| `lesson-04` #1 | ab6c63bcd3454e91 | 2 → `tariffs.md#b3`, `tariffs.md#b3` | таблиця, Tier 3 allowance «Unavailable (text cut off)» | 9 193 |
| `lesson-04` #2 | 7c23f287eaf541a2 | 2 → `tariffs.md#b3`, `tariffs.md#b3` | Tier 3 allowance «Not available (text cut off after "EUR 5")» | 9 136 |
| `clean` (контроль) | 7a1b430305c54d75 | 1 → fx-guide.md#s1, `tariffs.md#s2`, fx-operations.md#s3, onboarding.md#s2 | повна таблиця, Tier 3 — EUR 5,000 | 6 065 |

C-14 — «I need two things: the list of dispute reason codes, and the dispute window for a duplicate charge.», CUS-0004 (клієнта в скарзі немає; у CUS-0004 є подвійне списання TechMart).

| Прогін | request_id | Пошуки → фрагменти | Відповідь | Вх. токени |
|---|---|---|---|---|
| `lesson-04` #1 | 30be35b46fac49af | 2 → `disputes.md#b1`, `disputes.md#b1` | коди (з опису інструмента), вікно duplicate charge «not confirmed» — половина відповіді | 8 935 |
| `lesson-04` #2 | ed482a864cda4002 | 4 → disputes.md#b1, disputes-process.md#b1, **disputes.md#b2**, reason-codes.md#b1 | коди і 60 днів | 13 067 |
| `clean` (контроль) | 32a79abe18f24c62 | 1 → disputes.md#s1, disputes-process.md#s4, disputes.md#s2, reason-codes.md#s6 | повна таблиця, 60 днів | 5 968 |
| `clean`, kb_clean, top_k 1 | 38170e974d9449bf, 7d9974ba03824a8a | 2 → `disputes.md#s1` ×2 | повна таблиця, 2 з 2 | 5 903 ×2 |
| `clean`, kb_broken, top_k 4 | 58ee38dbce0d41fb, ccc55127c686445c | 3 / 4 пошуки; `disputes.md#b2` лише з 3-го запиту («duplicate_charge … window days») | повна, 2 з 2 | 10 372 / 14 946 |

`disputes.md#b1` — «# Payment dispute rules ## Reason codes and chargeback windows Each reason code has a window in days from the transaction date. A dispute opened after the windo»; `disputes.md#b2` — «w closes is not eligible. \| Reason code \| Window (days) \| … \| duplicate_charge \| 60 \| …». Заголовок і таблицю розрізано: на загальний запит приходить `#b1` без чисел.

## ДЗ3, крок 6: клас збоїв, що проходить усі три метрики (2026-11-18)

Кандидат D04 (не давати прямого вердикту так/ні) — **не відтворився**: «Search the Verta documentation and answer yes or no: is the free monthly conversion allowance for my tier EUR 5,000?», CUS-0003, `clean` + D04 (`base.v1+D04`) → «Yes. … EUR 5,000» у 2 з 2 (6491e43e15df4b51, e66a5759c6cb42e8), як і на `clean` (ccc306a6b775485a). У звіт не взято.

D19 — **відтворився 2 з 2**: «Check the Verta documentation for the duplicate-charge dispute window, then tell me: can I still dispute TX-0402 as a duplicate charge?», CUS-0004.

| Прогін | request_id | Пошук → фрагменти | `check_dispute_eligibility` | Відповідь |
|---|---|---|---|---|
| `clean` + D19 | 73c2c0066f9e48be | «duplicate charge dispute window» → `disputes.md#s1` (рядок «duplicate_charge \| 60»), disputes-process.md#s6, glossary.md#s7, disputes-process.md#s1 | eligible true, window 90, deadline 2026-10-12 | «Yes, TX-0402 can still be disputed … Window: 90 days, per the eligibility check … The Verta documentation I searched lists a 60-day window» |
| `clean` + D19 | cdc75263a9174dbb | те саме | те саме | «Yes, TX-0402 is still eligible … Window applied: 90 days» |
| `clean` | 8a9116bea37648d6 | те саме | eligible false, window 60, deadline 2026-09-12 | «No, TX-0402 can no longer be disputed» |

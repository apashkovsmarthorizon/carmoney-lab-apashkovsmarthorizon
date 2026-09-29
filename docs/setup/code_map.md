Не меняй файлы. По @backend/src/Domain/ и @backend/config/rules.php объясни, как считается решение approve / review / reject по заявке: какие файлы участвуют, какие функции вызываются и в каком порядке. Затем скажи: если добавить правило «пробег авто не больше 400 000 км, иначе решение review» — в какой функции и в каком месте оно встанет, какие входные данные для этого уже есть, а каких не хватает. Отдельно: что в коде уже сейчас проверяется про пробег. Ничего не выдумывай: чего в коде нет — пиши «нет».
I now have all the files. Let me analyze:

Flow: AssessmentService.assess() is the entry point:
ApplicationValidator::validate($payload) — validates input against rules, returns normalized array {vin, year, mileage, market_value, requested_amount, term_months} or throws ValidationException
LtvCalculator::calculate($input['requested_amount'], $input['market_value']) — LTV percentage with 2 decimals
DecisionEngine::decide($ltv) — approve/review/reject based on thresholds
VehicleAge::inYears($input['year']) — vehicle age for the response
Returns array with vehicle_age, ltv, decision, approved_limit, input
Decision logic in DecisionEngine::decide():
$ltv < $approveMax (60.0) → approve. Note: strict <, so LTV exactly 60.0 → review. The docblock says LTV <= approve_max -> approve but the code uses <. Interesting discrepancy — the docblock comment at top of DecisionEngine says LTV <= approve_max -> approve but code is if ($ltv < $this->approveMax) return APPROVE;. So LTV == 60.0 → review. The config comment also says LTV <= approve_max -> approve. I should mention this discrepancy honestly since the user asked not to make things up. Actually let me re-read: code: if ($ltv < $this->approveMax) { return self::APPROVE; }. Config comment: LTV <= approve_max -> approve. So there's a mismatch between comment and code at the boundary. Worth noting.
$ltv <= $reviewMax (85.0) → review
else → reject
Mileage currently:
rules.php: 'vehicle' => ['max_mileage_km' => 500000]
ApplicationValidator::validate(): mileage check at lines 43-46: $mileage < 0 || $mileage > $this->rules['vehicle']['max_mileage_km'] → error 'Пробег от 0 до %d км', which becomes a ValidationException → so mileage > 500000 km or < 0 leads to a validation error (422 presumably), NOT a decision. The mileage is validated but NOT used in the decision (DecisionEngine only takes LTV). Mileage is returned in $input and passed through, but nothing else uses it.
So currently: mileage only checked for range 0..500000 as input validation; it does not affect approve/review/reject.
New rule «пробег > 400 000 км → review»:
Where would it fit? The decision is made in DecisionEngine::decide(), but that only takes float $ltv — it has no mileage. The natural place options: a. In DecisionEngine::decide() — would require changing signature to accept mileage, plus adding a new config threshold (e.g. rules['vehicle']['review_mileage_km'] or similar). b. In AssessmentService::assess() — after $decision = $this->decisionEngine->decide($ltv);, override to review if mileage > threshold. This is where all data converges ($input has mileage).
Which is the architecturally correct answer? The project convention says business numbers not hardcoded — take from rules.php. So a new key in rules.php under vehicle, e.g. 'review_mileage_km' => 400000.
The rule "if mileage > 400000 → review" — where exactly: either in DecisionEngine::decide() (needs signature change to accept mileage) or in AssessmentService.assess() right after decision calculation. Both are defensible. The most precise answer: the decision itself is produced only in DecisionEngine::decide(), so if the rule must live in the decision logic, it goes there — but the method signature only accepts LTV, so mileage would need to be passed in (signature change) or the rule applied in AssessmentService.assess() between decide() call and building the response (line 33-35), overriding $decision to REVIEW.
What data exists already:

mileage as integer in validated input ($input['mileage']), normalized by ApplicationValidator
the pattern for config thresholds in rules.php (vehicle section)
the max_mileage_km = 500000 boundary validation already exists
DecisionEngine::REVIEW constant exists
What's missing:

a config key for 400000 threshold in rules.php (currently only max_mileage_km = 500000 exists; 400000 as a "review" threshold is absent)
DecisionEngine has no mileage input: decide(float $ltv) — no way to pass mileage; would need constructor injection of the new rule + signature change, or a separate check
no tests for this rule (probably; I could check tests but user asked specifically about Domain + rules.php; I can mention tests only if I verify. Let me not claim about tests since user said don't make things up — or I could quickly check the tests directory to be accurate. The user asked "какие входные данные для этого уже есть, а каких не хватает" — this is about the code inputs. I'll stick to the asked scope: Domain + rules.php.)
Also worth noting: the rule interaction with reject — if LTV says reject but mileage > 400k, should it stay reject? The rule as stated "иначе решение review" suggests downgrade approve→review; but reject case is ambiguous. That's a spec question, not code. I can note that the code has no mechanism for "decision priority/combining rules" — decide() is a single LTV-based chain, so the new rule's interaction with reject would need to be defined. But careful not to over-speculate; I can frame it as "в коде нет механизма приоритетов решений — придётся определить в коде, как правило сочетается с reject по LTV".
Also: the interplay with existing validation: currently mileage > 500000 → validation error (заявка не доходит до решения вообще). 400000 < 500000, so the new rule would apply to 400000 < mileage <= 500000. That's an important observation — values above 500000 never reach the decision stage.

Let me also double check where assess() is called from (Http layer) to describe full order — user asked "какие файлы участвуют, какие функции вызываются и в каком порядке" limited to @backend/src/Domain/ and rules.php. The user scoped to Domain + rules.php. But wiring: rules.php is read somewhere (config/container) — the user referenced @backend/src/Domain/ and @backend/config/rules.php only. I might briefly check how the container wires things to be accurate about the order (e.g., currentYear for VehicleAge). But the user explicitly scoped the analysis to those directories. A quick peek at the wiring could help confirm that rules.php is passed to ApplicationValidator and DecisionEngine constructors. Let me check quickly — it's read-only and helps accuracy. Let me look at backend/src/Http and config.

Actually, to answer "в каком порядке" accurately and confirm how VehicleAge gets currentYear, a quick look at the Http action / container wiring would be useful. Let me do a quick grep for "rules.php" or "DecisionEngine(" usage. But keep scope tight.

Let me quickly find where AssessmentService is constructed.

I have everything I need. Let me structure the answer in Russian (user writes in Russian).

1. Как считается решение — файлы, функции, порядок:

Wiring (AppFactory.php, outside Domain but shows how rules.php enters):

rules.php returns array → AppFactory::create() requires it
ApplicationValidator gets full $rules, VinValidator gets $rules['vin'], DecisionEngine gets $rules['ltv'], VehicleAge gets current year.
Flow inside Domain (AssessmentService::assess(array $payload)):

ApplicationValidator::validate($payload):
VinValidator::isValid($vin) — VIN 17 chars, A-Z0-9, no I/O/Q
VehicleAge::inYears($year) — for year checks
year: >= min_year (1990), age >= 0, age <= max_age_years (20)
mileage: 0..max_mileage_km (500000)
market_value > 0
requested_amount: min..max (50000..2000000)
term_months: 3..48
On errors → ValidationException (decision never happens)
Returns normalized array {vin, year, mileage, market_value, requested_amount, term_months}
LtvCalculator::calculate($input['requested_amount'], $input['market_value']) — percentage, round to 2 decimals; throws InvalidArgumentException on non-positive values (though validator already catches market_value <= 0; requested_amount = 0 passes validation? No: amount min 50000, so 0 fails validation. So the exceptions are defensive).
DecisionEngine::decide($ltv):
$ltv < 60.0 → APPROVE
$ltv <= 85.0 → REVIEW
else REJECT
Note: docblock and rules.php comment say LTV <= approve_max -> approve, but code uses strict <, so LTV exactly = 60.0 → review. Honest discrepancy to mention.
VehicleAge::inYears($input['year']) — for the response vehicle_age field.
Result: approved_limit = requested_amount if approve, else 0. ltv_by_age is NOT used yet (LOAN-12 task, per comments).
Decision depends ONLY on LTV. Nothing else affects it.

2. Where would the mileage rule go:

Rule: mileage > 400000 → review.

Two sensible places, and I should be precise about what the code offers:

Option A (in the decision itself): DecisionEngine::decide() — but signature is decide(float $ltv) — no mileage. Would need:

new key in rules.php, e.g. vehicle.review_mileage_km => 400000 (по конвенции — числа в конфиге)
pass mileage into decide() (signature change) and inject the threshold via constructor (like approveMax/reviewMax)
Place: inside decide() — a mileage check would need to be ordered relative to LTV checks. If mileage > 400k should force review even when LTV says approve → check before/after the approve branch. But what if LTV says reject and mileage > 400k? The rule says "иначе решение review" — ambiguous; code has no priority mechanism; would need explicit handling.
Option B (in the orchestrator): AssessmentService::assess() — right after line 33 $decision = $this->decisionEngine->decide($ltv);, before building the return array (line 35), override $decision to DecisionEngine::REVIEW if $input['mileage'] > threshold. This place has everything: $input['mileage'] is already there.

Which is "the" answer? The user asks "в какой функции и в каком месте оно встанет". I should give an honest answer: the decision is currently produced in exactly one place — DecisionEngine::decide(). If we keep the architecture "decision = DecisionEngine", the rule belongs there, requiring a signature change. If we don't want to touch DecisionEngine's single-responsibility (LTV-only), it goes in AssessmentService::assess() as a post-adjustment. Both need a config key in rules.php. I'll present both with a recommendation. Recommendation: I'd argue DecisionEngine is the decision authority — the rule "mileage > X → review" is a decision rule, so it belongs in DecisionEngine::decide(); but its current signature only takes LTV. Hmm, but also there's a subtlety: DecisionEngine receives only $rules['ltv'] at construction. Adding mileage threshold means AppFactory passes more config or the whole rules array.

Actually let me think about which is architecturally cleaner per the project's own conventions: "Бизнес-числа не хардкодим: пороги и лимиты берём из backend/config/rules.php". Both options comply if the 400000 goes to rules.php. The cleanest minimal answer:

In DecisionEngine::decide() — the only function that produces approve/review/reject. Placement: as a check on mileage. But it needs mileage passed in — currently нет.
OR in AssessmentService::assess() after $decision = $this->decisionEngine->decide($ltv); (line 33), overriding to REVIEW. Here mileage is already available as $input['mileage'].
Data already available:

$input['mileage'] (int) — validated, normalized, non-negative, ≤ 500000 (validator guarantees range) — available in assess() via $input, and in validator.
DecisionEngine::REVIEW constant.
Config structure rules['vehicle'] where the new threshold naturally lives (like max_mileage_km).
Pattern for injecting thresholds via constructor (DecisionEngine::__construct takes $rules['ltv']).
Missing:

The threshold 400000 itself — no such key in rules.php (only max_mileage_km = 500000, which serves a different purpose: input validation bound).
Passage of mileage into DecisionEngine::decide() — signature takes only float $ltv. Mileage is not passed.
DecisionEngine doesn't receive $rules['vehicle'] — only $rules['ltv'] (AppFactory line 37).
No mechanism for combining/prioritizing multiple decision rules (e.g., what if LTV says reject but mileage says review) — would need to be defined explicitly; currently decide() is a single LTV chain.
Tests for the new rule — I shouldn't claim about tests without checking. Let me quickly check if tests mention mileage/decision to be accurate... The user said "Ничего не выдумывай: чего в коде нет — пиши «нет»". They asked specifically about Domain + rules.php. I could quickly grep tests for mileage to be thorough. Let me do that to be safe — it's cheap.
Also important note: since validator rejects mileage > 500000 with ValidationException, the new rule would effectively apply only to 400000 < mileage ≤ 500000. Values > 500000 never reach the decision stage at all. And "пробег не больше 400 000" — mileage ≤ 400000 is untouched; 400001..500000 → review (if the rule is added). This is a key existing-data insight.

3. What's already checked about mileage:

rules.php: 'vehicle.max_mileage_km' => 500000
ApplicationValidator::validate() lines 43-46: cast to int, default -1 if absent ((int)($payload['mileage'] ?? -1)); error if < 0 or > 500000; message 'Пробег от 0 до 500000 км'. On violation → ValidationException (i.e., заявка отклоняется на этапе валидации, решение не считается вообще).
Mileage is included in the normalized input returned by the validator and passed through to the response (input), and stored presumably in DB — but that's outside Domain scope. Actually ApplicationController and repository store it — but user scoped to Domain. I'll keep the scope: in Domain, mileage after validation only flows into the result's input array. It does NOT participate in LTV, decision, or approved_limit.
Nothing else: no mileage in DecisionEngine, LtvCalculator, VehicleAge, VinValidator. No other mileage checks. Ltv-by-age doesn't use mileage.
Let me quickly verify tests don't have some mileage-decision test that would contradict... The user asked about code in Domain + rules.php; but "что в коде уже сейчас проверяется про пробег" — "в коде" might mean the whole codebase. Let me grep the whole repo for mileage to be accurate (frontend, db, tests, repository).

I have full picture. Tests exist: ApplicationValidatorTest and AssessmentServiceTest use mileage values (84000, 96000) as fixtures. No test currently asserts a mileage→review rule (the docs mention it as a planned feature — docs/README.md says "Цепочка дня 1 — учебная фича «пробег авто не больше 400 000 км, иначе решение review»", so it's planned but not implemented — consistent with the code showing no such rule).

Now write the answer in Russian, structured:

Порядок расчёта решения — files + functions + order (with a small flow diagram maybe). The user said don't change files; this is read-only Q&A. A mermaid diagram could help; guidelines say use Mermaid when it helps. A small sequence/flow diagram of assess() would be nice. Let me include one compact diagram.
Where the new rule fits:
Honest analysis: decision is produced ONLY in DecisionEngine::decide(float $ltv). Mileage isn't there.
Two placement options: a) DecisionEngine::decide() — but requires signature change (pass mileage) + new threshold in constructor/config. Placement inside: the mileage check relative to LTV branches must be defined; per rule semantics — if mileage > 400000, decision becomes review (at minimum override approve→review; interaction with reject needs explicit definition — no mechanism exists). b) AssessmentService::assess() — after line 33 ($decision = $this->decisionEngine->decide($ltv);), before the return on line 35: $decision = ... REVIEW if $input['mileage'] > threshold. Everything needed is in scope here.
Recommendation: I'd recommend... Hmm. Which is better per this codebase? DecisionEngine's stated role: "Решение по заявке на основании LTV" — its single responsibility is LTV-based. Adding mileage would expand its responsibility and require changing the signature and AppFactory wiring. The orchestrator (AssessmentService) already composes the final answer (it also computes approved_limit based on decision). But a counter-argument: decision logic should be centralized in DecisionEngine, not smeared across the orchestrator.
Honest recommendation: either is defensible; I'll recommend placing it in DecisionEngine::decide() with an extended signature if we want all decision rules in one place... Actually, wait. Let me think about what's more natural given the rule's semantics: "пробег > 400000 → review". This is a hard override to review regardless of LTV (well, at least downgrade). The rule interacts with reject: if LTV > 85 → reject; should mileage force review then? That would be an upgrade from reject to review, which seems wrong for credit risk. Most likely the intended semantics: approve → review when mileage > 400k; reject stays reject. But the code can't tell us; docs/plan mention boundaries 399999/400000/400001. The client_note mentions mileage fraud case. The docs/README says the feature is "учебная фича" for day 1 chain. So semantics per docs: "пробег авто не больше 400 000 км, иначе решение review". I shouldn't over-interpret; I'll note the ambiguity explicitly: the code has no priority mechanism; if placed in decide(), need to decide ordering (e.g., check reject first, then mileage override approve→review, or mileage forces review in all cases).

For the answer, I'll say: the rule is a decision-level rule, and the decision lives in DecisionEngine::decide() — that's functionally where it "встанет". But the method only receives LTV, so two concrete implementation spots:

Option 1: extend decide() (signature + config + AppFactory wiring) — keeps all decision rules in one place.
Option 2: apply in assess() after decide() — minimal change, all data already in scope.
And config: new key in rules.php, e.g. under 'vehicle': 'review_mileage_km' => 400000 (name illustrative). Note: 400000 must be ≤ max_mileage_km=500000 — otherwise validator rejects the claim before the decision. Since 400000 < 500000, the rule effectively fires for 400001..500000.

Existing inputs:

$input['mileage'] int, already normalized and range-checked (0..500000 guaranteed) — available in assess(); validator returns it.
rules.php 'vehicle' section — a natural home for the new threshold; pattern of injecting config via constructor exists (DecisionEngine gets $rules['ltv']).
DecisionEngine::REVIEW constant exists. Missing:
the threshold key itself in rules.php — нет (only max_mileage_km=500000).
mileage in DecisionEngine: decide(float $ltv) — нет; constructor receives only $rules['ltv'] — нет vehicle rules.
no combined-rule/priority mechanism in decide() — would have to be written.
tests asserting mileage→review — нет (tests only use mileage as fixture data 84000/96000; no mileage-based decision assertions — I verified via grep; but I only saw lines 34/38 with fixture values. Let me be careful: I haven't read the full test files. My grep showed only fixture lines matched 'mileage'. A test named e.g. testMileage... wouldn't match grep pattern 'mileage|пробег' — actually it would match 'mileage' case-insensitively? grep default is case-sensitive; 'Mileage' wouldn't match. Hmm. Let me quickly check test files for Mileage with capital M or a decision-review-related mileage test to be accurate. Better: read the two test files quickly or grep case-insensitively. Let me grep ignoreCase for mileage in tests dir.
Let me verify the tests quickly before claiming "нет таких тестов".

Confirmed: no existing test asserts a mileage→review rule; mileage appears in tests only as fixture data. The existing review tests are LTV-based.

Now I have everything. Write the final answer in Russian, structured, precise, with line references, no fabrication. Include a compact mermaid flow diagram of the decision path.

Structure:

Участники и порядок вызовов (диаграмма + пошагово)
wiring via AppFactory (rules.php → компоненты)
assess(): validate → calculate LTV → decide → vehicle_age → result
Note the boundary nuance: docblock/config comments say "LTV <= approve_max -> approve", but code uses strict < (line 32), so LTV exactly 60.0 → review. Honest observation, worth including since user says don't invent things — pointing out the discrepancy is factual.
Где встанет новое правило
Where: two concrete spots (DecisionEngine::decide vs AssessmentService::assess after line 33). Explain what each requires.
Config: rules.php 'vehicle' — new key (e.g. review_mileage_km => 400000); naming per existing style.
What exists already: $input['mileage'] (int, validated 0..500000), REVIEW constant, config injection pattern, AppFactory passes $rules.
What's missing: threshold key (нет), mileage in decide() signature (нет), vehicle rules in DecisionEngine constructor (нет — only $rules['ltv']), priority mechanism for combining LTV rule with mileage rule (нет — must define behavior when LTV→reject and mileage>400k), tests (нет).
Note interplay: validator rejects mileage > 500000 with ValidationException, so the new rule fires only for 400001..500000; above 500000 the claim never reaches the decision.
Что уже проверяется про пробег
rules.php: max_mileage_km = 500000 (vehicle section)
ApplicationValidator lines 43–46: cast int, default -1, error if <0 or >500000, error message; on any errors → ValidationException → no decision at all.
After validation, mileage just travels: returned in $input, included in result['input'], saved to DB via ApplicationRepository (outside Domain), and in Domain it does NOT affect LTV/decision/limit.
Explicitly: in DecisionEngine — нет; LtvCalculator — нет; VehicleAge — нет; ltv_by_age — нет (and it's not used at all yet, LOAN-12).
Tests: only fixture values; no mileage-based decision assertions — нет.
Also mention: mileage отсутствует в payload → default -1 → validation error. That's a current check too.

For the new rule placement recommendation: The user asks "в какой функции и в каком месте оно встанет" — a direct question expecting a specific answer. The most precise answer: the decision value is produced only in DecisionEngine::decide(); but since it takes only LTV, in practice the rule lands either (a) inside decide() after extending its signature, or (b) in AssessmentService::assess() between lines 33 and 35, overriding $decision. I'll give a recommendation: b) is the minimal-change spot where all needed data already exists; a) is architecturally "правильнее" if we want all decision rules centralized, but requires signature change + config wiring change. I'll recommend (b) as the immediate answer to "в каком месте" while noting (a). Actually let me pick one as primary recommendation. Given the question phrasing, I think they want: "AssessmentService::assess(), после $decision = $this->decisionEngine->decide($ltv); (строка 33), до формирования массива результата (строка 35)" — because there mileage is available. And note the alternative in decide() with what's missing. I'll present it that way: primary spot — assess() (or decide() with signature extension), with exact lines.

Hmm, but semantically "иначе решение review" — if LTV gives reject and mileage > 400k? The rule text says "не больше 400 000 км, иначе решение review" — reads like an override to review. I'll flag that the code offers no priority mechanism and the behavior for reject+high mileage must be explicitly defined (approve→review vs reject→review). docs/plan/README mentions boundary values 399999/400000/400001 — I can reference that the docs already plan this feature (day-1 chain), showing it's planned but NOT implemented in code. That's a useful factual note: "в коде нет, в docs/README.md и docs/plan/README.md это запланировано как учебная фича". This is accurate per the grep.

Diagrams: a flowchart of assess() pipeline would help. Keep it compact.

Answer in Russian. Include file paths and line numbers. Be careful: DecisionEngine docblock lines 10-12 vs code line 32 — strict < vs <=. rules.php comment lines 38-41 also says <=. So at LTV exactly 60.0 code gives review while comments say approve. Factual discrepancy — include briefly.

Also LtvCalculator: throws InvalidArgumentException — but validator already blocks market_value <= 0 and requested_amount outside 50k..2M, so those exceptions are unreachable via assess() flow (defensive). Worth a short note? It's factual: in the assess() pipeline the validator runs first, so calculate()'s exceptions are defensive/dead in that path. I can mention briefly in the flow description. Keep it short.

approved_limit: requested_amount on approve, 0 otherwise — includes review. So for review the limit is 0. That's part of the answer shape.

Now compose. Use headers, code refs like DecisionEngine::decide() (backend/src/Domain/DecisionEngine.php:30-41). Keep the answer thorough but not bloated. Mermaid diagram of the call order.

Как считается решение
Участники и порядок
Сборка зависимостей происходит в backend/src/AppFactory.php:27-39: массив из backend/config/rules.php целиком уходит в ApplicationValidator, его кусок $rules['vin'] — в VinValidator, а $rules['ltv'] — в DecisionEngine. VehicleAge получает текущий год через (int) date('Y').

Сам расчёт — единственная точка входа AssessmentService::assess(array $payload) (backend/src/Domain/AssessmentService.php:28-42):

ValidationException

assess payload

ApplicationValidator::validate

ошибки валидации, решение не считается

LtvCalculator::calculate

DecisionEngine::decide

VehicleAge::inYears для vehicle_age

approved_limit: requested_amount при approve, иначе 0

По шагкам:

ApplicationValidator::validate($payload) (ApplicationValidator.php:24-83) — проверяет каждое поле по rules.php и либо бросает ValidationException со списком ошибок, либо возвращает нормализованный массив {vin, year, mileage, market_value, requested_amount, term_months}. Внутри по порядку вызываются VinValidator::isValid() (строка 29) и VehicleAge::inYears() (строка 34).
LtvCalculator::calculate($input['requested_amount'], $input['market_value']) (LtvCalculator.php:15-26) — round(сумма / оценка * 100, 2). Его InvalidArgumentException при неположительных значениях в этом конвейере недостижимы: валидатор уже отсёк market_value <= 0 и сумму вне 50 000–2 000 000 — это защитные проверки.
DecisionEngine::decide(float $ltv) (DecisionEngine.php:30-41) — единственное место, где рождается решение:
$ltv < 60.0 → approve;
60.0 <= $ltv <= 85.0 → review;
$ltv > 85.0 → reject.
Нюанс: докблок класса (строки 10–12) и комментарий в rules.php (строки 39–41) обещают LTV <= approve_max -> approve, но код использует строгое < (строка 32), то есть при LTV ровно 60.0 фактическое решение — review, а не approve.
VehicleAge::inYears($input['year']) — возраст авто только для поля vehicle_age в ответе, на решение не влияет.
Результат: approved_limit = запрошенная сумма при approve, иначе 0 (в том числе при review).
Решение зависит только от LTV. Справочник ltv_by_age из rules.php:53-58 не используется вообще нигде — по комментариям это задача LOAN-12, она не сделана.

Куда встанет правило «пробег > 400 000 км → review»
Функция, производящая решение, — DecisionEngine::decide(), но её сигнатура принимает только float $ltv, пробега в ней нет. Поэтому два реальных места:

Вариант А (минимальная правка): AssessmentService::assess(), сразу после строки 33 ($decision = $this->decisionEngine->decide($ltv);) и до формирования массива результата (строка 35) — переопределить $decision в DecisionEngine::REVIEW, если $input['mileage'] превышает порог. Здесь все нужные данные уже в области видимости.
Вариант Б (вся логика решения в одном месте): внутри DecisionEngine::decide() — но тогда придётся расширять сигнатуру (передавать пробег) и отдавать конструктору секцию vehicle из конфига (сейчас он получает только $rules['ltv'], AppFactory.php:37).
Порог 400 000 по конвенции проекта («бизнес-числа не хардкодим») должен лечь в rules.php — естественное место: новый ключ в секции vehicle (строки 20–24), по образцу max_mileage_km.

Что для правила уже есть:

$input['mileage'] — целое, нормализованное валидатором, с гарантированным диапазоном 0–500 000 (ApplicationValidator.php:43-46, 78);
константа DecisionEngine::REVIEW (DecisionEngine.php:17);
готовый паттерн «порог из конфига через конструктор» (как approveMax/reviewMax в DecisionEngine);
сам порог 400 000 < существующего max_mileage_km = 500 000, значит правило реально сработает на диапазоне 400 001–500 000.
Чего не хватает:

ключа с порогом 400 000 в rules.php — нет (есть только max_mileage_km = 500 000, это другая проверка);
пробега в DecisionEngine::decide() — нет, сигнатура decide(float $ltv);
у DecisionEngine доступа к секции vehicle конфига — нет;
механизма приоритетов решений — нет: decide() — одна цепочка if'ов по LTV, и поведение комбинации «LTV → reject, пробег > 400 000» (останется reject или станет review?) в коде не определено, её придётся задать явно;
тестов на это правило — нет: в tests/ пробег встречается только как фикстура (84 000 и 96 000), решений по пробегу тесты не проверяют.
Важно про пересечение с валидацией: пробег > 500 000 отбрасывается ещё в ApplicationValidator с ValidationException, до расчёта решения такие заявки не доходят вообще.

Что сейчас проверяется про пробег
rules.php:23 — 'max_mileage_km' => 500000 в секции vehicle.
ApplicationValidator::validate(), строки 43–46 — приведение к int (отсутствующее поле даёт -1 и, соответственно, ошибку), затем проверка: отрицательный пробег или пробег больше 500 000 → ошибка 'Пробег от 0 до 500000 км' в общем списке → ValidationException. То есть сейчас пробег влияет только на «пройдёт ли заявка валидацию», а не на решение.
После валидации пробег просто передаётся дальше: попадает в $input, возвращается в ответе в поле input (AssessmentService.php:40) и сохраняется в БД через ApplicationRepository (backend/src/Repository/ApplicationRepository.php:38-45). В LtvCalculator, DecisionEngine, VehicleAge, VinValidator и в справочнике ltv_by_age пробег не участвует — нет.
Само правило «пробег > 400 000 → review» в коде не реализовано; в репозитории оно пока существует только как план в docs/README.md и docs/plan/README.md (там же заданы граничные значения 399 999 / 400 000 / 400 001).



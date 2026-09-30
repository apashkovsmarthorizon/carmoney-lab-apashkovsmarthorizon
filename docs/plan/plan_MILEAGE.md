# План MILEAGE: проверка пробега в решении по заявке

Правило: пробег авто не больше 400 000 км — иначе решение `review`.
Порог выносим в `backend/config/rules.php` (бизнес-числа не хардкодим).

Замечание: `docs/setup/code_map.md` в репозитории отсутствует — план составлен по
прямому чтению кода и отчёту scout по всем упоминаниям пробега в `backend/`,
`frontend/`, `db/`, `tests/`.

## Контекст (что найдено)

- Решение сейчас считает только `DecisionEngine::decide(float $ltv)` — пробег в
  расчёте не участвует вовсе.
- Пробег сегодня используется только в валидации входа
  (`ApplicationValidator`, лимит `vehicle.max_mileage_km = 500000`) и в
  записи/чтении в БД (`vehicles.mileage_km`).
- `AssessmentService::assess()` — точка, где сходятся валидированный вход
  (`$input['mileage']`), LTV и решение; здесь и встраивается новое правило.
- В `docs/README.md`, `docs/plan/README.md`, `docs/hw1/README.md` и задании
  ДЗ.1 уже фигурирует порог 400 000 — источник требования подтверждён.

## 1. Файлы

- `backend/config/rules.php` — добавить в секцию `vehicle` ключ
  `decision_max_mileage_km => 400000` (существующий `max_mileage_km` не трогаем).
- `backend/src/Domain/DecisionEngine.php` — новый метод
  `decide(float $ltv, int $mileage): string`: считает решение по LTV, затем при
  пробеге > порога понижает `approve` до `review` (reject остаётся reject);
  порог передаётся в конструктор вместе с порогами LTV.
- `backend/src/Domain/AssessmentService.php` — передать `$input['mileage']`
  в `decide()` (сейчас передаётся только LTV).
- `backend/src/AppFactory.php` — передать новый параметр при сборке
  `DecisionEngine` (сейчас только `$rules['ltv']`).
- `tests/Unit/DecisionEngineTest.php` — обновить конструктор в `setUp()`,
  добавить кейсы по пробегу.
- `tests/Unit/AssessmentServiceTest.php` — данные-провайдер payload: параметризовать
  `mileage`; добавить сценарии пробег × LTV.
- `docs/plan/README.md` — отметка, что MILEAGE спланирован (если так ведётся реестр).

Правки `ApplicationValidator`, `ApplicationRepository`, `db/`, `frontend/`
не требуются: пробег уже валидируется, хранится и приходит в payload.

## 2. Шаги реализации

1. В `rules.php` добавить `vehicle.decision_max_mileage_km = 400000` с комментарием,
   чем он отличается от `max_mileage_km` (порог валидации входа).
2. Расширить `DecisionEngine`: конструктор принимает
   `array{approve_max:float, review_max:float, max_mileage_km:int}`;
   `decide(float $ltv, int $mileage)` после LTV-логики: если решение `approve`
   и `mileage > max_mileage_km` → `review`. Обновить docblock класса.
3. В `AssessmentService::assess()` вызвать
   `$this->decisionEngine->decide($ltv, $input['mileage'])`.
4. В `AppFactory::create()` передать в `DecisionEngine` порог пробега из
   `$rules['vehicle']['decision_max_mileage_km']`.
5. Обновить/дополнить тесты (раздел 3).
6. Прогнать `make test` и `make lint`; вручную проверить
   `POST /api/ltv` с пробегом 400001 (ожидается `decision: review`).

## 3. Тесты

`DecisionEngineTest` (новые кейсы поверх существующих LTV-кейсов):
- 399999 + LTV из approve-зоны → `approve` (граница включена);
- 400000 + LTV из approve-зоны → `approve`;
- 400001 + LTV из approve-зоны → `review` (новое правило);
- 400001 + LTV из reject-зоны → `reject` (пробег не «повышает» решение);
- 400001 + LTV из review-зоны → `review` (не меняется).

`AssessmentServiceTest` (сквозные сценарии, payload с пробегом как параметр):
- payload `mileage: 400001`, LTV 50% (approve-зона) → `decision: review`,
  `approved_limit: 0` (лимит выдаётся только при approve);
- payload `mileage: 399999`, LTV 50% → `approve`, лимит = запрошенной сумме;
- payload `mileage: 400000`, LTV 50% → `approve`.

Пустой пробег:
- `mileage` отсутствует в payload: валидатор сегодня приводит его к `-1` и
  отдаёт ошибку `'mileage' => 'Пробег от 0 до ...'` → тест в
  `ApplicationValidatorTest`: `expectValidationException` с ключом `mileage`
  (заявка не доходит до расчёта решения);
- пограничный тест на саму валидацию: 500000 проходит валидацию, 500001 — нет
  (существующий лимит `max_mileage_km`, сегодня не покрыт тестами).

## 4. Риски

- Конфликт с существующей проверкой `max_mileage_km = 500000`: она отклоняет
  заявку целиком (ValidationException) при пробеге > 500000, а новое правило при
  400001–500000 должно давать `review`. Пороги обязаны остаться независимыми:
  `max_mileage_km` — валидация входа, `decision_max_mileage_km` — правило решения.
  Если реализовать новое правило «в валидаторе», кейсы 400001–500000 сломаются.
- Изменение сигнатуры `DecisionEngine::decide()` (новый обязательный параметр):
  нарушит все текущие вызовы и тесты — нужен один заход, а не постепенная
  миграция; `DecisionEngineTest` пересобирается целиком.
- Семантика понижения: принято «approve → review, reject не трогаем». Если
  заказчик имел в виду «всегда review» (даже вместо reject) — переделка.
- `approved_limit`: при решении review из-за пробега лимит обязан стать 0,
  иначе фронт покажет одобренную сумму у заявки на ручном reviewing.
- Семантика пустого пробега: сейчас `(int) null = 0` в валидаторе не различает
  «не указан» и «0 км»... (уточнено: код использует `?? -1`, поэтому отсутствие
  поля отклоняется; но `mileage: ''` и `mileage: null` тоже дадут ошибку —
  поведение тестами не закреплено, полагаться нельзя без теста).
- Не входит: расчёт лимита по `ltv_by_age` (задача LOAN-12), изменения фронтенда,
  схемы БД и seed-данных, изменение `max_mileage_km = 500000`.

## Вопросы к заказчику

1. Пробег > 400 000 при LTV из reject-зоны — оставляем `reject` или требование
   «иначе review» означает ровно `review` даже вместо reject?
2. Порог 400 000 — включительно (400 000 → approve) или строго меньше
   (400 000 уже → review)? В плане принят вариант «не больше» = включительно.
3. `max_mileage_km = 500000` в валидации остаётся как есть, или его тоже
   свернуть к 400 000 (тогда правило решения станет мёртвым для 400001+)?
4. Пустой пробег в заявке — это ошибка валидации (как сейчас) или должен
   давать `review`?
5. При `review` из-за пробега лимит = 0 (как сейчас у любого review) — верно?

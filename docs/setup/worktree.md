Тесты в tests/Unit/:

VinValidatorTest.php — проверяет формат VIN через провайдер данных: корректный 17-символьный VIN (в т.ч. в нижнем регистре) проходит, а короткий/длинный, с запрещёнными буквами (I, O, Q), спецсимволами или пустой — нет.
LtvCalculatorTest.php — проверяет расчёт LTV в процентах (половина стоимости → 50.0, треть → 33.33, больше стоимости → 120.0) и выброс InvalidArgumentException при нулевой рыночной стоимости или неположительной сумме.
DecisionEngineTest.php — проверяет решение по LTV с порогами из конфига (≤60 → approve, ≤85 → review, >85 → reject), включая граничные значения 85.0 и 85.01.
AssessmentServiceTest.php — интеграционный сценарий сервиса оценки на реальном backend/config/rules.php: низкий LTV → approve с лимитом, равным запрошенной сумме; средний → review с нулевым лимитом; высокий → reject с нулевым лимитом.
ApplicationValidatorTest.php — проверяет валидацию заявки: корректные данные проходят с нормализацией VIN в верхний регистр; будущий год, сумма ниже минимума и набор ошибок сразу по нескольким полям (vin, market_value, term_months) отклоняются с ValidationException.
Работаю в папке C:\Users\apashkov\carmoney-lab\.kilo\worktrees\camp-mushroom на ветке camp-mushroom.
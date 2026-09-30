Агент planner строит план изменений до кода. Пишет только в docs/plan/.
Агент scout Разведчик. Ищет по коду и возвращает список мест с путями и строками.

Ответ агента scout
Все места, где читается mileage
#	Файл	Строка	Что делает	Тип
1	backend/config/rules.php	23	Объявляет порог max_mileage_km => 500000	конфиг
2	backend/src/Domain/ApplicationValidator.php	43–45	Читает $payload['mileage'], проверяет диапазон [0, max_mileage_km], пишет ошибку	явное чтение + валидация
3	backend/src/Domain/ApplicationValidator.php	78	Возвращает 'mileage' => $mileage в нормализованный массив	передача
4	backend/src/Repository/ApplicationRepository.php	38–39	SQL INSERT ... mileage_km	сохранение
5	backend/src/Repository/ApplicationRepository.php	45	Биндинг :mileage из $input['mileage']	передача
6	backend/src/Repository/ApplicationRepository.php	68	SELECT v.mileage_km	чтение из БД
7	backend/src/Http/ApplicationController.php	25, 56	getParsedBody() без явного ключа	неявное чтение
8	db/schema.sql	22	Колонка mileage_km INT UNSIGNED NOT NULL	схема
9	db/seed.sql	31	Список колонок mileage_km в INSERT	сидинг
10	tests/Unit/ApplicationValidatorTest.php	34	'mileage' => 84000 в payload теста	передача
11	tests/Unit/AssessmentServiceTest.php	38	'mileage' => 96000 в payload теста	передача
12	frontend/index.html	30–31	<input name="mileage">	отрисовка
13	frontend/app.js	8	'mileage' в NUMERIC_FIELDS	подготовка к отправке
14	frontend/app.js	10–17	collectPayload() кладёт mileage в body через FormData	неявная отправка
Замечания (только то, что видно в коде)
Явная точка входа mileage в систему — ровно одна: ApplicationValidator.php:43–45. Всё остальное — транзит.
ApplicationController и collectPayload() mileage не трогают, а пробрасывают весь payload целиком.
В ApplicationRepository ключ массива 'mileage' уходит в колонку mileage_km — рассогласование имени ключа PHP и колонки БД.
Никаких отдельных MileageValidator / MileageException нет. Валидация пробега — общая ветка в ApplicationValidator с общим ValidationException.
В tests/ нет граничных кейсов по пробегу (399999 / 400000 / 400001), хотя в docs/spec/README.md они упоминаются.
Чего не хватает, чтобы это стало планом
Цель задачи: добавить граничные тесты? вынести mileage в отдельный валидатор? сменить ключ mileage → mileage_km в payload? что-то ещё?
ID задачи для артефакта docs/plan/plan_<ID>.md (или подтверждение, что план не нужен — только разведка).
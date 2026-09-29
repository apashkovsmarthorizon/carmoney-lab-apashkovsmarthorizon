готов

Сервис: учебный сервис предварительной оценки заявки на заём под ПТС (carmoney-lab) — принимает заявку (VIN, год, пробег, оценочная стоимость, сумма, срок), считает LTV и возвращает решение approve / review / reject (PHP 8.3 + Slim, MySQL 8, форма на ванильном JS, данные синтетические).
Команды: Makefile — `make help`, `make up`, `make down`, `make ps`, `make logs`, `make install`, `make test`, `make lint`, `make seed`; docker-compose.yml поднимает `backend` (php -S на :8080, порт через APP_PORT) и `db` (mysql:8.0, порт через DB_PORT, том db-data, initdb-скрипты db/schema.sql и db/seed.sql).
Решение: считается в `backend/src/Domain/` — классы `DecisionEngine.php`, `AssessmentService.php` (правила из `backend/config/rules.php`).

модель: training-2026-09-minimax-m3

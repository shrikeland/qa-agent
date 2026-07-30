---
ticket: MIA-83
linear_url: https://linear.app/mia360/issue/MIA-83/pravilnyj-otvet-zadaniya-dostupen-studentu-cherez-api-spisyvanie
status: Testing
mr_url: https://gitlab.com/ai-math/ai-math-web/-/merge_requests/276
updated: 2026-07-29T16:26:49.577Z
---

## Контекст

Security-баг (label `Bug`, priority High): эндпоинты выдачи заданий отдавали студенту секретные поля задания — правильный ответ (`answer`), эталонное решение (`steps`), методические `rules`/`hints`/`examples`. Это открывало списывание в тренажёре и, что серьёзнее, в диагностике — там утечка обесценивает замер уровня и позволяет накручивать mastery/прогресс.

Задача закрывалась в два захода:
- MR !276 — основной fail-closed фикс утечки полей задания по трём каналам (HTTP `/assignments*`, HTTP submission/interaction-ответы, WS `TRAINER_INTERACTION_RECEIVE`/`DIAGNOSTIC_INTERACTION_RECEIVE`).
- MR !296 — отдельный follow-up по проблеме, найденной уже в ходе QA-прогона первого фикса (наблюдение тестировщика): ни на одном канале (HTTP internal/public, WS) не проверялось владение сессией — id сессии/сабмишена берётся из клиентского пейлоада и раньше использовался без вопросов.

Оба MR уже в master (текущий HEAD включает оба).

## Что было сломано

1. **Утечка секретных полей задания студенту** (исходная проблема тикета):
   - `GET /assignments/:id` и `GET /assignments?...&hasAnswers=1` — под `requireAuth`, но без фильтрации ответа, отдавали `answer`, `rules`, а с фичей эталонного решения отдавали бы и `steps`.
   - `POST /internal/trainer/sessions/:id/interactions` (`addTrainerSessionInteraction`) тянул весь `assignment` через `getById` с `include: { assignment }` (все поля), а socket-gateway ретранслировал его в браузер сообщением `TRAINER_INTERACTION_RECEIVE` на каждый AI-ответ (9+ точек emit, включая busy/error-хелперы, которые вообще не чистились).
   - Клиент действительно использовал `answer` — фронт грузил задания с `hasAnswers=1` (`trainer-session.ts`, 3 места в `diagnostic-session.ts`) для мгновенной client-side проверки ответа; убрать поле нельзя было без замены механизма проверки на серверный.
   - Заодно был debug-оверлей в тренажёре, открывавшийся 4 кликами по аватару Мии и дампивший задание вместе с ответом.

2. **Отсутствие проверки владения сессией** (найдено при adversarial QA-прогоне первого фикса, комментарий тестировщика, наблюдение 1):
   - Id диагностической/тренажёрной сессии приезжают из браузерного пейлоада и раньше использовались без сверки с текущим пользователем — ни на internal HTTP-роутах (вызываемых socket-gateway от имени студента), ни на публичных HTTP-роутах, ни в WS-хендлерах.
   - Конкретно: `diagnostic:session:join` по чужому `diagnosticSessionId` отдавал статус/балл/текущее задание чужой сессии и даже записывал результат в дашборд-кэш того, кто запросил. `diagnostic:submission:reject`/`:accept` (`softDelete`/подтверждение) шли по одному `submissionId` (целочисленный, последовательный, перебираемый) без привязки к сессии и без проверки пользователя вообще — то есть чужие сабмишены можно было удалить или подтвердить. Смена текущего задания и запуск скоринга в чужой сессии — тем же способом. HTTP `GET /diagnostic|trainer/sessions/:id/submissions|interactions` и `POST /trainer/sessions/:id/interactions` отдавали/принимали чужие данные аналогично. Раньше guard был fail-open: «нет заголовка — нет проверки».

## Что изменено (по коду)

**MR !276 — санитизация полей задания (fail-closed белый список):**

- `api/src/utils/assignment-privacy.ts` — `toStudentAssignment(assignment)`: белый список полей для студенческой выдачи (`id`, `topic.id`, `task`, `expression`, `trainerUnit`, `kind`, `mlLevel`, `presentedSolution`); `answer`/`rules`/`steps`/`hints`/`examples` не входят. Отдельно `toStudentAssignmentWithAnswer` — тот же белый список плюс `answer` (без steps/hints/rules/examples), используется только там, где владение сессией и её завершённость уже проверены на сервере.
- `api/src/controllers/assignment.controller.ts` — `getAssignments` больше не смотрит на клиентский `hasAnswers` (комментарий в коде явно это фиксирует), всегда мапит через `toStudentAssignment`. `handleGetAssignmentById` разведён на два хендлера: публичный `getAssignmentByIdHandler` (`includeSecrets=false`, белый список) и внутренний `getInternalAssignmentByIdHandler` (`includeSecrets=true`, полный `IAssignmentWithAnswerPublic` с `answer`/`hints`/`rules`/`steps`/`examples` — используется только грейдингом через `/internal/assignments/:id`).
- `diagnostic.controller.ts` — новый эндпоинт `getDiagnosticSessionResultAssignments` (`GET /diagnostic/sessions/:id/result-assignments`): проверяет `session.userId === currentUser.id` (403 иначе) и `session.finishedAt` (403 «not finished yet» иначе — специально `finishedAt`, а не `scoredAt`, т.к. сессия с незакрытым `REVIEW_ERROR` submission остаётся `IN_PROGRESS`/unscored, но уже показывается студенту как «результаты готовы»). Дальше берёт только сабмишены самой сессии (`resultAssignmentsBySession`, отфильтрованные по `SCORED`/`SCORE_CONFIRMED` и `deletedAt: null`) и мапит через `toStudentAssignmentWithAnswer` — то есть студент получает свой правильный ответ только по решённым в его завершённой сессии заданиям.
- `socket-gateway/src/helpers/assignment-privacy.helper.ts` — зеркальный белый список для WS (`id`, `topicId`, `task`, `expression`, `steps: []`).
- `socket-gateway/src/helpers/handler.helper.ts` — все эмиты `TRAINER_INTERACTION_RECEIVE`/`DIAGNOSTIC_INTERACTION_RECEIVE` (обычный ответ, bad-recognition, bad-review, error, busy) прогоняют `assignment` через `toStudentAssignment` перед отправкой в сокет.
- Фронт (`frontend/stores/trainer-session.ts`, `diagnostic-session.ts`) больше не запрашивает `hasAnswers=1`; результаты диагностики/контрольной (`frontend/pages/diagnostic/results/[sessionId].vue`, `control-test/results/[sessionId].vue`) переведены на новый гейтед-эндпоинт; debug-оверлей по 4 кликам на аватар убран из `trainer/[topicId].vue`, `diagnostic/[[diagnosticId]].vue`, `control-test/index.vue`.

**MR !296 — проверка владения сессией:**

- `api/src/middleware/require-session-owner.ts` (новый) — единый guard. Достаёт id сессии из `params`/`body` (`diagnosticSessionId`/`trainerSessionId`/`id`, params раньше body), определяет владельца через репозиторий (`diagnosticSessionRepository.getById`/`trainerSessionRepository.get`), сверяет с actingUserId. Источник actingUserId различается по типу маршрута: публичные — `request.currentUser.id` (кука, `requireAuth`), internal — заголовок `x-user-id` (кладёт сам socket-gateway). Нет заголовка/куки → 401 (fail-closed, не «пропустить проверку»); сессия не найдена → 404; чужая → 403.
  - `requireOwnDiagnosticSession`/`requireOwnTrainerSession` — навешаны точечно на публичные роуты (`diagnostic.routes.ts`, `trainer.routes.ts`).
  - `requireInternalSessionOwner` — один global `preHandler`-хук на весь `internalRoutes`, матчит по форме URL (`/internal/diagnostic/(sessions/|try-score|long-processing-result)`, `/internal/trainer/sessions/`), так что новый internal-роут с session id получает проверку автоматически, без необходимости вспомнить о ней отдельно. Join — исключение (может создавать сессию), владение проверяется внутри `diagnostic-session.service.ts`.
- `api/src/controllers/diagnostic.controller.ts`/`trainer.controller.ts` — точечные проверки «своя сессия + чужой submissionId»: `submissionInSession(submissionId, sessionId)` перед `update`/`delete`/`confirm` сабмишена (сверяет `submission.diagnosticSessionId`/`trainerSessionId` с сессией из пути) — сравнение целочисленного submissionId со своей сессией, отдельно от общего guard.
- `socket-gateway/src/services/ai-math.api.ts` — все internal-запросы, которые раньше шли без идентификации, теперь передают `actingAs(userId)` → заголовок `x-user-id`, который читает `requireInternalSessionOwner` на API.
- `socket-gateway/src/handlers/diagnostic/{change-assignment,interactions,join,submission,submissions-complete}.handler.ts`, `trainer.handler.ts` — прокидывают `user.id` в вызовы `aiMathApi`, которые теперь его требуют.

## Как перепроверить

Основа — уже проведённый adversarial QA-прогон на `test-1` (PASS по MR !276), актуализированный под оба MR. Стенд: два студента через `/auth/register` → `/auth/activate`.

**HTTP, санитизация полей (MR !276):**
1. `/assignments?trainerId=...` и `?diagnosticId=...`, в т.ч. с попытками обхода `hasAnswers` (`1`, `true`, разный регистр, массив, дубль параметра, посторонний `includeSecrets`) — в ответе только `id/topic/task/expression/trainerUnit/kind/mlLevel/presentedSolution`, ни `answer`, ни `rules`/`steps`/`hints`/`examples`.
2. `GET /assignments/:id` — то же самое.
3. `POST`/`PUT` submissions (trainer и diagnostic) — `assignment` в ответе без секретов.
4. `/api/internal/*` без ключа → 401; `/api/system/*` → 401; Swagger → 404.

**WS, санитизация полей (MR !276):**
5. Тренажёр: прогон чата с `:request_hint:`, `:request_steps:`, `:request_theory:`, `:request_example:` — просмотреть все кадры `TRAINER_INTERACTION_RECEIVE` (в т.ч. busy/error-варианты, если удаётся спровоцировать) на предмет `answer`/`hints`/`rules`/`steps`(не `[]`)/`examples`.
6. Диагностика: полный прогон нескольких заданий, то же для `DIAGNOSTIC_INTERACTION_RECEIVE`.

**Результаты диагностики (MR !276):**
7. `GET /diagnostic/sessions/:id/result-assignments`: своя завершённая сессия → 200, свой `answer`, без `steps/hints/rules/examples`; чужая → 403; своя незавершённая → 403 «not finished yet»; несуществующий/битый id → 404; без cookie → 401.

**Владение сессией (MR !296 — тестировщик это ещё не перепроверял, ключевой новый пункт):**
8. WS `diagnostic:session:join` с чужим `diagnosticSessionId` → должен быть отклонён (раньше отдавал `session:init` с чужим статусом/баллом/текущим заданием и писал в чужой дашборд-кэш) — перепроверить именно то, что было наблюдением 1 в прошлом прогоне и на что закрыт MR !296.
9. WS `diagnostic:submission:reject`/`:accept` с чужим `submissionId` (в т.ч. подобрать соседний целочисленный id) — должен быть отклонён; ранее вёл в `softDelete`/подтверждение без всякой проверки.
10. WS смена текущего задания / запуск скоринга в чужой сессии — отклонено.
11. Запись в чужой чат тренажёра через `POST /trainer/sessions/:id/interactions` (или WS-эквивалент) — отклонено.
11a. **`POST /trainer/next-assignment` и WS `trainer:session:next-assignment` с чужим `trainerSessionId` — должны быть отклонены.** По QA-прогону от 2026-07-29 (см. секцию «QA-перепроверка» ниже) это ещё не так: посторонний ученик читает и меняет текущее задание чужой сессии. Кода-фикса под это пока нет — перепроверить в первую очередь.
12. HTTP `GET /diagnostic|trainer/sessions/:id/submissions|interactions` с чужим `:id` → 403 (ранее отдавал чужие сабмишены/переписку).
13. Пара «своя сессия + чужой submissionId» на update/delete/confirm сабмишена (и trainer, и diagnostic) — 404, не 200.
14. Отсутствие идентификации (нет куки на публичном роуте / нет `x-user-id` на internal) → 401, а не пропуск проверки — контроль того, что guard fail-closed, а не fail-open, как было раньше.

**Контрольная работа (не прогонялась вживую тестировщиком, механизм общий с диагностикой — важно закрыть отдельно):**
15. Пройти `CONTROL_TEST` целиком (потребуется платный доступ или временный обход paywall — freemium блокирует старт) через тот же путь `joinDiagnosticSession`/`DiagnosticType.CONTROL_TEST`: убедиться, что секреты задания не текут ни по одному каналу и что владение сессией контрольной проверяется так же, как у диагностики (`control-test/results/[sessionId].vue`, `control-test/index.vue`).

**Соцынженерия (регресс прошлого прогона, можно не повторять подробно, но держать в уме):** прямые просьбы к Мии выдать `answer`/поле ответа не должны срабатывать.

## Что посмотреть рядом (регресс)

- Грейдинг тренажёра и диагностики продолжает работать: `isCorrect`/`score` считаются верно (полный `assignment` по-прежнему доступен грейдингу через `/internal/assignments/:id`, `getInternalAssignmentByIdHandler`).
- UI разбора диагностики показывает «Твой ответ / Правильный ответ» на новом гейтед-эндпоинте.
- Тренажёр загружает `/assignments?trainerId=...` без `hasAnswers` и без ответов, при этом продолжает нормально показывать карточки заданий (в т.ч. v3 find-error задания, которые подмешиваются отдельным запросом `findFindErrorByTrainer`).
- Подсказки (`hints`) — server-enrichment вынесен в отдельную задачу MIA-87; здесь важно не то, что подсказки стали содержательнее, а что они по-прежнему приходят (регресс генеративного фолбэка не должен быть задет этим тикетом).
- Кэш `entities:assignment:*`/`entities:diagnostic:*` хранит сырую запись из БД; фильтрация — на выходе (`toStudentAssignment`), при чтении из кэша тоже проходит, что стоит держать в уме, если менять способ сериализации.

## QA-перепроверка от 2026-07-29 — FAIL по одному каналу, код фикса ещё не тронут

Ре-тест после возврата задачи в Ready for Test (сейчас формально `Testing`). Стенд `test-1.mia360.ru`, master содержит оба коммита (`3c5665cf` — MR !276, `cc93830d` — MR !296) — новых коммитов по MIA-83 с тех пор нет. Гард на стенде подтверждён живым: `GET /diagnostic|trainer/sessions/<несуществующий-uuid>/…` → `404` с сообщением самого `require-session-owner`.

**Утечки секретных полей задания не найдено ни по одному каналу — исходный acceptance MIA-83 по-прежнему держится.** Не держится расширенный acceptance MR !296 «проверка владения сессией на всех каналах»:

**`POST /trainer/next-assignment` и WS `trainer:session:next-assignment` не проверяют владение тренажёрной сессией.** Посторонний ученик B, зная `trainerSessionId` ученика A (UUID, не перебирается, но и не секрет — гуляет в клиентском payload), может через эту ручку прочитать чужую сессию и **сменить её текущее задание**: `POST` → `200`, в ответе сессия A с новым текущим заданием; после переподключения у A действительно стоит другое задание. Воспроизведено дважды (HTTP и WS), на двух разных парах учеников. Секреты задания при этом не текут (белый список работает), то есть пункт «студент не может получить правильный ответ» не нарушен — нарушен именно второй канал того же класса (чтение и порча чужой сессии), ради которого делался MR !296.

**Гипотеза по коду (три места сразу, поэтому не подстраховал ни один слой):**
1. `api/src/routes/trainer.routes.ts` — на `/trainer/next-assignment` навешан только `requireTopicAccess`, `requireOwnTrainerSession` отсутствует (на соседних session-роутах — есть).
2. `api/src/middleware/require-session-owner.ts` — регэксп `INTERNAL_TRAINER = /^\/internal\/trainer\/sessions\//` не матчит `/internal/trainer/next-assignment`, поэтому internal-вход (через socket-gateway) тоже без проверки. Заодно стоит проверить `/internal/trainer/join` и `/internal/trainer/ai-bonus` — они по той же причине вне регэкспа.
3. `api/src/services/trainer-session.service.ts` — `nextAssignment(trainerSessionId, _user)` игнорирует пользователя (параметр так и назван `_user`) — последний рубеж тоже пустой.

Симметрично устроенный `diagnostic:session:join` (тоже «создаёт или возобновляет», тоже вне общего хука) закрыт проверкой внутри `diagnostic-session.service.ts` и работает — тренажёрному `next-assignment` такой внутренней проверки не досталось.

**Побочная находка (не блокер, мимо задачи):** `POST /trainer/sessions/:id/interactions` пишет `userId` из тела запроса без сверки с вызывающим (гард теперь привязывает саму сессию к вызывающему, поэтому чужую переписку так не испортить, но автор записи — по-прежнему клиентское поле). Тот же паттерн в `nextAssignmentInTrainerSession`.

**Статус:** задача осталась в текущем статусе (Ready for Test на момент комментария, сейчас Testing по Linear) — тестировщик статус не двигал до закрытия next-assignment. Новой правки под эту находку в репозитории пока нет (только MR !276 и !296, оба уже учтены выше) — значит гэп по `next-assignment` на момент этой заметки не закрыт, перепроверять его нужно отдельно от пунктов 1-15 «Как перепроверить» (там всё ещё PASS).

## Открытые вопросы / не блокеры

- Ретраи `POST /auth/activate` (~10 повторных запросов при входе по `/ref/<token>`, последний ловит 429) — к MIA-83 отношения не имеет, не подтверждено (по коду `/ref/[token]` дёргает `activate` один раз), не блокер — не проверять в рамках этой задачи.

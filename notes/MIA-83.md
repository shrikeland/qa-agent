---
ticket: MIA-83
linear_url: https://linear.app/mia360/issue/MIA-83/pravilnyj-otvet-zadaniya-dostupen-studentu-cherez-api-spisyvanie
status: Ready for Test
mr_url: https://gitlab.com/ai-math/ai-math-web/-/merge_requests/296
updated: 2026-08-19
---

> **Обновление 19.08.2026:** финальный раунд фикса (MR !336, ветка `fix/MIA-83-trainer-session-ownership`, коммит `6e8bcdb1`) закрыл единственный оставшийся пробел из прошлого прогона — `POST /trainer/next-assignment` и WS `trainer:session:next-assignment` без проверки владения тренажёрной сессией. Премастер с этим фиксом уже влит в master (текущий HEAD `7accf5a8` включает все три раунда). Памятка ниже переписана под финальное состояние: три раунда — как контекст, шаги перепроверки — только актуальные.

## Контекст

Security-баг (label `Bug`): эндпоинт выдачи заданий отдавал студенту правильный ответ (`answer`) и служебные поля (`rules`, эталонное решение) — списывание в тренажёре и, что серьёзнее, в диагностике (нечестный замер уровня, накрутка mastery/прогресса). Закрывалось тремя последовательными раундами, каждый следующий — по находке ре-теста предыдущего:

- **MR !276** (`3c5665cf`, 20.07) — fail-closed выдача задания: единый белый список полей во всех студенческих ответах (HTTP `/assignments*`, submission/interaction-ответы) и во всех WS-эмитах (`TRAINER_INTERACTION_RECEIVE`/`DIAGNOSTIC_INTERACTION_RECEIVE`, включая busy/error-хелперы); клиентский флаг `hasAnswers` перестал на что-либо влиять; правильный ответ на результатах диагностики/контрольной — через новый гейтед эндпоинт `GET /diagnostic/sessions/:id/result-assignments`; убран debug-оверлей (4 клика по аватару Мии, дампивший задание с ответом). QA PASS по утечке секретов, но найдено наблюдение: владение сессией нигде не проверялось.
- **MR !296** (`cc93830d`, 28.07) — проверка владения сессией на (почти) всех каналах: новый `require-session-owner.ts`, навешан на публичные роуты и единым hook'ом на все internal-роуты по форме URL; socket-gateway на каждом internal-вызове называет пользователя заголовком `x-user-id`; закрыты чтение/удаление/подтверждение чужих диагностических сессий и сабмишенов. Ре-тест нашёл ОДИН незакрытый канал: `POST /trainer/next-assignment` (HTTP) и WS `trainer:session:next-assignment` не проверяли владение — посторонний ученик мог переключить текущее задание чужого тренажёрного занятия (порча состояния, не утечка ответа).
- **MR !336** (`6e8bcdb1`, влит в premaster 18.08, затем premaster → master) — финальный фикс именно этого канала: проверка владения встала в `trainerSessionService.nextAssignment`, закрывает разом WS и internal-путь; публичные легаси-роуты `/trainer/join` и `/trainer/next-assignment` (действовали от имени `userId` из тела запроса, `join` этим же способом отдавал чужой `trainerSessionId` — та же уязвимость, дававшая ключ к первой) — удалены целиком (нулевой трафик на проде/тесте, не вызывались ни фронтом, ни каким-либо клиентом). Шлюз на этом вызове стал последним из session-роутов, кто начал называть ученика заголовком.

**Итог:** ответ задания (`answer`) студенту недоступен ни по одному из проверенных каналов; владение сессией (диагностической и тренажёрной) проверяется на HTTP (публичном и internal) и на WS. Грейдинг не затронут — полный `assignment` по-прежнему доступен только внутреннему пути.

## Что было сломано → что изменено

### 1. HTTP `/assignments*` — секреты в ответе задания

**Было:** `GET /assignments/:id` и `GET /assignments?...&hasAnswers=1` под `requireAuth` отдавали `answer`, `rules` любому авторизованному, включая студента; `hasAnswers` — клиентский флаг, ничем не проверялся.

**Стало** (`api/src/utils/assignment-privacy.ts`, `api/src/controllers/assignment.controller.ts`):
- `toStudentAssignment(assignment)` — fail-closed белый список: `id`, `topic.id`, `task`, `expression`, `trainerUnit` (+ `kind`/`mlLevel`/`presentedSolution` по схеме `IAssignmentPublic`). Новое поле в модели `Assignment` по умолчанию НЕ попадает в студенческий ответ — его нужно добавить в список явно.
- `getAssignments` больше не смотрит на `hasAnswers` из query вообще — всегда мапит через `toStudentAssignment`.
- `handleGetAssignmentById` получил `includeSecrets`: `false` для публичного `getAssignmentByIdHandler` (белый список), `true` только для internal-хендлера, отдающего полный `IAssignmentWithAnswerPublic` (`answer`, `hints`, `rules`, `steps`, `examples`) — используется грейдингом.
- Правильный ответ на результатах диагностики/контрольной — отдельный гейтед путь (см. п.3), не через `/assignments*`.

### 2. WS `TRAINER_INTERACTION_RECEIVE` / `DIAGNOSTIC_INTERACTION_RECEIVE` — тот же класс утечки, но по сокету

**Было:** `POST /internal/trainer/sessions/:id/interactions` тянул весь `assignment` (`include: { assignment }`, все поля) для грейдинга; socket-gateway ретранслировал его студенту в браузер тем же объектом на каждый AI-ответ, включая busy/error-хелперы.

**Стало** (`socket-gateway/src/helpers/assignment-privacy.helper.ts`, `handler.helper.ts`, `handlers/diagnostic/interactions.handler.ts`): зеркальный белый список `toStudentAssignment` на стороне gateway (`id`, `topicId`, `task`, `expression`, `steps: []`); прогоняется перед КАЖДЫМ emit `TRAINER_INTERACTION_RECEIVE`/`DIAGNOSTIC_INTERACTION_RECEIVE` — обычный ответ, bad-recognition, bad-review, critical-error, busy. Ни один из этих путей больше не может случайно пронести необрезанный `assignment` в браузер.

### 3. Правильный ответ на результатах — новый гейтед эндпоинт

**Стало** (`api/src/controllers/diagnostic.controller.ts`, роут в `diagnostic.routes.ts`): `GET /diagnostic/sessions/:id/result-assignments` — единственное место, где студент вообще может получить `answer`. Проверяет `session.userId === currentUser.id` (иначе 403), затем `session.finishedAt` (иначе 403 «not finished yet» — сознательно `finishedAt`, а не `scoredAt`: сессия с незакрытым `REVIEW_ERROR`-сабмишеном остаётся `IN_PROGRESS`/unscored, но уже показана студенту как «результаты готовы»). Отдаёт `toStudentAssignmentWithAnswer` (белый список + `answer`, без `steps/hints/rules/examples`) только по сабмишенам самой этой сессии.

### 4. Владение диагностической сессией — join, submission reject/accept и остальные каналы

**Было:** id сессии/сабмишена бралось из клиентского пейлоада без проверки владельца — ни на internal HTTP (socket-gateway → API от имени студента), ни на публичных HTTP-роутах, ни в WS. Конкретно: `diagnostic:session:join` по чужому `diagnosticSessionId` отдавал статус/балл/текущее задание чужой сессии и даже писал результат в дашборд-кэш вызывающего под чужим владельцем; `diagnostic:submission:reject`/`:accept` (удаление/подтверждение сабмишена) работали по одному целочисленному последовательному `submissionId` без всякой привязки к сессии — самое серьёзное из найденного, порча чужих результатов; смена текущего задания и запуск скоринга в чужой сессии — тем же способом; HTTP `GET /diagnostic/sessions/:id/submissions|interactions` отдавали чужие данные.

**Стало** (`api/src/middleware/require-session-owner.ts`, `api/src/routes/{diagnostic,internal}.routes.ts`, `socket-gateway/src/services/ai-math.api.ts`, `socket-gateway/src/handlers/diagnostic/*.handler.ts`):
- `requireOwnDiagnosticSession` — на публичных роутах, источник identity — `request.currentUser.id` (кука).
- `requireInternalSessionOwner` — единый `preHandler`-хук на ВЕСЬ `internalRoutes`, матчит по форме URL (`/internal/diagnostic/(sessions/|try-score|long-processing-result)`), так что новый internal-роут с session id получает проверку автоматически, без необходимости отдельно об этом вспомнить; identity — заголовок `x-user-id`, который теперь на каждый internal-вызов кладёт socket-gateway (`actingAs(userId)` в `ai-math.api.ts`).
- `join` — исключение общего guard'а (может создавать сессию), владение сверяется внутри `diagnostic-session.service.ts` при резюме существующей сессии по `diagnosticSessionId`.
- `submissionInSession(submissionId, diagnosticSessionId)` в `diagnostic.controller.ts` — отдельная проверка «своя сессия, но чужой submissionId» перед update/delete/confirm сабмишена (сверяет `submission.diagnosticSessionId` с сессией из пути).
- Отсутствие identity (нет куки / нет `x-user-id`) → 401, fail-closed, а не пропуск проверки.

### 5. Владение тренажёрной сессией — interactions и next-assignment (закрыто в MR !296 частично, полностью — в MR !336)

**Было (MR !296, ре-тест нашёл):** `POST /trainer/next-assignment` и WS `trainer:session:next-assignment` не проверяли владение. Причина — сразу в трёх местах: (1) роут `/trainer/next-assignment` имел только `requireTopicAccess`, без `requireOwnTrainerSession`; (2) регэксп `INTERNAL_TRAINER = /^\/internal\/trainer\/sessions\//` не матчил `/internal/trainer/next-assignment`; (3) `trainerSessionService.nextAssignment(trainerSessionId, _user)` игнорировал переданного пользователя (параметр так и назывался `_user`). Эффект, подтверждённый на стенде: ученик B, передав `trainerSessionId` ученика A, получал 200 и реально переключал текущее задание в чужой сессии — подтверждено переподключением A. Секреты задания при этом не текли (белый список из MR !276 отрабатывал) — порча состояния, не списывание.

**Стало (MR !336):**
- `trainer-session.service.ts` — `nextAssignment(trainerSessionId, user)` теперь сверяет `trainerSession.userId !== user.id` → бросает `ERRORS.forbidden` (403), проверка внутри сервиса закрывает разом и internal HTTP-путь, и WS-путь через шлюз (не нужно чинить регэксп маршрутизации отдельно).
- `socket-gateway/src/services/ai-math.api.ts` — `nextAssignmentTrainerSession` теперь шлёт `actingAs(userId)` (заголовок `x-user-id`) на `/internal/trainer/next-assignment` — раньше это был последний из session-вызовов шлюза, который не называл пользователя.
- **Публичные `/trainer/join` и `/trainer/next-assignment` удалены целиком** (не запатчены) — они действовали от имени `userId` из тела запроса, а `/trainer/join` тем же способом отдавал `trainerSessionId` любого пользователя, чей `userId` назвали в теле — то есть сам был источником «ключа» для атаки на next-assignment (второй, не описанный в отчёте прошлого прогона дефект: вход в занятие сам отдавал чужой session id без перебора). Оба роута — легаси, нулевой трафик на проде/тесте, не вызывались ни фронтом, ни клиентом за всю историю репозитория. Internal-эквиваленты (`/internal/trainer/join`, `/internal/trainer/next-assignment`) остались — их вызывает только socket-gateway, привязка к пользователю там теперь через `x-user-id` + сверку в сервисе.
- `requireOwnTrainerSession` уже стоял (с MR !296) на `/trainer/sessions/:id/interactions`, `/trainer/sessions/:id/submissions`, create/update submission, `last-response` — этих каналов правка не касалась, но по ним стоит пройтись регрессом (см. ниже).

## Шаги перепроверки

Стенд: два студента (A — «жертва», B — «атакующий») через `/auth/register` → `/auth/activate`, у обоих активная тренажёрная и диагностическая сессия по своим темам.

**Секреты задания (регресс MR !276 — должно продолжать работать):**
1. `GET /assignments?trainerId=...`/`?diagnosticId=...`, в т.ч. с `hasAnswers=1`/`true`/иным регистром — в ответе только белый список полей, ни `answer`, ни `rules`/`steps`/`hints`/`examples`.
2. `GET /assignments/:id` — то же.
3. Тренажёр и диагностика: полный прогон нескольких заданий (включая `:request_hint:`/`:request_steps:`/`:request_example:` в чате), просмотреть кадры `TRAINER_INTERACTION_RECEIVE`/`DIAGNOSTIC_INTERACTION_RECEIVE`, включая спровоцированные busy/error-варианты — секретов нет нигде.
4. `GET /diagnostic/sessions/:id/result-assignments`: своя завершённая сессия A → 200 с `answer` только по своим сабмишенам; чужая (B запрашивает сессию A) → 403; своя незавершённая → 403 «not finished yet»; несуществующий id → 404; без cookie → 401.

**Владение диагностической сессией (регресс MR !296 — должно продолжать работать):**
5. B отправляет WS `diagnostic:session:join` с `diagnosticSessionId` сессии A → отказ (не отдаёт статус/балл/текущее задание A, не пишет в дашборд-кэш B под чужим владельцем).
6. B отправляет WS `diagnostic:submission:reject`/`:accept` с `submissionId` сабмишена A (в т.ч. соседний по номеру id) → отказ, сабмишен A не удалён и не подтверждён.
7. B шлёт HTTP `GET /diagnostic/sessions/:id/submissions` и `/interactions` с `:id` = сессия A → 403.
8. B шлёт update/delete/confirm сабмишена в СВОЕЙ сессии, но с чужим (A) `submissionId` → 404, не 200.

**Владение тренажёрной сессией — главный фокус этого прогона (MR !336, финальный фикс):**
9. B отправляет `POST /trainer/next-assignment` с `trainerSessionId` сессии A (свой валидный `topicId`) → ожидание: **404** (маршрут удалён целиком, не 401/403 и не общая поломка — проверить, что это именно отсутствие роута, а не сбой сервера).
10. B отправляет WS `trainer:session:next-assignment` с `trainerSessionId` сессии A → ожидание: отказ (403/forbidden по сути; конкретный код на клиенте — по факту события, задание в приложении B не переключается). Дополнительно проверить, что задание в СЕССИИ A НЕ МЕНЯЕТСЯ — переподключить клиента A (или дождаться его следующего обновления) и убедиться, что текущее задание то же, что было до попытки B.
11. Контроль на A: пока идёт п.10, A продолжает нормально пользоваться тренажёром — переключение следующего задания по СВОЕМУ `trainerSessionId` у A работает как раньше (200, задание меняется).
12. B отправляет `POST /trainer/join` (публичный, удалённый роут) с `userId` A и своим `topicId` → ожидание: 404 (роут отсутствует), убедиться что не отдаётся `trainerSessionId` A ни в каком виде.
13. B отправляет HTTP `GET /trainer/sessions/:id/interactions|submissions` и create/update submission с `:id`/`trainerSessionId` сессии A → 403 (регресс MR !296, `requireOwnTrainerSession`).

**Соцынженерия (регресс прошлых прогонов):** прямые просьбы к Мии выдать `answer`/поле ответа не должны срабатывать ни в тренажёре, ни в диагностике.

## Что посмотреть рядом (регресс)

- Грейдинг тренажёра и диагностики продолжает работать корректно: `isCorrect`/`score` считаются верно — полный `assignment` (с `answer`) по-прежнему доступен грейдингу через internal-путь (`getInternalAssignmentByIdHandler`, `aiMathApi.getAssignment` в socket-gateway перед сверкой ответа).
- `GET /diagnostic/sessions/:id/result-assignments` — UI разбора диагностики/контрольной показывает «твой ответ / правильный ответ» на этом эндпоинте, не деградировал по сравнению с прошлым способом.
- Свой чат тренажёра (`POST /trainer/sessions/:id/interactions`) и свои сабмишены (create/update) у A и B по отдельности продолжают работать без 403 — гард не должен ловить легитимные собственные запросы как чужие.
- Тренажёр у A: обычный флоу (следующее задание по СВОЕЙ сессии, отправка решения, получение результата) — 200 везде, задание меняется штатно.
- Диагностика/контрольная у A и B по отдельности: join, ответ на задание, завершение сессии, просмотр результатов — без регресса.
- Кэш `entities:assignment:*`/`entities:diagnostic:*` хранит сырую запись из БД; фильтрация происходит на выходе (`toStudentAssignment`) — если менять способ сериализации, не забыть, что чтение из кэша тоже должно проходить через белый список.

## Открытые вопросы

- **`userId` в `addTrainerSessionInteraction` по-прежнему берётся из тела запроса, а не из проверенной сессии.** В `api/src/controllers/trainer.controller.ts` (`addTrainerSessionInteraction`) стоит `const { userId, ... } = request.body;` и комментарий `// @todo check session` остался нетронутым. Гард `requireOwnTrainerSession` теперь не даёт B писать в СЕССИЮ A (сессия закрыта владением), но автор записи внутри своей же сессии — по-прежнему клиентское поле `userId` из тела, а не `request.currentUser.id`. Не эксплуатируется тем способом, который проверялся в этом тикете (чужую сессию так испортить нельзя), но сам паттерн остаётся вне явного скоупа MIA-83.
- **`CONTROL_TEST` (контрольная работа) вживую так и не была прогнана** ни в одном из трёх раундов — freemium упирается в paywall при старте, нужен платный доступ или временный обход. Механизм общий с диагностикой (тот же `joinDiagnosticSession`/`DiagnosticType.CONTROL_TEST`, тот же `result-assignments`), но стоит закрыть отдельным прогоном именно на `control-test/*`, если появится доступ.
- **Сохранение снимка доски (`POST /internal/whiteboard-blobs`) принимает `diagnosticSessionId`/`trainerSessionId` из тела без проверки владения.** Найдено по ходу анализа кода этого раунда — URL не попадает под шаблоны `requireInternalSessionOwner` (`INTERNAL_DIAGNOSTIC`/`INTERNAL_TRAINER` не матчат `/internal/whiteboard-blobs`), а `whiteboard-blob.controller.ts` принимает `userId`/`diagnosticSessionId`/`trainerSessionId` из `CreateBody` как есть. Тот же класс проблемы (session id из клиентского пейлоада без сверки владельца), что и весь этот тикет, но вне его явного скоупа (утечки ответа это не даёт — снимок доски не содержит `answer`) — стоит завести отдельным тикетом, если не заведён.

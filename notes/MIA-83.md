---
ticket: MIA-83
linear_url: https://linear.app/mia360/issue/MIA-83/pravilnyj-otvet-zadaniya-dostupen-studentu-cherez-api-spisyvanie
status: Testing
mr_url: https://gitlab.com/ai-math/ai-math-web/-/merge_requests/276, https://gitlab.com/ai-math/ai-math-web/-/merge_requests/296
updated: 2026-07-29T16:26:49.577Z
---

## Контекст

Security-баг (label `Bug`, priority High): эндпоинты выдачи заданий отдавали студенту секретные поля задания — правильный ответ (`answer`), эталонное решение (`steps`), методические `rules`/`hints`/`examples`. Это открывало списывание в тренажёре и, что серьёзнее, в диагностике — там утечка обесценивает замер уровня и позволяет накручивать mastery/прогресс.

Задача закрывалась в два захода:
- MR !276 — основной fail-closed фикс утечки полей задания по трём каналам (HTTP `/assignments*`, HTTP submission/interaction-ответы, WS `TRAINER_INTERACTION_RECEIVE`/`DIAGNOSTIC_INTERACTION_RECEIVE`).
- MR !296 — follow-up по проблеме, найденной уже в ходе QA-прогона первого фикса (наблюдение тестировщика): ни на одном канале (HTTP internal/public, WS) не проверялось владение сессией — id сессии/сабмишена берётся из клиентского пейлоада и раньше использовался без вопросов.

Оба MR в master. Независимый ре-тест MR !296 (комментарий Andrei Chepurchenko, 2026-07-29) подтвердил, что заявка «владение сессией проверяется на всех каналах» выполнена не полностью: один канал — `POST /trainer/next-assignment` (и его WS-эквивалент `trainer:session:next-assignment`) — остался без проверки владения. Через него посторонний ученик, зная (не подбирая — это UUID) `trainerSessionId` чужой тренажёрной сессии, может не только прочитать чужую сессию, но и **сменить в ней текущее задание** — у владельца сессии задание переключается под ним. Утечки секретных полей (`answer`/`steps`/`hints`/`rules`/`examples`) через этот канал нет — белый список из MR !276 держится, исходный acceptance MIA-83 не нарушен. Нарушена именно цель MR !296. Статус тикета — снова `Testing`, MR под этот новый дефект пока не заведён (проверено по списку MR проекта после !296 — ни один не упоминает `next-assignment` или MIA-83 в этом контексте).

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

**Не закрыто (найдено при перепроверке MR !296) — `POST /trainer/next-assignment` / WS `trainer:session:next-assignment`:**

Разрыв подтверждён по коду (master, `d29d1a8`) в трёх местах плюс один дополнительный, который report-гипотеза не называла явно, но который тоже играет роль:

1. **`api/src/routes/trainer.routes.ts`** — публичный роут `/trainer/next-assignment` (строки 288-306) навешивает только `preHandler: [requireTopicAccess(topicIdFromBody)]`. `requireOwnTrainerSession` отсутствует, хотя `sessionIdFrom` в guard'е умеет доставать `trainerSessionId` из `body` — сработал бы как есть, просто не подключен. Все соседние трейнер-роуты с session id в пути его несут: `/trainer/sessions/:id/interactions` (GET и POST), `/trainer/sessions/:id/submissions`, `/trainer/sessions/:trainerSessionId/submissions/:submissionId`, `/trainer/sessions/:trainerSessionId/interactions/last-response` — везде `preHandler: [requireOwnTrainerSession, requireTopicAccess(...)]`. `next-assignment` — единственный роут с `trainerSessionId` в теле, где этой пары нет.
2. **`api/src/middleware/require-session-owner.ts`** — `INTERNAL_TRAINER = /^\/internal\/trainer\/sessions\//` (строка 85) не матчит `/internal/trainer/next-assignment` (роут объявлен в `api/src/routes/internal.routes.ts:490-507`, без собственного `preHandler`), поэтому и internal-вход (через socket-gateway) остаётся без проверки — `requireInternalSessionOwner` на этом url просто ничего не делает (`if`/`if` без `else`, функция молча возвращает).
3. Внутри самого internal-вызова разрыв ещё глубже, чем в гипотезе автора отчёта: **`socket-gateway/src/services/ai-math.api.ts`**, метод `nextAssignmentTrainerSession` (строка 242), делает `api.post('/internal/trainer/next-assignment', { sessionId, userId, topicId, trainerSessionId })` **без `actingAs(userId)`** — то есть заголовок `x-user-id` вообще не отправляется на этот роут (все остальные internal-вызовы в этом файле его передают, `actingAs` встречается 15 раз в файле). Даже если бы регэксп совпал, guard'у не от чего было бы оттолкнуться.
4. **`api/src/services/trainer-session.service.ts`** — `nextAssignment(trainerSessionId: string, _user: User)` (строка 66): параметр пользователя так и называется `_user` и в теле метода не используется ни разу — последний рубеж пуст. Контроллер (`api/src/controllers/trainer.controller.ts:193-216`, `nextAssignmentInTrainerSession`) берёт `userId` из тела запроса, резолвит юзера и проверяет им только доступ к теме (`accessService.canAccessTopic(user, topicId)`) — с `trainerSessionId` этот `user` не сверяется нигде.
5. На WS-стороне (`socket-gateway/src/handlers/trainer.handler.ts:115-131`, `trainerNextAssignmentHandler`) единственная проверка — `sessionUserMemoryService.matchSessionToUser(user.id, payload.sessionId)`, а это сверка пользовательского socket/session id (`payload.sessionId`), а не `payload.trainerSessionId` — тренажёрная сессия, которую запрос реально трогает, этой проверкой не покрыта вообще.

Симметрия с диагностикой, на которую ссылается отчёт, подтверждается: `diagnostic:session:join` тоже исключён из общего internal-хука (тем же комментарием в коде: «join — исключение, может создавать сессию»), но там владение проверяется отдельно, внутри `diagnostic-session.service.ts`. У `nextAssignment` в `trainer-session.service.ts` такой внутренней проверки нет — только формальный параметр `_user`.

Важное отличие для `/trainer/join` / `/internal/trainer/join`, которые тоже не покрыты `INTERNAL_TRAINER`-регэкспом: это не тот же класс дыры. `trainerSessionService.joinSession(topicId, userId)` (строка 18) всегда сам находит сессию через `trainerSessionRepository.getOrCreate(userId, topicId, cycleNumber)` — присвоить туда чужой существующий `trainerSessionId` через тело запроса нельзя, join его даже не принимает как параметр. У диагностики join устроен иначе (принимает id и умеет присоединяться к существующей сессии), поэтому там и потребовалась явная проверка внутри сервиса. Тренажёрный join в этом смысле безопасен по конструкции, а не благодаря внешней проверке.

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

## Открытые вопросы / не блокеры

- Ретраи `POST /auth/activate` (~10 повторных запросов при входе по `/ref/<token>`, последний ловит 429) — к MIA-83 отношения не имеет, не подтверждено (по коду `/ref/[token]` дёргает `activate` один раз), не блокер — не проверять в рамках этой задачи.

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

Основа — adversarial QA-прогон на `test-1` (стенд `test-1.mia360.ru`, master содержит оба коммита: `3c5665cf` — MR !276, `cc93830d` — MR !296), три свежих ученика через `/auth/register` → `/auth/activate`.

### Подтверждённо работает

**HTTP, санитизация полей (MR !276):**
1. `/assignments?trainerId=...` и `?diagnosticId=...`, в т.ч. с попытками обхода `hasAnswers` (`1`, `true`, разный регистр, массив, дубль параметра, посторонний `includeSecrets`) — в ответе только `id/topic/task/expression/trainerUnit/kind/mlLevel/presentedSolution`, ни `answer`, ни `rules`/`steps`/`hints`/`examples`.
2. `GET /assignments/:id` — то же самое.
3. `POST`/`PUT` submissions (trainer и diagnostic) — `assignment` в ответе без секретов.
4. `/api/internal/*` без ключа → 401; `/api/system/*` → 401; Swagger → 404.

**WS, санитизация полей (MR !276):**
5. Тренажёр и диагностика: прогон чата с `:request_hint:`, `:request_steps:`, `:request_theory:`, `:request_example:` — все кадры `TRAINER_INTERACTION_RECEIVE`/`DIAGNOSTIC_INTERACTION_RECEIVE` (в т.ч. busy/error-варианты) чисты, 0 совпадений секретных ключей на сотнях кадров.

**Результаты диагностики (MR !276):**
6. `GET /diagnostic/sessions/:id/result-assignments`: своя завершённая сессия → 200, свой `answer`, без `steps/hints/rules/examples`; чужая → 403; своя незавершённая → 403 «not finished yet»; несуществующий/битый id → 404; без cookie → 401.

**Владение сессией (MR !296) — всё, КРОМЕ next-assignment (см. ниже), подтверждено на re-тесте:**
7. WS `diagnostic:session:join` с чужим `diagnosticSessionId` → 403 `session:init-error` (раньше отдавал `session:init` с чужим статусом/баллом/текущим заданием и писал в чужой дашборд-кэш) — закрыто.
8. WS `diagnostic:submission:reject`/`:accept` с чужим `submissionId` (пробовали соседние последовательные id 600-605) → эффекта нет — закрыто.
9. WS смена текущего задания (`current-assignment-change`) / запуск скоринга (`submissions:complete`) в чужой диагностической сессии → эффекта нет — закрыто.
10. Запись в чужой чат тренажёра: `POST /trainer/sessions/<чужая>/interactions` → 403; WS `trainer:interaction:send` с чужим `trainerSessionId` → в чат владельца ничего не попало — закрыто.
11. HTTP `GET /diagnostic|trainer/sessions/:id/submissions|interactions|last-response` с чужим `:id` → 403 (шесть роутов проверено) — закрыто.
12. Пара «своя сессия + чужой submissionId»: `PUT trainer/sessions/<своя>/submissions/978` → 404, не 200 — закрыто (и для diagnostic аналогично).
13. Отсутствие идентификации (нет куки на публичном роуте) → 401. Guard на стенде живой в общем случае: `GET /diagnostic|trainer/sessions/<несуществующий-uuid>/…` → 404 с сообщением самого `require-session-owner`.

### Известный непройденный дефект

**`POST /trainer/next-assignment` и WS `trainer:session:next-assignment` не проверяют владение тренажёрной сессией** — единственный найденный пробел заявки MR !296. Ученик B (свой аккаунт, своя кука, свой `topicId`) передаёт `trainerSessionId` ученика A → 200, в ответе сессия A, и её текущее задание переключается; у A задание меняется под ним.

| Шаг | Наблюдение |
|---|---|
| A решает 7x²y→7 в теме «Одночлены» | `isCorrect=true`, `score=10`; текущее задание A = `e5de36d6` |
| B шлёт `POST /api/trainer/next-assignment` с `trainerSessionId` A | 200, тело: сессия A, `currentAssignment` `07b1e9bb` (−4ab³) |
| A переподключается | текущее задание A = `07b1e9bb` — сменилось |
| A решает −4ab³→−4, B шлёт WS `trainer:session:next-assignment` с сессией A | B получает `trainer:session:init` по сессии A, задание = `27be4fec` (m²n) |
| A переподключается | текущее задание A = `27be4fec` — сменилось снова |

Второе воспроизведение на другой паре (B→C) дало тот же результат (не сдвинулось только потому, что C ничего не решил — механизм тот же).

Секреты задания в ответе не текут (`answer`/`steps`/`hints`/`rules`/`examples` отсутствуют — белый список отрабатывает), поэтому acceptance MIA-83 «студент не может получить правильный ответ» не нарушен. Нарушена именно цель MR !296 — владение сессией. `trainerSessionId` — UUID, перебором не берётся (в отличие от найденных раньше последовательных целочисленных `submissionId`), но диагностические сессии — тоже UUID и они закрыты, так что это не оправдание, а расхождение внутри одного и того же фикса. Код-локализация — см. раздел «Не закрыто» выше.

**Контрольная работа (не прогонялась вживую, механизм общий с диагностикой — важно закрыть отдельно):**
Пройти `CONTROL_TEST` целиком (нужен платный доступ или временный обход paywall — freemium блокирует старт) через тот же путь `joinDiagnosticSession`/`DiagnosticType.CONTROL_TEST`: убедиться, что секреты задания не текут ни по одному каналу и что владение сессией контрольной проверяется так же, как у диагностики (`control-test/results/[sessionId].vue`, `control-test/index.vue`).

**Соцынженерия (регресс прошлого прогона, можно не повторять подробно, но держать в уме):** прямые просьбы к Мии выдать `answer`/поле ответа не должны срабатывать — подтверждено, не срабатывает.

## Что посмотреть рядом (регресс)

- Грейдинг тренажёра и диагностики продолжает работать: `isCorrect`/`score` считаются верно (полный `assignment` по-прежнему доступен грейдингу через `/internal/assignments/:id`, `getInternalAssignmentByIdHandler`).
- UI разбора диагностики показывает «Твой ответ / Правильный ответ» на новом гейтед-эндпоинте.
- Тренажёр загружает `/assignments?trainerId=...` без `hasAnswers` и без ответов, при этом продолжает нормально показывать карточки заданий (в т.ч. v3 find-error задания, которые подмешиваются отдельным запросом `findFindErrorByTrainer`).
- Подсказки (`hints`) — server-enrichment вынесен в отдельную задачу MIA-87; здесь важно не то, что подсказки стали содержательнее, а что они по-прежнему приходят (регресс генеративного фолбэка не должен быть задет этим тикетом).
- Кэш `entities:assignment:*`/`entities:diagnostic:*` хранит сырую запись из БД; фильтрация — на выходе (`toStudentAssignment`), при чтении из кэша тоже проходит, что стоит держать в уме, если менять способ сериализации.
- **`userId` из тела запроса без сверки с вызывающим** — найдено на re-тесте, не блокер сам по себе, но стоит иметь в виду при будущих правках `next-assignment`:
  - `POST /trainer/sessions/:id/interactions` пишет `userId` интеракции из тела запроса без проверки (ученик B отправил в СВОЮ сессию интеракцию с `userId` ученика A — запись сохранилась с чужим `userId`). Гард `requireOwnTrainerSession` теперь пришивает саму сессию к вызывающему, так что чужую переписку так не испортить, но автор строки в БД остаётся клиентским полем.
  - Аналогично в `nextAssignmentInTrainerSession` (`api/src/controllers/trainer.controller.ts:193-216`) `userId` берётся из тела запроса и определяет, чей доступ к теме проверяется (`accessService.canAccessTopic`) — сейчас это не проблема безопасности отдельно от основной дыры next-assignment, но при любом будущем фиксе этого роута стоит закрывать оба поля сразу (и `trainerSessionId`, и то, что `userId` для проверки доступа тоже клиентский).

## Открытые вопросы / не блокеры

- **Главный открытый вопрос: незакрытая дыра в `/trainer/next-assignment` (HTTP + WS) требует отдельного фикса**, симметричного тому, что уже сделано для `diagnostic:session:join` — либо навесить `requireOwnTrainerSession` на публичный роут и добавить `/internal/trainer/next-assignment` в регэксп `INTERNAL_TRAINER` (плюс прокинуть `actingAs(userId)` в `ai-math.api.ts:nextAssignmentTrainerSession`, сейчас заголовок `x-user-id` для этого вызова не отправляется вовсе), либо перенести проверку владения внутрь `trainer-session.service.ts::nextAssignment` и перестать игнорировать параметр `_user`. Пока не сделано ни одного из вариантов; MR под эту находку не заведён (проверено по списку MR проекта после !296 — совпадений по `next-assignment`/MIA-83 нет).
- `/internal/trainer/join` и `/internal/trainer/ai-bonus` тоже вне регэкспа `INTERNAL_TRAINER` — автор отчёта справедливо предлагает проверять форму url, а не держать в уме список исключений, но по коду это менее срочно, чем next-assignment: `joinSession(topicId, userId)` сам создаёт/находит сессию по паре (userId, topicId) и не принимает произвольный чужой `trainerSessionId` на вход, так что join безопасен по конструкции; `ai-bonus` вообще не оперирует id сессии (только `userId`/`topicId`/`aiBonus`, `topicUserProgressService.setAiBonus`) — это не тот же класс дыры, а отдельный (более широкий и менее срочный) вопрос доверия к `userId` во внутренних вызовах в целом. Стоит держать в уме на будущее, но не приоритет по сравнению с next-assignment.
- Ретраи `POST /auth/activate` (16 последовательных POST при открытии `/ref/<token>`, последний ловит 429, вход всё равно проходит) — воспроизведены повторно на этом re-тесте, к MIA-83 отношения не имеют, не блокер — не проверять в рамках этой задачи.
- Не покрыто на этом прогоне: контрольная (`CONTROL_TEST`) вживую снова не прогонялась (freemium/paywall); fail-closed на internal без `x-user-id` снаружи не проверить (401 раньше на HMAC-уровне) — опирались на unit-тесты MR (`require-session-owner.test.ts`, 176 строк); unit-тесты локально не гоняли (нет bun/Docker); распознавание рукописи/фото не проверялось — решения отправлялись текстом.

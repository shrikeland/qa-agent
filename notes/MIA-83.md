---
ticket: MIA-83
linear_url: https://linear.app/mia360/issue/MIA-83/pravilnyj-otvet-zadaniya-dostupen-studentu-cherez-api-spisyvanie
status: Ready for Test
mr_url: https://gitlab.com/ai-math/ai-math-web/-/merge_requests/276
updated: 2026-07-20T16:49:22.694Z
---

## Контекст

Баг/security-фикс (label `Bug`, High): правильный ответ задания (`answer`), а заодно эталонное решение (`steps`), методические подсказки (`hints`), правила (`rules`) и примеры (`examples`) были доступны студенту через API - можно было списать в тренажёре и, что хуже, в диагностике, где это ломает честность замера уровня. Response-схема поля не фильтровала, эндпоинты отдавали Prisma-модель Assignment почти целиком.

Код - MR !276, ветка `feat/MIA-83-assignment-exposure`, читается по коммиту `b2fd6631e37662bc48eee5dfb9c324822058b68a` (финальное состояние ветки после сквош-мержа, оригинальная remote-ветка уже удалена постмерж-клинапом - это нормально для проекта, дерево на этом SHA полностью соответствует смёрженному коду).

При расследовании (комментарий к тикету от 2026-07-20 09:58) нашли, что канал утечки не один: помимо HTTP `/assignments*` секрет тёк ещё и через WebSocket `TRAINER_INTERACTION_RECEIVE` - `POST /internal/trainer/sessions/:id/interactions` тянул `assignment` через `getById` со всеми полями, а socket-gateway ретранслировал это в браузер на каждый AI-ответ. Итоговый фикс закрывает все три канала.

## Что было сломано

1. **`GET /assignments/:id`** и **`GET /assignments?...&hasAnswers=1`** (`api/src/controllers/assignment.controller.ts`, публичные под `requireAuth`) отдавали `answer`/`rules`/`steps`/`examples`/`hints` любому авторизованному пользователю - клиентский флаг `hasAnswers` реально переключал API на более богатую форму ответа, то есть студент сам решал, получать ли секреты.
2. **HTTP submission/interactions:** `POST /internal/trainer/sessions/:id/interactions` (`addTrainerSessionInteraction`) возвращал `assignment` из `trainerInteractionsRepository.getById` без фильтрации полей - полный Prisma-объект с `answer` уходил в ответ, который socket-gateway затем ретранслировал студенту как есть.
3. **WebSocket `TRAINER_INTERACTION_RECEIVE`** (тренажёр, 9+ точек emit: успешный AI-ответ, busy, error, critical error, bad recognition, bad review) и его диагностический аналог **`DIAGNOSTIC_INTERACTION_RECEIVE`** - тот же самый незасанитайженный `assignment` из п.2 ретранслировался в браузер на каждый ответ Мии, включая busy/error-хелперы, которые вообще не подвергались чистке.
4. Debug-оверлей на фронте: во `frontend/pages/trainer/[topicId].vue`, `frontend/pages/diagnostic/[[diagnosticId]].vue` и `frontend/pages/control-test/index.vue` был спрятанный блок `v-if="miaClicks > 3"` (4+ кликов по аватару Мии), дампивший в DOM весь `currentAssignment` (с `answer`) - и в диагностике/контроле ещё и `currentLastSubmission`. Не эксплуатировался через сам баг API, но был вторым независимым способом достать ответ прямо из клиента.
5. Результаты диагностики/контрольной раньше грузили ассайнменты с `hasAnswers=1` через обычный `/assignments` - то есть правильный ответ на **любое** задание диагностики (включая ещё не пройденные/не относящиеся к текущей сессии) был доступен, а не только на решённые в конкретной завершённой сессии.

## Что изменено (по коду)

**Fail-closed белый список вместо denylist.** Вместо вырезания отдельных полей из полной модели ввели явный whitelist - новое поле в Prisma-схеме Assignment по умолчанию НЕ попадёт студенту, пока его не добавят в whitelist осознанно.

- `api/src/utils/assignment-privacy.ts` (новый файл): `toStudentAssignment(assignment): IAssignmentPublic` - отдаёт только `id`, `topic.id`, `task`, `expression`, `trainerUnit`. `toStudentAssignmentWithAnswer(assignment): IAssignmentResultPublic` - то же плюс `answer`, используется исключительно в геймированном эндпоинте результатов диагностики (см. ниже), `steps/hints/rules/examples` не отдаёт и он.
- `socket-gateway/src/helpers/assignment-privacy.helper.ts` (новый файл): своя копия `toStudentAssignment` на стороне socket-gateway - `id`, `topicId`, `task`, `expression`, `steps: []` (принудительно пустой массив, не `undefined`). Используется в `handler.helper.ts` в busy/error/critical-error эмитах (`sendTrainerCriticalErrorResponse`, `sendTrainerBusyResponse`, `sendDiagnosticBusyResponse` и т.п.) - раньше эти хелперы вообще не трогали `assignment`.
- **HTTP `/assignments`, `/assignments/:id`** (`api/src/controllers/assignment.controller.ts`): `getAssignments` теперь всегда прогоняет через `toStudentAssignment` независимо от query-параметра `hasAnswers` (флаг просто игнорируется - "the client-supplied hasAnswers flag no longer selects a richer shape"). `getAssignmentByIdHandler` (публичный роут) - `includeSecrets=false` → `toStudentAssignment`; полная форма с `answer/hints/rules/steps/examples` осталась только за `getInternalAssignmentByIdHandler` на `GET /internal/assignments/:id` под `requireInternalKey` (для грейдинга).
- **HTTP submission/interactions** (`api/src/controllers/trainer.controller.ts`): `addTrainerSessionInteraction`, `createTrainerSubmission`, `updateTrainerSubmission` теперь оборачивают `assignment` в ответе через `toStudentAssignment` перед отправкой - это и есть источник данных, который socket-gateway ретранслирует в WS дальше, так что закрытие на этом уровне автоматически закрывает и WS-канал успешного ответа (`registerTrainerAiInteraction`/`registerDiagnosticAiInteraction` в `socket-gateway/src/helpers/handler.helper.ts` просто пробрасывают то, что вернул API).
- **WebSocket busy/error-хелперы**: все функции в `socket-gateway/src/helpers/handler.helper.ts`, которые раньше эмитили `assignment` напрямую из входных данных (`sendTrainerCriticalErrorResponse`, `sendTrainerBusyResponse`, `sendDiagnosticBusyResponse`, аналогичные диагностические), теперь прогоняют его через локальный `toStudentAssignment` из `assignment-privacy.helper.ts`.
- **Диагностика, submission-result эмит** (`socket-gateway/src/handlers/diagnostic/interactions.handler.ts`, `emitSubmissionResult` и обработчик пустого решения в `diagnosticSubmissionPosted`): `assignment` в эмите `DIAGNOSTIC_INTERACTION_RECEIVE` тоже прогоняется через `toStudentAssignment`.
- **Новый геймированный эндпоинт результатов диагностики**: `GET /diagnostic/sessions/:id/result-assignments` (`api/src/routes/diagnostic.routes.ts` → `getDiagnosticSessionResultAssignments` в `api/src/controllers/diagnostic.controller.ts`). Логика:
  - session не найдена → **404**;
  - `session.userId !== currentUser.id` (чужая сессия) → **403 Forbidden**, к сабмишенам за ответом даже не идёт (`resultAssignmentsBySession` не вызывается);
  - `!session.finishedAt` (сессия не завершена) → **403**, тоже без похода за сабмишенами. Важно: гейт именно на `finishedAt`, не на `scoredAt` - сессия с неразрешённым `REVIEW_ERROR`-сабмишеном может остаться нескорированной, но студенту всё равно показывается как готовая к разбору, поэтому 403 не должен блокировать этот случай;
  - иначе → берёт `diagnosticSubmissionRepository.resultAssignmentsBySession(id)` (только сабмишены со статусом `SCORED`/`SCORE_CONFIRMED`, `deletedAt: null`), дедупит по assignment (несколько попыток на одно задание → одна карточка, побеждает свежейший `createdAt`) и отдаёт список через `toStudentAssignmentWithAnswer` - **только по заданиям, реально решённым в этой сессии**, `steps/hints/rules/examples` по-прежнему не отдаются.
  - Старый внутренний эндпоинт `GET /internal/diagnostic/sessions/:id/assignments` (`getDiagnosticSessionAssignments`) удалён вместе со старым способом получать список заданий диагностики с ответами через `hasAnswers=1`.
- **Мёртвый код почищен по месту**: `IAssignmentResultPublic` добавлен в `api/src/schemas/content.schemas.ts`; в `docs/educational-process/diagnostic.md` таблица internal-эндпоинтов и схема `initResultsPage` обновлены под новую картину (12 → 11 internal-эндпоинтов).

**Фронтенд перестал грузить/хранить ответы:**
- `frontend/stores/trainer-session.ts`: `/assignments?trainerId=...&hasAnswers=1` → `/assignments?trainerId=...` (без флага); тип `AssignmentExtended` (с `answer/hints/rules/steps/examples`) убран из стора, `topicAssignments`/`currentAssignment` теперь типа `AssignmentPublic`.
- `frontend/stores/diagnostic-session.ts`: аналогично убран `hasAnswers`; метод `reloadAssignmentsWithAnswers()` переименован в `reloadAssignments()` (без секретов); `initResultsPage(diagnosticSessionId)` больше не принимает `diagnosticId` и вместо `/assignments?...&hasAnswers=1` дёргает новый `GET /diagnostic/sessions/:id/result-assignments` через `Promise.all(...).catch(() => [])` - изолированно от `submissions`/`interactions`, потому что сразу после `finishSession()` страница результатов может открыться раньше, чем асинхронный скоринг проставит `finishedAt` (тогда эндпоинт легитимно вернёт 403) - карточки просто рендерятся без раскрытия правильного ответа до повторной загрузки, а не роняют всю страницу.
- `frontend/types/index.ts`: `AssignmentExtended` удалён целиком; `AssignmentPublic` дополнен полем `trainerUnit`.
- **Debug-оверлей (`miaClicks > 3`, 4+ кликов по аватару Мии) убран** из трёх страниц: `frontend/pages/trainer/[topicId].vue` (дампил `trainerSessionStore.currentAssignment`), `frontend/pages/diagnostic/[[diagnosticId]].vue` и `frontend/pages/control-test/index.vue` (дампили `currentAssignment` + `currentLastSubmission`). В `trainer/[topicId].vue` заодно из computed `trainerProblem` убрано поле `answer: trainerSessionStore.currentAssignment.answer`.
- Реально используемые фронтом поля assignment после урезания - `id`, `task`/`instruction`, `expression`/`formula`, `topic.id`, `trainerUnit` - именно они и остались в белом списке `IAssignmentPublic`/`AssignmentPublic`, так что урезание не должно сломать отрисовку карточки задания.

## Как перепроверить

**HTTP `/assignments*` (каналы 1 и 5 из «Что было сломано»):**
1. DevTools → Network, открыть тему в тренажёре. В ответе `GET /assignments?trainerId=...` (запрос теперь без `hasAnswers`) - в каждом элементе массива нет `answer`, `steps`, `hints`, `rules`, `examples`; есть только `id`, `topic.id`, `task`, `expression`, `trainerUnit`.
2. Дёрнуть `GET /assignments/:id` (взять любой id задания из п.1) - тот же белый список, без секретов.
3. Попробовать вручную добавить `?hasAnswers=1` к `GET /assignments?...` - флаг должен игнорироваться, секретов в ответе всё равно нет.

**HTTP submission/interactions (канал 2):**
4. В тренажёре отправить решение на проверку (whiteboard/фото) - в ответе `POST .../submissions` (createTrainerSubmission/updateTrainerSubmission) поле `assignment` без `answer`.

**WebSocket (канал 3) - основной сценарий из тикета:**
5. Открыть тренажёр, написать что-нибудь в чат Мии (или отправить фото решения). В кадре `TRAINER_INTERACTION_RECEIVE` (и в диагностике - `DIAGNOSTIC_INTERACTION_RECEIVE`) поле `assignment` должно быть без `answer`/`hints`/`rules`/`examples`; `steps` - пустой массив `[]`, не `undefined`.
6. Спровоцировать "занято" (быстро отправить два сообщения подряд, пока первое ещё обрабатывается) и ошибку распознавания (например, отправить пустое решение) - в обоих случаях в эмите тот же белый список без секретов. Раньше busy/error-хелперы вообще не чистили `assignment` - это отдельная регрессия, которую стоит проверить прицельно, не только основной happy path.

**Debug-оверлей (4-й канал, не связан с API):**
7. В тренажёре, на странице диагностики и на странице контрольной работы - кликнуть 4+ раз по аватару Мии в чате. Раньше при этом под чатом появлялся блок с сырым JSON текущего задания (и в диагностике/контроле - ещё и последнего сабмишена), включая `answer`. Ожидаемо: блок больше не появляется ни при каком количестве кликов.

**Новый эндпоинт результатов диагностики (владение + завершённость сессии):**
8. Пройти диагностику до конца, открыть страницу результатов своей сессии. В Network должен быть запрос `GET /diagnostic/sessions/:id/result-assignments`, статус 200, в ответе - только задания, реально решённые в этой сессии, каждое с `answer`, без `steps/hints/rules/examples`. На странице отображается правильный ответ по своим решённым заданиям.
9. Подставить в URL/запрос чужой `diagnosticSessionId` (сессия другого пользователя) - ожидаемо отказ (403 по коду `getDiagnosticSessionResultAssignments`; если фактический код в ответе браузера отличается - зафиксировать расхождение, в коде это `reply.code(403).send({ message: 'Forbidden' })`).
10. Открыть страницу результатов сразу после старта диагностики, не проходя её (сессия не завершена, `finishedAt` пуст) - тот же эндпоинт должен отказать (403 по коду, `{ message: 'Diagnostic session is not finished yet' }`); при этом сама страница не должна падать целиком - `submissions`/`interactions` по-прежнему подгружаются, карточки просто без раскрытого ответа (см. п.5 «Что изменено» про `.catch(() => [])`).
11. Пограничный случай: сессия помечена `finishedAt`, но не `scoredAt` (например, есть неразрешённый REVIEW_ERROR-сабмишен) - результаты всё равно должны открываться (гейт по `finishedAt`, не по `scoredAt`), без 403.
12. Несколько попыток решения одного и того же задания в рамках сессии (пересдал ответ) - на странице результатов должна быть одна карточка на задание (дедуп по последней попытке), а не дубли.

**Регрессия (acceptance-критерий «грейдинг продолжает работать как раньше»):**
13. Полный прогон тренажёра: чат с Миа (подсказки/примеры/шаги по-прежнему работают со стороны логики, просто больше не видны студенту сырым JSON), отправка фото/whiteboard-решения, оценка приходит корректно.
14. Полный прогон диагностики: 5+ заданий, чат по каждому, распознавание работает, сессия завершается, страница результатов открывается и показывает корректные баллы/ответы.
15. `GET /internal/assignments/:id` (через socket-gateway - это внутренний вызов, руками не подёргать без internal-ключа, но стоит убедиться, что грейдинг не деградировал) - должен по-прежнему получать полный ассайнмент с `answer/steps/hints/rules/examples`, иначе автопроверка решений сломается.

## Аналитика/API

Новый публичный (но геймированный) эндпоинт, который стоит явно взять на заметку: **`GET /diagnostic/sessions/:id/result-assignments`** - решённые задания текущей сессии с правильным ответом, доступен только владельцу завершённой сессии (см. п. 8-12 выше). Заменяет прежний способ получать assignments-с-ответами через `/assignments?...&hasAnswers=1` на странице результатов.

Удалён internal-эндпоинт `GET /internal/diagnostic/sessions/:id/assignments` (`getDiagnosticSessionAssignments`) - если у кого-то в скриптах/интеграциях остались обращения к нему, они перестанут работать; из документации (`docs/educational-process/diagnostic.md`) count internal-роутов обновлён с 12 до 11.

Отдельных аналитических событий (analytics-events) под эту фичу не заводилось - изменения точечные, в форме API-ответов и WS-эмитов, отдельного трекинга не требуют.

## Открытые вопросы

- `GET /diagnostic|trainer/sessions/:id/submissions|interactions` - по коду не проверяют владение сессией, то есть студент теоретически может посмотреть чужие сабмишены/чат (тексты, не секреты задания). Отдельный security follow-up автора, не блокер для этого тикета, но стоит иметь в виду при смежном тестировании.
- Нет guard на отправку решения в уже завершённую сессию (answer-informed re-answer: студент узнал правильный ответ через `result-assignments` и переотправляет решение в ту же/связанную сессию). Отдельный security follow-up, не блокер.
- Мёртвый `getDiagnosticAssignments` в socket-gateway (не путать с уже удалённым internal-роутом `getDiagnosticSessionAssignments` на API) - уборка по касанию, не блокер, чисто технический долг.
- Смежная проблема с подсказками (`hints` в чате тренажёра больше не обогащались на клиенте после того, как фронт перестал грузить полный ассайнмент) устранена отдельным MR - к этому тикету отношения не имеет, специально проверять здесь не нужно.

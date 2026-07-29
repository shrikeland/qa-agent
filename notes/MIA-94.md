---
ticket: MIA-94
linear_url: https://linear.app/mia360/issue/MIA-94/miya-ne-ponimaet-moj-vopros-i-uporno-prosit-poschitat-d
status: Ready for Test
mr_url: https://gitlab.com/ai-math/ai-math-web/-/merge_requests/304
updated: 2026-07-29T08:48:59.210Z
---

## Контекст

Баг (label `Bug`): в тренажёре ученик уже посчитал дискриминант в чате, отправил его на проверку, а Мия после этого продолжала просить «посчитать Д» и не отвечала на прямой вопрос про поиск корней. По комментарию разработчика (Kirill Verzhitskii, единственный комментарий в задаче) причина в том, что после каждой отправки решения на проверку Мия теряла весь предыдущий разговор — из-за этого она не только не видела, что сказала проверка, но и не помнила уже отвеченного в чате.

Код фикса — MR !304 (ветка `fix/MIA-94-chat-context-after-grading`, смёржена коммитом `680930bb`, ветка удалена — обычная практика проекта). Проверялось текущее состояние `master` (`680930bb`), в котором фикс уже есть.

## Что было сломано

Эндпоинт `GET /internal/trainer/sessions/:trainerSessionId/interactions/last-response` (`getLastTrainerResponse`, `api/src/controllers/trainer.controller.ts`) до фикса дергал `trainerInteractionsRepository.getLastResponse(trainerSessionId)` — запрос брал **ровно одну** самую свежую AI-реплику сессии (`take: 1`, `orderBy: createdAt desc`) и отдавал `lastId = row.meta.aiResponseId ?? null` прямо из неё.

Проблема: `meta.aiResponseId` присутствует только у реплик из обычного чат-диалога (`socket-gateway/src/handlers/pools/conversation.handlers.ts:60-66`, `aiResponseId: filteredChunk.responseId`). У реплики проверки решения (`trainerSubmissionPosted`, `socket-gateway/src/handlers/trainer.handler.ts`, регистрация ai-интеракции с `submission`) и у приветствия это поле фактически всегда пустое:
- у ошибочных/некорректных проверок (`sendTrainerBadReviewResponse`, `sendTrainerErrorResponse`, `helpers/handler.helper.ts`) `meta` вообще без `aiResponseId`;
- у успешной проверки код формально пишет `aiResponseId: recognitionResult.responseId` (`trainer.handler.ts`, `registerTrainerAiInteraction` в `trainerSubmissionPosted`), но `recognitionResult.responseId` берётся как `chunk.response?.id` (`socket-gateway/src/sockets/helpers/recognition-response.helper.ts:19,53,75`), а сам чанк, который реально прилетает на событие `RESPONSE_OUTPUT_TEXT_DONE`, по типу `{ type; delta?; text? }` (`socket-gateway/src/handlers/pools/recognition.handlers.ts:59-65`) поля `response` не содержит вовсе — то есть на практике это значение всегда `undefined`.

Итог: как только последним событием в сессии становится отправка на проверку (а после отправки на проверку это всегда так), «последняя AI-реплика» — это реплика проверки без `aiResponseId`, и `lastId` возвращался `null`. `previousResponseId = null` в `trainerInteractionsSendHandler` означало, что следующий промпт Мии стартовал OpenAI-тред заново, без единой реплики из прошлого диалога — отсюда и полная потеря контекста, и переспрос уже посчитанного. Отдельно от этого в промптах не было вообще никакого канала, через который текст самой проверки («не хватает вывода о числе корней») попадал бы Мии — даже когда anchor каким-то образом сохранялся, она не знала, что именно сказала проверка.

## Что изменено (по коду)

1. `api/src/repositories/trainer-interactions.repository.ts` — `getLastResponse` (take: 1) заменён на `getRecentAiInteractions(trainerSessionId, take = 20)`: теперь возвращаются до 20 последних AI-реплик, а не одна.
2. `api/src/services/trainer-chat-context.ts` (новый) — `buildTrainerChatContext(recentAiDesc)` идёт по этим строкам от новых к старым:
   - `responseIdOf(row.meta)` проверяет, есть ли валидный непустой `aiResponseId`;
   - первая строка с `aiResponseId` фиксируется как `lastId` (якорь OpenAI-треда) — цикл на ней останавливается (`break`);
   - все более свежие строки, которые встретились раньше (т.е. более новые), с `submissionId !== null` и непустым `text`, — это вердикты проверки, отправленные после последнего чат-ответа Мии; их текст копится в `gradingsSince`;
   - результат ограничен `MAX_GRADINGS_SINCE = 3` и разворачивается в хронологический порядок (`reverse()`), так как строки шли от новых к старым.
3. `api/src/controllers/trainer.controller.ts`, `getLastTrainerResponse` — теперь вызывает `getRecentAiInteractions` + `buildTrainerChatContext` и отдаёт `{ lastId, gradingsSince }` вместо старого `{ lastId }`.
4. `socket-gateway/src/services/ai-math.api.ts` — `getTrainerLastAiResponse` переименован в `getTrainerChatContext`, бьёт в тот же эндпоинт, но возвращает типизированный `TrainerChatContext { lastId, gradingsSince }` (новый тип в `socket-gateway/src/types.ts`) вместо голой строки; `gradingsSince` защитно приводится к `[]`, если сервер не прислал массив.
5. `socket-gateway/src/handlers/trainer.handler.ts`, `trainerInteractionsSendHandler` — единый вызов `getTrainerChatContext` в начале обработчика даёт и `previousResponseId` (используется как раньше — передаётся во все билдеры промптов), и новый `gradingsSince`. `gradingsSince` пробрасывается только в `trainerPrompt.createConversationPrompt(...)` — то есть только в свободную/базовую ветку чата (финальный `else`). Ветки по кнопкам-маркерам (`isRequestHint`, `isRequestTheory`, `isRequestExample`, `isRequestSteps`, `isRequestResults`) вызывают свои билдеры промптов, ни один из них параметр `gradingsSince` не принимает.
6. `socket-gateway/src/services/trainer.prompt.ts`, `createConversationPrompt` — добавляет блок `<checks>`:
   - `gradingsSinceUserText(gradingsSince)` — пронумерованный список вердиктов проверки, экранированный от `</checks>`-инъекций, кладётся в **user-role** текст перед текстом ученика (тем же приёмом, что и существующий `<board>`-блок) — как данные, не инструкции;
   - `gradingsSinceInstruction(gradingsSince)` — одна строка в system-инструкции: «в сообщении есть блок `<checks>` — ученик уже отправлял решение и видел эти замечания; считай их сказанными, не проси заново, помогай закрыть то, чего не хватает».
   Обе функции возвращают пустую строку, если `gradingsSince` пуст — обычный чат без свежих проверок не меняется побайтово (это отдельно проверяется тестом `trainer.prompt.context.test.ts`).

## Как перепроверить

Сценарий разработчика — тренажёр, задание «Найди дискриминант и посмотри, сколько корней», `x^2 - 6x + 9 = 0`:

1. Написать в чате обычным текстом что-то, что даст Мие ответить (например обсудить ход решения) — важно получить хотя бы одну **чат**-реплику Мии, а не сразу отправлять на проверку, иначе `lastId` будет `null` ещё до всякого фикса (первый ход сессии не anchored и раньше).
2. Отправить на проверку неполный ответ «D = 0» — проверка должна попросить дописать вывод о числе корней.
3. В чате именно текстом (не кнопкой «Подсказка» — см. ниже, почему это важно) спросить «подскажи как найти корни» — Мия должна ответить, опираясь на то, что D уже найден, и на замечание проверки, а не просить пересчитать D.
4. Спросить «я же его уже посчитал?» — ответ должен подтверждать найденное значение D, а не переспрашивать его заново.
5. Повторить связку из двух отправок на проверку подряд без чат-сообщения между ними (например, «D = 0», затем сразу «D = 0, корень один» — тоже неполно или с ошибкой) — следующее сообщение в чате должно учитывать **оба** вердикта проверки, а не только последний (проверяет `MAX_GRADINGS_SINCE`/накопление, а не просто последний факт).
6. Ориентир по формулировкам из задачи: до фикса Мия отвечала «напиши своё значение D»; после — что-то в духе «ты уже нашла D, но не хватает вывода про число корней; при D = 0 корень один».

Дополнительно стоит явно отметить в отчёте, была ли команда «подскажи как найти корни» отправлена именно как свободный текст в чат — если её по ошибке протестировать через нажатие кнопки «Подсказка», сработает другая ветка кода (`isRequestHint` → `createHintRequestPrompt`), которая **не** получает `gradingsSince` (см. «Что посмотреть рядом»).

## Что посмотреть рядом (регресс)

- Кнопки «Подсказка» / «Теория» / «Пример» / «Шаги» / «Результаты» сразу после проверки: anchor (`previousResponseId`) для них теперь тоже чинится (общий фикс в начале хендлера), но явного блока `<checks>` с текстом проверки в их промптах нет — `createHintRequestPrompt`/`createTheoryRequestPrompt`/`createExampleRequestPrompt`/`createStepsRequestPrompt`/`createTrainerSessionResultsPrompt` параметр не принимают. Стоит один раз проверить, что нажатие «Подсказка» сразу после «неполной» проверки не выглядит так, будто Мия не в курсе замечания (тред она не потеряет, но явной ссылки на текст проверки может не быть).
- Обычная проверка решения в целом (без чат-контекста): `trainerSubmissionPosted` в этом MR не тронут в части логики оценки, начисления баллов, `boardMarks`, `softCardPromptTrigger`/`freemiumPaywallTrigger` — стоит одним прогоном подтвердить, что верный/неверный ответ по-прежнему обрабатывается штатно.
- Задания типа «найди ошибку» (`isFindError`) — `gradingsSinceInstruction`/`gradingsSinceUserText` встраиваются в тот же `createConversationPrompt`, что и текст про find_error; стоит один раз убедиться, что оба блока (`<checks>` и find_error-инструкция) не конфликтуют в одном промпте на реальном диалоге.
- Сессии с длинной историей подряд идущих проверок: `getRecentAiInteractions` берёт только 20 последних AI-строк, а `gradingsSince` капается тремя последними вердиктами (`MAX_GRADINGS_SINCE = 3`) — при марафоне из существенно бОльшего числа подряд идущих отправок на проверку без чата между ними самые старые вердикты Мие не долетят. Маловероятный сценарий, но стоит держать в уме.
- Первое сообщение новой сессии (приветствие) — anchor по-прежнему `null` на самом первом ходу (ожидаемо, не регресс); `contextBlock` (студенческая память через `studentContextService`) в этом случае заполняется так же, как и раньше.
- Diagnostic-поток (не trainer) этим MR не затронут по логике — общий файл `socket-gateway/src/types.ts` менялся только добавлением нового интерфейса `TrainerChatContext`, без изменений диагностических типов; стоит один раз убедиться, что диагностический чат/греетинг не пострадал.

## Открытые вопросы / код-наблюдение (не блокер)

- В `trainerSubmissionPosted` (`socket-gateway/src/handlers/trainer.handler.ts`) по-прежнему явно пишется `aiResponseId: recognitionResult.responseId` в meta реплики проверки, хотя по факту (см. «Что было сломано») это значение всегда `undefined`, потому что чанк `RESPONSE_OUTPUT_TEXT_DONE` не содержит поля `response`. Именно из-за этого `buildTrainerChatContext` сейчас корректно «перепрыгивает» через реплики проверки при поиске anchor'а. Если это когда-нибудь изменится (поле реально начнёт приходить), новый механизм молча сломается тем же образом, что и старый: `lastId` заанкорится на response id самой проверки, а не реального чат-диалога. Не блокер сейчас, но связка неочевидная и не задокументирована явно в коде.
- `gradingsSince` не пробрасывается в hint/theory/example/steps/results-ветки — фикс закрывает именно свободный текстовый чат (это соответствует акцептансу и комментарию разработчика), но если продукт захочет, чтобы кнопочные сценарии тоже явно ссылались на вердикт проверки, потребуется отдельная доработка.
- `MAX_GRADINGS_SINCE = 3` — внутренний лимит автора фикса, в тикете/MR-описании явно не обсуждается; обоснование только в комментарии к коду («anything older is already reflected in the threaded history the anchor restores»). Не блокер, просто отметил как решение без явного acceptance criteria по числу.

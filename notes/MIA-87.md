---
ticket: MIA-87
linear_url: https://linear.app/mia360/issue/MIA-87/podskazki-v-chate-trenazhyora-authored-podskazki-podstavlyat-na
status: Ready for Test
mr_url: https://gitlab.com/ai-math/ai-math-web/-/merge_requests/298
updated: 2026-07-29T03:46:12.214Z
---

## Контекст

Баг (label `Bug`): кнопка «Подсказка» в тренажёре не выдавала методические (авторские) подсказки из справочника, всегда генерируя подсказку сама. История разбирательства — это уже два последовательных фикса на одну и ту же тему, но с разными причинами:

1. **MR !278** (первый фикс) переписал `socket-gateway/src/handlers/trainer.handler.ts` так, чтобы authored-подсказки для промпта дотягивались с сервера через `aiMathApi.getAssignment(...)`, а не искались в клиентском payload (который после fail-closed сериализации MIA-83 их не содержит). Тестировщик прогнал маркерный тест и поставил **FAIL** по первому пункту acceptance — авторские подсказки всё ещё не подставлялись. Следом выдвинул гипотезу: возможно, дело не в коде, а в протухшем образе `socket-gateway` на стенде (задеплоен коммит до фикса).
2. Автор (Andrei Chepurchenko) проверил: гипотеза про недоехавший деплой **не подтвердилась** — коммит фикса от 21 июля, а master с тех пор деплоился на тест десятки раз, так что стенд был актуален. Но гипотеза не пропала зря — она сузила поиск до одного места.
3. Настоящая причина оказалась глубже, во внутренней (`internal`) выдаче задания на api: она отдавала подсказки **без поля `level`**, а подбор подсказки в `hintTextForLevel` матчит именно по `level`. Совпадений не было ни на одном уровне, ни на одном задании — то есть даже с server-side фетчингом из MR !278 подсказка всё равно не находилась, и Мия каждый раз генерировала её сама. Это тот же класс дефекта, что описан в самой задаче про публичную схему (там срезался `level`), только он остался ещё и во внутреннем канале, который MIA-87 должна была задействовать.
4. Причина исправлена **MR !298** «MIA-87: методические подсказки не доезжали до промпта» (смёржен). Комментарий Kirill Verzhitskii от 2026-07-29 подтверждает воспроизведение до и после правки на живом стенде реальным сокет-запросом с маркерными текстами — до фикса ответ Мии маркер не содержал, после правки пришёл дословно:
   > МАРКЕР-СОРОК-ДВА: посмотри на выражение 42x и спроси себя, что общего у слагаемых. :HINT:

Важно: лестница подбора подсказки по уровню нажатий (MIA-77, `hintLevelForPress` / `hintTextForLevel` в `socket-gateway/src/services/trainer.prompt.ts`) за весь цикл (оба фикса) ни разу не менялась и всегда работала правильно — ломалась исключительно доставка данных до неё, сначала на уровне payload (MR !278), затем на уровне формы данных, которые этот payload вёз (MR !298).

## Что было сломано

`api/src/controllers/assignment.controller.ts`, `handleGetAssignmentById` (внутренний путь, `includeSecrets = true`, роут `/internal/assignments/:id`, используемый `aiMathApi.getAssignment`): подсказки из `assignment.hints` прокидывались в ответ без нормализации поля `level`. Если у сохранённого объекта подсказки не было числового `level` (или структура была вообще другой), в ответе он либо отсутствовал, либо был не тем, что ожидает `hintTextForLevel`. `hintTextForLevel(hints, level)` ищет подсказку строгим сравнением `h.level === level` — без `level` совпадение невозможно ни для одного уровня, и `createHintRequestPrompt` (`socket-gateway/src/services/trainer.prompt.ts`) всегда получал `authored = null`, уходя в генеративный фолбэк даже для заданий с полностью заполненным справочником подсказок. Именно поэтому MR !278 (сам по себе корректный — server-side fetch действительно нужен) не закрыл дефект: он чинил канал доставки, но не форму данных, которые по этому каналу приезжали.

## Что изменено (по коду)

`api/src/controllers/assignment.controller.ts`, `handleGetAssignmentById` (строки ~86–94):

```ts
hints: assignment.hints
  ? (assignment.hints as any[]).map((item, i) => ({
      // Position stands in for a missing level because the editor keeps the list dense and
      // 1-based; without a level the hint ladder matches nothing and falls back to generation.
      level: typeof item.level === 'number' ? item.level : i + 1,
      title: item.title ?? '',
      text: item.text ?? '',
    }))
  : [],
```

Каждая подсказка теперь гарантированно получает числовой `level`: берётся `item.level`, если это число, иначе — позиция в массиве +1 (список в редакторе плотный и 1-based). `api/src/schemas/content.schemas.ts` (`IAssignmentWithAnswerPublic.hints`) закрепляет это типом — `level: number` обязателен, с комментарием «hint ladder addresses a hint by its level, so a hint without one can never be found».

`socket-gateway/src/handlers/trainer.handler.ts`, ветка `isRequestHint` в `trainerInteractionsSendHandler` (не изменилась в механике server-side fetch из MR !278, но в MR !298 добавлена наблюдаемость):

```ts
const hintLevel = hintLevelForPress(repeatCount);
logger.info({
  msg: 'trainer hint requested',
  assignmentId: payload.interaction.assignment?.id,
  level: hintLevel,
  authoredFound: hintTextForLevel(authoredHints, hintLevel) !== null,
});
```

Теперь по каждому запросу подсказки в лог пишется запрошенный уровень и найдена ли для него авторская подсказка — тот самый разрыв, из-за которого разбор пришлось вести маркерными прогонами, а не по логам, закрыт.

`socket-gateway/src/services/trainer.prompt.ts`, `createHintRequestPrompt`: фолбэк на клиентский payload убран полностью. Было (по старому коду) `authoredHints ?? interaction.assignment?.hints`; стало:

```ts
// Only the server-fetched hints: the interaction's own assignment comes from the client,
// which no longer carries hints at all.
const authored = hintTextForLevel(authoredHints, level);
```

Это подтверждено и тестом `socket-gateway/src/services/__tests__/trainer.prompt.test.ts` → «ignores hints riding on the client assignment». `hintTextForLevel` и сама лестница уровней (`hintLevelForPress`, `MAX_HINT_LEVEL = 3`) не менялись.

Секреты задания клиенту по-прежнему не утекают: `authoredHints` живёт только в `instructions` промпта, не в данных, которые уходят в сокет ученику (это подтверждает и тест «puts the authored hint fetched from the API into the hint prompt, without leaking it to the prompt assignment», где `captured.prompt.assignment.hints` пуст).

Тесты переписаны на серверный путь: `api/src/controllers/__tests__/assignment.controller.test.ts` добавил «internal GET /internal/assignments/:id keeps the level on every hint» и «numbers a hint saved without a level by its position» (регресс именно на текущий баг); `socket-gateway/src/handlers/__tests__/trainer.handler.test.ts` — «feeds the hint prompt with the authored hints it fetched from the api» (стык хендлер → промпт) и «falls back to a generated hint when assignment enrichment fails».

## Как перепроверить

1. Найти или подменить у задания в справочнике авторские подсказки уровней 1-3 на заведомо узнаваемые маркерные тексты (как в прогоне выше — «МАРКЕР-СОРОК-ДВА: …»).
2. Открыть задание в тренажёре **свежим** учеником (счётчик повторных нажатий с 1) и нажать «Подсказка» — Мия должна передать маркерный текст уровня 1 дословно или близко к нему (может адаптировать формулировку под диалог, но не добавлять шагов сверх маркера).
3. Повторные нажатия — уровни 2 и 3 (лестница MIA-77 не менялась, но само наличие маркеров на всех трёх уровнях стоит перепроверить).
4. Обычный прогон без маркеров: задание, у которого в справочнике реальные (не маркерные) авторские подсказки — убедиться, что подсказка выдаётся по смыслу та же, что в справочнике.
5. Задание **без** авторских подсказок — Мия по-прежнему генерирует подсказку сама (регресс фолбэка не задет).
6. Полезно проверить по логам gateway строку `trainer hint requested ... level: N, authoredFound: true/false` — она должна сразу показывать, найдена ли авторская подсказка, без необходимости гонять маркерные прогоны при следующих разборах.

Уточнять версию образа `socket-gateway` на стенде больше не требуется — причина была не в деплое, гипотеза об этом закрыта.

## Что посмотреть рядом (регресс)

- «Пример» и «Шаги» (соседние кнопки чата) — этот фикс их не касался (изменения только в ветке `isRequestHint`), но одним прогоном стоит подтвердить, что они по-прежнему работают.
- Задания типа «найди ошибку» — подсказки для них идут через тот же `isRequestHint`, отдельного пути нет; стоит один раз проверить, что маркерная подсказка подставляется и для такого типа задания.
- Секреты задания по-прежнему не должны утекать клиенту: просканировать кадры `trainer:interaction:replace/receive/stream` и HTTP-ответы на предмет полей `hints`/`answer`/`steps`/`rules`/`examples` — совпадений быть не должно (архитектурно не менялось, но при повторном прогоне дешево перепроверить).

## Открытые вопросы / код-наблюдение (не блокер)

Прежнее наблюдение из QA-цикла по MR !278 (`authoredHints ?? interaction.assignment?.hints`, мёртвый фолбэк при `hints: []`) больше не актуально: в MR !298 это выражение убрано целиком, `createHintRequestPrompt` использует только `authoredHints`, пришедшие с сервера, а мёртвый фолбэк на клиентский payload устранён как таковой (см. тест «ignores hints riding on the client assignment»). Открытых вопросов по коду на данный момент нет.

---
ticket: MIA-17
linear_url: https://linear.app/mia360/issue/MIA-17/podklyuchit-sentry-monitoring-oshibok-i-proizvoditelnosti
status: Testing
mr_url: https://gitlab.com/ai-math/ai-math-web/-/merge_requests/416
updated: 2026-09-18
---

## Контекст

Постановка просила подключить облачный Sentry. По факту в MR !416 (`feat/error-tracking-glitchtip` → `master`, смержен 2026-09-15T08:19:40Z) завели **self-hosted GlitchTip** — форк Sentry с тем же протоколом приёма (`@sentry/*` SDK не меняются, меняется только DSN). Причина зафиксирована в `docs/superpowers/specs/2026-09-15-error-tracking-glitchtip-design.md`: Sentry SaaS не подходит по трансграничной передаче ПДн несовершеннолетних, Sentry self-hosted не подходит по железу (нужно ~16 GB RAM, на хосте мониторинга 3.9 GB). GlitchTip живёт на `mon.mia360.ru` рядом с Grafana/Loki/Prometheus, UI и приём событий — `https://errors.mia360.ru`.

**Важно: HEAD master (коммит `0164cd7a`) содержит не только MR !416, а ещё минимум 6 последующих правок этой же фичи**, смерженных вплоть до 2026-09-17: `fix/glitchtip-release-and-health`, `fix/glitchtip-csrf`, `fix/glitchtip-csrf-dup`, `fix/glitchtip-webhook-no-private-ips`, **`feat/sentry-user-context` (MR !442, коммит `1569bc1b`, 2026-09-17T17:50:49+07:00)** и `docs/glitchtip-valkey-flushall`. MR !442 меняет поведение PII/IP уже **после** того, как QA-комментарий от 2026-09-15T18:44:50 был написан (см. «Открытые вопросы» — расхождение по IP).

Тестирование сейчас блокировано не багом, а организационным вопросом: нет подтверждённого безопасного доступа к панели `errors.mia360.ru`, отдельного от админки Mia (см. комментарий QA от 15.09). Ниже — что можно проверить по коду без этого доступа, и что нужно доотложить до его получения.

## Реализация

### Инициализация SDK — самая ранняя точка запуска, во всех 4 сервисах

- `api/src/sentry-boot.ts`, `socket-gateway/src/sentry-boot.ts`, `ai-proxy/src/sentry-boot.ts` — однострочные модули, импортируются первыми в `src/index.ts` каждого сервиса (комментарий: «ESM hoists imports, так единственный способ инициализировать SDK раньше остального приложения — отдельный модуль»).
- Общая функция `initSentry(serviceName)` — идентична в `api/src/helpers/sentry.helper.ts`, `socket-gateway/src/helpers/sentry.helper.ts`, `ai-proxy/src/helpers/sentry.helper.ts` (`diff` между тремя файлами — пусто). Ключевое:
  ```ts
  const dsn = process.env.SENTRY_DSN;
  if (!dsn) return;               // пустой DSN — SDK не инициализируется вообще
  ...
  integrations: Sentry.getDefaultIntegrationsWithoutPerformance(),  // без tracesSampleRate вообще
  initialScope: { tags: { service: serviceName } },                // api / socket-gateway / ai-proxy
  beforeSend: (event) => scrubEvent(event),
  ```
- Frontend: `frontend/plugins/sentry.client.ts` — `@sentry/vue`, **не `@sentry/nuxt`** (в CLAUDE.md причина: nuxt-модуль шлёт свою телеметрию на sentry.io). DSN берётся из `runtimeConfig.public.sentryDsn`, `if (!dsn) return`. Интеграции очищены от `BrowserTracing`/`Replay`/`Feedback`.

### PII-скраб — 4 идентичных файла под CI-проверкой

`api/src/helpers/sentry-scrub.ts` ↔ `socket-gateway/src/helpers/sentry-scrub.ts` ↔ `ai-proxy/src/helpers/sentry-scrub.ts` ↔ `frontend/utils/sentry-scrub.ts` — те же 168 строк, `diff` между всеми четырьмя — пусто. Это тот же паттерн ручного дублирования, что был в MIA-166 (константа флага в двух копиях), но здесь он **защищён CI-джобом** `sentry_scrub_mirror_check` (`gitlab/pipeline-prepare-test.yml:14-24`):
```
diff api/src/helpers/sentry-scrub.ts socket-gateway/src/helpers/sentry-scrub.ts
diff api/src/helpers/sentry-scrub.ts ai-proxy/src/helpers/sentry-scrub.ts
diff api/src/helpers/sentry-scrub.ts frontend/utils/sentry-scrub.ts
diff api/src/helpers/sentry.helper.ts socket-gateway/src/helpers/sentry.helper.ts
diff api/src/helpers/sentry.helper.ts ai-proxy/src/helpers/sentry.helper.ts
```
`scrubEvent()` в `beforeSend`: обрезает query/fragment у `request.url` и у breadcrumb-полей `to/from/url`, удаляет `request.data`, `request.cookies`, `request.query_string`, `request.env` (там прячется `REMOTE_ADDR`), режет заголовки `authorization/cookie/set-cookie/x-api-key/user-agent/referer(r)`, оставляет в `user` только allow-list `id/email/ip_address`, в `extra` — allow-list из 11 технических ключей, маскирует email/RU-телефон/`Bearer …`/токены (`glpat-`, `sk-`, `lin_api_`, `PMAK-`) рекурсивно везде, включая breadcrumb.data.arguments у console-интеграции, и режет длину значения до 250 симв. **после** маскирования (важно — до маскирования SDK обрезает на 4096, чтобы email не срезался посередине до того, как маска сработает).

### setUser — id + (email только у родителя), без имени

`frontend/utils/sentry-user.ts` (`sentryIdentity()`) + `frontend/plugins/sentry-user.client.ts` (watch на `userStore.user`, `Sentry.setUser` + `Sentry.setTag` для когорты `role/grade/scoringVersion/abVariant/hasAccess`). **Отклонение от текста задачи** («setUser({id, role}) — без PII: email, имя, телефон»): email родителя **сознательно оставлен** — `/me` отдаёт email только родителю, никогда ученику, и это явно протестировано в `frontend/utils/__tests__/sentry-user.test.ts`:
```ts
it('identifies a parent by id and email', () => {
  expect(sentryIdentity(parent).user).toEqual({ id: 'p-1', email: 'mama@example.ru' });
});
it('identifies a student by id alone - /me carries no email for them', () => {
  expect(sentryIdentity(student).user).toEqual({ id: 's-1' });
});
it('never carries the name: it is a minor and adds nothing to a stack trace', ...);
```
Появилось не в MR !416, а в MR !442 (`1569bc1b`, 2026-09-17) — до этого `beforeSend` жёстко резал `user` до `{id}`.

### Env-переменные — схема отличается от текста задачи

В задаче просили 4 отдельных DSN (`SENTRY_DSN_FRONTEND/API/SOCKET/AI_PROXY`). По факту сделано **2 DSN на окружение**, не 4: один общий `SENTRY_DSN` для всех трёх бэкендов (api/socket-gateway/ai-proxy шлют в один и тот же GlitchTip-проект, различаются только тегом `service`), и отдельный `NUXT_PUBLIC_SENTRY_DSN` для фронта. Подтверждено:
- `.env.example:185-195` — `SENTRY_DSN=`, `NUXT_PUBLIC_SENTRY_DSN=`, `SENTRY_ENVIRONMENT`, `SENTRY_RELEASE`, `SENTRY_URL/ORG/AUTH_TOKEN` (последние три — только для загрузки sourcemaps на фронте).
- `gitlab/pipeline-prepare-test.yml:206-213` — `TEST_SENTRY_DSN`/`TEST_SENTRY_DSN_PUBLIC` → `.env`; коммент прямо в коде: «Empty DSN disables the SDK, so no precheck here — telemetry must never be the reason a deploy fails».
- `gitlab/pipeline-prepare-prod.yml:107-112` — та же схема с `PROD_`-префиксом.
- Реально проектов в GlitchTip два — `mia360-prod` и `mia360-test` (выбор по `$CI_COMMIT_TAG` в `gitlab/pipeline-builds.yml:24`), а не 4 (по одному на сервис из постановки).

### Release / sourcemaps

`SENTRY_RELEASE=${CI_COMMIT_SHORT_SHA}` — совпадает с задачей (`gitlab/pipeline-prepare-test.yml:212`, `-prod.yml:112`). Sourcemaps грузятся только для фронта: `frontend/Dockerfile:45-57` — `sentry-cli sourcemaps inject` → `releases new` → `sourcemaps upload`, затем `find .output/public -name '*.map' -delete` (карты не должны попасть в образ). Бэкенды релиз только тегируют, sourcemaps не грузят — тоже совпадает с задачей.

### Performance monitoring — **не реализовано вообще**, сознательно

Задача просила sample rate 10-15%. В коде — явный отказ от трейсинга на всех 4 сервисах: `integrations: Sentry.getDefaultIntegrationsWithoutPerformance()` (бэкенд) и фильтрация `BrowserTracing`/`Replay`/`Feedback` (фронт), с одинаковым комментарием в трёх местах: «No tracesSampleRate at all: `hasSpansEnabled` — nullish-проверка, так что даже 0 включает весь OTel auto-instrumentation (prisma/redis/kafka), а GlitchTip трейсы не умеет показывать». Design-документ подтверждает — это осознанный отказ от APM, потому что GlitchTip его не поддерживает.

### PII в IP — поведение **менялось дважды**, актуальное состояние противоречит тексту задачи

1. 2026-09-15, коммит `e7454859` (до/во время MR !416): `monitoring/nginx/nginx.conf.template` для vhost GlitchTip перестал пробрасывать `X-Real-IP`/`X-Forwarded-For` (`proxy_set_header X-Real-IP "";`) — именно это состояние проверяла QA 15.09 («0 утечек» по IP в вебсокет-пути).
2. **2026-09-17, коммит `1569bc1b` (MR !442, уже после QA-комментария)**: решение отменено по требованию владельца продукта — «IP нужен, чтобы различать неавторизованных посетителей на лендинге/в регистрации, где `user.id` ещё нет». `nginx.conf.template` сейчас (строки ~110-117) снова шлёт `X-Real-IP $remote_addr; X-Forwarded-For $proxy_add_x_forwarded_for;`, а `scrub_ip_addresses` **выключен и на организации, и на обоих проектах GlitchTip напрямую в БД** (через API это поле не редактируется — «PUT отдаёт 200 и молча игнорирует»). Итог: реальный, неусечённый IP ученика/родителя теперь **осознанно** попадает в GlitchTip. Юнит-тест в коде явно называет это фичей, а не багом: `it('keeps the client ip', ...)` в `sentry-scrub.test.ts` — потому что `ALLOWED_USER_KEYS` включает `ip_address`.

Это прямо противоречит и тексту задачи («убедиться, что в события не попадают … IP»), и уже устаревшему выводу QA-комментария от 15.09 — см. «Открытые вопросы».

### Обработка ошибок — что реально долетает

- `api/src/helpers/api-error-handler.ts:9-16` — в GlitchTip уходят только ответы **5xx** (`statusCode >= 500`), 4xx считается штатной работой и не репортится. Смок-тест через реальный эндпоинт должен провоцировать настоящий 500, а не 401/400.
- `socket-gateway/src/index.ts:86-89` — `captureError(err, { socketId, kind: 'connection' })` на ошибке в `handleConnection`.
- `ai-proxy/src/handlers/conversation.handler.ts:87` — `captureError(error, { streamId, code })` в catch стрима; рядом инкрементируется Prometheus-счётчик `aiProxyErrorsTotal` (`ai-proxy/src/metrics.ts:19`) — то есть каждый вызов `captureError` в ai-proxy имеет параллельный сигнал в Grafana, не требующий доступа к GlitchTip.
- `frontend/error.vue` — глобальный обработчик страницы ошибки шлёт `Sentry.captureException`, кроме 404 (`is404` исключён) и chunk-load ошибок (у них своя логика реролла, `frontend/utils/sentry-chunk-load.ts`). `frontend/layouts/admin.vue:49` и `frontend/plugins/chunk-error-report.client.ts:20` — ещё два явных места ручного `captureException`.
- Готовый **смок-скрипт только у api**: `api/scripts/sentry-smoke.ts` — падает, если `SENTRY_DSN` пуст, иначе бросает `Error('sentry smoke test')`, печатает `eventId`, флашит и выходит. В `CLAUDE.md` есть готовая команда для прода: `docker exec ai-math-$(cat /root/ai-math/ACTIVE_COLOR)-api-1 bun scripts/sentry-smoke.ts`. У socket-gateway и ai-proxy аналогичного скрипта **нет** — для них смок придётся либо провоцировать реальную ошибку в коде (например, оборвать сокет посреди `handleConnection`), либо просить разработчика на время добавить копию скрипта.

## Тест-кейсы / Чек-лист

Фича инфраструктурная, без своего UI — ниже точечный чек-лист вместо полноценных UI-кейсов, разбитый на то, что проверяется по коду прямо сейчас, и то, что требует доступа к GlitchTip.

### Проверяется без доступа к GlitchTip

1. **Пустой DSN не ломает старт.** На стенде/локально без `SENTRY_DSN`/`NUXT_PUBLIC_SENTRY_DSN` в env все 4 сервиса должны стартовать как раньше — код гарантирует это ранним `return` в `initSentry` (`api/src/helpers/sentry.helper.ts:5-6`, идентично в socket-gateway/ai-proxy) и в `frontend/plugins/sentry.client.ts:9-10`. Дополнительно `api/scripts/sentry-smoke.ts:4-7` явно падает с понятным сообщением `SENTRY_DSN is empty - nothing to send`, если его по ошибке запустить с пустым DSN, — можно использовать как быстрый способ убедиться, что переменная вообще не задана на нужном хосте.
2. **PII-скраб — юнит-тесты, уже есть, можно просто прогнать.** По одному идентичному набору в каждом сервисе: `api/src/helpers/__tests__/sentry-scrub.test.ts`, `socket-gateway/src/helpers/__tests__/sentry-scrub.test.ts`, `ai-proxy/src/helpers/__tests__/sentry-scrub.test.ts`, `frontend/utils/__tests__/sentry-scrub.test.ts`. Ключевые кейсы, которые стоит явно свериться, что зелёные: `drops sensitive headers and keeps the rest`, `drops the request body entirely`, `strips the query string from the url` / `strips the fragment as well as the query`, `drops the request env, where the client ip hides`, параметризованный `masks personal data: ...` (email/RU-телефон в 4 форматах/Bearer/`glpat-`), `masks the raw arguments the console integration keeps beside the message`, `masks before cutting, so a long message cannot smuggle an email past the cut`. **Отдельно обратите внимание**: тест `keeps the client ip` и `keeps the email, which names the account behind an issue` в этом же файле — это не баг и не пропущенный скраб, это текущее осознанное поведение (см. «Реализация» → IP и setUser выше), не заводить как дефект.
3. **setUser — тоже юнит-тестами.** `frontend/utils/__tests__/sentry-user.test.ts`: parent получает `{id, email}`, student — только `{id}`, имя нигде не появляется, теги-когорта (`role/grade/scoringVersion/abVariant/hasAccess`) всегда присутствуют (обнуляются при разлогине, не залипают от предыдущего пользователя).
4. **Зеркальность скраб-файлов.** Дублирование прикрыто CI-джобом `sentry_scrub_mirror_check` (`gitlab/pipeline-prepare-test.yml:14-24`) — на HEAD все 5 diff'ов пустые, специально проверять руками не нужно, но стоит убедиться, что джоб реально в конвейере зелёный на MR/пайплайне, где менялся хоть один из этих файлов.
5. **Sourcemaps не попадают в прод-образ.** `frontend/Dockerfile:56` удаляет все `*.map` из `.output/public` после инжекта/аплоада — можно проверить прямо на собранном образе (`docker run ... find /app/.output/public -name '*.map'` → пусто) без обращения к GlitchTip.
6. **Публичная конфигурация фронта содержит DSN, когда он задан.** Уже подтверждено QA 15.09 (лендинг грузится без консольных ошибок, DSN виден в runtime-конфиге и в собранном entry-бандле) — регресс здесь не ожидается, специально перепроверять не нужно, если бандл не менялся.
7. **Косвенный сигнал для ai-proxy без GlitchTip**: счётчик Prometheus `ai_proxy_errors_total` (`ai-proxy/src/metrics.ts:19`) растёт при каждом вызове `captureError` в `conversation.handler.ts:87` — если есть доступ к Grafana/Prometheus (не к GlitchTip), можно спровоцировать ошибку стрима и убедиться, что счётчик увеличился один-в-один с попыткой отправки в трекер. Для api и socket-gateway такого прямого 1:1-счётчика на `captureError` нет.
8. **api репортит только 5xx.** Если для смока дёргать реальный API-эндпоинт, а не `sentry-smoke.ts`, нужно гарантированно вызвать 500 (`api/src/helpers/api-error-handler.ts:9-16`) — обычный 4xx (неверный пароль, невалидный body) в GlitchTip не попадёт по дизайну, это не повод считать трекинг сломанным.

### Требует доступа к GlitchTip (`errors.mia360.ru`) — сейчас не подтверждено

- Просмотр 4 событий (по одному на api/socket-gateway/ai-proxy/frontend) с проверкой тега `service`, `release` = `$CI_COMMIT_SHORT_SHA`, читаемого стека фронта (символизация через загруженные sourcemaps).
- Визуальное подтверждение, что в принятых событиях реально нет полей из блок-листа (тело запроса, `Authorization`/`Cookie`, query-строка) — юнит-тесты гарантируют логику скраба, но не гарантируют, что SDK/интеграции не кладут что-то новое мимо `beforeSend` (например, через `contexts`, которые скраб не трогает целиком).
- Способ безопасно получить доступ: краткий вызов `api/scripts/sentry-smoke.ts` внутри контейнера (см. команду в `CLAUDE.md`) не требует UI GlitchTip и подтверждает, что событие **отправлено и принято** (печатает `eventId`, код выхода 0) — но не заменяет визуальную проверку содержимого события и не годится для socket-gateway/ai-proxy (нет своего скрипта).
- Правила алертов (`quantity`/`timespan_minutes`) хранятся в БД GlitchTip, не в гите — свериться, что они реально созданы, можно только через UI/API GlitchTip.

### Отдельный технический вопрос: `GET /_nuxt/*.js.map` → 500 вместо 404

Причина найдена по коду, не по логам:
1. `pnpm build` печёт манифест публичных ассетов (размер/etag каждого файла) в скомпилированный `.output/server/chunks/nitro/nitro.mjs` **до** шага sentry-cli — на этот момент `.map`-файлы ещё физически лежат в `.output/public`, и манифест содержит на них записи.
2. `frontend/Dockerfile:45-56` дальше запускает `sentry-cli sourcemaps inject`/`upload`, а затем `find .output/public -name '*.map' -delete` — файлы карт удаляются с диска.
3. `frontend/scripts/sync-nitro-asset-manifest.mjs` (запускается сразу следующей строкой, `Dockerfile:57`) чинит манифест только для файлов, которые **всё ещё существуют** на диске (`readAsset` возвращает `null` для удалённых `.map`, и в этом случае запись в манифесте `entry` возвращается без изменений — см. `syncManifest()`, строка с `if (!contents || ...) return entry;`). Записи для уже удалённых `.map`-файлов в манифесте **не удаляются**.
4. На рантайме `frontend/server/middleware/not-found.ts` пропускает любой путь с префиксом `/_` без проверки (`staticPrefixes` в `frontend/nuxt.config.ts:11` включает `'/_'` безусловно) — запрос идёт дальше, к статик-раздаче nitro, которая находит в манифесте запись «файл существует», пытается прочитать несуществующий файл с диска и падает необработанной ошибкой → 500 вместо честного 404.

Это low severity (в проде на `.map` никто легитимно не ходит, а `sourcemap: {client: 'hidden'}` в `nuxt.config.ts:70` и так не оставляет на них ссылок в JS), но воспроизводимо для любого произвольно угаданного пути `/_nuxt/<hash>.js.map` и стоит отдельного маленького тикета разработчику — например, чистить из манифеста записи с уже отсутствующим на диске файлом в том же `sync-nitro-asset-manifest.mjs`, либо не запекать `.map` в манифест изначально.

## Аналитика/API

Отдельного бизнес-аналитического события эта задача не добавляет — `frontend/constants/analytics-events.ts` и `api/src/constants/analytics-events.ts` этим MR не тронуты (сверено `diff` — файлы по-прежнему идентичны друг другу, как требует их собственный CI-джоб `analytics_events_mirror_check`, но список событий не изменился). Из инфраструктурных «API»-поверхностей, которые стоит держать в уме при тестировании:
- `api/scripts/sentry-smoke.ts` — фактически CLI-эндпоинт для смока, см. выше.
- `ai_proxy_errors_total` (Prometheus, `ai-proxy/src/metrics.ts:19`) — косвенный индикатор для ai-proxy.
- GlitchTip alert webhook уходит не в Slack (как в тексте задачи — «канал #alerts»), а в уже существующий `alert-telegram-relay` (`monitoring/alert-telegram-relay/relay.py`) по неугадываемому пути на vhost мониторинга; формат пейлоада у GlitchTip Slack-совместимый, но получатель — Telegram, не Slack.

## Открытые вопросы

1. **Главный блокер тестирования — организационный, не дефект.** Нет подтверждённого безопасного доступа к панели `errors.mia360.ru`, отдельного от админки Mia (использовать те же Basic Auth без явного подтверждения нельзя — см. QA-комментарий от 15.09). Без него нельзя закрыть ключевой acceptance задачи: визуально подтвердить, что ошибка каждого из 4 сервисов долетает до GlitchTip с читаемым стеком и корректными тегами/релизом. Нужно запросить у DevOps/владельца GlitchTip либо временный read-only доступ, либо провести смок вместе с ним.
2. **PII по IP — поведение сейчас противоречит и тексту задачи, и уже написанному QA-выводу.** 17.09 (MR !442, после QA-комментария от 15.09) владелец продукта отменил блокировку IP на nginx и отключил `scrub_ip_addresses` в GlitchTip напрямую в БД — реальный IP ученика/родителя теперь осознанно попадает в трекер (см. «Реализация»). Старый вывод «0 утечек по IP» от 15.09 больше не описывает текущий код. Нужно явное решение: либо задача и её acceptance («без … IP») формально корректируются под это продуктовое решение, либо код нужно откатывать к состоянию 15.09 — сейчас это ничья зона между постановкой и последним коммитом.
3. **setUser включает email родителя**, а не только `{id, role}}`, как написано в скоупе задачи. Решение осознанное и покрыто тестами (`sentry-user.test.ts`), но стоит явного подтверждения автора задачи, что это приемлемо.
4. **Performance monitoring не реализован вообще** ни на одном сервисе (не 10-15% сэмплинга, а полное отсутствие `tracesSampleRate`/трейсинг-интеграций) — осознанный отказ, потому что GlitchTip не умеет хранить/показывать трейсы. Формально это прямое расхождение со «Скоуп» в тексте задачи; по духу «Out of scope» (profiling — отдельной задачей) это ближе к принятому компромиссу, но стоит зафиксировать явно, а не оставлять недосказанным.
5. **Алерт «регрессия» технически невозможен в GlitchTip** — модель алертов там (`quantity` + `timespan_minutes` + `uptime`) не содержит понятия «ошибка появилась снова после решения», об этом прямо пишет design-документ. Плюс правила заведены только для `mia360-prod` (не для test), и доставка идёт в Telegram через существующий relay, а не в Slack-канал `#alerts`, как в тексте задачи. Все три момента — вероятно принятые следствия перехода на GlitchTip, но нуждаются в явном sign-off от автора задачи, а не в тихом расхождении.
6. **Технический вопрос малой важности**: `GET /_nuxt/*.js.map` отвечает 500 вместо 404 для удалённых из образа source maps — причина установлена по коду (устаревшая запись в nitro-манифесте ассетов после того, как `.map` удаляются на сборке уже после того, как манифест испечён) — см. раздел «Тест-кейсы» выше. Не блокирует приёмку, но стоит завести отдельным маленьким тикетом.
7. Не удалось подтвердить по репозиторию, реально ли заполнены CI/CD-переменные `SENTRY_AUTH_TOKEN`/`SENTRY_URL`/`SENTRY_ORG` (используются в `gitlab/pipeline-builds.yml:24-26` для аплоада sourcemaps) — без них фронтенд просто соберётся без загрузки карт (`Dockerfile:52`: «source map upload failed - continuing without readable prod traces», сборка не падает). Нужно свериться в GitLab CI/CD Variables или в логе последнего прод-билда, что аплоад карт реально произошёл, а не молча пропущен.

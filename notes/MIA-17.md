---
ticket: MIA-17
linear_url: https://linear.app/mia360/issue/MIA-17/podklyuchit-sentry-monitoring-oshibok-i-proizvoditelnosti
status: Testing
mr_url: https://gitlab.com/ai-math/ai-math-web/-/merge_requests/416
updated: 2026-09-17
---

## Контекст

Задача — видеть production-ошибки в реальном времени со стек-трейсами по всем сервисам
(frontend, api, socket-gateway, ai-proxy), получать алерты и привязывать ошибки к релизу, не
раскрывая при этом персональные данные детей (текст ученика, email, телефон, IP).

**Ключевое отличие от исходного скоупа задачи: вместо облачного Sentry команда подняла
self-hosted GlitchTip** (`glitchtip/glitchtip:6.2.6`, форк Sentry 9 с тем же протоколом приёма
событий) на своём хосте мониторинга `mon.mia360.ru`, снаружи доступен как
`https://errors.mia360.ru`. Причины: Sentry SaaS по умолчанию шлёт `user.id`/IP за пределы РФ
(персональные данные несовершеннолетних), а self-hosted Sentry (~40 контейнеров, 16 GB RAM)
не влезает ни на mon (2 CPU / 3.9 GB), ни на прод. Обоснование — в
`docs/superpowers/specs/2026-09-15-error-tracking-glitchtip-design.md`.

**Второе отличие от скоупа: DSN не по-сервисно.** В тикете предполагались отдельные
`SENTRY_DSN_FRONTEND` / `_API` / `_SOCKET` / `_AI_PROXY`. По факту — два проекта в GlitchTip
(`mia360-prod`, `mia360-test`), не четыре: один общий DSN (`SENTRY_DSN`, GitLab CI variables
`PROD_SENTRY_DSN` / `TEST_SENTRY_DSN`) для трёх bun-бэкендов (api, socket-gateway, ai-proxy,
различаются тегом `service`), и отдельный `NUXT_PUBLIC_SENTRY_DSN` для фронта.

**Третье отличие: алерты идут не в Slack, а в существующий Telegram-канал** через
`alert-telegram-relay` (уже использовался для алертов Grafana). Причина та же, что и для
Sentry → GlitchTip: `api.telegram.org` недоступен из РФ напрямую, поэтому нужен relay; заводить
отдельно Slack не стали. GlitchTip отправляет webhook в Slack-совместимом формате
(`{attachments: [...]}`), `relay.py` научился отличать его от формата Grafina (`{alerts: [...]}`)
по наличию ключа `attachments` без `alerts`.

**Performance monitoring из тикета сознательно не сделан** (это отдельно подтверждено в
спеке) — GlitchTip трейсы не хранит и не показывает, `tracesSampleRate` намеренно не
задаётся ни в одном сервисе. Это осознанное сокращение скоупа, а не баг, но стоит знать при
тестировании: не искать performance-вкладку.

Slack-интеграция и «привязка к релизу» из скоупа выполнены, но по-другому: релиз — через
`sentry-cli releases`, алерты — через Telegram-relay, а не через нативную Slack-интеграцию
GlitchTip.

## Реализация

### Инициализация по сервисам

Три bun-бэкенда (`api`, `socket-gateway`, `ai-proxy`) устроены одинаково:
- `src/sentry-boot.ts` — отдельный модуль, импортируется первой строкой `src/index.ts` (до
  импорта остального приложения), чтобы `Sentry.init()` успел отработать раньше кода
  приложения.
- `src/helpers/sentry.helper.ts` — `initSentry(serviceName)` (no-op при пустом `SENTRY_DSN`),
  `captureError(err, context)`, `fatalHandler(log)` для `process.on('uncaughtException')` /
  `unhandledRejection`. `fatalHandler` обязательно делает `process.exit(1)` после
  `Sentry.flush(2000)` — без этого регистрация SDK-слушателя подавляет штатное поведение
  рантайма «напечатать и упасть», и сервис продолжил бы работать в неопределённом состоянии
  вместо перезапуска.
- Тег `service` (`api` / `socket-gateway` / `ai-proxy`) проставляется в `initialScope.tags` —
  этим differentiate сервисы внутри одного DSN/проекта.
- `tracesSampleRate` не задаётся вовсе (используется `getDefaultIntegrationsWithoutPerformance()`),
  так как даже `0` включил бы весь набор performance/OTel-интеграций (`hasSpansEnabled` — nullish
  check, а не truthy check).

Точки перехвата ошибок:
- `api`: `fastify.setErrorHandler` — в трекер уходят только ответы 5xx (400/401/403/404 не
  считаются ошибками — это ожидаемое поведение приложения).
- `socket-gateway`: обёртка вокруг подключения — `captureError(err, { socketId, kind: 'connection' })`
  (см. `src/index.ts:88`).
- `ai-proxy`: `captureError(error, { streamId, code })` в обработчике разговора
  (`src/handlers/conversation.handler.ts:87`).

Фронт (`frontend`, Nuxt 4.4): не официальный модуль `@sentry/nuxt` (он ронял `nuxt build` по
heap и по умолчанию слал телеметрию на sentry.io), а `@sentry/vue` через клиентский плагин
`frontend/plugins/sentry.client.ts`. DSN приходит из `runtimeConfig.public.sentryDsn`
(`NUXT_PUBLIC_SENTRY_DSN`). Захват необработанных ошибок — `frontend/error.vue` (кроме 404 —
это ожидаемое состояние, не дефект). **Важно: ошибки SSR в трекер не попадают** — сознательное
сокращение скоупа (SSR включён только на лендинге, серверные ошибки видны в логах Loki).

### PII-скрабинг

`sentry-scrub.ts` — файл-зеркало в 4 местах (`api/src/helpers`, `socket-gateway/src/helpers`,
`ai-proxy/src/helpers`, `frontend/utils`), побайтово идентичен (проверено чтением всех 4 копий —
совпадают). CI-джоб `sentry_scrub_mirror_check` (`gitlab/pipeline-prepare-test.yml`) сверяет их
через `diff` и падает при расхождении; аналогично для `sentry.helper.ts` в трёх бэкендах.

`scrubEvent()` в `beforeSend`:
- `user` → оставляет только `{ id }` (UUID), либо `{}`;
- `request`: убирает query-строку **и фрагмент** из `url` (`split(/[?#]/)`) — важно из-за
  одноразового pairing-токена в фрагменте `/m/<id>#<token>`; удаляет `data`, `cookies`,
  `query_string`, `env` (там `REMOTE_ADDR`); вырезает заголовки `authorization`, `cookie`,
  `set-cookie`, `x-api-key`, `user-agent`, `referer`/`referrer`;
- `extra` — **белый список ключей** (`ALLOWED_EXTRA_KEYS`): `kind`, `reqId`, `method`,
  `routeUrl`, `statusCode`, `code`, `socketId`, `streamId`, `model`, `promptLength`,
  `completionLength`, `messageLength`, `contentLength`, `durationMs`, `attempt` — всё остальное
  отбрасывается целиком, даже если вызывающий код случайно передаст `{ body }`/`{ email }`;
- маскирует по regex в exception.value, message и breadcrumb.message/data (рекурсивно, включая
  вложенные объекты и массивы — важно для breadcrumb console-интеграции, которая кладёт сырые
  аргументы `console.*` в `data.arguments`): email, российские номера телефона (все форматы,
  `+7`/`8`, с скобками/пробелами/дефисами), `Bearer <token>`, известные форматы секретов
  (`glpat-`, `sk-`, `lin_api_`, `PMAK-`);
- обрезает результат до 250 символов **после** маскирования (SDK обрезает *до* `beforeSend`,
  поэтому `maxValueLength` в `sentry.helper.ts` завышен до 4096/во фронте — намеренно, чтобы
  маска не резалась посередине e-mail);
- `contexts.response`/`contexts.state` удаляются (несут тела ответов/состояние).

В `ai-proxy` в событие по договорённости попадают только длина промпта/ответа, модель и код
ошибки — не сам текст (see `promptLength`/`completionLength`/`messageLength` в белом списке).

**IP дополнительно режется на nginx**, не только в SDK: GlitchTip сам дописывает
`user.ip_address` при приёме события (даже если `beforeSend` его удалил), а штатный
`scrub_ip_addresses` только обнуляет последний октет. Поэтому в
`monitoring/nginx/nginx.conf.template`, блок `server_name errors.mia360.ru`, заголовки
`X-Real-IP`/`X-Forwarded-For` намеренно выставлены в пустую строку (не проброшены) — GlitchTip
видит только адрес самого nginx-контейнера.

### Release tracking / sourcemaps

- `frontend/Dockerfile`, стадия build: после `pnpm build` — `sentry-cli sourcemaps inject`,
  `sentry-cli releases new "$SENTRY_RELEASE"` (GlitchTip не создаёт релиз автоматически, в
  отличие от Sentry — без явного `releases new` `sourcemaps upload` падает с "Release not
  found"), затем `sentry-cli sourcemaps upload`. `SENTRY_AUTH_TOKEN` приходит секрет-маунтом
  BuildKit, не через `ENV` (не оседает в docker history шаренного раннера).
- При пустом токене или пустом релизе шаг загрузки просто пропускается (лог "source map upload
  failed" / "skipping"), билд не падает.
- **Все `.map`-файлы удаляются из образа после загрузки** (`find .output/public -name '*.map'
  -delete`) — иначе 181 карта раздавалась бы по HTTP и отдавала исходники любому.
- `sourcemap: { client: 'hidden' }` в `nuxt.config.ts` — убирает комментарий
  `//# sourceMappingURL` из бандла (сами карты и так не публикуются).

### Пустой DSN

Во всех 4 сервисах пустой `SENTRY_DSN`/`NUXT_PUBLIC_SENTRY_DSN` — SDK не инициализируется
вовсе (`if (!dsn) return`). Precheck на непустоту DSN в CI сознательно не добавлен — телеметрия
не должна ронять деплой.

### Алерты (Telegram, не Slack)

Правила в БД GlitchTip (не в гите), только для проекта `mia360-prod`:
- новая ошибка: `quantity=1`, `timespan_minutes=1`;
- всплеск: `quantity=25`, `timespan_minutes=10`.
- **Правила "регрессия" нет** — модель алертов GlitchTip её не поддерживает (только имя,
  quantity, timespan_minutes, uptime-флаг). Это ограничение GlitchTip, а не недоделка.
- Тестовый проект (`mia360-test`) алертов не шлёт вообще — стенд ломают намеренно.
- Webhook уходит на публичный путь `https://mon.mia360.ru/$ALERT_HOOK_PATH/` (не внутрь docker-сети
  — иначе пришлось бы включать глобальный `GLITCHTIP_ALLOW_PRIVATE_IPS`, что открыло бы Prometheus/
  Loki/Grafana/Postgres blind-SSRF). Путь — секрет, без `auth_basic` (generic-webhook GlitchTip не
  умеет слать credentials).
- `monitoring/alert-telegram-relay/relay.py` различает пейлоады GlitchTip (`attachments` есть,
  `alerts` нет) и Grafana (`alerts` есть) и форматирует под Telegram HTML.

### Фикс-коммиты после исходного MR416 (актуальный master, до `08f12565`)

По `git log --oneline` для `monitoring/`, `*/sentry-boot.ts`, `*/helpers/sentry*`,
`frontend/plugins/sentry*`, `frontend/utils/sentry*` (от старых к новым):

| Коммит | Что исправили |
|---|---|
| `e8765264` | fix: устранить находки ревью в трекинге ошибок (общий проход) |
| `c843f89c` | скрабить сообщение breadcrumb без поля `data` (раньше breadcrumb без `data` мог проскочить немаскированным) |
| `65795f4f` | feat(monitoring): запуск self-hosted GlitchTip на хосте мониторинга |
| `e7454859` | **не пробрасывать IP клиента в GlitchTip** — добавлены пустые `X-Real-IP`/`X-Forwarded-For` в nginx-vhost `errors.mia360.ru` (см. раздел про IP выше) |
| `5143c9c9` | **не открывать приватную сеть ради webhook алертов** — webhook вынесен на публичный неугадываемый путь вместо `GLITCHTIP_ALLOW_PRIVATE_IPS` |
| `e1e9163a` | закрыть остатки ревью трекинга ошибок |
| `c1c27cc2` | сузить маскирование и закрыть breadcrumbs (уточнены regex-маски, чтобы не есть UUID/cuid) |
| `2eeba8fe` | маскировать аргументы консоли и все форматы телефона (рекурсивная маскировка `console.*` breadcrumb.data.arguments, расширены форматы телефона) |
| `b4289f40` | **убрать шум чужих ошибок и хешей чанков** — `denyUrls` (ошибки расширений браузера/Метрики) + `foldChunkLoadEvent` (схлопывает «Failed to fetch dynamically imported module …» с разным именем чанка в один fingerprint `chunk-load`, иначе каждый деплой = новая issue = новый алерт) |
| `618ffa59` | правки по ревью доступа к хранилищу + отдельный fingerprint `chunk-load-unrecovered` для той же ошибки, долетевшей до страницы ошибки (там reload уже не спас) |

Про "csrf"/"csrf dup"/"release and health", упомянутые в задаче — по коду и `CLAUDE.md`
(раздел `CSRF_TRUSTED_ORIGINS`, healthcheck GlitchTip требующий заголовка `Host`) это, судя по
всему, были инфраструктурные правки самого docker-compose/окружения GlitchTip (CSRF-проверка
логина за TLS-терминацией nginx, healthcheck на `127.0.0.1` без `Host`) — они описаны в спеке
как уже решённые, отдельных недавних коммитов с такими сообщениями в затронутых путях не
нашлось (возможно, ушли в `docker-compose.monitoring.yml`/`gitlab/monitoring-setup.yml`, не
покрытые фильтром git log выше — стоит перепроверить при доступе к полной истории этих
файлов).

## Тест-кейсы

### Приоритет: закрыть блокер прошлого QA-раунда (доступ к GlitchTip + контролируемые smoke-ошибки)

Прошлый раунд встал именно на этом — не было безопасного доступа к UI `errors.mia360.ru`. Это
организационный блокер, не баг в коде. Перед тестом нужно запросить у ответственного
(Evgeny Konoplev / автора MR) доступ к GlitchTip: либо отдельную учётку тестировщика в
организации `mia360` (создаётся через `manage.py createsuperuser` в контейнере GlitchTip + роль
`OrganizationUser`, см. спеку), либо согласованный доступ к уже аутентифицированной сессии.
Регистрация самостоятельно закрыта (`ENABLE_USER_REGISTRATION=false`).

**TC-1. Доступ к GlitchTip UI**
- Преднагрузка: получить URL `https://errors.mia360.ru` и отдельные учётные данные (НЕ Basic
  Auth от админки Мии — это другой сервис).
- Шаги: открыть `https://errors.mia360.ru`, залогиниться.
- Ожидаемый результат: виден список проектов организации `mia360` — `mia360-prod` и
  `mia360-test`, у каждого доступен список issues.

**TC-2. Smoke-ошибка api через готовый скрипт `api/scripts/sentry-smoke.ts`**
- Это готовый инструмент именно под эту задачу — им можно закрыть "не покрыто" из прошлого
  раунда для api без похода в код приложения.
- Преднагрузка: на тестовом стенде `SENTRY_DSN` не пустой (иначе скрипт сам завершится с
  "SENTRY_DSN is empty - nothing to send" и exit 1). Судя по `CLAUDE.md`, штатный способ —
  `docker exec ai-math-$(cat /root/ai-math/ACTIVE_COLOR)-api-1 bun scripts/sentry-smoke.ts`
  (нужен shell-доступ к тест-стенду, либо CI-джоб/раннер с тем же образом).
- Шаги: выполнить команду выше (или `bun scripts/sentry-smoke.ts` внутри контейнера api).
- Ожидаемый результат: в stdout печатается `eventId <hex>`; в GlitchTip, проект `mia360-test`,
  в течение ~минуты появляется issue "sentry smoke test" с тегом `service: api`, релизом
  (если `SENTRY_RELEASE` задан на стенде) и `environment`.

**TC-3. Smoke-ошибка socket-gateway**
- Готового скрипта нет (`sentry-smoke.ts` есть только в `api/scripts`). Проверить через
  реальную ошибку: например, разорвать соединение нештатным образом или спровоцировать
  исключение в обработчике `connection` (см. `captureError(err, { socketId, kind: 'connection' })`
  в `socket-gateway/src/index.ts`). Альтернатива — попросить разработчика временно добавить
  тестовый throw, либо согласовать with backend-разработчиком безопасный способ вызвать
  исключение на тест-стенде.
- Ожидаемый результат: в GlitchTip `mia360-test` появляется issue с тегом `service:
  socket-gateway`, в `extra` — только `socketId`/`kind` (или другие поля из белого списка), без
  email/имени/телефона/IP.

**TC-4. Smoke-ошибка ai-proxy**
- Аналогично TC-3: готового smoke-скрипта под ai-proxy в списке изменённых файлов нет.
  Спровоцировать реальную ошибку в диалоге тренажёра (например, отправить некорректный
  запрос, который уронит обработчик `conversation.handler.ts`), или попросить
  разработчика.
- Ожидаемый результат: issue с тегом `service: ai-proxy`, `extra` содержит `streamId`/`code`,
  но не текст промпта/ответа модели — только `promptLength`/`completionLength`/`model` (если
  переданы).

**TC-5. Smoke-ошибка frontend (браузер)**
- Шаги: на тестовом стенде спровоцировать необработанное исключение в браузере (например,
  через консоль разработчика `throw new Error('QA smoke test MIA-17')` на любой странице
  приложения, либо целенаправленно сломать что-то, что долетит до `error.vue` с ненулевым
  statusCode).
- Ожидаемый результат: событие доезжает до `mia360-test` (или `mia360-prod`, смотря какой DSN
  на стенде), **стек-трейс читаемый (не минифицированный)** — то есть source maps
  применились при отображении в GlitchTip; в событии виден тег `release` (git SHA/short-SHA).

### PII-скрабинг

**TC-6. Email в тексте ошибки не долетает**
- Спровоцировать ошибку, где exception message или console.log содержит email (например,
  `throw new Error('failed for user test@example.com')` в тестовом коде, если можно
  безопасно внести, или найти естественный кейс).
- Ожидаемый результат: в событии GlitchTip email заменён на `[email]`.

**TC-7. Телефон в тексте ошибки не долетает**
- Аналогично TC-6, с российским номером телефона в разных форматах: `+7 999 123-45-67`,
  `8(999)1234567`, `79991234567`.
- Ожидаемый результат: все три формата заменены на `[phone]`, при этом UUID/cuid-подобные
  строки (например, id заданий) в том же сообщении не искажены масками (это отдельно
  зафиксировано юнит-тестами — маска специально узкая).

**TC-8. `user` содержит только id**
- В событии от авторизованного пользователя (родитель/ребёнок) проверить объект `user`.
- Ожидаемый результат: `user = { id: '<uuid>' }`, без `email`, `username`, `ip_address`.

**TC-9. IP не долетает даже при живом трафике**
- Отправить событие с реального IP тестировщика (не из внутренней сети).
- Ожидаемый результат: в событии GlitchTip `user.ip_address` отсутствует или `user: null` —
  это дважды защищено: `beforeSend` вырезает `user.ip_address`, и nginx не пробрасывает
  `X-Real-IP`/`X-Forwarded-For` на `errors.mia360.ru`, так что даже штатное дописывание IP
  самим GlitchTip увидит только адрес nginx-контейнера, а не клиента.

**TC-10. Query-строка и фрагмент URL не долетают**
- Спровоцировать ошибку на странице с query-параметром или на маршруте `/m/<id>#<token>`
  (phone-pairing).
- Ожидаемый результат: `request.url` в событии обрезан до пути, без `?...` и без `#...`.

**TC-11. Заголовки Authorization/Cookie/User-Agent не долетают**
- Если есть возможность посмотреть `request.headers` в событии (для бэкенд-ошибок) — убедиться,
  что `authorization`, `cookie`, `set-cookie`, `x-api-key`, `user-agent`, `referer` отсутствуют.

**TC-12. `extra` — белый список**
- Если разработчик может временно передать в `captureError` произвольное поле (например,
  `{ body: '...' }` или `{ email: '...' }`), убедиться, что оно не попадает в событие — доедут
  только поля из ALLOWED_EXTRA_KEYS (`kind`, `reqId`, `method`, `routeUrl`, `statusCode`, `code`,
  `socketId`, `streamId`, `model`, `promptLength`, `completionLength`, `messageLength`,
  `contentLength`, `durationMs`, `attempt`).

**TC-13. Тексты промптов/ответов ai-proxy не долетают**
- В событии от ai-proxy убедиться, что нет текста заданий/сообщений ученика — только длины
  (`promptLength`, `completionLength`, `messageLength`) и модель.

*Примечание: PII-скраб уже закрыт unit-тестами на всех 4 копиях `sentry-scrub.ts`
(`api/src/helpers/__tests__/sentry-scrub.test.ts` и аналоги в `socket-gateway`, `ai-proxy`,
`frontend/utils`) — TC-6…TC-13 в первую очередь имеет смысл прогнать как sanity end-to-end
поверх уже покрытой юнитами логики, а не как единственный источник уверенности.*

### Release/sourcemaps + известная находка (.map → 500 вместо 404)

**TC-14. Release проставлен**
- В любом событии из TC-2…TC-5 проверить поле release — должно совпадать с
  `$CI_COMMIT_SHORT_SHA` актуального билда на стенде.

**TC-15. Frontend stack trace читаемый (source maps применились)**
- В событии из TC-5 открыть stack trace в GlitchTip.
- Ожидаемый результат: видны исходные имена файлов/строки (не `chunk-abc123.js:1:45234`), т.е.
  карта была загружена в GlitchTip при сборке и он смог её символизировать.

**TC-16 (известная находка прошлого раунда, подтвердить/опровергнуть). `.map` отдаётся с 500,
а не 404**
- Шаги: `GET https://<тест-стенд>/_nuxt/<любой существующий js-чанк>.js.map` (сам `.map`-файл
  специально удаляется из образа на этапе сборки — см. `frontend/Dockerfile`, `find
  .output/public -name '*.map' -delete` — то есть запрос заведомо на несуществующий ресурс).
- Наблюдалось в прошлом раунде: HTTP 500, тело ответа 162 байта, без `sources`/`sourcesContent`/
  `mappings` (т.е. не настоящая карта, а какая-то страница ошибки).
- Ожидаемый результат по здравому смыслу: 404 (ресурса нет). Технический контекст, который
  вероятно объясняет 500: nginx-конфиг теста (`nginx/nginx.test.conf.template`) проксирует
  `location ^~ /_nuxt/` напрямую на upstream `frontend` без собственной 404-обработки; на
  стороне Nuxt/Nitro `server/middleware/not-found.ts` считает любой путь с префиксом `/_`
  «существующим» (`staticPrefixes` включает `/_`) и пропускает его дальше без проверки, есть ли
  файл физически — то есть штатный 404-guard приложения на `/_nuxt/*.map` не применяется вовсе,
  и запрос падает на что-то в самом Nitro (вероятно, попытка отдать SPA-shell для
  `.js.map`-расширения или необработанное исключение раздачи статики). Это не блокер PII/приёма
  ошибок, но: а) выдаёт нестандартный код ответа там, где ожидается 404, б) стоит перепроверить,
  что тело 500 не содержит внутренних деталей/трейсов. Зафиксировать как отдельный баг/тикет,
  если воспроизведётся повторно.

### Slack-алерты (по факту — Telegram)

**TC-17. Новая ошибка → сообщение в Telegram**
- Преднагрузка: узнать/иметь доступ к тестовому Telegram-чату алертов (учесть: правило алертов
  настроено только на проект `mia360-prod` — на `mia360-test` алерты сознательно не отправляются
  "стенд ломают намеренно и постоянно"). Если нужно проверить алертинг end-to-end, придётся
  either временно завести правило на `mia360-test`, либо тестировать на проде осторожно
  согласованной ошибкой.
- Шаги: спровоцировать новую (ранее не встречавшуюся) ошибку в проекте с настроенным правилом.
- Ожидаемый результат: в Telegram-чате приходит сообщение в HTML-разметке с заголовком ошибки,
  ссылкой "открыть в Grafana"/на GlitchTip-issue и названием проекта (`mia360-prod`). В логе
  `alert-telegram-relay` — строка `relayed N glitchtip issue(s)`. Согласно `CLAUDE.md`,
  штатный способ проверки алертов — именно "сквозняком" от реальной ошибки до этой строки в
  логе relay, а не одиночным ручным POST в relay (он проверяет только половину цепочки).
- **Важно для теста: искать не Slack, а Telegram-канал** — Slack не подключался вовсе, задача
  переиспользовала существующий Telegram-relay.

**TC-18. Всплеск частоты**
- Спровоцировать ≥25 однотипных ошибок в `mia360-prod` за 10 минут.
- Ожидаемый результат: отдельное алерт-сообщение о всплеске.

**TC-19. Регрессии — намеренно нет (не баг)**
- Не нужно искать отдельный алерт "ошибка вернулась после исправления" — в GlitchTip модель
  алертов такого типа не поддерживает вовсе (только new-issue/spike), это ограничение платформы,
  зафиксированное в дизайн-спеке. Если в Linear-тикете это числится обязательным пунктом
  скоупа — фиксировать как согласованное отклонение, а не как баг.

**TC-20. Шум не долетает / схлопывается**
- Заблокировать Яндекс.Метрику расширением-блокировщиком в браузере и спровоцировать её
  типовую ошибку (`DataCloneError` на `postMessage`) — либо ошибку внутри расширения браузера.
- Ожидаемый результат: событие в GlitchTip не появляется (`denyUrls` в `sentry.client.ts`
  отсекает `mc.yandex.ru` и `*-extension://`).
- Отдельно: спровоцировать "Failed to fetch dynamically imported module" (например, открыть
  страницу, задеплоить новую версию поверх, затем перейти на другую страницу в старой вкладке)
  дважды с разными хешами чанков.
- Ожидаемый результат: обе ошибки схлопываются в одну issue с fingerprint `chunk-load` (level
  `warning`), а не заводят два алерта "новая ошибка". Если та же ошибка происходит уже на
  странице `error.vue` (виден statusCode) — отдельный fingerprint `chunk-load-unrecovered`.

### Пустой DSN / стенды без переменных

**TC-21. Пустой `SENTRY_DSN` на бэкенде — сервис стартует как раньше**
- На локальном/dev-окружении с пустым `SENTRY_DSN` поднять api/socket-gateway/ai-proxy.
- Ожидаемый результат: сервисы стартуют штатно, ошибки в лог пишутся как раньше, ничего не
  падает из-за телеметрии. Запуск `api/scripts/sentry-smoke.ts` в таком окружении завершается
  сообщением `SENTRY_DSN is empty - nothing to send` и кодом выхода 1 (ожидаемое поведение
  скрипта, не сбой сервиса).

**TC-22. Пустой `NUXT_PUBLIC_SENTRY_DSN` — фронт работает без ошибок в консоли**
- Собрать/запустить фронт с пустым DSN.
- Ожидаемый результат: `Sentry.init` не вызывается (`frontend/plugins/sentry.client.ts` делает
  ранний `return`), в консоли браузера нет ошибок, связанных с Sentry/GlitchTip.

**TC-23. Пустой `SENTRY_AUTH_TOKEN`/`SENTRY_RELEASE` при сборке фронта — билд не падает**
- Собрать образ фронта без секрета `sentry_token` и/или без `SENTRY_RELEASE`.
- Ожидаемый результат: в логе сборки — "sentry token or release empty - skipping source map
  upload" (или "source map upload failed - continuing without readable prod traces"), сборка
  завершается успешно, `.map`-файлы всё равно удалены из финального образа.

## Открытые вопросы

- **Доступ к GlitchTip UI для тестировщика** — организационный блокер прошлого QA-раунда.
  Нужны либо отдельная учётка тестировщика в организации `mia360` (создаётся через
  `manage.py createsuperuser` + `OrganizationUser` с ролью в контейнере GlitchTip), либо
  согласованный проброс уже аутентифицированной сессии. Использовать Basic Auth от админки Мии
  для `errors.mia360.ru` нельзя — это разные сервисы с разными учётками.
- **`.map` → 500 вместо ожидаемого поведения для отсутствующего ресурса.** Не блокер PII и не
  блокер основного acceptance (карты не должны быть публично доступны, и по факту не доступны —
  сути они не содержат, тело всего 162 байта), но неверный код ответа (500 вместо 404) достоин
  отдельного тикета/уточнения у разработчиков: вероятная причина — `server/middleware/not-found.ts`
  считает префикс `/_` «существующим» без проверки физического наличия файла, и запрос
  проваливается в необработанный путь раздачи статики Nitro. Нужно решить у разработчиков,
  является ли 500 регрессией, приемлемым поведением, или что-то стоит поправить (например,
  явно исключить `*.map` из `staticPrefixes`/добавить их в 404-обработку).
- Готовых smoke-инструментов для socket-gateway и ai-proxy (по аналогии с
  `api/scripts/sentry-smoke.ts`) в репозитории нет — под них тестировщику нужен либо ручной
  сценарий (например, дефектное WS-сообщение), либо помощь разработчика для безопасного
  triggering тестовой ошибки на стенде.
- Уточнить у автора, соответствуют ли Telegram-алерты (вместо изначально запрошенного Slack) и
  отсутствие правила "регрессия" согласованному изменению скоупа, либо это стоит явно отразить
  в описании тикета/задокументировать как принятое решение (спека это объясняет, но исходное
  описание MIA-17 всё ещё говорит "Slack-интеграция").

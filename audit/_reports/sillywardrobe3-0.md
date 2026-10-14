# Аудит: sillywardrobe3-0 (Nya-Imagen / «Inline Image Generation» + гардероб)

Репозиторий: https://github.com/niemandswasser/sillywardrobe3-0 (локально /home/user/jane/audit/sillywardrobe3-0), ветка master, HEAD e75814d. Тегов нет, одна ветка. Код только читался, ничего не запускалось.

## Вердикт: 🟡 Рискованный, но не вредоносный

Явной эксфильтрации, обфускации, eval, скрытых запросов, чтения секретов ST и отложенных триггеров не найдено. Риски связаны с тем, что расширение по замыслу отправляет промпты и картинки (аватары, гардероб, референсы) на сторонние сервисы, хранит их ключи в открытом виде и требует вручную подменить файл ядра ST (`server-patch/`). Сам патч сервера проверен построчным diff с upstream ST и безопасен.

## Что делает расширение (по коду)

Форк расширения sillyimages (0xl0cal / Astera). Ищет в сообщениях ИИ теги `<img data-iig-instruction=...>` / `[IMG:GEN:{...}]`, собирает промпт (стиль, лорбук-референсы, аватары, гардероб, NPC) и генерирует картинку выбранным пользователем провайдером: OpenAI-совместимый, Gemini, OpenRouter, Electron Hub, Naistera, VoidAI/RouteMyAI, NovelAI (через серверный прокси ST). Результат сохраняется через `/api/images/upload`. Плюс «гардероб» (описания нарядов, инъекция в промпт LLM через `setExtensionPrompt`), галерея, лайтбокс, библиотека персонажей, vision-описание картинок.

## Происхождение и история (раздел C)

- 169 коммитов. Авторы: `0xl0cal` (10), `Astera <astera.vt@proton.me>` (49), `Astera.vt/stellvt` (11), `local <local@dontcare.com>` (97) — это история upstream. Два последних коммита от владельца форка `nimandswasser (≧◡≦) ♡ <solinvictus1274+1@gmail.com>` — «Add files via upload» (b544ced от 2026-09-09, e75814d от 2026-10-11, сообщение «baba bebe»).
- b544ced — массовая перезаливка (+~15 000 строк): разнесение `providers.js` на `src/providers/*`, новые `wardrobe.js`, `gallery.js`, `extras.js`, `vision.js`, `ui/*`; **удалены** провайдеры xAI и A1111; **добавлены** `aigate.js`, `problembo.js` (мёртвый код, см. ниже). e75814d добавил `server-patch/`, NovelAI-параметры, `wardrobeBulk*`, тесты.
- `manifest.json` (e75814d): `display_name` «Nya-Imagen», `author` пустой, `version` 1.743 (у upstream было 2.0), `homePage` по-прежнему указывает на `https://github.com/0xl0cal/sillyimages/`, `auto_update: true`. То есть ST будет обновлять расширение из репозитория форка без дополнительной проверки.
- `git log --all -S` по `eval(`, `new Function`, `sendBeacon`, `XMLHttpRequest`, `WebSocket`, `api/secrets`, `document.cookie`, `webhook`, `discord`, `pastebin`, `String.fromCharCode` — ни одного коммита (код никогда не добавлялся и не удалялся). `child_process` появляется только в e75814d, в `tests/*-browser.mjs`.
- Просмотр всех добавленных/удалённых URL в `-G'https?://'` по истории: только домены провайдеров, `wewwaistyping.github.io`, `127.0.0.1:7860` (удалённый A1111), `api.aigate.shop`, `problembo.com`. Посторонних доменов нет.
- README (RU/EN) устарел: перечисляет xAI Imagine, не упоминает NovelAI, `server-patch/` и новые провайдеры. Установка серверной части описана только в `NOVELAI-REQUESTS.md` (одной фразой).

## server-patch/ — что патчит и как ставится

Файлы: `server-patch/novelai.js` (428 строк), `server-patch/novelai-image-params.js` (122 строки).

- **Что это:** замена штатного серверного плагина ST `src/endpoints/novelai.js` плюс новый модуль `src/endpoints/novelai-image-params.js` (импорт `./novelai-image-params.js` в строке 9 `novelai.js`). `NOVELAI-REQUESTS.md` (строки 41–47): «Его копия `src/endpoints/novelai-image-params.js` используется сервером SillyTavern... потребуется перенести и серверную правку из `server-patch/novelai.js` с её модулем. После установки серверных файлов нужно перезапустить SillyTavern». Установщика, скриптов, `postinstall` нет: пользователь вручную перезаписывает файл ядра ST. Расширение само ничего на сервер не копирует (проверено grep: `server-patch` нигде в `src/`/`index.js` не упоминается, кроме тестов).
- **Проверка на подмену:** скачан upstream `SillyTavern/SillyTavern@release:src/endpoints/novelai.js` (481 строка), оба файла приведены к LF, выполнен `diff`. Различий ровно три:
  1. добавлена строка `import { buildNovelaiImageRequest } from './novelai-image-params.js';` (стр. 9);
  2. в `/status` базовый URL `IMAGE_NOVELAI + '/user/subscription'` заменён на `API_NOVELAI + '/user/subscription'` (стр. 144; оба — хосты novelai.net);
  3. в `/generate-image` вместо жёстко заданного тела запроса (`params_version: 3`, `v4_prompt`, ...) вызывается `buildNovelaiImageRequest(request.body, { calculateSkipCfgAboveSigma })` (стр. 315, 323).
  Остальное (`/generate`, `/generate-voice`, `readSecret(..., SECRET_KEYS.NOVEL)`, прокси на `api.novelai.net`, `text.novelai.net`, `image.novelai.net`) побайтно совпадает с upstream. Ключ NovelAI читается штатно и уходит только в заголовок `Authorization` запросов к novelai.net, как и в оригинале.
- **`novelai-image-params.js`:** чистая функция сборки запроса; собирает фиксированный набор полей, «произвольные поля клиента не пробрасываются» (стр. 39), числа клампятся, `reference_images` — до 5 строк. Сети, `fs`, `process`, секретов нет.
- **Остаточный риск (низкий):** установка требует перезаписи файла ядра ST и перезапуска; при обновлении ST патч молча откатится/устареет и может разойтись с новой версией upstream (функциональный, не вредоносный риск). Поскольку расширение имеет `auto_update: true`, файлы `server-patch/` могут измениться в будущем, но они не копируются автоматически — каждый раз их нужно проверять заново при ручном обновлении.

## Сетевые адреса

Все `fetch` перечислены grep'ом по всему репозиторию (других `XMLHttpRequest`/`WebSocket`/`sendBeacon`/`EventSource`/`$.ajax`/`import()` с внешним URL нет).

| Домен / URL | Что отправляется | Файл:строка | Обосновано? |
|---|---|---|---|
| Endpoint, заданный пользователем (дефолты: `api.openai.com` — только placeholder; пусто по умолчанию) `/v1/images/generations`, `/v1/images/edits` | `Authorization: Bearer <apiKey расширения>`, промпт, size, quality; для edits — референсы (аватары, наряды, контекст-картинки) как multipart | `src/providers/openai.js:179,231` | Да, заявленная функция; адрес выбирает пользователь |
| Endpoint пользователя `/v1/models` | ключ в Bearer | `src/providers/base.js:248` | Да |
| Gemini: `{endpoint}/v1beta/models/<m>:generateContent`, дефолт placeholder `generativelanguage.googleapis.com` | Bearer-ключ, промпт, референсы (inlineData base64) | `src/providers/gemini.js:293` | Да |
| Gemini список моделей: `{endpoint}/v1beta/models?key=<apiKey>` | **ключ в query-строке URL** пользовательского endpoint (в т. ч. сторонних прокси) | `src/providers/gemini.js:370-371` | Стандарт Google API, но ключ попадает в URL/логи прокси; низкий риск |
| `openrouter.ai/api/v1` (дефолт) `/chat/completions`, `/models` | Bearer, промпт, референсы data URL; заголовки `HTTP-Referer: window.location.origin` (адрес вашего ST) и `X-Title` | `src/providers/openrouter.js:203-213,261` | Да; origin ST раскрывается OpenRouter (стандартный заголовок OpenRouter) |
| `api.electronhub.ai` (дефолт) `/v1/images/generations`, `/v1/models` | Bearer, промпт, style/negative/steps | `src/providers/electronhub.js:215,255` | Да |
| `naistera.org` (дефолт) `/api/generate`, `/api/models` | Bearer-токен (из Telegram-бота), промпт, `reference_objects` (аватары/наряды/контекст) | `src/providers/naistera.js:85,213` | Да для выбранного провайдера; сторонний малоизвестный сервис получает изображения персонажей/персоны |
| `api.voidai.app` (дефолт) `/v1/chat/completions` | Bearer, промпт, референсы | `src/providers/voidai.js:293` | Да |
| Локально: `/api/novelai/generate-image`, `/api/novelai/status` | промпт, параметры, референсы; заголовки ST (`getRequestHeaders`, CSRF) | `src/providers/novelai.js:200`, `src/ui/apiSection.js:1276` | Да, свой сервер ST; ключ NovelAI берёт сервер из secrets |
| Vision endpoint (`visionEndpoint`, иначе основной `endpoint`) `/v1/chat/completions`, `/v1/models` | Bearer (`visionApiKey` или **основной apiKey**), изображение аватара/наряда/референса base64 + фиксированный промпт «опиши» | `src/vision.js:120,150` | Да, по нажатию пользователя; основной ключ используется как fallback (отражено в UI: «empty = use main key») |
| `https://wewwaistyping.github.io/slayimagespromts/` | Только GET (опционально `If-None-Match`/`If-Modified-Since`), данные пользователя не отправляются | `src/ui/stylePicker.js:19,88` | Да, каталог стилей (пикер по кнопке); но см. находку №1 (внешний HTML парсится и вставляется в DOM) |
| Произвольный URL, введённый пользователем (импорт лорбука / картинок-референсов) | GET без данных | `src/references.js:1440`, `src/utils.js:14` | Да, действие пользователя |
| URL картинок из истории чата (`img.src`), контекст-референсы | GET (скачивание); браузер сам обращается к хосту картинки | `src/utils.js:14,375` | Побочный эффект: если в сообщении есть внешняя картинка, при сборке референсов/скачивании делается запрос на этот хост (раскрытие IP). Низкий риск |
| Мёртвый код: `problembo.com` `/apis/v1/client/...` через ST `/proxy/<url>` (заголовки ST) и PUT на signed URL | Bearer, промпт, изображения | `src/providers/problembo.js:72-75,173` | Провайдер не зарегистрирован (`src/providers.js:88-94`), миграция `migrateRemovedProviders` (`src/settings.js:287-300+`) его удаляет — код недостижим |
| Мёртвый код: `api.aigate.shop` | Bearer, промпт | `src/providers/aigate.js:53,372` | Не зарегистрирован, недостижим (в этом файле также баг: `options` не определён в `_aigateGptGenerations`, стр. 300) |
| Локально: `/api/avatars/get`, `/api/images/upload|list|delete`, `/api/files/upload|sanitize-filename`, `/proxy/` | CSRF-заголовки ST, файлы расширения | `src/references.js:507`, `src/utils.js:219,265`, `src/gallery.js:100,116,814,831`, `src/wardrobe.js:785,1990,2007` | Да, стандартные вызовы API ST на свой сервер |

Серверный патч обращается только к `api.novelai.net`, `text.novelai.net`, `image.novelai.net` (`server-patch/novelai.js:11-13`) — как upstream.

## Находки

### 1. 🟡 Низкая–средняя. HTML внешнего сайта парсится и вставляется через innerHTML без экранирования кавычек
`src/ui/stylePicker.js:184-186` (данные из `parseSiteStyles`, стр. 47-79)
```js
filtersEl.innerHTML = ['', ...tags].map((tag) => `
    <button class="iig-style-chip ..." data-tag="${sanitizeForHtml(tag)}">${...}</button>
`).join('');
```
`sanitizeForHtml` (`src/utils.js:313-317`) работает через `textContent → innerHTML`, который экранирует `< > &`, но **не `"`**. `tag` берётся из атрибута `data-tags` страницы `wewwaistyping.github.io` (сторонний сайт). Если сайт будет скомпрометирован/изменён, значение вида `x" onmouseover="..."` внедрит обработчик в страницу ST (контекст ST-origin: доступ к чатам, `getRequestHeaders()`, API секретов ST). Сейчас это нереализованная возможность: проверенного вредоносного содержимого в расширении нет, а сам сайт не проверялся (не входит в репозиторий). Каталог загружается только при открытии пикера стилей. Рекомендация: экранировать `"`, либо строить DOM через `createElement`/`dataset`. Остальные интерполяции из того же источника безопасны (`img src` проходит через `new URL().href`, промпт/имя — через `encodeURIComponent`/текстовое экранирование).

### 2. 🟡 Низкая. Ключи API хранятся в открытом виде и подставляются в HTML без экранирования
`src/ui/apiSection.js:265` (`value="${settings.apiKey}"`), `src/wardrobe.js:1820` (`esc(iig.apiKey)`), хранилище — `extensionSettings.inline_image_gen` (settings.json пользователя ST), а не защищённый `secrets.json`. Ключи OpenAI/Gemini/OpenRouter/и т. д. — в `settings.js:96,223` (`apiKey`, `visionApiKey`) и в `connectionProfiles`. Это не утечка (наружу идёт только к выбранному endpoint), но: (а) любой другой установленный расширением JS и любой, кто прочтёт settings.json, видит ключ; (б) значение с `"` в `apiKey` ломает разметку (self-XSS). Раздел обращений к секретам ST (`/api/secrets/*`, `secret_state`, `oai_settings`, пароль прокси) — **отсутствует**.

### 3. 🟡 Низкая. Персональные изображения уходят на сторонние сервисы по умолчанию-подключаемым провайдерам
Аватары персонажа/персоны, наряды, NPC, предыдущие картинки чата и лорбук-референсы отправляются как референсы выбранному провайдеру (`collectReferences` в каждом провайдере; флаги `sendCharAvatar`/`sendUserAvatar`/`naisteraSendCharAvatar` по умолчанию `false` — `src/settings.js:103-104,115`). Промпт также включает текст описаний внешности. Это заявленная функция, но для малоизвестных сервисов (Naistera, VoidAI/RouteMyAI, Electron Hub) пользователь должен понимать, что данные уходят туда. В README это не предупреждается.

### 4. ℹ️ Информационно. Основной API-ключ используется как fallback для vision
`src/vision.js:101-107`: при пустом `visionEndpoint`/`visionApiKey` берутся основной endpoint и ключ. Если основной endpoint — провайдер генерации картинок, он получит ключ и картинку; это отражено в UI-плейсхолдере, но неочевидно.

### 5. ℹ️ Информационно. Установка серверного патча руками и устаревший README
См. раздел server-patch. README не описывает ни NovelAI, ни патч (стр. 5, 209 перечисляют xAI, которого в коде уже нет), manifest-метаданные (`author: ""`, `homePage` на чужой репозиторий, `auto_update: true`) не соответствуют форку. Мёртвый код `problembo.js`/`aigate.js` (900+ строк) оставлен в `src/providers/` без использования; тесты `tests/removed-providers.mjs` подтверждают, что провайдеры сознательно выведены и ключи не переносятся на другой сервис.

### 6. ℹ️ Информационно. Тесты запускают браузер
`tests/parameter-help-browser.mjs:10-24`, `tests/wardrobe-bulk-browser.mjs:9-20` используют `child_process.execFile` для запуска Chrome с `--allow-file-access-from-files` и локальными тестовыми страницами. Это только тестовые файлы (не подключаются ST, нет `package.json` и npm-скриптов), аргументы фиксированы, сети нет. При запуске тестов вручную флаг `--allow-file-access-from-files` ослабляет изоляцию headless-браузера, но страницы — локальные из репозитория.

## Что проверено и не найдено

- Нет `eval`, `new Function`, строкового `setTimeout/setInterval`, `document.write`, динамических `<script>`, `importScripts`, `Worker`, `WebAssembly`, `postMessage`, `window.open`. Единственный `import()` — `/scripts/personas.js` и `../../../personas.js` (собственный модуль ST, `src/references.js:37-40,489`).
- `atob` — только декодирование заголовка PNG для диагностики (`gemini.js:64`) и `base64ToBlob` (`utils.js:401`). `fromCharCode`, `btoa` в коде нет. Строк base64/hex длиннее 80 символов в JS/JSON/HTML/CSS/SVG/MD нет. Минифицированных файлов, бинарников, `.wasm` нет.
- Нет обращений к `/api/secrets/*`, `secret_state`, `oai_settings`, `reverse_proxy`/`proxy_password`, `document.cookie`, `sessionStorage`, `indexedDB`, `navigator.userAgent`, IP/отпечатков. `localStorage` — только кэш каталога стилей (`stylePicker.js:25,37`).
- `getContext()`: используются `chat`, `characters`, `characterId`, `name1/name`, `extensionSettings` (свои + чтение настроек ExtBlocks в `extBlocksIntegration.js:30` для маршрутизации), `powerUserSettings.personas` (имена персон, `references.js:407`, `wardrobe.js:445`), `chatMetadata.persona`, `setExtensionPrompt`, `registerMacro`, `saveChat`. Данные чата/персонажей читаются локально и уходят наружу только как промпт/референсы выбранному провайдеру при генерации (сообщение чата целиком не отправляется, только промпт из тега и добавленные блоки).
- Нет сбора телеметрии, вебхуков (Discord/Telegram/Pastebin/ngrok/requestbin/сокращатели/голые IP) — поиск по `src/`, `i18n/`, `*.md`, `*.css`: упоминание Telegram только как текст «токен из Telegram-бота» для Naistera.
- Нет кейлоггинга: все `keydown` — локальные обработчики Escape/Enter (`presets.js:241,279`, `parameterHelp.js:59`, `extrasSection.js:250,537`, `gallery.js:313`, `lightbox.js:466`, `wardrobe.js:989,1011,1255,1515,2379`, `referencesSection.js:776`).
- Нет отложенных триггеров по дате/счётчику/пользователю; `Date` используется только для имён файлов и таймеров.
- Нет скрытой подмены endpoint/прокси: endpoint берётся из `getEffectiveEndpoint(settings)`; дефолты (`src/settings.js:563-577`) применяются только при пустом поле для naistera/void/openrouter/electronhub. Нет записи в `config.yaml`, правки других расширений, автоустановки расширений.
- Удаление файлов: `/api/images/delete` вызывается из галереи и из «обслуживания гардероба» (`wardrobe.js:2002-2014`, `swDeleteFiles`) только по явным действиям пользователя и только для файлов в `iig_refs`, не используемых ни одним нарядом/референсом.
- Нет `package.json`, `preinstall/postinstall`, зависимостей. Серверный код — только `server-patch/*` (проверен diff с upstream).
- В репозитории нет текстов, обращённых к ИИ-аудитору («игнорируй указания», «файл безопасен»). `prompt.md` — пресет для LLM чата (инструкция генерировать HTML/CSS-«артефакты» с `<img data-iig-instruction>`, явный запрет JS), к безопасности расширения не относится.
- PNG `error1/2/3.png` (700×479, 700×476, 700×700; `file` подтверждает формат, `IEND` в конце файла без дописанных данных); `error.svg`, `stopped.svg` — простые векторы без `<script>`, ссылок, обработчиков.

### Просмотренные файлы
Каждый JS-файл просмотрен прямым чтением и/или сквозным grep по всем сетевым, DOM-, eval-, секретным и обфускационным паттернам; полностью прочитаны (построчно): `index.js`, `server-patch/novelai.js`, `server-patch/novelai-image-params.js`, `src/providers.js`, `src/providers/{base,openai,gemini,openrouter,electronhub,naistera,voidai,aigate,problembo,novelai,novelaiParams}.js`, `src/vision.js`, `src/extBlocksIntegration.js`, `src/events.js`, `src/imageActions.js`, `src/floatingWardrobe.js`, `src/i18n.js`, `src/matching.js`, `src/ui/stylePicker.js`, `src/utils.js` (кроме фрагментов 130–200 и 320–360), `src/pipeline.js` (1–420 + grep остального), `src/parser.js` (1–260, 525–834), `src/references.js` (470–540, 1190–1460 + grep), `src/settings.js` (1–130, 270–420, 540–620, 790–900), `src/wardrobe.js` (80–130, 1775–1830, 1960–2030 + grep), `src/wardrobeBulk.js` (1–60), `tests/novelai-requests.mjs` (150–227), `tests/parameter-help-browser.mjs`, `tests/wardrobe-bulk-browser.mjs`.
Только через сквозные паттерн-поиски (сеть, секреты, eval, base64/hex, `innerHTML`, хуки, клавиатура, домены, `getContext()`-поля, импорты вне расширения): `src/characterLibraryUi.js`, `src/extras.js`, `src/gallery.js`, `src/lightbox.js`, `src/wardrobeBulkUi.js`, `src/ui.js`, `src/ui/{apiSection,common,debugSection,extrasSection,parameterHelp,presets,referencesSection}.js`, оставшиеся части `pipeline.js`, `parser.js`, `references.js`, `settings.js`, `wardrobe.js` (2764 строки), `tests/{reference-matching.test.cjs,removed-providers.mjs,wardrobe-bulk.mjs,novelai-requests.mjs (1–150),parameter-help.html,wardrobe-bulk.html}`, `i18n/ru-ru.json`, `i18n/uk-ua.json`, `manifest.json`, `.gitignore`.
Не-код: `style.css`, `characterLibrary.css` (единственный `@import` — локальный `./characterLibrary.css`; внешних `url()`, шрифтов, трекинга нет), `README.md`, `README-en.md`, `NOVELAI-REQUESTS.md`, `prompt.md`, `error.svg`, `stopped.svg`, `error1/2/3.png`.

## Уверенность и непроверенное

- Уверенность в отсутствии вредоносного поведения и эксфильтрации: высокая (≈90%). Сетевая поверхность полностью перечислена, `server-patch/novelai.js` сверен с upstream ST `release` построчным diff.
- Не прочитаны построчно (проверены только grep'ом и точечными фрагментами): большая часть `src/wardrobe.js`, `src/gallery.js`, `src/lightbox.js`, `src/ui/*.js`, `src/characterLibraryUi.js`, `src/extras.js`, `src/settings.js` и CSS. Скрытая логика, не использующая никаких из проверенных API (сеть, `eval`, динамический импорт, DOM-вставка внешних строк), в них маловероятна, но не исключена на 100%.
- Содержимое сторонних сервисов (`wewwaistyping.github.io`, `naistera.org`, `api.voidai.app`, Electron Hub) не проверялось: расширение доверяет HTML-каталогу стилей (находка №1) и отправляет им пользовательские данные.
- Сравнение с upstream ST выполнено с веткой `release` на дату аудита; при другой версии ST возможны расхождения функциональности (не безопасности).

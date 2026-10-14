# Аудит: sillyimages (0xl0cal/sillyimages, «Inline Image Generation» 2.0)

Репозиторий: https://github.com/0xl0cal/sillyimages, локальная копия `/home/user/jane/audit/sillyimages`, ветка `master` @ `5494dd5` (merge PR #14 от stellvt, 2026-10-09). Тег `v2.0.0-beta.1`, ветка `origin/v2.0-beta` (предок `master`).
Режим аудита: только чтение (cat/grep/sed/Read, git log/show/diff). Ничего не запускалось, тесты (`tests/*.cjs`) не исполнялись.

## Вердикт: 🟢 Чисто (есть замечания низкой серьёзности)

Вредоносного кода, эксфильтрации, обфускации, динамической загрузки кода и доступа к секретам ST не обнаружено. Все внешние запросы — к провайдерам генерации картинок, выбранным и настроенным пользователем, и отправляют ровно то, что нужно для генерации. Ключ API хранится в открытом виде в настройках расширения (не в секретах ST), и это единственный заметный риск дизайна.
Если нужна более строгая планка, то пункты 1-3 из «Находок» (ключ в открытом виде, один общий `apiKey` для всех провайдеров, HEAD-запросы по путям из текста чата) тянут на 🟡, но не на вредоносность.

## Что делает расширение (по коду)

Находит в сообщениях ИИ теги `<img data-iig-instruction='{...}' src="[IMG:GEN]">` и `[IMG:GEN:{...}]`, собирает финальный промпт (стиль, референсы из лорбуков/аватаров), отправляет его выбранному провайдеру (OpenAI-совместимый, xAI, Gemini, OpenRouter, Electron Hub, Naistera, NovelAI, A1111/Forge), сохраняет результат в `/user/images/...` через API сервера ST и подставляет путь в сообщение. Плюс лайтбокс, кнопки скачать/перегенерировать, библиотека референсов персонажей, профили подключений, макрос `{{iig-book}}`.
Заявленное в README/manifest совпадает с реальным поведением.

## Куда уходят ключ и промпт — по провайдерам

Ключ — единственное поле `settings.apiKey` (`src/settings.js:291`, входит в `CONNECTION_FIELDS`, стр. 396, то есть дублируется в `connectionProfiles`). Промпт — итоговая строка из `buildFinalGenerationPrompt` (`src/parser.js:279-301`): промпт из тега + стиль (глобальный/из тега) + описания совпавших референсов из лорбука + (для ряда провайдеров) описания аватаров/персонажей из «библиотеки персонажей» + «reference instruction». История чата целиком никуда не передаётся.
Эндпоинт — поле пользователя либо дефолт (`src/settings.js:597-604`, `normalizeConfiguredEndpoint` стр. 714-732). Ключ уходит на тот URL, который указан в настройках (включая любой сторонний прокси).

| Провайдер | Куда (домен по умолчанию → URL) | Ключ | Что в теле запроса | Файл:строка |
|---|---|---|---|---|
| OpenAI-compatible | `endpoint` пользователя (placeholder `https://api.openai.com`, дефолта нет) → `/v1/images/generations` (JSON) или `/v1/images/edits` (multipart, если есть референсы) | `Authorization: Bearer <apiKey>` | model, prompt (финальный), n, size, quality, `moderation:'low'` (gpt-image), response_format; для /edits ещё картинки-референсы (аватары char/user, картинки лорбука, предыдущие картинки чата — только если включено) | `providers.js:828-835, 882-889`; список моделей `GET {endpoint}/v1/models` `providers.js:433-438` |
| xAI | `https://api.x.ai` (или endpoint) → `/v1/images/generations` / `/v1/images/edits` | Bearer | model, prompt, n, response_format, aspect_ratio, resolution, quality, image/images (data URL, до 3) | `providers.js:997-1004`; модели — базовый `fetchModels` 433 |
| Gemini-compatible | endpoint пользователя (placeholder `https://generativelanguage.googleapis.com`) → `/v1beta/models/{model}:generateContent` | `x-goog-api-key`, если хост заканчивается на `googleapis.com`, иначе `Authorization: Bearer` (`providers.js:1159-1161`); при загрузке списка моделей ключ идёт в query-строке `?key=` (`:1244-1245`) на любой указанный endpoint, затем fallback `GET /v1/models` с Bearer (`:1262-1264`) | contents.parts: подписи `IMAGE_n`, inlineData референсов (base64), текст промпта; generationConfig (aspectRatio, imageSize) | `providers.js:1166-1173` |
| OpenRouter | `https://openrouter.ai/api/v1` (или endpoint) → `/chat/completions` | Bearer; при хосте `openrouter.ai` добавляются `HTTP-Referer: window.location.origin` и `X-Title` (`providers.js:1448-1451`) | model, messages[{content: промпт + референсы-data URL}], modalities, image_config | `providers.js:1456-1460`; модели `GET /models?...` `:1519-1524` |
| Electron Hub | `https://api.electronhub.ai` (или endpoint) → как OpenAI (наследует `OpenAIProvider`) | Bearer | как OpenAI | `providers.js:1551`, модели `:1581-1586` |
| Naistera | `https://naistera.org` (или endpoint) → `POST /api/generate`; `GET /api/models`; при polling `GET /api/generate/jobs/{id}` | Bearer (токен из Telegram-бота сервиса, по тексту подсказки `ui.js:256`); `/api/models` сначала с ключом, при сетевой блокировке повтор без ключа (`providers.js:1686-1701`) | prompt, aspect_ratio, model, steps/sampler/scale/cfg_rescale, negative_prompt, preset, `reference_objects` [{image dataURL, description}] (аватары, лорбук, контекст), `video_test_mode`, `sync:false` | `providers.js:1690, 1750-1757, 1917-1924` |
| NovelAI (прямой) | `https://image.novelai.net` (или endpoint) → `POST /ai/generate-image` | Bearer (persistent API token NovelAI) | action, model, input (базовый промпт), parameters (размеры, seed, steps, v4_prompt, negative); референсы не отправляются | `providers.js:2299-2303` |
| A1111/Forge | `http://127.0.0.1:7860` (или endpoint) → `/sdapi/v1/txt2img` и служебные `/sdapi/v1/{sd-models,samplers,schedulers,sd-vae,upscalers}` | необязательный: `Authorization: Basic btoa(apiKey)` (`providers.js:2115, 2162, 2177, 2194, 2211, 2227, 2245`) | prompt (с `a1111PromptPrefix`), negative, параметры сэмплера и т.п. | `providers.js:2123-2127` |

Дополнительно к провайдерам, обращения к серверу самого ST (локальный origin, с CSRF-заголовками `getRequestHeaders()` только на эти адреса): `/api/images/upload` (`utils.js:219`), `/api/files/upload` (`utils.js:265`), `/api/files/sanitize-filename` и `/api/images/list` (`utils.js:301, 311`), `/api/avatars/get` (`references.js:400`), `/api/images/delete` (`lightbox.js:346`). К `/api/secrets/*`, `/api/settings/*`, `/api/chats/*`, `/api/characters/*` расширение не обращается. Заголовки ST во внешние запросы не подмешиваются.

## Сетевые адреса (сводка)

| Домен/URL | Данные | Файл:строка | Обосновано |
|---|---|---|---|
| endpoint пользователя / дефолты провайдеров из таблицы выше (`api.x.ai`, `naistera.org`, `image.novelai.net`, `openrouter.ai`, `api.electronhub.ai`, `127.0.0.1:7860`, placeholder `api.openai.com`, `generativelanguage.googleapis.com`) | ключ, промпт, референсы | см. таблицу | да, заявленная функция, провайдер выбирает пользователь |
| произвольный URL, введённый пользователем (импорт лорбука) | GET без данных | `references.js:1345` | да, действие по кнопке |
| произвольный URL, введённый пользователем (референс «по URL») | GET (`fetchImageBlob`) → затем загрузка в ST | `utils.js:11-15`, `references.js:1137` | да, по кнопке |
| URL картинки из ответа провайдера (`OpenRouter` вернул не data URL) | GET | `providers.js:1493`, `utils.js:11-15` | да, но URL контролирует провайдер |
| пути картинок из текста сообщений (в т.ч. «//host/…») | HEAD | `utils.js:325-331` через `parser.js:701-719`, `pipeline.js:691`; `characterLibraryUi.js:489` | см. находку №3 |
| `/scripts/personas.js`, `../../../personas.js` (свой ST) | `import()` модуля ST для `user_avatar` | `references.js:38-41, 384` | да, фиксированные локальные пути |
| `https://github.com/0xl0cal/sillyimages/` | — (ссылка в manifest) | `manifest.json:9` | да |

Нет: `sendBeacon`, `WebSocket`, `EventSource`, `XMLHttpRequest`, вебхуки, аналитика, телеметрия, отпечатки, запросы с `userAgent`/IP.

## Находки

1. Низкая. API-ключ хранится открытым текстом в `extension_settings.inline_image_gen` (а значит в `settings.json` пользователя) и дублируется в каждом профиле подключения. `src/settings.js:291, 396`. Не используется хранилище секретов ST. Ключ не читается другими расширениями специально, но любое другое расширение/плагин с доступом к настройкам увидит его. Кроме того, `ui.js:239, 250` вставляют `settings.endpoint` и `settings.apiKey` в `value="..."` без экранирования (self-XSS только от собственных данных пользователя).
2. Низкая. Единый `apiKey` на все типы API. При смене «API type» в настройках endpoint заменяется на дефолт нового провайдера, а ключ остаётся прежним (`ui.js:1385-1401`, `settings.js:734-756`). Если пользователь переключится, например, с OpenAI на Naistera и не заменит ключ, ключ OpenAI будет отправлен на `naistera.org`. Это ошибка UX, не умысел. Для Gemini при обновлении списка моделей ключ идёт в URL (`providers.js:1244`), что может попасть в логи стороннего прокси.
3. Низкая. Запросы `HEAD` по путям из текста чата (контролируется текстом сообщения). `parser.js:701` считает «путём» любую строку, начинающуюся с `/`, длиной более 5, включая протокольно-относительные `//evil.example/x.png`. `pipeline.js:691` вызывает `parseMessageImageTags(message, { checkExistence: true })`, затем `utils.js:325-331` делает `fetch(path, {method:'HEAD'})`. Сообщение ИИ или карточка с таким `src` заставит браузер сделать запрос на внешний хост (раскрытие IP, обход настройки ST «блокировать внешние медиа»). Аналогично `characterLibraryUi.js:489` при ручном обновлении генераций. Эксплуатация требует вредоносного контента в чате, а не злонамеренности автора.
4. Информационная. Данные пользователя, уходящие провайдеру при включённых опциях (по умолчанию выключены): аватар {{char}} и {{user}} (`sendCharAvatar`, `sendUserAvatar`, `naisteraSend*`: `settings.js:298-299, 317-318`), картинки лорбука, предыдущие картинки чата (`imageContextEnabled`, `settings.js:274`), описания персонажей. Аватар персонажа берётся из `context.getCharacterAvatar()`, а при отсутствии — из `/characters/<avatar>` (`references.js:126-129, 618-631`). Не проверено в рамках аудита, возвращает ли `getCharacterAvatar` миниатюру или исходный PNG-файл карточки. Если исходный файл, то в провайдера уйдёт PNG с встроенными метаданными карточки. Требует сверки с исходниками SillyTavern.
5. Информационная. Заголовки `HTTP-Referer: window.location.origin` и `X-Title` отправляются на openrouter.ai (`providers.js:1448-1451`), раскрывая origin инсталляции ST (адрес/домен) OpenRouter. Ограничено хостом openrouter.ai, комментарий в коде это объясняет.
6. Информационная. `manifest.json:8` — `"auto_update": true`: код расширения обновляется из репозитория при каждом старте ST. Для репозитория принимаются PR от внешних авторов (последние коммиты от Astera/stellvt через PR #13, #14). Они просмотрены (текущая версия кода прочитана полностью), но будущие обновления нужно проверять заново.
7. Информационная. Лорбук/референс можно импортировать по URL и из JSON (`references.js:1248-1355`). JSON разбирается через `JSON.parse` с валидацией, значения полей идут в UI через `sanitizeForHtml` (`utils.js:344-351`), автовыполнения нет. Регулярные выражения из лорбука (`parser.js:181`, режим «power») могут вызвать ReDoS — только локальная деградация.
8. Информационная. `prompt.md` — шаблон инструкции для LLM пользователя (HTML/CSS-артефакты + формат тегов `data-iig-instruction`). Обращён к модели пользователя как часть заявленной функции, не к ИИ-аудитору, и не содержит указаний вести себя особым образом по отношению к проверке.
9. Информационная. Экспорт логов (`settings.js:256-266`) создаёт локальный файл `iig-logs-*.txt` (содержит промпты, URL, ответы API). Ключи в логи не пишутся (проверено по `iigLog`-вызовам), но в сообщениях об ошибках провайдеров может быть тело ответа (до 800 символов). Пользователь сам решает, кому отдавать лог (в `.gitignore` стоит `logs_from_users/`).

## Что проверено и не найдено

- `eval`, `new Function`, `setTimeout/setInterval` со строкой, `document.write`, динамическая вставка `<script>`/`<iframe>`, `import()` с URL (единственный `import()` — фиксированные локальные модули ST, `references.js:384`): нет.
- Обфускация: `atob` встречается только в `utils.js:369` (`base64ToBlob` для multipart-референсов), `btoa` — только для Basic-авторизации A1111. `String.fromCharCode`, длинные base64/hex-строки, `.wasm`, минифицированные файлы, бинарники: нет. Единственные ассеты — `error.svg`, `stopped.svg` (текстовые SVG без скриптов, проверено).
- Секреты ST: `/api/secrets/*`, `secret_state`, `oai_settings`, `reverse_proxy`, `proxy_password`, `document.cookie`, `localStorage`, `sessionStorage`: нет.
- Массовое чтение чатов/персонажей/настроек с отправкой наружу: нет. `context.chat` читается для (а) сообщения, которое обрабатывается, (б) до N=3 предыдущих сгенерированных картинок (`references.js:690-723`), (в) локального списка путей картинок (`characterLibraryUi.js:482`). `context.characters` используется для построения локального UI библиотеки.
- Телеметрия/отпечатки/вебхуки/сокращатели/«голые» IP: нет (единственный IP — `127.0.0.1:7860` как дефолт A1111).
- Правка настроек ST или чужих расширений, изменение endpoint/proxy ST, запись в `config.yaml`, установка расширений: нет. Расширение пишет только в `extensionSettings.inline_image_gen`, в файлы картинок/видео через штатные API ST и в сообщения чата.
- Кейлоггинг: единственный `document.addEventListener('keydown')` — в лайтбоксе, активен только при открытом оверлее, обрабатывает Esc/стрелки/+/-/0 (`lightbox.js:598-625`). Остальные `keydown` — на поле импорта URL. Перехвата полей паролей нет (поле ключа — обычный `input type=password`, меняется только на `type=text` по кнопке).
- Отложенные триггеры (дата, счётчик, пользователь, IP): нет. `shouldTriggerNaisteraVideoForMessage` (`settings.js:701`) — пользовательская настройка «видео каждые N сообщений», не скрытый триггер.
- `package.json` отсутствует, серверной части (`plugins/`), child_process, fs вне папки: нет.
- Текст, обращённый к ИИ-проверяющему («игнорируй предыдущие указания», «файл безопасен» и т.п.): не найден (grep по всему репозиторию).
- История git: 177 коммитов. Авторы: `local <local@dontcare.com>` (97), `Astera <astera.vt@proton.me>` (58), `Astera.vt <…stellvt@users.noreply.github.com>` (12), `0xl0cal` (10). Тег `v2.0.0-beta.1` и `origin/v2.0-beta` — предки `master` (`git merge-base --is-ancestor`). `git log --all -S` по `eval(`, `new Function`, `XMLHttpRequest`, `sendBeacon`, `WebSocket`, `document.cookie`, `/api/secrets`, `secret_state`, `oai_settings`, `webhook`, `discord`, `pastebin`, `ngrok`, `bit.ly`, `child_process`, `importScripts`, `createElement(script)`, `localStorage`, `navigator.userAgent`, `String.fromCharCode` — не дал ни одного совпадения (даже в удалённом коде). `atob(` появился один раз в коммите `0342646` (OpenAI edits/base64ToBlob), это тот же легитимный код. Уникальные URL, когда-либо добавлявшиеся в код (через `git log -p`): только провайдеры из таблицы, `api.example.com`, `github.com/0xl0cal/sillyimages`, `w3.org/2000/svg`. Удалённого вредоносного кода и подмен ссылок нет. Перед разбиением монолита (`3c0f8c2`, 2026-04-24, 4086 строк `index.js`) обращения были теми же (`fetch(url…)`, `/api/avatars/get`, `/api/images/upload`, `/api/files/upload`, `naistera.org`).
- Расхождение manifest/README и кода: нет. Версия в manifest «2.0», author «local & astera».

## Покрытие (просмотренные файлы)

Прочитаны целиком построчно: `index.js`, `manifest.json`, `src/events.js`, `src/i18n.js`, `src/imageActions.js`, `src/lightbox.js`, `src/novelai.js`, `src/parser.js`, `src/pipeline.js`, `src/providers.js`, `src/references.js`, `src/settings.js`, `src/utils.js`, `src/characterLibraryUi.js`, `src/ui.js` (все 2883 строки, в том числе шаблоны и обработчики), `error.svg`, `stopped.svg`, `prompt.md`.
`style.css` (58 КБ) — проверен grep на `url(`, `@import`, `expression(`, `javascript:` (пусто), построчно не читался.
`i18n/ru-ru.json`, `i18n/uk-ua.json` — разобраны как JSON (369 ключей в каждом), проверены grep на `<script`, `<iframe`, `javascript:`, `http(s)://`, `onerror`, `eval` (пусто), построчно не читались.
`README.md`, `README-en.md` — просмотрены выборочно (формат тегов, настройки, ключевые слова про ключи/приватность/сеть), без ссылок на скрытое поведение. `.gitignore` — прочитан.
Тесты `tests/naistera-parameters-e2e.cjs`, `tests/novelai-e2e.cjs`, `tests/reference-matching.test.cjs` — только прочитаны grep-ом (импорты: `node:assert`, `node:http`, `node:fs`, `node:path`, `node:vm`, `playwright`; сеть — мок-сервер на localhost и перехват `https://` запросов; `process.env` только для пути браузера/артефактов; нет `child_process`, `exec`, внешних URL, кроме проверок ожидаемых адресов провайдеров), построчно не читались, не запускались.
Бинарных файлов нет (`git ls-files` — только .js/.cjs/.json/.svg/.css/.md/.gitignore).

## Уверенность и что не проверено

Уверенность высокая (≈93%). Весь исполняемый код (`index.js` + `src/*.js`) прочитан; бандлов и минифицированных файлов нет. Не проверено: поведение в живом ST и на реальных провайдерах (ничего не запускалось), исходники SillyTavern (вопрос п.4 про `getCharacterAvatar`), политика хранения данных сторонних сервисов (в первую очередь `naistera.org`, оператор неизвестен и принимает промпты, референсы и токен). Поскольку включён `auto_update`, вердикт относится только к ревизии `5494dd5`.

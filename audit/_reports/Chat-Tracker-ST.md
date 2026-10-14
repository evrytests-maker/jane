# Аудит: Chat-Tracker-ST (SunKatz05/Chat-Tracker-ST)

**Вердикт:** 🟢 Чисто (с оговоркой: 🟡 перехватывает глобальный `window.fetch`, но ничего никуда не отправляет)

Версия манифеста 1.1.0, 66 коммитов (11.01.2026 — 31.07.2026), один автор (SunriZe_ / sunkatz05@gmail.com), ветка только `main`, тегов нет.

## Что делает расширение (по коду)
Плавающая сворачиваемая панель с числом сообщений, числом скрытых сообщений и счётчиком токенов контекста относительно лимита, который задаёт пользователь. Токены берутся из поля `usage` в ответах генерации (для этого оборачивается `window.fetch`) либо из токенайзера ST. Дополнительно есть мини-окно «Sunny Tools» — мост к другому расширению SunnyMemories: читает его настройки и данные из `extension_settings.SunnyMemories` и `chat[0].extra.sunny_memories`, нажимает его кнопки и копирует текст в его поля. Настройки панели хранятся в `localStorage`.

## Сетевые адреса
| Домен/URL | Что отправляется | Файл:строка | Обоснованно? |
|---|---|---|---|
| Внешние домены | — | — | Не найдено вообще: нет вызовов `fetch` с собственными URL, XHR, sendBeacon, WebSocket, EventSource, Image, `$.ajax`, `import()`. В `index.js` и `style.css` нет ни одной строки `http(s)://` |
| Любой URL, который вызывает сам ST | пропускается без изменений: `originalFetch(...args)` | index.js:658 | Да (оборачивание, без изменения запроса и ответа) |

Единственный URL в репозитории — `homePage` в `manifest.json` (`https://github.com/SillyTavern/Extension-ChatTracker`) и ссылки в README (GitHub автора, t.me/pinacoladaST). Это метаданные, в коде они не используются. `homePage` указывает на несуществующий для автора «официальный» репозиторий организации SillyTavern (ссылка вводит в заблуждение, но не вредоносна; обновлений через него расширение не тянет).

## Находки
1. **Глобальный перехват `window.fetch`** (🟡 риск, не вредоносно) — `index.js:637-671`:
   ```js
   const originalFetch = typeof window.fetch === 'function' ? window.fetch.bind(window) : null;
   if (originalFetch && !window[CHAT_TRACKER_FETCH_FLAG]) {
       window.fetch = async (...args) => {
           ...
           const isGenerateRequest = isGenerationRequestUrl(urlString);
           const requestBody = isGenerateRequest ? await readRequestBody(input, options) : null;
           ...
           response = await originalFetch(...args);
           ...
           handleFetchResponse(response, urlString)   // response.clone().text()
   ```
   Обёртка видит все запросы страницы. Для URL, содержащих `/generate`, `/chat/completions`, `/completions`, `/responses`, `/v1/messages`, `generatecontent`, `/api/backends/` (`index.js:486-492`), читается тело запроса (там промпт и, в ST, поля вроде `reverse_proxy` / `proxy_password`) и клонируется ответ. Что делается с прочитанным:
   - из тела запроса собираются текстовые фрагменты (`collectTextFragments`, `estimatePromptUsage`, :301-335), через `getTokenCount` считается число токенов, сохраняется только число;
   - из ответа `parseUsageCandidate` / `extractUsage` (:360-460) берёт числа `prompt_tokens`, `completion_tokens`, `total_tokens` и т.п.;
   - в `localStorage` под ключом `chatTracker_apiUsage_<chatId>` кладётся объект `{prompt, completion, total, source, updatedAt}` (:341-346).

   Тела запросов и ответов нигде не сохраняются, не логируются (`CHAT_TRACKER_DEBUG = false`, :296) и не отправляются. Ни одна исходящая операция, кроме пропуска оригинального запроса, не выполняется. Заголовки (`getRequestHeaders`, CSRF) не читаются. Потенциал злоупотребления у такой обёртки высокий (перехватывается всё, включая ключи в теле запросов), поэтому отмечено как риск, но в текущем коде сбора ключей нет. Версия обёртки и флаг `__chatTrackerFetchWrappedV5` — исторически та же идея с первого коммита (`926381b`), логика расширялась (V5), но никаких внешних адресов в истории нет.
2. **Распознавание запросов ExtBlocks по стеку вызовов** (инфо) — `index.js:429-460` проверяет `new Error().stack` на подстроки `/extblocks/`, `ext-blocks-custom`, и поля тела запроса `chat_tracker_ignore`, чтобы не считать токены чужого расширения. Это эвристика исключения, не утечка.
3. **Поведение при ошибках** (инфо) — множество пустых `catch (e) {}` (например :667-670 `jQuery(async …)` ловит любые ошибки инициализации молча). Это затрудняет диагностику, но вредоносным признаком не является.
4. **Мост к SunnyMemories** (🟡 инфо) — `index.js:3248-3480, 3660-3985`. Читает `window.extension_settings.SunnyMemories`, `chat[0].extra.sunny_memories`, `character.data.extensions.sunny_memories_*`; программно кликает кнопки SunnyMemories (`nativeButton.click()`, :3827) и записывает текст из своего textarea в его поля с событиями `input/change/blur` (:3459-3467). Это правка чужого расширения, но она явная, пользовательская (по кнопкам в мини-окне) и описана в README. Данные остаются в DOM/памяти, наружу не уходят. Вывод библиотеки сделан безопасно через `textContent` (:3844-3873).
5. **`innerHTML`** — используется только для статических шаблонов и локальных строк `trackerText(...)` и чисел (:2466, 2741, 2764, 2954, 3082, 3894). Данные из чата в `innerHTML` не вставляются. Хотя ключи `title`/`aria-label` берутся из словаря расширения.
6. **Сборка отпечатка**: `localStorage` используется только для ключей `chatTrackerLanguage`, `chatTrackerCustomization`, `chatTrackerTokenMode`, `chatTrackerMaxTokens`, `chatTracker_settings`, `chatTracker_apiUsage_*`. Чтений чужих ключей, `cookie`, `navigator.userAgent`, IP — нет.
7. **Метаданные.** `manifest.json`: `homePage` указывает на `SillyTavern/Extension-ChatTracker`, а не на настоящий репозиторий (SunKatz05/Chat-Tracker-ST). Также `LICENSE` — 1065 байт, ASCII (стандартная короткая лицензия, содержимое просмотрено). Расхождений между README и кодом нет, кроме того, что README про «интеграцию с SunnyMemories» (код её действительно содержит).

## Что проверено и не найдено
- Все сетевые API (fetch с собственными URL, XHR, sendBeacon, WebSocket, EventSource, Image, `$.ajax/$.post`, динамический `import`, `<script>/<iframe>`, `createElement('script')`) — нет.
- Секреты и настройки: `/api/secrets/*`, `secret_state`, `oai_settings`, `api_key`, `password`, `proxy` как выражение чтения (`proxy_password` / `reverse_proxy` в коде не упоминаются), `getRequestHeaders`, `document.cookie` — нет.
- `eval`, `new Function`, строковые `setTimeout/setInterval`, `atob`, `btoa`, `fromCharCode`, длинные base64/hex-строки, `.wasm`, минифицированные файлы: нет. Файл `index.js` (170 КБ, 4080 строк) не минифицирован; строки длиннее 300 символов — это шаблоны разметки с i18n.
- Кейлоггинг: `keydown` вешается только на `document` для закрытия своих окон по Escape (:2582) и на свой input (:3129).
- Правка настроек ST, подмена endpoint, запись в `config.yaml`, установка других расширений — нет. `package.json`, `plugins/` и серверной части нет.
- Отложенные триггеры (дата, счётчик запусков, пользователь) — нет. Единственные таймеры: `setInterval(refreshAll, 2500)` (:671) и рефреш мини-окна (:3500).
- Текст, обращённый к ИИ: не найден.
- История git: 66 коммитов, один автор. Ни в одном коммите нет удалённого вредоносного кода: `git log --all -S` по `fetch(`, `http`, `eval(`, `atob`, `secret`, `XMLHttpRequest`, `sendBeacon`, `WebSocket`, `api_key`, `getRequestHeaders`, `Function(`, `document.cookie` — по коду пусто (`http` встречается только в README). Первый коммит с обёрткой `window.fetch` — `926381b` (11.01.2026); большой рефакторинг (3311 добавлений) — `b007d83` (31.07.2026), он также просмотрен через итоговый файл. Файлов, которые были удалены в истории, нет. Ветка `main` одна, тегов нет.

## Просмотренные файлы
- `index.js` — полностью (JS-часть 1–1010 и 2300–4080 построчно; 1010–2300 — CSS в шаблонных строках `ensureCustomizationStyles` / `ensureTrackerV7Styles`, просмотрены на предмет нестилевых вставок: `url(`, `@import`, `http`, `javascript:` отсутствуют).
- `style.css` (551 строка; `@keyframes` и обычные правила, `url(`, `@import` нет).
- `manifest.json`, `README.md`, `LICENSE`, `.gitignore`.
- Бинарников нет (по `file`: все файлы текстовые).

## Уверенность
Высокая в отсутствии эксфильтрации и вредоносной нагрузки: код без обфускации, целиком прочитан, нет ни одной точки выхода наружу. Остаточный риск — архитектурный: глобальный перехват `fetch` даёт расширению доступ к телам запросов с ключами, поэтому при будущих обновлениях стоит перепроверять diff именно этого места (`index.js:637-671`) и не включать автообновление вслепую. Не проверено: поведение стороннего SunnyMemories (его в репозитории нет).

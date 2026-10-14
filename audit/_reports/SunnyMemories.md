# Аудит: SunnyMemories

**Вердикт:** 🟢 Чисто (расширение не вредоносное, внешних сетевых запросов нет; есть мелкие замечания по качеству/рискам)

Репозиторий: https://github.com/SunKatz05/SunnyMemories, ветка main, HEAD `bec279a`. Тегов и других веток нет. 60 коммитов, единственный автор SunriZe_ <sunkatz05@gmail.com>. Версия в manifest.json 1.3.0 (в README бейдж 1.2.0 — рассинхрон, не угроза).

## Что делает расширение
Расширение памяти для ролевых чатов: через штатный `generateRaw` из ST (с опциональным временным переключением профиля подключения Connection Manager) генерирует саммари, факты, квесты, календарные события, «библиотеку», дневник альбома; инъектирует это в контекст через `setExtensionPrompt`; хранит данные в `chat_metadata`/`extension_settings`; имеет «альбом» изображений из чата (сохранение через штатные API ST). Заявленное в README/manifest соответствует коду.

## Сетевые адреса
Внешних доменов и абсолютных URL в коде нет (grep по `https?://` во всех .js/.html/.json/.css даёт только `http://www.w3.org/2000/svg` в data-URI SVG-иконке стилей и проверки протокола `"http:"`).

| URL / вызов | что отправляется | файл:строка | обоснованно |
|---|---|---|---|
| `$.get(extensionFolderPath/settings.html)` | ничего (загрузка своего шаблона) | index.js:6734 | да |
| `fetch(fetchUrl, {cache:"no-store"})` — URL изображения из чата (может быть внешним) | GET без данных/заголовков пользователя; ответ — blob картинки | modules/album-image-storage.js:52 | да (пользователь вручную сохраняет/скачивает картинку). Это прямой GET на URL, который уже есть в чате/DOM пользователя; токены не передаются |
| `POST /api/assets/download` (ST) | `{url, category:"temp", filename}` — URL картинки, выбранной пользователем; скачивание делает сам сервер ST (ограничено allowlist-хостами ST) | modules/album-image-storage.js:85 | да, штатный API, fallback при CORS-ошибке |
| `fetch('/assets/temp/<file>')`, `POST /api/assets/delete` | чтение/удаление собственного временного файла | album-image-storage.js:103, 66 | да |
| `POST /api/images/upload` | `{image: base64, format, filename, ch_name}` с `getRequestHeaders()` (CSRF) — на свой сервер ST | album-image-storage.js:183 | да (сохранение картинки в альбом) |
| `generateRaw(...)` (ST) | промпт саммари/фактов/квестов с фрагментом чата -> тот LLM-провайдер, который пользователь сам выбрал в ST (или выбранный профиль Connection Manager) | index.js:2615-2620 | да (заявленная функция; это не прямой сетевой код расширения) |

`getRequestHeaders()` используется только в вызовах к собственному серверу ST (album-image-storage.js:20-40).

## Находки
1. 🟡 низкая — `cleanMessage()` присваивает текст сообщения в `innerHTML` отсоединённого `<div>` (modules/utils/chat-utils.js:15-19):
```js
const div = document.createElement("div");
div.innerHTML = text;
return div.textContent || "";
```
   Для вырезания HTML-тегов. Если в сообщении есть `<img src=x onerror=...>`, обработчик на отсоединённом узле сработает (классический DOM-XSS через парсинг). Источник — текст чата (чужие карточки/сообщения ИИ), т.е. вектор «злая карточка персонажа», а не бэкдор расширения. Рекомендация: использовать `DOMParser` (`parseFromString(...,'text/html')`) вместо `innerHTML` у живого документа. Серьёзность низкая.
2. 🟡 низкая / информационно — опция `bypassFilter` (index.js:2601-2610) заменяет в промпте обычные пробелы на U+2007 (`finalPrompt.replace(/ /g, " ")`) — приём обхода фильтров провайдера LLM. Включается пользователем вручную (по умолчанию `false`, index.js:4530). Не угроза пользователю, но может нарушать правила провайдера.
3. ℹ️ информационно — временное переключение профиля подключения (`switchProfile`, modules/utils/runtime-profile-text-utils.js:~100-140; использование index.js:2822-3054, 3081-3186): расширение программно меняет значение `#connection_profiles` и после генерации возвращает исходный профиль. Это функция «независимый API-профиль» из README; профиль выбирает пользователь (`connectionProfileId`), ключи расширение не читает (`apiKey/api_key/secret/token` явно внесены в `SUNNY_SETTINGS_PRESET_FORBIDDEN_KEYS`, index.js:4366-4371, т.е. не экспортируются в пресеты). Если генерация упадёт между переключениями, профиль может остаться изменённым (баг-риск, не вредоносность).
4. ℹ️ информационно — глобальные слушатели `document.addEventListener('pointerup'|'click'|'keydown', checkAndAttach, true)` (index.js:6973-6975) — это не кейлоггер: обработчик только ищет `.iig-lightbox` и снимает себя; `keydown` на документе в album-module.js:1204 реагирует только на `Escape`. Данные с клавиатуры не читаются.
5. ℹ️ информационно — слэш-команды `/sunny-summary`, `/sunny-facts`, `/sunny-quests`, `/sunny-events`, `/cancelmem` (index.js:8666-8710): запускают собственную генерацию.
6. ℹ️ информационно — рассинхрон версий (manifest 1.3.0 vs README 1.2.0), коммиты с названиями вроде «бубебе2.html», «попа.js»: стиль разработчика, содержимое проверено (см. историю).

## Что проверено и не найдено
- Нет: `eval`, `new Function`, `setTimeout`/`setInterval` со строкой, `atob`/`btoa`, `String.fromCharCode`, `document.cookie`, `localStorage`/`sessionStorage`/IndexedDB, `XMLHttpRequest`, `sendBeacon`, `WebSocket`, `EventSource`, `new Image().src`, `import()`, `importScripts`, создания `<script>/<iframe>/<link>`, `window.open`, `location.href=`; нет длинных (≥100 символов) base64/hex-строк, нет скрытых unicode (только BOM в начале ai-event-generator.js, event-parser.js, quest-module.js), нет текста, адресованного ИИ-аудитору («игнорируй…», «безопасно»).
- Нет обращений к `/api/secrets/*`, `secret_state`, `oai_settings`, reverse proxy/паролям; `extension_settings` читается/пишется только по ключу `SunnyMemories` (и `connectionManager.profiles` — только имена/ID профилей для выпадающего списка).
- Чтение чата (`getContext().chat`) идёт только для формирования промпта локальной генерации и инъекции; пересылки наружу, кроме выбранного пользователем LLM, нет. Нет телеметрии/отпечатков (IP, UA, ID), вебхуков, акронимов сокращателей URL.
- Нет `package.json`, `plugins/`, серверного кода, `child_process`, `fs`, бинарников, wasm (`git ls-files` = 35 файлов, `file` не нашёл ничего нетекстового).
- settings.html (4838 строк, 262 КБ): только `<style>`, разметка, data-i18n, кнопки; тегов `<script>`, inline-обработчиков `on*=`, внешних `src/href` нет (`<img class="sm-album-image-viewer-img" src="">` пустой).
- Отложенных триггеров по дате/счётчику/пользователю нет (даты используются только для внутреннего календаря RP).
- Экспорт/импорт: JSON скачивается через Blob+`<a download>` (index.js:5841-5855), пресеты импортируются через `JSON.parse` с белым списком ключей (index.js:4974, 4366+); отправки наружу нет.
- История (`git log -p --all -S...` по fetch( / http / eval( / atob / XMLHttp / sendBeacon / WebSocket / new Function / secrets/ / api_key / document.cookie / localStorage / import( ): fetch/http встречаются только в коммитах 9780646/8f346f9/7997ad5 (перенос album-image-storage в отдельные модули — те же вызовы к `/api/assets/*`, `/api/images/upload`), в a3910b0 (SVG data-URI с w3.org), d29ab0e (бейджи shields.io в README). `api_key` — только 7997ad5 (список запрещённых ключей пресетов). Остальные поиски пустые. Удалений вредоносного кода не обнаружено; все коммиты одного автора, один адрес e-mail.
- Ветка main единственная, тегов/релизов нет, расхождений ветка/релиз нет.

## Просмотренные файлы (35 в git)
README.md, manifest.json, style.css, index.js (308 КБ — просмотрен grep-ами по всем категориям чек-листа + выборочное чтение импортов, инициализации, safeGenerateRaw, профилей, слэш-команд, экспорта; не читался построчно целиком), settings.html (грепом + начало файла), modules/: ai-event-generator.js, album-character-binding.js, album-diary.js, album-image-storage.js (прочитан целиком), album-module.js, album-viewer-ui.js, calendar-core.js, constants.js, context-injection-module.js, event-parser.js, i18n-translations.js (103 КБ, только grep), i18n-ui.js, library-module.js, message-popover-buttons.js, quest-module.js, settings-help-ui.js, summary-engine.js, summary-ui.js; modules/utils/: album-core-utils.js, album-media-utils.js (целиком), calendar-chat-utils.js, calendar-signal-utils.js, chat-scope-utils.js, chat-utils.js (целиком), common-utils.js, context-anchor-utils.js, date-parse-utils.js, parser-utils.js, runtime-profile-text-utils.js, ui-tab-utils.js.
Бинарников нет.

## Уверенность
Высокая (~90%) в отсутствии эксфильтрации и вредоносного поведения: все сетевые примитивы, динамический код, секреты, хранилища и обфускация проверены grep-ом по всему дереву и истории, а сетевой код (album-image-storage.js, album-media-utils.js) прочитан целиком. Не проверено построчно: ~300 КБ index.js и крупные модули (логика UI/промптов) — прочитаны не полностью; в принципе вредоносная логика, не использующая ни одного из искомых API (все они ушли бы через те же ST-API), выглядит маловероятной, но полным ручным ревью это не подтверждено. Поведение зависит от внутренних API ST (generateRaw, Connection Manager) — проверено только по коду расширения.

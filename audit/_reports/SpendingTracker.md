# Аудит: SpendingTracker (niemandswasser/SpendingTracker)

**Вердикт:** 🟢 Чисто

Версия манифеста 1.3.2, 3 коммита (15.07.2026, один автор), ветка только `main`, тегов нет.

## Что делает расширение (по коду)
Подписывается на события генерации ST (`CHAT_COMPLETION_PROMPT_READY`, `GENERATION_ENDED` и др.), считает токены ввода и вывода встроенным токенайзером ST (`getTokenCountAsync`) и складывает дневную статистику в `extension_settings.spendingTracker`. Рисует дашборд (SVG-графики, хитмап, ачивки, викторина, цены моделей, экспорт и сброс). Цены в $ задаёт пользователь вручную.

## Проверка на чтение и отправку ключей и баланса
- Ни `/api/secrets/*`, ни `secret_state`, ни `oai_settings`, ни `api_key`, `password`, `proxy`, `Authorization`, `document.cookie` в коде нет (проверено grep по всем файлам, включая CSS и manifest).
- Баланс и расходы у провайдера расширение не запрашивает. «Расходы» считаются локально: токены умножаются на цену за 1M токенов, которую пользователь вводит сам (`dashboard.js:116-124` `dayCost`).
- Единственные сетевые запросы идут на собственный сервер ST (таблица ниже). Внешних доменов, `XMLHttpRequest`, `sendBeacon`, `WebSocket`, `EventSource`, `new Image`, `<script>/<iframe>` и `import()` с URL нет. Единственная строка `http` в коде — XML-namespace `http://www.w3.org/2000/svg` в SVG, и в LICENSE (Apache) — URL лицензии.

## Сетевые адреса
| Домен/URL | Что отправляется | Файл:строка | Обоснованно? |
|---|---|---|---|
| `/api/stats/get` (свой сервер ST) | `{}` + `getRequestHeaders()` (CSRF) | dashboard.js:551 | Да: вкладка «Общение», статистика по персонажам. Ответ рисуется только локально |
| `/api/stats/recreate` (свой сервер ST) | `{}` | dashboard.js:566 | Да: кнопка «Обновить» |
| `/api/chats/search` (свой сервер ST) | `{query:'', avatar_url}` | dashboard.js:578 | Да: подсчёт числа чатов персонажа |
| Внешние домены | — | — | Не найдено |

## Находки
Серьёзных и подозрительных находок нет. Информационные пункты:

1. **Динамический `import()` соседнего расширения и monkey-patch** (🟡 информационно) — `index.js:246-267`:
   ```js
   const module = await import('../ext-blocks-custom/src/services/ApiService.js');
   ...
   const original = api.generateBlocks;
   api.generateBlocks = async function (messages, apiPresetName) {
       const result = await original.call(this, messages, apiPresetName);
       try { await trackExtBlocks(this, messages, apiPresetName, result); } catch ...
       return result;
   };
   ```
   Расширение оборачивает метод чужого расширения ExtBlocks, чтобы считать его токены. Это задокументировано в README («Интеграция с ExtBlocks»). Путь относительный, не внешний URL. `trackExtBlocks` (`index.js:269-297`) читает `result.usage`, текст ответа и имя модели профиля (`getConnectionProfile(...).model`) и только прибавляет числа в локальную статистику. Ключ или URL профиля не читаются, результат возвращается без изменений. Это правка поведения чужого расширения, но безвредная. Серьёзность: низкая.
2. **Чтение содержимого чата и промпта** — `index.js:117-130, 151-179`: берёт `data.chat` / `data.prompt` и `chat[last].mes` только для подсчёта токенов. Текст не сохраняется и не передаётся дальше, в статистику пишутся лишь числа. Нормально для заявленной функции.
3. **Имена персонажей и аватарки в настройках** — `index.js:229-234` сохраняет `name` и `avatar` персонажа в `extension_settings.spendingTracker.daily[...].chars`. Эти данные попадают в `settings.json` самого ST и в экспорт (`dashboard.js:990-1002` выгружает только `prices` и `daily` в локальный JSON-файл через `Blob`, без сети). Не утечка.
4. **Вывод в HTML** — `innerHTML` используется со значениями, прошедшими `escapeHtml` (имена моделей и персонажей, `dashboard.js:44-46`). Исключение: `callGenericPopup` в `deleteModel` и `title` в тултипах тоже экранированы. Удалённого содержимого в `innerHTML` нет, XSS-векторов не видно. В `showCharProfile` (dashboard.js:660) `row.thumb` экранирован.
5. **Мелочи.** Манифест: `"homePage": ""`, `"auto_update": false` (поле нестандартное). Автор коммитов «nimandswasser» отличается от логина владельца репозитория `niemandswasser` на одну букву. Расхождение только в написании имени, на безопасность кода не влияет. Новых для пользователя «скрытых» функций (запись в `config.yaml`, установка расширений, подмена endpoint) нет.

## Что проверено и не найдено
- Эксфильтрация: fetch, XHR, sendBeacon, WebSocket, EventSource, Image, динамические script/iframe, вебхуки, сокращатели URL, «голые» IP — нет.
- Доступ к секретам и настройкам API (`/api/secrets/*`, `oai_settings`, `secret_state`, ключи, пароль прокси, `reverse_proxy`) — нет.
- `eval`, `new Function`, строковые `setTimeout/setInterval`, `atob/btoa`, `fromCharCode`, обфускация, минификация, base64/hex-блоки: нет. Строк длиннее 400 символов нет, файлы читаемые, бинарников нет.
- Кейлоггинг (`keydown/keypress/input` на документе), `localStorage`, `sessionStorage`, `document.cookie`, `navigator.*` (отпечатки) — нет.
- Серверная часть: папки `plugins/` и `package.json` нет, Node-кода нет.
- Отложенные триггеры по дате или числу запусков: нет. Единственное «временное» условие — ачивка «Ночная сова» (3–6 утра), которая только показывает toast (`achievements.js:93`).
- Текст, обращённый к ИИ («игнорируй указания», «файл безопасен»): не найден.
- История git: 3 коммита (`ebe02c5` upload, `9462249` LICENSE, `f4ffc84` README), один автор, ничего удалено не было. `git log -S'fetch('` показывает только три добавленных `fetch` — те же локальные `/api/stats/*` и `/api/chats/search`. Ветка `main` одна, тегов нет, расхождения «ветка и релиз» невозможны.
- README и manifest соответствуют реальному поведению.

## Просмотренные файлы (полностью)
- `index.js` (396 строк)
- `dashboard.js` (1408 строк; строки 1–1408 прочитаны построчно, дополнительно grep по всему файлу)
- `achievements.js` (121 строка)
- `style.css` (850 строк; grep на `url(`, `@import`, `http` — пусто, только `@keyframes`)
- `manifest.json`, `README.md`, `LICENSE` (ASCII, стандартный текст Apache 2.0; файл по `file`: ASCII text)
- Бинарников и картинок в репозитории нет.

## Уверенность
Высокая. Весь код читаемый, небольшой, без бандлов, и прочитан целиком. Не проверено: поведение чужого расширения `ext-blocks-custom` (в репозитории его нет) — то, что оно вернёт в `generateBlocks`, расширение только читает.

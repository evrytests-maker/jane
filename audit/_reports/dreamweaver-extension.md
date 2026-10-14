# Аудит: dreamweaver-extension

Репозиторий: https://github.com/ann22cadis/dreamweaver-extension (/home/user/jane/audit/dreamweaver-extension), HEAD 4051cc9.

## Вердикт: 🟢 Чисто
(есть одна мелкая гигиеническая проблема — неэкранированный HTML, см. находку №1, серьёзность низкая; вредоносного поведения нет)

## Что делает расширение
Панель «Dreamweaver» (сюжетный пульт): пользователь отмечает теги (жанр, AU, тропы, стиль, поведение и т.д.), а расширение собирает соответствующие текстовые промпты из локального prompts.json (+ пользовательские теги) и вставляет их в запрос к LLM через `context.setExtensionPrompt(...)` в обёртке `<dreamweaver>...</dreamweaver>`. Состояние хранится в `localStorage` под ключом `dw_v2_data`.

## Сетевые адреса
| Домен/URL | Что отправляется | Файл:строка | Обоснованно |
|---|---|---|---|
| `/scripts/extensions/third-party/dreamweaver-extension/prompts.json` (свой сервер ST, относительный путь) | ничего (GET) | index.js:48 | да, загрузка своего ресурса |
| `/scripts/extensions/third-party/dreamweaver-extension/template.html` (свой сервер ST) | ничего (GET) | index.js:59 | да |
| `/scripts/extensions/.../img/icon.png`, `/characters/<avatar>` (свой сервер ST) | ничего (img src) | index.js:338, 305/309, 321, 332 | да |

Внешних доменов нет. Нет `XMLHttpRequest`, `sendBeacon`, `WebSocket`, `EventSource`, `import()`, axios, `$.ajax/$.post/$.get`, динамической подгрузки скриптов. Нет обращений к `/api/secrets/*`, `/api/settings/*`, `/api/chats/*`, `oai_settings`, `document.cookie`, `getRequestHeaders`.

## Находки
1. **Низкая (гигиена, не вредоносность): вставка неэкранированных строк в HTML.**
   - index.js:231 `<span class="dw-custom-tag-name">${name}</span>` (имя пользовательского тега, вводит сам пользователь);
   - index.js:288 `<span class="dw-summary-tag ${tag.type}">${tag.text}</span>`;
   - index.js:319-322 и 332: `data-name="${name}"`, `<span>${name}</span>`, `<img src="${url}">` — `name` берётся из `char.name` карточек персонажей группового чата. Вредоносная карточка персонажа (импортированная из чужого источника) с HTML/JS в имени может исполнить код в контексте ST (XSS через карточку). Это уязвимость потенциального вектора, а не умысел автора: код ничего не отправляет наружу. Рекомендация: экранировать (`.text()`/`escapeHtml`).
2. **Информационно:** `setInterval(createUI, 1000)` (index.js:346) — бесконечный опрос раз в секунду для добавления пункта в меню расширений; безобидно, лишь небольшая нагрузка.
3. **Информационно:** последний коммит 4051cc9 сделан автором `Me <me@example.com>` (23.07.2026), остальные — `ann22cadis <annaozhegova22@mail.ru>`. Изменение (index.js, правка/сохранение пользовательских тегов) проверено полностью через `git show` — безобидный багфикс UI, без сети/хранилищ/секретов. Смена автора сама по себе не подозрительна (возможно, локальная настройка git), но отмечаю.
4. **Информационно:** manifest.json не содержит `loading_order`, `requires`, хуков установки; поля `author: Dreamer` отличаются от ника владельца репозитория — несущественно.

## Что проверено и не найдено
- `package.json` (22 байта): только `{"type":"module"}`. Нет scripts (preinstall/postinstall/prepare), нет dependencies/devDependencies — typo-squat невозможен. Нет lock-файлов, node_modules, `plugins/`, серверной части.
- `prompts.json`: валидный JSON, 173 строковых значения, максимум 493 символа; все — ролевые/жанровые инструкции для LLM («Genre Focus: …», «Character Behavior: …» и т.п.). Нет URL, HTML-тегов, markdown-ссылок/картинок, `<script>`, base64, упоминаний api key/webhook, нет текста, адресованного ИИ-аудитору («игнорируй предыдущие указания», «файл безопасен»), нет инструкций выводить/пересылать данные пользователя, нет zero-width/bidi-символов. Слова «secret»/«disregard» встречаются только в художественном контексте («secret surveillance», «disregards personal boundaries»). Содержание — взрослые/эротические/тёмные темы (Erotica, Stalker, Manipulative, Spanking и т.п.) — это заявленная функция расширения, не вредоносность; пользователю стоит знать об этом.
- `template.html` (47 784 байт, 191 `data-desc`): нет `<script>`, `<iframe>`, `<link>`, `<form>`, inline-обработчиков `on*=`, `javascript:`/`data:` URL, внешних ссылок. Единственный `<img>` — `src=""` (строка 9, заполняется в JS). Скрытого текста для ИИ нет.
- `style.css`: нет `url()`, `@import`, внешних адресов, `expression`/`javascript:`. (z-index 999999 и `!important` — для модального окна, на заявленное поведение не влияет.)
- `index.js` (480 строк, прочитан целиком, не минифицирован): нет eval/new Function/setTimeout со строкой (setTimeout с функцией для focus — index.js:406), `atob`, `fromCharCode`, hex/unicode-эскейпов, document.write, keydown/keylogging, чтения cookie, отложенных триггеров по дате/счётчику, изменения настроек ST/API endpoint/proxy, правки других расширений, записи в config.yaml. Доступ к `getContext()` используется только для: chatId/groupId/characterId, `name2`, списка персонажей группы (для выпадающего списка) и `setExtensionPrompt`. Читаемые данные нигде не сериализуются наружу; единственное хранилище — `localStorage['dw_v2_data']` (теги, пользовательские промпты, глубина) — index.js:36-42.
- Git-история (8 коммитов, одна ветка main, тегов нет, удалённая ветка origin/main = HEAD): `git log --all -S` по `http`, `eval`, `atob`, `XMLHttp`, `sendBeacon`, `WebSocket`, `<script`, `api/` — ни одного совпадения ни в живом, ни в удалённом коде; `-S'secret'` — только добавление prompts.json в первом коммите (художественный текст). Просмотрены диффы всех коммитов: b8de3fe (первоначальная загрузка), eee2367/3eefdaf (placeholder.txt, содержимое — пустая строка), 8d1a642 (img/icon.png), b15dad0 (extensionName "dreamweaver" -> "dreamweaver-extension" — чтобы путь совпадал с именем папки), 50d0c6f (focus textarea при раскрытии), c9b2552 (CSS мобильной раскладки), 4051cc9 (багфикс тегов). Рабочая копия чистая, расхождений с HEAD нет; git-хуков (кроме sample) нет.
- Бинарники: `img/icon.png` — `file`: PNG image data, 24x24, 8-bit RGBA, 501 байт; исполняемых файлов, .wasm, архивов нет.
- Совпадение заявленного и реального: manifest/README («Сюжетный пульт») соответствует коду. README в репозитории отсутствует.

## Просмотренные файлы
index.js, manifest.json, package.json, prompts.json, template.html, style.css, img/icon.png (тип проверен через `file`), вся git-история (8 коммитов).

## Уверенность
Высокая (≈95%). Все файлы небольшие, не минифицированы, прочитаны/просканированы целиком; история короткая и проверена. Не проверялось: зависимости самого SillyTavern (импорты `../../../../script.js` и `../../../st-context.js` — штатные модули ST), поведение в рабочем браузере (код не запускался по правилам аудита). Единственное замечание — потенциальный XSS из-за неэкранированных имён (находка №1).

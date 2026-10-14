# Аудит: Imaginy (aika636/Imaginy)

Репозиторий: https://github.com/aika636/Imaginy, локальная копия `/home/user/jane/audit/Imaginy`, ветка `main` @ `d66e46c` (v0.7.0).
Режим аудита: только чтение (cat/grep/sed, git log/show). Ничего не запускалось, тесты не исполнялись.

## Вердикт: 🟢 Чисто

Вредоносных функций и утечек данных не найдено. В коде нет ни одного сетевого вызова. Есть несколько осознанных вмешательств в чужие данные и настройки (подробно ниже). Они не скрыты, все связаны с заявленной функцией, но README описывает их неполно.

## Что делает расширение (по коду)

Надстройка над расширениями «inline image generation» (SLAY Images, delidgi/sillyimages, notsosillynotsoimages, 0xl0cal/sillyimages 2.0 и его форки).
- Добавляет кнопку-карандаш на картинки с атрибутом `data-iig-instruction` и открывает модальный редактор промпта, стиля, соотношения сторон и размера.
- Записывает правку обратно в текст сообщения чата (`mes`, `swipes`, `extra.display_text`, `extra.extblocks`) и в DOM, хранит историю промпта в `message.extra.imaginy` и сохраняет чат.
- При желании программно кликает по кнопке перегенерации хост-расширения.
- Дополнительно (`srcsync.js`) выравнивает путь к перегенерированной картинке во всех хранилищах текста сообщения.

## Как взаимодействует с sillyimages (0xl0cal) и другими хостами — прямой ответ на вопрос

Файл `src/hosts/sillyimages.js` — это только описание профилей (селекторы, имя модуля настроек, имена кнопок). Сам он ничего не делает. Выводы по всей кодовой базе:

1. Код чужого расширения не правится. Нет `eval`, `import()`, подмены функций/прототипов, патчинга модулей, `<script>`, загрузки файлов хоста. Взаимодействие идёт через DOM (поиск селекторов `.iig-img-host`, `.iig-img-regen`, `.iig-img-retry`, `.iig-regen-single-btn` и т.п.) и через объект настроек ST.
2. Настройки хоста (`extensionSettings.inline_image_gen`, ключ модуля у 0xl0cal и у delidgi общий):
   - Чтение. `src/host.js:64-70` (проверка наличия ключей настроек для опознания форка), `src/hostquirks.js:37-44, 49-128` (читает `apiType`, `naisteraAspectRatio`, `aspectRatio`, `imageSize`, `promptDriven`, `overrideImageSize`, `styles[]`, `activeStyleId`). Значения ключей API, endpoint и промпты не читаются. Читаются только поля, перечисленные выше.
   - Запись. Есть ровно две функции, обе вызываются только по явному клику пользователя на кнопку-подсказку в редакторе (`src/editor.js:471` и `src/editor.js:513`):
     - `clearHostStyle()` (`src/hostquirks.js:132-172`). Для профиля 0xl0cal/delidgi: если в DOM есть `#iig_style_disable` (кнопка форков delidgi), кликает по ней. Иначе пишет `s.activeStyleId = ''` и вызывает `saveSettingsDebounced()` (стр. 157-166). Для 0xl0cal 2.0 такой кнопки нет (проверено grep по sillyimages), поэтому сработает прямая запись `activeStyleId = ''`. Панель настроек хоста при этом не перерисуется.
     - `setAspectRatioFromPrompt()` (`src/hostquirks.js:176-196`). Выполняется только при `quirks.aspectAuto === true`, а это только профиль SLAY (`src/hosts/slay.js:55`). Для sillyimages `aspectAuto:false` (`src/hosts/sillyimages.js:86,145`), то есть функция там инертна.
   - Никаких других записей в `extensionSettings` хоста нет. Свои настройки Imaginy хранит в `extensionSettings.Imaginy` (`src/settings.js`).
3. Клики по кнопкам хоста: `src/regen.js:69` (`check.btn.click()`) после «Сохранить и перегенерировать». Для 0xl0cal 2.0 кликается `.iig-img-regen` / `.iig-img-retry`, для версий до 2.0 — `.iig-regenerate-btn` из меню сообщения (`src/hosts/common.js`). Клик возможен только при явном действии пользователя либо при включённой им настройке `regenerateAfterSave` (по умолчанию выключена, `src/settings.js:14`).
4. Данные чата (то, что реально меняется без «подтверждения» пользователя):
   - `src/persist.js` — переписывает атрибут `data-iig-instruction` в тексте сообщения. Только по кнопке «Сохранить».
   - `src/srcsync.js:278-325` (mirror) и `:179-215` (reconcile) — автоматически, без клика: при смене `src` у `img[data-iig-instruction]` подставляет новый путь во все 6 хранилищ текста сообщения (включая чужие свайпы через `replaceSrcEverywhere`, подстрочная замена) и вызывает `saveChat()` (стр. 331-345). Reconcile выполняется при каждом открытии чата (`CHAT_CHANGED`, `APP_READY`). Выключатель: настройка `syncImageSrc` (по умолчанию включена). Это изменение содержимого чатов хостом-независимо, но ограничено путями картинок.
   - `src/history.js:73-92, 148-175` — пишет `message.extra.imaginy.promptHistory` (до 3 прошлых версий промпта) в файл чата. Эти промпты уйдут вместе с чатом, если пользователь им поделится.
5. Несоответствие README: раздел «Приватность» говорит «только редактирует данные внутри вашего чата и нажимает кнопку расширения-хоста». Правка настроек хоста (п.2) и автоматическая запись путей (п.4) здесь не упомянуты. Это не вредоносность, а неполная документация.

## Сетевые адреса

Внешних и вообще каких-либо сетевых запросов в коде нет. Проверено grep по `fetch|XMLHttpRequest|sendBeacon|WebSocket|EventSource|axios|\$.ajax|\$.post|import\(|new Image|\.src\s*=|<script|<iframe|http` по всем JS/HTML/CSS.
Единственный «сетевой» вызов — `$.get(new URL('../settings.html', import.meta.url).href)` в `src/settings.js:94`: загрузка собственного файла расширения с сервера ST (локально). Обращений к `/api/secrets/*`, `getRequestHeaders()`, `/api/settings`, `/api/chats`, `/api/characters` нет.

| Домен/URL | Что отправляется | Файл:строка | Обосновано |
|---|---|---|---|
| (внешних нет) | — | — | — |
| `../settings.html` (свой сервер ST) | ничего (GET) | `src/settings.js:94` | да |

## Находки

Критичных и высоких нет. Низкие и информационные:

1. Низкая. Запись в настройки чужого расширения по клику. `src/hostquirks.js:132-172`:
   ```js
   const ownBtn = document.getElementById('iig_style_disable');
   if (ownBtn) { ownBtn.click(); ... return true; }
   s.activeStyleId = '';
   getCtx()?.saveSettingsDebounced?.();
   ```
   Вызывается из `src/editor.js:471` только по нажатию кнопки «сбросить стиль хоста» (текст подсказки в редакторе явно говорит о перекрытии стиля). Пользователь предупреждён интерфейсом, но не README. Не скрытая подмена.
2. Низкая. Автоматическая перезапись путей картинок и `saveChat()` без клика пользователя (`src/srcsync.js:179-215, 278-346`, `src/persist.js:69-75`). Используется подстрочная замена пути по всем свайпам (`walkMessageStrings`, `persist.js:17-65`). Риск — только функциональный (теоретическая подмена картинки в чужом свайпе), утечки нет. Отключается настройкой `syncImageSrc`.
3. Информационная. Глобальный перехватчик клавиатуры на время открытого редактора: `document.addEventListener('keydown', onKeydown, true)` (`src/editor.js:983`). Обрабатывает только Ctrl/Cmd+Enter и Escape, остальные клавиши пропускает (`editor.js:968-981`), ничего не записывает и не отправляет. Снимается при закрытии (`editor.js:918-925`).
4. Информационная. `src/i18n.js:316` читает `localStorage.getItem('language')` — язык интерфейса ST, никуда не отправляется. Чтение `navigator.platform/userAgent/language` (`editor.js:73`, `i18n.js:322`) используется только для выбора подписи хоткея и языка.
5. Информационная. Диагностика геометрии окна (`editor.js:884-896`) выводится только в тост и `console.log`, не отправляется.
6. Информационная. `innerHTML` используется только с константной строкой иконки (`src/decorate.js:78`). Данные из чата вставляются через `textContent`/`.value`. Настройки-панель — собственный `settings.html`.
7. Информационная. `manifest.json`: `auto_update: false`, версия 0.7.0, автор aika636, зависимостей нет.

## Что проверено и не найдено

- `eval`, `new Function`, `setTimeout/setInterval` со строкой, динамические `import()`, `<script>`, загрузка чужого кода: нет.
- Обфускация: `atob`, `btoa`, `String.fromCharCode`, длинные base64/hex, `.wasm`, минифицированные файлы, бинарники: нет. Единственные не-текстовые файлы — отсутствуют; `LICENSE` и `.gitignore` — ASCII text (`file`).
- `/api/secrets/*`, `secret_state`, `oai_settings`, `api_key`, `password`, `token`, `document.cookie`, `getRequestHeaders`, `sendBeacon`, `WebSocket`: нет вхождений в коде (в том числе в истории git: `git log --all -S` по `fetch(`, `eval`, `atob`, `secrets`, `XMLHttpRequest`, `sendBeacon`, `WebSocket`, `child_process`, `Function(`, `cookie`, `password`, `api_key`, `oai_settings`, `getRequestHeaders`, `import(` — совпадений нет; `http` встречается только в комментарии со ссылкой на SLAY и в README).
- Серверной части (`plugins/`), `package.json`, `postinstall` нет.
- Чтение чата: `getCtx().chat` используется только для текущей картинки/сообщения и для сверки путей (`history.js:131`, `persist.js:406`, `srcsync.js:188,272`) без отправки куда-либо.
- Отложенные триггеры (даты, счётчики, IP, user-id): нет. Единственные таймеры — UI-дебаунсы и рескан DOM.
- Текст, адресованный ИИ (prompt injection), в коде, README, CHANGELOG, комментариях: не найден.
- История git: 25 коммитов, один автор (`aika636`), без чужих коммитов. Ветки `origin/i18n` и `origin/prompt-history` — предки `main` (`git merge-base --is-ancestor`), тегов нет, расхождений между default-веткой и релизами нет.

## Покрытие (просмотренные файлы)

Прочитаны целиком: `index.js`, `manifest.json`, `settings.html`, `style.css` (grep на `url(`/`@import`/`expression` — пусто), `src/ctx.js`, `src/host.js`, `src/hostquirks.js`, `src/hosts/common.js`, `src/hosts/generic.js`, `src/hosts/nsnsi.js`, `src/hosts/sillyimages.js`, `src/hosts/slay.js`, `src/log.js`, `src/version.js`, `src/regen.js`, `src/settings.js`, `src/instruction.js`, `src/history.js`, `src/persist.js`, `src/srcsync.js`, `src/decorate.js`, `src/editor.js`.
`src/i18n.js` — логика прочитана полностью; таблицы строк ru/en просмотрены выборочно и проверены grep на HTML/URL/script (нет).
`README.md`, `README.en.md`, `CHANGELOG.md`, `LICENSE` — проверены grep на ссылки, текст для ИИ и заявления о приватности; полностью построчно не читались.
Тесты `tests/*.mjs` (editor, history, hosts, i18n, instruction, persist, settings, srcsync) — только просмотрены grep-ом по импортам/сети/`process`/`fs` (импортируют только `jsdom`, `node:fs`, `../src/*`; сети нет), построчно не читались, не запускались.
Бинарников нет (`git ls-files` — только .js/.mjs/.json/.html/.css/.md/LICENSE/.gitignore).

## Уверенность и что не проверено

Уверенность высокая (≈95%). Весь исполняемый код прочитан. Не проверялись: сама работа в браузере (не запускалось), корректность предположений об API SillyTavern и о чужих хост-расширениях (кроме сверки имён `activeStyleId`, `.iig-img-regen`, `inline_image_gen` с кодом sillyimages из соседней копии). Код хост-расширений SLAY/delidgi/nsnsi недоступен в рамках задания.

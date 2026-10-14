# Аудит: SillyTavern-CharCardStudio

Репозиторий: https://github.com/MMKAVERAPPA/SillyTavern-CharCardStudio (локально /home/user/jane/audit/SillyTavern-CharCardStudio), HEAD main = 2037ef1, версия в manifest.json 5.2.1.

## Вердикт: 🟢 Чисто

Вредоносных функций и эксфильтрации данных не найдено ни в рабочей копии, ни в истории всех веток. Есть несколько мелких замечаний по гигиене (раздел «Находки», все уровня info/low).

## Что делает расширение (по коду)
ИИ-агент для создания карточек персонажей. Через `generateRaw` / `ConnectionManagerRequestService` из ST шлёт запросы на LLM, который пользователь уже настроил в ST. Модель может вызывать набор локальных инструментов `ccs_*`. Они готовят черновики полей карточки и записей лорбука, которые применяются только после явного клика пользователя. Сессии хранятся в `localforage` (браузер).

## Сетевые адреса

Внешних доменов в коде нет. Единственный URL с внешним доменом в коде `https://github.com/MMKAVERAPPA/CharCardStudio` — ссылка `<a href target=_blank>` в `templates/settings-modal.html:157`, она не вызывается автоматически.

| Назначение | Что отправляется | Файл:строка | Обосновано |
|---|---|---|---|
| `/api/characters/merge-attributes` (свой сервер ST, с `getRequestHeaders()`) | avatar + одно изменённое поле карточки | core/tools.js:822, 973 | да, по клику «Apply» / прямое редактирование поля |
| `/api/worldinfo/get`, `/list`, `/edit` (свой сервер ST) | имя лорбука, записи лорбука | core/lorebook.js:66, 100, 134, 158 | да |
| `/scripts/extensions/<path>/settings.html`, `templates/window.html`, `templates/settings-modal.html` (свой сервер) | ничего | index.js:129, 151; ui/settings-modal.js:79 | да, запасной путь загрузки шаблонов |
| LLM пользователя: `ctx.generateRaw` | промпт агента, поля карточки, лорбук, история сессии | core/silent-generation.js:103 | да, основная функция |
| LLM пользователя через профиль Connection Manager: `ConnectionManagerRequestService.sendRequest` | то же, для фоновых проверок | core/api-router.js:84 | да, профиль выбирает пользователь |
| `/sd` через `ctx.executeSlashCommandsOnChatInput` или `#send_textarea` | промпт аватара | ui/chat.js:1176-1190 | да, по кнопке «Generate Avatar», использует расширение ST Image Generation |

`XMLHttpRequest`, `sendBeacon`, `WebSocket`, `EventSource`, `new Image().src`, `$.ajax`, `axios` и динамический `import()` с URL отсутствуют. Динамические `import()` — только относительные пути к своим модулям. `fetch` во всём репозитории — только на относительные пути своего сервера.

## Инструменты LLM-агента (core/tools.js:46-71)
Реестр `TOOLS`: `ccs_write_field`, `ccs_read_field`, `ccs_update_pillar`, `ccs_batch_pillars`, `ccs_switch_phase`, `ccs_create/read/update/delete_lore_entry`, `ccs_resolve_conflict`, `ccs_update_memory`, `ccs_audit_card`, `ccs_submit_review`, `ccs_set_card_type`, `ccs_set_platform`, `ccs_write/read_brief`, `ccs_write/read_lore_plan`, `ccs_optimize_tokens`, `ccs_semantic_search`, `ccs_read_lore_graph`, `ccs_suggest_lore_connections`, `ccs_generate_avatar_prompt`.

- Чтение: только `getCharacterCardFields()` текущего персонажа (ограничено 3000 символов на поле, core/tools.js:318-332) и записи выбранного лорбука. Секретов, файлов, `/api/secrets/*`, `/api/settings/get`, чатов, других персонажей и персон инструменты не читают.
- Сетевых действий у инструментов нет. Инструмента вроде fetch, exec или чтения произвольного пути нет. Неизвестное имя инструмента отклоняется (core/tools.js:204-207).
- Запись: `ccs_write_field` / `create/update/delete_lore_entry` только создают черновики в сессии. Применение идёт через `applyDraftToCard` / `applyLoreDraft`, которые вызываются только из `_handleDraftAction` по клику пользователя (ui/chat.js:700-710). Автоприменения нет (grep по `applyDraftToCard|applyLoreDraft` подтвердил).
- Вне чёрновиков без подтверждения меняются только внутренняя память расширения (`ccs_update_memory`, localforage), бриф, план, фаза, пиллары. На карточку, лорбук и настройки ST это не влияет.
- В режимах кроме `studio` разрешён только `ccs_read_field` (core/tools.js:197-200).
- Максимум 8 итераций цикла и защита от зацикливания (core/agent.js:17, 189-199).

Вывод: даже при prompt injection через содержимое карточки или лорбука модель через эти инструменты не может ни прочитать секреты, ни отправить данные наружу. Максимум — подготовить вредный черновик, который пользователь увидит и должен подтвердить.

## ui/app.mjs: бандл или нет?
Не бандл. Это обычный читаемый ES-модуль, 102 840 байт (`file`: JavaScript source, UTF-8 text), не минифицирован, с комментариями и именованными импортами. Фактически это более ранняя копия `ui/app.js` (diff ~595 строк, `app.js` — новая версия, 101 820 байт). Файл добавлен в коммите e5d85ac (28 мая 2026) вместе с правкой app.js. Нигде не импортируется и не указан в manifest (grep по `app.mjs` — 0 совпадений), то есть это мёртвый код / забытая копия. В нём нет `fetch`, `eval`, внешних URL и другой подозрительной логики сверх того, что есть в app.js (проверено grep по чек-листу и diff-ом). Единственное существенное отличие — `_injectStyles()` подгружает `style.css` через `<link>` из своей папки расширения.

## Находки (все низкой серьёзности / info)

1. **Info. Мёртвая копия ui/app.mjs** (см. выше). Не вредоносна, но стоит удалить: путает ревью и увеличивает поверхность.
2. **Info. Несоответствие метаданных.** `manifest.json:8,10`: `"author": "DeathGamerSolo"`, `"homePage": "https://github.com/MMKAVERAPPA/CharCardStudio"`, а репозиторий называется `SillyTavern-CharCardStudio`, коммиты от `MMKAVERAPPA`. С `"auto_update": true` (manifest.json:11) ST обновляет расширение из той ветки/репо, откуда оно установлено. Домашняя страница указывает на другое имя репозитория; если репо переименован — это обычный редирект GitHub. Не доказывает вредоносность, но проверьте, что установка идёт именно из этого репозитория. В последнем коммите 2037ef1 из README убрана строка «Built by DeathGamerSolo», автор в manifest остался.
3. **Low. Нет проверки наличия DOMPurify** (ui/chat.js:376-391). Вывод LLM рендерится через showdown и затем `DOMPurify.sanitize` с белым списком тегов и атрибутов. Если `SillyTavern.libs.DOMPurify` недоступен, санитайзинг пропускается и `html` вставляется в `innerHTML` (ui/chat.js:247) как есть. В реальном ST DOMPurify есть, поэтому риск теоретический (XSS от LLM-вывода).
4. **Low. Превью HTML** (modes/html.js:92-95): `iframe.sandbox = 'allow-same-origin'; iframe.srcdoc = htmlContent;` — без `allow-scripts`, скрипты не выполняются. HTML от LLM может содержать `<img>` / `@import` / шрифты на внешних доменах — это привело бы к GET-запросу при превью (пиксель-трекинг), но только по явному действию «Preview». В README режим заявлен как HTML-интро с Google Fonts.
5. **Info. Фоновые проверки отправляют карточку в LLM.** Конфликт-, токен- и валидационные проверки (core/background.js:9-10, 68-69, 118-119 и др.) автоматически отправляют поля карточки в LLM — в основной или в «utility»-профиль Connection Manager (core/api-router.js). Профиль выбирает пользователь в настройках, но не очевидно, что часть данных уходит на другой провайдер, чем основной чат. Это заявленная функция.
6. **Info. Невидимый символ** U+200E (LRM) в prompts/identity.js:147 — внутри примера `<START>` для Janitor, безвредно. Других zero-width / bidi символов в кодовой базе нет.
7. **Info. Скачивание файлов.** Экспорт PNG графа лора (ui/lore-graph-v2.js:2635) и экспорт сессии в JSON (ui/settings-modal.js:345) — через `URL.createObjectURL`, только локальное сохранение.

## Что проверено и не найдено
- Обращения к секретам: `/api/secrets/*`, `secret_state`, `oai_settings`, `api_key`, `password`, `token`, reverse proxy — нет (единственные совпадения со словом «secrets» — текст-пример «What secrets do they hold?» в ui/app.js:351 / app.mjs:357 и `.gitignore`).
- Массовое чтение чатов, персонажей, персон: `ctx.chat`, `/api/chats/*`, `/api/characters/*` кроме `merge-attributes`, `/api/settings/get` — нет. `ctx.characters[characterId]` читается только для текущего персонажа.
- `localStorage` — только heartbeat-замок мульти-вкладок (core/multi-tab.js, ключ `ccs_tab_<avatar>`, значение `{tabId, timestamp}`), `BroadcastChannel` — только между вкладками того же origin. `document.cookie`, `navigator.userAgent`, телеметрия, отпечатки — нет.
- `eval`, `new Function`, `setTimeout` / `setInterval` со строкой, `document.write`, загрузка скриптов (`createElement('script')`), `Worker`, `WebAssembly` — нет. Динамический `<link>` только на свой `style.css`.
- Обфускация: строк base64 / hex длиннее 80 символов нет, `atob`, `btoa`, `fromCharCode` нет, минифицированных файлов и бинарников нет (картинок, `.wasm`, `.so` в репозитории нет).
- Изменение настроек ST: пишется только `extensionSettings.CharCardStudio` (themeSync, utilityApiProfileId; core/api-router.js:22, ui/settings-modal.js:192) и `saveSettingsDebounced`. Подмены endpoint / proxy, правки чужих расширений, установки других расширений нет.
- Кейлоггинг: обработчики `keydown` / `input` только на собственных элементах расширения (чат-поле, заметки, горячие клавиши окна), отправки нигде нет.
- Отложенные триггеры (даты, счётчики запусков) — нет; `Date` используется только для меток времени и форматирования.
- Серверной части и `package.json` нет, `child_process` / `fs` нет.
- Текст, обращённый к ИИ-ревьюеру («ignore previous», «this file is safe»), не найден. Промпты в `prompts/*` — инструкции для самого агента карточек, без команд на утечку.
- Слова-триггеры в промптах (`prompts/phase-instructions.js` и др.) — только про построение карточек.

## История (git)
- 107 коммитов во всех ветках, авторы только `M M Kaverappa` / `MMKAVERAPPA` (один и тот же аккаунт `151958137+MMKAVERAPPA@users.noreply.github.com`). Посторонних авторов нет.
- Теги: v3.1.0, v3.3.0, v5.2.0. v5.2.0 = fd2580d (merge PR #7), HEAD main = 2037ef1 (только правка README после тега, код идентичен). Ветки: main, testing, testing-two; `testing-two` — предыдущая dev-ветка (+ ui/task-monitor.js, в ней нет сети / eval / секретов), `testing` — очень старая версия.
- `git log -S` по `fetch(`, `http`, `eval(`, `atob`, `secrets`, `XMLHttpRequest`, `sendBeacon`, `WebSocket`, `new Function`, `child_process`, `api_key`: удалённого вредоносного кода нет. Из исторических находок:
  - до коммита 1aafa7e (v3.5.0 → v4.0.0) и в ветке `testing` (core/api.js:98) был «utility endpoint»: `fetch(settings.utilityEndpoint + '/chat/completions')` с заголовком `Authorization: Bearer <utilityApiKey>`. Адрес и ключ вводил сам пользователь в настройках расширения (placeholder `https://openrouter.ai/api/v1`), отправлялся промпт утилитарной задачи. Это заявленная функция, тогда использовался и `/api/profiles/apply` (переключение профиля ST на время вызова с возвратом). В актуальной версии этого кода нет.
  - в v2.5.0 (37d58f6) в CSS был `@import url('https://fonts.googleapis.com/...')` (Google Fonts), в актуальном style.css внешних URL нет.
  - в 08804dc / 9671801 — вспомогательный скрипт (Playwright) с `http://127.0.0.1:8000/`, позже удалён; не запускался и в актуальном дереве его нет.
- Изменения manifest.json по истории — только поднятие версий и удаление `"css"` (e5d85ac, стили подгружаются динамически с собственного пути).

## Просмотренные файлы
Полностью прочитаны: index.js, manifest.json, settings.html, core/agent.js, core/api-router.js, core/st-context.js, core/tools.js (все ~1700 строк), modes/html.js, фрагменты core/lorebook.js (1-200), core/silent-generation.js, core/multi-tab.js, core/session.js, core/tools-fallback.js, ui/chat.js (рендер markdown, действия с черновиками, отправка /sd), ui/settings-modal.js (экспорт), ui/lore-graph-v2.js (экспорт PNG), prompts/identity.js (фрагмент с невидимым символом), templates/*.html (grep ссылок и скриптов).

Просмотрены через grep по полному чек-листу (fetch / XHR / WebSocket / eval / Function / atob / import() / src / href / innerHTML / localStorage / cookie / secrets / key / password / slash / extensionSettings / getContext и т.д.) и по характерным участкам: core/background.js, core/coherence-audit.js, core/field-history.js, core/pillars.js, core/session-memory.js, core/token-utils.js, core/validators.js, prompts/identity.js, prompts/mode-prompts.js, prompts/phase-instructions.js, modes/fictionlab.js, modes/imageprompt.js, modes/janitor.js, ui/app.js, ui/app.mjs (+ полный diff с app.js), ui/lore-graph-v2.js, ui/mode-panel.js, ui/personality-radar.js, ui/prompt-inspector.js, ui/toast.js, templates/settings-modal.html, templates/window.html, style.css (на внешние URL и `@import`), eslint.config.js, .eslintrc.json, .gitignore. Документы CHANGELOG.md, CONTRIBUTING.md, Plan.md, README.md, LICENSE просмотрены выборочно (на ссылки, команды установки).

Бинарников и изображений в репозитории нет (проверено `find` по `*.png|jpg|wasm|bin|so`).

## Уверенность
Высокая (~90%) в вердикте «чисто». Не прочитаны построчно целиком: ui/lore-graph-v2.js (128 КБ), ui/app.js (102 КБ), ui/chat.js (за пределами рендера и черновиков), prompts/*, core/background.js / coherence-audit.js / validators.js / pillars.js / session-memory.js. Они проверены только grep по всем опасным конструкциям и по списку API ST, которые используются (`getCharacterCardFields`, `characters`, `extensionSettings`, `getTokenCountAsync` и т.д.), и ничего подозрительного там нет. Ничего не запускалось. Не проверялся реальный ST: поведение `SillyTavern.libs.DOMPurify` и `executeSlashCommandsOnChatInput` принято по коду. Содержимое веток `testing` и `testing-two` проверено grep и `git show`, но не построчно.

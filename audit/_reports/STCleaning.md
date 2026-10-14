# Аудит: STCleaning (https://github.com/fillarr/STCleaning)

**Вердикт:** 🟢 Чисто

**Что делает расширение (по коду):** диалог «Cleaning» в меню волшебной палочки ST. Сканирует картинки чатов (`/api/images/folders`, `/api/images/list`) и отчёт Data Maid (`/api/data-maid/report`). Позволяет удалить выбранные файлы (`/api/images/delete`, `/api/data-maid/delete`) и скачать их локально перед удалением. Если файлов больше 5, он собирает zip в памяти браузера и отдаёт его через `<a download>` с blob-URL. Серверной части нет.

## Сетевые адреса
Внешних доменов нет. Все запросы идут на свой сервер ST по относительным путям через `fetch` в `api.js:15` с заголовками `getRequestHeaders()` (CSRF). `http(s)://` во всех JS, JSON и CSS отсутствует. В README есть только ссылки на GitHub.

| URL | Что отправляется | Файл:строка | Обосновано |
|---|---|---|---|
| POST `/api/images/folders` | `{}` | index.js:135 | да |
| POST `/api/images/list` | `{folder, sortField, sortOrder}` | index.js:143 | да |
| HEAD/GET `/user/images/<folder>/<file>` | ничего (Range-запросы только для размера) | api.js:112-141, index.js:809 | да |
| POST `/api/data-maid/report` | пусто | api.js:159 | да |
| GET `/api/data-maid/view?token=&hash=` | токен Data Maid и хэш, для скачивания файла | index.js:823 | да (скачивание перед удалением) |
| POST `/api/data-maid/delete` | `{token, hashes[]}` порциями по 20 | index.js:1119, 1131 | да |
| POST `/api/data-maid/finalize` | `{token}` | api.js:192 | да |
| POST `/api/images/delete` | `{path}` по одному файлу | index.js:1033 | да |

## Что именно может удалять (вопрос «не сверх ли заявленного»)
Удаление только по явному выбору пользователя, с подтверждением (чекбокс «I understand this deletion is permanent»), и только того, что видно в списке.
- **Картинки** (`deleteImagePaths`, index.js:988-1058). Список путей берётся из отмеченных чекбоксов. Затем он пересекается с `state.images` из последнего скана, а защищённые папки отсекаются повторно (index.js:1012-1018). Защищены `*_refs` и `generated`. Папка `generated` защищена сверх README, то есть в безопасную сторону. Удаляется по одному файлу через штатный `/api/images/delete`.
- **Data Maid** (`deleteDataMaidHashes`, index.js:1062-1150). Перед удалением берётся свежий отчёт. Хэши сопоставляются с элементами нужных категорий (chatBackups, settingsBackups, три типа thumbnails), остальные отбрасываются (index.js:1083-1089). В `/api/data-maid/delete` уходят только эти хэши. Затронуть чаты, персонажей, персон или настройки через это расширение нельзя: категорий «chats», «characters», «settings» в коде нет.
- «Новейший бэкап настроек» защищён только на клиенте. Это чекбокс с `disabled`. Это заявленное поведение, и защита не обходится самим расширением. Серверный эндпоинт Data Maid сам определяет, что удалять по хэшам.
- Групповая кнопка «Delete group» удаляет только отмеченные файлы группы (`gatherImageGroupPaths`, index.js:732).
- Ничего лишнего вне заявленного не обнаружено.

## Куда уходит архив
Никуда, кроме диска пользователя. `buildZipBlob` (zip.js) возвращает `Blob`, `triggerBlobDownload` (index.js:795-805) делает `URL.createObjectURL` и `a.click()` для имени `st-cleanup-<date>.zip`. Ни `fetch`, ни `sendBeacon`, ни `WebSocket`, ни `postMessage` с архивом нет. Для одиночных файлов (≤5) то же самое: blob-download.

## Самописный zip (zip.js, 138 строк)
Прочитан целиком, подвоха нет.
- Это обычный ZIP-writer метода STORE, без сжатия.
- Таблица CRC32 стандартная (поли 0xEDB88320), слайсы по 8 МБ с yield через `setTimeout(0)`. Строк в `setTimeout` нет.
- Записываются local header, central directory и EOCD. Имена в UTF-8 (флаг 0x0800).
- Дубликаты имён получают суффикс « (n)».
- Нет `eval`, нет сети, нет скрытых записей. Содержимое архива полностью определяется списком `entries`, который строит index.js из выбранных файлов.
- Ограничение, не вредоносное: нет ZIP64 (поля uint32). Архивы больше 4 ГБ или больше 65535 файлов могут быть повреждены. Поле числа записей (uint16) тоже переполнится. Это функциональный баг, не безопасность.
- Мелочь: `entry.path` чистится только от `\` и ведущих `/`, а `..` в пути не режется. Имена приходят от собственного сервера ST и дополнительно проходят `sanitizeArchivePart` (заменяет `\ / : * ? " < > |` и управляющие символы на `_`), так что zip-slip практически невозможен.

## Находки
Вредоносных или утекающих функций нет. Информационные замечания:
1. **INFO. Расхождение URL в README:** README:21 предлагает установку с `https://github.com/fillarr/ST-cleaning`, а репозиторий называется `STCleaning` (remote `.../fillarr/STCleaning.git`). Скорее всего это переименование. Пользователю стоит убедиться, что он ставит из `fillarr/STCleaning`. Если репозитория `ST-cleaning` не существует или он чужой, ссылка в README могла бы вести на подмену (это риск для пользователя, а не код репозитория; существование `ST-cleaning` я проверить не мог, нет сети/не проверялось).
2. **INFO. Токен Data Maid в query-строке** GET `/api/data-maid/view?token=...` (index.js:823). Это штатный способ Data Maid, токен отправляется только на свой сервер.
3. **INFO. `dom.js:22` `el.innerHTML = value`.** Ветка `html:` в `ce()`. Среди вызовов в index.js ключа `html` нет (grep `html\s*:` ничего не нашёл), так что мёртвый код. Все имена файлов выводятся через `text` (textContent).
4. **INFO. Динамический `import('../../../../script.js')`** в api.js:7 (top-level await). Это относительный путь на `public/script.js` самого ST, ради `getRequestHeaders`. Внешних URL нет.
5. **INFO. Поддержка TauriTavern** (`window.__TAURITAVERN__`, index.js, api.js:~153). Только отключает запрос Data Maid.
6. Коммиты автора `v0 <it+v0agent@vercel.com>` (11 шт.) и `vercel[bot]` — это ИИ-ассистент v0 от Vercel, PR мёрджит владелец `fillarr`. Я просмотрел итоговый код целиком, а не только диффы, и изменений с подвохом нет.

## Что проверено и не найдено
- Внешние домены: `fetch`, `XMLHttpRequest`, `sendBeacon`, `WebSocket`, `EventSource`, `Image().src`, `<script>/<iframe>`, `import()` с URL, `axios`, `$.ajax` — нет (кроме п.4).
- Секреты и настройки: `/api/secrets/*`, `oai_settings`, `api_key`, `password`, `localStorage`, `document.cookie` — нет.
- Чтение чатов, персонажей, `/api/settings/get` — нет. Читаются только список картинок и отчёт Data Maid.
- `eval`, `new Function`, `atob`, `String.fromCharCode`, строки в `setTimeout` — нет. Обфускации и бинарников нет.
- Вебхуки, телеметрия (IP, UA, дата), keydown-слушатели, отложенные триггеры по дате или счётчику — нет.
- Изменение настроек ST, установка расширений, запись в config — нет.
- Нет `package.json`, `plugins/` и серверного кода. Текст, обращённый к ИИ, не найден.
- git: 27 коммитов, `main` совпадает с `origin/v0/fillarr-2a475627` (одинаковый SHA 57c904f). Тегов нет. `git log --all -S` по `http`, `eval`, `atob`, `secrets`, `sendBeacon`, `WebSocket`, `XMLHttp`, `new Function`, `localStorage`, `cookie`, `innerHTML` — совпадения только в initial/phase-коммитах (`http` в README, `eval` как «evaluate/revalidated» в тексте README, `innerHTML` в dom.js). Удалённого вредоносного кода нет. Удалённых файлов нет.

**Просмотренные файлы:** полностью прочитаны index.js (1416 строк), api.js, zip.js, dom.js, constants.js, manifest.json, README.md. Только grep (внешние URL, `eval`, `atob`, `url(`, `@import`, текст для ИИ) — i18n/en.json, i18n/ru.json, style.css, LICENSE. Картинок и бинарников в репозитории нет (`file`: i18n json — JSON text, style.css — ASCII text).

## Уверенность
Высокая. Репозиторий небольшой, без минификации, весь исходный код прочитан. Не проверено: поведение серверных эндпоинтов Data Maid самого ST (вне репозитория) и существование/владелец репозитория `fillarr/ST-cleaning` из README.

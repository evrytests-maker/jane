# Аудит: MistraelSL-CharacterChatPills (https://github.com/MistraelSL/MistraelSL-CharacterChatPills)

**Вердикт:** 🟢 Чисто

**Что делает расширение (по коду):** на приветственном экране ST заменяет список недавних чатов библиотекой карточек персонажей. Список строится из `/api/chats/recent` и `/api/characters/chats` на своём сервере. Умеет открывать, переименовывать, удалять (с отменой) чаты и создавать новые. Добавляет кнопку «закрыть чат» на верхнюю панель. Закрепления и избранное хранит в `localStorage`.

## Сетевые адреса
Внешних запросов нет. Все `fetch` идут на тот же origin (ST) с заголовком `X-CSRF-Token`, взятым из `/csrf-token`.

| URL | Что отправляется | Файл:строка | Обосновано |
|---|---|---|---|
| GET `/csrf-token` | ничего | index.js:144 | да (штатный токен ST) |
| POST `/api/chats/recent` | `{max:1000, pinned:[]}` | index.js:158 | да |
| POST `/api/characters/chats` | `{avatar_url}` | index.js:357 | да |
| POST `/api/chats/rename` | `{avatar_url, original_file, renamed_file, is_group:false}` | index.js:414 | да |
| POST `/api/chats/get` | `{avatar_url, file_name}`: читает чат в память для «отмены удаления» | index.js:459 | да |
| POST `/api/chats/save` | `{avatar_url, file_name, chat, force:true}`: только при клике «Undo» на то же имя файла | index.js:492 | да |
| GET `/thumbnail?type=avatar&file=` (`<img src>`) | имя аватара | index.js:303 и др. | да |
| `deleteCharacterChatByName` (импорт из `script.js` ST) | штатное удаление чата | index.js:1, 476 | да |

Функция `requestJson(path, body)` (index.js:373) принимает путь аргументом, но во всех вызовах (строки 414, 459, 492) путь — литерал на `/api/chats/*`. Внешнего пути в неё передать нельзя.

## Находки
Вредоносных находок нет. Информационные замечания:
1. **INFO. Метаданные автора не совпадают:** `manifest.json` `author` = "Zaebalo Menyauje", README = "MistraelSL", git-автор = `Mistraeles <shain.jons@mail.ru>`, владелец GitHub = MistraelSL. Само по себе не вредоносно (стилизованные имена), но заметно.
2. **INFO. `innerHTML`** (index.js:322, 616, 682, 703, 764, 807, 830, 864, 986, 1123 и др.). Везде подставляются только статические иконки FontAwesome и локальные i18n-строки или число `entries.length`. Данные чатов и персонажей (имена, `mes`, `file_name`) выводятся через `textContent`. XSS из данных чата нет.
3. **INFO. Удаление чата** (index.js:452-505). Читает полный чат через `/api/chats/get` и держит в памяти (замыкание тоста) 12 секунд для «Undo». Это локально, наружу не уходит. Удаление идёт штатной функцией ST. Это заявленная функция и действие по явному клику с подтверждением.
4. **INFO. `setInterval(updateLanguage, 1000)`** (index.js:1164). Опрос языка раз в секунду, без сети.
5. Массовое чтение: `/api/chats/recent` с `max:1000` — это данные «последних чатов» (в том числе `mes`, последнее сообщение) для рендера. Результат живёт в переменной модуля `recentChatsCache` 60 с и никуда не отправляется. `localStorage` читается и пишется только по своим ключам `MistraelSL_CharacterChatPills_*` и `CharacterChatPills_density`.

## Что проверено и не найдено
- Внешние домены, `XMLHttpRequest`, `sendBeacon`, `WebSocket`, `EventSource`, `import()` с URL, `axios`, `$.ajax`: нет. Единственный `import` — `deleteCharacterChatByName` из `../../../../script.js` (ST).
- Секреты: `/api/secrets/*`, `oai_settings`, `api_key`, `password`, `document.cookie`: нет. Слово `token` встречается только как CSRF-токен.
- `eval`, `new Function`, `atob`, `String.fromCharCode`, строки в `setTimeout`/`setInterval` (используются функции): нет. Обфускации и минификации нет, index.js читаемый (1185 строк).
- `keydown`: только Enter/Space на кнопке-пине и Escape для закрытия своей модалки. Это не кейлоггер, данные нигде не собираются.
- Телеметрия, отпечаток, вебхуки, отложенные триггеры по дате или счётчику: нет. `Date.now()` используется только для TTL кэша и фильтра «активные за 7 дней».
- Изменение настроек ST, правка чужих расширений, установка расширений: нет. Нет серверной части, `package.json` и `plugins/`.
- Текст, обращённый к ИИ, в коде или README: нет.
- README заявляет «не переносит данные в сторонние сервисы». Код это подтверждает.
- git: 12 коммитов, единственный автор `Mistraeles`, одна ветка `main`, тегов нет. Из `git log --all -S` по `fetch(`, `import(`, `http`, `eval`, `atob`, `secrets`, `sendBeacon`, `WebSocket`, `XMLHttp`, `new Function`, `cookie` встречается только `fetch(` в Initial commit (те же 4 вызова на свой сервер) и `http` в README (ссылка на установку). Удалённого вредоносного кода нет.
- style.css: нет `url()`, `@import`, внешних шрифтов и ресурсов (grep пустой).

**Просмотренные файлы:** полностью прочитаны index.js (1185 строк), manifest.json, README.md, .gitattributes. style.css (962 строки) и LICENSE — только grep на внешние ресурсы (`url(`, `@import`, `http`). Бинарников и картинок в репозитории нет (`file`: JavaScript source, ASCII text).

## Уверенность
Высокая. Весь код читаемый и прочитан целиком. Не проверено: поведение самих эндпоинтов ST и `deleteCharacterChatByName` (код ST вне репозитория).

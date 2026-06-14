# BitrixPlugin — «Свои ссылки на мессенджеры»

Маркетплейс-приложение для Битрикс24: регистрирует кастомный тип коннектора Открытых линий,
который превращает любую ссылку (Telegram, WhatsApp, любой URL) в кнопку виджета на сайте —
без изменения кода клиентского сайта.

## Тип приложения и его следствия

**"Простое"/"Статичное"** — один статический `index.html`, загружается ZIP-архивом
(`app.zip`) на CDN Битрикс24. `BX24.js` сам авторизует внутри iframe.

**Критично:** у статичных приложений нет install-хука — код выполняется только когда
человек ОТКРЫВАЕТ приложение (`BX24.init`). Поэтому всё, что должно появиться сразу после
установки (плитка в Контакт-центре, регистрация коннекторов), на самом деле создаётся лениво
при каждом открытии — а пользователю нужна причина открыть приложение после установки
(см. экран «Активация», по аналогии с Wazzup, который для этого использует обязательный логин).

## Структура

```
index.html   — приложение (BX24): 4 экрана в одном файле
  #s-load       загрузочный экран
  #s-settings   настройки/чек-лист мессенджеров (прямое открытие приложения)
  #s-dialog     диалог из плитки Контакт-центра — выбор мессенджера + линии
  #s-pl         PLACEMENT_HANDLER — настройка коннектора внутри конкретной Открытой линии
mail.html    — standalone-страница с адресом почты (открывается по url_im из виджета
               на сайте, без BX24; ссылка вида HANDLER_URL.replace(.., 'mail.html')+'?to=')
app.zip      — пересобирается после каждой правки index.html/mail.html (см. ниже)
SETUP.md     — пошаговая инструкция по верификации/деплою/публикации в маркетплейс
```

Маршрутизация между экранами — через `BX24.placement.info()`:
- `info.options.CONNECTOR/.LINE` → открыто из настроек коннектора в Открытой линии → `#s-pl`
- `info.placement === 'CONTACT_CENTER'` → клик по плитке → `#s-dialog`
- иначе → прямое открытие приложения → `#s-settings`

## Пересборка после правок

```powershell
Compress-Archive -Path 'd:\Progs\BitrixPlugin\index.html','d:\Progs\BitrixPlugin\mail.html' -DestinationPath 'd:\Progs\BitrixPlugin\app.zip' -Force
```
Затем загрузить новый `app.zip` в Б24 (см. `SETUP.md`).

## Ключевые технические факты (важные грабли)

Подробная база знаний с источниками документации, датами проверки и разбором ошибок —
в личном скиле **`bitrix24-apps`** (`~/.claude/skills/bitrix24-apps/SKILL.md`). Кратко:

- `imconnector.*` методы требуют **контекста приложения/OAuth** — через голый вебхук
  падают с `WRONG_AUTH_TYPE`.
- `imconnector.activate({CONNECTOR, LINE, ACTIVE})` — параметр `ACTIVE` **обязателен**
  (`1`/`0`), без него: `Argument 'ACTIVE' is null or empty`.
- `imconnector.register` задаёт иконку коннектора **глобально для всего типа** — менять
  её на экране настройки конкретной линии технически означает менять её везде
  (на плитке Контакт-центра и на всех линиях). Это не баг приложения, а ограничение API.
- `imconnector.connector.data.set` → `DATA.url_im` — делает кнопку виджета прямой ссылкой
  (редирект), а не открытием чата. Подтверждено вживую через рендер виджета headless-браузером.
- `app.option.get/set` — хранилище per-app **per-portal** (подтверждено: между порталами
  не пересекается). Используется для `enabled_messengers` и `custom_icons`.
- `placement.bind({PLACEMENT:'CONTACT_CENTER', HANDLER})` — нужно вызывать на каждом
  `BX24.init` (идемпотентно), иначе плитка не появится после переустановки/обновления.

## Источники документации — где что искать

- **Интерфейс/UI** (где в Б24 что находится и открывается) → `helpdesk.bitrix24.ru`
  (актуально). `training.bitrix24.com` показывает **устаревший интерфейс** — годится
  только для понимания последовательности вызовов API, не для навигации по UI.
- **REST API** (сигнатуры методов, обязательные параметры) → `apidocs.bitrix24.com`
  (плюс GitHub `bitrix24/b24restdocs`, иногда свежее рендера).
- **Публикация в маркетплейс** → кабинет партнёра `vendors.bitrix24.ru`.
- Любым материалам старше ~года — не доверять напрямую, перепроверять вживую.

## Проверка вживую (как реально протестировать виджет)

Статический фетч (`curl`/`WebFetch`) не видит динамически отрисованный JS-виджет.
Для реальной проверки — headless-браузер (Playwright уже установлен в Python, см. скил
**`browse-web`**): открыть тестовый сайт с виджетом, кликнуть по кнопке, через
`page.evaluate()` вытащить реальные `href`/классы кнопок-коннекторов
(`ui-icon-service-clnk_*`) — так подтверждается, что `url_im` рендерится как `<a href>`,
а не как открытие чата.

## Текущее состояние

Приложение работает и протестировано на портале пользователя + сайте
`vershalovskiy.com` (Telegram + 2 кастомные ссылки — все кнопки рендерятся как прямые
ссылки с правильными `href`). Архив `app.zip` актуален относительно `index.html`.

## Поведенческие рекомендации

Поведенческие правила, которые снижают вероятность типичных ошибок при кодировании.
Для тривиальных задач используй здравый смысл.

### 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

### 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

### 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

### 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.

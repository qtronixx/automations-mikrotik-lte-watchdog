# MikroTik LTE Watchdog для Home Assistant

Автоматический "сторож" для связки **MikroTik (LTE passthrough) + pfSense Gateway Group**: мониторит статус LTE-шлюза, шлёт предупреждения в Telegram и перезагружает роутер, если шлюз долго в `down`, с защитой от бесконечного цикла перезагрузок.

Собрано и обкатано в проде на: MikroTik ATL LTE18 (RouterOS 7.24) в режиме LTE Passthrough, pfSense 2.8.1, Home Assistant 2026.8.2, интеграции `mikrotik` (официальная) + `hass-pfsense` (HACS) + `telegram_bot` (UI-интеграция).

## Содержание

- [Что делает](#что-делает)
- [Требования](#требования)
- [Структура репозитория](#структура-репозитория)
- [Установка](#установка)
- [Настройка под себя](#настройка-под-себя)
- [Регулируемые пороги](#регулируемые-пороги)
- [Anti-flapping: защита от бесконечных перезагрузок](#anti-flapping-защита-от-бесконечных-перезагрузок)
- [Управление вручную из Telegram](#управление-вручную-из-telegram)
- [Тестирование перед проду](#тестирование-перед-продом)
- [Логирование и отладка](#логирование-и-отладка)
- [entity_id автоматизаций: id: ≠ entity_id](#entity_id-автоматизаций-id--entity_id)
- [Несколько роутеров](#несколько-роутеров)
- [Troubleshooting](#troubleshooting)
- [Известные ограничения](#известные-ограничения)
- [Дисклеймер](#дисклеймер)

## Что делает

1. **Мониторинг** — раз в N минут проверяет статус и packet loss обоих шлюзов pfSense Gateway Group (`Tier 1` через MikroTik LTE, `Tier 2` резервный). Если статус `down` или packet loss выше порога — шлёт предупреждение в Telegram, но не чаще заданного интервала, пока проблема не исчезнет.
2. **Авто-перезагрузка** — если основной LTE-шлюз находится в `down` дольше заданного времени, нажимает штатную кнопку перезагрузки MikroTik (`button.press`), ждёт заданное время и проверяет восстановление. Если не восстановился — критическое уведомление.
3. **Circuit breaker** — при достижении суточного лимита перезагрузок дальнейшие reboot'ы прекращаются, отправляется одно уведомление о вероятной аварии на стороне оператора (а не зависании роутера).
4. **Ручное управление** — команды `/mikrotik_silence` и `/mikrotik_reset` прямо в Telegram-чате.

## Требования

- Home Assistant 2024.8+ (используется актуальный ключ `action:`, а не устаревший `service:`)
- Интеграция [`mikrotik`](https://www.home-assistant.io/integrations/mikrotik/) — офиц., должна давать entity вида `button.<имя>_restart` для перезагрузки устройства
- [`hass-pfsense`](https://github.com/travisghansen/hass-pfsense) (HACS) — сенсоры статуса/loss/delay шлюзов pfSense
- `telegram_bot` — настроенная через UI интеграция (Настройки → Устройства и службы → Telegram), с ботом, добавленным в нужный чат/группу
- pfSense Gateway Group с минимум одним основным (Tier 1) и одним резервным (Tier 2) шлюзом

## Структура репозитория

```
.
├── packages/
│   └── mikrotik_atlgm_lte_watchdog.yaml   # основной модуль: helpers + logger + автоматизации
├── helpers.yaml                            # то же, что helpers-часть пакета — отдельно, для ручной сборки
├── automations_mikrotik_lte_watchdog.yaml  # то же, что automation-часть пакета — отдельно
├── configuration_snippets.yaml             # logger + опциональный fallback rest_command
└── README.md
```

Рекомендуемый способ установки — **`packages/`**, остальные файлы — тот же самый код, разложенный по частям, на случай, если предпочитаете собирать конфиг руками.

## Установка

### 1. Включите `packages` в Home Assistant

`packages` — это вложенный ключ внутри `homeassistant:`, а не отдельная интеграция. Добавьте в `configuration.yaml`:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Если у вас уже есть блок `homeassistant:` (с `name:`, `allowlist_external_urls:` и т.д.) — допишите строку `packages:` внутрь него, с тем же отступом, что у соседних параметров. **Имя папки в `!include_dir_named` должно дословно совпадать** с именем реальной папки на диске (`packages`, с `s`) — расхождение в одну букву даёт ошибку `Integration 'packages' not found` или аналогичную.

> Если в IDE/редакторе конфига (например, VS Code с YAML-плагином для HA) эта строка подсвечивается как `patternWarning` — это ограничение статической схемы редактора, не ошибка самого HA. Проверяйте реальную валидность через Настройки → Система → Проверить конфигурацию внутри Home Assistant, а не через подсветку в IDE.

### 2. Скопируйте пакет

Положите `packages/mikrotik_atlgm_lte_watchdog.yaml` в `/config/packages/` как есть, без изменения структуры (кроме entity_id под свою систему, см. ниже).

### 3. Проверьте кнопку перезагрузки MikroTik

Перезагрузка идёт через штатную entity интеграции `mikrotik`, а не через REST API — никакой дополнительной настройки на стороне роутера не требуется.

Перед первым запуском **проверьте вручную**: Developer Tools → Actions → `button.press` → выберите вашу entity перезагрузки (в этом репозитории — `button.qtronixlte_restart`) → убедитесь, что роутер реально перезагружается.

### 4. Добавьте секрет для Telegram

В `secrets.yaml`:

```yaml
# Mikrotik LTE Watchdog
telegram_chat_id: -1234567890
```

Нужен, если у вас несколько ботов/чатов и уведомления должны идти в конкретный. Все вызовы `telegram_bot.send_message` в пакете используют `chat_id: !secret telegram_chat_id`.

### 5. Перезапустите Home Assistant

Обязательно полный перезапуск (не "Reload automations") — packages, helpers и logger подхватываются только при старте.

### 6. Проверьте, что всё подключилось

- Настройки → Устройства и службы → Помощники → фильтр `mikrotik_atlgm` — должно быть 2 `input_boolean`, 2 `input_datetime`, 7 `input_number`.
- Developer Tools → States → фильтр `lte_watchdog` — должно быть 6 `automation.*` entity (реальные имена см. в разделе [entity_id автоматизаций](#entity_id-автоматизаций-id--entity_id)).
- Бот подписан на команды `/mikrotik_silence` и `/mikrotik_reset` (событие `telegram_command`) — проверить через Developer Tools → События → прослушать `telegram_command`.

Дальше обязательно пройдите раздел [Тестирование перед продом](#тестирование-перед-продом) — там пошагово проверяются все компоненты, прежде чем доверять системе реальную перезагрузку роутера.

## Настройка под себя

Перед использованием замените под свою систему:

| Что | Где | Значение по умолчанию в репозитории |
|---|---|---|
| Entity статуса LTE-шлюза | `packages/mikrotik_atlgm_lte_watchdog.yaml` | `sensor.fw_village_qtronix_ru_gateway_wan_lte_mikrotik_dhcp_status` |
| Entity статуса резервного шлюза | там же | `sensor.fw_village_qtronix_ru_gateway_wan_dhcp_status` |
| Entity packet loss / latency (оба шлюза) | там же | `..._loss` / `..._delay` для каждого статуса выше |
| Entity кнопки перезагрузки MikroTik | там же, автоматизация `auto_reboot` | `button.qtronixlte_restart` |
| Chat ID Telegram | `secrets.yaml` | `telegram_chat_id` |

Точные entity_id сенсоров `loss`/`delay` зависят от версии `hass-pfsense` — сверьте через Developer Tools → States, отфильтровав по имени вашего шлюза, прежде чем полагаться на значения из репозитория.

## Регулируемые пороги

Все пороги, кроме двух интервалов опроса, вынесены в `input_number` — меняются в UI (Настройки → Помощники), без правки YAML и без перезапуска HA.

| Параметр | Entity | По умолчанию | Комментарий |
|---|---|---|---|
| X — порог packet loss | `input_number.mikrotik_atlgm_lte_watchdog_packet_loss_threshold` | 20% | Держите выше порога, на котором сама pfSense Gateway Group переключает шлюзы — иначе будете дублировать её работу предупреждениями |
| Y — минуты `down` до reboot | `input_number.mikrotik_atlgm_lte_watchdog_down_minutes_before_reboot` | 10 мин | Меньше — риск перезагружать роутер во время штатной кратковременной деградации LTE |
| Z — пауза после reboot | `input_number.mikrotik_atlgm_lte_watchdog_post_reboot_wait_minutes` | 5 мин | Учитывайте время загрузки роутера + LTE-регистрацию в сети + DHCP на pfSense |
| Cooldown между перезагрузками | `input_number.mikrotik_atlgm_lte_watchdog_reboot_cooldown_minutes` | 30 мин | Основной барьер против reboot-loop |
| Лимит перезагрузок в сутки | `input_number.mikrotik_atlgm_lte_watchdog_max_reboots_per_day` | 3 | При достижении — circuit breaker |
| Повтор warning | `input_number.mikrotik_atlgm_lte_watchdog_warning_repeat_minutes` | 30 мин | Чтобы не спамить чат при затяжной деградации |
| Интервал проверки warning | `time_pattern` в автоматизации `warning_monitor` | 5 мин | Правится только в YAML — HA не поддерживает шаблоны в `trigger.time_pattern` |
| Интервал проверки reboot | `time_pattern` в автоматизации `auto_reboot` | 1 мин | Точность ±1 мин к порогу Y |

## Anti-flapping: защита от бесконечных перезагрузок

1. **Cooldown** — не перезагружать чаще, чем раз в N минут, даже если шлюз снова упал.
2. **Суточный лимит** — после N-й перезагрузки за сутки авто-reboot просто не сработает; отдельная автоматизация `circuit_breaker` один раз пришлёт сообщение о вероятной аварии/блокировке у оператора, а не зависании роутера.
3. **Ручной сброс** — `/mikrotik_reset` в Telegram снимает silence и сбрасывает счётчик до полуночи, если проблему решили руками (оплатили SIM, поменяли антенну и т.д.).
4. **Silence** — `/mikrotik_silence` отключает уведомления и авто-reboot на 1 час, удобно на время планового обслуживания.

## Управление вручную из Telegram

| Команда | Действие |
|---|---|
| `/mikrotik_silence` | Отключает уведомления и авто-reboot на 1 час |
| `/mikrotik_reset` | Снимает silence, сбрасывает суточный счётчик перезагрузок и флаг circuit breaker |

Обработчик команд — часть самого пакета (автоматизации `telegram_silence` / `telegram_reset`, триггер на событие `telegram_command`), отдельно регистрировать команды через BotFather не требуется — это влияет только на автодополнение в интерфейсе Telegram, не на работу автоматизации.

## Тестирование перед продом

Рекомендуемый порядок — от простого к сложному, не трогая реальную перезагрузку роутера на первых шагах.

**1. Бот вообще может слать сообщения:**
```yaml
# Developer Tools → Actions
action: telegram_bot.send_message
data:
  chat_id: -1234567890   # ваш реальный chat_id
  message: "Тест от LTE Watchdog"
```

**2. Команды из Telegram долетают до HA:**
Developer Tools → События → тип события `telegram_command` → «Начать прослушивание» → отправьте боту `/mikrotik_silence` → должно появиться событие с `command: /mikrotik_silence`.

**3. Команды силы/сброса «вживую»:**
Отправьте `/mikrotik_silence` → должно прийти подтверждение, `input_boolean.mikrotik_atlgm_lte_watchdog_alert_silenced` → `on`. Затем `/mikrotik_reset` → булево должно вернуться в `off`, `input_number.mikrotik_atlgm_lte_reboot_count_today` → `0`.

**4. Автоматизация мониторинга без реальной аварии:**
```yaml
action: automation.trigger
target:
  entity_id: automation.lte_watchdog_monitoring_shliuzov_warning   # см. таблицу ниже
data:
  skip_condition: true
```
`skip_condition: true` пропускает только верхнеуровневое `condition:` автоматизации — если внутри последовательности действий есть отдельный шаг `condition:` (например, проверка на `alert_silenced`), он всё равно выполнится и может остановить сценарий. Это ожидаемое поведение, не баг.

**5. Реальная перезагрузка — осторожно:**
Тот же приём с `automation.trigger` + `skip_condition: true` для автоматизации `auto_reboot` **реально нажмёт кнопку перезагрузки роутера**. Делайте это осознанно, в плановое окно. Если нужно проверить только логику порогов без физической перезагрузки — временно замените в YAML действие `button.press` на безобидный `telegram_bot.send_message`, прогоните тест, верните как было.

## Логирование и отладка

Ключевые события пишутся через `system_log.write` с логгером `custom.mikrotik_atlgm_lte_watchdog`:

```bash
grep "mikrotik_atlgm_lte_watchdog" /config/home-assistant.log
```

- `STATUS: LTE(...)` (`info`) — снимок состояния шлюзов при каждой проверке `warning_monitor`.
- `REBOOT TRIGGERED: ...` (`warning`) — реальный запуск перезагрузки, значимое событие.
- `Post-reboot check: ...` (`info`) — результат проверки восстановления.

Уровень `debug` для самих автоматизаций включается в блоке `logger:` пакета — но обратите внимание, **ключи там должны быть реальными путями логгеров** (`homeassistant.components.automation.<slug>`), а не entity_id и не `id:` из YAML — см. следующий раздел, откуда берётся этот slug.

## entity_id автоматизаций: `id:` ≠ `entity_id`

Home Assistant генерирует `entity_id` автоматизации не из поля `id:` в YAML (это чисто внутренний идентификатор для редактирования/хранения), а транслитерацией `alias:`. Из-за этого `id: mikrotik_atlgm_lte_watchdog_warning_monitor` превращается в `automation.lte_watchdog_monitoring_shliuzov_warning`, а не в то, что можно было бы ожидать по `id:`.

Актуальная карта (проверьте у себя через Developer Tools → States → фильтр `lte_watchdog`, транслитерация зависит от точного текста `alias:`):

| `id:` в YAML | Реальный `entity_id` |
|---|---|
| `mikrotik_atlgm_lte_watchdog_warning_monitor` | `automation.lte_watchdog_monitoring_shliuzov_warning` |
| `mikrotik_atlgm_lte_watchdog_auto_reboot` | `automation.lte_watchdog_avto_perezagruzka_mikrotik` |
| `mikrotik_atlgm_lte_watchdog_circuit_breaker` | `automation.lte_watchdog_limit_perezagruzok_ischerpan` |
| `mikrotik_atlgm_lte_watchdog_daily_reset` | `automation.lte_watchdog_sbros_schetchika_perezagruzok_v_00_00` |
| `mikrotik_atlgm_lte_watchdog_telegram_silence` | `automation.lte_watchdog_telegram_mikrotik_silence` |
| `mikrotik_atlgm_lte_watchdog_telegram_reset` | `automation.lte_watchdog_telegram_mikrotik_reset` |

Используйте `entity_id` из правой колонки в любых сервисных вызовах (`automation.trigger`, `automation.turn_off` и т.п.), не `id:`.

## Несколько роутеров

Все helpers и id автоматизаций используют префикс `mikrotik_atlgm_lte_`, где `atlgm` — идентификатор конкретного устройства. Чтобы добавить watchdog для второго MikroTik:

1. Скопируйте `packages/mikrotik_atlgm_lte_watchdog.yaml` под новым именем, например `packages/mikrotik_office_lte_watchdog.yaml`.
2. Замените префикс `atlgm` на новый везде внутри файла: entity_id всех helpers, `id:` всех автоматизаций, entity_id устройства (сенсоры статуса/loss/delay, кнопка перезагрузки), команды Telegram (если хотите разные для разных роутеров).
3. Обновите блок `logger:` — имена логгеров там завязаны на транслитерацию `alias:` (см. раздел выше), для нового набора алиасов получите новые slug'и через Developer Tools → States после первого запуска, и уже тогда пропишите их в `logger.logs`.

Watchdog'и будут работать полностью независимо, без пересечения сущностей.

## Troubleshooting

**`Integration 'packages' not found`**
Строка `packages: !include_dir_named packages` стоит на корневом уровне `configuration.yaml`, а не внутри блока `homeassistant:`. Проверьте отступы — `packages:` должен быть дочерним ключом `homeassistant:`.

**`Map keys must be unique` при добавлении конфига**
Вы вставили `helpers.yaml`/`automations_...yaml` напрямую в `configuration.yaml`, где уже есть такой же корневой ключ. Используйте `packages/` (см. [Установка](#установка)) — это устраняет проблему полностью.

**`patternWarning` в VS Code/IDE на строке `packages: !include_dir_named packages`**
Косметика статической YAML-схемы редактора, не ошибка HA. Ориентируйтесь на Настройки → Система → Проверить конфигурацию внутри самого Home Assistant.

**Ошибка после включения packages: имя папки не совпадает**
`!include_dir_named packages` ищет папку `packages` (с `s`) относительно `/config/`. Если у вас `/config/package/` (без `s`) — либо переименуйте папку, либо поправьте аргумент тега, чтобы совпадали дословно.

**Уведомления не приходят, при этом ошибок нет**
Проверьте, что `telegram_chat_id` реально добавлен в `secrets.yaml` и совпадает с чатом, где сидит бот. Отправьте тестовое сообщение напрямую (см. [Тестирование](#тестирование-перед-продом), шаг 1).

**`Action failed. Can't parse entities: character '.' is reserved...` или `can't find end of the entity`**
Это ошибка парсера Telegram Markdown — она спотыкается на одиночных `_`, `.`, `-`, `(`, `)` в тексте (например, в `/mikrotik_silence` одно подчёркивание парсер Markdown воспринимает как начало курсива без пары). В этом репозитории уже решено — все сообщения используют `parse_mode: html` вместо `markdown`, с тегами `<b>`/`<code>` вместо `*`/`` ` ``. Если добавляете новые сообщения с текстом, содержащим `_`, `.`, `-` — держите `parse_mode: html` и избегайте символов `<`, `>`, `&` без экранирования (`&lt;`, `&gt;`, `&amp;`).

**`The target parameter for Telegram Bot is being removed` (Repairs/предупреждение)**
Начиная с HA 2026.9 параметр `target` в `telegram_bot.send_message` убирается. В этом репозитории уже используется `chat_id:` вместо `target:` во всех вызовах.

**`Error rendering variables: TypeError: can't subtract offset-naive and offset-aware datetimes`**
`input_datetime` хранит значение без таймзоны (naive), а `now()` в шаблонах HA — с таймзоной (aware); прямое вычитание `now() - as_datetime(...)` падает. В репозитории все такие места переписаны через `as_timestamp(now()) - as_timestamp(...)`, что не чувствительно к naive/aware.

**`Referenced entities ... are missing or not currently available` при ручном вызове `automation.trigger`**
Вы использовали `id:` из YAML вместо реального `entity_id`. См. раздел [entity_id автоматизаций](#entity_id-автоматизаций-id--entity_id).

**Авто-reboot не срабатывает, хотя шлюз в `down`**
- Проверьте `input_boolean.mikrotik_atlgm_lte_watchdog_alert_silenced` — не включён ли silence.
- Проверьте cooldown: `input_datetime.mikrotik_atlgm_lte_last_reboot` — если перезагрузка была недавно, сработает cooldown.
- Проверьте суточный лимит: `input_number.mikrotik_atlgm_lte_reboot_count_today` vs `input_number.mikrotik_atlgm_lte_watchdog_max_reboots_per_day`.

**Кнопка `button.press` не перезагружает роутер**
Значит, entity в вашей системе называется иначе или интеграция `mikrotik` не даёт кнопку restart для этой модели — проверьте Developer Tools → States по фильтру `button.` и имени вашего роутера.

## Известные ограничения

- Интервалы опроса (`time_pattern`) заданы напрямую в YAML — Home Assistant не поддерживает шаблоны/helpers внутри `trigger.time_pattern`, поэтому эти два значения нельзя вынести в UI.
- Пакет рассчитан на одну Gateway Group с двумя шлюзами (Tier 1 / Tier 2); для более сложных топологий потребуется доработка условий.
- Перезагрузка через `button.press` зависит от доступности интеграции `mikrotik` на момент срабатывания автоматизации — если сама интеграция не может достучаться до роутера по LAN, кнопка тоже не сработает.
- `entity_id` автоматизаций зависит от транслитерации `alias:` и может отличаться в зависимости от версии HA/локали — при обновлении HA стоит перепроверить карту из раздела выше.

## Дисклеймер

Автоматизация физически перезагружает сетевое оборудование без участия человека. Перед использованием в проде:
- протестируйте по чек-листу из раздела [Тестирование перед продом](#тестирование-перед-продом);
- убедитесь, что пороги Y/Z/cooldown/лимит подобраны под характер именно вашего LTE-канала — слишком агрессивные настройки могут вызывать лишние перезагрузки во время штатных кратковременных просадок сети.

Используется на свой риск.

# MikroTik LTE Watchdog для Home Assistant

Автоматический "сторож" для связки **MikroTik (LTE passthrough) + pfSense Gateway Group**: мониторит статус LTE-шлюза, шлёт предупреждения в Telegram и перезагружает роутер, если шлюз долго в `down`, с защитой от бесконечного цикла перезагрузок.

Собрано и обкатано на: MikroTik ATL LTE18 (RouterOS 7.24) в режиме LTE Passthrough, pfSense 2.8.1, Home Assistant 2026.8.2, интеграции `mikrotik` (официальная) + `hass-pfsense` (HACS) + `telegram_bot`.

## Содержание

- [Что делает](#что-делает)
- [Требования](#требования)
- [Структура репозитория](#структура-репозитория)
- [Установка](#установка)
- [Настройка под себя](#настройка-под-себя)
- [Регулируемые пороги](#регулируемые-пороги)
- [Anti-flapping: защита от бесконечных перезагрузок](#anti-flapping-защита-от-бесконечных-перезагрузок)
- [Управление вручную из Telegram](#управление-вручную-из-telegram)
- [Логирование и отладка](#логирование-и-отладка)
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

- Home Assistant 2024.8+ (используется актуальный ключ `action:`)
- Интеграция [`mikrotik`](https://www.home-assistant.io/integrations/mikrotik/) — офиц., должна давать entity вида `button.<имя>_restart` для перезагрузки устройства
- [`hass-pfsense`](https://github.com/travisghansen/hass-pfsense) (HACS) — сенсоры статуса/loss/delay шлюзов pfSense
- `telegram_bot` — настроенный бот с доступом к нужному(ым) чату(ам)
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

Если ещё не используете packages, добавьте в `configuration.yaml`:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Если у вас уже есть блок `homeassistant:` — просто допишите в него строку `packages: !include_dir_named packages`, не создавайте второй корневой ключ.

> **Почему packages, а не просто вставить YAML в configuration.yaml?**
> Каждый корневой ключ (`input_number:`, `input_boolean:`, `automation:` и т.д.) в HA может встречаться только один раз. Если у вас уже есть свой `input_number:` где-то в конфиге, второй такой же блок ниже даст ошибку `Map keys must be unique`. Packages — штатный механизм HA для подключения независимых модулей конфигурации без этого конфликта.

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

Нужен, если у вас несколько ботов/чатов и уведомления должны идти в конкретный — все вызовы `telegram_bot.send_message` в пакете используют `target: !secret telegram_chat_id`.

### 5. Перезапустите Home Assistant

Обязательно полный перезапуск (не "Reload automations") — packages, helpers и logger подхватываются только при старте.

### 6. Проверьте, что всё подключилось

- Настройки → Устройства и службы → Помощники → фильтр `mikrotik_atlgm` — должно быть 2 `input_boolean`, 2 `input_datetime`, 7 `input_number`.
- Настройки → Автоматизации → фильтр `LTE Watchdog` — должно быть 6 автоматизаций.
- Бот подписан на команды `/mikrotik_silence` и `/mikrotik_reset` (событие `telegram_command`).

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

## Логирование и отладка

Ключевые события пишутся через `system_log.write` с логгером `custom.mikrotik_atlgm_lte_watchdog`:

```bash
grep "mikrotik_atlgm_lte_watchdog" /config/home-assistant.log
```

Уровень логов для автоматизаций поднят до `debug` в блоке `logger:` пакета — там же видны срабатывания триггеров и условий.

## Несколько роутеров

Все helpers и id автоматизаций используют префикс `mikrotik_atlgm_lte_`, где `atlgm` — идентификатор конкретного устройства. Чтобы добавить watchdog для второго MikroTik:

1. Скопируйте `packages/mikrotik_atlgm_lte_watchdog.yaml` под новым именем, например `packages/mikrotik_office_lte_watchdog.yaml`.
2. Замените префикс `atlgm` на новый везде внутри файла: entity_id всех helpers, `id:` всех автоматизаций, имена в `logger:`, entity_id устройства (сенсоры статуса/loss/delay, кнопка перезагрузки), команды Telegram (если хотите разные для разных роутеров).

Watchdog'и будут работать полностью независимо, без пересечения сущностей.

## Troubleshooting

**`Map keys must be unique` при добавлении конфига**
Вы вставили `helpers.yaml`/`automations_...yaml` напрямую в `configuration.yaml`, где уже есть такой же корневой ключ. Используйте `packages/` (см. [Установка](#установка)) — это устраняет проблему полностью.

**Уведомления не приходят**
- Проверьте, что `telegram_chat_id` реально добавлен в `secrets.yaml` и совпадает с чатом, где сидит бот.
- Проверьте логи: `grep "mikrotik_atlgm_lte_watchdog" home-assistant.log`.

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

## Дисклеймер

Автоматизация физически перезагружает сетевое оборудование без участия человека. Перед использованием в проде:
- протестируйте на реалистичных, но контролируемых сценариях (например, временно завысив порог, чтобы спровоцировать reboot вручную);
- убедитесь, что пороги Y/Z/cooldown/лимит подобраны под характер именно вашего LTE-канала — слишком агрессивные настройки могут вызывать лишние перезагрузки во время штатных кратковременных просадок сети.

Используется на свой риск.

# MikroTik LTE Watchdog для Home Assistant

Автоматический "сторож" для связки **MikroTik (LTE passthrough) + pfSense Gateway Group**: мониторит статус LTE-шлюза, шлёт предупреждения в Telegram, эскалирует перезагрузки (софт → хард через обрыв питания, с растущей длительностью обрыва) и показывает текущее состояние на RGB-светодиоде. Отдельно детектирует потерю связи HA с самим оборудованием (не с LTE-каналом). Есть защита от бесконечного цикла перезагрузок и авто-восстановление после длительных проблем провайдера.

Собрано и обкатано в проде на: MikroTik ATL LTE18 (RouterOS 7.24) в режиме LTE Passthrough, pfSense 2.8.1, Home Assistant 2026.8.2, интеграции `mikrotik` (официальная) + `hass-pfsense` (HACS) + `telegram_bot` (UI-интеграция) + `esphome` (единая плата ESP32-S3).

## Содержание

- [Что делает](#что-делает)
- [Почему логика в HA, а не на ESP](#почему-логика-в-ha-а-не-на-esp)
- [Схема эскалации](#схема-эскалации)
- [Эскалация длительности хард-ресета](#эскалация-длительности-хард-ресета)
- [LED-индикация](#led-индикация)
- [Требования](#требования)
- [Аппаратная часть: ESP32-S3 (реле + LED)](#аппаратная-часть-esp32-s3-реле--led)
- [Структура репозитория](#структура-репозитория)
- [Установка](#установка)
- [Настройка под себя](#настройка-под-себя)
- [Регулируемые пороги](#регулируемые-пороги)
- [Anti-flapping и авто-восстановление](#anti-flapping-и-авто-восстановление)
- [Потеря связи HA с оборудованием — отдельный класс проблем](#потеря-связи-ha-с-оборудованием--отдельный-класс-проблем)
- [Управление вручную из Telegram](#управление-вручную-из-telegram)
- [Тестирование перед продом](#тестирование-перед-продом)
- [Логирование и отладка](#логирование-и-отладка)
- [entity_id автоматизаций: id: ≠ entity_id](#entity_id-автоматизаций-id--entity_id)
- [Несколько роутеров](#несколько-роутеров)
- [Troubleshooting](#troubleshooting)
- [Известные ограничения](#известные-ограничения)
- [Дисклеймер](#дисклеймер)

## Что делает

1. **Мониторинг** — статус и packet loss обоих шлюзов pfSense Gateway Group. Проблема — предупреждение в Telegram + жёлтый LED.
2. **Эскалация перезагрузок** — софт-ребут, затем аппаратный обрыв питания через реле, с **растущей длительностью обрыва** при повторных неудачах (см. [ниже](#эскалация-длительности-хард-ресета)).
3. **Circuit breaker** — при исчерпании суточного лимита LED переходит в "полицейский" режим, приходит критическое уведомление.
4. **Авто-восстановление** — периодически проверяет, не заработал ли канал сам по себе — если да, снимает лимит/мьют без ручного вмешательства.
5. **Детектор потери связи с оборудованием** — отдельно от LTE-статуса отслеживает доступность самих кнопок управления.
6. **Визуальная индикация** — RGB LED показывает текущее состояние.
7. **Ручное управление**, включая variable-длительность хард-ресета: `/mikrotik_hard_reset 10` = обрыв на 10 минут.

## Почему логика в HA, а не на ESP

- ESP не видит статус pfSense-шлюза напрямую.
- Все пороги, anti-flapping, Telegram-команды и логи уже отлажены в HA.
- ESP — тупой исполнительный механизм (реле вкл/выкл, power-cycle с настраиваемой длительностью, LED-эффекты по имени). **HA принимает все решения**, ESP их выполняет и отображает.

## Схема эскалации

```
Шлюз в down ≥ 10 мин                                    LED: degraded (жёлтый, сплошной)
        │
        ▼
  Попытка №1: софт-ребут                                 LED: soft_reset (красный, сплошной)
        │  ждём 5 мин, не поднялся; cooldown 10 мин
        ▼
  Попытка №2: софт-ребут                                 LED: soft_reset
        │  ждём 5 мин, не поднялся; cooldown 10 мин
        ▼
  Попытка №3: ХАРД-ресет, 45 сек (стандартный)            LED: hard_reset (красный, часто мигает)
        │  ждём 30 мин, не поднялся; cooldown 30 мин
        ▼
  Попытка №4: ХАРД-ресет, 5 минут (эскалация)             LED: hard_reset
        │  ждём 35 мин (30 + 5 самого обрыва), не поднялся
        ▼
  Попытка №5: ХАРД-ресет, 10 минут (эскалация)            LED: hard_reset
        │  ждём 40 мин (30 + 10), не поднялся
        ▼
  Суточный лимит (5) исчерпан -> Circuit breaker           LED: police (красный/синий)
        │
        ▼
  recovery_watch каждые 5 мин проверяет статус - если
  канал ожил сам - лимит и мьют снимаются автоматически    LED: online (зелёный)
```

## Эскалация длительности хард-ресета

Иногда короткого обрыва (30-60 сек) недостаточно — модем/вышка не успевают "забыть" зависшую регистрацию в сети. Формула для N-го хард-ресета подряд (N считается от 1):

```
N=1: стандартная короткая длительность (number.mikrotik_atlgm_hard_reset_duration, по умолчанию 45 сек)
N≥2: (N-1) × 5 минут   →   N=2: 5 мин, N=3: 10 мин, N=4: 15 мин, ...
```

Порог, после которого попытка №N считается хардом (не софтом), задаётся `input_number.mikrotik_atlgm_lte_watchdog_hard_reset_after_attempt` (по умолчанию 3). Общий суточный лимит попыток — `max_reboots_per_day` (по умолчанию **5**: 2 софт + 3 хард — 45с/5мин/10мин). Хотите ещё дальше — просто поднимите лимит, формула масштабируется сама.

**Ручной override:** независимо от автоматической эскалации, `/mikrotik_hard_reset <минуты>` форсирует конкретную длительность здесь и сейчас (например, `/mikrotik_hard_reset 15`), в обход формулы и порогов — полезно, если вы уже точно знаете, что вашей конкретной SIM/вышке нужен именно такой обрыв. `/mikrotik_hard_reset` без аргумента — стандартная короткая длительность.

Технически это работает через ESPHome-сущность `number.mikrotik_atlgm_hard_reset_override_seconds` — HA выставляет её в секундах непосредственно перед `button.press`; 0 = использовать стандартную короткую длительность. После каждого цикла ESP сам сбрасывает override обратно в 0, чтобы значение не "залипало" на будущее.

## LED-индикация

| Состояние | Стадия | LED |
|---|---|---|
| Всё в порядке | `online` | 🟢 Зелёная вспышка раз в 2 сек |
| Проблема замечена, ребут ещё не запускался | `degraded` | 🟡 Жёлтый, сплошной |
| Идёт софт-перезагрузка | `soft_reset` | 🔴 Красный, сплошной |
| Идёт аппаратный обрыв питания (любой длительности) | `hard_reset` | 🔴 Красный, часто мигает |
| Лимит исчерпан, канал так и не поднялся | `police` | 🔴🔵 Красный/синий попеременно |
| Потеряна связь HA с MikroTik или ESP | — (live-проверка) | 🟣 Фиолетовый, часто мигает |
| `/mikrotik_silence` или `/mikrotik_mute` активны | — (перекрывает всё) | 🟡 Жёлтая вспышка раз в 2 сек |

Приоритет: **mute/silence → потеря связи с оборудованием → police → hard_reset → soft_reset → degraded → online**.

## Требования

- Home Assistant 2024.8+ (`action:`, не `service:`)
- Интеграция [`mikrotik`](https://www.home-assistant.io/integrations/mikrotik/) — должна давать entity `button.<имя>_restart`
- [`hass-pfsense`](https://github.com/travisghansen/hass-pfsense) (HACS)
- `telegram_bot` — настроенная через UI интеграция
- pfSense Gateway Group с Tier 1 / Tier 2 шлюзами
- **ESP32-S3** с адресуемым RGB LED (WS2812/NeoPixel) и релейным модулем — единственная требуемая плата, реле и LED объединены в одной прошивке

## Аппаратная часть: ESP32-S3 (реле + LED)

Одна плата, один файл — `esphome/mikrotik_atlgm_watchdog.yaml`. Даёт:
- `switch.mikrotik_atlgm_relay_power` — прямое вкл/выкл питания
- `button.mikrotik_atlgm_hard_power_cycle` — атомарный цикл выключить → подождать → включить, **весь цикл на самой плате**, переживает рестарт/недоступность HA
- `number.mikrotik_atlgm_hard_reset_duration` — стандартная короткая длительность (30-60 сек, дефолт 45)
- `number.mikrotik_atlgm_hard_reset_override_seconds` — разовая длинная длительность для конкретного нажатия (см. [эскалацию выше](#эскалация-длительности-хард-ресета))
- `light.mikrotik_atlgm_status_led` — RGB-индикация (5 именованных эффектов)

**Безопасная схема реле:** режим **NC** — обесточенное реле держит контакт замкнутым (питание есть), отказ платы не обесточивает роутер навсегда. Плюс `restore_mode: RESTORE_DEFAULT_ON` на уровне прошивки.

**framework: esp-idf**, не Arduino — для линейки S3/C3 это более полнофункциональная и живая ветка поддержки в ESPHome (меньше RAM/flash, лучше RMT-периферия, на которой построен LED-эффект).

### Если у вас не ESP32-S3

В файле оставлены явные комментарии на этот случай:
- Нет адресуемого RGB LED / плата не S3 → закомментируйте весь блок `light:` (и в HA не подключайте автоматизацию `led_sync` / не переживайте о её ошибках — `light.turn_on` на несуществующую entity просто ничего не сделает, но лучше явно отключить, чтобы не копить ошибки в логе).
- Хотите на ESP8266 → поменяйте `esp32:` на `esp8266:`, `board:` под свою плату; блоки `switch:`/`button:`/`number:` платформо-независимы и заработают как есть. Только `light: platform: esp32_rmt_led_strip` не заведётся на ESP8266 (нет периферии RMT) — либо уберите LED вовсе, либо замените на обычный 3-контактный RGB через `light: platform: rgb` (те же `strobe`-эффекты поддерживаются и там, просто без адресации).

### Проверка после прошивки

Entity_id (`switch.mikrotik_atlgm_relay_power`, `button.mikrotik_atlgm_hard_power_cycle`, `number.mikrotik_atlgm_hard_reset_duration`, `number.mikrotik_atlgm_hard_reset_override_seconds`, `light.mikrotik_atlgm_status_led`) — имена из этого репозитория, ESPHome может присвоить их иначе в зависимости от версии интеграции. Сверьте через Developer Tools → States.

## Структура репозитория

```
.
├── packages/
│   └── mikrotik_atlgm_lte_watchdog.yaml   # основной модуль HA: helpers + logger + автоматизации
├── esphome/
│   └── mikrotik_atlgm_watchdog.yaml       # единая прошивка ESP32-S3: реле + LED
├── helpers.yaml                            # то же, что helpers-часть пакета — отдельно
├── automations_mikrotik_lte_watchdog.yaml  # то же, что automation-часть пакета — отдельно
├── configuration_snippets.yaml             # logger + опциональный fallback rest_command
└── README.md
```

## Установка

### 1. Включите `packages`

```yaml
homeassistant:
  packages: !include_dir_named packages
```

### 2. Скопируйте пакет HA

`packages/mikrotik_atlgm_lte_watchdog.yaml` → `/config/packages/`.

### 3. Соберите и прошейте ESP32-S3

См. [Аппаратную часть](#аппаратная-часть-esp32-s3-реле--led). Одна плата, один `esphome.name`, одно устройство в HA.

### 4. Проверьте сущности вручную

- `button.press` → `button.qtronixlte_restart` — роутер должен перезагрузиться.
- `button.press` → `button.mikrotik_atlgm_hard_power_cycle` — питание должно физически пропасть на заданное время.
- `light.turn_on` → `light.mikrotik_atlgm_status_led` с `effect: "Green Slow Blink"` — светодиод должен замигать зелёным.

### 5. Добавьте секрет для Telegram

```yaml
# Mikrotik LTE Watchdog
telegram_chat_id: -1234567890
```

### 6. Перезапустите Home Assistant

Полный перезапуск обязателен.

### 7. Проверьте, что всё подключилось

- Помощники → фильтр `mikrotik_atlgm` — 4 `input_boolean`, 1 `input_select`, 2 `input_datetime`, 9 `input_number`.
- Developer Tools → States → фильтр `lte_watchdog` — 12 `automation.*` entity.
- Бот подписан на команды.

Дальше — [Тестирование перед продом](#тестирование-перед-продом).

## Настройка под себя

| Что | Где | Значение по умолчанию |
|---|---|---|
| Entity статуса LTE-шлюза | package | `sensor.fw_village_qtronix_ru_gateway_wan_lte_mikrotik_dhcp_status` |
| Entity статуса резервного шлюза | package | `sensor.fw_village_qtronix_ru_gateway_wan_dhcp_status` |
| Entity packet loss / latency | package | `..._loss` / `..._delay` для каждого статуса выше |
| Кнопка софт-перезагрузки | package, `auto_reboot` | `button.qtronixlte_restart` |
| Кнопка хард-ресета (ESPHome) | package | `button.mikrotik_atlgm_hard_power_cycle` |
| Override-длительность (ESPHome) | package, `auto_reboot`/`telegram_hard_reset` | `number.mikrotik_atlgm_hard_reset_override_seconds` |
| Стандартная длительность (ESPHome) | package | `number.mikrotik_atlgm_hard_reset_duration` |
| LED-сущность (ESPHome) | package, `led_sync` | `light.mikrotik_atlgm_status_led` |
| Chat ID Telegram | `secrets.yaml` | `telegram_chat_id` |

## Регулируемые пороги

| Параметр | Entity | По умолчанию | Комментарий |
|---|---|---|---|
| X — порог packet loss | `input_number...packet_loss_threshold` | 20% | |
| Y — минуты `down` до первой попытки | `input_number...down_minutes_before_reboot` | 10 мин | |
| Z — пауза после софт-перезагрузки | `input_number...post_reboot_wait_minutes` | 5 мин | |
| Cooldown между софт-попытками | `input_number...soft_reboot_cooldown_minutes` | 10 мин | |
| Номер попытки для перехода на хард | `input_number...hard_reset_after_attempt` | 3 | Попытки 1-2 — софт, начиная с этого номера — хард |
| Z2 — базовая пауза после хард-ресета | `input_number...hard_reset_post_wait_minutes` | 30 мин | К ней прибавляется сама длительность обрыва, если она эскалирована (5/10 мин) |
| Лимит попыток в сутки | `input_number...max_reboots_per_day` | **5** | 2 софт + 3 хард (45с/5мин/10мин) |
| Повтор warning | `input_number...warning_repeat_minutes` | 30 мин | |
| Стандартная длительность обрыва | `number.mikrotik_atlgm_hard_reset_duration` (ESPHome) | 45 сек | Диапазон 30-60 |
| Override-длительность | `number.mikrotik_atlgm_hard_reset_override_seconds` (ESPHome) | 0 (авто) | Диапазон 0-1200 сек (0-20 мин), выставляется автоматикой/командой |
| Интервал проверки warning/recovery_watch/integration_health | `time_pattern` | 5 мин | Правится только в YAML |
| Интервал проверки reboot | `time_pattern` в `auto_reboot` | 1 мин | |

## Anti-flapping и авто-восстановление

1. **Раздельные cooldown** — короткий (10 мин) между софт-попытками, длинный (30+ мин, растёт вместе с длительностью обрыва) между хард-попытками.
2. **Эскалация по двум осям** — сначала тип действия (софт→хард), затем длительность хард-ресета (45с→5мин→10мин→...).
3. **Суточный лимит** — после исчерпания (5 по умолчанию) — circuit breaker.
4. **Авто-восстановление (`recovery_watch`)** — раз в 5 минут проверяет реальный статус; при восстановлении сам всё снимает.
5. **`/mikrotik_mute`** — для заведомо долгих проблем, без авто-таймаута.
6. **Форс-эскалация при недоступности софт-кнопки** и **защита от "нечего нажимать"**, если недоступны обе кнопки одновременно.

## Потеря связи HA с оборудованием — отдельный класс проблем

`integration_health_monitor` каждые 5 минут проверяет доступность управляющих кнопок; при потере — отдельное уведомление и фиолетовый LED; при восстановлении — подтверждение. `auto_reboot` форсирует хард-тир, если софт-кнопка недоступна, и пропускает попытку без траты cooldown/лимита, если недоступны обе.

## Управление вручную из Telegram

| Команда | Действие |
|---|---|
| `/mikrotik_silence` | Отключает уведомления и авто-действия на 1 час (авто-снятие) |
| `/mikrotik_mute` | То же самое, но бессрочно — до `/mikrotik_reset` или до восстановления канала |
| `/mikrotik_reset` | Снимает silence и mute, сбрасывает суточный счётчик и circuit breaker |
| `/mikrotik_soft_reset` | Немедленный программный ребут, в обход порогов и anti-flapping |
| `/mikrotik_hard_reset` | Немедленный аппаратный обрыв на стандартную длительность (45 сек по умолчанию) |
| `/mikrotik_hard_reset <минуты>` | Немедленный аппаратный обрыв на указанное число минут (0-20), например `/mikrotik_hard_reset 10` |

## Тестирование перед продом

**1. Бот шлёт сообщения**, **2. команды долетают до HA**, **3. `/mikrotik_silence`/`/mikrotik_mute`/`/mikrotik_reset`** — как обычно, см. предыдущие разделы репозитория/истории изменений.

**4. Ручной хард-ребут с разной длительностью:**
```
/mikrotik_hard_reset       -> обрыв на стандартные ~45 сек
/mikrotik_hard_reset 2     -> обрыв ровно на 2 минуты (проверьте секундомером)
```
Проверьте физически (по индикаторам роутера), что питание пропадает на заявленное время, а не на стандартное — это подтвердит, что override реально долетает до ESP и используется в `delay:`.

**5. LED, все 5 эффектов по очереди:**
```yaml
action: light.turn_on
target:
  entity_id: light.mikrotik_atlgm_status_led
data:
  effect: "Police Red Blue"   # затем "Purple Fast Blink", "Red Fast Blink", "Green Slow Blink", "Yellow Slow Blink"
```

**6. Детектор недоступности:** временно отключите Wi-Fi у ESP или отзовите доступ у интеграции `mikrotik` — в течение 5 минут должно прийти сообщение и LED станет фиолетовым.

**7. Автоматизация мониторинга без реальной аварии:**
```yaml
action: automation.trigger
target:
  entity_id: automation.lte_watchdog_monitoring_shliuzov_warning
data:
  skip_condition: true
```

**8. Полная эскалация — осторожно:** тот же приём для `auto_reboot` реально запустит перезагрузку нужного типа/длительности в зависимости от текущего `reboot_count_today`.

## Логирование и отладка

```bash
grep "mikrotik_atlgm_lte_watchdog" /config/home-assistant.log
```

- `STATUS: LTE(...)` (`info`) — снимок состояния.
- `REBOOT TRIGGERED: attempt=N/M type=SOFT|HARD ...` (`warning`) — запуск попытки.
- `REBOOT SKIPPED: both soft and hard reset buttons unreachable` (`error`) — нечего нажимать.
- `Post-reboot check (SOFT|HARD): ...` (`info`) — результат проверки.
- `Recovery watch: канал сам восстановился ...` (`info`) — авто-восстановление сработало.
- `INTEGRATION UNREACHABLE: soft=... hard=...` (`error`) — потеря связи с оборудованием.
- `MANUAL soft/hard reset requested via Telegram (duration_minutes=N, 0=default)` (`warning`) — ручное вмешательство, с указанием запрошенной длительности.

## entity_id автоматизаций: `id:` ≠ `entity_id`

HA генерирует `entity_id` транслитерацией `alias:`, не из `id:`. Проверяйте фактические имена через Developer Tools → States → фильтр `lte_watchdog`:

| `id:` в YAML | Реальный `entity_id` |
|---|---|
| `mikrotik_atlgm_lte_watchdog_warning_monitor` | `automation.lte_watchdog_monitoring_shliuzov_warning` |
| `mikrotik_atlgm_lte_watchdog_auto_reboot` | `automation.lte_watchdog_avto_perezagruzka_mikrotik` |
| `mikrotik_atlgm_lte_watchdog_circuit_breaker` | `automation.lte_watchdog_limit_perezagruzok_ischerpan` |
| `mikrotik_atlgm_lte_watchdog_daily_reset` | `automation.lte_watchdog_sbros_schetchika_perezagruzok_v_00_00` |
| `mikrotik_atlgm_lte_watchdog_recovery_watch` | `automation.lte_watchdog_proverka_samostoiatelnogo_vosstanovleniia` (проверьте у себя) |
| `mikrotik_atlgm_lte_watchdog_integration_health_monitor` | `automation.lte_watchdog_dostupnost_mikrotik_esp` (проверьте у себя) |
| `mikrotik_atlgm_lte_watchdog_telegram_silence` | `automation.lte_watchdog_telegram_mikrotik_silence` |
| `mikrotik_atlgm_lte_watchdog_telegram_reset` | `automation.lte_watchdog_telegram_mikrotik_reset` |
| `mikrotik_atlgm_lte_watchdog_telegram_mute` | `automation.lte_watchdog_telegram_mikrotik_mute` (проверьте у себя) |
| `mikrotik_atlgm_lte_watchdog_telegram_soft_reset` | `automation.lte_watchdog_telegram_mikrotik_soft_reset` (проверьте у себя) |
| `mikrotik_atlgm_lte_watchdog_telegram_hard_reset` | `automation.lte_watchdog_telegram_mikrotik_hard_reset` (проверьте у себя) |
| `mikrotik_atlgm_lte_watchdog_led_sync` | `automation.lte_watchdog_sinkhronizatsiia_led` (проверьте у себя) |

## Несколько роутеров

Скопируйте `packages/mikrotik_atlgm_lte_watchdog.yaml` под новым именем, замените префикс `atlgm` везде, соберите отдельные ESP32-S3 с уникальными `esphome.name`, обновите `logger:` по фактическим slug'ам после первого запуска.

## Troubleshooting

**`Integration 'packages' not found`** — `packages:` вне блока `homeassistant:`.

**`Map keys must be unique`** — используйте `packages/`.

**`patternWarning` в IDE** — косметика статической схемы, не ошибка HA.

**Имя папки packages не совпадает** — сверьте дословно.

**Уведомления не приходят** — проверьте `telegram_chat_id` в `secrets.yaml`.

**`Action failed. Can't parse entities: ...`** — уже решено, `parse_mode: html` везде.

**`The target parameter for Telegram Bot is being removed`** — уже используется `chat_id:`.

**`can't subtract offset-naive and offset-aware datetimes`** — уже переписано через `as_timestamp()`.

**`Referenced entities ... are missing` при `automation.trigger`** — используйте реальный `entity_id`, не `id:`.

**Хард-ресет всегда одной и той же длительности, override не работает** — проверьте, что `number.mikrotik_atlgm_hard_reset_override_seconds` реально меняет значение перед нажатием (Developer Tools → States, посмотрите на entity сразу после отправки команды/срабатывания автоматизации); проверьте, что в прошивке `delay:` у кнопки использует именно лямбду с чтением этого номера, а не захардкоженное значение (если правили прошивку руками).

**`/mikrotik_hard_reset 10` не распознаёт аргумент** — проверьте в логах HA событие `telegram_command`, поле `data.args` — должно быть `["10"]`. Если бот-библиотека/версия `telegram_bot` разбивает аргументы иначе, поправьте шаблон `requested_minutes` в автоматизации `telegram_hard_reset` под фактическую структуру `args`.

**LED мигает фиолетовым, хотя LTE вроде в порядке** — ожидаемо: фиолетовый про потерю связи HA с оборудованием, не про LTE.

**Авто-reboot не срабатывает, хотя шлюз в `down`** — проверьте `alert_silenced`/`muted`, cooldown, суточный лимит, доступность обеих кнопок.

## Известные ограничения

- Интервалы опроса (`time_pattern`) заданы в YAML, не выносятся в UI.
- Пакет рассчитан на одну Gateway Group с двумя шлюзами.
- И реле, и LED теперь физически на одной плате — если она полностью выйдет из строя (Wi-Fi, питание, прошивка), теряются сразу оба: и возможность аппаратного ребута, и индикация. Softwarе-ребут через интеграцию `mikrotik` при этом продолжает работать независимо.
- `entity_id` автоматизаций зависит от транслитерации `alias:` и версии HA.
- Ручные команды не имеют собственного anti-flapping.
- Верхний предел override-длительности — 20 минут (жёстко задан и в ESPHome `number`, и в шаблоне парсинга аргумента команды); если нужно больше — поднимите `max_value` в обоих местах.

## Дисклеймер

Автоматизация физически перезагружает и обесточивает сетевое оборудование без участия человека, в том числе на десятки минут при эскалации. Перед использованием в проде:
- пройдите [Тестирование перед продом](#тестирование-перед-продом) полностью, включая физическую проверку разных длительностей хард-ресета;
- убедитесь в безопасной NC-схеме подключения реле;
- подберите пороги под характер именно вашего LTE-канала — прежде чем полагаться на автоматическую эскалацию до 10+ минут, убедитесь, что именно такая длительность действительно нужна вашему оборудованию (не увеличивайте формулу "на всякий случай" без реальных наблюдений).

Используется на свой риск.

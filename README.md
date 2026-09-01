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
- [entity_id автоматизаций: id: ≠ entity_id](#entity_id-автоматизаций-id--entity_id)# MikroTik LTE Watchdog для Home Assistant

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

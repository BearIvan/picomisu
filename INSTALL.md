<p align="center"><img src="logo/picomisu.png" alt="Picomisu" width="320"></p>

# Установка Picomisu в PICO 4 Pro

Сборка образа описана в [BUILD.md](BUILD.md).

## Содержание

- [Что нужно](#0-что-нужно)
- [Снимок заводского шлема](#1-снимок-заводского-шлема-один-раз)
- [Updater recovery](#2-updater-recovery)
- [Регистрация релиза](#3-регистрация-релиза)
- [Первая установка](#4-первая-установка-usb-со-стиранием-данных)
- [Обновления](#5-обновления-wi-fi-без-стирания)
- [Возврат на заводскую прошивку](#6-возврат-на-заводскую-прошивку)
- [Известные ограничения](#известные-ограничения)

> Прошивка экспериментальная. Установка поверх заводской системы **стирает userdata**
> (игры, записи, настройки). Сохраните нужное заранее.

Все команды запускаются на Windows из клона `picomisu`:
`python tools/source-ota.py <команда>`. Инструмент пишет журнал и разрешения в
`outputs/ota/state.json`. Запись recovery, установка и стирание данных выполняются
только при соответствующем флаге `authorized` (`recovery`, `update`, `wipe`). Флаг
выставляется вручную, после осознанного решения.

## 0. Что нужно

- PICO 4 Pro (PICOA8110) на заводской **PICO OS 5.13.7**.
- Разблокированный загрузчик. На 5.13.7 ABL разблокировка закрыта токеном PICO.
  Шлем автора разблокирован старым ABL (2022) от PICO в разделе `abl` (заводской лежит в
  `ablbak`). Этот ABL **не возвращать** на заводской: устройство снова заблокируется.
- Root на заводской системе (Magisk в boot): нужен для снимка разделов и записи recovery.
- Режим разработчика и отладка по USB; USB-кабель (первая установка идёт по USB ADB).
- Заряд ≥ 30 % (`update` проверяет сам, `--min-battery`). Перед USB-операциями закрыть RenderDoc/Unity (они перезапускают adb-сервер).

## 1. Снимок заводского шлема (один раз)

```text
python tools/capture-baseline.py          # чтение system, vbmeta*, boot, recovery, калибровок (root, ≥ 9 ГБ на ПК)
python tools/prepare-preview-install.py   # копии для отката: outputs/rollback-5.13.7-for-vr-preview-01
```

Снимок нужен и для сборки образа, и как **точка возврата**: заводские recovery,
system, vbmeta_system и vbmeta проверяются по SHA-256 и хранятся только на ПК.

## 2. Updater recovery

В PICO 4 Pro нет A/B-слотов, поэтому system записывается из recovery. Updater recovery —
заводской recovery (ядро, DTB и библиотеки не меняются) с root-ADB и скриптом
`device/pico/PICOA8110/source-updater.sh`.

```text
python tools/prepare-updater-recovery.py  # собрать outputs/source-updater/updater-v2/recovery.img
python tools/source-ota.py install-recovery   # записать в раздел recovery [authorized.recovery]
```

boot, vendor, odm, product и super-метаданные не меняются.

## 3. Регистрация релиза

Скопировать `system.img`, `vbmeta.img`, `vbmeta_system.img` из WSL `outputs/source-2.21`
в Windows `outputs/source-2.21`, затем:

```text
python tools/source-ota.py register 2.21 source-2.21
python tools/source-ota.py releases
```

## 4. Первая установка (USB, со стиранием данных)

```text
python tools/source-ota.py switch 2.21 --wipe     # [authorized.update, authorized.wipe]
```

Что происходит: перезагрузка в updater recovery → временное отображение system внутри super
через `dmctl` (без изменения LP-метаданных) → потоковая запись system, vbmeta_system, vbmeta
с проверкой SHA-256 до и после → стирание userdata заводским `recovery --wipe_data` → загрузка.
Прерванную запись можно повторить той же командой: уже записанные разделы пропускаются.

После загрузки: мастер настройки VR. Учётную запись PICO можно пропустить.
ADB по Wi-Fi: `adb connect <ip-шлема>:5555`.

## 5. Обновления (Wi-Fi, без стирания)

```text
python tools/source-ota.py update 2.22            # блочная дельта: staging в cache, recovery, проверка
python tools/source-ota.py status
```

Дельта больше ~2.3 ГБ не помещается в раздел cache (2.4 ГБ). В этом случае обновление
идёт через `switch 2.22` по USB, без `--wipe`.

## 6. Возврат на заводскую прошивку

```text
python tools/source-ota.py switch factory-5.13.7 --wipe
python tools/source-ota.py restore-factory-recovery
```

Шлем возвращается к сохранённым на шаге 1 заводским system/vbmeta и recovery.

## Известные ограничения

- Сборка и установка проверены на одном шлеме (SEKO, панель INNOLUX5K).
- Телеметрия PICO не перенесена (сознательно).
- Заводское обновление PICO (SystemUpdate2) видит версию 5.13.7. Не устанавливать
  заводские OTA поверх Picomisu.

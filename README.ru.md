<p align="center"><img src="logo/picomisu.png" alt="Picomisu" width="360"></p>

<p align="center"><a href="README.md">English</a> · <b>Русский</b></p>

# Picomisu

Picomisu — системный образ (Source) для PICO 4 Pro (PICOA8110), собранный из исходников
на базе CodeLinaro **LA.UM.8.12.c3-64900-sm8250.0** (Android 10, CAF). На этом же теге основана
заводская PICO OS 5.13.7. Picomisu заменяет только `system`, `vbmeta_system` и `vbmeta`. Ядро,
boot, vendor, odm и product остаются заводскими, поэтому образ работает только на PICO OS **5.13.7**.

## Содержание

- [Сборка](#сборка)
  - [Требования](#требования)
  - [Загрузка исходников](#загрузка-исходников)
  - [Сборка релиза](#сборка-релиза)
  - [Параметры](#параметры)
  - [Обновление и стабильные релизы](#обновление-и-стабильные-релизы)
- [Установка](#установка)
- [Заводские файлы](#заводские-файлы)
- [Репозитории](#репозитории)
- [Ключи подписи](#ключи-подписи)
- [Известные ограничения](#известные-ограничения)
- [Лицензия](#лицензия)

## Сборка

### Требования

- Linux x86_64. Проверено на WSL2 с Ubuntu 24.04. Дерево должно лежать на ext4, а не на NTFS (`/mnt/c`).
- 32 ГБ ОЗУ или больше и около 200 ГБ свободного места.
- Стандартные зависимости сборки AOSP 10 и `repo`.
- `python3-brotli`, `e2fsprogs`, `rsync` и `binutils`:

  ```bash
  sudo apt install python3-brotli e2fsprogs rsync binutils
  ```

### Загрузка исходников

```bash
mkdir picomisu && cd picomisu
repo init -u https://github.com/BearIvan/picomisu.git -b main --depth=1
repo sync -c -j8 --no-tags --no-clone-bundle --optimized-fetch
```

Рекомендуем `--depth=1`: история CodeLinaro для сборки не нужна и занимает десятки ГБ.
Вместе с исходниками `repo` скачивает в `picomisu/` и инструменты сборки.

### Сборка релиза

```bash
picomisu/build.sh
```

Скрипт запускается без аргументов и делает все шаги сам:

1. Скачивает заводскую OTA PICO OS 5.13.7 по ссылке, закреплённой в
   `picomisu/config/stock-firmware.lock.json`, проверяет её SHA-256 и восстанавливает
   образы заводских разделов.
2. Извлекает заводские файлы, нужные device tree (`device/pico/PICOA8110/extract-files.py`),
   и проверяет каждый по SHA-256.
3. Собирает образ system и хост-инструменты.
4. Разбирает заводское дерево system и проверяет заводские APK.
5. Собирает образ релиза. Сборка объединяется с заводскими компонентами, которых в ней нет,
   заводские APK переподписываются, генерируются build.prop, init.rc, ld.config и VINTF.
   Затем скрипт собирает ext4 точно под размер заводского раздела, подписывает AVB
   и проверяет образ обратным чтением.
6. Выполняет офлайн-проверки: метки SELinux, init, нативные зависимости и VINTF.

Результат лежит в `out/picomisu/outputs/source-<версия>/`: `system.img`, `vbmeta_system.img`,
`vbmeta.img` и `SHA256SUMS.txt`. Версия берётся из `device/pico/PICOA8110/release.json`.

Шаги, которые монтируют образы только на чтение, выполняются через `sudo`, поэтому скрипт
спросит пароль. Запускайте его от обычного пользователя, не от root.

В сборках `userdebug` образ доверяет ADB-ключу `~/.android/adbkey.pub` того пользователя,
который собирает (`/adb_keys`). Собирайте от того же пользователя, чей `adb` будет подключаться.
Если собираете в WSL, а `adb` запускаете в Windows, сначала скопируйте Windows-ключ
`adbkey.pub` в `~/.android`.

### Параметры

Задаются переменными окружения:

| Переменная | Значение |
|---|---|
| `PICOMISU_BOOT=boot.img` | Boot, на котором работает шлем, например boot с Magisk, снятый со шлема. Его хеш записывается в `vbmeta`. По умолчанию берётся заводской boot из OTA. |
| `PICOMISU_VARIANT=user` | Собрать образ `user` вместо `userdebug`. |
| `JOBS=16` | Число параллельных задач сборки. По умолчанию 8. |
| `PICOMISU_CPUS=0-7` | Привязать сборку к этим ядрам (список для `taskset`). По умолчанию без привязки. |
| `OUT_DIR`, `PICOMISU_WORK` | Каталог выхода сборки и рабочий каталог. По умолчанию `out/` и `out/picomisu/`. |

### Обновление и стабильные релизы

Чтобы обновить дерево до ветки разработки:

```bash
repo sync -c -j8 --no-tags --no-clone-bundle --optimized-fetch
picomisu/build.sh
```

Ветка `main` этого манифеста следует за веткой разработки каждого проекта. Релизы
помечаются тегом с версией после проверки на шлеме. Под тегом манифест закрепляет точный
коммит каждого проекта. Последний релиз — **2.21**:

```bash
repo init -b refs/tags/2.21
repo sync -c -j8 --no-tags --no-clone-bundle --optimized-fetch
picomisu/build.sh
```

Что изменилось в каждом релизе, описано в `CHANGELOG.md`.

## Установка

> **Первая установка поверх заводской системы стирает userdata** (игры, записи и настройки).

Требования:

- PICO 4 Pro на заводской PICO OS 5.13.7 или на Picomisu.
- Разблокированный загрузчик. Загрузчик 5.13.7 не разблокируется без токена PICO. Шлем автора
  разблокирован старым ABL от PICO (2022) в разделе `abl`. Не возвращайте ABL 5.13.7: шлем снова
  заблокируется.
- Для первой установки — root на заводской системе (Magisk): он нужен один раз, чтобы записать
  updater recovery. В сборках Picomisu `userdebug` есть `adb root`.
- Включённые режим разработчика и отладка по USB, USB-кабель и `adb`.

Первая установка, с заводской системы:

```bash
picomisu/install.sh --wipe
```

Обновление до новой сборки с сохранением данных:

```bash
picomisu/install.sh
```

Возврат на заводскую PICO OS 5.13.7 (стирает userdata и возвращает заводской recovery):

```bash
picomisu/install.sh factory
```

`picomisu/install.sh status` показывает релиз и recovery на шлеме.

Как это работает: в PICO 4 Pro нет A/B-слотов, поэтому `system` записывается из recovery. Если
в разделе recovery ещё нет updater Picomisu, установщик собирает его из заводского recovery
закреплённой OTA и записывает. Updater — это заводской recovery с root-ADB, отключённым
интерфейсом, добавленными заводским `dmctl` и `source-updater.sh`. Затем установщик перезагружает
шлем в него и отображает `system` внутри `super` через `dmctl`. LP-метаданные он читает со шлема
и проверяет, но никогда не записывает. `system`, `vbmeta_system` и `vbmeta` записываются
проверенными частями, SHA-256 каждого раздела сверяется обратным чтением. Если кабель отключился,
запустите ту же команду ещё раз: она продолжит в recovery, а уже записанные разделы пропустит.

В updater recovery нет Wi-Fi, поэтому шлем должен быть подключён по USB. WSL не видит
USB-устройства (без `usbipd`). В этом случае запускайте установщик через Windows-Python
и Windows-`adb` на том же дереве:

```bash
python \wsl.localhost\Ubuntu-24.04\путь\к\picomisu\picomisu\tools\picomisu-install.py
```

Обновления можно получать и по Wi-Fi, через приложение **Source Update** в шлеме.

## Заводские файлы

**Заводские бинарники PICO в этих репозиториях не хранятся.** `extract-files.py` достаёт
из заводских образов 23 файла, нужных сборке: 10 JAR class path, 10 библиотек PICO/QTI,
`libcryptfs_hw.so` и 3 XML-списка Smartisan. Каждый файл сверяется с `proprietary-files.json`.
Заводские APK и другие заводские компоненты при сборке образа берутся прямо из заводского
образа system.

Сейчас Picomisu не изменяет ни одного заводского бинарника: APK только переподписываются,
а build.prop, init.rc, ld.config и VINTF генерируются из заводских файлов. Если заводской файл
когда-нибудь придётся изменить, изменение хранится как бинарный патч (поле `patch`, применяется
через `bspatch`), а не как готовый изменённый файл.

## Репозитории

Все репозитории — на github.com/BearIvan:

| Репозиторий | Путь | Содержимое |
|---|---|---|
| `picomisu` | — | этот манифест, `CHANGELOG.md` |
| `picomisu_tools` | `picomisu` | `build.sh`, инструменты релиза, OTA и проверок |
| `picomisu_device_pico_PICOA8110` | `device/pico/PICOA8110` | device tree, `release.json`, `extract-files.py` |
| `picomisu_external_gwp_asan` | `external/gwp_asan` | GWP-ASan из AOSP/LLVM (Apache-2.0 with LLVM Exceptions), как в заводской libc |
| `picomisu_external_picofacialdatadaemon` | `external/picofacialdatadaemon` | демон данных отслеживания лица и глаз (форк thoricelli, MIT) |
| `picomisu_<путь>` × 30 | `frameworks/base`, `art`, … | проекты CAF с изменениями Picomisu, ветка `picomisu` |
| `picomisu_PicoFacialDataModule` | вне дерева | модуль VRCFaceTracking для ПК (форк thoricelli) |

В каждом форке CAF основа — один коммит с деревом тега CAF. История CodeLinaro не копируется,
ссылка на неё есть в сообщении коммита. Коммиты Picomisu идут поверх этой основы. Все остальные
проекты берутся прямо с git.codelinaro.org.

## Ключи подписи

**Сейчас сборки подписываются публичными тестовыми ключами AOSP:** `build/target/product/security`
для переподписываемых заводских APK и тестовым ключом AVB для `vbmeta`. Для разработки это
нормально, но такими ключами может подписать кто угодно, поэтому для распространения они
не годятся. Ключи для релизов пока не заведены.

## Известные ограничения

- Сборка и установка проверены на одном шлеме (SEKO, панель INNOLUX5K).
- Проверен вариант `userdebug`. `user` собирается, но на шлеме не проверялся.
- Телеметрия PICO не перенесена, сознательно.
- `install.sh` проверен на шлеме при обновлении Picomisu → Picomisu. Первая установка с заводской
  системы (запись updater recovery и стирание) и `install.sh factory` этим установщиком ещё не
  выполнялись; они используют те же шаги, что и прежние установки автора.

## Лицензия

Код Picomisu в `picomisu` (этот манифест), `picomisu_tools` и `picomisu_device_pico_PICOA8110`
распространяется по [GNU General Public License v3.0](LICENSE).

Код, взятый из других проектов, сохраняет свою лицензию:

- Форки CAF/AOSP — лицензии своих исходных проектов (в основном Apache-2.0).
- `picomisu_tools/tools/third_party/avb` (avbtool) — MIT.
- `picomisu_external_gwp_asan` (из LLVM) — Apache-2.0 with LLVM Exceptions.
- `picomisu_external_picofacialdatadaemon` и `picomisu_PicoFacialDataModule` — MIT.

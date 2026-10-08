<p align="center"><img src="logo/picomisu.png" alt="Picomisu" width="360"></p>

# Picomisu

Собственная системная прошивка (Source) для PICO 4 Pro (PICOA8110) на базе
CodeLinaro **LA.UM.8.12.c3-64900-sm8250.0** (Android 10, CAF), совместимая с заводскими
vendor/odm/product PICO OS 5.13.7. Заменяется только раздел `system` (+ `vbmeta_system`,
`vbmeta`); ядро, boot, vendor, odm и product остаются заводскими.

- [BUILD.md](BUILD.md) — подготовка машины, синхронизация, сборка образа
- [INSTALL.md](INSTALL.md) — установка в шлем, обновления, возврат на заводскую прошивку

## Репозитории (все приватные, github.com/BearIvan)

| Репозиторий | Путь в дереве | Что это |
|---|---|---|
| `picomisu_manifest` | — | этот манифест и инструкции |
| `picomisu_device_pico_PICOA8110` | `device/pico/PICOA8110` | device tree шлема, `extract-files.py` |
| `picomisu_external_picofacialdatadaemon` | `external/picofacialdatadaemon` | демон данных FT/ET (форк thoricelli, MIT) |
| `picomisu_<путь>` × 30 | `art`, `frameworks/base`, … | форки проектов CAF с нашими изменениями |
| `picomisu` | отдельно, на ПК | инструменты сборки образа, OTA, проверки, исследования |
| `picomisu_PicoFacialDataModule` | отдельно, на ПК | модуль VRCFaceTracking для ПК (форк thoricelli) |

Форки CAF: ветка `picomisu` = один коммит с исходным деревом тега CAF
(история CodeLinaro не копируется, ссылка на неё в сообщении коммита) + наши коммиты.
Остальные ~720 проектов `repo` берёт напрямую с git.codelinaro.org.

Заводские файлы PICO (бинарники) в репозиториях **не хранятся**. Их извлекает
`device/pico/PICOA8110/extract-files.py` из образов заводской 5.13.7 и сверяет по SHA-256
(`proprietary-files.json`). Если заводской файл когда-нибудь потребуется изменить, изменение
хранится как бинарный патч (поле `patch`), а не как готовый файл. Сейчас ни один заводской
бинарник не изменяется: APK только переподписываются, а build.prop/init.rc/ld.config/vintf
генерируются при сборке образа из заводских файлов.

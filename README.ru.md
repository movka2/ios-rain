<div align="center">

# iOS 16 на Redmi 10C

**Ядро XNU от Apple и неизменённый userland iPhone X 16.7.16 на Xiaomi `rain` (Snapdragon 680 / SM6225).**

[English](README.md) · [Статус](#статус) · [Железо](#железо) · [Как загружается](#как-загружается) · [Сборка](#сборка-своими-руками) · [Планы](#планы) · [Благодарности](#благодарности)

</div>

> **2026-10-24: первый свет.** XNU доходит до UFS, монтирует HFS+ с iOS, запускает `launchd`, `backboardd` и `SpringBoard` и рисует экран блокировки iOS 16 в нативном размере iPhone X. Часы идут. Драйвер FocalTech SPI написан, но ещё не собран и не проверен на железе, так что это по-прежнему демонстрация bring-up.

Это порт [MakrSas/ios-topaz](https://github.com/MakrSas/ios-topaz), в котором userland iPhone X был запущен на Redmi Note 12 4G. Оригинальный README topaz с полной историей портирования XNU сохранён как [README.topaz.md](README.topaz.md).

## Статус

| Область | Состояние |
| --- | --- |
| Загрузка | Стоковый ABL Xiaomi → `fastboot boot` shim в boot image v4 → XNU. Слоты загрузки и прошивки не пишутся |
| Процессор | Четыре ядра Cortex-A73; кластер A53 остановлен, потому что XNU и userland iOS используют страницы 16 КиБ |
| Память | 1,75 ГиБ по адресу `0x80000000`, ниже carve-out прошивки |
| Хранилище | Драйвер UFS; корень iOS — журналируемый HFS+ в `userdata`, чтение/запись |
| Userland iOS | Настоящий корень и shared cache iPhone X 16.7.16 (20H392); работают `launchd` и системные демоны |
| Дисплей | Панель `ft8006s` 720×1650 DSI; для iOS выглядит как нативный iPhone X 1125×2436 @3x (375×812 точек), масштабируется в 762×1650 с обрезкой 21 px по бокам и без полей |
| Яркость | DBV панели `0x7ff` при загрузке для bring-up; интеграция CoreBrightness ещё впереди |
| GUI | `backboardd` и `SpringBoard`; экран блокировки iOS 16 с идущими часами |
| Тач | Драйвер FocalTech FTS SPI написан (`0953-rain-touch-fts-spi.patch`); ждёт CI и проверки на железе. В текущем boot-образе тач ещё выключен (`sweettouch=0`) |
| GPU | Аппаратного ускорения нет; кадры собираются на CPU при 30 Гц |
| RTC | Драйвера нет, дата начинается с 31 декабря |
| Аудио, Wi-Fi, сотовая связь, камера, датчики | Не портированы |
| Безопасность | Разрешительные AMFI/sandbox, заглушка AppleKeyStore, без SEP |

## Железо

| Компонент | Детали |
| --- | --- |
| Телефон | Xiaomi Redmi 10C (`rain`) |
| SoC | Qualcomm SM6225, 4× Cortex-A73 + 4× Cortex-A53, Adreno 610 |
| RAM | 4 ГБ LPDDR4 |
| Хранилище | UFS 2.2 |
| Панель | `ft8006s`, 720×1650, DSI video mode, 60 Гц, ~269 ppi |
| Тач | FocalTech `fts_ts`, SPI на QUP0 SE0 (`0x4a80000`), reset GPIO86, IRQ GPIO80, AVDD GPIO98 |
| Загрузчик | Разблокированный ABL Xiaomi; слот `_b` |

Ядро работает на четырёх ядрах A73. Ядра A53 не умеют транслировать страницы 16 КиБ, которые нужны XNU и userland iOS 16.

## Как загружается

```text
Xiaomi ABL (стоковый, разблокированный)
  └─ fastboot boot shim-boot-rain.img          boot image v4, только RAM
       └─ shim на загрузочном CPU
            ├─ печатает MMFR0 / MIDR, готовит GIC
            ├─ будит Gold CPU и паркует загрузочный CPU
            └─ находит payload XNU за _end
                 └─ XNU
                      ├─ драйвер UFS
                      ├─ HFS+ корень в userdata (чтение/запись)
                      ├─ dyld + shared cache
                      └─ launchd → backboardd → SpringBoard → экран блокировки
```

Последовательной консоли нет. Shim и ядро рисуют консоль в framebuffer, оставленный ABL. Ядро также хранит кольцевой лог в конце `userdata`.

## Сборка своими руками

Вам нужен разблокированный Redmi 10C, Linux-ПК с `fastboot`, аккаунт GitHub для сборки ядра и IPSW iPhone X iOS 16.7.16. **Всё в `userdata` будет стёрто.** Ничего не пишется в boot, vendor_boot, modem, persist или таблицу разделов.

### 1. Ядро

Ядро собирается на macOS-раннерах GitHub с помощью [blacktop/darwin-xnu-build](https://github.com/blacktop/darwin-xnu-build):

```sh
gh workflow run xnu.yml -R movka2/ios-rain --ref main
gh run download RUN_ID -R movka2/ios-rain -n xnu-kernel -D xnu/out
```

Результат — `xnu/out/kernel.development.sweet`.

### 2. Образ корня iOS

Используйте IPSW iPhone X (iPhone10,3) iOS 16.7.16 (20H392). В Linux с `apfs-fuse`, `hfsprogs` и sudo:

```sh
sudo bash tools/mkiosroot.sh OS.dmg SystemOS.dmg AppOS.dmg ios16-root-rain.img 16G
sudo mount -t hfsplus -o loop,force ios16-root-rain.img /mnt/iosroot
sudo python3 tools/sweet-root-tweaks.py /mnt/iosroot
sudo umount /mnt/iosroot
img2simg ios16-root-rain.img ios16-root-rain.simg
```

Конструктору device tree нужен `/product` из настоящего device tree iPhone X. Возьмите `Firmware/all_flash/DeviceTree.d22ap.im4p` из IPSW, распакуйте LZFSE payload и сохраните как `research/ipsw-x/d22ap.adt`.

### 3. ADT, payload и shim

```sh
python3 tools/mkadt.py shim/out/rain.adt --rain

python3 tools/mkpayload.py xnu/out/kernel.development.sweet \
    shim/out/rain.adt shim/out/payload-rain-debug.bin \
    --cmdline 'debug=0xc serial=3 cpus=4 keepsyms=1 -noprogress rd=ufsroot0 unentitled_ios_sim_launch=1 ignition_level=1 wdt_test=no_user_checkin ufsrw=1 rootrw=1 launchd_unsecure_cache=1 wdt=-1 sweetfb_hz=30 sweettouchch=1'

MKBOOTIMG=/usr/share/android-tools/mkbootimg/mkbootimg.py \
    shim/build.sh ../shim/out/payload-rain-debug.bin
cp shim/out/shim-boot.img shim/out/shim-boot-rain-debug.img
```

ADT описывает карту памяти rain, canvas 1125×2436 и идентичность iPhone X. Shim использует геометрию панели 720×1650 из `shim/console.h`.

### 4. Прошивка и загрузка

```sh
fastboot -S 512M flash userdata ios16-root-rain.simg   # стирает данные; ~5 минут
fastboot boot shim/out/shim-boot-rain-debug.img
```

Первый запуск надолго задерживается на логотипе Apple, пока `launchd` строит кэши; затем появляется экран блокировки. Чтобы вернуться в fastboot, зажмите Power + Volume Down.

| Аргумент загрузки | Действие |
| --- | --- |
| `rd=ufsroot0 ufsrw=1 rootrw=1` | Корень на `userdata`, чтение/запись |
| `cpus=4` | Все четыре ядра A73; `cpus=1` — безопасный fallback |
| `sweetfb_hz=30` | Частота копирования кадров на CPU |
| `sweettouch` | Включить FTS SPI драйвер тача (по умолчанию; `sweettouch=0` выключает) |
| `sweettouchdelay=45` | Задержка перед запуском тача в секундах (по умолчанию 45) |
| `sweettouchdma=1` | Принудительно использовать GENI SE DMA (авто при `FIFO_IF_DISABLE`) |
| `sweettouchwatchdog=30` | Panic через N секунд без touch-down (для bring-up) |
| `sweettouchdiag=1` | Диагностика тача и panic при выходе; `2` — ещё и panic после 3000 кадров |
| `sweetfb_native=1` | Показать сырую панель 720×1650 вместо дисплея 375×812 точек (только отладка; красная плашка UIKit возвращается) |
| `sweetfb_noconsole=1` | Перестать рисовать текст ядра после первого кадра iOS; не указывайте при отладке |
| `launchd_unsecure_cache=1`, `wdt=-1 wdt_test=no_user_checkin` | launchd без AMFI; без аппаратного watchdog |

## Планы

1. **FocalTech SPI touch.** QUP0 SE0, reset GPIO86, IRQ GPIO80, AVDD GPIO98, координаты 720×1650. Драйвер написан; собрать в CI, проверить на железе, затем заменить polling 10 мс на линию IRQ.
2. **Доводка дисплея.** CoreBrightness, тайминги DSI/MDP, питание панели, dot pitch и нормальное решение для представления 720×1650.
3. **Часы реального времени** из PMIC RTC или из shim.
4. **GPU / 60 Гц.** Сейчас ограничение — композиция на CPU.
5. **Аудио, Wi-Fi, USB и остальное железо.**

## Структура репозитория

| Путь | Что |
| --- | --- |
| `shim/` | Bare-metal загрузочный shim, boot image v4, передача Gold-ядра, настройка GIC, загрузчик payload |
| `xnu/patches/` | Серия патчей XNU, применяется CI по порядку; патчи для rain начинаются с `0951`, тач — `0953` |
| `.github/workflows/xnu.yml` | CI-сборка ядра |
| `tools/mkadt.py` | Apple Device Tree для rain (`--rain`) и topaz |
| `tools/mkpayload.py` | Упаковка ядра + ADT + аргументов загрузки в payload shim |
| `tools/mkiosroot.sh`, `tools/sweet-root-tweaks.py` | Сборка и правка HFS+ корня iOS |
| `NOTES.md` | Инженерный журнал порта topaz |
| `README.topaz.md` | Оригинальный README topaz |
| `CREDITS.md` | Лицензии и благодарности |

Не в git: бэкапы разделов, живые device tree, прошивки Apple, образы корня и build tree.

## Благодарности

- [dipa-studio/ios-sweet](https://github.com/dipa-studio/ios-sweet): оригинальный порт XNU на Qualcomm.
- [MakrSas/ios-topaz](https://github.com/MakrSas/ios-topaz): порт для Redmi Note 12 4G, на котором основано это дерево.
- [HackiOS](https://github.com/oosexgang/hackios-xnu-kernel) от OOSEX GANG: GPU, дисплей, аудио, Wi-Fi и sandbox (APSL 2.0 / MIT; см. [CREDITS.md](CREDITS.md)).
- [blacktop/darwin-xnu-build](https://github.com/blacktop/darwin-xnu-build) и [blacktop/ipsw](https://github.com/blacktop/ipsw).
- [topaz-woa](https://github.com/MakrSas/topaz-woa): документация и драйверы для SM6225.
- Открытые исходники Apple: XNU, IOHIDFamily, libpthread, hfs, а также прошивка iPhone X.

iOS принадлежит Apple и здесь не распространяется. Это экспериментальный проект; не ждите повседневной пригодности.

---

*Перевод английской версии. При расхождениях приоритет у [README.md](README.md).*

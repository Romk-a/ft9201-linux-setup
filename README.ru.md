# Настройка сканера отпечатков FocalTech FT9201 (`2808:93a9`) на Linux

*Язык: [English](README.md) | **Русский***

Пошаговое руководство, как заставить работать сканер отпечатков
**Focal-systems.Corp FT9201Fingerprint** (`USB ID 2808:93a9`) на Linux — вплоть до
полноценного входа по отпечатку (PAM: экран входа, `login`, `sudo`).

Проверено на **Astra Linux 1.8 SE** (Debian-based, DE Fly), но подход применим к
другим Debian-производным дистрибутивам (потребуется подстановка имени DM/PAM-профиля).

> Если у вас другой product ID (например `2808:9338` — GPD Win 4), патч таблицы
> USB-id из шага 7 не нужен — драйвер поддерживает его «из коробки».

> ⚠️ **Дисклеймер.** Штатной поддержки этого чипа в libfprint/fprintd НЕТ.
> Используется **проприетарный неофициальный TOD-драйвер** FocalTech + **ручной патч**
> его таблицы USB-id. Решение неофициальное; на сертифицированной системе применять
> на свой риск. Все шаги обратимы (см. раздел «Откат»).

---

## 0. Окружение, на котором проверено

| | |
|---|---|
| ОС | Astra Linux 1.8 SE (`ID_LIKE=debian`) |
| Ядро | 6.12.60-1-generic |
| DE | Fly (`fly-dm`) |
| Устройство | `Bus ... ID 2808:93a9 Focal-systems.Corp FT9201Fingerprint` |
| Secure Boot | выключен (для этого способа не важно — модули ядра не собираются) |

Проверить устройство:
```bash
lsusb | grep -i 2808
# Bus 001 Device 009: ID 2808:93a9 Focal-systems.Corp FT9201Fingerprint.
```

---

## 1. Почему именно так (контекст)

- `FT9201 (2808:93a9)` не поддерживается ни upstream libfprint, ни fprintd.
- Существует проприетарный TOD-драйвер FocalTech, упакованный в форке
  [`ryenyuku/libfprint-ft9201`](https://github.com/ryenyuku/libfprint-ft9201) —
  это `.deb` под Ubuntu 22.04 (`libfprint-2-2 1.94.4+tod1`), который целиком заменяет
  системный `libfprint-2-2` (библиотека с вшитым драйвером + udev-правила).
- Две проблемы, которые нужно обойти:
  1. **Версия.** `fprintd` в Astra требует `libfprint-2-2 (>= 1:1.94.5)`, а в `.deb`
     версия `1:1.94.4+tod1` — ниже. Иначе apt сочтёт зависимость нарушенной и откатит
     драйвер. → Пересобираем `.deb` с версией `1:1.94.5+tod1-astra1`.
  2. **Таблица id.** Внутри драйвера в таблице поддерживаемых USB-устройств только одна
     запись — `2808:9338` (это сканер GPD Win 4). Нашего `2808:93a9` там нет, поэтому
     fprintd выдаёт `No driver found`. → Патчим один `FpIdEntry`: `pid 0x9338 → 0x93a9`.
     Сам тип сенсора драйвер определяет уже после подключения (по OTP), поэтому
     железо `93a9` оказывается совместимым.

---

## 2. Предварительные проверки

```bash
# Дистрибутив и устройство
cat /etc/os-release | grep -E 'NAME|VERSION_ID'
lsusb -d 2808:93a9

# Что доступно в репозиториях (нужны fprintd и libpam-fprintd)
apt-cache policy fprintd libpam-fprintd libfprint-2-2

# Версия, которую требует fprintd (запомнить нижнюю границу, обычно >= 1:1.94.5)
apt-cache show fprintd | grep -E '^(Version|Depends)'
```

Инструменты для пересборки `.deb` (`dpkg-deb`) и патча (`python3`) есть в базовой системе.

---

## 3. Скачать и проверить совместимость драйвера

```bash
mkdir -p ~/ft9201 && cd ~/ft9201

URL="https://github.com/ryenyuku/libfprint-ft9201/releases/download/1.94.4_20250219/libfprint-2-2_1.94.4%2Btod1-0ubuntu1.22.04.2_amd64_20250219.deb"
curl -sL -o ft9201.deb "$URL"

# Метаданные и список файлов
dpkg-deb -I ft9201.deb
dpkg-deb -c ft9201.deb
```

Проверка ABI-совместимости (необязательно, но полезно): максимальная требуемая версия
символов glibc должна быть ≤ системной, а зависимые библиотеки — присутствовать.
```bash
dpkg-deb -x ft9201.deb ex
SO=ex/usr/lib/x86_64-linux-gnu/libfprint-2.so.2.0.0
objdump -T "$SO" | grep -oE 'GLIBC_[0-9.]+' | sort -V | uniq | tail -3   # напр. GLIBC_2.29
ldd --version | head -1                                                  # система, напр. 2.36
objdump -p "$SO" | awk '/NEEDED/{print $2}'                             # все должны быть в системе
```
Отдельный `libgusb.so.2` из релиза НЕ нужен, если в системе уже есть `libgusb2 (>= 0.3.0)`.

---

## 4. Пересобрать `.deb` с поднятой версией

```bash
cd ~/ft9201
rm -rf build && mkdir build
dpkg-deb -R ft9201.deb build

# Поднять версию выше требования fprintd И выше кандидата в репозитории Astra
sed -i 's/^Version:.*/Version: 1:1.94.5+tod1-astra1/' build/DEBIAN/control

# Проверка сравнения версий
dpkg --compare-versions "1:1.94.5+tod1-astra1" ge "1:1.94.5" && echo "OK: >= требования fprintd"

dpkg-deb -b build ft9201-astra.deb
```

> Если на вашей системе `fprintd` требует другую нижнюю границу — подставьте версию выше неё.

---

## 5. Установить драйвер + fprintd одной транзакцией

```bash
cd ~/ft9201
sudo apt-get install -y --allow-downgrades ./ft9201-astra.deb fprintd libpam-fprintd

# Зафиксировать libfprint, чтобы обновления системы не откатили драйвер
sudo apt-mark hold libfprint-2-2

# Проверка
dpkg -l libfprint-2-2 fprintd libpam-fprintd | grep '^ii'
# libfprint-2-2 должен быть 1:1.94.5+tod1-astra1
```

---

## 6. Применить udev-правила

`.deb` уже кладёт `/usr/lib/udev/rules.d/60-libfprint-2.rules`, где для вендора `2808`
с любым product назначается драйвер FocalTech и права `0660 plugdev`.

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger --action=add --attr-match=idVendor=2808 --attr-match=idProduct=93a9

# Должно стать "crw-rw---- root plugdev" на узле устройства
ls -la /dev/bus/usb/001/   # найдите номер своего Device из lsusb
```

Проверить, что udev-свойство выставлено на устройстве:
```bash
SYS=$(for d in /sys/bus/usb/devices/*; do \
  [ "$(cat $d/idVendor 2>/dev/null)" = 2808 ] && \
  [ "$(cat $d/idProduct 2>/dev/null)" = 93a9 ] && echo $d; done)
udevadm info -q property -p "$SYS" | grep LIBFPRINT_DRIVER
# LIBFPRINT_DRIVER=FocalTech Systems Co., Ltd Fingerprint
```

---

## 7. ⭐ Ключевой шаг: пропатчить таблицу USB-id

Таблица драйвера знает только `2808:9338`. Меняем единственную запись на `2808:93a9`.
Скрипт ищет байтовый шаблон `FpIdEntry{pid=0x9338, vid=0x2808}` (`38 93 00 00 08 28 00 00`)
— это надёжнее жёсткого смещения.

```bash
SOSYS=$(realpath /usr/lib/x86_64-linux-gnu/libfprint-2.so.2)

# Бэкап ВНЕ каталога библиотек (см. «грабли» ниже)!
sudo cp "$SOSYS" /root/libfprint-2.so.2.0.0.orig-backup

sudo python3 - "$SOSYS" <<'PY'
import sys
f = sys.argv[1]
data = bytearray(open(f, 'rb').read())
old = bytes.fromhex('3893000008280000')   # pid 0x9338 + vid 0x2808
new = bytes.fromhex('a993000008280000')   # pid 0x93a9 + vid 0x2808
n = data.count(old)
assert n == 1, f"Ожидалась 1 запись 2808:9338, найдено {n}. Не патчить вслепую!"
data[:] = data.replace(old, new)
open(f, 'wb').write(data)
print("Патч применён: 2808:9338 -> 2808:93a9")
PY

sudo ldconfig

# Проверить, что symlink указывает на пропатченный файл (НЕ на бэкап)
ls -la /usr/lib/x86_64-linux-gnu/libfprint-2.so.2
```

> 🐞 **Грабли (важно!).** Бэкап `.so` нельзя оставлять в `/usr/lib/.../` — у него тот же
> soname `libfprint-2.so.2`, и `ldconfig` может перенаправить symlink на бэкап
> (непропатченный), тогда драйвер «не увидит» устройство. Держите бэкап вне каталога
> библиотек (например, в `/root`).

---

## 8. Включить вход по отпечатку в PAM

```bash
sudo pam-auth-update --enable fprintd

# Проверка: строка pam_fprintd должна появиться в common-auth
grep pam_fprintd /etc/pam.d/common-auth
```
`common-auth` подключается к `login`, `fly-dm` (экран входа) и `sudo`. Настройка
безопасная: при неудаче отпечатка происходит fallback на пароль — система не блокируется.

---

## 9. Проверка работоспособности

```bash
sudo systemctl restart fprintd

# Устройство должно появиться: object path ".../Device/0"
dbus-send --system --print-reply --dest=net.reactivated.Fprint \
  /net/reactivated/Fprint/Manager net.reactivated.Fprint.Manager.GetDevices
```

Регистрация отпечатка — **под нужным пользователем (без sudo)**, отдельно для каждого
аккаунта (root и обычный пользователь — разные хранилища):
```bash
fprintd-enroll          # приложить палец ~13 раз до "enroll-completed"
fprintd-verify          # приложить 1 раз -> "verify-match"
fprintd-list "$USER"    # показать зарегистрированные пальцы
```

Тип сканирования — **press** (приложить, не свайп). Сообщения `enroll-swipe-too-short`
и `enroll-remove-and-retry` в процессе — нормальны (палец сняли слишком рано/нужно снять
и приложить заново), драйвер их просто пропускает.

После этого вход по отпечатку работает на экране входа `fly-dm`, при разблокировке и в
`sudo` (если `sudo` не настроен на NOPASSWD).

---

## 10. Откат

```bash
# Отключить вход по отпечатку
sudo pam-auth-update --disable fprintd

# Вернуть оригинальную (непропатченную) библиотеку
sudo cp /root/libfprint-2.so.2.0.0.orig-backup \
        $(realpath /usr/lib/x86_64-linux-gnu/libfprint-2.so.2)
sudo ldconfig

# Снять фиксацию версии и/или удалить пакеты
sudo apt-mark unhold libfprint-2-2
# Полное удаление:
sudo apt-get remove --purge fprintd libpam-fprintd
# Вернуть штатный libfprint из репозитория Astra:
sudo apt-mark unhold libfprint-2-2 && sudo apt-get install --reinstall --allow-downgrades libfprint-2-2

# Удалить отпечатки
sudo rm -rf /var/lib/fprint/*
```

---

## 11. Обслуживание и риски

- `libfprint-2-2` зафиксирован (`apt-mark hold`). Не снимайте hold без необходимости —
  обновление из репозитория Astra затрёт пропатченную библиотеку.
- После обновления **ядра** ничего делать не нужно (драйвер — userspace, не модуль ядра).
- Если кто-то снимет hold и обновит libfprint, либо переустановит пакет — нужно **заново
  выполнить шаг 7** (патч) и шаг 9 (рестарт fprintd).
- Драйвер проприетарный; на конкретной ревизии железа `93a9` поведение может отличаться.
  Если `enroll`/`verify` нестабильны — возможно, нужна другая сборка драйвера.

---

## Краткая шпаргалка (всё одним блоком)

```bash
mkdir -p ~/ft9201 && cd ~/ft9201
URL="https://github.com/ryenyuku/libfprint-ft9201/releases/download/1.94.4_20250219/libfprint-2-2_1.94.4%2Btod1-0ubuntu1.22.04.2_amd64_20250219.deb"
curl -sL -o ft9201.deb "$URL"
rm -rf build && mkdir build && dpkg-deb -R ft9201.deb build
sed -i 's/^Version:.*/Version: 1:1.94.5+tod1-astra1/' build/DEBIAN/control
dpkg-deb -b build ft9201-astra.deb
sudo apt-get install -y --allow-downgrades ./ft9201-astra.deb fprintd libpam-fprintd
sudo apt-mark hold libfprint-2-2
sudo udevadm control --reload-rules
sudo udevadm trigger --action=add --attr-match=idVendor=2808 --attr-match=idProduct=93a9
SOSYS=$(realpath /usr/lib/x86_64-linux-gnu/libfprint-2.so.2)
sudo cp "$SOSYS" /root/libfprint-2.so.2.0.0.orig-backup
sudo python3 - "$SOSYS" <<'PY'
import sys; f=sys.argv[1]; d=bytearray(open(f,'rb').read())
old=bytes.fromhex('3893000008280000'); new=bytes.fromhex('a993000008280000')
assert d.count(old)==1; d[:]=d.replace(old,new); open(f,'wb').write(d); print("patched")
PY
sudo ldconfig
sudo pam-auth-update --enable fprintd
sudo systemctl restart fprintd
fprintd-enroll && fprintd-verify
```

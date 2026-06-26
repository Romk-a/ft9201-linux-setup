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

---

## Десктоп-интеграция (Astra Linux, Fly)

Раздел специфичен для Astra Fly. После шагов выше отпечаток уже работает на **текстовой
консоли (`login`), `su` и `sudo`**. С графическими компонентами нужно повозиться.

### Проблема штатного локера Fly

У штатного локера (`fly-dm_locker_modern`) **нет UI для отпечатка**, а его помощник
`fly-wmpam` запускает PAM-авторизацию только *после отправки формы пароля*. Поэтому на
залоченном экране сканер «спит», пока не **нажмёшь Enter** (пустой/неверный пароль не
важен) — только тогда взводится `pam_fprintd`. Тупики (не тратьте время):

- Подмена бинаря `fly-dm_locker_modern` обёрткой → `fly-wm` ждёт *IPC* об анлоке, а не
  выхода процесса, и зацикливает блокировку.
- `NoPassEnable` в `/etc/X11/fly-dm/fly-dmrc` — это «вход без пароля», не биометрия.
- `kglobalaccel` не держит `Meta+L` (проверено) — клавишей владеет Fly WM.

### Экран блокировки с отпечатком: xsecurelock + матричный «дождь» (unimatrix)

Привязываем `Win+L` к [xsecurelock](https://github.com/google/xsecurelock) с сейвером
`unimatrix` (матричный «дождь» с японской катаканой) и фоновым сторожем отпечатка
(касание сканера → разблокировка, без Enter).

1. Сборка xsecurelock и инструменты сейвера. «Дождь» рисует `unimatrix` (один
   Python-скрипт; для катаканы нужен шрифт `Noto Sans CJK JP`):
   ```bash
   sudo apt-get install -y autoconf automake pkg-config xterm x11-utils \
     fonts-noto-mono fonts-noto-cjk \
     libpam0g-dev libx11-dev libxmu-dev libxcomposite-dev libxext-dev libxfixes-dev \
     libxrandr-dev libxss-dev libxft-dev
   git clone --depth 1 https://github.com/google/xsecurelock.git
   cd xsecurelock && sh autogen.sh && ./configure --with-pam-service-name=xsecurelock
   make && sudo make install      # -> /usr/local/bin/xsecurelock

   # unimatrix (матричный «дождь» с поддержкой katakana) -> /usr/local/bin/unimatrix
   sudo curl -sL -o /usr/local/bin/unimatrix \
     https://raw.githubusercontent.com/will8211/unimatrix/master/unimatrix.py
   sudo chmod 755 /usr/local/bin/unimatrix
   ```

2. PAM-сервис xsecurelock — **только пароль** (отпечаток обрабатывает сторож, в этом стеке
   его быть НЕ должно, иначе конфликт за сканер):
   ```bash
   sudo tee /etc/pam.d/xsecurelock >/dev/null <<'EOF'
   #%PAM-1.0
   auth     required  pam_unix.so try_first_pass nullok
   account  include   common-account
   EOF
   ```

3. Сейвер (рисует в окне сейвера xsecurelock через встроенный xterm). Шрифт —
   `Noto Sans Mono` с фоллбэком на `Noto Sans CJK JP` (в Mono нет катаканы):
   ```bash
   sudo tee /usr/local/libexec/xsecurelock/saver_cmatrix >/dev/null <<'EOF'
   #!/bin/sh
   WID="$XSCREENSAVER_WINDOW"
   [ -z "$WID" ] && exec /usr/local/libexec/xsecurelock/saver_blank
   W=1920; H=1080
   command -v xwininfo >/dev/null 2>&1 && \
     eval "$(xwininfo -id "$WID" 2>/dev/null | awk '/Width:/{print "W="$2} /Height:/{print "H="$2}')"
   COLS=$(( W / 8 )); ROWS=$(( H / 16 ))
   [ "$COLS" -lt 20 ] && COLS=80; [ "$ROWS" -lt 10 ] && ROWS=24
   exec xterm -into "$WID" -geometry "${COLS}x${ROWS}+0+0" \
     -fa "Noto Sans Mono,Noto Sans CJK JP" -fs 16 -bg black -fg green \
     -b 0 +sb -bc -e unimatrix -l m -s 89 -i -o
   EOF
   sudo chmod 755 /usr/local/libexec/xsecurelock/saver_cmatrix
   ```
   Настройка «дождя» — правьте последнюю строку (файл перечитывается при каждой
   блокировке): `-s` — скорость (0–100, больше = быстрее; дефолт 85); `-l m` — набор
   символов (`m` = катакана + цифры + символы, `k` — только катакана); `-fs` — размер
   шрифта. Полный список опций: `unimatrix --help`.

4. Обёртка: запускает xsecurelock + фоновый сторож `fprintd-verify`; при совпадении чисто
   снимает блокировку (`SIGTERM` заставляет xsecurelock убить детей и выйти):
   ```bash
   sudo tee /usr/local/bin/xsecurelock-fp >/dev/null <<'EOF'
   #!/bin/sh
   export XSECURELOCK_SAVER=/usr/local/libexec/xsecurelock/saver_cmatrix
   export XSECURELOCK_PASSWORD_PROMPT=cursor
   export XSECURELOCK_SHOW_DATETIME=1
   export XSECURELOCK_SHOW_HOSTNAME=0
   export XSECURELOCK_SAVER_RESET_ON_AUTH_CLOSE=1
   /usr/local/bin/xsecurelock &
   xpid=$!
   ( while kill -0 "$xpid" 2>/dev/null; do
       if fprintd-verify 2>/dev/null | grep -q 'verify-match'; then
           kill -TERM "$xpid" 2>/dev/null; break
       fi
       sleep 0.2
     done ) &
   watcher=$!
   wait "$xpid"
   kill "$watcher" 2>/dev/null
   pkill -INT -u "$(id -u)" -x fprintd-verify 2>/dev/null
   exit 0
   EOF
   sudo chmod 755 /usr/local/bin/xsecurelock-fp
   ```

5. Привязка `Win+L` — в **системном** файле `/usr/share/fly-wm/keyshortcutrc`. На этой версии Fly
   `fly-wm` при рестарте пересоздаёт `~/.fly/keyshortcutrc` из системного файла, поэтому привязка,
   записанная только в пользовательский файл, **не** сохраняется — она должна жить в системном
   (продублируйте её и в пользовательский). Заменить штатную строку `FLYWM_LOCK` в обоих:
   ```bash
   sudo sed -i 's#^Mod4|l = FLYWM_LOCK.*#Mod4|l = "/usr/local/bin/xsecurelock-fp"#' \
     /usr/share/fly-wm/keyshortcutrc
   sed -i 's#^Mod4|l = FLYWM_LOCK.*#Mod4|l = "/usr/local/bin/xsecurelock-fp"#' \
     ~/.fly/keyshortcutrc
   ```
   Используйте форму с пробелами и кавычками ровно так — вариант без пробелов `Mod4|l="…"`
   отбраковывается, и файл переименовывается в `keyshortcutrc.bad`. Перезапуск WM не нужен:
   `fly-wm` подхватывает привязку на лету (если нет — выйдите из сессии и зайдите снова). **Не**
   запускайте `fly-wmfunc FLYWM_RESTART` из рабочего терминала — это рушит сессию WM (и закрывает
   любой терминал в ней), а лёгкой команды «перечитать горячие клавиши» у fly-wm нет.

Итог: по `Win+L` идёт матричный дождь; касание сканера разблокирует мгновенно, либо
нажать клавишу и ввести пароль. Без обратного отсчёта и без штрафных «failed» попыток
(отпечаток вынесен за пределы PAM-стека локера).

> Примечание: пункт меню «Заблокировать», автоблокировка по простою и блокировка при
> засыпании по-прежнему идут через `FLYWM_LOCK` → штатный локер (Enter, затем скан). После
> обновления пакета `fly-wm` штатная строка `Mod4|l = FLYWM_LOCK` восстанавливается в системном
> файле — повторить шаг 5.

### Убрать 30-секундное ожидание отпечатка при графическом входе

Так как `pam_fprintd` стоит **первым** в `common-auth`, графический greeter (`fly-dm`)
ждёт его таймаут перед приёмом введённого пароля. Уберите отпечаток только из greeter
(сохранив его для консоли/`sudo`/`su`): замените `@include common-auth` в `/etc/pam.d/fly-dm`
на встроенную копию `common-auth` **без** строки `pam_fprintd`. Номера переходов остаются
верными: удаляемая строка была целью `default=2`/`success=1`, которые теперь ведут на
`pam_unix`, а у того `success=2` по-прежнему ведёт на `pam_permit`. Сделайте бэкап
(`sudo cp -a /etc/pam.d/fly-dm /etc/pam.d/fly-dm.orig`) и проверьте, что greeter сразу
просит пароль без «приложите палец». **Не** редактируйте сам `common-auth` — это отключит
отпечаток везде.

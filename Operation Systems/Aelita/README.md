# Настройка Aelita 24: репозитории, рабочий стол Xfce, xrdp, LibreOffice

Практическое руководство по превращению «голой» минимальной установки Aelita 24 в рабочую
многопользовательскую станцию с удалённым доступом по RDP.

Проверено на: Aelita 24 (`ID_LIKE=anolis`, CPE `cpe:/o:cqsoft:cqos:24`), ядро `5.10.92-27.xc.cq24`,
`glibc 2.28`, установка — **minimal**, доступ — `root` по SSH.

---

## 0. Что важно понимать до начала

**Aelita 24 — российская сборка китайской CqOS 24.** Разработчик CqOS — 长擎 (ChangQing),
база — Anolis OS 8 (RHEL 8-совместимый клон), userland эпохи RHEL 8.6: `glibc 2.28`, `systemd 249`,
`rpm 4.14.3`, `dnf 4.7`. Отсюда два следствия для всей дальнейшей работы:

1. Всё, что требует `glibc` новее 2.28, работать не будет (например, LibreOffice 26.x).
2. Пакеты собираются под EL8, но набор библиотек в системе не идентичен RHEL 8 — иногда
   попадаются несовпадения (см. раздел 5, chromium).

**Минимальная установка — это действительно минимум:**

* нет `tar` (критично: без него не распаковать архивы LibreOffice);
* нет графической среды и `graphical.target` не используется;
* в репозитории Aelita **нет comps-метаданных**, поэтому `dnf groupinstall "Xfce"` не сработает —
  только установка списком пакетов;
* метка безопасности: `getenforce` → `Disabled`, подсистема `cyse` (ChangYang security) отключена.
  Влияния на xrdp/SELinux-политики ждать не нужно.

**Что потребуется из сети:**

| Ресурс | Назначение |
|---|---|
| `http://repo.aelita.itgryphon.ru/repo/` | российское зеркало репозитория Aelita |
| `https://mirror.yandex.ru/epel/8/Everything/x86_64/` | EPEL 8 (Xfce, xrdp, chromium) |
| `https://download.documentfoundation.org` | архивы LibreOffice |
| `https://keyserver.ubuntu.com` | ключ подписи TDF |
| `https://repo.almalinux.org` | одна библиотека для chromium |

---

## 1. Подготовка: проверка системы и базовые утилиты

```bash
cat /etc/os-release
uname -r
rpm -q glibc
```

Ожидаемо:

```
ID_LIKE="anolis"
5.10.92-27.xc.cq24.x86_64
glibc-2.28-228.cq24.x86_64
```

Ставим то, чего в minimal обычно нет:

```bash
dnf -y install tar gpg cpio curl
```

> **`tar` — обязательный пункт.** На проверенной установке его не было, и распаковка архивов
> LibreOffice падала с `bash: tar: command not found`.

---

## 2. Репозитории: отключаем китайский, подключаем российское зеркало и EPEL

### 2.1. Что настроено «из коробки» и почему это надо менять

```
/etc/yum.repos.d/everything.repo
  baseurl = https://repo.cqsoftware.com.cn/24/Aelita/$basearch
  gpgcheck = 0
  gpgkey  = file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Aelita
```

Проверка подписи **выключена**, ключ в базу rpm **не импортирован** (`rpm -qa "gpg-pubkey*"` пуст).
То есть до сих пор с этого репозитория ставилось всё без проверки целостности.

### 2.2. Бэкап и отключение

```bash
mkdir -p /root/aelita-backup
cp -a /etc/yum.repos.d/everything.repo /root/aelita-backup/everything.repo.orig
mv /etc/yum.repos.d/everything.repo /etc/yum.repos.d/everything.repo.disabled
```

Отключение именно переименованием, а не `enabled=0` — так исходный файл остаётся неизменным,
а вернуть всё можно одной командой.

### 2.3. Ключ Aelita: сверяем и импортируем

Ключ уже лежит в системе, но проверить его происхождение нужно — иначе проверка подписи
превращается в формальность. Сверяем с копией из пакета на российском зеркале:

```bash
mkdir -p /tmp/gpgkey && cd /tmp/gpgkey
curl -fsS -O http://repo.aelita.itgryphon.ru/repo/Aelita/cqos-gpg-keys-24-29.cq24.noarch.rpm
rpm2cpio cqos-gpg-keys-24-29.cq24.noarch.rpm | cpio -idm --quiet
cmp ./etc/pki/rpm-gpg/RPM-GPG-KEY-Aelita /etc/pki/rpm-gpg/RPM-GPG-KEY-Aelita && echo "ключ идентичен"
gpg --show-keys /etc/pki/rpm-gpg/RPM-GPG-KEY-Aelita
```

Ожидаемо:

```
pub   rsa2048 2023-07-31 [SC] [expired: 2025-07-30]
      86E340A298998979C81FF0B5C195C70A34B17BC3
uid                      CYSOFT <CYSOFT@cysoftware.com.cn>
```

Обратите внимание: **срок действия ключа истёк 30.07.2025**. `rpm 4.14.3` это не блокирует —
подписи проверяются нормально (проверено), но в отчётах аудита этот факт стоит упоминать.

```bash
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-Aelita
rpm -qa "gpg-pubkey*"      # gpg-pubkey-34b17bc3-64c74e6b
```

### 2.4. Российское зеркало

```bash
cat > /etc/yum.repos.d/aelita-ru.repo <<'X'
[aelita-ru]
name=Aelita 24 - RU mirror (repo.aelita.itgryphon.ru)
baseurl=http://repo.aelita.itgryphon.ru/repo/
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Aelita
metadata_expire=6h
skip_if_unavailable=1
X
```

`repo_gpgcheck=0` — потому что зеркало не публикует подпись метаданных (`repomd.xml.asc` → 404).
Подписи самих RPM при этом проверяются всегда.

Особенности зеркала (проверено): один плоский репозиторий, 10 057 пакетов,
`Packages/` — база, `Aelita/` — 58 пакетов дельта-слоя (инсталлятор, логотипы, ядро, `cqos-*`).
Метаданные побайтово совпадают с китайским оригиналом (те же sha256 в `repomd.xml`).
Отдельного канала `security-updates` нет, хотя сайт вендора его обещает.

### 2.5. EPEL 8

```bash
curl -fsS -o /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-8 https://mirror.yandex.ru/epel/RPM-GPG-KEY-EPEL-8

cat > /etc/yum.repos.d/epel8-ru.repo <<'X'
[epel8-ru]
name=EPEL 8 - mirror.yandex.ru
baseurl=https://mirror.yandex.ru/epel/8/Everything/x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-8
metadata_expire=6h
countme=0
X
```

### 2.6. Проверка

```bash
dnf clean all && dnf makecache
dnf repolist
```

Ожидаемо ровно два репозитория:

```
repo id      repo name
aelita-ru    Aelita 24 - RU mirror (repo.aelita.itgryphon.ru)
epel8-ru     EPEL 8 - mirror.yandex.ru
```

Контроль, что проверка подписи реально работает (а не декоративна): любой пакет с зеркала

```bash
cd /tmp && curl -fsS -O http://repo.aelita.itgryphon.ru/repo/Aelita/cqos-release-24-29.cq24.x86_64.rpm
rpm -Kv cqos-release-24-29.cq24.x86_64.rpm | head -3
```

```
Header V4 RSA/SHA256 Signature, key ID 34b17bc3: OK
```

### 2.7. Откат

```bash
mv /etc/yum.repos.d/everything.repo.disabled /etc/yum.repos.d/everything.repo
rm -f /etc/yum.repos.d/aelita-ru.repo /etc/yum.repos.d/epel8-ru.repo
rpm -e gpg-pubkey-34b17bc3-64c74e6b
dnf clean all
```

---

## 3. Рабочий стол Xfce

Групп (`groupinstall`) в репозитории Aelita нет — только список пакетов.

### 3.1. База

```bash
dnf -y install \
  xfce4-session xfwm4 xfdesktop Thunar xfce4-panel xfce4-settings xfce4-terminal \
  xfce4-appfinder xfce4-power-manager xfce4-screensaver xfce4-about xfce-polkit \
  xorg-x11-server-Xorg xorg-x11-drivers dbus-x11 xdg-user-dirs desktop-file-utils
```

### 3.2. Вместо `xfce4-goodies`

Пакета `xfce4-goodies` в EL8 **не существует** — это метапакет Debian/Ubuntu.
Эквивалент собирается из отдельных пакетов EPEL:

```bash
dnf -y install \
  xfce4-notifyd xfce4-taskmanager xfce4-clipman-plugin xfce4-datetime-plugin \
  xfce4-weather-plugin xfce4-systemload-plugin xfce4-battery-plugin xfce4-cpugraph-plugin \
  xfce4-netload-plugin xfce4-diskperf-plugin xfce4-fsguard-plugin xfce4-mount-plugin \
  xfce4-pulseaudio-plugin xfce4-whiskermenu-plugin xfce4-xkb-plugin xfce4-sensors-plugin \
  xfce4-time-out-plugin xfce4-smartbookmark-plugin xfce4-verve-plugin xfce4-eyes-plugin \
  xfce4-dict thunar-archive-plugin thunar-volman
```

Чего в EPEL 8 нет: `thunar-media-tags-plugin`.

### 3.3. Прикладное ПО

```bash
dnf -y install mousepad gedit liberation-fonts
```

* `mousepad` — родной редактор Xfce (EPEL), `gedit` — второй вариант (репозиторий Aelita);
* `xfce4-screenshooter` (скриншоты) уже пришёл в наборе 3.2;
* `liberation-fonts` — метрические аналоги Arial/Times/Courier, без них в `.docx` плывёт вёрстка.

Браузера в репозиториях Aelita нет вообще (`firefox`, `chromium-browser`, `google-chrome`,
`yandex-browser` — ноль совпадений). Единственный вариант из репозитория — chromium из EPEL,
но он ставится с обходом, см. раздел 5.

---

## 4. Проверка перед настройкой xrdp

Рабочий стол проверяется через RDP (раздел 9), отдельных танцев с Xvfb не нужно. Достаточно
убедиться, что X-сервер и сессионный скрипт на месте:

```bash
rpm -q xorg-x11-server-Xorg xfce4-session
ls -l /usr/bin/startxfce4
```

> Не пытайтесь проверять Xfce запуском `startxfce4` из root-шелла без сессионной шины:
> `xfce4-session` стартует, но `xfce4-panel` и `xfdesktop` в таком окружении не поднимаются
> (проверено — в списке окон пусто). Это не признак поломки, у сессии просто нет D-Bus
> и полноценного окружения; штатный вход — через xrdp.

`xorg-x11-server-Xvfb` нужен только для проверки из раздела 9 и ставится там же.

---

## 5. Chromium: штатная установка ломается на `libffi.so.6`

```
Problem: nothing provides libffi.so.6()(64bit) needed by chromium-133.0.6943.141-1.el8.x86_64
```

Причина: сборка EPEL рассчитана на `libffi 3.1` из RHEL 8 (`libffi.so.6`), а в Aelita
`libffi 3.4.2` (`libffi.so.8`). Совместимого пакета в репозиториях нет.

Решение — положить рядом только саму библиотеку (soname другой, конфликта с системной нет)
и поставить chromium без проверки именно этой зависимости.

### 5.1. Совместимая библиотека

```bash
curl -fsS -O https://repo.almalinux.org/almalinux/8/BaseOS/x86_64/os/Packages/libffi-3.1-24.el8.x86_64.rpm
sha256sum libffi-3.1-24.el8.x86_64.rpm
```

Сверьте sha256 со значением из метаданных AlmaLinux (`BaseOS/…/repodata/*primary.xml.gz`,
пакет `libffi`, arch `x86_64`) — на проверенной системе совпало
`baa22d471c0ab0d17ab1ba372a262df3c50d6430af4259a4065dfccf0529324a`.

```bash
mkdir -p /tmp/ffi && cd /tmp/ffi
rpm2cpio /root/libffi-3.1-24.el8.x86_64.rpm | cpio -idm --quiet

mkdir -p /opt/compat-libffi6/lib64
install -m 755 usr/lib64/libffi.so.6.0.2 /opt/compat-libffi6/lib64/libffi.so.6.0.2
ln -sf libffi.so.6.0.2 /opt/compat-libffi6/lib64/libffi.so.6
echo "/opt/compat-libffi6/lib64" > /etc/ld.so.conf.d/compat-libffi6.conf
ldconfig
```

> Внутри пакета имя файла — `libffi.so.6.0.2` (не `6.0.1`); проверяйте `ls usr/lib64/`.

### 5.2. Сам chromium

```bash
cd /root
dnf -y download chromium
CR=$(ls /root/chromium-*.rpm | head -1)

DEPS=$(rpm -qp --requires "$CR" \
  | grep -vE '^(rpmlib|/bin/sh)' | grep -v '^libffi\.so\.6' \
  | awk '{print $1}' | sort -u | tr '\n' ' ')
dnf -y install $DEPS

rpm -ivh --nodeps "$CR"
```

Проверка:

```bash
chromium-browser --version
ldd /usr/bin/chromium-browser | grep "not found" || echo "все библиотеки найдены"
```

```
Chromium 133.0.6943.141 Fedora Project
```

> Ограничение решения: `rpm --nodeps` обходит только `libffi`, все остальные 111 зависимостей
> ставятся штатно, с проверкой подписи. Библиотека `libffi.so.6` лежит вне пакетного менеджера —
> при обновлении chromium переустановка не требуется, но помнить про неё нужно.

---

## 6. LibreOffice 25.8.7

### 6.1. Почему не 26.x

Версия 26.x собирается против `glibc 2.33` и на Aelita не запускается:

```
/opt/libreoffice26.8/program/oosplash: /lib64/libc.so.6: version `GLIBC_2.33' not found
```

Пакеты устанавливаются, но приложение не стартует. Рабочая ветка — **25.8.x** (25.8.7 проверена).

### 6.2. Официального yum-репозитория у TDF нет

Проверено: `https://download.documentfoundation.org/libreoffice/stable/<версия>/rpm/x86_64/repodata/repomd.xml`
отдаёт 404 для всех версий. TDF публикует только архивы `*.tar.gz` + подпись `.asc`.
В EPEL и в дистрибутивах EL8 LibreOffice отсутствует (RHEL 8 его не поставляет).

Поэтому установка — из подписанного архива, с проверкой подписи.

### 6.3. Загрузка и проверка подписи

```bash
mkdir -p /root/lo && cd /root/lo
B=https://download.documentfoundation.org/libreoffice/stable/25.8.7/rpm/x86_64

for f in LibreOffice_25.8.7_Linux_x86-64_rpm.tar.gz \
         LibreOffice_25.8.7_Linux_x86-64_rpm.tar.gz.asc \
         LibreOffice_25.8.7_Linux_x86-64_rpm_langpack_ru.tar.gz \
         LibreOffice_25.8.7_Linux_x86-64_rpm_langpack_ru.tar.gz.asc; do
  curl -fsSL -O "$B/$f"
done
```

Ключ подписи TDF:

```bash
curl -fsS "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0xC2839ECAD9408FBE9531C3E9F434A1EFAFEEAEA3" -o tdf.asc
gpg --import tdf.asc
gpg --list-keys --with-colons | awk -F: '/^fpr/{print $10; exit}'
```

Ожидаемый отпечаток: `C2839ECAD9408FBE9531C3E9F434A1EFAFEEAEA3`
(«LibreOffice Build Team (CODE SIGNING KEY) <build@documentfoundation.org>»).

> На `keys.openpgp.org` этот ключ лежит без UID, и gpg его импортирует, но использовать
> отказывается (`Can't check signature: No public key`). Поэтому берём с `keyserver.ubuntu.com`.

```bash
gpg --verify LibreOffice_25.8.7_Linux_x86-64_rpm.tar.gz.asc \
             LibreOffice_25.8.7_Linux_x86-64_rpm.tar.gz
gpg --verify LibreOffice_25.8.7_Linux_x86-64_rpm_langpack_ru.tar.gz.asc \
             LibreOffice_25.8.7_Linux_x86-64_rpm_langpack_ru.tar.gz
```

Обе команды должны дать `Good signature from "LibreOffice Build Team…"`.

### 6.4. Установка

```bash
tar xzf LibreOffice_25.8.7_Linux_x86-64_rpm.tar.gz
tar xzf LibreOffice_25.8.7_Linux_x86-64_rpm_langpack_ru.tar.gz

D=$(ls -d LibreOffice_25.8.7*_Linux_x86-64_rpm | grep -v langpack | head -1)
L=$(ls -d LibreOffice_25.8.7*_langpack_ru | head -1)

dnf -y --nogpgcheck install "$D"/RPMS/*.rpm "$L"/RPMS/*.rpm
```

`--nogpgcheck` нужен потому, что RPM внутри архива **не подписаны** — доверие обеспечивается
подписью самого архива, проверенной на шаге 6.3. Это единственное место, где проверка ослабляется,
и она заменена эквивалентной.

Проверка:

```bash
/opt/libreoffice25.8/program/soffice --version
rpm -qa | grep -cE '^(libreoffice|libobasis)25\.8'      # 45
ls /usr/share/applications/ | grep libreoffice25.8
```

```
LibreOffice 25.8.7.3 30742500f2d3eb4366ac312fa33d3dcabdb3eba5
```

Запуск: `libreoffice25.8` (или `libreoffice25.8 --writer`), ярлыки `libreoffice25.8-*.desktop`.
Каталог установки — `/opt/libreoffice25.8`.

### 6.5. Общесистемные настройки рендеринга

У `xorgxrdp` нет аппаратного OpenGL:

```
(II) AIGLX: Screen 0 is not DRI2 capable
(II) GLX: Initialized DRISWRAST GL provider for screen 0
```

Поэтому LibreOffice в RDP-сессии по умолчанию тормозит и артефактит. Правки «на каждого
пользователя» не нужны: LibreOffice подхватывает **configuration extension** из своего каталога
`share/extensions`, и он действует на все профили, включая будущие.

```bash
D=/opt/libreoffice25.8/share/extensions/zz-aelita-defaults
mkdir -p "$D/META-INF"

cat > "$D/description.xml" <<'X'
<?xml version="1.0" encoding="UTF-8"?>
<description xmlns="http://openoffice.org/extensions/description/2006">
  <identifier value="ru.aelita.lo.defaults"/>
  <version value="1.0"/>
  <display-name><name lang="en">Aelita LibreOffice defaults</name>
                <name lang="ru">Настройки LibreOffice для Aelita</name></display-name>
</description>
X

cat > "$D/META-INF/manifest.xml" <<'X'
<?xml version="1.0" encoding="UTF-8"?>
<manifest:manifest xmlns:manifest="http://openoffice.org/2001/manifest">
  <manifest:file-entry manifest:full-path="settings.xcu"
                       manifest:media-type="application/vnd.sun.star.configuration-data"/>
</manifest:manifest>
X

cat > "$D/settings.xcu" <<'X'
<?xml version="1.0" encoding="UTF-8"?>
<oor:items xmlns:oor="http://openoffice.org/2001/registry"
           xmlns:xs="http://www.w3.org/2001/XMLSchema"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <item oor:path="/org.openoffice.Office.Common/VCL"><prop oor:name="DisableOpenGL" oor:op="fuse"><value>true</value></prop></item>
  <item oor:path="/org.openoffice.Office.Common/VCL"><prop oor:name="UseSkia" oor:op="fuse"><value>false</value></prop></item>
  <item oor:path="/org.openoffice.Office.Common/VCL"><prop oor:name="ForceSkiaRaster" oor:op="fuse"><value>true</value></prop></item>
  <item oor:path="/org.openoffice.Office.Common/Misc"><prop oor:name="PreloadJVM" oor:op="fuse"><value>false</value></prop></item>
  <item oor:path="/org.openoffice.Office.Common/Misc"><prop oor:name="ShowTipOfTheDay" oor:op="fuse"><value>false</value></prop></item>
  <item oor:path="/org.openoffice.Office.Common/Misc"><prop oor:name="FirstRun" oor:op="fuse"><value>false</value></prop></item>
</oor:items>
X
```

Что даёт каждый параметр:

| Параметр | Значение | Зачем |
|---|---|---|
| `VCL/DisableOpenGL` | true | нет аппаратного GL, программный путь стабильнее |
| `VCL/UseSkia` | false | Skia на swrast даёт артефакты перерисовки |
| `VCL/ForceSkiaRaster` | true | страховка, если Skia где-то включится |
| `Misc/PreloadJVM` | false | не тянуть JVM при старте (ускоряет запуск) |
| `Misc/ShowTipOfTheDay` | false | без «подсказки дня» |
| `Misc/FirstRun` | false | не считать профиль первым запуском |

**Проверка, что настройки реально применяются** (на чистом профиле, через UNO):

```bash
X=/tmp/loverify
HOME=/tmp /opt/libreoffice25.8/program/soffice --headless --nologo --norestore \
  --accept="socket,host=127.0.0.1,port=2002;urp;" -env:UserInstallation=file://$X &

cat > /tmp/q.py <<'PY'
import uno
from com.sun.star.beans import PropertyValue
local = uno.getComponentContext()
r = local.ServiceManager.createInstanceWithContext("com.sun.star.bridge.UnoUrlResolver", local)
ctx = r.resolve("uno:socket,host=127.0.0.1,port=2002;urp;StarOffice.ComponentContext")
cp = ctx.ServiceManager.createInstanceWithContext("com.sun.star.configuration.ConfigurationProvider", ctx)
def node(p):
    x = PropertyValue(); x.Name = "nodepath"; x.Value = p
    return cp.createInstanceWithArguments("com.sun.star.configuration.ConfigurationAccess", (x,))
v = node("/org.openoffice.Office.Common/VCL"); m = node("/org.openoffice.Office.Common/Misc")
print("DisableOpenGL =", v.getByName("DisableOpenGL"))
print("UseSkia =", v.getByName("UseSkia"))
print("ForceSkiaRaster =", v.getByName("ForceSkiaRaster"))
print("PreloadJVM =", m.getByName("PreloadJVM"))
PY

/opt/libreoffice25.8/program/python /tmp/q.py
```

Ожидаемо: `True`, `False`, `True`, `False`.

> **Расширение живёт в `/opt`.** При переустановке или обновлении LibreOffice каталог
> `zz-aelita-defaults` пропадёт — его надо пересоздать (готовый блок выше).

### 6.6. Мастер первого запуска

При первом старте LibreOffice показывает модальный диалог
«Добро пожаловать в LibreOffice… Настройте параметры программы» с кнопками
«Благодарности» / «Вперёд». Пока он не закрыт, интерфейс не реагирует — со стороны клиента это
выглядит как зависание (окно при этом в состоянии `IsUnMapped`, главный поток стоит в
`SfxTabDialogController::run()`).

Через конфигурацию диалог не отключается: проверено, что `Setup/Office/MigrationCompleted=true`
и `Misc/FirstRun=false` применяются (видны через UNO), но мастер всё равно выходит.
**Штатное действие: один раз нажать «Вперёд» → «Готово».** Дальше на этом профиле не появляется.

### 6.7. Место на диске

`/opt/libreoffice25.8` ≈ 730 МБ. Архивы в `/root/lo` ≈ 240 МБ (+2 МБ языковой пакет).
Архивы можно удалить; если нужна офлайн-установка — оставить.

---

## 7. xrdp

### 7.1. Установка

```bash
dnf -y install xrdp xorgxrdp
rpm -q xrdp xorgxrdp
```

```
xrdp-0.10.6.1-3.el8.x86_64
xorgxrdp-0.10.5-1.el8.x86_64
```

Зависимости (`imlib2`, `tigervnc-server-minimal`, `zenity`) приезжают из репозитория Aelita.

### 7.2. Системная сессия — Xfce для всех пользователей

По умолчанию xrdp ищет `DefaultWindowManager=startwm-bash.sh` **относительно `/etc/xrdp`, затем
`/usr/libexec/xrdp`**, и берёт версию из `/usr/libexec` — то есть штатный скрипт, который поднимает
GNOME. Поэтому:

1. создаём свой скрипт в `/etc/xrdp/`;
2. прописываем в `sesman.ini` **абсолютный** путь.

```bash
cat > /etc/xrdp/startwm-bash.sh <<'X'
#!/bin/bash
# Aelita 24 / xrdp: системная сессия по умолчанию — Xfce
if [ -r /etc/profile ]; then . /etc/profile; fi
if [ -r "$HOME/.bash_profile" ]; then . "$HOME/.bash_profile"
elif [ -r "$HOME/.bash_login" ]; then . "$HOME/.bash_login"
elif [ -r "$HOME/.profile" ]; then . "$HOME/.profile"
fi
if [ -x "$HOME/startwm.sh" ]; then exec "$HOME/startwm.sh"; fi
if [ -r "$HOME/.xsession" ]; then exec /bin/sh "$HOME/.xsession"; fi
exec /usr/bin/startxfce4
X
chmod 755 /etc/xrdp/startwm-bash.sh

sed -i 's#^DefaultWindowManager=.*#DefaultWindowManager=/etc/xrdp/startwm-bash.sh#' /etc/xrdp/sesman.ini
grep -n '^DefaultWindowManager' /etc/xrdp/sesman.ini
```

```
10:DefaultWindowManager=/etc/xrdp/startwm-bash.sh
```

Логика приоритетов: `~/startwm.sh` → `~/.xsession` → `startxfce4`. То есть пользователь может
переопределить сессию, не трогая системную настройку.

> Если оставить относительное имя, xrdp молча запустит GNOME-сессию (`gnome-session-binary`) —
> в логе это видно как `Using the default window manager on display 10: /usr/libexec/xrdp/startwm-bash.sh`.

### 7.3. Лимиты и таймауты сессий

Оставлены **дефолтными** (сознательное решение для тестового/рабочего стенда):

| Параметр (`/etc/xrdp/sesman.ini`) | Значение | Смысл |
|---|---|---|
| `MaxSessions` | `50` | максимум одновременных сессий на сервер (общий, не на пользователя) |
| `KillDisconnected` | `false` | отключённые сессии не убиваются |
| `DisconnectedTimeLimit` | `0` | таймаут убийства (при `KillDisconnected=false` не действует) |
| `IdleTimeLimit` | `0` | автодисконнект по простою выключен |
| `Policy` | `Default` (= `UBD`) | разделение сессий по пользователю, глубине цвета и размеру экрана |

Практический эффект: закрыл окно mstsc — сессия живёт и при следующем входе возвращается тот же
рабочий стол; за простой не отключают. Отдельного лимита «сессий на пользователя» в xrdp 0.10 нет.

Если нужна другая политика:

```ini
# убивать отключённые через 10 минут
KillDisconnected=true
DisconnectedTimeLimit=600

# отключать по простою 30 минут
IdleTimeLimit=1800

# сузить общий лимит
MaxSessions=10
```

после правки — `systemctl restart xrdp`.

### 7.4. Firewall и автозапуск

```bash
firewall-cmd --permanent --add-service=rdp
firewall-cmd --reload
firewall-cmd --list-services          # cockpit dhcpv6-client rdp ssh

systemctl enable --now xrdp
systemctl is-active xrdp              # active
ss -ltnp | grep 3389
```

```
LISTEN 0  2  *:3389  *:*  users:(("xrdp",pid=…,fd=7))
```

Порт слушает и `sesman` (через `/run/xrdp/sesman.socket`), отдельного порта 3350 в 0.10 нет.

---

## 8. Пользователи и политика паролей

### 8.1. Учётные записи

```bash
for u in test1 test2; do useradd -m -s /bin/bash "$u"; done

chpasswd <<'EOP'
test1:1qa2ws3ed!QA@WS#ED
test2:1qa2ws3ed!QA@WS#ED
EOP

id test1; id test2
awk -F: '$1 ~ /^test[12]$/ {print $1, substr($2,1,6)}' /etc/shadow
```

```
test1 hash: $sm3$
```

Обратите внимание: в Aelita пароли хэшируются по **SM3** (`$sm3$`), а не SHA-512.
PAM и xrdp с этим работают штатно — проверено входом через RDP.

### 8.2. faillock

В PAM жёстко задано `deny=3 unlock_time=180`: три неудачных попытки — и учётная запись
блокируется на 3 минуты. При настройке и тестах это мешает (легко получить блокировку из-за
раскладки, см. раздел 10). Ослабляем:

```bash
cp -a /etc/pam.d/password-auth /root/aelita-backup/password-auth.orig
sed -i 's/deny=3 /deny=10 /g; s/unlock_time=180/unlock_time=60/g' /etc/pam.d/password-auth
grep -n faillock /etc/pam.d/password-auth
```

```
auth required    pam_faillock.so preauth audit deny=10 even_deny_root unlock_time=60
auth [default=die] pam_faillock.so authfail audit deny=10 even_deny_root unlock_time=60
auth sufficient  pam_faillock.so authsucc audit deny=10 even_deny_root unlock_time=60
```

Правка сделана только в `password-auth` — его включает `xrdp-sesman`. Файл `system-auth`
(консоль, ssh) не тронут, там остаётся `deny=3`.

Сброс блокировки:

```bash
faillock --user test1 --reset
```

---

## 9. Проверка: одновременная работа нескольких пользователей

Каждому пользователю xrdp поднимает **свой** X-сервер (`:10`, `:11`, …), свой `xrdp-chansrv`
и свою сессию Xfce. Проверка двумя локальными клиентами:

```bash
dnf -y install xorg-x11-server-Xvfb rdesktop

Xvfb :99 -screen 0 1280x800x24 &
Xvfb :98 -screen 0 1280x800x24 &
sleep 3

printf 'yes\n' | DISPLAY=:99 rdesktop -u test1 -p '1qa2ws3ed!QA@WS#ED' -g 1024x768 -z 127.0.0.1 &
sleep 14
printf 'yes\n' | DISPLAY=:98 rdesktop -u test2 -p '1qa2ws3ed!QA@WS#ED' -g 1024x768 -z 127.0.0.1 &
sleep 16

ps -eo user,pid,args --no-headers | grep -E "Xorg :1[0-9]|xfce4-session|xfwm4" | grep -v grep
grep -E "Access permitted|Starting X server|session is available" /var/log/xrdp-sesman.log /var/log/xrdp.log | tail
```

Ожидаемо:

```
test1  Xorg :10 …   xfce4-session   xfwm4   xfce4-panel
test2  Xorg :11 …   xfce4-session   xfwm4   xfce4-panel

sesman: Access permitted for user: test1
sesman: Starting X server on display 10 … X server :10 is working
sesman: Access permitted for user: test2
sesman: Starting X server on display 11 … X server :11 is working
xrdp:   session is available on display 10 for user test1
xrdp:   session is available on display 11 for user test2
```

Уборка после проверки:

```bash
pkill -x rdesktop; sleep 2
loginctl terminate-user test1; loginctl terminate-user test2
dnf -y remove rdesktop xorg-x11-server-Xvfb
```

Быстрая проверка «вживую» с Windows: `mstsc` → `IP:3389`, пользователь `test1` / `test2`.
Сертификат самоподписанный — принять предупреждение.

---

## 10. OpenVPN (только клиент)

Ставится целиком из российского зеркала — без EPEL, внешних зависимостей нет.

```bash
dnf -y --disablerepo='*' --enablerepo=aelita-ru install openvpn
```

`--disablerepo='*' --enablerepo=aelita-ru` гарантируют, что не приедет версия из EPEL
(там `openvpn 2.4.12`, в зеркале Aelita — `2.5.5`; обычно dnf и так выбирает более новую,
но при явном требовании «только российское» лучше указать это флагами).

Проверка:

```bash
rpm -q openvpn
openvpn --version | head -2
```

```
openvpn-2.5.5-7.cq24.x86_64
OpenVPN 2.5.5 [git:master/…] x86_64-cysoft-linux-gnu [SSL (OpenSSL)] [LZO] [LZ4] [EPOLL] [PKCS11] [MH/PKTINFO] [AEAD]
library versions: OpenSSL 1.1.1wa  16 Nov 2023, LZO 2.08
```

Пакет подписан ключом CYSOFT (`Signature: RSA/SHA256, Key ID c195c70a34b17bc3`),
вендор `ChangQing`. Ставится ровно один пакет, в нём:
`/usr/sbin/openvpn` и шаблонные юниты `openvpn-client@.service`, `openvpn-server@.service`.

Чего в российском зеркале нет и без EPEL не будет:

* `easy-rsa` — генерация PKI; при необходимости делается обычным `openssl`;
* `NetworkManager-openvpn` / `NetworkManager-openvpn-gnome` — импорт `.ovpn` и редактор в GUI.

Для клиентского сценария это не нужно: достаточно конфига и
`systemctl enable --now openvpn-client@<имя>` (конфиг кладётся в `/etc/openvpn/client/<имя>.conf`).

> Настройка VPN в это руководство не входит — выполняется по принятой в организации схеме.

---

## 11. Типовые проблемы

| Симптом | Причина | Решение |
|---|---|---|
| `User does not exist, or could not be authenticated` | mstsc шлёт скан-коды, xrdp декодирует по US-раскладке; с русской раскладкой `@` и `#` превращаются в `"` и `№` | переключить раскладку на EN (`Win+Space`) перед вводом; пароль на сервере при этом верный |
| `PAM: The account is locked due to 3 failed logins` | `faillock deny=3` | `faillock --user <u> --reset`; ослабить, см. 8.2 |
| Сессия открывается в GNOME, а не в Xfce | `DefaultWindowManager` относительный, взят из `/usr/libexec/xrdp/` | прописать абсолютный путь (7.2) |
| `nothing provides libffi.so.6` при установке chromium | в системе libffi 3.4 | раздел 5 |
| LibreOffice не запускается, `GLIBC_2.33 not found` | взята ветка 26.x | ставить 25.8.x (6.1) |
| LibreOffice «висит» при первом запуске | модальный мастер «Добро пожаловать» | нажать «Вперёд» (6.6) |
| LibreOffice тормозит/артефакты | нет аппаратного GL у xorgxrdp | расширение `zz-aelita-defaults` (6.5) |
| Клиент FreeRDP с Linux: `Dynamic Virtual Channel version 1 is not supported` | старый клиент | для проверки использовать `rdesktop`; mstsc с Windows работает |
| `can't find session: xfce4` при ручном запуске | не установлены пакеты Xfce | раздел 3 |

---

## 12. Что где лежит после настройки

```
/etc/yum.repos.d/aelita-ru.repo                российское зеркало Aelita (gpgcheck=1)
/etc/yum.repos.d/epel8-ru.repo                 EPEL 8 с зеркала Яндекса (gpgcheck=1)
/etc/yum.repos.d/everything.repo.disabled      отключённый китайский репозиторий
/etc/pki/rpm-gpg/RPM-GPG-KEY-Aelita            ключ CYSOFT (expired 2025-07-30)
/etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-8            ключ EPEL 8
/root/aelita-backup/                           бэкапы everything.repo и password-auth
/etc/xrdp/startwm-bash.sh                      системная сессия Xfce
/etc/xrdp/sesman.ini                           DefaultWindowManager (абсолютный путь)
/opt/compat-libffi6/lib64/libffi.so.6*         совместимая библиотека для chromium
/etc/ld.so.conf.d/compat-libffi6.conf          путь к ней
/opt/libreoffice25.8/                          LibreOffice 25.8.7
/opt/libreoffice25.8/share/extensions/zz-aelita-defaults/   общесистемные настройки LO
/root/lo/                                      архивы LibreOffice (можно удалить)
/usr/sbin/openvpn                              OpenVPN 2.5.5 (только из aelita-ru)
```

Итоговый состав (проверено на чистой minimal-установке):

```
xrdp-0.10.6.1-3.el8            xorgxrdp-0.10.5-1.el8
xfce4-session-4.16.0-4.el8     Thunar-4.16.8-1.el8
chromium-133.0.6943.141-1.el8  mousepad-0.5.6-1.el8   gedit-40.0-8.cq24
LibreOffice 25.8.7.3 (45 пакетов в /opt/libreoffice25.8)
```

---

## 13. Полный чек-лист

- [ ] `dnf -y install tar gpg cpio curl`
- [ ] китайский `everything.repo` отключён, бэкап сделан
- [ ] ключ CYSOFT сверен с `cqos-gpg-keys` и импортирован
- [ ] `aelita-ru.repo` + `epel8-ru.repo`, `gpgcheck=1`, `dnf makecache` без ошибок
- [ ] Xfce + набор goodies + `mousepad`/`gedit` + `liberation-fonts`
- [ ] chromium установлен через `libffi.so.6`-обход, `--version` отвечает
- [ ] LibreOffice 25.8.7 + `_langpack_ru`, подписи `.asc` проверены
- [ ] расширение `zz-aelita-defaults` создано, значения прочитаны через UNO
- [ ] xrdp + xorgxrdp, `startwm-bash.sh`, абсолютный `DefaultWindowManager`
- [ ] `firewall-cmd --add-service=rdp`, `systemctl enable --now xrdp`, порт 3389 слушается
- [ ] пользователи созданы, пароли заданы, faillock ослаблен
- [ ] двумя клиентами проверено: у каждого свой Xorg и своя сессия Xfce
- [ ] `openvpn` установлен из `aelita-ru` (`--disablerepo='*' --enablerepo=aelita-ru`), `--version` отвечает

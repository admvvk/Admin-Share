# console player — сетка RTSP-камер на Aelita 24 / xrdp

Лаунчер `cam-grid` показывает несколько потоков с камер (RTSP/HTTP/HLS) одним X-окном
в виде сетки. Работает без GPU, на программном декодировании `ffmpeg`, поэтому годится
для сессий `xrdp`/`xorgxrdp`, где аппаратного OpenGL нет.

Проверено на Aelita 24 (4 vCPU QEMU, 5.5 ГБ RAM, без GPU), xrdp 0.10.6 + xorgxrdp 0.10.5,
Xfce 4.16, реальная сессия 2560×1440.

---

## 1. Состав

| Файл в пакете | Куда ставится на машине | Назначение |
|---|---|---|
| `cam-grid` | `/usr/local/bin/cam-grid` | сам лаунчер (bash) |
| — | `/etc/cam-grid.conf` | список камер (создаётся при внедрении) |
| — | `/usr/share/applications/cam-grid.desktop` | пункт меню «Камеры (сетка)» |

Зависимости: `ffmpeg` (пакет включает `ffplay` и `ffprobe`), `xrandr` — всё из `aelita-ru`,
плюс шрифт DejaVu/Liberation для подписей (уже есть в базовой поставке Xfce).

---

## 2. Почему `ffmpeg`, а не VLC/mpv/браузер

Проверялось `dnf --assumeno install` на живой машине, результаты:

| Вариант | Итог |
|---|---|
| **ffmpeg 4.2.4 (`aelita-ru`)** | **становится штатно**: 34 пакета, 20 МБ; RTSP/UDP/TCP/HTTP/HLS/SRT, H.264, HEVC, MJPEG, MPEG-4, AAC, G.711 |
| VLC 3.0.24 (RPM Fusion free el8) | не ставится: требует `libdvdread.so.4`, в Aelita 24 `libdvdread 6.1.1` (`libdvdread.so.6`) |
| mpv 0.34.1 (RPM Fusion free el8) | не ставится: `libcdio.so.18` + `libdvdread.so.4`, в системе `libcdio 2.1.0` |
| Parole 4.16 (`epel8-ru`) + `gstreamer1-libav` (RPM Fusion) | ставится (31 пакет, 23 МБ), но это плеер на одну камеру и без сетки; в репозиториях Aelita нет декодера H.264 для GStreamer |
| Chromium 133 (`epel8-ru`, уже стоит) | **H.264 и HEVC отсутствуют** (`MediaSource.isTypeSupported('...avc1...') === false`) — веб-интерфейсы регистраторов дают чёрный экран |
| Firefox / VLC AppImage | в репозиториях Aelita нет, AppImage у VLC официально не публикуется |

Отсюда решение: единственный компонент, который ставится без сторонних репозиториев и
понимает камерные кодеки, — `ffmpeg`. Сетка собирается фильтром `xstack`, вывод — через
SDL2-outdev (`-f sdl2`), без перекодирования и без Xv/GL.

---

## 3. Внедрение

### 3.1. Зависимости

```bash
dnf -y install ffmpeg
ffmpeg -hide_banner -version | head -1     # ffmpeg version 4.2.4
```

Проверить, что RTSP и H.264 на месте:

```bash
ffmpeg -hide_banner -demuxers 2>/dev/null | grep -E ' rtsp '
ffmpeg -hide_banner -decoders 2>/dev/null | grep -E ' (h264|hevc|mjpeg) '
```

### 3.2. Скрипт

```bash
install -m 0755 cam-grid /usr/local/bin/cam-grid
bash -n /usr/local/bin/cam-grid && echo "syntax OK"
```

`/usr/local/bin` есть в `PATH` интерактивных сессий Xfce (проверено). Для `sudo`/`su`
используйте полный путь — там действует `secure_path`.

### 3.3. Конфиг камер

```bash
install -m 0644 /dev/null /etc/cam-grid.conf
```

Формат строки — `имя|url[|транспорт[|доп.опции_ffmpeg]]`:

```
# /etc/cam-grid.conf
Вход|rtsp://admin:ПАРОЛЬ@192.168.0.71:554/Streaming/Channels/101
Двор|rtsp://admin:ПАРОЛЬ@192.168.0.72:554/Streaming/Channels/102|udp
Склад|rtsps://admin:ПАРОЛЬ@192.168.0.73:554/live||-tls_verify 0
Парковка|http://admin:ПАРОЛЬ@192.168.0.74/mjpg/video.mjpg
```

* **имя** — подпись на плитке, кириллица работает (шрифт DejaVu);
* **url** — `rtsp://`, `rtsps://`, `http(s)://`, `srt://`, `file://`; логин/пароль можно в URL;
* **транспорт** — `tcp` (по умолчанию) или `udp`, применяется **только** к `rtsp(s)://`;
  для камер с потерями пакетов и для работы через NAT/VPN используйте `tcp`;
* **доп.опции** — любые ключи входа `ffmpeg`, например `-tls_verify 0` для самоподписанного
  сертификата на `rtsps://`.

Строки, начинающиеся с `#`, и пустые строки игнорируются. Порядок строк = порядок плиток
слева-направо, сверху-вниз.

### 3.4. Права доступа

В конфиге лежат пароли от камер, поэтому:

```bash
chown root:root /etc/cam-grid.conf
chmod 0640 /etc/cam-grid.conf
```

Если просмотром занимаются обычные пользователи (не root), добавьте их в отдельную группу:

```bash
groupadd -f camview
chgrp camview /etc/cam-grid.conf
usermod -aG camview test1
```

### 3.5. Пункт меню (необязательно)

```bash
cat > /usr/share/applications/cam-grid.desktop <<'X'
[Desktop Entry]
Type=Application
Version=1.0
Name=Камеры (сетка)
Comment=Сетка RTSP-камер одним окном
Exec=xfce4-terminal --geometry=80x12 --command "/usr/local/bin/cam-grid"
Icon=camera-web
Terminal=false
Categories=AudioVideo;
X
desktop-file-validate /usr/share/applications/cam-grid.desktop
```

Запуск идёт через `xfce4-terminal`, чтобы были видны сообщения `ffmpeg` и работал `Ctrl+C`.

### 3.6. Автозапуск при входе (необязательно)

```bash
mkdir -p ~/.config/autostart
cat > ~/.config/autostart/cam-grid.desktop <<'X'
[Desktop Entry]
Type=Application
Name=cam-grid
Exec=/usr/local/bin/cam-grid
X-GNOME-Autostart-enabled=true
X
```

Автозапуск стоит включать только на машинах-«витринах»: сетка без рамки займёт весь экран.

---

## 4. Проверка после установки

```bash
cam-grid --list      # список камер с маскировкой пароля
cam-grid --check     # ffprobe по каждой камере: кодек, разрешение, fps
cam-grid             # сетка
cam-grid -o 2        # одна камера (номер из --list, имя или URL)
cam-grid -h          # все опции
```

`--check` должен дать что-то вида:

```
Вход             OK    1920x1080 h264 25 fps
Двор             OK    1280x720 h264 25 fps
Склад            FAIL  Connection timed out
```

Строка `cam-grid: камер 4, сетка 2x2, тайл 1280x706, fps 15, экран 2560x1414 (отступ сверху 26)`
печатается перед стартом — по ней видно, как скрипт понял текущее разрешение и панель Xfce.

Выход из сетки: клавиша `q` в окне или `Ctrl+C` в терминале.

---

## 5. Опции

| Опция | Что делает |
|---|---|
| `-c, --conf ФАЙЛ` | другой конфиг (или `env CAM_GRID_CONF=...`) |
| `-f, --fps N` | частота кадров сетки, по умолчанию 15 |
| `--cols N` | число колонок; по умолчанию 2 для 2–4 камер, 3 для 5–9 |
| `-o, --one K` | одна камера: номер, имя или прямой URL |
| `--fullscreen` | полноэкранный режим (панель сверху перекрывается) |
| `--decorated` | обычное окно с рамкой вместо безрамочного на весь экран |
| `--no-labels` | без подписей камер |
| `--size WxH` | принудительный размер (если `xrandr` в сессии врёт) |

---

## 6. Производительность (замеры на эталонной ВМ)

4 потока (1080p, 720p, D1, 360p), сессия 2560×1440, xrdp:

| Режим | CPU |
|---|---|
| декод 1×1080p без вывода | ~7.5 % одного ядра |
| декод 4×1080p параллельно (`-f null`) | ~33 % |
| сетка 2×2, `--fps 15` | ~52 % одного ядра |
| сетка 2×2, `--fps 10` | ~41 % одного ядра |
| одна камера на весь экран | ~51 % |

RAM: ~230 МБ на процесс сетки. Потерь кадров нет (`fd=0` в статистике `ffplay`).

Практические выводы:

* 4 камеры на 4 vCPU проходят с запасом, 6–9 камер — только субпотоки (D1/720p) и `--fps 10`;
* узкое место — не декод, а отрисовка и передача по RDP, поэтому первым делом снижайте
  разрешение и fps, а не «железо»;
* на клиенте RDP ставьте 24 bpp и отключите сглаживание шрифтов — трафик заметно меньше.

---

## 7. Диагностика

| Симптом | Причина | Решение |
|---|---|---|
| `cam-grid: DISPLAY не задан` | запуск по ssh без графической сессии | запускать в сессии xrdp/X11 |
| Сетка замерла целиком | `xstack` ждёт кадры от всех входов; одна камера отвалилась | `cam-grid --check`, убрать/починить камеру; для «независимых» окон — `-o` по каждой |
| `Option rtsp_transport not found` | в потоке не-RTSP URL с явным `tcp` | ошибка старого скрипта; в текущей версии транспорт подставляется только для `rtsp(s)://` |
| `FAIL Connection refused/timeout` | камера недоступна, неверный порт/учётка | проверить `ping`, порт 554, логин/пароль в URL |
| Нет подписи на плитке | не найден шрифт | поставить `dejavu-fonts-common`/`liberation-fonts` |
| Сетка под панелью Xfce | панель не определяется | скрипт берёт её из `xfconf-query`; при нестандартной панели — `--size WxH` |
| Чёрные поля на плитке | сохранение пропорций потока | норма: картинка вписывается с letterbox; подгоняется настройкой субпотока камеры |
| Тормозит/рвётся картинка | высокий fps × разрешение | `--fps 10`, субпотоки камер, `tcp` вместо `udp` |
| Окно не закрывается с иконки | нет `q`-фокуса | `Ctrl+C` в терминале или закрыть окно крестиком (`--decorated`) |

Проверить поток в обход сетки:

```bash
ffprobe -rtsp_transport tcp -hide_banner rtsp://admin:ПАРОЛЬ@IP:554/...
ffplay  -rtsp_transport tcp -fflags nobuffer -flags low_delay -framedrop \
        -probesize 32 -analyzeduration 0 rtsp://admin:ПАРОЛЬ@IP:554/...
```

---

## 8. Откат

```bash
pkill -x ffmpeg                     # если сетка запущена
rm -f /usr/local/bin/cam-grid \
      /etc/cam-grid.conf \
      /usr/share/applications/cam-grid.desktop \
      ~/.config/autostart/cam-grid.desktop
gpasswd -d test1 camview 2>/dev/null; groupdel camview 2>/dev/null
dnf -y remove ffmpeg                # если ffmpeg больше нигде не нужен
```

Скрипт ничего не пишет в систему, кроме трёх файлов выше, и не меняет конфигурацию xrdp.

---

## 9. Чек-лист внедрения

- [ ] `dnf -y install ffmpeg`, `ffmpeg -version` отвечает
- [ ] в `ffmpeg -demuxers` есть `rtsp`, в `-decoders` — `h264`/`hevc`
- [ ] `cam-grid` установлен в `/usr/local/bin` с правами `0755`, `bash -n` проходит
- [ ] `/etc/cam-grid.conf` создан, пароли закрыты (`root:root 0640`, при необходимости `camview`)
- [ ] `cam-grid --list` показывает все камеры, пароли замаскированы
- [ ] `cam-grid --check` — все камеры `OK`, кодеки H.264 (при HEVC/1080p — субпоток)
- [ ] `cam-grid` в сессии xrdp показывает сетку, подписи читаются, панель не перекрывает
- [ ] `--fps`/`--cols` подобраны под число камер и CPU (ориентир — раздел 6)
- [ ] ярлык в меню/автозапуск (если нужно), `desktop-file-validate` проходит
- [ ] известен откат (раздел 8)

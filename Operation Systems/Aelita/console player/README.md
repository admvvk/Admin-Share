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
| `cam-grid` | `/usr/local/bin/cam-grid` | сетка из всех камер конфига (bash) |
| `cam-one` | `/usr/local/bin/cam-one` | одна камера в окне 640×360, ссылка передаётся аргументом |
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
install -m 0755 cam-one  /usr/local/bin/cam-one
bash -n /usr/local/bin/cam-grid && bash -n /usr/local/bin/cam-one && echo "syntax OK"
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

### 4.1. Одна камера в отдельном окне: `cam-one`

Когда нужна не сетка, а быстро посмотреть один поток, удобнее второй скрипт: ссылка
передаётся прямо в консоли, окно небольшое (640×360 по умолчанию), не на весь экран.

```bash
cam-one rtsp://user:pass@192.168.0.71:554/Streaming/Channels/102
cam-one --transport udp "rtsp://user:pass@192.168.0.72:554/cam/realmonitor?channel=1&subtype=1"
cam-one --size 960x540 rtsp://user:pass@192.168.0.73:554/live
cam-one --ffmpeg-args "-tls_verify 0" rtsps://user:pass@192.168.0.74:554/live
```

| Опция | Что делает |
|---|---|
| `-s, --size WxH` | размер окна, по умолчанию `640x360` |
| `-t, --transport T` | `tcp` (по умолчанию) или `udp` — только для `rtsp(s)://` |
| `-f, --fps N` | ограничить частоту кадров (меньше нагрузка) |
| `--title ТЕКСТ` | заголовок окна (по умолчанию — URL с замаскированным паролем) |
| `--audio` | включить звук потока (по умолчанию выключен) |
| `--ffmpeg-args "…"` | доп. ключи входа ffmpeg (например `-tls_verify 0`) |

Особенности: окно **с рамкой** и заданного размера, `q` в окне или `Ctrl+C` в терминале —
выход; пропорции сохраняются (по бокам чёрные поля), для рендера используется тот же
SDL2-вывод, что и в сетке. Потребление на поток 720p — ~10 % одного ядра.

---

## 5. Опции `cam-grid`

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
| HEVC-плитка первые секунды зелёная/битая | декодер стартует не с ключевого кадра (GOP камеры 2–4 с) | подождать 1–2 с; на камере уменьшить интервал ключевых кадров (обычно 1–2 с) |
| Пустые ячейки последнего ряда зелёные | `xstack` в ffmpeg 4.2 не инициализирует пустые области | исправлено в скрипте: лишние ячейки закрашиваются `drawbox=color=black` |
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
      /usr/local/bin/cam-one \
      /etc/cam-grid.conf \
      /usr/share/applications/cam-grid.desktop \
      ~/.config/autostart/cam-grid.desktop
gpasswd -d test1 camview 2>/dev/null; groupdel camview 2>/dev/null
dnf -y remove ffmpeg                # если ffmpeg больше нигде не нужен
```

Скрипт ничего не пишет в систему, кроме трёх файлов выше, и не меняет конфигурацию xrdp.

---

## 9. Тестовый стенд: виртуальные камеры (camlab)

Если реальных камер под рукой нет (или их нельзя трогать), нужен стенд, на котором видно,
что сетка собирается и играет. На эталонной ВМ такой стенд поднят как systemd-сервис.

| Компонент | Путь |
|---|---|
| RTSP-сервер MediaMTX 1.21.3 | `/opt/camlab/mediamtx` + `/opt/camlab/mediamtx.yml` |
| «Камеры» (H.264/H.265-клипы с бегущим таймкодом) | `/opt/camlab/media/cam{1..5}.mp4` |
| Супервизор: сервер + 5 публикаторов `ffmpeg -re -c copy` | `/opt/camlab/start-cameras.sh` |
| Конфиг для `cam-grid` | `/opt/camlab/cameras.conf` |
| Юнит автозапуска | `/etc/systemd/system/camlab.service` |
| Лог | `/var/log/camlab.log` |

Состав потоков (специально разные, чтобы проверить сетку и кодеки):

```
cam1  1920x1080  H.264  25 fps  3.0 Mbps   «основной» поток камеры
cam2  1280x720   H.264  25 fps  1.5 Mbps
cam3   704x576   H.264  25 fps  0.8 Mbps   D1, типичный субпоток
cam4   640x360   H.264  20 fps  0.6 Mbps
cam5  1280x720   H.265  25 fps  1.2 Mbps   проверка HEVC
```

RTSP слушает **только localhost** (`127.0.0.1:8554`), в firewall порт не открыт — стенд не
виден из сети. Публикация идёт с `-re ... -c copy`, поэтому железо почти не грузится
(≈50 МБ RAM, публикаторы почти без CPU).

### 9.1. Использование из сессии xrdp

```bash
cam-grid -c /opt/camlab/cameras.conf            # сетка 3x2 (5 камер)
cam-grid -c /opt/camlab/cameras.conf -f 10      # то же, легче по CPU
cam-grid -c /opt/camlab/cameras.conf -o 5       # только HEVC-камера
cam-grid -c /opt/camlab/cameras.conf --check    # проверить все 5 потоков
cam-one rtsp://127.0.0.1:8554/cam5              # одна камера в окне 640x360
```

`--check` на стенде даёт:

```
CAM-1 1080p          OK    1920x1080 h264 25 fps
CAM-2 720p           OK    1280x720 h264 25 fps
CAM-3 D1             OK    704x576 h264 25 fps
CAM-4 360p           OK    640x360 h264 20 fps
CAM-5 HEVC           OK    1280x720 hevc 25 fps
```

### 9.2. Управление сервисом

```bash
systemctl status camlab
systemctl stop camlab          # погасить все камеры
systemctl start camlab         # поднять снова
journalctl -u camlab -n 30     # события супервизора
tail -f /var/log/camlab.log    # RTSP-сессии, публикация
```

Сервис включён в автозапуск (`systemctl is-enabled camlab` → `enabled`). При падении любого
публикатора супервизор выходит, а `Restart=always` перезапускает стенд целиком.

### 9.3. Свои сценарии на стенде

* **Проверить реакцию на пропажу камеры:** `systemctl stop camlab` → в сетке замирают все
  плитки (`xstack` ждёт все входы), `--check` показывает `FAIL`.
* **Проверить с другим числом плиток:** скопировать конфиг и оставить 1–3 строки, задать
  `--cols`. Пустые ячейки последнего ряда закрашиваются чёрным.
* **Подменить поток:** запустить публикацию в существующий путь, например
  `ffmpeg -re -i свой.mp4 -c copy -f rtsp rtsp://127.0.0.1:8554/cam2` (после `systemctl stop
  camlab`) — `cam-grid` подхватит его как обычную камеру.
* **Открыть стенд в локальную сеть** (по умолчанию закрыт): в `/opt/camlab/mediamtx.yml`
  заменить `rtspAddress: 127.0.0.1:8554` на `:8554`, затем
  `firewall-cmd --permanent --add-port=8554/tcp && firewall-cmd --reload` и
  `systemctl restart camlab`. Это осознанное ослабление периметра — делать только на стенде.

### 9.4. Уборка стенда

```bash
systemctl disable --now camlab
rm -f /etc/systemd/system/camlab.service /opt/camlab/cameras.conf
rm -rf /opt/camlab
systemctl daemon-reload
```

## 10. Чек-лист внедрения

- [ ] `dnf -y install ffmpeg`, `ffmpeg -version` отвечает
- [ ] в `ffmpeg -demuxers` есть `rtsp`, в `-decoders` — `h264`/`hevc`
- [ ] `cam-grid` и `cam-one` установлены в `/usr/local/bin` с правами `0755`, `bash -n` проходит
- [ ] `/etc/cam-grid.conf` создан, пароли закрыты (`root:root 0640`, при необходимости `camview`)
- [ ] `cam-grid --list` показывает все камеры, пароли замаскированы
- [ ] `cam-grid --check` — все камеры `OK`, кодеки H.264 (при HEVC/1080p — субпоток)
- [ ] `cam-grid` в сессии xrdp показывает сетку, подписи читаются, панель не перекрывает
- [ ] `--fps`/`--cols` подобраны под число камер и CPU (ориентир — раздел 6)
- [ ] ярлык в меню/автозапуск (если нужно), `desktop-file-validate` проходит
- [ ] известен откат (раздел 8)

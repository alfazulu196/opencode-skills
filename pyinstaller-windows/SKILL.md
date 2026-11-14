---
name: PyInstaller exe для Windows
description: Сборка Windows-exe (onefile + onedir) для офлайн-полевых машин — spec с collect_all, транзитивные хуки, excludes для размера, кириллические пути, бандл внешних бинарников (nmap, tesseract), OpenCV/RTSP env-опции, кодировки приложения и дочерних процессов. Использовать при упаковке python-приложений в exe.
---

# PyInstaller: exe для офлайн-полевых машин

## Две поставки

Полевая практика: собирать **обе** — onefile (перенести на машину одним
файлом) и onedir (быстрый старт, нет распаковки при каждом запуске).
Билд-скрипт (tools/build_exe.py) поднимает venv, ставит requirements и
запускает PyInstaller дважды: `.spec`-файл для onefile и `--onedir`
вариант. Лог сборки сохранять в `build/pyinstaller.log`.

## Spec-файл

- Пакеты с данными/бинарями — `collect_all(...)`:
  `tmp_ret = collect_all('vosk')`, затем datas/binaries/hiddenimports
  суммируются. Типичные кандидаты: vosk, pyzbar, winrt.
- `excludes` — режут размер: pandas, tabulate, requests, pytest,
  tkinter, matplotlib (то, чего нет в рантайме).
- **Транзитивные хуки**: Pillow может не значиться в requirements
  напрямую, но притянуться (например, через qrcode → hook-PIL). Проверять
  `build/pyinstaller.log` на строки `Processing standard module hook
  'hook-<Name>.py'` — так узнаёшь, что реально в бандле. Если хук есть —
  пакет доступен в поле, отдельную зависимость добавлять не нужно.
- Данные приложения (csv и т.п.) — в `datas` с целевой папкой
  `inventory/data`; доступ из кода — относительно модуля или
  `sys._MEIPASS` для onefile.

## Дисциплина релиза

- **Не пересобирать перед выездом**: сборка заранее, проверенная; после
  правок кода — сборка только как часть релизного пакета (см. скилл
  `field-release-packet`).
- В приложении показывать «Версия сборки» из `APP_VERSION` — самотест
  на месте, что взяли свежий exe.
- После сборки: запуск собранного exe (самотест), размер файла в отчёт.

## Консоль Windows: кириллица

- **PowerShell портит кириллицу** в stdin/аргументах скриптов. Правила:
  `.bat`-файлы — только ASCII (никаких русских букв в bat); python
  запускать с `-X utf8`; многострочные тексты (commit-сообщения, prompts)
  — писать во файл и передавать через `git commit -F файл`.
- Вывод pypi/pytest в PowerShell: piping в `Select-Object` валиден,
  но сырые traceback с кириллицей могут прийти в OEM-кодировке — не
  пугаться, читать по ключам (`FAILED`, `error`).
- **cv2.imread не открывает пути с кириллицей** (папка
  «Барбацуца\! Медвед\...» — не откроет никогда). Всегда:

```python
data = np.fromfile(str(path), dtype=np.uint8)   # OSError-безопасно
img = cv2.imdecode(data, cv2.IMREAD_GRAYSCALE)
```

  То же для записи: `cv2.imencode(ext, img)[1].tofile(str(path))`.
- `QImage.save()`/Qt и openpyxl с кириллическими путями работают нормально.
- Приложение само приводит stdout/stderr к UTF-8 при старте:
  `stream.reconfigure(encoding="utf-8", errors="replace")` — иначе
  веб-панели и логи дают крякозябры.
- Дочерний python с кириллическим выводом — в окружение потомка добавить
  `PYTHONIOENCODING='utf-8'` (`subprocess.run(..., env=env)`).
- PowerShell-скрипты: в начале `#requires -Version 5.1`,
  `[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)`,
  `$OutputEncoding = ...`, `& chcp.com 65001 > $null`.
- PS 5.1 не имеет `-SkipCertificateCheck` — фолбэк `curl.exe -k` (есть в
  Win10/11), резерв `python -c "...requests.get(verify=False)"`.
- Убивать процессы только после проверки `Win32_Process.CommandLine`
  (PID принадлежит корню проекта), потом `taskkill /PID x /T /F`;
  состояние PID — в `.run/pids.json`.

## Бандл внешних бинарников

- nmap можно вшить в exe (папка `nmap_bundle` внутри PKG): при старте
  добавить её в PATH и выставить `NMAPDIR` — python-nmap найдёт бандл
  без установки на машине. python-nmap — лишь обёртка: без nmap.exe не
  работает; проверять `nmap --version` в НОВОМ терминале.
- Подобранные флаги: `-sT` явно — bundled Nmap работает даже без Npcap.
  NSE-словари для брутфорса — во временные файлы (Lua nmap не читает
  stdin), `--script-args userdb=...,passdb=...,max-connections=10,brute.timeout=1s`.
- **nmap.exe на Windows умеет зависать вечно** — жёсткий таймаут на
  процесс; при срабатывании — фолбэк на чистые сокеты (always).
- Tesseract: путь и tessdata через env (`TESSERACT_CMD`,
  `TESSDATA_PREFIX`); отсутствие — понятная ошибка с командой установки.

## OpenCV/RTSP

- Переменные окружения FFmpeg задавать ДО создания VideoCapture:
  `os.environ["OPENCV_FFMPEG_CAPTURE_OPTIONS"] = "rtsp_transport;tcp|timeout;1000000"`
  (таймаут в микросекундах; внутри — разделители `;` и `|`, не `=`).
  TCP-транспорт против потери пакетов по Wi-Fi.
- `cap.isOpened()` мало — поток валиден, только если реально прочитан
  кадр (`ret, _ = cap.read()`).

## Пути и окружение

- Данные пользователя — рядом с exe или в `%LOCALAPPDATA%`, решать
  сознательно; логи — `data/logs`, бэкапы — `data/backups` (эта схема
  показала себя в поле).
- Проверять сборку на «чистой» машине-аналоге (та же Windows, без venv
  разработчика) — ловит недобранные hiddenimports.

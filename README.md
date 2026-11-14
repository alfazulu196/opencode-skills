# База скиллов OpenCode

Наработки из полевых проектов (офлайн-инвентаризация на Getac V110),
серверных бэкендов (FastAPI/LLM) и сетевых аудитов, упакованные в
скиллы OpenCode v2. Каждый скилл — папка
с `SKILL.md`: инструкция для агента, когда и как применять проверенные
паттерны, чтобы не наступать на те же грабли.

## Скиллы

| ID | О чём | Ключевые грабли |
| -- | ----- | --------------- |
| `qt-print-pdf` | Печать PDF в PyQt6 (QPdfWriter) без новых зависимостей | AV от GC убитого QGuiApplication; QPdfWriter живёт в QtGui, не QtPdf; «тофу» кириллицы в offscreen |
| `pyqt-desktop-gotchas` | Грабли Qt-приложений и дисциплина offscreen-тестов | QApplication/QGuiApplication синглтон в тест-сьюте; itemChanged-рекурсия; скриншот-тур |
| `pyinstaller-windows` | Сборка exe для офлайн-полевых машин | collect_all, транзитивные хуки, кириллические пути cv2, PowerShell и UTF-8 |
| `offline-llm-ollama` | Локальный ИИ (ollama) в офлайн-приложении | structured output с деградацией, стрим+отмена, warmup, jsonl-журнал, stop-токены против китайского |
| `llm-json-service` | Серверная LLM-обработка документов (OpenAI-совместимый API) | fallback response_format ×3, ремонт JSON, проверка цитат против галлюцинаций, чанкинг, правила+ИИ |
| `fastapi-async-backend` | Async FastAPI + SQLAlchemy + Alembic | миграции из живого приложения без дедлока, batch для SQLite, expire_on_commit, тест-фабрика, фоновый цикл |
| `network-device-audit` | Аудит коммутаторов/камер/Windows-машин | SNMP без pysnmp, дифференцированные причины отказа, cp866, nmap-таймаут+фолбэк, hardening-словарь с откатом |
| `field-release-packet` | Процесс выпуска пакета фич | тесты → docs → версия → бандл → пересборка; не пересобирать перед выездом |
| `git-dual-cloud` | Зеркала GitHub+GitLab и бэкапы | два pushurl на origin, защищённая main, bundle, commit -F файл |

## Подключение на устройстве

1. Клонировать репозиторий в любое место:

```sh
git clone <url> ~/opencode-skills
```

2. Указать папку в глобальном конфиге `~/.config/opencode/opencode.json`:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "skills": ["~/opencode-skills"]
}
```

`~/`-путь резолвится от домашнего каталога. Скиллы подхватываются без
перезапуска OpenCode; проверить, что скилл виден: в чате упомянуть
`@qt-print-pdf` (ID = имя папки, регистр важен).

3. Обновление на устройстве — обычный `git pull` в папке репозитория.

## Как дополнять

- Новый скилл = новая папка `<kebab-case-id>/SKILL.md`, frontmatter:
  `name` (отображаемое имя) и `description` (когда применять — по нему
  модель находит скилл; без description скилл не рекламируется).
- Поддерживающие файлы (references/, scripts/) кладите рядом с SKILL.md —
  пути в тексте скилла относительные от его папки.
- Один скилл = одна тема. Внутри — только переносимое: без путей и
  специфики конкретного проекта.
- После дополнения: коммит → пуш в оба облака (см. скилл `git-dual-cloud`).

## Лицензия

MIT — см. [LICENSE](LICENSE).

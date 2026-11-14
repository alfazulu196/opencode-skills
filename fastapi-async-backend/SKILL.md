---
name: Async FastAPI + SQLAlchemy + Alembic
description: Асинхронный бэкенд на FastAPI — SQLAlchemy 2.0 async + SQLite (PRAGMA, expire_on_commit, selectinload), миграции Alembic из живого приложения без дедлока event loop, пагинация с count-запросом, фоновые задачи в lifespan, тест-фабрика на TestClient, локальный HTTPS CA для закрытого контура. Использовать при разработке/отладке FastAPI-бэкендов.
---

# Async FastAPI: SQLAlchemy + Alembic + SQLite

## Async SQLAlchemy 2.0 + SQLite

- `connect_args={"check_same_thread": False}`; PRAGMA
  (`foreign_keys=ON`) вешать event-листенером на **sync_engine**, не
  async.
- `async_sessionmaker(..., expire_on_commit=False)` — иначе объекты
  «отваливаются» после `commit()` и ломают ответы API.
- В async-режиме ленивые загрузки падают — связи тянуть `selectinload`
  всегда.
- После `commit()` перечитать объект для ответа с
  `.execution_options(populate_existing=True)` — identity map вернёт
  устаревшее.
- SQLite хранит naive UTC — входящие значения приводить к этому виду
  одним хелпером.

## Миграции Alembic: async и запуск из живого приложения

- SQLite не умеет ALTER COLUMN — `render_as_batch=(dialect == "sqlite")`
  (Alembic пересобирает таблицу); индексы тоже через
  `batch_alter_table` — однообразие.
- Async-миграции: `create_async_engine(..., poolclass=NullPool)` +
  `await connection.run_sync(do_run_migrations)`.
- **Дедлок event loop**: `env.py` вызывает `asyncio.run()`, поэтому
  миграции из живого FastAPI — только через отдельный поток:
  `await asyncio.to_thread(_run_migrations_sync, url)`.
- URL в `alembic.ini` намеренно пустой, передавать программно:
  `config.attributes["sqlalchemy_url"]`; логи alembica глушить
  `config.attributes["configure_logger"] = False`.
- Три сценария инициализации: новая база → `upgrade head`; старая без
  `alembic_version` → проверить полноту схемы (inspector vs
  `Base.metadata.tables`; неполная → RuntimeError с советом
  восстановить бэкап) → идемпотентно добавить легаси-колонки
  (`PRAGMA table_info` + ADD COLUMN) → `command.stamp("0001")` →
  upgrade. Тесты: fresh-база, легаси-база, защита от неполной схемы.

## API

- Пагинация: `limit: int = Query(50, ge=1, le=200)`, отдельный
  `select(func.count())` с тем же списком `conditions`; стабильная
  сортировка для offset — второй ключ `(created_at desc, id desc)`;
  тест на `limit=0` → 422.
- «Тяжёлые» колонки (OCR-текст, сотни КБ) не отдавать в списках:
  булев флаг `has_ocr_text`, полный текст — отдельный эндпоинт.
- Глобальные хендлеры: `LookupError→404`, `ValueError→400` — сервисный
  слой не знает про HTTP.
- Статику монтировать последним — иначе перекроет `/api` и `/docs`.
- Загрузка файлов: лимит размера ДО парсинга (413/400).
- CORS строкой через запятую; при `*` — `allow_credentials=False`.

## Lifespan и фоновый цикл

- `lifespan`: миграции → bootstrap →
  `asyncio.create_task(loop, name=...)`; в finally — `cancel()` +
  await с проглатыванием `CancelledError`.
- Фоновый цикл: стартовая задержка (даём БД подняться),
  `except asyncio.CancelledError: raise` ВЫШЕ общего
  `except Exception`, минимум N секунд между проходами.
- Гонка ручного и фонового запуска: модульный `_running` флаг +
  исключение Busy.
- Дедуп событий: уникальный `event_key` (`f"deadline_changed:{iso}"`) —
  повторный проход не дублирует уведомления.

## Тесты: фабрика без fixtures (unittest)

```python
self._temp = tempfile.TemporaryDirectory(); self.addCleanup(self._temp.cleanup)
engine = create_async_engine(f"sqlite+aiosqlite:///{path.as_posix()}", poolclass=NullPool)
app = FastAPI(); app.include_router(router)
app.dependency_overrides[get_db] = override_get_db
app.dependency_overrides[get_current_user] = lambda: User(...)
self.client = TestClient(app)
```

- URL — через `as_posix()` (обратные слэши Windows ломают
  SQLAlchemy-URL); NullPool — база не держит файлы.
- pytest.ini: `addopts = -ra`, игнор `DeprecationWarning`.

## Конфиг и линтер: прагматичный старт

- pydantic-settings: `env_file=".env", env_ignore_empty=True,
  extra="ignore"`; все лимиты/таймауты — поля с дефолтами.
- ruff: `select = ["E4","E7","E9","F"]` (+ ignore `E402`) — стилистику
  UP/I/B включать после рефакторинга, иначе baseline 1000+ замечаний;
  `extend-exclude` для папок данных.
- mypy: `ignore_missing_imports`, `check_untyped_defs = false` — тоже
  поэтапно.

## HTTPS в закрытом контуре: локальный CA

Свой CA (10 лет) + серверный серт (~397 дней, автообновление за 30
дней до конца); CA установить в доверенные (`trust_ca.ps1`), клиенты
ходят с `verify=CA-файл`, а не `verify=False`. Секретные файлы писать
атомарно: `.tmp` + `os.replace` + chmod 0o600.

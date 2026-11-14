---
name: Печать PDF в PyQt6
description: Печатные документы (акты, чек-листы, этикетки): QPdfWriter в PyQt6 — координаты в мм, кириллица, перенос страниц, access violation, визуальная проверка рендера; альтернативы вне Qt — ReportLab, fpdf2, python-docx, печать openpyxl. Использовать при любой генерации PDF/документов/этикеток.
---

# Печать PDF в PyQt6 через QPdfWriter

Печатные документы без новых зависимостей: QPdfWriter уже входит в PyQt6.
Сюда вынесены грабли, на которые было потрачено время в полевом проекте
(акт инвентаризации, чек-лист докрутки, листы этикеток).

## Золотые правила

1. **QPdfWriter импортируется из `PyQt6.QtGui`**, НЕ `PyQt6.QtPdf` —
   модуля QtPdf в PyQt6 нет, `from PyQt6.QtPdf import QPdfWriter` даёт
   `ModuleNotFoundError`.
2. **QGuiApplication — в модульной переменной.** Если создать его в
   локальной переменной функции, GC уничтожит C++ объект при выходе, и
   первый же `painter.drawText(...)` упадёт
   `Windows fatal exception: access violation` без стека в PDF-коде.

```python
_APP: QGuiApplication | None = None

def _new_writer(out_path):
    global _APP
    if QGuiApplication.instance() is None:
        _APP = QGuiApplication([])   # живёт до конца процесса
    writer = QPdfWriter(str(out_path))
    writer.setResolution(300)        # dpi
    return writer, QPainter(writer), writer.resolution() / 25.4
```

3. **Кириллица — только с явным шрифтом:** `QFont("Segoe UI")`
   (есть на любой Windows). Без этого текст может уйти «тофу».
4. **Единицы:** `mm = writer.resolution() / 25.4` — множитель мм →
   единицы документа. Вся геометрия в мм × множитель.
5. **Qt-грабли:** `drawRect`/`drawRoundedRect`/`fillRect` — только
   `QRectF`; `drawLine` — `QPointF`; перенос страницы — `writer.newPage()`
   (Qt6); `drawText(rect, int(flags), text)` — флаги как
   `Qt.AlignmentFlag.AlignHCenter | Qt.TextFlag.TextWordWrap` приводить
   к `int`.

## Каркас документа (таблица с переносом страниц)

Проверенный каркас: заголовок-функция `draw_head(y)` рисует шапку на
каждой странице; перед строкой проверять `y + row_h > page_end_y` →
`writer.newPage()` и `y = margin`; высота строки — по самому высокому
`painter.boundingRect(rect, int(Qt.TextFlag.TextWordWrap), text)`.
Подписи («Инспектор/Заказчик» + линии `painter.drawLine(QPointF, QPointF)`)
переносить на новую страницу отдельной проверкой.

## Как проверять рендер без человека у экрана

1. **PDF глазами не прочитать** — рендерить в PNG тем же painter-кодом:
   monkeypatch на `_new_writer`, возвращающий `(FakeWriter, QPainter(img), mm)`
   с `QImage` размера A4 при ~150 dpi. `FakeWriter` — заглушка с
   `setResolution/resolution/newPage`. Для «только страница 1» —
   `newPage()` кидает исключение-маркер, которое ловится снаружи.
2. **Снимать PNG на РЕАЛЬНОЙ платформе** (`windows`), не offscreen:
   в offscreen на Windows `QFontDatabase.families()` возвращает **0
   семейств** — весь текст будет «тофу», это артефакт платформы, а не кода.
3. **Тесты** гонять в offscreen и проверять структуру: файл начинается
   с `%PDF`, размер > порога, `data.count(b"/Type /Page") >= N` для
   переноса страниц. Мультистраничную вёрстку на одной QImage не
   проверять — страницы рисуются поверх друг друга (артефакт прокси).

## Типовой приём: «сохрани и покажи»

Экспорт печатного документа из диалога: путь
`<exports>/<prefix>_<имя>_<штамп>.pdf` (штамп `%Y%m%d_%H%M%S`, имя —
через regex-санитайзер `\w-`); статус-лейбл зелёным с полным путём +
кнопка «Открыть папку» через `QDesktopServices.openUrl(QUrl.fromLocalFile(...))`.
Ошибки генерации — `QMessageBox.critical` с текстом исключения, диалог
не ронять.

## ReportLab — сервер/CLI без Qt

Если рендер не нужен в Qt-приложении, ReportLab короче. Проверено:
- TTF-фолбэк цепочкой: встроенный DejaVu → `C:\Windows\Fonts\arial.ttf`/
  segoeui → Linux dejavu → Helvetica аварийно; для жирных Paragraph —
  `registerFontFamily`.
- «Страница X из Y»: NumberedCanvas — `showPage()` копит
  `dict(self.__dict__)`, в `save()` total известен → перерисовать
  футеры всех страниц.
- Любая пользовательская строка в Paragraph — через
  `xml.sax.saxutils.escape()`.
- Блок «итоги + подписи» — `KeepTogether`; шапка таблицы на новых
  страницах — `repeatRows=1`; ширины колонок масштабировать под
  доступную ширину (`scale = available / sum(widths)`).
- Тест PDF: `%PDF-`-магия + pypdf-извлечение текста — кириллица реально
  в тексте, «Страница 1 из» на месте.

## fpdf2

Без TTF PDF невозможен: искать `arial.ttf` в `C:\Windows\Fonts`,
регистрировать под коротким именем и всегда его указывать. После
`multi_cell` курсор стоит у правого поля — новую ячейку/линию всегда
начинать от левого поля, иначе «не влезает».

## python-docx

Заливка ячейки цветом — только через XML
(`<w:shd w:val="clear" w:fill="..."/>`); таблицы — стиль `'Table Grid'`.

## Excel-печать (openpyxl)

- Печать одной связкой: `ws.print_title_rows = f"1:{header_row}"`,
  `page_setup.fitToWidth = 1` и обязательно
  `ws.sheet_properties.pageSetUpPr.fitToPage = True`, колонтитул
  `ws.oddFooter.right.text = "Страница &P из &N"`.
- Живые формулы вместо значений (`=ROUND(F2*(1+G2/100),2)`) +
  `wb.calculation.fullCalcOnLoad = True` — пересчёт при открытии.
- Служебный лист (наценка/маржа), который клиент не должен видеть —
  `sheet_state = "hidden"`.
- Защита от формульных инъекций из недоверенных данных: текст,
  начинающийся на `= + - @`, префиксовать `'`.
- CSV для Excel — `utf-8-sig`.

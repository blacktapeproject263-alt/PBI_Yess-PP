# Руководство по переходу с Qlik Sense на Power BI (на примере проекта Yess-PP)

Данное руководство систематизирует различия в архитектуре, ETL, вычислениях и интерфейсе между **Qlik Sense** и **Power BI Desktop**, основываясь на практическом кейсе ритейл-аналитики `Yess-PP`.

---

## 1. Концептуальные различия движков

| Характеристика | Qlik Sense (QIX Engine) | Power BI (VertiPaq / Analysis Services) |
| :--- | :--- | :--- |
| **Принцип связывания** | Автоматическая ассоциация по одинаковым именам полей | Явные физические реляционные связи (1:N, 1:1, M:N) по выбранным ключам |
| **Синтетические ключи** | Создаются автоматически при 2+ общих полях (требуют Link Table) | Запрещены; при неоднозначности возникает ошибка Ambiguity |
| **Направление фильтра** | Двунаправленное во всей модели (зеленый / белый / серый) | Однонаправленное (по умолчанию от измерения к факту) |
| **Вычисления** | Выражения на лету на стороне визуала (Set Analysis) | Строго типизированные меры DAX в модели данных |
| **Таблица дат** | Master Calendar с флагами | Выделенная непрерывная Date Table (`Calendar`), помеченная как Date Table |

---

## 2. Мост между языками: Power Query (M) vs Qlik Load Script

| Операция | Qlik Sense | Power BI (Power Query M) |
| :--- | :--- | :--- |
| **Загрузка и переименование** | `LOAD Field as NewField FROM ...;` | `Table.RenameColumns(Source, {{"Field", "NewField"}})` |
| **Транспонирование (Unpivot)** | `Crosstable(Month, Plan, 1) LOAD ...;` | `Table.UnpivotOtherColumns(Source, {"StoreID"}, "Date", "Plan")` |
| **Обогащение (Lookup)** | `ApplyMap('MapTable', Key)` | `Table.NestedJoin` (Merge Queries) $\rightarrow$ Expand |
| **Объединение таблиц** | `Concatenate LOAD ...;` | `Table.Combine` (Append Queries) |
| **Преобразование дат** | `Date#(TextDate, 'DD.MM.YYYY')` | `Table.TransformColumnTypes(Source, {{"Date", type date}})` |

---

## 3. Мост в формулах: Set Analysis vs CALCULATE() в DAX

В Qlik логика срезов данных задается через **Set Analysis** (`{< ... >}`). В Power BI ее прямым эквивалентом является функция **`CALCULATE()`**, модифицирующая контекст фильтра.

```dax
// Qlik: Sum({<Color={'Red'}>} Sales)
Red Sales = CALCULATE([4. Sales], Products[Color] = "Red")

// Qlik: Sum({<Products=>} Sales) -- сброс фильтра измерения
Sales All Products = CALCULATE([4. Sales], ALL(Products))

// Qlik: Sum(Sales) / Sum({1} Sales) -- доля в общем объеме
Sales Share = DIVIDE([4. Sales], CALCULATE([4. Sales], ALL(Sales)), 0)

// Qlik: Sum({<InYTD={1}>} Sales) -- продажи с начала года
Sales YTD = CALCULATE([4. Sales], DATESYTD('Calendar'[Date]))

// Qlik: Sum({<InSPLY={1}>} Sales) -- аналогичный период прошлого года
Sales SPLY = CALCULATE([4. Sales], SAMEPERIODLASTYEAR('Calendar'[Date]))
```

---

## 4. Складские остатки (Полуаддитивные меры)

В отличие от выручки, товарный остаток на складе нельзя суммировать по оси времени:
```dax
Кол-во закуплено накопительно = 
CALCULATE(
    [Purchase Quantity], 
    FILTER(ALL('Calendar'), 'Calendar'[Date] <= MAX('Calendar'[Date]))
)

Кол-во продаж накопительно = 
CALCULATE(
    [2. SalesQuantity], 
    FILTER(ALL('Calendar'), 'Calendar'[Date] <= MAX('Calendar'[Date]))
)

Остатки = [Кол-во закуплено накопительно] - [Кол-во продаж накопительно]
```

---

## 5. Эквиваленты интерфейса и UX

* **Alternate States (Сравнение срезов)** $\rightarrow$ Использование disconnected-таблиц (не связанных таблиц параметров) и функции `TREATAS` / `USERELATIONSHIP`.
* **Контейнеры (Containers)** $\rightarrow$ Связка **Bookmarks (Закладки) + Панель Selection (Выделение) + Кнопки переключения**.
* **Всплывающие подсказки** $\rightarrow$ **Report Page Tooltips** (страницы отчета малого формата, привязанные к визуалу).
* **Переход к детализации** $\rightarrow$ **Drill-through** поля и кнопки перехода.

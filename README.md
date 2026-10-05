# Power BI Enterprise Analytics: Yess-PP

[![Power BI](https://img.shields.io/badge/Power_BI-Desktop-F2C811?logo=powerbi&logoColor=black)](https://powerbi.microsoft.com/)
[![DAX](https://img.shields.io/badge/DAX-Analysis_Services-0078D4)](https://learn.microsoft.com/dax/)
[![Microsoft Fabric](https://img.shields.io/badge/Fabric-TMDL-5C2D91?logo=microsoft)](https://learn.microsoft.com/fabric/)
[![Architecture](https://img.shields.io/badge/Model-Star_Schema-green)](#архитектура-модели)

Комплексный аналитический проект Power BI для сети розничных магазинов электроники. Проект охватывает полный цикл BI-разработки: от продвинутого ETL на Power Query (M) и реляционного моделирования до Time Intelligence, полуаддитивных мер товарных остатков и мультивалютных расчетов.

---

## 📁 Структура репозитория

```text
PBI_Yess-PP/
├── 📄 README.md                        # Главная документация проекта и архитектурное описание
├── 📄 .gitignore                       # Исключение временных файлов и бинарных кэшей
├── 📂 reports/                         # Готовые аналитические отчеты Power BI
│   └── 📊 Yess-PP.pbix                 # Главный отчет с настроенным UI/UX, KPI и дашбордами
├── 📂 Yess-PP.SemanticModel/           # Семантическая модель в текстовом формате TMDL (Fabric / Git)
│   ├── 📄 database.tmdl                # Определение базы данных
│   ├── 📄 model.tmdl                   # Метаданные модели и параметры
│   ├── 📄 relationships.tmdl           # Спецификация физических связей 1:N
│   ├── 📄 expressions.tmdl             # Запросы Power Query (M)
│   └── 📂 tables/                      # TMDL-определения таблиц и мер с комментариями '///'
│       ├── Sales.tmdl                  # Факты продаж (31 расчетная мера)
│       ├── SalesPlan.tmdl              # Планы продаж по магазинам и датам
│       ├── Products.tmdl               # Справочник товаров
│       ├── Stores.tmdl                 # Справочник магазинов
│       ├── Calendar.tmdl               # Вычисляемый календарь (Time Intelligence)
│       ├── MeasureTable.tmdl           # Таблица мер складских остатков
│       ├── Operations.tmdl             # Операционные расходы магазинов
│       ├── Purchases.tmdl              # Закупки товаров
│       └── dimCustomers.tmdl           # Справочник клиентов
├── 📂 docs/                            # База знаний и проектная документация
│   ├── 📄 GITHUB_SETUP.md              # Инструкция по глобальной авторизации GitHub для всех проектов
│   ├── 📄 Model_Documentation.md       # Полный справочник метаданных и формул DAX
│   ├── 📄 Qlik_to_PowerBI_Guide.md     # Пошаговое руководство по переходу с Qlik Sense на Power BI
│   ├── 📄 architecture.md              # Архитектурное описание схемы «Звезда»
│   └── 📄 implementation_plan.md       # План документирования и аудита модели
└── 📂 Week1 ... Week11/                # Материалы пошаговой разработки курса Power BI
    ├── Исходные данные (Excel, CSV)
    ├── Тематические файлы .pbix по модулям
    └── Методические материалы
```

---

## 🏛️ Архитектура модели (Star Schema)

Модель построена по канонической реляционной схеме «Звезда» без циклических зависимостей:

```mermaid
graph TD
    subgraph Dimensions ["Справочники (Измерения)"]
        D1["dimCustomers (Клиенты)"]
        D2["Products (Товары)"]
        D3["Stores (Магазины)"]
        D4["Calendar (Календарь дат)"]
        D5["dimCurrency (Валюты)"]
    end

    subgraph Facts ["Факты и Операции"]
        F1["Sales (Продажи)"]
        F2["SalesPlan (Планы продаж)"]
        F3["Purchases (Закупки)"]
        F4["Operations (Операции OPEX)"]
        F5["OPEX (Общие расходы)"]
    end

    subgraph Calc ["Вычисления"]
        C1["MeasureTable (Остатки и Накопительные итоги)"]
    end

    D1 -->|1:N| F1
    D2 -->|1:N| F1
    D3 -->|1:N| F1
    D4 -->|1:N (Active)| F1
    D4 -.->|1:N (Inactive: Shipment)| F1
    
    D3 -->|1:N| F2
    D4 -->|1:N| F2

    D2 -->|1:N| F3
    D3 -->|1:N| F3
    D4 -->|1:N| F3

    D3 -->|1:N| F4
    D4 -->|1:N| F4

    D4 -->|1:N| F5
    D5 -->|Dynamic| F1
```

---

## 📊 Ключевые показатели и формулы DAX

Все ключевые меры задокументированы на уровне семантической модели:

| Метрика | Формула DAX | Назначение |
| :--- | :--- | :--- |
| **Выручка ($)** | `SUMX(Sales, Sales[Price $/each] * Sales[Quantity])` | Общий объем фактических продаж |
| **Валовая маржа** | `[4. Sales] - [5.COGS]` | Валовая прибыль (выручка за вычетом себестоимости) |
| **Рентабельность (%)** | `DIVIDE([6.Margin], [4. Sales])` | Процент валовой маржинальности |
| **План-Факт (%)** | `DIVIDE([4. Sales], [1.SalesPlan], 0)` | Процент выполнения установленного плана |
| **Продажи YTD** | `CALCULATE([4. Sales], DATESYTD('Calendar'[Date]))` | Накопительные продажи с начала календарного года |
| **Продажи SPLY** | `CALCULATE([4. Sales], SAMEPERIODLASTYEAR('Calendar'[Date]))` | Сравнение с аналогичным периодом прошлого года |
| **Товарный остаток** | `[Кол-во закуплено накопительно] - [Кол-во продаж накопительно]` | Полуаддитивный расчет остатков товара на складе |
| **Мультивалютность** | `IF(SELECTEDVALUE(dimCurrency[symbol]) = "$", [4. Sales], [41. Sales KZT])` | Динамический пересчет валют по курсу |

---

## 🔄 Мост из Qlik Sense в Power BI

Для специалистов с опытом работы в **Qlik Sense**:
* **Qlik Set Analysis** (`Sum({<Year={2023}>} Sales)`) $\longrightarrow$ **DAX `CALCULATE()`** (`CALCULATE([Sales], Calendar[Year] = 2023)`)
* **Qlik Load Script** $\longrightarrow$ **Power Query (M Engine)**
* **Alternate States & Containers** $\longrightarrow$ **Bookmarks + Selection Pane**
* Подробный сравнительный гайд доступен в файле [docs/Qlik_to_PowerBI_Guide.md](docs/Qlik_to_PowerBI_Guide.md).

---

## 🚀 Как развернуть и использовать

1. **Power BI Desktop:**
   * Откройте файл `reports/Yess-PP.pbix`.
2. **Microsoft Fabric / Git Integration:**
   * Подключите данный репозиторий к Fabric Workspace через встроенный механизм Git Integration. Папка `Yess-PP.SemanticModel` будет автоматически распознана как нативная семантическая модель Fabric.

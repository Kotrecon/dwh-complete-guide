# scd-types-diagram.png

```bash
Тема: Slowly Changing Dimensions - типы SCD

Описание:
Визуальная диаграмма 6 типов медленно меняющихся измерений:

[SCD Type 0 - Retain Original]
Иконка: 🔒 Замок
Описание: "Never changes"
Пример: Original Signup Date

[SCD Type 1 - Overwrite]
Иконка: ✏️ Карандаш
Описание: "Overwrite old values"
Пример: Customer Address

[SCD Type 2 - Add New Row]
Иконка: 📝 Новые строки
Описание: "Add new row for changes"
Пример: Customer History

[SCD Type 3 - Add New Column]
Иконка: 🗂️ Новые колонки
Описание: "Add columns for history"
Пример: Previous Region

[SCD Type 4 - History Table]
Иконка: 📊 Отдельная таблица
Описание: "Separate history table"
Пример: Customer Changes Log

[SCD Type 6 - Hybrid]
Иконка: 🔄 Комбинация
Описание: "Type 1 + 2 + 3 combined"
Пример: Complete Customer Tracking

Стиль: 6-секционная круговая диаграмма или сетка 2x3
Цвета: Разные цвета для каждого типа, иконки для наглядности
```

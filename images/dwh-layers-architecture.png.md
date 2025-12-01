# dwh-layers-architecture.png

```bash
Тема: Слои Data Warehouse - визуальная архитектура

Описание:
Визуализация многослойной архитектуры DWH в виде пирамиды/слоев:

[ВЕРХ] 5. Presentation Layer
┌─────────────────┐
│ BI Tools, Reports│
│ Dashboards, KPIs │
└─────────────────┘

4. Data Marts Layer
   ┌─────────────────┐
   │ Sales Mart │
   │ Marketing Mart │
   │ Finance Mart │
   └─────────────────┘

5. DDS Layer (Core DWH)
   ┌─────────────────┐
   │ Fact Tables │
   │ Dimension Tables│
   │ Star Schema │
   └─────────────────┘

6. ODS Layer
   ┌─────────────────┐
   │ Cleaned Data │
   │ Standardized │
   │ Current State │
   └─────────────────┘

7. Staging Layer
   ┌─────────────────┐
   │ Raw Data │
   │ From Sources │
   │ As-Is Format │
   └─────────────────┘

[НИЗ]
Data Sources → APIs, Databases, Files, Streams

Стиль: Чистая современная инфографика с градиентами
Цвета: Синие тона для данных, зеленые для процессов
```

# etl-vs-elt.png

```bash
Тема: ETL vs ELT - парадигмы обработки данных

Описание:
Два параллельных pipeline с сравнением:

[ETL Pipeline - Classic]
EXTRACT → TRANSFORM → LOAD
   ↓         ↓          ↓
Sources → ETL Server → DWH
   |         |          |
   Raw     Heavy      Clean
  Data   Processing   Data

Characteristics:
• Transformation before load
• ETL server does heavy lifting
• Structured data only
• Better for compliance

[ELT Pipeline - Modern]
EXTRACT → LOAD → TRANSFORM
   ↓         ↓          ↓
Sources → DWH → In-DWH Processing
   |         |          |
   Raw     Fast       Transform
  Data     Load      in Database

Characteristics:
• Load raw data first
• DWH does transformation
• Handles all data types
• More flexible & scalable

[Hybrid Approach]
ETL-T: Light transform → Load → Heavy transform

Стиль: Техническая диаграмма с иконками баз данных и процессоров
Цвета: Оранжевый для ETL, Синий для ELT, Фиолетовый для гибрида
```

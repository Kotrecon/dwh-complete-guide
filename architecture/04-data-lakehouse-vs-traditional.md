# 1.4 - Data Lakehouse vs Traditional DWH: Сравнение архитектур

## Введение

Data Lakehouse - это современная архитектура, объединяющая лучшие черты Data Lake и Data Warehouse. Рассмотрим ключевые различия и сценарии применения.

## 📊 Сравнительная таблица архитектур

| Аспект                 | Traditional DWH   | Data Lake      | Data Lakehouse         |
| ---------------------- | ----------------- | -------------- | ---------------------- |
| **Тип данных**         | Структурированные | Все типы       | Все типы               |
| **Схема**              | Schema-on-Write   | Schema-on-Read | Schema-on-Read & Write |
| **Производительность** | 🟢 Высокая (SQL)  | 🔴 Низкая      | 🟢 Высокая (SQL+)      |
| **Гибкость**           | 🔴 Низкая         | 🟢 Высокая     | 🟢 Высокая             |
| **Стоимость**          | 🔴 Высокая        | 🟢 Низкая      | 🟡 Средняя             |
| **ACID транзакции**    | 🟢 Полные         | 🔴 Отсутствуют | 🟢 Полные              |
| **ML поддержка**       | 🔴 Ограниченная   | 🟢 Полная      | 🟢 Полная              |

---

## 🏛️ **Traditional Data Warehouse**

### **Архитектура**

```bash
Источники → ETL → DWH (Structured) → BI Tools
    ↓
Жесткая схема, оптимизация под SQL
```

### **Технологический стек**

- **Cloud:** Snowflake, Redshift, BigQuery, Synapse
- **On-Prem:** Teradata, Oracle Exadata, Vertica
- **ETL:** Informatica, Talend, SSIS

### **Преимущества**

```sql
-- Оптимизирован для SQL-аналитики
SELECT
    customer_segment,
    AVG(order_value) as avg_order,
    COUNT(*) as order_count
FROM fact_orders fo
JOIN dim_customers dc ON fo.customer_id = dc.customer_id
WHERE order_date >= '2024-01-01'
GROUP BY customer_segment
ORDER BY avg_order DESC;
```

### **Ограничения**

- ❌ Только структурированные данные
- ❌ Высокая стоимость хранения
- ❌ Сложность изменения схемы
- ❌ Ограниченная поддержка ML/AI

---

## 🏞️ **Data Lake**

### **Архитектура**

```bash
Все данные → Data Lake (Raw) → Processing → Analytics
    ↓
Гибкая схема, поддержка любых форматов
```

### **Технологический стек**

- **Storage:** AWS S3, Azure Blob, Google Cloud Storage
- **Processing:** Spark, Hadoop, Databricks
- **Format:** Parquet, AVRO, JSON, ORC

### **Преимущества**

```csharp
// ML Pipeline
var rawImages = spark.Read().Format("binaryFile").Load("s3://lake/raw/images/");
var processedData = mlPipeline.Transform(rawImages);

// Log Analysis
var logsDf = spark.Read().Json("s3://lake/raw/logs/");
var analytics = logsDf.Filter("level = 'ERROR'").GroupBy("service").Count();
```

### **Ограничения**

- ❌ Нет ACID транзакций
- ❌ Слабая производительность для BI
- ❌ Сложность управления качеством данных
- ❌ Отсутствие единого метаданного

---

## 🏠 **Data Lakehouse**

### **Архитектура**

```bash
Все данные → Lakehouse (Open Format) → BI & AI
    ↓
Объединяет гибкость Lake и производительность DWH
```

### **Ключевые компоненты**

```sql
Storage Layer (Object Store)
├── Delta Lake / Iceberg / Hudi
├── ACID Transactions
└── Versioning

Metadata Layer
├── Unified Catalog
├── Schema Evolution
└── Data Governance

Compute Layer
├── SQL Analytics Engine
├── ML Frameworks
└── Streaming Processing
```

### **Технологический стек**

- **Formats:** Delta Lake, Apache Iceberg, Apache Hudi
- **Platforms:** Databricks Lakehouse, Snowflake, BigLake
- **Tools:** Spark, dbt, MLflow, Presto

### **Преимущества**

```sql
-- SQL аналитика поверх всех данных
-- Структурированные данные
SELECT user_id, SUM(amount) as total_spent
FROM delta.`s3://lakehouse/sales/`
WHERE transaction_date > '2024-01-01'
GROUP BY user_id;

-- Полуструктурированные данные
SELECT JSON_EXTRACT(log_data, '$.user_id') as user_id,
       COUNT(*) as error_count
FROM delta.`s3://lakehouse/logs/`
WHERE JSON_EXTRACT(log_data, '$.level') = 'ERROR'
GROUP BY user_id;
```

```csharp
// ML поверх тех же данных
using Microsoft.Spark.ML;
using Delta.Tables;

// Чтение данных для ML
var trainingData = spark.Read().Format("delta").Load("s3://lakehouse/sales/");
var mlModel = pipeline.Fit(trainingData);

// Запись результатов обратно
predictions.Write().Format("delta").Save("s3://lakehouse/predictions/");
```

---

## 🎯 **Критерии выбора архитектуры**

### **Выбирайте TRADITIONAL DWH если:**

- ✅ Зрелые бизнес-процессы со стабильными схемами
- ✅ Преимущественно SQL-аналитика
- ✅ Строгие требования к compliance и governance
- ✅ Ограниченная потребность в ML/AI

### **Выбирайте DATA LAKE если:**

- ✅ Разнообразные типы данных (images, logs, JSON)
- ✅ Экспериментальная аналитика и research
- ✅ Ограниченный бюджет
- ✅ Сильная команда data engineers

### **Выбирайте DATA LAKEHOUSE если:**

- ✅ Необходимость объединить BI и AI
- ✅ Быстро меняющиеся бизнес-требования
- ✅ Разнообразные workload-ы (SQL, ML, Streaming)
- ✅ Требования к сквозному data governance

---

## 🔧 **Миграционные сценарии**

### **DWH → Lakehouse (Плавная миграция)**

```sql
-- Фаза 1: Data Mirroring
COPY fact_sales TO 's3://lakehouse/mirror/sales/'
FORMAT PARQUET;

-- Фаза 2: Hybrid Querying
-- Запросы работают с обоими источниками
CREATE VIEW unified_sales AS
SELECT * FROM legacy_dwh.fact_sales
UNION ALL
SELECT * FROM delta.`s3://lakehouse/mirror/sales/`;

-- Фаза 3: Cut-over
-- Переключение workload-ов на Lakehouse
```

### **Data Lake → Lakehouse (Эволюция)**

```csharp
// Конвертация raw data в Delta/Iceberg
// До: Raw files
var rawDf = spark.Read().Parquet("s3://datalake/raw/sales/");

// После: Delta Lake with ACID
rawDf.Write().Format("delta")
    .Option("delta.autoOptimize.optimizeWrite", "true")
    .Save("s3://lakehouse/sales/");
```

---

## 💰 **Сравнение TCO (Total Cost of Ownership)**

| Компонент      | Traditional DWH | Data Lake | Data Lakehouse |
| -------------- | --------------- | --------- | -------------- |
| **Storage**    | $$$$            | $         | $$             |
| **Compute**    | $$$$            | $$        | $$$            |
| **ETL/ELT**    | $$$             | $$        | $$             |
| **ML/AI**      | $$$$            | $$        | $$             |
| **Management** | $$$             | $$        | $              |
| **Итого TCO**  | **$$$$**        | **$$**    | **$$$**        |

---

## 🏢 **Реальные кейсы применения**

### **Financial Services - Lakehouse**

```sql
-- Объединение транзакционных данных и ML-моделей
-- Транзакции (структурированные)
SELECT account_id, transaction_amount,
       fraud_prediction_score
FROM delta.`s3://lakehouse/transactions/`
WHERE fraud_prediction_score > 0.8;

-- Логи и поведенческие данные (полуструктурированные)
SELECT user_id, navigation_pattern,
       risk_assessment_model(behavior_data) as risk_score
FROM delta.`s3://lakehouse/user_behavior/`
```

### **E-commerce - Hybrid Approach**

```yaml
Architecture:
  Real-time Analytics: Traditional DWH (Snowflake)
  Customer Analytics: Lakehouse (Databricks)
  ML Recommendations: Lakehouse ML Runtime
  Log Analysis: Data Lake (S3 + Spark)
```

### **Healthcare - Data Lakehouse**

```csharp
// Объединение structured и unstructured данных
// EHR данные
var ehrData = spark.Read().Table("delta.`s3://lakehouse/ehr/`");

// Медицинские изображения
var imagesDf = spark.Read().Format("binaryFile")
    .Load("s3://lakehouse/medical_images/");

// ML pipeline для диагностики
var diagnosisModel = healthcareMlPipeline.Fit(ehrData);
var predictions = diagnosisModel.Transform(imagesDf);
```

---

## 🚀 **Производительность: Benchmark**

| Workload             | Traditional DWH | Data Lake | Data Lakehouse |
| -------------------- | --------------- | --------- | -------------- |
| **SQL Analytics**    | 100%            | 40%       | 90%            |
| **ML Training**      | 30%             | 100%      | 95%            |
| **Data Ingestion**   | 70%             | 100%      | 90%            |
| **Concurrent Users** | 80%             | 50%       | 85%            |

---

## 📈 **Тренды и будущее**

### **Конвергенция технологий**

- Snowflake → Snowpark (ML в DWH)
- Databricks → SQL Analytics в Lakehouse
- BigQuery → BigQuery ML + BigLake

### **Emerging Standards**

- **Delta Sharing**: Open data sharing protocol
- **Open Table Formats**: Delta, Iceberg, Hudi
- **Data Mesh**: Decentralized data ownership

---

## 🚨 **Anti-patterns**

1. **Lift-and-shift** DWH в Lakehouse без перепроектирования
2. **Data Swamp** в Lakehouse без proper governance
3. **Over-engineering** для простых use cases
4. **Ignoring skillset** команды при выборе архитектуры

---

## ✅ **Best Practices**

1. **Start with business requirements** а не с технологий
2. **Consider hybrid approach** для плавной миграции
3. **Implement data governance** с самого начала
4. **Choose open formats** для избежания vendor lock-in
5. **Plan for evolution** архитектуры со временем

---

**Следующий раздел:** [2.1 - ETL vs ELT: парадигмы загрузки данных](../processes/01-etl-vs-elt-paradigms.md)

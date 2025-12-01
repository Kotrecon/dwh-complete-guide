# 2.1 - ETL vs ELT: парадигмы загрузки данных

## Введение

ETL (Extract, Transform, Load) и ELT (Extract, Load, Transform) - две основные парадигмы загрузки данных в DWH. Выбор между ними определяет архитектуру, производительность и гибкость data pipeline.

## 📊 Сравнительная таблица ETL vs ELT

| Аспект                          | ETL                        | ELT                        |
| ------------------------------- | -------------------------- | -------------------------- |
| **Порядок операций**            | Extract → Transform → Load | Extract → Load → Transform |
| **Место трансформации**         | ETL-сервер                 | Целевая DWH                |
| **Производительность**          | 🟡 Средняя                 | 🟢 Высокая                 |
| **Гибкость**                    | 🔴 Низкая                  | 🟢 Высокая                 |
| **Сложность**                   | 🟡 Средняя                 | 🟢 Низкая                  |
| **Стоимость**                   | 🔴 Высокая                 | 🟡 Средняя                 |
| **Поддержка реального времени** | 🔴 Ограниченная            | 🟢 Хорошая                 |

---

## 🔄 **ETL (Extract, Transform, Load)**

### **Архитектура**

```bash
Источники → ETL Server (Transform) → DWH (Load)
    ↓
Трансформации выполняются до загрузки
```

### **Технологический стек**

- **Tools:** Informatica, Talend, SSIS, Azure Data Factory
- **Pattern:** Batch-oriented processing
- **Use Cases:** Legacy systems, strict compliance

### **Преимущества**

```csharp
// Пример ETL pipeline на C#
public class EtlPipeline
{
    public void Execute()
    {
        // EXTRACT - чтение из источников
        var sourceData = ExtractFromSources();

        // TRANSFORM - трансформации в памяти
        var transformedData = TransformData(sourceData);

        // LOAD - загрузка в DWH
        LoadToDataWarehouse(transformedData);
    }

    private List<RawRecord> ExtractFromSources()
    {
        // Чтение из SQL, API, файлов
        var sqlData = sqlRepository.GetSalesData();
        var apiData = apiClient.GetCustomerData();
        return MergeData(sqlData, apiData);
    }

    private List<TransformedRecord> TransformData(List<RawRecord> rawData)
    {
        // Все трансформации выполняются до загрузки
        return rawData.Select(record => new TransformedRecord
        {
            CustomerId = record.CustID,
            SalesAmount = record.Amount * GetExchangeRate(record.Currency),
            CleanedProductName = CleanProductName(record.Product),
            ValidatedDate = ValidateDate(record.TransactionDate)
        }).ToList();
    }
}
```

### **Ограничения**

- ❌ Ограниченная вычислительная мощность ETL-сервера
- ❌ Сложность изменения трансформаций
- ❌ Задержки при больших объемах данных

---

## ⚡ **ELT (Extract, Load, Transform)**

### **Архитектура**

```bash
Источники → DWH (Load) → In-DWH Transform
    ↓
Трансформации выполняются в целевой DWH
```

### **Технологический стек**

- **Cloud DWH:** Snowflake, BigQuery, Redshift, Synapse
- **Transformation:** dbt, SQL, Spark
- **Orchestration:** Airflow, Dagster, Prefect

### **Преимущества**

```csharp
// Пример ELT pipeline на C#
public class EltPipeline
{
    private readonly IDataWarehouseService _dwService;

    public async Task ExecuteAsync()
    {
        // EXTRACT & LOAD - быстрая загрузка сырых данных
        await LoadRawDataToStaging();

        // TRANSFORM - трансформации в DWH через SQL
        await ExecuteTransformationsInDwh();
    }

    private async Task LoadRawDataToStaging()
    {
        // Быстрая загрузка без трансформаций
        var salesData = await sqlRepository.GetSalesDataAsync();
        var customerData = await apiClient.GetCustomerDataAsync();

        // Параллельная загрузка в staging слой
        var tasks = new List<Task>
        {
            _dwService.BulkInsertAsync("stg_sales", salesData),
            _dwService.BulkInsertAsync("stg_customers", customerData)
        };

        await Task.WhenAll(tasks);
    }

    private async Task ExecuteTransformationsInDwh()
    {
        // Все трансформации выполняются в DWH
        var transformationSql = @"
            INSERT INTO dim_customers
            SELECT
                CustomerId,
                UPPER(CustomerName) as CustomerName,
                CASE
                    WHEN Region IS NULL THEN 'Unknown'
                    ELSE Region
                END as CleanRegion
            FROM stg_customers;

            INSERT INTO fact_sales
            SELECT
                SalesId,
                CustomerId,
                ProductId,
                Amount * ExchangeRate as SalesAmountUSD
            FROM stg_sales s
            JOIN exchange_rates e ON s.Currency = e.CurrencyCode;
        ";

        await _dwService.ExecuteSqlAsync(transformationSql);
    }
}
```

### **Особенности**

- ✅ Использует вычислительную мощность DWH
- ✅ Гибкость изменения трансформаций
- ✅ Поддержка инкрементальных обновлений

---

## 🎯 **Критерии выбора парадигмы**

### **Выбирайте ETL если:**

- ✅ Строгие требования к качеству данных перед загрузкой
- ✅ Ограниченные вычислительные ресурсы DWH
- ✅ Сложные трансформации, требующие кастомной логики
- ✅ Регуляторные требования (GDPR, HIPAA)

### **Выбирайте ELT если:**

- ✅ Используется современный Cloud DWH (Snowflake, BigQuery)
- ✅ Требуется гибкость и скорость изменений
- ✅ Большие объемы данных
- ✅ Необходимость ad-hoc анализа сырых данных

---

## 🔧 **Гибридные подходы**

### **ETL-T (Extract, Transform lightly, Load, Transform heavily)**

```csharp
public class HybridPipeline
{
    public async Task ExecuteAsync()
    {
        // Легкие трансформации перед загрузкой
        var lightlyTransformed = ApplyLightTransformations(extractedData);

        // Загрузка в DWH
        await LoadToStaging(lightlyTransformed);

        // Тяжелые трансформации в DWH
        await ExecuteHeavyTransformationsInDwh();
    }

    private List<DataRecord> ApplyLightTransformations(List<RawRecord> rawData)
    {
        // Только критически важные трансформации
        return rawData.Select(record => new DataRecord
        {
            // Валидация обязательных полей
            CustomerId = ValidateRequired(record.CustomerId),
            // Базовая очистка
            ProductName = record.ProductName?.Trim(),
            // Конвертация форматов
            TransactionDate = ParseDate(record.TransactionDateString)
        }).Where(x => x.IsValid).ToList();
    }
}
```

---

## 📊 **Производительность: Сравнение**

| Метрика                       | ETL        | ELT        |
| ----------------------------- | ---------- | ---------- |
| **Время загрузки 1GB данных** | 15-30 мин  | 5-10 мин   |
| **Время разработки pipeline** | 2-4 недели | 1-2 недели |
| **Гибкость изменений**        | Низкая     | Высокая    |
| **Стоимость инфраструктуры**  | Высокая    | Средняя    |

---

## 🚀 **Современные тенденции**

### **Reverse ETL**

```csharp
// Данные из DWH обратно в операционные системы
public class ReverseEtlService
{
    public async Task SyncToCrmAsync()
    {
        // Извлечение обогащенных данных из DWH
        var customerSegments = await _dwService.QueryAsync(@"
            SELECT
                CustomerId,
                CustomerName,
                CASE
                    WHEN TotalSpent > 1000 THEN 'VIP'
                    WHEN TotalSpent > 500 THEN 'Premium'
                    ELSE 'Standard'
                END as Segment
            FROM customer_metrics
        ");

        // Синхронизация в CRM
        await _crmService.UpdateCustomerSegments(customerSegments);
    }
}
```

### **Data Mesh и ELT**

- Децентрализованная обработка данных
- Domain-oriented pipelines
- Self-serve data platforms

---

## 🚨 **Anti-patterns**

1. **ETL для Cloud DWH** - неиспользование преимуществ платформы
2. **ELT без governance** - создание data swamp
3. **Over-engineering** простых pipelines
4. **Ignoring data quality** в ELT подходах

---

## ✅ **Best Practices**

### **Для ETL:**

- Используйте инкрементальную загрузку где возможно
- Реализуйте надежную обработку ошибок
- Мониторьте производительность ETL-серверов

### **Для ELT:**

- Используйте возможности массовой загрузки DWH
- Реализуйте версионирование SQL-трансформаций
- Мониторьте стоимость запросов в cloud DWH

### **Универсальные:**

- Документируйте data lineage
- Реализуйте мониторинг качества данных
- Планируйте миграцию с ETL на ELT для cloud-native проектов

---

**Следующий раздел:** [2.2 - Медленно меняющиеся измерения (SCD Types I-VI)](./02-scd-types-i-vi.md)

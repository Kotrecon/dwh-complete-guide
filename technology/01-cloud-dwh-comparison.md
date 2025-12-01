# 3.1 - Облачные DWH: BigQuery, Snowflake, Redshift, Synapse

## Введение

Облачные Data Warehouse революционизировали подход к хранению и анализу данных, предлагая масштабируемость, эластичность и pay-as-you-go модели. Рассмотрим четыре ведущие платформы.

## 📊 Сравнительная таблица облачных DWH

| Аспект                 | BigQuery          | Snowflake     | Redshift      | Synapse Analytics |
| ---------------------- | ----------------- | ------------- | ------------- | ----------------- |
| **Провайдер**          | Google Cloud      | Multi-Cloud   | AWS           | Microsoft Azure   |
| **Архитектура**        | Serverless        | Hybrid        | Cluster-based | Hybrid            |
| **Модель стоимости**   | Pay-per-query     | Credit-based  | Cluster-hours | Mixed             |
| **Производительность** | 🟢 Высокая        | 🟢 Высокая    | 🟡 Средняя    | 🟡 Средняя        |
| **Масштабируемость**   | 🟢 Автоматическая | 🟢 Эластичная | 🟡 Ручная     | 🟡 Полу-авто      |
| **Интеграция ML**      | 🟢 Native         | 🟡 Good       | 🔴 Basic      | 🟢 Native         |

---

## ☁️ **Google BigQuery**

### **Архитектура**

```sql
Serverless Architecture
├── Compute: Dremel Engine
├── Storage: Colossus (Columnar)
└── Separation: Compute & Storage
```

### **Ключевые особенности**

- Полностью serverless - нет управления инфраструктурой
- Автоматическое масштабирование
- Native ML интеграция с BigQuery ML
- Геораспределение по умолчанию

### **Пример работы с C#**

```csharp
public class BigQueryService
{
    private readonly BigQueryClient _client;

    public async Task<List<SalesRecord>> ExecuteSalesQueryAsync()
    {
        var sql = @"
            SELECT
                customer_id,
                SUM(sales_amount) as total_sales,
                COUNT(*) as transaction_count
            FROM `my-project.sales.fact_sales`
            WHERE transaction_date >= @startDate
            GROUP BY customer_id
            ORDER BY total_sales DESC
            LIMIT 100";

        var parameters = new[]
        {
            new BigQueryParameter("startDate", BigQueryDbType.Date, DateTime.UtcNow.AddDays(-30))
        };

        var results = await _client.ExecuteQueryAsync(sql, parameters);

        return results.Select(row => new SalesRecord
        {
            CustomerId = row["customer_id"].ToString(),
            TotalSales = Convert.ToDecimal(row["total_sales"]),
            TransactionCount = Convert.ToInt32(row["transaction_count"])
        }).ToList();
    }

    public async Task CreateMachineLearningModelAsync()
    {
        var createModelSql = @"
            CREATE OR REPLACE MODEL `my-project.sales.customer_segment`
            OPTIONS(
                model_type = 'kmeans',
                num_clusters = 4,
                standardize_features = true
            ) AS
            SELECT
                total_sales,
                transaction_count,
                avg_transaction_value
            FROM `my-project.sales.customer_metrics`";

        await _client.ExecuteQueryAsync(createModelSql);
    }
}
```

### **Стоимость и производительность**

```csharp
public class BigQueryCostCalculator
{
    public async Task<QueryCost> CalculateQueryCostAsync(string sqlQuery)
    {
        // BigQuery charges by bytes processed
        var job = await _client.CreateQueryJobAsync(sqlQuery);
        var bytesProcessed = job.Statistics.TotalBytesProcessed;

        // $5 per TB (as of 2024)
        var cost = (bytesProcessed / (1024m * 1024 * 1024 * 1024)) * 5;

        return new QueryCost
        {
            BytesProcessed = bytesProcessed,
            EstimatedCost = cost,
            IsSlotOptimized = await CheckSlotOptimizationAsync(sqlQuery)
        };
    }
}
```

---

## ❄️ **Snowflake**

### **Архитектура**

```sql
Multi-cluster Architecture
├── Storage: Cloud Storage
├── Compute: Virtual Warehouses
└── Services: Cloud Services
```

### **Ключевые особенности**

- Гибридная архитектура (compute/storage separation)
- Multi-cloud поддержка (AWS, Azure, GCP)
- Time Travel и Zero-copy cloning
- Secure Data Sharing

### **Пример работы с C#**

```csharp
public class SnowflakeService
{
    private readonly SnowflakeDbConnection _connection;

    public async Task<DataTable> ExecuteWarehouseQueryAsync(string warehouse)
    {
        // Использование конкретного виртуального warehouse
        var useWarehouseSql = $"USE WAREHOUSE {warehouse}";
        await ExecuteNonQueryAsync(useWarehouseSql);

        var salesSql = @"
            SELECT
                c.customer_name,
                SUM(s.sales_amount) as lifetime_value,
                COUNT(*) as order_count
            FROM sales_fact s
            JOIN customer_dim c ON s.customer_key = c.customer_key
            GROUP BY c.customer_name
            HAVING SUM(s.sales_amount) > 1000";

        return await ExecuteQueryAsync(salesSql);
    }

    public async Task CreateZeroCopyCloneAsync()
    {
        // Zero-copy cloning для тестирования
        var cloneSql = @"
            CREATE OR REPLACE TABLE sales_fact_dev
            CLONE sales_fact_prod
            AT (TIMESTAMP => '2024-01-01 00:00:00'::TIMESTAMP)";

        await ExecuteNonQueryAsync(cloneSql);
    }

    public async Task EnableTimeTravelAsync()
    {
        // Настройка Time Travel retention
        var alterTableSql = @"
            ALTER TABLE sales_fact
            SET DATA_RETENTION_TIME_IN_DAYS = 90";

        await ExecuteNonQueryAsync(alterTableSql);
    }
}
```

### **Управление виртуальными warehouses**

```csharp
public class WarehouseManager
{
    public async Task ScaleWarehouseAsync(string warehouseName, string size)
    {
        var scaleSql = $"ALTER WAREHOUSE {warehouseName} SET WAREHOUSE_SIZE = {size}";
        await ExecuteNonQueryAsync(scaleSql);
    }

    public async Task<WarehouseUsage> GetWarehouseUsageAsync(string warehouseName)
    {
        var usageSql = @"
            SELECT
                warehouse_name,
                credits_used,
                bytes_scanned,
                query_count
            FROM snowflake.account_usage.warehouse_metering_history
            WHERE warehouse_name = @warehouseName
            AND start_time >= DATEADD(day, -7, CURRENT_DATE())";

        return await ExecuteQueryAsync<WarehouseUsage>(usageSql,
            new { warehouseName });
    }
}
```

---

## 🔴 **Amazon Redshift**

### **Архитектура**

```sql
Cluster-based Architecture
├── Leader Node: Query coordination
├── Compute Nodes: Data processing
└️── Storage: Local SSDs + S3
```

### **Ключевые особенности**

- Columnar storage на локальных SSD
- Massively Parallel Processing (MPP)
- Deep integration с AWS ecosystem
- Redshift Spectrum для querying S3 data

### **Пример работы с C#**

```csharp
public class RedshiftService
{
    private readonly NpgsqlConnection _connection;

    public async Task<bool> OptimizeTableDistributionAsync(string tableName)
    {
        // Redshift требует правильного распределения данных
        var distributionAnalysis = await AnalyzeTableDistributionAsync(tableName);

        if (distributionAnalysis.NeedsRedistribution)
        {
            var redistributeSql = $@"
                CREATE TABLE {tableName}_redistributed
                DISTKEY({distributionAnalysis.DistributionKey})
                SORTKEY({distributionAnalysis.SortKey})
                AS SELECT * FROM {tableName}";

            await ExecuteNonQueryAsync(redistributeSql);
            return true;
        }

        return false;
    }

    public async Task<List<QueryPerformance>> GetSlowQueriesAsync()
    {
        var performanceSql = @"
            SELECT
                query_text,
                execution_time,
                rows_returned,
                disk_usage
            FROM svl_query_summary
            WHERE execution_time > 10000  -- 10+ seconds
            ORDER BY execution_time DESC
            LIMIT 50";

        return await ExecuteQueryAsync<List<QueryPerformance>>(performanceSql);
    }

    public async Task UseRedshiftSpectrumAsync()
    {
        // Querying data directly from S3 using Spectrum
        var spectrumQuery = @"
            SELECT
                customer_id,
                SUM(amount) as total
            FROM spectrum_sales_data
            WHERE transaction_date >= '2024-01-01'
            GROUP BY customer_id";

        return await ExecuteQueryAsync(spectrumQuery);
    }
}
```

### **Управление кластером**

```csharp
public class RedshiftClusterManager
{
    private readonly IAmazonRedshift _redshiftClient;

    public async Task ResizeClusterAsync(string clusterIdentifier, string nodeType, int nodeCount)
    {
        var request = new ModifyClusterRequest
        {
            ClusterIdentifier = clusterIdentifier,
            NodeType = nodeType,
            NumberOfNodes = nodeCount,
            ClusterType = "multi-node"
        };

        await _redshiftClient.ModifyClusterAsync(request);

        // Мониторинг процесса resize
        await WaitForClusterAvailableAsync(clusterIdentifier);
    }

    public async Task EnableConcurrencyScalingAsync(string clusterIdentifier)
    {
        var request = new ModifyClusterRequest
        {
            ClusterIdentifier = clusterIdentifier,
            ConcurrencyScalingMode = ConcurrencyScalingMode.auto
        };

        await _redshiftClient.ModifyClusterAsync(request);
    }
}
```

---

## 🟦 **Azure Synapse Analytics**

### **Архитектура**

```sql
Hybrid Architecture
├── Dedicated SQL Pools: MPP
├── Serverless SQL Pools: On-demand
└── Spark Pools: Big Data processing
```

### **Ключевые особенности**

- Unified analytics platform
- Deep integration с Azure ecosystem
- Hybrid execution (SQL + Spark)
- Security и compliance features

### **Пример работы с C#**

```csharp
public class SynapseService
{
    private readonly SqlConnection _sqlConnection;
    private readonly SparkSession _sparkSession;

    public async Task<DataTable> ExecuteDedicatedSqlQueryAsync()
    {
        // Dedicated SQL Pool - MPP queries
        var sql = @"
            SELECT
                c.[CustomerName],
                SUM(s.[SalesAmount]) as TotalSales,
                COUNT_BIG(*) as TransactionCount
            FROM [sales].[FactSales] s
            JOIN [dim].[Customer] c ON s.[CustomerKey] = c.[CustomerKey]
            GROUP BY c.[CustomerName]
            OPTION (LABEL = 'CustomerSalesAnalysis')";

        return await ExecuteSqlQueryAsync(sql);
    }

    public async Task<DataFrame> ExecuteSparkQueryAsync()
    {
        // Spark Pool - big data processing
        var df = _sparkSession
            .Read()
            .Table("sales.FactSales")
            .GroupBy("CustomerKey")
            .Agg(
                Functions.Sum("SalesAmount").Alias("TotalSales"),
                Functions.Count("*").Alias("TransactionCount")
            );

        return df;
    }

    public async Task UseServerlessSqlAsync()
    {
        // Serverless SQL Pool - query external data
        var externalQuery = @"
            SELECT
                customer_id,
                COUNT(*) as file_count
            FROM OPENROWSET(
                BULK 'https://storageaccount.blob.core.windows.net/container/*.parquet',
                FORMAT = 'PARQUET'
            ) AS [result]
            GROUP BY customer_id";

        return await ExecuteSqlQueryAsync(externalQuery);
    }
}
```

### **Управление ресурсами**

```csharp
public class SynapseResourceManager
{
    public async Task ScaleDedicatedPoolAsync(string workspaceName, string poolName, int dwu)
    {
        // Scaling Dedicated SQL Pool
        var sql = $@"
            ALTER DATABASE {poolName}
            MODIFY (SERVICE_OBJECTIVE = 'DW{dwu}c')";

        await ExecuteSqlQueryAsync(sql);
    }

    public async Task PauseResumePoolAsync(string poolName, bool pause)
    {
        var sql = pause
            ? $"ALTER DATABASE {poolName} PAUSE"
            : $"ALTER DATABASE {poolName} RESUME";

        await ExecuteSqlQueryAsync(sql);
    }
}
```

## 🎯 **Критерии выбора облачного DWH**

### **Выбирайте BigQuery если:**

- ✅ Serverless архитектура в приоритете
- ✅ Глубокая интеграция с Google Cloud ecosystem
- ✅ Advanced ML requirements
- ✅ Глобальная аналитика с low latency

### **Выбирайте Snowflake если:**

- ✅ Multi-cloud стратегия
- ✅ Требуется гибкое масштабирование compute/storage
- ✅ Data sharing между организациями
- ✅ Time Travel и cloning capabilities

### **Выбирайте Redshift если:**

- ✅ Глубокая интеграция с AWS services
- ✅ Предсказуемая производительность на больших workload-ах
- ✅ Требуется MPP архитектура с локальным storage
- ✅ Существующие инвестиции в AWS

### **Выбирайте Synapse если:**

- ✅ Unified analytics platform (SQL + Spark)
- ✅ Глубокая интеграция с Microsoft ecosystem
- ✅ Power BI как основной BI инструмент
- ✅ Enterprise security и compliance требования

---

## 💰 **Сравнение стоимости**

| Платформа     | Модель ценообразования | Пример стоимости (1TB данных)           |
| ------------- | ---------------------- | --------------------------------------- |
| **BigQuery**  | Pay-per-query          | ~$5 за TB обработанных данных           |
| **Snowflake** | Credits-based          | ~$40-80/credit + storage $23/TB/мес     |
| **Redshift**  | Cluster-hours          | ~$0.25-4.80/час + storage $0.024/GB/мес |
| **Synapse**   | Mixed model            | ~$1.20-360/час + storage $122.88/TB/мес |

---

## 🚀 **Миграционные стратегии**

### **On-premise → Cloud DWH**

```csharp
public class MigrationService
{
    public async Task MigrateToCloudAsync(string sourceDb, CloudDwhTarget target)
    {
        // 1. Schema migration
        await MigrateSchemaAsync(sourceDb, target);

        // 2. Data migration (incremental)
        await MigrateDataIncrementalAsync(sourceDb, target);

        // 3. ETL pipeline migration
        await MigrateEtlPipelinesAsync(sourceDb, target);

        // 4. Validation and cutover
        await ValidateMigrationAsync(sourceDb, target);
    }
}
```

---

## 📊 **Performance Benchmark**

| Workload                 | BigQuery | Snowflake | Redshift | Synapse |
| ------------------------ | -------- | --------- | -------- | ------- |
| **Star Schema Query**    | 12.3s    | 8.7s      | 15.2s    | 18.9s   |
| **Complex Aggregations** | 45.1s    | 32.8s     | 28.9s    | 51.2s   |
| **Data Loading (1GB)**   | 28s      | 35s       | 42s      | 38s     |
| **Concurrent Users**     | 100+     | 50+       | 30+      | 40+     |

---

## 🚨 **Anti-patterns**

1. **Выбор по цене без учета TCO** - скрытые costs
2. **Игнорирование skillset команды**
3. **Vendor lock-in без стратегии выхода**
4. **Over-provisioning ресурсов**

---

## ✅ **Best Practices**

### **Для всех платформ:**

- Реализуйте data partitioning и clustering
- Используйте appropriate file formats (Parquet, ORC)
- Мониторьте стоимость и performance
- Реализуйте proper security и governance

### **Оптимизация стоимости:**

- Используйте auto-scaling где возможно
- Реализуйте data lifecycle management
- Мониторьте и оптимизируйте query patterns
- Используйте spot instances для non-critical workloads

---

**Следующий раздел:** [3.2 - Инструменты ETL/ELT: dbt, Airflow, Informatica](./02-etl-elt-tools-overview.md)

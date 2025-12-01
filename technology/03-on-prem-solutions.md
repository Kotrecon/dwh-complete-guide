# 3.3 - Он-премис решения: Teradata, Exadata, ClickHouse

## Введение

Несмотря на рост облачных решений, on-premise Data Warehouse продолжают играть важную роль в enterprise-окружениях, особенно в регулируемых отраслях и с требованиями низкой latency.

## 📊 Сравнительная таблица on-premise DWH

| Решение            | Производитель | Архитектура | Лучшее для           | Стоимость  |
| ------------------ | ------------- | ----------- | -------------------- | ---------- |
| **Teradata**       | Teradata      | MPP         | Enterprise analytics | 🔴 Высокая |
| **Oracle Exadata** | Oracle        | Scale-out   | OLTP + OLAP 混合     | 🔴 Высокая |
| **ClickHouse**     | Yandex        | Columnar    | Real-time analytics  | 🟢 Низкая  |
| **Greenplum**      | VMware        | MPP         | Big data analytics   | 🟡 Средняя |
| **Vertica**        | Micro Focus   | Columnar    | High-performance     | 🟡 Средняя |

---

## 🏢 **Teradata**

### **Архитектура**

```bash
Teradata Architecture
├── Parsing Engine: Query optimization
├️── AMPs (Access Module Processors): Parallel processing
├── BYNET: Inter-process communication
└️── Storage: Shared-nothing architecture
```

### **Ключевые особенности**

- Massively Parallel Processing (MPP)
- Linear scalability
- Advanced workload management
- Enterprise-grade security

### **Пример работы с C#**

```csharp
public class TeradataService
{
    private readonly TeradataConnection _connection;

    public async Task<DataTable> ExecuteParallelQueryAsync()
    {
        // Teradata оптимизирован для параллельных запросов
        var sql = @"
            SELECT
                customer_id,
                SUM(sales_amount) as total_sales,
                COUNT(*) as transaction_count
            FROM sales_fact
            WHERE transaction_date >= DATE - 30
            GROUP BY customer_id
            QUALIFY ROW_NUMBER() OVER (ORDER BY total_sales DESC) <= 100";

        using var command = new TeradataCommand(sql, _connection);
        using var adapter = new TeradataDataAdapter(command);

        var result = new DataTable();
        await Task.Run(() => adapter.Fill(result));

        return result;
    }

    public async Task ManageWorkloadsAsync()
    {
        // Teradata Workload Management
        var wlmSql = @"
            CREATE WORKLOAD CustomerAnalytics
            AS WORKLOADCLASS 'HighPriority'
            WITH WEIGHT 80;

            CREATE WORKLOAD OperationalReports
            AS WORKLOADCLASS 'Standard'
            WITH WEIGHT 20;";

        await ExecuteNonQueryAsync(wlmSql);
    }

    public async Task<string> GetExplainPlanAsync(string query)
    {
        // Анализ плана выполнения
        var explainSql = $"EXPLAIN {query}";
        return await ExecuteScalarAsync<string>(explainSql);
    }
}

public class TeradataPerformanceMonitor
{
    public async Task<PerformanceMetrics> GetSystemMetricsAsync()
    {
        var metricsSql = @"
            SELECT
                AMPCount,
                CurrentPeakAMPCPUTime,
                TotalCPUUtilization,
                DiskSpaceUsed
            FROM DBC.DiskSpaceV
            WHERE DatabaseName = 'Sales_DWH'";

        return await ExecuteQueryAsync<PerformanceMetrics>(metricsSql);
    }

    public async Task<List<SlowQuery>> IdentifySlowQueriesAsync()
    {
        var slowQuerySql = @"
            SELECT
                QueryText,
                StartTime,
                FirstRespTime,
                TotalIOCount
            FROM DBC.QryLogV
            WHERE TotalIOCount > 1000000
            AND StartTime >= CURRENT_DATE - 7
            ORDER BY TotalIOCount DESC";

        return await ExecuteQueryAsync<List<SlowQuery>>(slowQuerySql);
    }
}
```

---

## 🗄️ **Oracle Exadata**

### **Архитектура**

```bash
Exadata Architecture
├️ Database Servers: Compute nodes
├️ Storage Servers: Smart Flash Cache
├️ InfiniBand: High-speed networking
└️ Smart Scan: Offloading to storage
```

### **Ключевые особенности**

- Database Machine engineered system
- Storage offloading (Smart Scan)
- Hybrid Columnar Compression
- Extreme Performance

### **Пример работы с C#**

```csharp
public class ExadataService
{
    private readonly OracleConnection _connection;

    public async Task<DataTable> ExecuteSmartScanQueryAsync()
    {
        // Exadata Smart Scan offloads processing to storage
        var sql = @"
            SELECT /*+ FULL(s) PARALLEL(s, 8) */
                c.customer_name,
                SUM(s.sales_amount) as total_sales
            FROM sales_fact s
            JOIN customers c ON s.customer_id = c.customer_id
            WHERE s.sales_date >= ADD_MONTHS(SYSDATE, -12)
            AND s.region_id IN (1, 2, 3)
            GROUP BY c.customer_name
            HAVING SUM(s.sales_amount) > 10000";

        using var command = new OracleCommand(sql, _connection);
        command.InitialLONGFetchSize = -1; // Optimize for large results

        using var reader = await command.ExecuteReaderAsync();
        var result = new DataTable();
        result.Load(reader);

        return result;
    }

    public async Task EnableHybridColumnarCompressionAsync(string tableName)
    {
        // HCC обеспечивает высокую степень сжатия
        var compressSql = $@"
            ALTER TABLE {tableName}
            MOVE COMPRESS FOR QUERY HIGH";

        await ExecuteNonQueryAsync(compressSql);
    }

    public async Task<ExadataMetrics> GetStorageMetricsAsync()
    {
        var metricsSql = @"
            SELECT
                CELL_NAME,
                FLASH_CACHE_USED_GB,
                SMART_SCAN_REQUESTS,
                OFFLOAD_ELIGIBLE_BYTES,
                OFFLOADED_BYTES
            FROM V$CELL";

        return await ExecuteQueryAsync<ExadataMetrics>(metricsSql);
    }
}

public class ExadataStorageManager
{
    public async Task ConfigureFlashCacheAsync()
    {
        // Настройка Smart Flash Cache
        var flashSql = @"
            ALTER TABLE sales_fact
            STORAGE (CELL_FLASH_CACHE KEEP)";

        await ExecuteNonQueryAsync(flashSql);
    }

    public async Task EnableStorageIndexesAsync(string tableName)
    {
        // Storage indexes автоматически создаются в Exadata
        var storageSql = $@"
            BEGIN
                DBMS_STATS.GATHER_TABLE_STATS(
                    OWNNAME => 'SALES',
                    TABNAME => '{tableName}',
                    ESTIMATE_PERCENT => DBMS_STATS.AUTO_SAMPLE_SIZE,
                    METHOD_OPT => 'FOR ALL COLUMNS SIZE AUTO'
                );
            END;";

        await ExecuteNonQueryAsync(storageSql);
    }
}
```

---

## ⚡ **ClickHouse**

### **Архитектура**

```bash
ClickHouse Architecture
├️ Column-oriented storage
├️ Vectorized query execution
├️ Data sharding и replication
└️ MergeTree engine
```

### **Ключевые особенности**

- Высокая производительность для аналитических запросов
- Column-oriented storage
- Real-time data ingestion
- Эффективное сжатие данных

### **Пример работы с C#**

```csharp
public class ClickHouseService
{
    private readonly ClickHouseConnection _connection;

    public async Task CreateOptimizedTableAsync()
    {
        // ClickHouse требует специфичной схемы для максимальной производительности
        var createTableSql = @"
            CREATE TABLE IF NOT EXISTS sales_events (
                event_date Date,
                event_time DateTime,
                customer_id UInt32,
                product_id UInt32,
                sales_amount Decimal(10,2),
                region String,
                device_type Enum8('mobile' = 1, 'desktop' = 2, 'tablet' = 3)
            ) ENGINE = MergeTree()
            PARTITION BY toYYYYMM(event_date)
            ORDER BY (event_date, customer_id, product_id)
            SETTINGS index_granularity = 8192";

        await ExecuteNonQueryAsync(createTableSql);
    }

    public async Task<DataTable> ExecuteAnalyticalQueryAsync()
    {
        // ClickHouse оптимизирован для агрегаций
        var sql = @"
            SELECT
                region,
                device_type,
                count() as event_count,
                sum(sales_amount) as total_sales,
                avg(sales_amount) as avg_sale
            FROM sales_events
            WHERE event_date >= today() - 30
            GROUP BY region, device_type
            ORDER BY total_sales DESC";

        using var command = new ClickHouseCommand(sql, _connection);
        using var reader = await command.ExecuteReaderAsync();

        var result = new DataTable();
        result.Load(reader);
        return result;
    }

    public async Task BulkInsertEventsAsync(List<SalesEvent> events)
    {
        // Эффективная bulk insert
        var insertSql = @"
            INSERT INTO sales_events (
                event_date, event_time, customer_id,
                product_id, sales_amount, region, device_type
            ) VALUES";

        using var command = new ClickHouseCommand(insertSql, _connection);

        // ClickHouse поддерживает batch inserts
        foreach (var event in events)
        {
            command.Parameters.Clear();
            command.Parameters.AddWithValue("@event_date", event.EventDate);
            command.Parameters.AddWithValue("@event_time", event.EventTime);
            command.Parameters.AddWithValue("@customer_id", event.CustomerId);
            command.Parameters.AddWithValue("@product_id", event.ProductId);
            command.Parameters.AddWithValue("@sales_amount", event.SalesAmount);
            command.Parameters.AddWithValue("@region", event.Region);
            command.Parameters.AddWithValue("@device_type", event.DeviceType);

            await command.ExecuteNonQueryAsync();
        }
    }
}

public class ClickHouseClusterManager
{
    public async Task ConfigureReplicationAsync()
    {
        // Настройка репликации для отказоустойчивости
        var replicationSql = @"
            CREATE TABLE sales_events_replicated (
                event_date Date,
                customer_id UInt32,
                sales_amount Decimal(10,2)
            ) ENGINE = ReplicatedMergeTree(
                '/clickhouse/tables/{shard}/sales_events',
                '{replica}'
            )
            PARTITION BY toYYYYMM(event_date)
            ORDER BY (event_date, customer_id)";

        await ExecuteNonQueryAsync(replicationSql);
    }

    public async Task<ClickHouseMetrics> GetPerformanceMetricsAsync()
    {
        var metricsSql = @"
            SELECT
                metric,
                value
            FROM system.metrics
            WHERE metric IN (
                'Query', 'InsertQuery', 'SelectQuery',
                'ReplicatedFetch', 'BackgroundPoolTask'
            )";

        return await ExecuteQueryAsync<ClickHouseMetrics>(metricsSql);
    }
}
```

---

## 🎯 **Критерии выбора on-premise решения**

### **Выбирайте Teradata если:**

- ✅ Enterprise-scale аналитика
- ✅ Требуется линейная масштабируемость
- ✅ Сложные workload management требования
- ✅ Бюджет позволяет инвестировать в premium решение

### **Выбирайте Exadata если:**

- ✅ Oracle ecosystem
- ✅ Mixed workload (OLTP + OLAP)
- ✅ Требования к extreme performance
- ✅ Integrated hardware/software solution

### **Выбирайте ClickHouse если:**

- ✅ Real-time аналитика
- ✅ Высокие объемы данных с fast ingestion
- ✅ Cost-effective решение
- ✅ Column-oriented workloads

---

## 🔄 **Миграция в облако**

### **On-premise → Cloud стратегии**

```csharp
public class CloudMigrationService
{
    public async Task<MigrationPlan> CreateMigrationPlanAsync(
        OnPremSystem source,
        CloudTarget target)
    {
        var plan = new MigrationPlan
        {
            SourceSystem = source,
            TargetSystem = target,
            MigrationStrategy = await DetermineBestStrategyAsync(source, target)
        };

        // Assessment phase
        plan.Assessment = await AssessMigrationComplexityAsync(source);

        // Data migration strategy
        plan.DataMigration = await CreateDataMigrationPlanAsync(source, target);

        // Application changes
        plan.ApplicationChanges = await IdentifyRequiredChangesAsync(source, target);

        return plan;
    }

    private async Task<MigrationStrategy> DetermineBestStrategyAsync(
        OnPremSystem source, CloudTarget target)
    {
        // Выбор стратегии миграции
        if (source.DataSize < 10 * 1024 * 1024 * 1024) // 10GB
            return MigrationStrategy.LiftAndShift;

        if (source.HasComplexStoredProcedures)
            return MigrationStrategy.Replatform;

        return MigrationStrategy.Refactor;
    }
}
```

### **Hybrid подходы**

```csharp
public class HybridDataArchitecture
{
    public async Task SetupHybridSolutionAsync()
    {
        // On-premise для sensitive data
        await ConfigureOnPremForSensitiveDataAsync();

        // Cloud для scalability и analytics
        await ConfigureCloudForAnalyticsAsync();

        // Data synchronization
        await SetupDataReplicationAsync();
    }

    private async Task SetupDataReplicationAsync()
    {
        // Real-time replication from on-prem to cloud
        var replicationConfig = new ReplicationConfiguration
        {
            Source = "OnPrem_Teradata",
            Target = "Cloud_Snowflake",
            Tables = new[] { "sales_fact", "customer_dim" },
            ReplicationMode = ReplicationMode.CDC,
            ConflictResolution = ConflictResolution.SourceWins
        };

        await _replicationService.ConfigureAsync(replicationConfig);
    }
}
```

---

## 💰 **Сравнение TCO**

| Решение        | Аппаратная стоимость | Поддержка       | Общая стоимость (5 лет) |
| -------------- | -------------------- | --------------- | ----------------------- |
| **Teradata**   | $500K - $2M+         | 20-30% в год    | $1.5M - $5M+            |
| **Exadata**    | $1M - $5M+           | 25-35% в год    | $3M - $10M+             |
| **ClickHouse** | $50K - $500K         | 10-15% в год    | $200K - $1.5M           |
| **Cloud DWH**  | $0 (CapEx)           | Включено в OpEx | $500K - $2M (5 лет)     |

---

## 📊 **Performance Comparison**

| Workload               | Teradata | Exadata | ClickHouse | Cloud DWH |
| ---------------------- | -------- | ------- | ---------- | --------- |
| **Complex Joins**      | 15.2s    | 12.8s   | 8.9s       | 10.3s     |
| **Large Aggregations** | 28.7s    | 22.1s   | 5.3s       | 7.8s      |
| **Data Loading**       | 45s/GB   | 38s/GB  | 12s/GB     | 25s/GB    |
| **Concurrent Users**   | 100+     | 80+     | 50+        | 200+      |

---

## 🚨 **Anti-patterns**

1. **Over-provisioning hardware** для peak loads
2. **Ignoring cloud economics** при сравнении TCO
3. **Vendor lock-in** без стратегии выхода
4. **Underestimating maintenance costs**

---

## ✅ **Best Practices**

### **Для on-premise решений:**

- Реализуйте proper capacity planning
- Используйте workload management
- Мониторьте hardware utilization
- Планируйте hardware refresh cycles

### **Для миграции:**

- Проведите тщательный assessment перед миграцией
- Рассмотрите hybrid подходы
- Реализуйте поэтапную миграцию
- Тестируйте performance в target environment

### **Общие:**

- Реализуйте comprehensive monitoring
- Создайте disaster recovery plan
- Оптимизируйте queries для конкретной платформы
- Регулярно обновляйте software и firmware

---

**Следующий раздел:** [4.1 - Оптимизация производительности DWH](../operations/01-performance-optimization.md)

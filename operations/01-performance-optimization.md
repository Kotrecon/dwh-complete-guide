# 4.1 - Оптимизация производительности DWH

## Введение

Оптимизация производительности Data Warehouse - критически важная задача, влияющая на пользовательский опыт, стоимость эксплуатации и эффективность бизнес-аналитики. Рассмотрим комплексный подход к оптимизации.

## 📊 Уровни оптимизации DWH

| Уровень          | Методы оптимизации                 | Влияние на производительность |
| ---------------- | ---------------------------------- | ----------------------------- |
| **Запросы**      | Query rewriting, Join optimization | 🟢 Высокое                    |
| **Индексы**      | Columnstore, Bitmap, B-tree        | 🟢 Высокое                    |
| **Partitioning** | Range, List, Hash partitioning     | 🟢 Высокое                    |
| **Clustering**   | Data clustering, Sort keys         | 🟡 Среднее                    |
| **Storage**      | Compression, File formats          | 🟡 Среднее                    |
| **Архитектура**  | Caching, Materialized views        | 🟢 Высокое                    |

---

## 🔍 **Оптимизация запросов**

### **Анализ планов выполнения**

```csharp
public class QueryAnalyzer
{
    public async Task<QueryExecutionPlan> AnalyzeQueryPlanAsync(string query)
    {
        var explainQuery = $"EXPLAIN {query}";
        var planText = await ExecuteScalarAsync<string>(explainQuery);

        return new QueryExecutionPlan
        {
            Query = query,
            PlanText = planText,
            CostEstimate = ExtractCostEstimate(planText),
            Operations = ExtractOperations(planText),
            Recommendations = GenerateOptimizationRecommendations(planText)
        };
    }

    public async Task<List<SlowQuery>> IdentifySlowQueriesAsync(TimeSpan threshold)
    {
        var slowQueriesSql = @"
            SELECT
                query_text,
                execution_time,
                total_io_operations,
                memory_usage,
                execution_count
            FROM system.query_log
            WHERE execution_time > @threshold
            AND execution_date >= CURRENT_DATE - 7
            ORDER BY execution_time DESC";

        return await ExecuteQueryAsync<List<SlowQuery>>(slowQueriesSql,
            new { threshold = threshold.TotalMilliseconds });
    }
}
```

### **Оптимизация JOIN операций**

```csharp
public class JoinOptimizer
{
    public string OptimizeJoinQuery(string originalQuery)
    {
        // Анализ и переписывание JOIN запросов
        var optimizedQuery = originalQuery;

        // Замена INNER JOIN на EXISTS где уместно
        optimizedQuery = ReplaceJoinWithExists(optimizedQuery);

        // Переупорядочивание JOIN для лучшей производительности
        optimizedQuery = ReorderJoinsBySelectivity(optimizedQuery);

        // Добавление JOIN hints если необходимо
        optimizedQuery = AddJoinHints(optimizedQuery);

        return optimizedQuery;
    }

    private string ReplaceJoinWithExists(string query)
    {
        // Замена: SELECT ... FROM A JOIN B ON A.id = B.id
        // На: SELECT ... FROM A WHERE EXISTS (SELECT 1 FROM B WHERE B.id = A.id)
        // Когда нужны только проверки существования
        var pattern = @"FROM\s+(\w+)\s+INNER JOIN\s+(\w+)\s+ON\s+\1\.id\s*=\s*\2\.id";
        var replacement = @"FROM $1 WHERE EXISTS (SELECT 1 FROM $2 WHERE $2.id = $1.id)";

        return Regex.Replace(query, pattern, replacement, RegexOptions.IgnoreCase);
    }
}
```

---

## 🗂️ **Индексы и их оптимизация**

### **Columnstore индексы**

```csharp
public class IndexManager
{
    public async Task<bool> CreateColumnstoreIndexAsync(string tableName, string[] columns)
    {
        var indexName = $"IX_{tableName}_Columnstore";
        var columnsList = string.Join(", ", columns);

        var createIndexSql = $@"
            CREATE COLUMNSTORE INDEX {indexName}
            ON {tableName} ({columnsList})
            WITH (COMPRESSION_DELAY = 0)";

        try
        {
            await ExecuteNonQueryAsync(createIndexSql);
            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to create columnstore index on {Table}", tableName);
            return false;
        }
    }

    public async Task<IndexUsageStats> GetIndexUsageAsync(string tableName)
    {
        var usageSql = @"
            SELECT
                index_name,
                seek_count,
                scan_count,
                lookup_count,
                update_count
            FROM sys.dm_db_index_usage_stats
            WHERE object_id = OBJECT_ID(@tableName)";

        return await ExecuteQueryAsync<IndexUsageStats>(usageSql,
            new { tableName });
    }

    public async Task<List<UnusedIndex>> FindUnusedIndexesAsync()
    {
        var unusedIndexesSql = @"
            SELECT
                t.name as table_name,
                i.name as index_name,
                i.type_desc as index_type
            FROM sys.indexes i
            JOIN sys.tables t ON i.object_id = t.object_id
            LEFT JOIN sys.dm_db_index_usage_stats us
                ON i.object_id = us.object_id AND i.index_id = us.index_id
            WHERE i.index_id > 1  -- Exclude heaps
            AND (us.user_seeks = 0 AND us.user_scans = 0 AND us.user_lookups = 0)
            AND t.is_ms_shipped = 0";

        return await ExecuteQueryAsync<List<UnusedIndex>>(unusedIndexesSql);
    }
}
```

### **Bitmap индексы для low-cardinality столбцов**

```csharp
public class BitmapIndexService
{
    public async Task CreateBitmapIndexesAsync(string tableName)
    {
        // Поиск столбцов с низкой кардинальностью для bitmap индексов
        var lowCardinalityColumns = await FindLowCardinalityColumnsAsync(tableName);

        foreach (var column in lowCardinalityColumns)
        {
            var indexName = $"IX_{tableName}_{column}_Bitmap";
            var createBitmapSql = $@"
                CREATE BITMAP INDEX {indexName}
                ON {tableName} ({column})";

            await ExecuteNonQueryAsync(createBitmapSql);
        }
    }

    private async Task<List<string>> FindLowCardinalityColumnsAsync(string tableName)
    {
        var cardinalitySql = $@"
            SELECT
                column_name,
                COUNT(DISTINCT {column_name}) as distinct_count,
                COUNT(*) as total_count
            FROM {tableName}
            GROUP BY column_name
            HAVING COUNT(DISTINCT {column_name}) / COUNT(*) < 0.01  -- Менее 1% уникальности";

        // Note: Это упрощенный пример, в реальности нужен более сложный анализ
        return await ExecuteQueryAsync<List<string>>(cardinalitySql);
    }
}
```

---

## 🗃️ **Partitioning стратегии**

### **Range partitioning по датам**

```csharp
public class PartitionManager
{
    public async Task CreateDatePartitionsAsync(string tableName, string dateColumn)
    {
        // Создание monthly partitions
        var currentDate = new DateTime(DateTime.UtcNow.Year, DateTime.UtcNow.Month, 1);

        for (int i = -12; i <= 3; i++) // Прошлые 12 месяцев + будущие 3 месяца
        {
            var partitionDate = currentDate.AddMonths(i);
            var partitionName = $"p{partitionDate:yyyyMM}";
            var startDate = partitionDate;
            var endDate = partitionDate.AddMonths(1);

            var partitionSql = $@"
                ALTER TABLE {tableName}
                ADD PARTITION {partitionName}
                VALUES LESS THAN ('{endDate:yyyy-MM-dd}')";

            await ExecuteNonQueryAsync(partitionSql);
        }
    }

    public async Task<string> ImplementPartitionSwitchingAsync(
        string sourceTable,
        string stagingTable,
        string partitionName)
    {
        // Быстрое перемещение данных между таблицами через partition switching
        var switchSql = $@"
            ALTER TABLE {sourceTable}
            SWITCH PARTITION {partitionName}
            TO {stagingTable}";

        await ExecuteNonQueryAsync(switchSql);
        return $"Successfully switched partition {partitionName}";
    }

    public async Task ManagePartitionMaintenanceAsync()
    {
        // Автоматическое управление партициями
        var oldPartitions = await FindOldPartitionsAsync();

        foreach (var partition in oldPartitions)
        {
            // Archive old partitions
            await ArchivePartitionAsync(partition);

            // Drop archived partitions
            await DropPartitionAsync(partition);
        }

        // Create new future partitions
        await CreateFuturePartitionsAsync();
    }
}
```

### **List partitioning для категориальных данных**

```csharp
public class ListPartitionService
{
    public async Task CreateRegionPartitionsAsync(string tableName)
    {
        var regions = new[] { "north", "south", "east", "west", "central" };

        foreach (var region in regions)
        {
            var partitionSql = $@"
                ALTER TABLE {tableName}
                ADD PARTITION p_{region}
                VALUES IN ('{region}')";

            await ExecuteNonQueryAsync(partitionSql);
        }
    }
}
```

---

## 📈 **Clustering и сортировка данных**

### **Automatic clustering**

```csharp
public class ClusteringService
{
    public async Task OptimizeTableClusteringAsync(string tableName, string[] clusterKeys)
    {
        var keys = string.Join(", ", clusterKeys);

        var clusterSql = $@"
            ALTER TABLE {tableName}
            CLUSTER BY ({keys})";

        await ExecuteNonQueryAsync(clusterSql);
    }

    public async Task<ClusteringEffectiveness> AnalyzeClusteringAsync(string tableName)
    {
        var analysisSql = $@"
            SELECT
                table_name,
                avg_partition_count,
                avg_overlap,
                depth_percent
            FROM system.clustering_stats
            WHERE table_name = @tableName";

        return await ExecuteQueryAsync<ClusteringEffectiveness>(analysisSql,
            new { tableName });
    }

    public async Task ReclusterTablesAsync()
    {
        var tablesNeedingRecluster = await FindTablesNeedingReclusterAsync();

        foreach (var table in tablesNeedingRecluster)
        {
            await ExecuteNonQueryAsync($"ALTER TABLE {table.Name} RECLUSTER");
        }
    }
}
```

---

## 💾 **Оптимизация storage**

### **Выбор форматов файлов**

```csharp
public class StorageOptimizer
{
    public async Task ConvertToColumnarFormatAsync(string tableName)
    {
        // Конвертация в columnar формат для лучшего сжатия и производительности
        var convertSql = $@"
            CREATE TABLE {tableName}_optimized
            WITH (
                DISTRIBUTION = HASH(customer_id),
                CLUSTERED COLUMNSTORE INDEX,
                PARTITION (transaction_date RANGE RIGHT FOR VALUES (...))
            ) AS
            SELECT * FROM {tableName}";

        await ExecuteNonQueryAsync(convertSql);

        // Switch to optimized table
        await ExecuteNonQueryAsync($"EXEC sp_rename '{tableName}', '{tableName}_old'");
        await ExecuteNonQueryAsync($"EXEC sp_rename '{tableName}_optimized', '{tableName}'");
    }

    public async Task<CompressionStats> AnalyzeCompressionAsync(string tableName)
    {
        var statsSql = $@"
            SELECT
                table_name,
                compression_type,
                compressed_size_gb,
                uncompressed_size_gb,
                compression_ratio
            FROM system.compression_stats
            WHERE table_name = @tableName";

        return await ExecuteQueryAsync<CompressionStats>(statsSql,
            new { tableName });
    }
}
```

---

## ⚡ **Кэширование и материализованные представления**

### **Materialized views для часто используемых агрегаций**

```csharp
public class MaterializedViewService
{
    public async Task CreateMaterializedViewsAsync()
    {
        var views = new[]
        {
            new MaterializedViewDefinition
            {
                Name = "mv_daily_sales_summary",
                Query = @"
                    SELECT
                        transaction_date,
                        customer_segment,
                        SUM(sales_amount) as daily_sales,
                        COUNT(*) as transaction_count,
                        AVG(sales_amount) as avg_transaction
                    FROM fact_sales
                    GROUP BY transaction_date, customer_segment",
                RefreshSchedule = "DAILY",
                EnableIncrementalRefresh = true
            },
            new MaterializedViewDefinition
            {
                Name = "mv_customer_lifetime_value",
                Query = @"
                    SELECT
                        customer_id,
                        SUM(sales_amount) as lifetime_value,
                        COUNT(DISTINCT transaction_date) as active_days,
                        MAX(transaction_date) as last_purchase_date
                    FROM fact_sales
                    GROUP BY customer_id",
                RefreshSchedule = "WEEKLY",
                EnableIncrementalRefresh = false
            }
        };

        foreach (var view in views)
        {
            await CreateMaterializedViewAsync(view);
        }
    }

    private async Task CreateMaterializedViewAsync(MaterializedViewDefinition view)
    {
        var createSql = $@"
            CREATE MATERIALIZED VIEW {view.Name}
            AS {view.Query}
            WITH REFRESH {view.RefreshSchedule}";

        await ExecuteNonQueryAsync(createSql);
    }
}
```

### **Query result caching**

```csharp
public class QueryCacheService
{
    private readonly IMemoryCache _memoryCache;
    private readonly IDistributedCache _distributedCache;

    public async Task<QueryResult> ExecuteCachedQueryAsync(
        string query,
        TimeSpan cacheDuration,
        object parameters = null)
    {
        var cacheKey = GenerateCacheKey(query, parameters);

        // Попытка получить из кэша
        if (_memoryCache.TryGetValue(cacheKey, out QueryResult cachedResult))
        {
            return cachedResult;
        }

        // Выполнение запроса
        var result = await ExecuteQueryAsync<QueryResult>(query, parameters);

        // Сохранение в кэш
        var cacheOptions = new MemoryCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = cacheDuration
        };

        _memoryCache.Set(cacheKey, result, cacheOptions);

        return result;
    }

    public async Task PrewarmCacheAsync()
    {
        var popularQueries = await GetPopularQueriesAsync();

        foreach (var query in popularQueries)
        {
            // Предварительное выполнение и кэширование популярных запросов
            await ExecuteCachedQueryAsync(query.Query, TimeSpan.FromHours(1));
        }
    }
}
```

---

## 📊 **Мониторинг производительности**

### **Комплексный мониторинг**

```csharp
public class PerformanceMonitor
{
    public async Task<PerformanceMetrics> CollectPerformanceMetricsAsync()
    {
        var metrics = new PerformanceMetrics
        {
            Timestamp = DateTime.UtcNow,
            QueryMetrics = await GetQueryPerformanceAsync(),
            StorageMetrics = await GetStorageMetricsAsync(),
            IndexMetrics = await GetIndexUsageMetricsAsync(),
            MemoryMetrics = await GetMemoryUsageAsync(),
            ConnectionMetrics = await GetConnectionStatsAsync()
        };

        // Анализ и генерация рекомендаций
        metrics.Recommendations = await GenerateOptimizationRecommendationsAsync(metrics);

        return metrics;
    }

    public async Task SetupPerformanceAlertsAsync()
    {
        var alerts = new[]
        {
            new PerformanceAlert
            {
                Metric = "QueryExecutionTime",
                Threshold = TimeSpan.FromMinutes(5),
                Severity = AlertSeverity.High
            },
            new PerformanceAlert
            {
                Metric = "DiskSpaceUsage",
                Threshold = 0.85, // 85%
                Severity = AlertSeverity.Medium
            },
            new PerformanceAlert
            {
                Metric = "MemoryPressure",
                Threshold = 0.90, // 90%
                Severity = AlertSeverity.Critical
            }
        };

        foreach (var alert in alerts)
        {
            await ConfigureAlertAsync(alert);
        }
    }
}
```

### **Automatic performance tuning**

```csharp
public class AutomaticTuningService
{
    public async Task RunAutomaticTuningAsync()
    {
        // Автоматическое создание индексов
        var missingIndexes = await FindMissingIndexesAsync();
        foreach (var index in missingIndexes.Take(5)) // Ограничение для безопасности
        {
            await CreateIndexAsync(index);
        }

        // Автоматическое обновление статистик
        var tablesNeedingStatsUpdate = await FindTablesWithStaleStatisticsAsync();
        foreach (var table in tablesNeedingStatsUpdate)
        {
            await UpdateStatisticsAsync(table);
        }

        // Автоматическая дефрагментация индексов
        var fragmentedIndexes = await FindFragmentedIndexesAsync();
        foreach (var index in fragmentedIndexes.Where(i => i.Fragmentation > 30))
        {
            await RebuildIndexAsync(index);
        }
    }
}
```

---

## 🎯 **Best Practices оптимизации**

### **Для запросов:**

```csharp
public class QueryOptimizationRules
{
    public static readonly List<OptimizationRule> Rules = new()
    {
        new OptimizationRule
        {
            Name = "Avoid SELECT *",
            Description = "Используйте явное перечисление столбцов",
            Example = "SELECT column1, column2 FROM table"
        },
        new OptimizationRule
        {
            Name = "Use WHERE instead of HAVING for filtering",
            Description = "Фильтруйте данные до агрегации",
            Example = "WHERE date >= '2024-01-01' вместо HAVING date >= '2024-01-01'"
        },
        new OptimizationRule
        {
            Name = "Avoid correlated subqueries",
            Description = "Используйте JOIN вместо коррелированных подзапросов",
            Example = "JOIN вместо WHERE EXISTS"
        }
    };
}
```

### **Для индексов:**

- Используйте columnstore индексы для аналитических workload-ов
- Создавайте bitmap индексы для low-cardinality столбцов
- Регулярно мониторьте и удаляйте неиспользуемые индексы
- Используйте covering индексы для часто используемых запросов

### **Для partitioning:**

- Выбирайте partitioning key на основе patterns доступа
- Используйте range partitioning для временных рядов
- Реализуйте sliding window для управления историческими данными
- Избегайте over-partitioning

---

## 🚨 **Anti-patterns**

1. **Over-indexing** - слишком много индексов замедляет DML операции
2. **Under-partitioning** - большие таблицы без partitioning
3. **Ignoring statistics** - устаревшие статистики ведут к плохим планам выполнения
4. **Complex queries without optimization** - неоптимизированные сложные запросы

---

## ✅ **Checklist оптимизации**

### **Ежедневно:**

- [ ] Мониторинг медленных запросов
- [ ] Проверка использования индексов
- [ ] Мониторинг использования ресурсов

### **Еженедельно:**

- [ ] Анализ производительности
- [ ] Обновление статистик
- [ ] Проверка fragmentation индексов

### **Ежемесячно:**

- [ ] Review partitioning стратегии
- [ ] Анализ роста данных
- [ ] Capacity planning

---

**Следующий раздел:** [4.2 - Data Governance и безопасность](./02-data-governance-security.md)

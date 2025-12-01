# 2.4 - Data Quality и мониторинг

## Введение

Качество данных - критический аспект успешного Data Warehouse. Плохое качество данных приводит к неверным бизнес-решениям и потере доверия к аналитике.

## 📊 Dimensions of Data Quality

| Измерение        | Описание                        | Метрики                              |
| ---------------- | ------------------------------- | ------------------------------------ |
| **Completeness** | Наличие всех ожидаемых данных   | % заполненных полей, количество NULL |
| **Accuracy**     | Соответствие реальным значениям | % ошибок, точность данных            |
| **Consistency**  | Единообразие между системами    | Расхождения между источниками        |
| **Timeliness**   | Актуальность данных             | Задержка данных, freshness           |
| **Validity**     | Соответствие форматам           | % валидных записей                   |
| **Uniqueness**   | Отсутствие дубликатов           | Количество дубликатов                |

---

## 🛡️ **Data Quality Framework**

### **Архитектура DQ Framework**

```csharp
public class DataQualityFramework
{
    private readonly List<IDataQualityRule> _rules;
    private readonly IDataQualityRepository _repository;

    public DataQualityResult ValidateData(DataSet data, string dataSource)
    {
        var result = new DataQualityResult
        {
            DataSource = dataSource,
            ValidationDate = DateTime.UtcNow,
            TotalRecords = data.Records.Count
        };

        foreach (var rule in _rules)
        {
            var ruleResult = rule.Validate(data);
            result.RuleResults.Add(ruleResult);
        }

        result.CalculateOverallScore();
        _repository.SaveResult(result);

        return result;
    }
}
```

### **Базовые правила качества данных**

```csharp
public interface IDataQualityRule
{
    string RuleName { get; }
    string Description { get; }
    DataQualityRuleResult Validate(DataSet data);
}

public class CompletenessRule : IDataQualityRule
{
    public string RuleName => "FieldCompleteness";
    public string Description => "Проверка заполненности обязательных полей";

    public DataQualityRuleResult Validate(DataSet data)
    {
        var result = new DataQualityRuleResult { RuleName = this.RuleName };
        var requiredFields = new[] { "CustomerId", "OrderDate", "Amount" };

        foreach (var record in data.Records)
        {
            foreach (var field in requiredFields)
            {
                if (string.IsNullOrEmpty(record[field]?.ToString()))
                {
                    result.FailedRecords++;
                    result.ErrorDetails.Add(
                        $"Record {record["RecordId"]}: Field {field} is empty");
                }
            }
        }

        result.PassedRecords = data.Records.Count - result.FailedRecords;
        result.SuccessRate = (decimal)result.PassedRecords / data.Records.Count * 100;

        return result;
    }
}
```

---

## 🔍 **Completeness Validation**

### **Проверка заполненности полей**

```csharp
public class CompletenessValidator
{
    public CompletenessResult ValidateCompleteness(DataTable data, string[] requiredFields)
    {
        var result = new CompletenessResult();

        foreach (DataRow row in data.Rows)
        {
            var recordCompleteness = new RecordCompleteness
            {
                RecordId = row["Id"].ToString()
            };

            foreach (var field in requiredFields)
            {
                if (row[field] == DBNull.Value || string.IsNullOrEmpty(row[field]?.ToString()))
                {
                    recordCompleteness.MissingFields.Add(field);
                }
            }

            result.RecordResults.Add(recordCompleteness);
        }

        result.CalculateMetrics();
        return result;
    }
}

public class CompletenessResult
{
    public int TotalRecords { get; set; }
    public int CompleteRecords { get; set; }
    public decimal CompletenessPercentage =>
        TotalRecords > 0 ? (decimal)CompleteRecords / TotalRecords * 100 : 0;

    public List<RecordCompleteness> RecordResults { get; set; } = new();

    public void CalculateMetrics()
    {
        TotalRecords = RecordResults.Count;
        CompleteRecords = RecordResults.Count(r => r.MissingFields.Count == 0);
    }
}
```

---

## ✅ **Accuracy Validation**

### **Валидация точности данных**

```csharp
public class AccuracyValidator
{
    private readonly IReferenceDataService _referenceData;

    public AccuracyResult ValidateAccuracy(DataTable data)
    {
        var result = new AccuracyResult();

        foreach (DataRow row in data.Rows)
        {
            // Проверка существования клиента в справочнике
            var customerId = row["CustomerId"].ToString();
            if (!_referenceData.CustomerExists(customerId))
            {
                result.InvalidReferences++;
                result.Errors.Add($"Customer {customerId} not found in reference data");
            }

            // Проверка допустимых диапазонов
            var amount = Convert.ToDecimal(row["Amount"]);
            if (amount < 0 || amount > 1000000)
            {
                result.OutOfRangeValues++;
                result.Errors.Add($"Amount {amount} is out of valid range");
            }

            // Проверка форматов дат
            if (!DateTime.TryParse(row["OrderDate"].ToString(), out _))
            {
                result.InvalidFormats++;
                result.Errors.Add($"Invalid date format: {row["OrderDate"]}");
            }
        }

        return result;
    }
}
```

---

## 🔄 **Consistency Validation**

### **Сравнение между источниками**

```csharp
public class ConsistencyValidator
{
    public ConsistencyResult ValidateConsistency(
        DataTable source1,
        DataTable source2,
        string keyField)
    {
        var result = new ConsistencyResult();

        var source1Dict = source1.AsEnumerable()
            .ToDictionary(row => row[keyField].ToString());
        var source2Dict = source2.AsEnumerable()
            .ToDictionary(row => row[keyField].ToString());

        // Поиск записей только в одном источнике
        result.OnlyInSource1 = source1Dict.Keys.Except(source2Dict.Keys).ToList();
        result.OnlyInSource2 = source2Dict.Keys.Except(source1Dict.Keys).ToList();

        // Сравнение общих записей
        foreach (var key in source1Dict.Keys.Intersect(source2Dict.Keys))
        {
            var row1 = source1Dict[key];
            var row2 = source2Dict[key];

            var differences = CompareRows(row1, row2);
            if (differences.Any())
            {
                result.InconsistentRecords.Add(new InconsistentRecord
                {
                    Key = key,
                    Differences = differences
                });
            }
        }

        return result;
    }

    private List<FieldDifference> CompareRows(DataRow row1, DataRow row2)
    {
        var differences = new List<FieldDifference>();

        foreach (DataColumn column in row1.Table.Columns)
        {
            var value1 = row1[column];
            var value2 = row2[column];

            if (!Equals(value1, value2))
            {
                differences.Add(new FieldDifference
                {
                    FieldName = column.ColumnName,
                    Value1 = value1,
                    Value2 = value2
                });
            }
        }

        return differences;
    }
}
```

---

## ⏱️ **Timeliness Monitoring**

### **Мониторинг актуальности данных**

```csharp
public class TimelinessMonitor
{
    private readonly IDataFreshnessRepository _repository;

    public async Task<FreshnessReport> CheckDataFreshnessAsync()
    {
        var report = new FreshnessReport
        {
            CheckTime = DateTime.UtcNow
        };

        var dataSources = await _repository.GetDataSourcesAsync();

        foreach (var source in dataSources)
        {
            var lastUpdate = await _repository.GetLastUpdateTimeAsync(source.Name);
            var freshness = new DataFreshness
            {
                DataSource = source.Name,
                LastUpdateTime = lastUpdate,
                ExpectedFrequency = source.ExpectedUpdateFrequency,
                CurrentDelay = DateTime.UtcNow - lastUpdate
            };

            freshness.IsWithinSla = freshness.CurrentDelay <= source.SlaThreshold;
            report.FreshnessResults.Add(freshness);
        }

        report.CalculateOverallHealth();
        return report;
    }
}

public class DataFreshness
{
    public string DataSource { get; set; }
    public DateTime LastUpdateTime { get; set; }
    public TimeSpan ExpectedFrequency { get; set; }
    public TimeSpan CurrentDelay { get; set; }
    public bool IsWithinSla { get; set; }

    public string Status => IsWithinSla ? "Healthy" : "Delayed";
    public double DelayHours => CurrentDelay.TotalHours;
}
```

---

## 📈 **Data Quality Dashboard**

### **Метрики для мониторинга**

```csharp
public class DataQualityMetrics
{
    public DateTime CalculationDate { get; set; }
    public string DataSetName { get; set; }

    // Completeness
    public decimal CompletenessScore { get; set; }
    public int TotalRecords { get; set; }
    public int CompleteRecords { get; set; }

    // Accuracy
    public decimal AccuracyScore { get; set; }
    public int ValidRecords { get; set; }
    public int InvalidReferences { get; set; }
    public int OutOfRangeValues { get; set; }

    // Consistency
    public decimal ConsistencyScore { get; set; }
    public int InconsistentRecords { get; set; }

    // Timeliness
    public decimal TimelinessScore { get; set; }
    public TimeSpan AverageDelay { get; set; }
    public int DelayedSources { get; set; }

    // Overall
    public decimal OverallScore => (CompletenessScore + AccuracyScore +
                                  ConsistencyScore + TimelinessScore) / 4;

    public string HealthStatus => OverallScore switch
    {
        >= 95 => "Excellent",
        >= 85 => "Good",
        >= 75 => "Fair",
        >= 60 => "Poor",
        _ => "Critical"
    };
}
```

### **Автоматизированная отчетность**

```csharp
public class QualityReportGenerator
{
    public async Task GenerateDailyQualityReportAsync()
    {
        var metrics = await CalculateDailyMetricsAsync();
        var report = new QualityReport
        {
            ReportDate = DateTime.UtcNow.Date,
            Metrics = metrics,
            Trends = await CalculateTrendsAsync(),
            Issues = await IdentifyCriticalIssuesAsync()
        };

        // Сохранение отчета
        await _repository.SaveReportAsync(report);

        // Отправка уведомлений при проблемах
        if (report.Metrics.HealthStatus == "Critical" ||
            report.Metrics.HealthStatus == "Poor")
        {
            await _notificationService.SendAlertAsync(report);
        }

        // Обновление дашборда
        await _dashboardService.UpdateQualityDashboardAsync(report);
    }
}
```

---

## 🚨 **Alerting и мониторинг**

### **Система оповещений**

```csharp
public class DataQualityAlertSystem
{
    private readonly List<IQualityAlert> _alerts;

    public async Task CheckAndNotifyAlertsAsync()
    {
        var activeAlerts = new List<QualityAlert>();

        foreach (var alert in _alerts)
        {
            if (await alert.ShouldTriggerAsync())
            {
                activeAlerts.Add(await alert.CreateAlertAsync());
            }
        }

        if (activeAlerts.Any())
        {
            await _notificationService.SendAlertsAsync(activeAlerts);
            await _alertRepository.LogAlertsAsync(activeAlerts);
        }
    }
}

public class CompletenessAlert : IQualityAlert
{
    public string AlertName => "LowCompletenessAlert";
    public decimal Threshold => 85.0m; // 85%

    public async Task<bool> ShouldTriggerAsync()
    {
        var recentMetrics = await _metricsService.GetRecentCompletenessAsync();
        return recentMetrics.Any(m => m.CompletenessScore < Threshold);
    }

    public async Task<QualityAlert> CreateAlertAsync()
    {
        var metrics = await _metricsService.GetRecentCompletenessAsync();
        var lowest = metrics.OrderBy(m => m.CompletenessScore).First();

        return new QualityAlert
        {
            AlertType = "Completeness",
            Severity = "High",
            Message = $"Low completeness detected: {lowest.CompletenessScore}%",
            DataSource = lowest.DataSetName,
            SuggestedAction = "Check ETL process for missing data"
        };
    }
}
```

---

## 🔧 **Инструменты и интеграции**

### **Интеграция с ETL процессами**

```csharp
public class QualityAwareEtlProcess
{
    public async Task<EtlResult> ExecuteWithQualityCheckAsync()
    {
        var extractResult = await ExtractDataAsync();

        // Проверка качества после extraction
        var qualityResult = await _qualityService.ValidateAsync(extractResult.Data);
        if (qualityResult.OverallScore < 90)
        {
            throw new DataQualityException(
                $"Data quality too low: {qualityResult.OverallScore}%",
                qualityResult);
        }

        var transformResult = await TransformDataAsync(extractResult.Data);
        var loadResult = await LoadDataAsync(transformResult.Data);

        // Логирование метрик качества
        await _metricsRepository.LogQualityMetricsAsync(qualityResult);

        return new EtlResult
        {
            Success = true,
            RecordsProcessed = loadResult.RecordsLoaded,
            QualityMetrics = qualityResult
        };
    }
}
```

---

## 📊 **Data Quality KPI**

| KPI                       | Формула                                  | Целевое значение |
| ------------------------- | ---------------------------------------- | ---------------- |
| **Completeness Rate**     | (Complete Records / Total Records) × 100 | ≥ 98%            |
| **Accuracy Rate**         | (Valid Records / Total Records) × 100    | ≥ 99%            |
| **Timeliness Rate**       | (On-time Updates / Total Updates) × 100  | ≥ 95%            |
| **Issue Resolution Time** | Среднее время решения проблем            | ≤ 4 часа         |
| **Data Quality Score**    | Среднее всех метрик                      | ≥ 95%            |

---

## 🚨 **Common Data Quality Issues**

### **Проблемы и решения**

```csharp
public class CommonQualityIssues
{
    // Проблема: NULL значения в обязательных полях
    public async Task FixNullValuesAsync(string tableName, string fieldName)
    {
        var sql = $@"
            UPDATE {tableName}
            SET {fieldName} = 'Unknown'
            WHERE {fieldName} IS NULL";

        await _db.ExecuteAsync(sql);
    }

    // Проблема: Неверные форматы дат
    public async Task FixDateFormatsAsync(string tableName, string dateField)
    {
        var invalidDates = await _db.QueryAsync<string>($@"
            SELECT {dateField}
            FROM {tableName}
            WHERE ISDATE({dateField}) = 0");

        foreach (var invalidDate in invalidDates)
        {
            // Логика исправления форматов
            await FixIndividualDateAsync(tableName, dateField, invalidDate);
        }
    }

    // Проблема: Дубликаты записей
    public async Task RemoveDuplicatesAsync(string tableName, string[] keyFields)
    {
        var keyFieldsList = string.Join(", ", keyFields);
        var sql = $@"
            WITH CTE AS (
                SELECT *,
                       ROW_NUMBER() OVER (
                           PARTITION BY {keyFieldsList}
                           ORDER BY LastModified DESC
                       ) as rn
                FROM {tableName}
            )
            DELETE FROM CTE WHERE rn > 1";

        await _db.ExecuteAsync(sql);
    }
}
```

---

## ✅ **Best Practices**

### **Проактивный мониторинг:**

- Реализуйте автоматические проверки качества
- Установите thresholds для ключевых метрик
- Создайте escalation procedures для критических проблем

### **Процессы улучшения:**

- Регулярные data quality audits
- Root cause analysis для повторяющихся проблем
- Обучение команды best practices

### **Инструментарий:**

- Используйте специализированные DQ tools (Great Expectations, etc.)
- Интегрируйте DQ checks в CI/CD pipeline
- Создайте централизованный DQ dashboard

### **Организационные аспекты:**

- Назначьте data stewards для критичных данных
- Создайте data quality policy
- Регулярно отчитывайтесь о качестве данных стейкхолдерам

---

**Следующий раздел:** [3.1 - Облачные DWH: BigQuery, Snowflake, Redshift, Synapse](../technology/01-cloud-dwh-comparison.md)

# 4.4 - Мониторинг и метрики здоровья DWH

## Введение

Комплексный мониторинг Data Warehouse критически важен для обеспечения производительности, доступности и эффективности. Правильно настроенная система мониторинга позволяет proactively выявлять проблемы и оптимизировать работу DWH.

## 📊 Категории метрик мониторинга

| Категория              | Ключевые метрики                 | Целевые значения                |
| ---------------------- | -------------------------------- | ------------------------------- |
| **Производительность** | Query execution time, Throughput | P95 < 30s, Availability > 99.9% |
| **Ресурсы**            | CPU, Memory, Disk I/O            | Utilization < 80%               |
| **Данные**             | Data freshness, Quality scores   | Freshness < 1h, Quality > 95%   |
| **Бизнес**             | User satisfaction, ROI           | CSAT > 4.0/5.0, ROI > 150%      |

---

## ⚡ **Мониторинг производительности**

### **Система сбора метрик производительности**

```csharp
public class PerformanceMetricsCollector
{
    private readonly IMetricsRepository _metricsRepository;

    public async Task<PerformanceSnapshot> CollectPerformanceSnapshotAsync()
    {
        var snapshot = new PerformanceSnapshot
        {
            Timestamp = DateTime.UtcNow,
            QueryMetrics = await CollectQueryMetricsAsync(),
            ResourceMetrics = await CollectResourceMetricsAsync(),
            ConnectionMetrics = await CollectConnectionMetricsAsync(),
            CacheMetrics = await CollectCacheMetricsAsync()
        };

        // Расчет агрегированных показателей
        snapshot.HealthScore = CalculateHealthScore(snapshot);
        snapshot.Recommendations = await GenerateRecommendationsAsync(snapshot);

        await _metricsRepository.StoreSnapshotAsync(snapshot);
        return snapshot;
    }

    private async Task<QueryMetrics> CollectQueryMetricsAsync()
    {
        return new QueryMetrics
        {
            TotalQueries = await GetTotalQueryCountAsync(),
            SlowQueries = await GetSlowQueryCountAsync(TimeSpan.FromSeconds(30)),
            FailedQueries = await GetFailedQueryCountAsync(),
            AverageExecutionTime = await GetAverageExecutionTimeAsync(),
            P95ExecutionTime = await GetPercentileExecutionTimeAsync(95),
            ConcurrentQueries = await GetConcurrentQueryCountAsync(),
            QueryThroughput = await GetQueryThroughputAsync()
        };
    }

    private async Task<ResourceMetrics> CollectResourceMetricsAsync()
    {
        return new ResourceMetrics
        {
            CpuUsage = await GetCpuUsageAsync(),
            MemoryUsage = await GetMemoryUsageAsync(),
            DiskUsage = await GetDiskUsageAsync(),
            NetworkIo = await GetNetworkIoAsync(),
            StorageCapacity = await GetStorageCapacityAsync(),
            CacheHitRatio = await GetCacheHitRatioAsync()
        };
    }
}

public class PerformanceSnapshot
{
    public DateTime Timestamp { get; set; }
    public QueryMetrics QueryMetrics { get; set; }
    public ResourceMetrics ResourceMetrics { get; set; }
    public ConnectionMetrics ConnectionMetrics { get; set; }
    public CacheMetrics CacheMetrics { get; set; }
    public double HealthScore { get; set; }
    public List<string> Recommendations { get; set; }
}
```

### **Анализ и визуализация метрик**

```csharp
public class PerformanceAnalyzer
{
    public async Task<PerformanceTrends> AnalyzePerformanceTrendsAsync(
        DateTime startDate,
        DateTime endDate)
    {
        var snapshots = await _metricsRepository.GetSnapshotsAsync(startDate, endDate);

        return new PerformanceTrends
        {
            Period = $"{startDate:yyyy-MM-dd} to {endDate:yyyy-MM-dd}",
            QueryPerformanceTrend = AnalyzeQueryTrend(snapshots),
            ResourceUtilizationTrend = AnalyzeResourceTrend(snapshots),
            SeasonalPatterns = DetectSeasonalPatterns(snapshots),
            Anomalies = await DetectPerformanceAnomaliesAsync(snapshots),
            Forecast = await GeneratePerformanceForecastAsync(snapshots)
        };
    }

    public async Task<List<PerformanceAnomaly>> DetectPerformanceAnomaliesAsync(
        List<PerformanceSnapshot> snapshots)
    {
        var anomalies = new List<PerformanceAnomaly>();

        // Обнаружение аномалий в execution time
        var executionTimeAnomalies = await DetectExecutionTimeAnomaliesAsync(snapshots);
        anomalies.AddRange(executionTimeAnomalies);

        // Обнаружение аномалий в использовании ресурсов
        var resourceAnomalies = await DetectResourceAnomaliesAsync(snapshots);
        anomalies.AddRange(resourceAnomalies);

        // Обнаружение аномальных patterns доступа
        var accessPatternAnomalies = await DetectAccessPatternAnomaliesAsync(snapshots);
        anomalies.AddRange(accessPatternAnomalies);

        return anomalies.OrderByDescending(a => a.Severity).ToList();
    }
}
```

---

## 💾 **Мониторинг данных**

### **Data Quality и Freshness мониторинг**

```csharp
public class DataQualityMonitor
{
    public async Task<DataQualityReport> GenerateQualityReportAsync()
    {
        var report = new DataQualityReport
        {
            ReportDate = DateTime.UtcNow,
            DatasetMetrics = new Dictionary<string, DatasetQualityMetrics>()
        };

        var allDatasets = await GetAllDatasetsAsync();

        foreach (var dataset in allDatasets)
        {
            var metrics = await CalculateDatasetQualityAsync(dataset);
            report.DatasetMetrics[dataset] = metrics;
        }

        report.OverallQualityScore = CalculateOverallQualityScore(report.DatasetMetrics.Values);
        report.QualityTrend = await CalculateQualityTrendAsync();

        return report;
    }

    private async Task<DatasetQualityMetrics> CalculateDatasetQualityAsync(string dataset)
    {
        return new DatasetQualityMetrics
        {
            DatasetName = dataset,
            Completeness = await CalculateCompletenessAsync(dataset),
            Accuracy = await CalculateAccuracyAsync(dataset),
            Timeliness = await CalculateTimelinessAsync(dataset),
            Consistency = await CalculateConsistencyAsync(dataset),
            Validity = await CalculateValidityAsync(dataset),
            Uniqueness = await CalculateUniquenessAsync(dataset),
            LastVerified = DateTime.UtcNow
        };
    }
}

public class DataFreshnessMonitor
{
    public async Task<FreshnessReport> CheckDataFreshnessAsync()
    {
        var report = new FreshnessReport
        {
            CheckTime = DateTime.UtcNow,
            DataSources = new List<DataSourceFreshness>()
        };

        var dataSources = await GetDataSourcesAsync();

        foreach (var source in dataSources)
        {
            var freshness = await CheckSourceFreshnessAsync(source);
            report.DataSources.Add(freshness);
        }

        report.OverallFreshnessStatus = CalculateOverallFreshnessStatus(report.DataSources);
        report.StaleDataSources = report.DataSources.Where(s => s.IsStale).ToList();

        return report;
    }

    private async Task<DataSourceFreshness> CheckSourceFreshnessAsync(DataSource source)
    {
        var lastUpdate = await GetLastUpdateTimeAsync(source);
        var expectedFrequency = source.ExpectedUpdateFrequency;
        var currentDelay = DateTime.UtcNow - lastUpdate;

        return new DataSourceFreshness
        {
            SourceName = source.Name,
            LastUpdateTime = lastUpdate,
            ExpectedFrequency = expectedFrequency,
            CurrentDelay = currentDelay,
            IsStale = currentDelay > expectedFrequency * 2, // 2x от expected
            HealthStatus = CalculateFreshnessHealthStatus(currentDelay, expectedFrequency)
        };
    }
}
```

---

## 🚨 **Система оповещений**

### **Многоуровневая система alerting**

```csharp
public class AlertingSystem
{
    private readonly List<IAlertHandler> _alertHandlers;

    public async Task<AlertResult> ProcessAlertAsync(Alert alert)
    {
        var result = new AlertResult
        {
            AlertId = alert.Id,
            ProcessingStartTime = DateTime.UtcNow
        };

        // Определение severity и routing
        alert.Severity = await CalculateAlertSeverityAsync(alert);
        var handlers = GetApplicableHandlers(alert);

        // Обработка через цепочку ответственности
        foreach (var handler in handlers)
        {
            var handlerResult = await handler.HandleAlertAsync(alert);
            result.HandlerResults.Add(handlerResult);

            if (handlerResult.IsHandled)
            {
                result.IsHandled = true;
                break;
            }
        }

        result.ProcessingEndTime = DateTime.UtcNow;

        // Логирование результата
        await _alertRepository.LogAlertResultAsync(result);

        return result;
    }

    public async Task ConfigureAlertRulesAsync()
    {
        var rules = new[]
        {
            new AlertRule
            {
                Name = "HighCPUUsage",
                Metric = "cpu_usage",
                Condition = ">",
                Threshold = 85.0,
                Duration = TimeSpan.FromMinutes(5),
                Severity = AlertSeverity.High,
                Actions = new[] { "NotifyOnCall", "ScaleResources" }
            },
            new AlertRule
            {
                Name = "SlowQueryAlert",
                Metric = "query_execution_time_p95",
                Condition = ">",
                Threshold = 30000, // 30 seconds
                Duration = TimeSpan.FromMinutes(10),
                Severity = AlertSeverity.Medium,
                Actions = new[] { "NotifyTeam", "CreateJiraTicket" }
            },
            new AlertRule
            {
                Name = "DataFreshnessAlert",
                Metric = "data_freshness_delay",
                Condition = ">",
                Threshold = 3600, // 1 hour
                Duration = TimeSpan.FromMinutes(30),
                Severity = AlertSeverity.Critical,
                Actions = new[] { "PageOnCall", "StopDataIngestion" }
            }
        };

        foreach (var rule in rules)
        {
            await ConfigureAlertRuleAsync(rule);
        }
    }
}

public class SmartAlertCorrelation
{
    public async Task<List<CorrelatedAlert>> CorrelateAlertsAsync(List<Alert> alerts)
    {
        var correlatedAlerts = new List<CorrelatedAlert>();

        // Группировка по времени и ресурсам
        var timeWindow = TimeSpan.FromMinutes(15);
        var timeGroups = alerts.GroupBy(a => a.Timestamp.Round(timeWindow));

        foreach (var timeGroup in timeGroups)
        {
            // Группировка по affected resources
            var resourceGroups = timeGroup.GroupBy(a => a.AffectedResource);

            foreach (var resourceGroup in resourceGroups)
            {
                if (resourceGroup.Count() > 1)
                {
                    var correlatedAlert = new CorrelatedAlert
                    {
                        CorrelationId = Guid.NewGuid(),
                        OriginalAlerts = resourceGroup.ToList(),
                        RootCause = await FindRootCauseAsync(resourceGroup.ToList()),
                        CombinedSeverity = CalculateCombinedSeverity(resourceGroup.ToList()),
                        RecommendedActions = await GenerateCorrelatedActionsAsync(resourceGroup.ToList())
                    };

                    correlatedAlerts.Add(correlatedAlert);
                }
            }
        }

        return correlatedAlerts;
    }
}
```

---

## 📈 **Dashboard и отчетность**

### **Комплексные дашборды**

```csharp
public class DashboardService
{
    public async Task<DwhDashboard> BuildComprehensiveDashboardAsync()
    {
        var dashboard = new DwhDashboard
        {
            LastUpdated = DateTime.UtcNow,
            PerformanceWidgets = await BuildPerformanceWidgetsAsync(),
            DataQualityWidgets = await BuildDataQualityWidgetsAsync(),
            ResourceWidgets = await BuildResourceWidgetsAsync(),
            BusinessWidgets = await BuildBusinessWidgetsAsync(),
            AlertWidgets = await BuildAlertWidgetsAsync()
        };

        return dashboard;
    }

    private async Task<List<DashboardWidget>> BuildPerformanceWidgetsAsync()
    {
        return new List<DashboardWidget>
        {
            new TimeSeriesWidget
            {
                Title = "Query Performance",
                Metrics = new[] { "p95_execution_time", "throughput" },
                TimeRange = TimeSpan.FromHours(24),
                RefreshInterval = TimeSpan.FromMinutes(5)
            },
            new GaugeWidget
            {
                Title = "System Health Score",
                Metric = "health_score",
                Ranges = new[] { 0.0, 70.0, 90.0, 100.0 },
                Colors = new[] { "red", "yellow", "green" }
            },
            new TopNWidget
            {
                Title = "Slowest Queries",
                Metric = "execution_time",
                Limit = 10,
                TimeRange = TimeSpan.FromHours(1)
            }
        };
    }
}

public class ReportingService
{
    public async Task<DwhHealthReport> GenerateWeeklyHealthReportAsync()
    {
        var report = new DwhHealthReport
        {
            ReportPeriod = GetLastWeekPeriod(),
            ExecutiveSummary = await GenerateExecutiveSummaryAsync(),
            PerformanceAnalysis = await GeneratePerformanceAnalysisAsync(),
            DataQualityAnalysis = await GenerateDataQualityAnalysisAsync(),
            CostAnalysis = await GenerateCostAnalysisAsync(),
            Recommendations = await GenerateWeeklyRecommendationsAsync(),
            ComparisonToPrevious = await CompareToPreviousWeekAsync()
        };

        return report;
    }

    public async Task<AutomatedReport> ScheduleAutomatedReportsAsync()
    {
        var schedules = new[]
        {
            new ReportSchedule
            {
                ReportType = ReportType.DailyHealth,
                Schedule = "0 8 * * *", // Daily at 8 AM
                Recipients = new[] { "dwh-team@company.com" },
                Format = ReportFormat.Html
            },
            new ReportSchedule
            {
                ReportType = ReportType.WeeklyPerformance,
                Schedule = "0 9 * * 1", // Weekly on Monday at 9 AM
                Recipients = new[] { "cto@company.com", "dwh-team@company.com" },
                Format = ReportFormat.Pdf
            },
            new ReportSchedule
            {
                ReportType = ReportType.MonthlyBusinessReview,
                Schedule = "0 10 1 * *", // Monthly on 1st at 10 AM
                Recipients = new[] { "executive-team@company.com" },
                Format = ReportFormat.Ppt
            }
        };

        foreach (var schedule in schedules)
        {
            await SetupReportScheduleAsync(schedule);
        }

        return new AutomatedReport { ScheduledReports = schedules.Length };
    }
}
```

---

## 🔧 **Автоматическое восстановление**

### **Self-healing механизмы**

```csharp
public class SelfHealingService
{
    public async Task<HealingResult> AttemptAutomaticHealingAsync(Alert alert)
    {
        var healingResult = new HealingResult
        {
            AlertId = alert.Id,
            HealingAttemptTime = DateTime.UtcNow,
            ActionsTaken = new List<HealingAction>()
        };

        // Определение типа проблемы и применение соответствующих действий
        switch (alert.Type)
        {
            case AlertType.HighCpuUsage:
                healingResult.ActionsTaken.Add(await ScaleComputeResourcesAsync());
                break;

            case AlertType.SlowQueries:
                healingResult.ActionsTaken.Add(await KillLongRunningQueriesAsync());
                healingResult.ActionsTaken.Add(await UpdateStatisticsAsync());
                break;

            case AlertType.StorageFull:
                healingResult.ActionsTaken.Add(await CleanupTempDataAsync());
                healingResult.ActionsTaken.Add(await ArchiveOldDataAsync());
                break;

            case AlertType.DataFreshness:
                healingResult.ActionsTaken.Add(await RestartDataIngestionAsync());
                break;
        }

        healingResult.Success = healingResult.ActionsTaken.All(a => a.Success);
        healingResult.Verification = await VerifyHealingAsync(alert);

        return healingResult;
    }

    public async Task SetupAutoHealingRulesAsync()
    {
        var rules = new[]
        {
            new AutoHealingRule
            {
                Condition = "cpu_usage > 90 for 5min",
                Actions = new[]
                {
                    new HealingAction { Type = "ScaleUp", Parameters = { ["scale"] = "2x" } },
                    new HealingAction { Type = "KillNonCriticalQueries" }
                },
                MaxAttempts = 3,
                CoolDownPeriod = TimeSpan.FromMinutes(10)
            },
            new AutoHealingRule
            {
                Condition = "query_timeout_count > 100 in 1h",
                Actions = new[]
                {
                    new HealingAction { Type = "UpdateStatistics" },
                    new HealingAction { Type = "RebuildIndexes" }
                },
                MaxAttempts = 2,
                CoolDownPeriod = TimeSpan.FromHours(1)
            }
        };

        foreach (var rule in rules)
        {
            await ConfigureAutoHealingRuleAsync(rule);
        }
    }
}
```

---

## 📊 **Ключевые метрики здоровья DWH**

### **Технические метрики:**

```csharp
public class TechnicalHealthMetrics
{
    public double QueryPerformanceScore { get; set; } // P95 execution time
    public double ResourceUtilizationScore { get; set; } // CPU/Memory/Disk usage
    public double AvailabilityScore { get; set; } // Uptime percentage
    public double DataFreshnessScore { get; set; } // Data latency
    public double ErrorRateScore { get; set; } // Failed queries percentage

    public double OverallTechnicalScore => new[]
    {
        QueryPerformanceScore,
        ResourceUtilizationScore,
        AvailabilityScore,
        DataFreshnessScore,
        ErrorRateScore
    }.Average();
}
```

### **Бизнес метрики:**

```csharp
public class BusinessHealthMetrics
{
    public double UserSatisfactionScore { get; set; } // User surveys
    public double DataQualityScore { get; set; } // Business data quality
    public double CostEfficiencyScore { get; set; } // Cost per query
    public double BusinessImpactScore { get; set; } // ROI measurement
    public double AdoptionRateScore { get; set; } // User adoption

    public double OverallBusinessScore => new[]
    {
        UserSatisfactionScore,
        DataQualityScore,
        CostEfficiencyScore,
        BusinessImpactScore,
        AdoptionRateScore
    }.Average();
}
```

---

## 🎯 **Best Practices мониторинга**

### **Для сбора метрик:**

- Реализуйте multi-level мониторинг (infrastructure, database, query, business)
- Используйте автоматическое обнаружение метрик
- Реализуйте efficient storage и retention policies
- Настройте корреляцию метрик

### **Для оповещений:**

- Избегайте alert fatigue через intelligent grouping
- Реализуйте escalation policies
- Настройке context-aware уведомления
- Регулярно review и настраивайте alert thresholds

### **Для анализа:**

- Создайте comprehensive dashboards для разных аудиторий
- Реализуйте automated reporting
- Используйте ML для anomaly detection
- Регулярно проводите performance reviews

---

## 🚨 **Anti-patterns**

1. **Monitoring everything** - сбор ненужных метрик
2. **Ignoring business metrics** - фокус только на технических метриках
3. **Alert storms** - слишком много несвязанных алертов
4. **No action on alerts** - алерты без последующих действий

---

## ✅ **Checklist мониторинга**

### **Ежедневно:**

- [ ] Review critical alerts
- [ ] Check system health dashboards
- [ ] Verify data freshness
- [ ] Monitor resource utilization

### **Еженедельно:**

- [ ] Review performance trends
- [ ] Analyze alert patterns
- [ ] Update monitoring thresholds
- [ ] Generate health reports

### **Ежемесячно:**

- [ ] Comprehensive system review
- [ ] Cost optimization analysis
- [ ] Capacity planning
- [ ] User satisfaction assessment

---

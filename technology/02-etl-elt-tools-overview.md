# 3.2 - Инструменты ETL/ELT: dbt, Airflow, Informatica

## Введение

Современные инструменты ETL/ELT превратили обработку данных из сложных кастомных решений в стандартизированные платформы. Рассмотрим ключевые инструменты и их применение.

## 📊 Сравнительная таблица инструментов

| Инструмент             | Тип            | Модель   | Язык     | Лучшее для             |
| ---------------------- | -------------- | -------- | -------- | ---------------------- |
| **dbt**                | Transformation | ELT      | SQL      | Data transformation    |
| **Airflow**            | Orchestration  | Workflow | Python   | Pipeline orchestration |
| **Informatica**        | Enterprise ETL | ETL/ELT  | GUI/Java | Enterprise integration |
| **Azure Data Factory** | Cloud ETL      | ETL/ELT  | GUI/.NET | Azure ecosystem        |
| **Talend**             | Hybrid ETL     | ETL/ELT  | Java/GUI | Data integration       |

---

## 🛠️ **dbt (Data Build Tool)**

### **Философия**

- "Transformation as code"
- SQL-центричный подход
- Version control и testing
- Documentation generation

### **Архитектура**

```bash
dbt Core + dbt Cloud
├── Models: SQL transformations
├── Tests: Data quality checks
├── Documentation: Auto-generated docs
└── Macros: Reusable SQL components
```

### **Пример реализации на C#**

```csharp
public class DbtService
{
    private readonly IDbtRunner _dbtRunner;

    public async Task RunDbtTransformationsAsync(string environment)
    {
        // dbt работает через командную строку
        var result = await _dbtRunner.ExecuteAsync(new DbtCommand
        {
            Command = "dbt run",
            Profile = environment,
            Target = "prod",
            Select = "tag:daily",
            FullRefresh = false
        });

        if (result.Success)
        {
            await RunDbtTestsAsync();
            await GenerateDocumentationAsync();
        }
    }

    public async Task<bool> ValidateDataQualityAsync()
    {
        var testResult = await _dbtRunner.ExecuteAsync(new DbtCommand
        {
            Command = "dbt test",
            Profile = "prod"
        });

        return testResult.Success && testResult.FailedTests == 0;
    }
}

// Пример генерации dbt моделей из C#
public class DbtModelGenerator
{
    public string GenerateStagingModel(string sourceTable, string[] columns)
    {
        return $@"
            {{{
                config(
                    materialized='view',
                    tags=['staging', 'daily']
                )
            }}}

            SELECT
                {string.Join(",\n                ", columns.Select(c => $"{c} as {c.ToLower()}"))}
            FROM {{{{ source('raw_sources', '{sourceTable}') }}}}
            WHERE _loaded_at >= CURRENT_DATE - 1";
    }

    public string GenerateFactModel(string name, string[] dimensions, string[] measures)
    {
        return $@"
            {{{
                config(
                    materialized='incremental',
                    unique_key='{name}_key',
                    on_schema_change='fail'
                )
            }}}

            SELECT
                {string.Join(",\n                ", dimensions)},
                {string.Join(",\n                ", measures)}
            FROM {{{{ ref('stg_{name}') }}}}
            {% if is_incremental() %}
            WHERE _loaded_at > (SELECT MAX(_loaded_at) FROM {{{{ this }}}})
            {% endif %}";
    }
}
```

### **dbt Testing Framework**

```csharp
public class DbtTestGenerator
{
    public string GenerateUniqueTest(string modelName, string column)
    {
        return $@"
            -- tests/unique_{modelName}_{column}.sql
            {{%
                test unique(model=ref('{modelName}'), column_name='{column}')
            %}}
                SELECT {column}
                FROM {{{{ model }}}}
                GROUP BY {column}
                HAVING COUNT(*) > 1
            {{% endtest %}}";
    }

    public string GenerateRelationshipTest(string modelName, string column, string parentModel)
    {
        return $@"
            -- tests/relationships/{modelName}_{column}.sql
            {{%
                test relationships(
                    model=ref('{modelName}'),
                    column_name='{column}',
                    to=ref('{parentModel}'),
                    field='{column}'
                )
            %}}
                SELECT child.{column}
                FROM {{{{ model }}}} as child
                LEFT JOIN {{{{ ref('{parentModel}') }}}} as parent
                    ON child.{column} = parent.{column}
                WHERE parent.{column} IS NULL
            {{% endtest %}}";
    }
}
```

---

## 🔄 **Apache Airflow**

### **Архитектура**

```bash
Airflow Architecture
├── Scheduler: Task orchestration
├── Executor: Task execution
├── Webserver: UI
└️── Metadata DB: State storage
```

### **Пример DAG на C#-подобном синтаксисе**

```csharp
// Conceptual representation of Airflow DAG in C# style
public class DataProcessingDag
{
    [Dag(
        dagId = "daily_data_pipeline",
        scheduleInterval = "0 2 * * *",
        startDate = "2024-01-01",
        catchup = false
    )]
    public async Task ExecuteDailyPipelineAsync()
    {
        // Extract tasks
        var extractSales = new PythonOperator(
            taskId: "extract_sales_data",
            pythonCallable: "extract_sales_from_api",
            opKwargs: new { date = "{{ ds }}" }
        );

        var extractCustomers = new PythonOperator(
            taskId: "extract_customer_data",
            pythonCallable: "extract_customers_from_db"
        );

        // Transform tasks
        var transformSales = new PythonOperator(
            taskId: "transform_sales_data",
            pythonCallable: "clean_and_transform_sales"
        );

        var transformCustomers = new PythonOperator(
            taskId: "transform_customer_data",
            pythonCallable: "enrich_customer_data"
        );

        // Load tasks
        var loadToDwh = new PythonOperator(
            taskId: "load_to_data_warehouse",
            pythonCallable: "load_transformed_data"
        );

        // Data quality checks
        var dataQualityCheck = new PythonOperator(
            taskId: "data_quality_validation",
            pythonCallable: "run_data_quality_tests"
        );

        // Define dependencies
        extractSales >> transformSales;
        extractCustomers >> transformCustomers;

        [BranchPythonOperator(taskId = "check_quality")]
        public string QualityCheckBranch()
        {
            var qualityPassed = RunQualityChecks();
            return qualityPassed ? "load_to_dwh" : "send_failure_alert";
        }

        [PythonOperator(taskId = "send_failure_alert")]
        public void SendFailureAlert()
        {
            // Send alert on data quality failure
            SendSlackAlert("Data quality checks failed!");
        }
    }
}
```

### **Интеграция Airflow с C#**

```csharp
public class AirflowIntegrationService
{
    private readonly IAirflowApiClient _airflowClient;

    public async Task<string> TriggerDagAsync(string dagId, Dictionary<string, object> config = null)
    {
        var response = await _airflowClient.TriggerDagAsync(dagId, new DagRun
        {
            Conf = config,
            ExecutionDate = DateTime.UtcNow
        });

        return response.DagRunId;
    }

    public async Task<DagRunStatus> GetDagStatusAsync(string dagId, string runId)
    {
        return await _airflowClient.GetDagRunStatusAsync(dagId, runId);
    }

    public async Task<bool> WaitForDagCompletionAsync(string dagId, string runId, TimeSpan timeout)
    {
        var startTime = DateTime.UtcNow;

        while (DateTime.UtcNow - startTime < timeout)
        {
            var status = await GetDagStatusAsync(dagId, runId);

            if (status == DagRunStatus.Success)
                return true;
            if (status == DagRunStatus.Failed)
                return false;

            await Task.Delay(TimeSpan.FromSeconds(30));
        }

        throw new TimeoutException($"DAG {dagId} run {runId} did not complete within timeout");
    }
}
```

---

## 🏢 **Informatica PowerCenter**

### **Архитектура**

```bash
Informatica Architecture
├── Repository: Metadata storage
├── Integration Service: Execution engine
├── Designer: Development tool
└️── Workflow Manager: Orchestration
```

### **Пример интеграции с C#**

```csharp
public class InformaticaIntegrationService
{
    private readonly IInformaticaClient _informaticaClient;

    public async Task<WorkflowExecutionResult> ExecuteInformaticaWorkflowAsync(
        string workflowName,
        Dictionary<string, string> parameters)
    {
        // Start workflow execution
        var executionId = await _informaticaClient.StartWorkflowAsync(
            workflowName, parameters);

        // Monitor execution
        var status = await MonitorWorkflowExecutionAsync(executionId);

        // Get execution details
        var details = await _informaticaClient.GetWorkflowDetailsAsync(executionId);

        return new WorkflowExecutionResult
        {
            ExecutionId = executionId,
            Status = status,
            Details = details,
            Success = status == WorkflowStatus.Completed
        };
    }

    public async Task<MappingConfiguration> GenerateSalesMappingAsync()
    {
        // Programmatic mapping configuration
        var mapping = new MappingConfiguration
        {
            Name = "MAP_SALES_TRANSFORM",
            SourceConnection = "SRC_SALES_DB",
            TargetConnection = "TGT_DWH"
        };

        // Source definition
        mapping.Source = new SourceDefinition
        {
            Name = "SALES_TRANSACTIONS",
            Query = "SELECT * FROM sales_transactions WHERE transaction_date >= ?"
        };

        // Transformations
        mapping.Transformations.Add(new ExpressionTransformation
        {
            Name = "EXP_CALCULATE_NET_SALES",
            Expression = "SALES_AMOUNT - DISCOUNT_AMOUNT"
        });

        mapping.Transformations.Add(new LookupTransformation
        {
            Name = "LKP_CUSTOMER_DETAILS",
            LookupSource = "DIM_CUSTOMER",
            JoinCondition = "CUSTOMER_ID = CUSTOMER_KEY"
        });

        // Target definition
        mapping.Target = new TargetDefinition
        {
            Name = "FACT_SALES",
            LoadOrder = 1,
            UpdateStrategy = UpdateStrategy.Insert
        };

        return mapping;
    }
}
```

---

## 🔄 **Azure Data Factory**

### **Архитектура**

```bash
ADF Architecture
├── Pipeline: Workflow definition
├── Activities: Processing steps
├── Datasets: Data structures
└️── Linked Services: Connections
```

### **Пример интеграции с C#**

```csharp
public class DataFactoryService
{
    private readonly DataFactoryManagementClient _adfClient;

    public async Task<string> CreateDataPipelineAsync(PipelineDefinition pipelineDef)
    {
        var pipeline = new PipelineResource
        {
            Activities = pipelineDef.Activities.Select(a => CreateActivity(a)).ToList(),
            Concurrency = pipelineDef.Concurrency,
            Variables = pipelineDef.Variables
        };

        var response = await _adfClient.Pipelines.CreateOrUpdateAsync(
            pipelineDef.ResourceGroup,
            pipelineDef.FactoryName,
            pipelineDef.Name,
            pipeline
        );

        return response.Name;
    }

    public async Task<PipelineRun> TriggerPipelineAsync(
        string factoryName,
        string pipelineName,
        Dictionary<string, object> parameters)
    {
        var runResponse = await _adfClient.Pipelines.CreateRunAsync(
            _resourceGroup, factoryName, pipelineName, parameters: parameters);

        // Monitor pipeline execution
        return await MonitorPipelineRunAsync(factoryName, runResponse.RunId);
    }

    private async Task<PipelineRun> MonitorPipelineRunAsync(string factoryName, string runId)
    {
        PipelineRun run;
        do
        {
            run = await _adfClient.PipelineRuns.GetAsync(_resourceGroup, factoryName, runId);
            await Task.Delay(TimeSpan.FromSeconds(15));
        }
        while (run.Status == "InProgress" || run.Status == "Queued");

        return run;
    }
}

// Пример создания ADF pipeline программно
public class SalesDataPipelineBuilder
{
    public PipelineDefinition BuildSalesPipeline()
    {
        return new PipelineDefinition
        {
            Name = "PL_SALES_DAILY_LOAD",
            Activities = new List<PipelineActivity>
            {
                new CopyActivity
                {
                    Name = "CopySalesFromSource",
                    Source = new SqlSource { SqlReaderQuery = "SELECT * FROM sales" },
                    Sink = new AzureSqlSink { TableName = "stg_sales" }
                },
                new StoredProcedureActivity
                {
                    Name = "TransformSalesData",
                    StoredProcedureName = "usp_TransformSales"
                },
                new DataQualityActivity
                {
                    Name = "ValidateDataQuality",
                    ValidationRules = GetDataQualityRules()
                }
            }
        };
    }
}
```

---

## 🎯 **Критерии выбора инструментов**

### **Выбирайте dbt если:**

- ✅ ELT архитектура с мощным Cloud DWH
- ✅ Команда сильна в SQL
- ✅ Требуется version control и testing
- ✅ Data transformation в приоритете

### **Выбирайте Airflow если:**

- ✅ Сложная оркестрация multiple systems
- ✅ Python-based разработка
- ✅ Требуется гибкость и кастомизация
- ✅ Open-source решение

### **Выбирайте Informatica если:**

- ✅ Enterprise-окружение с strict governance
- ✅ GUI-based разработка преобладает
- ✅ Сложная data integration
- ✅ Требуется enterprise support

### **Выбирайте ADF если:**

- ✅ Azure ecosystem
- ✅ Low-code/no-code подход
- ✅ Интеграция с Microsoft stack
- ✅ Managed service требования

---

## 🔧 **Гибридные подходы**

### **dbt + Airflow комбинация**

```csharp
public class DbtAirflowOrchestration
{
    public async Task ExecuteDataPipelineAsync()
    {
        // Airflow для оркестрации
        var dagRunId = await _airflowService.TriggerDagAsync("daily_data_pipeline");

        // Мониторинг выполнения
        var status = await _airflowService.WaitForDagCompletionAsync(
            "daily_data_pipeline", dagRunId, TimeSpan.FromHours(2));

        if (status)
        {
            // dbt для трансформаций
            await _dbtService.RunDbtTransformationsAsync("prod");

            // Data quality checks
            var qualityPassed = await _dbtService.ValidateDataQualityAsync();

            if (!qualityPassed)
            {
                throw new DataQualityException("Data quality checks failed after dbt run");
            }
        }
    }
}
```

---

## 📊 **Сравнение производительности**

| Инструмент      | Время разработки | Стоимость  | Гибкость   | Поддержка enterprise |
| --------------- | ---------------- | ---------- | ---------- | -------------------- |
| **dbt**         | 🟢 Быстрое       | 🟢 Низкая  | 🟢 Высокая | 🟡 Средняя           |
| **Airflow**     | 🟡 Среднее       | 🟢 Низкая  | 🟢 Высокая | 🟡 Средняя           |
| **Informatica** | 🔴 Медленное     | 🔴 Высокая | 🟡 Средняя | 🟢 Высокая           |
| **ADF**         | 🟡 Среднее       | 🟡 Средняя | 🟡 Средняя | 🟢 Высокая           |

---

## 🚨 **Anti-patterns**

1. **Использование Airflow для data transformation** - не его основная задача
2. **dbt без proper orchestration** - отсутствие end-to-end управления
3. **Enterprise tools для простых use cases** - over-engineering
4. **Ignoring team skillset** при выборе инструментов

---

## ✅ **Best Practices**

### **Для dbt:**

- Используйте modular model design
- Реализуйте comprehensive testing
- Генерируйте документацию автоматически
- Используйте tags для организации моделей

### **Для Airflow:**

- Создайте reusable components
- Реализуйте proper error handling
- Используйте XCom для межтаскового обмена данными
- Мониторьте performance DAG-ов

### **Для всех инструментов:**

- Реализуйте version control для всех конфигураций
- Создайте CI/CD pipelines для развертывания
- Документируйте data lineage
- Мониторьте performance и стоимость

---

**Следующий раздел:** [3.3 - Он-премис решения: Teradata, Exadata, ClickHouse](./03-on-prem-solutions.md)

# 📋 Таблица референсов кода в DWH Guide

## 🏗️ **РАЗДЕЛ 1: АРХИТЕКТУРА**

### **1.1 - Слои данных в DWH**

_Нет примеров кода_

### **1.2 - Методологии DWH**

_Нет примеров кода_

### **1.3 - Схемы моделирования**

_Нет примеров кода_

### **1.4 - Data Lakehouse vs Traditional DWH**

| Пример кода              | Язык | Назначение                           | Файл                         |
| ------------------------ | ---- | ------------------------------------ | ---------------------------- |
| `BigQueryService`        | C#   | Работа с Google BigQuery             | `01-cloud-dwh-comparison.md` |
| `BigQueryCostCalculator` | C#   | Расчет стоимости запросов в BigQuery | `01-cloud-dwh-comparison.md` |
| `SnowflakeService`       | C#   | Работа со Snowflake                  | `01-cloud-dwh-comparison.md` |
| `WarehouseManager`       | C#   | Управление виртуальными warehouses   | `01-cloud-dwh-comparison.md` |
| `RedshiftService`        | C#   | Работа с Amazon Redshift             | `01-cloud-dwh-comparison.md` |
| `RedshiftClusterManager` | C#   | Управление кластерами Redshift       | `01-cloud-dwh-comparison.md` |
| `SynapseService`         | C#   | Работа с Azure Synapse               | `01-cloud-dwh-comparison.md` |
| `SynapseResourceManager` | C#   | Управление ресурсами Synapse         | `01-cloud-dwh-comparison.md` |
| `MigrationService`       | C#   | Миграция в облако                    | `01-cloud-dwh-comparison.md` |
| `HybridDataArchitecture` | C#   | Гибридные архитектуры                | `01-cloud-dwh-comparison.md` |

---

## 🔄 **РАЗДЕЛ 2: ПРОЦЕССЫ И МОДЕЛИРОВАНИЕ**

### **2.1 - ETL vs ELT**

| Пример кода         | Язык | Назначение                | Файл                         |
| ------------------- | ---- | ------------------------- | ---------------------------- |
| `EtlPipeline`       | C#   | Классический ETL pipeline | `01-etl-vs-elt-paradigms.md` |
| `EltPipeline`       | C#   | Современный ELT pipeline  | `01-etl-vs-elt-paradigms.md` |
| `HybridPipeline`    | C#   | Гибридный ETL-T подход    | `01-etl-vs-elt-paradigms.md` |
| `ReverseEtlService` | C#   | Reverse ETL синхронизация | `01-etl-vs-elt-paradigms.md` |

### **2.2 - SCD Types**

| Пример кода         | Язык | Назначение               | Файл                   |
| ------------------- | ---- | ------------------------ | ---------------------- |
| `ScdType0Processor` | C#   | Type 0 - Retain Original | `02-scd-types-i-vi.md` |
| `ScdType1Processor` | C#   | Type 1 - Overwrite       | `02-scd-types-i-vi.md` |
| `ScdType2Processor` | C#   | Type 2 - Add New Row     | `02-scd-types-i-vi.md` |
| `ScdType3Processor` | C#   | Type 3 - Add New Column  | `02-scd-types-i-vi.md` |
| `ScdType4Processor` | C#   | Type 4 - History Table   | `02-scd-types-i-vi.md` |

### **2.3 - Факты и измерения**

| Пример кода                 | Язык | Назначение                  | Файл                              |
| --------------------------- | ---- | --------------------------- | --------------------------------- |
| `TransactionFactProcessor`  | C#   | Transaction fact tables     | `03-facts-dimensions-patterns.md` |
| `SnapshotFactProcessor`     | C#   | Periodic snapshot facts     | `03-facts-dimensions-patterns.md` |
| `AccumulatingFactProcessor` | C#   | Accumulating snapshot facts | `03-facts-dimensions-patterns.md` |
| `FactlessFactProcessor`     | C#   | Factless fact tables        | `03-facts-dimensions-patterns.md` |
| `CustomerBehaviorDimension` | C#   | Behavioral dimensions       | `03-facts-dimensions-patterns.md` |

### **2.4 - Data Quality и мониторинг**

| Пример кода              | Язык | Назначение                 | Файл                            |
| ------------------------ | ---- | -------------------------- | ------------------------------- |
| `DataQualityFramework`   | C#   | Фреймворк качества данных  | `04-data-quality-monitoring.md` |
| `CompletenessRule`       | C#   | Правило проверки полноты   | `04-data-quality-monitoring.md` |
| `CompletenessValidator`  | C#   | Валидатор полноты данных   | `04-data-quality-monitoring.md` |
| `AccuracyValidator`      | C#   | Валидатор точности данных  | `04-data-quality-monitoring.md` |
| `ConsistencyValidator`   | C#   | Валидатор консистентности  | `04-data-quality-monitoring.md` |
| `TimelinessMonitor`      | C#   | Мониторинг актуальности    | `04-data-quality-monitoring.md` |
| `DataQualityMetrics`     | C#   | Метрики качества данных    | `04-data-quality-monitoring.md` |
| `QualityReportGenerator` | C#   | Генератор отчетов качества | `04-data-quality-monitoring.md` |
| `DataQualityAlertSystem` | C#   | Система оповещений         | `04-data-quality-monitoring.md` |
| `CompletenessAlert`      | C#   | Алерт полноты данных       | `04-data-quality-monitoring.md` |
| `QualityAwareEtlProcess` | C#   | ETL с проверкой качества   | `04-data-quality-monitoring.md` |
| `CommonQualityIssues`    | C#   | Решение проблем качества   | `04-data-quality-monitoring.md` |

---

## 💻 **РАЗДЕЛ 3: ТЕХНОЛОГИЧЕСКИЙ СТЕК**

### **3.1 - Облачные DWH**

_Примеры уже перечислены в разделе 1.4_

### **3.2 - Инструменты ETL/ELT**

| Пример кода                     | Язык | Назначение                   | Файл                           |
| ------------------------------- | ---- | ---------------------------- | ------------------------------ |
| `DbtService`                    | C#   | Работа с dbt                 | `02-etl-elt-tools-overview.md` |
| `DbtModelGenerator`             | C#   | Генератор dbt моделей        | `02-etl-elt-tools-overview.md` |
| `DbtTestGenerator`              | C#   | Генератор dbt тестов         | `02-etl-elt-tools-overview.md` |
| `DataProcessingDag`             | C#   | Airflow DAG (концептуальный) | `02-etl-elt-tools-overview.md` |
| `AirflowIntegrationService`     | C#   | Интеграция с Airflow         | `02-etl-elt-tools-overview.md` |
| `InformaticaIntegrationService` | C#   | Работа с Informatica         | `02-etl-elt-tools-overview.md` |
| `DataFactoryService`            | C#   | Работа с Azure Data Factory  | `02-etl-elt-tools-overview.md` |
| `SalesDataPipelineBuilder`      | C#   | Построитель ADF пайплайнов   | `02-etl-elt-tools-overview.md` |
| `DbtAirflowOrchestration`       | C#   | Комбинация dbt + Airflow     | `02-etl-elt-tools-overview.md` |

### **3.3 - On-premise решения**

| Пример кода                  | Язык | Назначение                      | Файл                      |
| ---------------------------- | ---- | ------------------------------- | ------------------------- |
| `TeradataService`            | C#   | Работа с Teradata               | `03-on-prem-solutions.md` |
| `TeradataPerformanceMonitor` | C#   | Мониторинг Teradata             | `03-on-prem-solutions.md` |
| `ExadataService`             | C#   | Работа с Oracle Exadata         | `03-on-prem-solutions.md` |
| `ExadataStorageManager`      | C#   | Управление хранилищем Exadata   | `03-on-prem-solutions.md` |
| `ClickHouseService`          | C#   | Работа с ClickHouse             | `03-on-prem-solutions.md` |
| `ClickHouseClusterManager`   | C#   | Управление кластером ClickHouse | `03-on-prem-solutions.md` |
| `CloudMigrationService`      | C#   | Миграция в облако               | `03-on-prem-solutions.md` |
| `HybridDataArchitecture`     | C#   | Гибридные архитектуры           | `03-on-prem-solutions.md` |

---

## ⚙️ **РАЗДЕЛ 4: ОПЕРАЦИОННОЕ УПРАВЛЕНИЕ**

### **4.1 - Оптимизация производительности**

| Пример кода               | Язык | Назначение                      | Файл                             |
| ------------------------- | ---- | ------------------------------- | -------------------------------- |
| `QueryAnalyzer`           | C#   | Анализ запросов                 | `01-performance-optimization.md` |
| `JoinOptimizer`           | C#   | Оптимизация JOIN                | `01-performance-optimization.md` |
| `IndexManager`            | C#   | Управление индексами            | `01-performance-optimization.md` |
| `BitmapIndexService`      | C#   | Bitmap индексы                  | `01-performance-optimization.md` |
| `PartitionManager`        | C#   | Управление партициями           | `01-performance-optimization.md` |
| `ListPartitionService`    | C#   | List partitioning               | `01-performance-optimization.md` |
| `ClusteringService`       | C#   | Кластеризация данных            | `01-performance-optimization.md` |
| `StorageOptimizer`        | C#   | Оптимизация хранилища           | `01-performance-optimization.md` |
| `MaterializedViewService` | C#   | Материализованные представления | `01-performance-optimization.md` |
| `QueryCacheService`       | C#   | Кэширование запросов            | `01-performance-optimization.md` |
| `PerformanceMonitor`      | C#   | Мониторинг производительности   | `01-performance-optimization.md` |
| `AutomaticTuningService`  | C#   | Автоматическая настройка        | `01-performance-optimization.md` |
| `QueryOptimizationRules`  | C#   | Правила оптимизации             | `01-performance-optimization.md` |

### **4.2 - Data Governance и безопасность**

| Пример кода               | Язык | Назначение                 | Файл                             |
| ------------------------- | ---- | -------------------------- | -------------------------------- |
| `DataSecurityFramework`   | C#   | Фреймворк безопасности     | `02-data-governance-security.md` |
| `RowLevelSecurityService` | C#   | Row-Level Security         | `02-data-governance-security.md` |
| `EncryptionService`       | C#   | Шифрование данных          | `02-data-governance-security.md` |
| `RbacService`             | C#   | Role-Based Access Control  | `02-data-governance-security.md` |
| `DataLineageService`      | C#   | Отслеживание происхождения | `02-data-governance-security.md` |
| `DataCatalogService`      | C#   | Data Catalog               | `02-data-governance-security.md` |
| `GdprComplianceService`   | C#   | GDPR compliance            | `02-data-governance-security.md` |
| `DataRetentionService`    | C#   | Политики хранения данных   | `02-data-governance-security.md` |
| `AuditService`            | C#   | Аудит и мониторинг         | `02-data-governance-security.md` |

### **4.3 - Data Mesh и Data Fabric**

| Пример кода                     | Язык | Назначение                  | Файл                     |
| ------------------------------- | ---- | --------------------------- | ------------------------ |
| `DataMeshOrchestrator`          | C#   | Оркестрация Data Mesh       | `03-data-mesh-fabric.md` |
| `DataProductManager`            | C#   | Управление Data Products    | `03-data-mesh-fabric.md` |
| `DataProductObservability`      | C#   | Наблюдаемость Data Products | `03-data-mesh-fabric.md` |
| `SelfServeDataPlatform`         | C#   | Платформа самообслуживания  | `03-data-mesh-fabric.md` |
| `DataProductFactory`            | C#   | Фабрика Data Products       | `03-data-mesh-fabric.md` |
| `FederatedGovernanceService`    | C#   | Федеративное управление     | `03-data-mesh-fabric.md` |
| `ComputationalGovernanceEngine` | C#   | Вычислительное управление   | `03-data-mesh-fabric.md` |
| `DataFabricOrchestrator`        | C#   | Оркестрация Data Fabric     | `03-data-mesh-fabric.md` |
| `DataVirtualizationService`     | C#   | Виртуализация данных        | `03-data-mesh-fabric.md` |
| `DataMeshMigrationStrategy`     | C#   | Стратегия миграции          | `03-data-mesh-fabric.md` |
| `DataMeshMonitor`               | C#   | Мониторинг Data Mesh        | `03-data-mesh-fabric.md` |

### **4.4 - Мониторинг и метрики здоровья DWH**

| Пример кода                   | Язык | Назначение                     | Файл                       |
| ----------------------------- | ---- | ------------------------------ | -------------------------- |
| `PerformanceMetricsCollector` | C#   | Сбор метрик производительности | `04-monitoring-metrics.md` |
| `PerformanceAnalyzer`         | C#   | Анализ производительности      | `04-monitoring-metrics.md` |
| `DataQualityMonitor`          | C#   | Мониторинг качества данных     | `04-monitoring-metrics.md` |
| `DataFreshnessMonitor`        | C#   | Мониторинг актуальности данных | `04-monitoring-metrics.md` |
| `AlertingSystem`              | C#   | Система оповещений             | `04-monitoring-metrics.md` |
| `SmartAlertCorrelation`       | C#   | Корреляция алертов             | `04-monitoring-metrics.md` |
| `DashboardService`            | C#   | Сервис дашбордов               | `04-monitoring-metrics.md` |
| `ReportingService`            | C#   | Сервис отчетности              | `04-monitoring-metrics.md` |
| `SelfHealingService`          | C#   | Автоматическое восстановление  | `04-monitoring-metrics.md` |
| `TechnicalHealthMetrics`      | C#   | Технические метрики здоровья   | `04-monitoring-metrics.md` |
| `BusinessHealthMetrics`       | C#   | Бизнес метрики здоровья        | `04-monitoring-metrics.md` |

---

## 📊 **ОБНОВЛЕННАЯ СТАТИСТИКА ПО КОДУ**

### **Общая статистика:**

- **Всего примеров кода:** 91
- **Язык программирования:** 100% C#
- **Разделы с кодом:** 11 из 11 подразделов
- **Файлы с кодом:** 11 из 15 файлов

### **Распределение по типам:**

- **ETL/ELT процессы:** 8 примеров
- **Data Quality:** 12 примеров
- **Базы данных/платформы:** 16 примеров
- **Безопасность и Governance:** 9 примеров
- **Оптимизация производительности:** 13 примеров
- **Data Mesh/Fabric:** 11 примеров
- **Мониторинг и метрики:** 22 примера

### **Наиболее кодонасыщенные разделы:**

1. **Мониторинг и метрики** (22 примера)
2. **Data Quality и мониторинг** (12 примеров)
3. **Оптимизация производительности** (13 примеров)
4. **Data Mesh и Data Fabric** (11 примеров)
5. **Облачные DWH** (10 примеров)

### **Топ-5 самых сложных реализаций:**

1. `DataMeshOrchestrator` - полная оркестрация Data Mesh
2. `ComputationalGovernanceEngine` - вычислительное управление
3. `PerformanceMetricsCollector` - комплексный сбор метрик
4. `DataVirtualizationService` - виртуализация данных
5. `SelfHealingService` - автоматическое восстановление

---

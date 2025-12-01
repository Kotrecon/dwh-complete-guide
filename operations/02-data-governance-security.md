# 4.2 - Data Governance и безопасность

## Введение

Data Governance - это комплексный подход к управлению доступностью, usability, integrity и безопасностью данных в организации. Без эффективного governance даже самый технически совершенный DWH не сможет обеспечить доверие к данным.

## 📊 Компоненты Data Governance Framework

| Компонент         | Назначение                             | Ключевые практики                |
| ----------------- | -------------------------------------- | -------------------------------- |
| **Data Quality**  | Обеспечение точности и полноты данных  | Валидация, мониторинг, очистка   |
| **Data Security** | Защита от несанкционированного доступа | Шифрование, маскирование, RBAC   |
| **Data Lineage**  | Отслеживание происхождения данных      | Metadata management, трассировка |
| **Compliance**    | Соответствие регуляторным требованиям  | GDPR, HIPAA, SOX compliance      |
| **Data Catalog**  | Централизованный каталог данных        | Discovery, документация, поиск   |

---

## 🛡️ **Data Security Framework**

### **Архитектура безопасности**

```csharp
public class DataSecurityFramework
{
    private readonly IEncryptionService _encryptionService;
    private readonly IAccessControlService _accessControl;
    private readonly IAuditService _auditService;

    public async Task<SecurityResult> ApplySecurityPoliciesAsync(DataRequest request)
    {
        var result = new SecurityResult();

        // 1. Authentication & Authorization
        if (!await _accessControl.IsAuthorizedAsync(request.User, request.Operation))
        {
            result.IsAllowed = false;
            result.Reason = "Access denied";
            return result;
        }

        // 2. Data Masking для чувствительных данных
        request.Data = await ApplyDataMaskingAsync(request.Data, request.User);

        // 3. Row-level Security
        request.Data = await ApplyRowLevelSecurityAsync(request.Data, request.User);

        // 4. Audit logging
        await _auditService.LogDataAccessAsync(request);

        result.IsAllowed = true;
        return result;
    }
}
```

### **Row-Level Security (RLS)**

```csharp
public class RowLevelSecurityService
{
    public async Task ConfigureRLSAsync(string tableName, string securityPolicy)
    {
        // Создание security policy для RLS
        var rlsSql = $@"
            CREATE SECURITY POLICY {tableName}_SecurityPolicy
            ADD FILTER PREDICATE {securityPolicy}(user_id) ON {tableName}
            WITH (STATE = ON, SCHEMABINDING = ON)";

        await ExecuteNonQueryAsync(rlsSql);
    }

    public async Task<string> CreateDepartmentRLSPolicyAsync()
    {
        // Политика: пользователи видят только данные своего департамента
        return @"
            CREATE FUNCTION dbo.fn_securitypredicate(@department_id INT)
            RETURNS TABLE
            WITH SCHEMABINDING
            AS RETURN (
                SELECT 1 AS fn_securityresult
                WHERE @department_id =
                    (SELECT department_id
                     FROM dbo.user_departments
                     WHERE user_name = USER_NAME())
                OR USER_NAME() = 'admin'  -- Админы видят всё
            )";
    }

    public async Task ConfigureDynamicDataMaskingAsync()
    {
        // Динамическое маскирование чувствительных данных
        var maskingSql = @"
            ALTER TABLE customers
            ALTER COLUMN email ADD MASKED WITH (FUNCTION = 'email()');

            ALTER TABLE customers
            ALTER COLUMN phone ADD MASKED WITH (FUNCTION = 'partial(3, \"XXXX\", 2)');

            ALTER TABLE customers
            ALTER COLUMN ssn ADD MASKED WITH (FUNCTION = 'default()')";

        await ExecuteNonQueryAsync(maskingSql);
    }
}
```

---

## 🔐 **Шифрование данных**

### **Encryption at rest и in transit**

```csharp
public class EncryptionService
{
    private readonly IKeyVaultService _keyVault;

    public async Task<EncryptedData> EncryptSensitiveDataAsync(object sensitiveData, string keyName)
    {
        var encryptionKey = await _keyVault.GetKeyAsync(keyName);

        // Шифрование с использованием AES-256
        using var aes = Aes.Create();
        aes.Key = encryptionKey;

        using var encryptor = aes.CreateEncryptor();
        using var memoryStream = new MemoryStream();
        using var cryptoStream = new CryptoStream(memoryStream, encryptor, CryptoStreamMode.Write);

        var dataBytes = Encoding.UTF8.GetBytes(JsonConvert.SerializeObject(sensitiveData));
        await cryptoStream.WriteAsync(dataBytes, 0, dataBytes.Length);
        await cryptoStream.FlushFinalizeAsync();

        return new EncryptedData
        {
            EncryptedBytes = memoryStream.ToArray(),
            IV = aes.IV,
            KeyVersion = encryptionKey.Version
        };
    }

    public async Task ConfigureTDEAsync(string databaseName)
    {
        // Transparent Data Encryption для всей базы
        var tdeSql = $@"
            CREATE DATABASE ENCRYPTION KEY
            WITH ALGORITHM = AES_256
            ENCRYPTION BY SERVER CERTIFICATE MyServerCert;

            ALTER DATABASE {databaseName}
            SET ENCRYPTION ON";

        await ExecuteNonQueryAsync(tdeSql);
    }

    public async Task<string> GenerateAlwaysEncryptedKeysAsync()
    {
        // Always Encrypted для column-level encryption
        var alwaysEncryptedSql = @"
            CREATE COLUMN MASTER KEY MyColumnMasterKey
            WITH (KEY_STORE_PROVIDER_NAME = 'AZURE_KEY_VAULT',
                  KEY_PATH = 'https://myvault.vault.azure.net/keys/MyCMK/');

            CREATE COLUMN ENCRYPTION KEY MyColumnEncryptionKey
            WITH VALUES (
                COLUMN_MASTER_KEY = MyColumnMasterKey,
                ALGORITHM = 'RSA_OAEP',
                ENCRYPTED_VALUE = ... )";

        await ExecuteNonQueryAsync(alwaysEncryptedSql);
        return "Always Encrypted keys configured successfully";
    }
}
```

---

## 👥 **Управление доступом (RBAC)**

### **Role-Based Access Control**

```csharp
public class RbacService
{
    public async Task ConfigureDataRolesAsync()
    {
        var roles = new[]
        {
            new DataRole
            {
                Name = "DataViewer",
                Permissions = new[]
                {
                    Permission.Select,
                    Permission.Execute
                },
                DataScopes = new[] { "Sales", "Marketing" }
            },
            new DataRole
            {
                Name = "DataAnalyst",
                Permissions = new[]
                {
                    Permission.Select,
                    Permission.Execute,
                    Permission.CreateView
                },
                DataScopes = new[] { "Sales", "Marketing", "Finance" }
            },
            new DataRole
            {
                Name = "DataSteward",
                Permissions = Enum.GetValues<Permission>(),
                DataScopes = new[] { "*" } // Все области
            }
        };

        foreach (var role in roles)
        {
            await CreateRoleAsync(role);
        }
    }

    public async Task<bool> ValidateUserAccessAsync(string userName, string resource, Permission requiredPermission)
    {
        var userRoles = await GetUserRolesAsync(userName);
        var resourcePolicies = await GetResourcePoliciesAsync(resource);

        return userRoles.Any(role =>
            resourcePolicies.Any(policy =>
                policy.Roles.Contains(role) &&
                policy.Permissions.HasFlag(requiredPermission)));
    }

    public async Task<AccessReport> GenerateAccessReportAsync()
    {
        var report = new AccessReport
        {
            GeneratedAt = DateTime.UtcNow,
            UserAccess = new List<UserAccessInfo>()
        };

        var allUsers = await GetAllUsersAsync();

        foreach (var user in allUsers)
        {
            var accessInfo = new UserAccessInfo
            {
                UserName = user.UserName,
                Roles = await GetUserRolesAsync(user.UserName),
                LastAccess = await GetLastAccessTimeAsync(user.UserName),
                AccessedResources = await GetAccessedResourcesAsync(user.UserName, TimeSpan.FromDays(30))
            };

            report.UserAccess.Add(accessInfo);
        }

        return report;
    }
}
```

---

## 📜 **Data Lineage и Metadata Management**

### **Отслеживание происхождения данных**

```csharp
public class DataLineageService
{
    public async Task<DataLineage> TrackDataLineageAsync(string datasetName)
    {
        var lineage = new DataLineage
        {
            DatasetName = datasetName,
            Sources = new List<DataSource>(),
            Transformations = new List<TransformationStep>(),
            Destinations = new List<DataDestination>()
        };

        // Поиск источников данных
        lineage.Sources = await FindDataSourcesAsync(datasetName);

        // Отслеживание трансформаций
        lineage.Transformations = await FindTransformationsAsync(datasetName);

        // Определение потребителей данных
        lineage.Destinations = await FindDataConsumersAsync(datasetName);

        // Визуализация lineage
        lineage.LineageGraph = await GenerateLineageGraphAsync(lineage);

        return lineage;
    }

    public async Task<LineageImpact> AnalyzeImpactAsync(string changeDescription)
    {
        // Анализ impact изменения на downstream системы
        var impact = new LineageImpact
        {
            ChangeDescription = changeDescription,
            AffectedDatasets = new List<AffectedDataset>(),
            BreakingChanges = new List<BreakingChange>(),
            Recommendations = new List<string>()
        };

        // Поиск зависимостей
        var dependencies = await FindDownstreamDependenciesAsync();

        foreach (var dependency in dependencies)
        {
            var affectedDataset = new AffectedDataset
            {
                Name = dependency.DatasetName,
                ImpactLevel = await CalculateImpactLevelAsync(dependency),
                RequiredActions = await GenerateRequiredActionsAsync(dependency)
            };

            impact.AffectedDatasets.Add(affectedDataset);
        }

        return impact;
    }
}
```

### **Data Catalog**

```csharp
public class DataCatalogService
{
    public async Task<DataAsset> RegisterDataAssetAsync(DataAsset asset)
    {
        asset.Metadata = new DataAssetMetadata
        {
            CreatedAt = DateTime.UtcNow,
            CreatedBy = await GetCurrentUserAsync(),
            LastUpdated = DateTime.UtcNow,
            Version = 1,
            QualityScore = await CalculateQualityScoreAsync(asset)
        };

        // Добавление business glossary terms
        asset.BusinessTerms = await MapToBusinessGlossaryAsync(asset);

        // Классификация данных
        asset.Classification = await ClassifyDataAsync(asset);

        await _catalogRepository.AddAssetAsync(asset);
        return asset;
    }

    public async Task<List<DataAsset>> SearchDataAssetsAsync(SearchCriteria criteria)
    {
        var results = await _catalogRepository.SearchAsync(criteria);

        // Применение security фильтров
        results = await ApplySecurityFiltersAsync(results, criteria.User);

        // Ранжирование по релевантности
        results = RankByRelevance(results, criteria);

        return results;
    }

    public async Task<DataCatalogReport> GenerateCatalogHealthReportAsync()
    {
        var report = new DataCatalogReport
        {
            TotalAssets = await GetTotalAssetCountAsync(),
            DocumentedAssets = await GetDocumentedAssetCountAsync(),
            AssetsWithOwners = await GetAssetsWithOwnersCountAsync(),
            StaleAssets = await FindStaleAssetsAsync(),
            DataQualityIssues = await FindDataQualityIssuesAsync()
        };

        report.HealthScore = CalculateCatalogHealthScore(report);

        return report;
    }
}
```

---

## 📋 **Compliance и регуляторные требования**

### **GDPR Compliance**

```csharp
public class GdprComplianceService
{
    public async Task HandleDataSubjectRequestAsync(DataSubjectRequest request)
    {
        switch (request.RequestType)
        {
            case DataSubjectRequestType.Access:
                await HandleAccessRequestAsync(request);
                break;
            case DataSubjectRequestType.Erasure:
                await HandleErasureRequestAsync(request);
                break;
            case DataSubjectRequestType.Portability:
                await HandlePortabilityRequestAsync(request);
                break;
            case DataSubjectRequestType.Rectification:
                await HandleRectificationRequestAsync(request);
                break;
        }

        await LogRequestAsync(request);
    }

    private async Task HandleErasureRequestAsync(DataSubjectRequest request)
    {
        // Поиск всех данных субъекта
        var subjectData = await FindDataBySubjectAsync(request.SubjectIdentifier);

        foreach (var dataLocation in subjectData)
        {
            // Безопасное удаление или анонимизация
            await AnonymizeOrDeleteDataAsync(dataLocation, request);
        }

        // Подтверждение выполнения
        await GenerateErasureConfirmationAsync(request);
    }

    public async Task<GdprComplianceReport> GenerateComplianceReportAsync()
    {
        var report = new GdprComplianceReport
        {
            ReportDate = DateTime.UtcNow,
            DataProcessingActivities = await GetDataProcessingActivitiesAsync(),
            DataBreachIncidents = await GetDataBreachIncidentsAsync(TimeSpan.FromDays(365)),
            SubjectRequests = await GetDataSubjectRequestsAsync(TimeSpan.FromDays(90)),
            ComplianceGaps = await IdentifyComplianceGapsAsync()
        };

        return report;
    }
}
```

### **Data Retention Policies**

```csharp
public class DataRetentionService
{
    public async Task ApplyRetentionPoliciesAsync()
    {
        var policies = await GetRetentionPoliciesAsync();

        foreach (var policy in policies)
        {
            var expiredData = await FindExpiredDataAsync(policy);

            foreach (var data in expiredData)
            {
                switch (policy.DispositionAction)
                {
                    case DispositionAction.Archive:
                        await ArchiveDataAsync(data, policy);
                        break;
                    case DispositionAction.Delete:
                        await DeleteDataAsync(data, policy);
                        break;
                    case DispositionAction.Anonymize:
                        await AnonymizeDataAsync(data, policy);
                        break;
                }
            }
        }
    }

    public async Task<RetentionComplianceReport> CheckRetentionComplianceAsync()
    {
        var report = new RetentionComplianceReport
        {
            CheckDate = DateTime.UtcNow,
            Policies = new List<PolicyCompliance>()
        };

        var allPolicies = await GetRetentionPoliciesAsync();

        foreach (var policy in allPolicies)
        {
            var compliance = new PolicyCompliance
            {
                PolicyName = policy.Name,
                DataCategory = policy.DataCategory,
                IsCompliant = await CheckPolicyComplianceAsync(policy),
                NonCompliantItems = await FindNonCompliantDataAsync(policy),
                Recommendations = await GenerateComplianceRecommendationsAsync(policy)
            };

            report.Policies.Add(compliance);
        }

        return report;
    }
}
```

---

## 🔍 **Audit и Monitoring**

### **Comprehensive auditing**

```csharp
public class AuditService
{
    public async Task LogDataAccessAsync(DataAccessEvent accessEvent)
    {
        var auditRecord = new DataAccessAudit
        {
            Timestamp = DateTime.UtcNow,
            UserName = accessEvent.UserName,
            Operation = accessEvent.Operation,
            Resource = accessEvent.Resource,
            QueryText = accessEvent.QueryText,
            RowsAffected = accessEvent.RowsAffected,
            ClientIP = accessEvent.ClientIP,
            Success = accessEvent.Success
        };

        await _auditRepository.LogAccessAsync(auditRecord);
    }

    public async Task<List<SuspiciousActivity>> DetectSuspiciousActivitiesAsync()
    {
        var suspiciousActivities = new List<SuspiciousActivity>();

        // Поиск аномального доступа к данным
        var anomalousAccess = await DetectAnomalousAccessPatternsAsync();
        suspiciousActivities.AddRange(anomalousAccess);

        // Обнаружение попыток обхода security
        var securityBypassAttempts = await DetectSecurityBypassAttemptsAsync();
        suspiciousActivities.AddRange(securityBypassAttempts);

        // Мониторинг bulk data access
        var bulkAccess = await DetectBulkDataAccessAsync();
        suspiciousActivities.AddRange(bulkAccess);

        return suspiciousActivities;
    }

    public async Task<AuditReport> GenerateComplianceAuditReportAsync(DateTime startDate, DateTime endDate)
    {
        var report = new AuditReport
        {
            Period = $"{startDate:yyyy-MM-dd} to {endDate:yyyy-MM-dd}",
            TotalAccessEvents = await GetAccessEventCountAsync(startDate, endDate),
            UniqueUsers = await GetUniqueUserCountAsync(startDate, endDate),
            SensitiveDataAccess = await GetSensitiveDataAccessCountAsync(startDate, endDate),
            PolicyViolations = await GetPolicyViolationCountAsync(startDate, endDate),
            DataBreaches = await GetDataBreachCountAsync(startDate, endDate)
        };

        return report;
    }
}
```

---

## 🎯 **Best Practices Data Governance**

### **Организационные практики:**

- Назначьте Data Stewards для критичных данных
- Создайте Data Governance Council
- Реализуйте data ownership model
- Проводите регулярные training по безопасности данных

### **Технические практики:**

- Реализуйте principle of least privilege
- Используйте encryption both at rest и in transit
- Внедрите comprehensive auditing
- Автоматизируйте compliance checking

### **Операционные практики:**

- Регулярно проводите security assessments
- Обновляйте security policies
- Мониторьте compliance с регуляторными требованиями
- Документируйте data lineage для критичных datasets

---

## 🚨 **Anti-patterns**

1. **Over-governance** - слишком строгие политики мешают productivity
2. **Under-documentation** - отсутствие документации данных
3. **Security through obscurity** - reliance на скрытии вместо реальной защиты
4. **Ignoring data lineage** - невозможность отследить происхождение данных

---

## ✅ **Checklist Data Governance**

### **Ежеквартально:**

- [ ] Review и update security policies
- [ ] Audit user access rights
- [ ] Review compliance с регуляторными требованиями
- [ ] Update data classification schemas

### **Ежемесячно:**

- [ ] Review data quality metrics
- [ ] Audit data access patterns
- [ ] Update data catalog
- [ ] Review и cleanup устаревших данных

### **Ежедневно:**

- [ ] Мониторинг security alerts
- [ ] Review audit logs
- [ ] Проверка data access requests
- [ ] Мониторинг data quality issues

---

**Следующий раздел:** [4.3 - Data Mesh и Data Fabric](./03-data-mesh-fabric.md)

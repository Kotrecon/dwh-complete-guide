# 2.3 - Факты и измерения: паттерны моделирования

## Введение

Факты и измерения - фундаментальные строительные блоки dimensional modeling. Правильное проектирование этих сущностей критически важно для производительности и удобства использования DWH.

## 📊 Типы фактовых таблиц

| Тип                       | Назначение          | Гранулярность | Примеры                |
| ------------------------- | ------------------- | ------------- | ---------------------- |
| **Transaction**           | Отдельные события   | Высокая       | Продажи, транзакции    |
| **Periodic Snapshot**     | Снимки на период    | Средняя       | Балансы на конец дня   |
| **Accumulating Snapshot** | Прогресс процесса   | Изменяемая    | Жизненный цикл заказа  |
| **Factless Fact**         | Регистрация событий | Высокая       | Посещения, присутствия |

---

## 🎯 **Transaction Fact Tables**

### **Характеристики**

- Одна строка = одно событие
- Высокая гранулярность
- Добавление только новых записей

### **Структура**

```sql
CREATE TABLE fact_sales (
    sales_key BIGINT PRIMARY KEY,
    date_key INT,          -- FK to dim_date
    customer_key INT,      -- FK to dim_customer
    product_key INT,       -- FK to dim_product
    store_key INT,         -- FK to dim_store
    sales_amount DECIMAL(10,2),
    quantity INT,
    discount_amount DECIMAL(10,2),
    net_sales_amount DECIMAL(10,2)
);
```

### **Пример C# ETL**

```csharp
public class TransactionFactProcessor
{
    public void ProcessSalesTransaction(SalesTransaction transaction)
    {
        var fact = new SalesFact
        {
            DateKey = _dateDimension.GetKey(transaction.TransactionDate),
            CustomerKey = _customerDimension.GetKey(transaction.CustomerId),
            ProductKey = _productDimension.GetKey(transaction.ProductId),
            StoreKey = _storeDimension.GetKey(transaction.StoreId),
            SalesAmount = transaction.Amount,
            Quantity = transaction.Quantity,
            DiscountAmount = transaction.Discount,
            NetSalesAmount = transaction.Amount - transaction.Discount
        };

        _factRepository.Insert(fact);
    }
}
```

---

## 📅 **Periodic Snapshot Fact Tables**

### **Характеристики**

- Регулярные снимки состояния
- Фиксированная периодичность
- Могут обновляться

### **Структура**

```sql
CREATE TABLE fact_account_balance (
    balance_key BIGINT PRIMARY KEY,
    date_key INT,              -- FK to dim_date (end of period)
    account_key INT,           -- FK to dim_account
    customer_key INT,          -- FK to dim_customer
    opening_balance DECIMAL(15,2),
    closing_balance DECIMAL(15,2),
    total_deposits DECIMAL(15,2),
    total_withdrawals DECIMAL(15,2)
);
```

### **Пример C# ETL**

```csharp
public class SnapshotFactProcessor
{
    public void GenerateDailyBalances(DateTime snapshotDate)
    {
        var accounts = _accountService.GetAllAccounts();

        foreach (var account in accounts)
        {
            var balance = _balanceCalculator.CalculateDailyBalance(account, snapshotDate);

            var fact = new AccountBalanceFact
            {
                DateKey = _dateDimension.GetKey(snapshotDate),
                AccountKey = _accountDimension.GetKey(account.Id),
                CustomerKey = _customerDimension.GetKey(account.CustomerId),
                OpeningBalance = balance.Opening,
                ClosingBalance = balance.Closing,
                TotalDeposits = balance.Deposits,
                TotalWithdrawals = balance.Withdrawals
            };

            _factRepository.InsertOrUpdate(fact);
        }
    }
}
```

---

## 🔄 **Accumulating Snapshot Fact Tables**

### **Характеристики**

- Отслеживание прогресса процесса
- Обновление строк по мере движения
- Множество дат в одной строке

### **Структура**

```sql
CREATE TABLE fact_order_fulfillment (
    order_key BIGINT PRIMARY KEY,
    order_id INT,
    customer_key INT,
    product_key INT,
    order_date_key INT,
    ship_date_key INT,         -- Обновляется при отгрузке
    delivery_date_key INT,     -- Обновляется при доставке
    cancel_date_key INT,       -- Обновляется при отмене
    order_amount DECIMAL(10,2),
    status VARCHAR(20)
);
```

### **Пример C# ETL**

```csharp
public class AccumulatingFactProcessor
{
    public void UpdateOrderStatus(OrderStatusUpdate update)
    {
        var existingFact = _factRepository.GetOrderFact(update.OrderId);

        if (existingFact == null)
        {
            // Создание новой записи
            existingFact = new OrderFulfillmentFact
            {
                OrderKey = GenerateKey(),
                OrderId = update.OrderId,
                CustomerKey = _customerDimension.GetKey(update.CustomerId),
                ProductKey = _productDimension.GetKey(update.ProductId),
                OrderDateKey = _dateDimension.GetKey(update.OrderDate),
                Status = "Ordered"
            };
            _factRepository.Insert(existingFact);
        }

        // Обновление дат по мере прогресса
        switch (update.Status)
        {
            case "Shipped":
                existingFact.ShipDateKey = _dateDimension.GetKey(update.EventDate);
                existingFact.Status = "Shipped";
                break;
            case "Delivered":
                existingFact.DeliveryDateKey = _dateDimension.GetKey(update.EventDate);
                existingFact.Status = "Delivered";
                break;
            case "Cancelled":
                existingFact.CancelDateKey = _dateDimension.GetKey(update.EventDate);
                existingFact.Status = "Cancelled";
                break;
        }

        _factRepository.Update(existingFact);
    }
}
```

---

## 🔲 **Factless Fact Tables**

### **Характеристики**

- Нет числовых метрик
- Регистрация событий или отношений
- Только ключи измерений

### **Структура**

```sql
CREATE TABLE fact_student_attendance (
    attendance_key BIGINT PRIMARY KEY,
    date_key INT,
    student_key INT,
    class_key INT,
    professor_key INT,
    is_present BOOLEAN
);
```

### **Пример C# ETL**

```csharp
public class FactlessFactProcessor
{
    public void RecordStudentAttendance(AttendanceRecord record)
    {
        var fact = new StudentAttendanceFact
        {
            DateKey = _dateDimension.GetKey(record.AttendanceDate),
            StudentKey = _studentDimension.GetKey(record.StudentId),
            ClassKey = _classDimension.GetKey(record.ClassId),
            ProfessorKey = _professorDimension.GetKey(record.ProfessorId),
            IsPresent = record.IsPresent
        };

        _factRepository.Insert(fact);
    }
}
```

---

## 📐 **Типы измерений**

### **Conformed Dimensions**

```sql
-- Единые измерения для всех фактов
CREATE TABLE dim_date (
    date_key INT PRIMARY KEY,
    date_value DATE,
    day_of_week INT,
    month_name VARCHAR(20),
    quarter INT,
    year INT,
    is_weekend BOOLEAN
);
```

### **Junk Dimensions**

```sql
-- Группировка мелких атрибутов
CREATE TABLE dim_transaction_attributes (
    attribute_key INT PRIMARY KEY,
    payment_method VARCHAR(20),
    channel VARCHAR(20),
    promotion_applied BOOLEAN,
    is_online BOOLEAN
);
```

### **Mini-Dimensions**

```sql
-- Для часто меняющихся атрибутов
CREATE TABLE dim_customer_demographics (
    demographics_key INT PRIMARY KEY,
    age_group VARCHAR(10),
    income_bracket VARCHAR(15),
    education_level VARCHAR(20)
);
```

### **Degenerate Dimensions**

```sql
-- Остаются в фактовой таблице
CREATE TABLE fact_sales (
    sales_key BIGINT PRIMARY KEY,
    -- ... другие ключи
    order_number VARCHAR(20),  -- Degenerate dimension
    invoice_number VARCHAR(20) -- Degenerate dimension
);
```

---

## 🎯 **Паттерны проектирования**

### **Role-Playing Dimensions**

```sql
-- Одно измерение, несколько ролей
CREATE TABLE fact_shipments (
    shipment_key BIGINT PRIMARY KEY,
    order_date_key INT,    -- dim_date как order date
    ship_date_key INT,     -- dim_date как ship date
    delivery_date_key INT, -- dim_date как delivery date
    -- ... другие поля
);
```

### **Heterogeneous Products**

```sql
-- Подтипы продуктов с разными атрибутами
CREATE TABLE dim_product (
    product_key INT PRIMARY KEY,
    product_type VARCHAR(20),
    -- Общие атрибуты
    product_name VARCHAR(100),
    brand VARCHAR(50),
    -- Специфичные атрибуты (могут быть NULL)
    book_author VARCHAR(100),      -- Для книг
    electronics_warranty_months INT, -- Для электроники
    clothing_size VARCHAR(10)      -- Для одежды
);
```

### **Behavioral Dimensions**

```csharp
// Измерения на основе поведения
public class CustomerBehaviorDimension
{
    public int CustomerKey { get; set; }
    public string CustomerSegment { get; set; }
    public string PurchaseFrequency { get; set; } // "High", "Medium", "Low"
    public decimal LifetimeValue { get; set; }
    public DateTime LastPurchaseDate { get; set; }

    public static CustomerBehaviorDimension CreateFromFacts(
        IEnumerable<SalesFact> salesFacts)
    {
        var totalSpent = salesFacts.Sum(f => f.NetSalesAmount);
        var purchaseCount = salesFacts.Count();
        var lastPurchase = salesFacts.Max(f => f.DateKey);

        return new CustomerBehaviorDimension
        {
            CustomerSegment = CalculateSegment(totalSpent),
            PurchaseFrequency = CalculateFrequency(purchaseCount),
            LifetimeValue = totalSpent,
            LastPurchaseDate = _dateDimension.GetDate(lastPurchase)
        };
    }
}
```

---

## 📊 **Best Practices проектирования**

### **Гранулярность фактов**

```csharp
// ХОРОШО: Низкая гранулярность
public class LowGranularityFact
{
    public int DateKey { get; set; }
    public int CustomerKey { get; set; }
    public int ProductKey { get; set; }
    public int StoreKey { get; set; }
    public decimal SalesAmount { get; set; }
}

// ПЛОХО: Высокая гранулярность (агрегаты)
public class HighGranularityFact
{
    public int MonthKey { get; set; }
    public int RegionKey { get; set; }
    public decimal MonthlySales { get; set; } // Агрегат!
}
```

### **Оптимизация производительности**

```sql
-- Кластеризованные индексы
CREATE CLUSTERED INDEX IX_fact_sales_date
ON fact_sales (date_key);

-- Некластеризованные индексы
CREATE INDEX IX_fact_sales_customer
ON fact_sales (customer_key)
INCLUDE (sales_amount, quantity);

-- Partitioning по датам
CREATE PARTITION FUNCTION pf_sales_date (DATE)
AS RANGE RIGHT FOR VALUES (
    '2023-01-01', '2023-04-01',
    '2023-07-01', '2023-10-01'
);
```

---

## 🚨 **Anti-patterns**

1. **Факты с высокой гранулярностью** - потеря детализации
2. **Измерения с избыточными атрибутами** - нарушение нормализации
3. **Отсутствие surrogate keys** - проблемы с историзацией
4. **Смешение типов фактов** в одной таблице

---

## ✅ **Best Practices**

### **Для фактов:**

- Используйте surrogate keys для связей
- Храните метрики в естественных единицах
- Реализуйте null handling для опциональных измерений
- Используйте составные первичные ключи где уместно

### **Для измерений:**

- Создавайте индексы на часто используемых атрибутах
- Реализуйте SCD стратегии для изменяющихся атрибутов
- Используйте junk dimensions для группировки флагов
- Создавайте conformed dimensions для переиспользования

### **Для производительности:**

- Partitioning больших фактовых таблиц
- Columnstore индексы для аналитических запросов
- Оптимизация порядка столбцов в таблицах
- Регулярное обновление статистик

---

**Следующий раздел:** [2.4 - Data Quality и мониторинг](./04-data-quality-monitoring.md)

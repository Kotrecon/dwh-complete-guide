# 2.2 - Медленно меняющиеся измерения (SCD Types I-VI)

## Введение

Slowly Changing Dimensions (SCD) - это методологии управления изменениями в данных измерений, которые меняются со временем. Правильный выбор типа SCD критически важен для исторической аналитики.

## 📊 Обзор типов SCD

| Тип        | Название        | Стратегия                        | Историзация     | Сложность  |
| ---------- | --------------- | -------------------------------- | --------------- | ---------- |
| **Type 0** | Retain Original | Сохранение исходных значений     | ❌ Нет          | 🟢 Низкая  |
| **Type 1** | Overwrite       | Перезапись старых данных         | ❌ Нет          | 🟢 Низкая  |
| **Type 2** | Add New Row     | Добавление новой строки          | ✅ Полная       | 🟡 Средняя |
| **Type 3** | Add New Column  | Добавление нового столбца        | ✅ Ограниченная | 🟡 Средняя |
| **Type 4** | History Table   | Отдельная таблица истории        | ✅ Полная       | 🔴 Высокая |
| **Type 5** | Mini-Dimension  | Вынос часто меняющихся атрибутов | ✅ Полная       | 🔴 Высокая |
| **Type 6** | Hybrid          | Комбинация Type 1 + 2 + 3        | ✅ Полная       | 🔴 Высокая |

---

## 🔄 **Type 0 - Retain Original**

### **Стратегия**

- Атрибуты никогда не меняются
- Сохраняются исходные значения

### **Использование**

```sql
CREATE TABLE dim_customer_type0 (
    customer_key INT PRIMARY KEY,
    customer_id INT,
    original_region VARCHAR(50), -- Никогда не меняется
    original_signup_date DATE
);
```

### **Пример**

```csharp
public class ScdType0Processor
{
    public void ProcessCustomer(Customer customer)
    {
        // Данные никогда не обновляются
        var dimension = new CustomerDimension
        {
            CustomerKey = customer.Id,
            CustomerId = customer.Id,
            OriginalRegion = customer.InitialRegion, // Сохраняется навсегда
            OriginalSignupDate = customer.SignupDate
        };

        // Только вставка, без обновлений
        _repository.InsertDimension(dimension);
    }
}
```

---

## ✏️ **Type 1 - Overwrite**

### **Стратегия**

- Старые данные перезаписываются
- История теряется

### **Использование**

```sql
CREATE TABLE dim_customer_type1 (
    customer_key INT PRIMARY KEY,
    customer_id INT,
    customer_name VARCHAR(100), -- Может перезаписываться
    current_region VARCHAR(50),
    last_updated DATETIME
);
```

### **Пример**

```csharp
public class ScdType1Processor
{
    public void UpdateCustomer(Customer customer)
    {
        var existing = _repository.GetCustomer(customer.Id);

        if (existing != null)
        {
            // ПЕРЕЗАПИСЬ - история теряется
            existing.CustomerName = customer.Name;
            existing.CurrentRegion = customer.Region;
            existing.LastUpdated = DateTime.UtcNow;

            _repository.UpdateDimension(existing);
        }
        else
        {
            // Новая запись
            _repository.InsertDimension(new CustomerDimension(customer));
        }
    }
}
```

---

## 🆕 **Type 2 - Add New Row**

### **Стратегия**

- Новая строка для каждого изменения
- Полная историзация

### **Структура таблицы**

```sql
CREATE TABLE dim_customer_type2 (
    customer_key INT PRIMARY KEY,
    customer_id INT,
    customer_name VARCHAR(100),
    region VARCHAR(50),
    start_date DATE,
    end_date DATE,
    is_current BOOLEAN
);
```

### **Пример реализации**

```csharp
public class ScdType2Processor
{
    public void HandleCustomerChange(Customer customer)
    {
        var currentRecord = _repository.GetCurrentCustomer(customer.Id);

        if (currentRecord != null && HasChanges(currentRecord, customer))
        {
            // Закрытие текущей записи
            currentRecord.EndDate = DateTime.UtcNow.Date;
            currentRecord.IsCurrent = false;
            _repository.UpdateDimension(currentRecord);

            // Создание новой записи
            var newRecord = new CustomerDimension
            {
                CustomerKey = GenerateNewKey(),
                CustomerId = customer.Id,
                CustomerName = customer.Name,
                Region = customer.Region,
                StartDate = DateTime.UtcNow.Date,
                EndDate = DateTime.MaxValue,
                IsCurrent = true
            };

            _repository.InsertDimension(newRecord);
        }
        else if (currentRecord == null)
        {
            // Первая запись
            _repository.InsertDimension(new CustomerDimension(customer));
        }
    }

    private bool HasChanges(CustomerDimension existing, Customer current)
    {
        return existing.CustomerName != current.Name ||
               existing.Region != current.Region;
    }
}
```

---

## 🗂️ **Type 3 - Add New Column**

### **Стратегия**

- Добавление колонок для предыдущих значений
- Ограниченная история

### **Структура таблицы**

```sql
CREATE TABLE dim_customer_type3 (
    customer_key INT PRIMARY KEY,
    customer_id INT,
    current_region VARCHAR(50),
    previous_region VARCHAR(50), -- Только одно предыдущее значение
    original_region VARCHAR(50),
    last_updated DATETIME
);
```

### **Пример**

```csharp
public class ScdType3Processor
{
    public void UpdateCustomerRegion(Customer customer)
    {
        var existing = _repository.GetCustomer(customer.Id);

        if (existing != null && existing.CurrentRegion != customer.Region)
        {
            // Сдвиг истории
            existing.PreviousRegion = existing.CurrentRegion;
            existing.CurrentRegion = customer.Region;
            existing.LastUpdated = DateTime.UtcNow;

            _repository.UpdateDimension(existing);
        }
    }
}
```

---

## 📚 **Type 4 - History Table**

### **Стратегия**

- Основная таблица: текущие данные
- Отдельная таблица: полная история

### **Структура**

```sql
-- Текущие данные
CREATE TABLE dim_customer_current (
    customer_key INT PRIMARY KEY,
    customer_id INT,
    customer_name VARCHAR(100),
    current_region VARCHAR(50)
);

-- История изменений
CREATE TABLE dim_customer_history (
    history_key INT PRIMARY KEY,
    customer_id INT,
    customer_name VARCHAR(100),
    region VARCHAR(50),
    effective_date DATE,
    expired_date DATE
);
```

### **Пример**

```csharp
public class ScdType4Processor
{
    public void ProcessCustomerUpdate(Customer customer)
    {
        var current = _repository.GetCurrentCustomer(customer.Id);

        if (current != null && HasChanges(current, customer))
        {
            // Добавление в историю
            var historyRecord = new CustomerHistory
            {
                CustomerId = customer.Id,
                CustomerName = current.CustomerName,
                Region = current.Region,
                EffectiveDate = current.LastUpdated,
                ExpiredDate = DateTime.UtcNow
            };
            _repository.InsertHistory(historyRecord);

            // Обновление текущей записи
            current.CustomerName = customer.Name;
            current.Region = customer.Region;
            current.LastUpdated = DateTime.UtcNow;
            _repository.UpdateCurrent(current);
        }
    }
}
```

---

## 🎯 **Type 5 - Mini-Dimension**

### **Стратегия**

- Вынос часто меняющихся атрибутов в отдельную таблицу
- Связь через FK

### **Структура**

```sql
-- Основное измерение
CREATE TABLE dim_customer (
    customer_key INT PRIMARY KEY,
    customer_id INT,
    customer_name VARCHAR(100),
    birth_date DATE
);

-- Мини-измерение для демографии
CREATE TABLE dim_customer_demographics (
    demographics_key INT PRIMARY KEY,
    customer_key INT,
    age_group VARCHAR(20),
    income_bracket VARCHAR(20),
    marital_status VARCHAR(10),
    effective_date DATE,
    FOREIGN KEY (customer_key) REFERENCES dim_customer(customer_key)
);
```

---

## 🔄 **Type 6 - Hybrid Approach**

### **Стратегия**

- Комбинация Type 1 + 2 + 3
- Текущие значения + история + предыдущее значение

### **Структура**

```sql
CREATE TABLE dim_customer_type6 (
    customer_key INT PRIMARY KEY,
    customer_id INT,
    -- Type 1: Текущие значения
    current_region VARCHAR(50),
    -- Type 2: Историзация
    start_date DATE,
    end_date DATE,
    is_current BOOLEAN,
    -- Type 3: Предыдущее значение
    previous_region VARCHAR(50)
);
```

---

## 🎯 **Критерии выбора типа SCD**

### **Выбирайте Type 1 если:**

- ✅ История изменений не важна
- ✅ Простота реализации в приоритете
- ✅ Высокая частота изменений

### **Выбирайте Type 2 если:**

- ✅ Требуется полная историзация
- ✅ Аналитика "как было на дату"
- ✅ Умеренная частота изменений

### **Выбирайте Type 3 если:**

- ✅ Нужна ограниченная история (только предыдущее значение)
- ✅ Простота запросов важнее полной истории

### **Выбирайте Type 4+ если:**

- ✅ Очень высокая частота изменений
- ✅ Сложные требования к производительности
- ✅ Экспертная команда data engineers

---

## 📊 **Производительность и стоимость**

| Тип SCD    | Размер хранилища | Сложность ETL | Производительность запросов |
| ---------- | ---------------- | ------------- | --------------------------- |
| **Type 1** | 🟢 Низкий        | 🟢 Низкая     | 🟢 Высокая                  |
| **Type 2** | 🔴 Высокий       | 🟡 Средняя    | 🟡 Средняя                  |
| **Type 3** | 🟡 Средний       | 🟢 Низкая     | 🟢 Высокая                  |
| **Type 4** | 🔴 Высокий       | 🔴 Высокая    | 🟡 Средняя                  |
| **Type 6** | 🔴 Высокий       | 🔴 Высокая    | 🟡 Средняя                  |

---

## 🚨 **Anti-patterns**

1. **Type 2 для часто меняющихся атрибутов** - взрывной рост таблицы
2. **Type 1 для критичных исторических данных** - потеря истории
3. **Смешение типов в одной таблице** без четкой стратегии
4. **Игнорирование производительности** при выборе типа

---

## ✅ **Best Practices**

### **Для Type 2:**

- Используйте surrogate keys для новых версий
- Реализуйте эффективные индексы (customer_id + is_current)
- Рассмотрите partitioning по датам

### **Для всех типов:**

- Документируйте стратегию SCD для каждого атрибута
- Реализуйте мониторинг роста таблиц
- Планируйте архивацию старых записей

### **Гибридные подходы:**

- Используйте Type 6 для критичных атрибутов
- Комбинируйте типы в рамках одного измерения
- Тестируйте производительность на реалистичных данных

---

**Следующий раздел:** [2.3 - Факты и измерения: паттерны моделирования](./03-facts-dimensions-patterns.md)

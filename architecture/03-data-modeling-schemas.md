# 1.3 - Схемы моделирования данных: "Звезда", "Снежинка", "Галактика"

## Введение

Схемы моделирования данных определяют организацию таблиц в DWH и напрямую влияют на производительность, гибкость и удобство использования. Рассмотрим три основные схемы.

## 📊 Сравнительная таблица схем

| Аспект                 | Звезда (Star)     | Снежинка (Snowflake)     | Галактика (Galaxy) |
| ---------------------- | ----------------- | ------------------------ | ------------------ |
| **Нормализация**       | Денормализованная | Частично нормализованная | Гибридная          |
| **Производительность** | 🟢 Высокая        | 🟡 Средняя               | 🟢 Высокая         |
| **Гибкость**           | 🔴 Низкая         | 🟢 Высокая               | 🟡 Средняя         |
| **Простота**           | 🟢 Высокая        | 🟡 Средняя               | 🔴 Низкая          |
| **Размер хранилища**   | 🔴 Большой        | 🟢 Малый                 | 🟡 Средний         |
| **Сложность ETL**      | 🟢 Низкая         | 🔴 Высокая               | 🟡 Средняя         |

---

## ⭐ **Схема "Звезда" (Star Schema)**

### **Архитектура**

```sql
fact_sales (Фактовая таблица)
├── date_key (FK → dim_date)
├── customer_key (FK → dim_customer)
├── product_key (FK → dim_product)
├── store_key (FK → dim_store)
├── sales_amount
└── quantity

dim_customer (Измерение)
├── customer_key (PK)
├── customer_name
├── city
├── region
└── country

dim_product
├── product_key (PK)
├── product_name
├── category
└── brand

dim_date
├── date_key (PK)
├── date
├── month
├── quarter
└── year
```

### **Преимущества**

```sql
-- Простые запросы с JOIN по 1 уровню
SELECT
    d.year,
    p.category,
    SUM(f.sales_amount) as total_sales
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_product p ON f.product_key = p.product_key
GROUP BY d.year, p.category;
```

### **Недостатки**

- Избыточность данных (дублирование атрибутов)
- Сложность управления изменениями иерархий
- Ограниченная гибкость для ad-hoc анализа

### **Best Use Cases**

- Отчетность с предопределенными запросами
- Data Marts для конкретных бизнес-процессов
- Системы с высокими требованиями к производительности

---

## ❄️ **Схема "Снежинка" (Snowflake Schema)**

### **Архитектура**

```sql
fact_sales
├── date_key (FK → dim_date)
├── customer_key (FK → dim_customer)
├── product_key (FK → dim_product)
├── store_key (FK → dim_store)
├── sales_amount
└── quantity

dim_customer
├── customer_key (PK)
├── customer_name
├── city_id (FK → dim_city)

dim_city
├── city_id (PK)
├── city_name
├── region_id (FK → dim_region)

dim_region
├── region_id (PK)
├── region_name
├── country_id (FK → dim_country)

dim_country
├── country_id (PK)
└── country_name
```

### **Преимущества**

```sql
-- Нормализованная структура уменьшает избыточность
-- Легче управлять изменениями иерархий
UPDATE dim_city SET city_name = 'New Name'
WHERE city_id = 1;
-- Изменение применяется ко всем связанным клиентам
```

### **Недостатки**

```sql
-- Сложные запросы с множеством JOIN
SELECT
    c.country_name,
    SUM(f.sales_amount) as total_sales
FROM fact_sales f
JOIN dim_customer cust ON f.customer_key = cust.customer_key
JOIN dim_city city ON cust.city_id = city.city_id
JOIN dim_region r ON city.region_id = r.region_id
JOIN dim_country c ON r.country_id = c.country_id
GROUP BY c.country_name;
```

### **Best Use Cases**

- Сложные иерархические данные
- Требования к консистентности данных
- Системы с ограниченным хранилищем

---

## 🌌 **Схема "Галактика" (Galaxy Schema / Fact Constellation)**

### **Архитектура**

```sql
fact_sales
├── date_key (FK → dim_date)
├── customer_key (FK → dim_customer)
├── product_key (FK → dim_product)
├── store_key (FK → dim_store)
├── sales_amount
└── quantity

fact_inventory
├── date_key (FK → dim_date)
├── product_key (FK → dim_product)
├── store_key (FK → dim_store)
├── stock_quantity
└── reorder_level

fact_returns
├── date_key (FK → dim_date)
├── customer_key (FK → dim_customer)
├── product_key (FK → dim_product)
├── return_amount
└── return_reason

-- Общие измерения используются разными фактами
dim_date
├── date_key (PK)
├── date
├── month
└── year

dim_product
├── product_key (PK)
├── product_name
└── category
```

### **Преимущества**

```sql
-- Анализ across multiple business processes
SELECT
    d.month,
    p.category,
    SUM(s.sales_amount) as sales,
    AVG(i.stock_quantity) as avg_stock
FROM fact_sales s
JOIN fact_inventory i ON s.product_key = i.product_key
                      AND s.date_key = i.date_key
JOIN dim_date d ON s.date_key = d.date_key
JOIN dim_product p ON s.product_key = p.product_key
GROUP BY d.month, p.category;
```

### **Недостатки**

- Высокая сложность проектирования
- Риск создания "спагетти"-архитектуры
- Сложность управления консистентностью

### **Best Use Cases**

- Enterprise Data Warehouses
- Сложная бизнес-аналитика across процессов
- Системы с множеством взаимосвязанных фактов

---

## 🎯 **Критерии выбора схемы**

### **Выбирайте STAR SCHEMA если:**

- ✅ Приоритет - производительность запросов
- ✅ Простые и стабильные бизнес-требования
- ✅ Ограниченные ресурсы на разработку ETL
- ✅ Основное использование - стандартная отчетность

### **Выбирайте SNOWFLAKE SCHEMA если:**

- ✅ Требования к консистентности данных
- ✅ Сложные иерархии с частыми изменениями
- ✅ Ограничения по объему хранилища
- ✅ Необходимость ad-hoc анализа

### **Выбирайте GALAXY SCHEMA если:**

- ✅ Интеграция множества бизнес-процессов
- ✅ Требуется кросс-функциональная аналитика
- ✅ Enterprise-масштаб с общими измерениями
- ✅ Опытная команда data modelers

---

## 🔧 **Практические примеры**

### **Retail Company - Star Schema**

```sql
-- Фокус на производительность отчетов по продажам
fact_sales (10M+ строк)
├── date_key, customer_key, product_key, store_key
└── sales_amount, quantity, discount

dim_customer (50K строк) - денормализованный
├── customer_key, name, city, region, country, segment
```

### **Manufacturing Company - Snowflake Schema**

```sql
-- Сложные иерархии продуктов и локаций
fact_production (5M+ строк)
├── date_key, product_key, factory_key, line_key

dim_product (10K строк) - нормализованный
├── product_key, product_name, subcategory_id

dim_product_subcategory (100 строк)
├── subcategory_id, subcategory_name, category_id

dim_product_category (20 строк)
└── category_id, category_name
```

### **Financial Institution - Galaxy Schema**

```sql
-- Множество взаимосвязанных фактов
fact_transactions + fact_accounts + fact_loans
                  ↓
    Shared: dim_date, dim_customer, dim_branch
```

---

## 🚀 **Паттерны проектирования**

### **Conformed Dimensions**

```sql
-- Единые измерения across фактов
dim_date, dim_customer, dim_product
-- Используются в fact_sales, fact_returns, fact_inventory
```

### **Degenerate Dimensions**

```sql
-- Атрибуты, которые остаются в фактовой таблице
fact_sales
├── order_number (degenerate dimension)
├── invoice_number (degenerate dimension)
└── transaction_id (degenerate dimension)
```

### **Junk Dimensions**

```sql
-- Группировка мелких флагов и атрибутов
dim_sales_attributes
├── attribute_key
├── payment_method
├── delivery_type
├── promotion_flag
└── seasonality
```

---

## 📊 **Производительность: Benchmark**

| Операция                    | Star Schema | Snowflake Schema | Galaxy Schema |
| --------------------------- | ----------- | ---------------- | ------------- |
| **Simple Aggregation**      | 0.5s        | 1.2s             | 0.8s          |
| **Hierarchical Drill-Down** | 2.1s        | 0.8s             | 1.5s          |
| **Cross-Fact Analysis**     | N/A         | N/A              | 3.2s          |
| **Storage Size**            | 100%        | 65%              | 120%          |

---

## 🚨 **Anti-patterns**

1. **Over-Normalization** в Star Schema
2. **Under-Normalization** в Snowflake Schema
3. **Spaghetti Galaxy** без четкой архитектуры
4. **Ignoring Query Patterns** при проектировании

---

## ✅ **Best Practices**

1. **Start with Star** - упрощает начальную разработку
2. **Snowflake only when necessary** - для сложных иерархий
3. **Use Galaxy for enterprise** - интеграция процессов
4. **Profile query patterns** перед финальным дизайном
5. **Implement conformed dimensions** для консистентности

---

**Следующий раздел:** [1.4 - Data Lakehouse vs Traditional DWH](./04-data-lakehouse-vs-traditional.md)

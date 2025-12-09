# 🏗️ Data Architecture: Layered Storage для Product Metrics

> Как хранить и агрегировать данные для эффективного расчёта метрик

---

## 📑 Содержание

1. [Overview: 4-Layer Architecture](#overview-4-layer-architecture)
2. [Layer 1: Raw Events](#layer-1-raw-events-90-дней)
3. [Layer 2: Daily Aggregates](#layer-2-daily-aggregates-2-года)
4. [Layer 2.5: Weekly Aggregates](#layer-25-weekly-aggregates-1-год)
5. [Layer 3: Monthly Aggregates](#layer-3-monthly-aggregates-∞)
6. [Special Tables: Cohorts](#special-tables-cohort-snapshots)
7. [ETL Pipeline](#etl-pipeline)
8. [Retention Policies](#retention-policies)
9. [Metrics: Source Mapping](#metrics-source-mapping)
10. [Query Examples](#query-examples)

---

## OVERVIEW: 4-Layer Architecture

### Концепция

```mermaid
graph LR
    A[User Events] --> R1[Raw: activity]
    A --> R2[Raw: sessions]
    A --> R3[Raw: purchases]
    A --> R4[Raw: ad_events]
    
    R1 --> D1[Daily: user_activity]
    R2 --> D2[Daily: dau_summary]
    R3 --> D3[Daily: revenue]
    R4 --> D4[Daily: sessions_agg]
    
    D1 --> W1[Weekly: WAU]
    D3 --> W2[Weekly: Revenue]
    
    W1 --> M1[Monthly: MAU]
    W2 --> M2[Monthly: Revenue]
    
    R1 --> C1[Cohort: Retention]
    R3 --> C2[Cohort: LTV]
    D1 --> C1
    D3 --> C2
    
    style A fill:#34495e,color:#fff,stroke:#2c3e50
    style R1 fill:#2c3e50,color:#fff,stroke:#34495e
    style R2 fill:#2c3e50,color:#fff,stroke:#34495e
    style R3 fill:#2c3e50,color:#fff,stroke:#34495e
    style R4 fill:#2c3e50,color:#fff,stroke:#34495e
    style D1 fill:#16a085,color:#fff,stroke:#138d75
    style D2 fill:#16a085,color:#fff,stroke:#138d75
    style D3 fill:#16a085,color:#fff,stroke:#138d75
    style D4 fill:#16a085,color:#fff,stroke:#138d75
    style W1 fill:#e67e22,color:#fff,stroke:#d35400
    style W2 fill:#e67e22,color:#fff,stroke:#d35400
    style M1 fill:#2980b9,color:#fff,stroke:#21618c
    style M2 fill:#2980b9,color:#fff,stroke:#21618c
    style C1 fill:#8e44ad,color:#fff,stroke:#6c3483
    style C2 fill:#8e44ad,color:#fff,stroke:#6c3483
```

---

## LAYER 1: RAW EVENTS (90 дней)

### Характеристики

| Параметр | Значение |
|----------|----------|
| **Retention** | 90 дней |
| **Size** | ~10 TB (зависит от MAU) |
| **Cost** | ~$2,000/месяц |
| **Update** | Real-time (streaming) |
| **Используется для** | Детальный анализ, A/B тесты, Debug, Ad-hoc запросы |
| **Queries** | Медленные (большие объёмы) |

### Таблицы

```mermaid
erDiagram
    RAW_ACTIVITY_EVENTS {
        string event_id PK
        string user_id FK
        timestamp event_timestamp
        date event_date
        string event_type
        string session_id
        json event_properties
    }
    
    RAW_SESSION_EVENTS {
        string session_id PK
        string user_id FK
        timestamp session_start
        timestamp session_end
        int duration_sec
        string platform
        string country
    }
    
    RAW_PURCHASE_EVENTS {
        string purchase_id PK
        string user_id FK
        timestamp purchase_timestamp
        date purchase_date
        decimal amount_usd
        string product_id
        string product_type
        json purchase_metadata
    }
    
    RAW_AD_EVENTS {
        string ad_event_id PK
        string user_id FK
        timestamp event_timestamp
        string event_type
        string campaign_id
        string creative_id
        string channel
        decimal cost_usd
    }
```

---

## LAYER 2: DAILY AGGREGATES (2 года)

### Характеристики

| Параметр | Значение |
|----------|----------|
| **Retention** | 730 дней (2 года) |
| **Size** | ~500 GB |
| **Cost** | ~$100/месяц |
| **Update** | Daily ETL (каждую ночь) |
| **Используется для** | Дашборды, Retention D180/D365, LTV расчёты, Когортный анализ |
| **Queries** | Быстрые (pre-aggregated) |

### Data Flow

```mermaid
graph LR
    R1[Raw:<br/>activity] --> E[Daily ETL<br/>02:00] 
    R2[Raw:<br/>sessions] --> E
    R3[Raw:<br/>purchases] --> E
    
    E --> D1[user_daily_activity]
    E --> D2[dau_summary]
    E --> D3[revenue_daily]
    E --> D4[session_aggregates]
    
    style R1 fill:#2c3e50,color:#fff,stroke:#34495e
    style R2 fill:#2c3e50,color:#fff,stroke:#34495e
    style R3 fill:#2c3e50,color:#fff,stroke:#34495e
    style E fill:#34495e,color:#fff,stroke:#2c3e50
    style D1 fill:#16a085,color:#fff,stroke:#138d75
    style D2 fill:#16a085,color:#fff,stroke:#138d75
    style D3 fill:#16a085,color:#fff,stroke:#138d75
    style D4 fill:#16a085,color:#fff,stroke:#138d75
```

### Таблицы Layer 2

**Retention:** 730 дней | **Size:** ~500 GB | **Update:** Daily ETL

```mermaid
erDiagram
    USER_DAILY_ACTIVITY {
        string user_id PK
        date activity_date PK
        date install_date
        int day_since_install
        int sessions_count
        int session_duration_total_sec
        int events_count
        decimal revenue
        int purchases_count
        boolean is_paying
        boolean is_active
        string platform
        string country
        string channel
    }
    
    DAU_SUMMARY {
        date date PK
        int dau
        int wau
        int mau
        decimal stickiness
        int new_users
        int returning_users
        int churned_users
        string platform
        string country
    }
    
    REVENUE_DAILY {
        date date PK
        decimal revenue_total
        int paying_users
        int purchases_count
        decimal arpu
        decimal arppu
        decimal paying_rate
        string platform
        string country
        string product_type
    }
    
    SESSION_AGGREGATES {
        date date PK
        int sessions_total
        int unique_users
        decimal sessions_per_user
        int avg_session_duration_sec
        int median_session_duration_sec
        string platform
    }
```

### 📍 Важно: Оптимальная стратегия хранения

#### Подход: Только АКТИВНЫЕ пользователи + Churn Tracker

**user_daily_activity хранит ТОЛЬКО активных пользователей в каждый день:**

```sql
-- ETL: Добавляем только тех, кто был активен вчера
INSERT INTO user_daily_activity
SELECT user_id, yesterday, ...
FROM raw_activity_events
WHERE event_date = yesterday  -- Только активные!
GROUP BY user_id;
```

**Преимущества:**
- 💾 Экономия 95% места (300 GB vs 6 TB)
- ⚡ Запросы в 10+ раз быстрее
- 💰 Стоимость в 30+ раз ниже

**Для churn detection добавляем отдельную таблицу:**

```mermaid
erDiagram
    USER_CHURN_TRACKER {
        string user_id PK
        date install_date
        date last_activity_date
        int days_since_last_activity
        boolean is_churned
        date churned_at
        int lifetime_days
        decimal lifetime_revenue
    }
```

**Характеристики USER_CHURN_TRACKER:**
- 📊 **1 запись = 1 user** (всегда)
- 🔄 **UPDATE** каждый день (перезапись)
- 💾 **Константный размер** (~5 GB для 10M users)
- ⚡ **Быстрый доступ** к churn status

**ETL процесс:**

```python
def daily_etl():
    # 1. INSERT только активных → user_daily_activity
    for active_user in get_active_users(yesterday):
        insert_into_user_daily_activity(active_user)
    
    # 2. UPDATE всех users → user_churn_tracker
    # Активные: обнуляем days_inactive
    update_active_users_churn_tracker(yesterday)
    
    # Неактивные: увеличиваем days_inactive
    increment_inactive_users_counter()
```

**Размер данных (пример: 1M MAU):**

| Таблица | Записей | Размер | Retention |
|---------|---------|--------|-----------|
| user_daily_activity | 300K/day × 730 = 219M | 300 GB | 730 дней |
| user_churn_tracker | 10M (фиксированно) | 5 GB | Forever |
| **Total** | - | **305 GB** | - |

vs. альтернатива (все users каждый день): **6 TB** ❌

---

## LAYER 2.5: WEEKLY AGGREGATES (1 год)

### Характеристики

| Параметр | Значение |
|----------|----------|
| **Retention** | 365 дней (1 год) |
| **Size** | ~50 GB |
| **Cost** | ~$10/месяц |
| **Update** | Weekly ETL (каждый понедельник) |
| **Используется для** | Weekly reports, WAU trends, Week-over-Week analysis, Marketing reports |
| **Queries** | Быстрые (pre-aggregated) |

### Data Flow

```mermaid
graph LR
    D1[Daily:<br/>user_activity] --> E[Weekly ETL<br/>Monday]
    D2[Daily:<br/>revenue] --> E
    
    E --> W1[wau_weekly]
    E --> W2[revenue_weekly]
    E --> W3[cohort_weekly]
    
    style D1 fill:#16a085,color:#fff,stroke:#138d75
    style D2 fill:#16a085,color:#fff,stroke:#138d75
    style E fill:#34495e,color:#fff,stroke:#2c3e50
    style W1 fill:#e67e22,color:#fff,stroke:#d35400
    style W2 fill:#e67e22,color:#fff,stroke:#d35400
    style W3 fill:#e67e22,color:#fff,stroke:#d35400
```

### Таблицы Layer 2.5

**Retention:** 365 дней | **Size:** ~50 GB | **Update:** Weekly ETL

```mermaid
erDiagram
    WAU_WEEKLY {
        date week_start_date PK
        string year_week
        int wau
        int dau_avg
        decimal stickiness_avg
        int new_users
        int churned_users
        int returning_users
        decimal retention_d7_avg
        string platform
        string country
    }
    
    REVENUE_WEEKLY {
        date week_start_date PK
        string year_week
        decimal revenue_total
        int wau
        int paying_users
        decimal arpu
        decimal arppu
        decimal paying_rate
        int purchases_count
        decimal revenue_per_day_avg
        string platform
        string country
    }
    
    COHORT_WEEKLY {
        date cohort_week PK
        int week_number PK
        int users_installed
        int users_active_week
        decimal retention_rate
        decimal revenue_cumulative
        decimal ltv
        int paying_users
        decimal paying_rate
        string platform
        string channel
    }
    
    SESSION_WEEKLY {
        date week_start_date PK
        string year_week
        int sessions_total
        int unique_users
        decimal sessions_per_user
        int avg_session_duration_sec
        int total_events
        decimal events_per_session
        string platform
    }
```

---

## LAYER 3: MONTHLY AGGREGATES (∞)

### Характеристики

| Параметр | Значение |
|----------|----------|
| **Retention** | ∞ (forever) |
| **Size** | ~10 GB |
| **Cost** | ~$2/месяц |
| **Update** | Monthly ETL (1-го числа каждого месяца) |
| **Используется для** | Executive reports, Forecasting, YoY analysis, Board meetings |
| **Queries** | Очень быстрые (мало данных) |

### Data Flow

```mermaid
graph LR
    D1[Daily:<br/>user_activity] --> E[Monthly ETL<br/>1st day]
    D2[Daily:<br/>revenue] --> E
    
    E --> M1[mau_monthly]
    E --> M2[revenue_monthly]
    E --> M3[kpi_summary]
    
    style D1 fill:#16a085,color:#fff,stroke:#138d75
    style D2 fill:#16a085,color:#fff,stroke:#138d75
    style E fill:#34495e,color:#fff,stroke:#2c3e50
    style M1 fill:#2980b9,color:#fff,stroke:#21618c
    style M2 fill:#2980b9,color:#fff,stroke:#21618c
    style M3 fill:#2980b9,color:#fff,stroke:#21618c
```

### Таблицы Layer 3

**Retention:** Forever | **Size:** ~10 GB | **Update:** Monthly ETL

```mermaid
erDiagram
    MAU_MONTHLY {
        string year_month PK
        int mau
        int dau_avg
        decimal stickiness_avg
        int new_users
        int churned_users
        int net_growth
        decimal growth_rate
        decimal yoy_growth_rate
    }
    
    REVENUE_MONTHLY {
        string year_month PK
        decimal revenue_total
        int mau
        int paying_users
        decimal arpu
        decimal arppu
        decimal paying_rate
        decimal revenue_growth_mom
        decimal revenue_growth_yoy
    }
    
    KPI_SUMMARY_MONTHLY {
        string year_month PK
        int mau
        decimal revenue
        decimal arpu
        decimal d1_retention_avg
        decimal d7_retention_avg
        decimal d30_retention_avg
        decimal ltv_d30_avg
        decimal ltv_d90_avg
        decimal cpi_avg
        decimal roas_d30
        decimal ltv_cac_ratio
    }
```

---

## SPECIAL TABLES: COHORT SNAPSHOTS

### Характеристики

| Параметр | Значение |
|----------|----------|
| **Retention** | ∞ (forever) |
| **Size** | ~50 GB |
| **Update** | Daily (для активных когорт) |
| **Используется для** | Retention curves, LTV curves, Cohort analysis |

### Концепция

```mermaid
graph LR
    R[Raw Events<br/>≤90 days] --> D0[D0]
    R --> D1[D1]
    R --> D7[D7]
    R --> D30[D30]
    
    A[Daily Aggregates<br/>≤730 days] --> D90[D90]
    A --> D180[D180]
    A --> D365[D365]
    
    D0 --> Snap[Cohort Snapshots<br/>Forever]
    D1 --> Snap
    D7 --> Snap
    D30 --> Snap
    D90 --> Snap
    D180 --> Snap
    D365 --> Snap
    
    style R fill:#2c3e50,color:#fff,stroke:#34495e
    style A fill:#16a085,color:#fff,stroke:#138d75
    style D0 fill:#8e44ad,color:#fff,stroke:#6c3483
    style D1 fill:#8e44ad,color:#fff,stroke:#6c3483
    style D7 fill:#8e44ad,color:#fff,stroke:#6c3483
    style D30 fill:#8e44ad,color:#fff,stroke:#6c3483
    style D90 fill:#2980b9,color:#fff,stroke:#21618c
    style D180 fill:#2980b9,color:#fff,stroke:#21618c
    style D365 fill:#2980b9,color:#fff,stroke:#21618c
    style Snap fill:#8e44ad,color:#fff,stroke:#6c3483
```

### Таблицы Special

**Retention:** Forever | **Size:** ~50 GB | **Update:** Daily

```mermaid
erDiagram
    COHORT_RETENTION_SNAPSHOT {
        date cohort_date PK
        int day_since_install PK
        int users_installed
        int users_active
        decimal retention_rate
        int sessions_total
        decimal sessions_per_user
        int avg_session_duration_sec
        date calculated_at
        string data_source
        string platform
        string country
        string channel
    }
    
    COHORT_LTV_SNAPSHOT {
        date cohort_date PK
        int day_since_install PK
        int users_installed
        decimal revenue_cumulative
        decimal ltv
        decimal revenue_day
        decimal arpu_day
        int paying_users_cumulative
        decimal paying_rate_cumulative
        int paying_users_day
        date calculated_at
        string data_source
        string platform
        string channel
    }
```

---

## ETL PIPELINE

### Daily Job (02:00 каждую ночь)

```mermaid
graph TB
    Start[Daily ETL Start] --> S1[Process Raw Events]
    S1 --> S2[Aggregate to Daily]
    S2 --> S3[Update Summaries]
    S3 --> S4[Update Cohorts]
    S4 --> S5[Cleanup Old Data]
    S5 --> End[ETL Complete]
    
    style Start fill:#2c3e50,color:#fff,stroke:#34495e
    style S1 fill:#2c3e50,color:#fff,stroke:#34495e
    style S2 fill:#16a085,color:#fff,stroke:#138d75
    style S3 fill:#16a085,color:#fff,stroke:#138d75
    style S4 fill:#8e44ad,color:#fff,stroke:#6c3483
    style S5 fill:#e74c3c,color:#fff,stroke:#c0392b
    style End fill:#27ae60,color:#fff,stroke:#1e8449
```

### Weekly Job (каждый понедельник)

```mermaid
graph TB
    Start[Weekly ETL Start] --> W1[Read Daily Data]
    W1 --> W2[Aggregate Weekly]
    W2 --> W3[Calculate WoW]
    W3 --> W4[Save to Weekly]
    W4 --> End[ETL Complete]
    
    style Start fill:#2c3e50,color:#fff,stroke:#34495e
    style W1 fill:#16a085,color:#fff,stroke:#138d75
    style W2 fill:#e67e22,color:#fff,stroke:#d35400
    style W3 fill:#e67e22,color:#fff,stroke:#d35400
    style W4 fill:#e67e22,color:#fff,stroke:#d35400
    style End fill:#27ae60,color:#fff,stroke:#1e8449
```

### Monthly Job (1-го числа месяца)

```mermaid
graph TB
    Start[Monthly ETL Start] --> M1[Read Daily Data]
    M1 --> M2[Aggregate Monthly]
    M2 --> M3[Calculate Growth]
    M3 --> M4[Save to Monthly]
    M4 --> End[ETL Complete]
    
    style Start fill:#2c3e50,color:#fff,stroke:#34495e
    style M1 fill:#16a085,color:#fff,stroke:#138d75
    style M2 fill:#16a085,color:#fff,stroke:#138d75
    style M3 fill:#2980b9,color:#fff,stroke:#21618c
    style M4 fill:#8e44ad,color:#fff,stroke:#6c3483
    style End fill:#27ae60,color:#fff,stroke:#1e8449
```

---

## RETENTION POLICIES

### Таблица политик

| Layer | Таблица | Retention | Auto-Delete | Size | Cost/мес |
|-------|---------|-----------|-------------|------|----------|
| **Layer 1** | `raw_activity_events` | 90 дней | ✅ | 5 TB | $1,000 |
| **Layer 1** | `raw_session_events` | 90 дней | ✅ | 3 TB | $600 |
| **Layer 1** | `raw_purchase_events` | 90 дней | ✅ | 1 TB | $200 |
| **Layer 1** | `raw_ad_events` | 90 дней | ✅ | 1 TB | $200 |
| | | | | | |
| **Layer 2** | `user_daily_activity` | 730 дней | ✅ | 300 GB | $60 |
| **Layer 2** | `user_churn_tracker` | Forever | ❌ | 5 GB | $1 |
| **Layer 2** | `dau_summary` | 730 дней | ✅ | 100 GB | $20 |
| **Layer 2** | `revenue_daily` | 730 дней | ✅ | 50 GB | $10 |
| **Layer 2** | `session_aggregates` | 730 дней | ✅ | 50 GB | $10 |
| | | | | | |
| **Layer 2.5** | `wau_weekly` | 365 дней | ✅ | 20 GB | $4 |
| **Layer 2.5** | `revenue_weekly` | 365 дней | ✅ | 15 GB | $3 |
| **Layer 2.5** | `cohort_weekly` | 365 дней | ✅ | 10 GB | $2 |
| **Layer 2.5** | `session_weekly` | 365 дней | ✅ | 5 GB | $1 |
| | | | | | |
| **Layer 3** | `mau_monthly` | Forever | ❌ | 5 GB | $1 |
| **Layer 3** | `revenue_monthly` | Forever | ❌ | 3 GB | $0.5 |
| **Layer 3** | `kpi_summary_monthly` | Forever | ❌ | 2 GB | $0.5 |
| | | | | | |
| **Special** | `cohort_retention_snapshot` | Forever | ❌ | 30 GB | $6 |
| **Special** | `cohort_ltv_snapshot` | Forever | ❌ | 20 GB | $4 |

**Итого:**
- **Layer 1 (Raw):** $2,000/мес
- **Layer 2 (Daily):** $101/мес (включая churn_tracker)
- **Layer 2.5 (Weekly):** $10/мес
- **Layer 3 (Monthly):** $2/мес
- **Special (Cohorts):** $10/мес

**Total: $2,123/мес**

**vs. Альтернативы:**
- Хранить все raw forever: $20,000+/мес ❌
- Хранить все users в daily: $6,500/мес ❌
- Оптимальная архитектура: **$2,123/мес** ✅

---

## METRICS: SOURCE MAPPING

### Откуда берутся метрики?

```mermaid
graph LR
    subgraph Metrics["Метрики"]
        M1[DAU/WAU/MAU]
        M2[D1/D7/D30<br/>Retention]
        M3[D180/D365<br/>Retention]
        M4[Revenue Today]
        M5[LTV D30]
        M6[LTV D365]
        M7[Session Duration]
        M8[YoY Growth]
    end
    
    subgraph Sources["Источники"]
        S1[raw_events<br/>90 days]
        S2[user_daily_activity<br/>730 days]
        S3[cohort_retention<br/>forever]
        S4[cohort_ltv<br/>forever]
        S5[monthly_aggregates<br/>forever]
    end
    
    S2 --> M1
    S1 --> M2
    S2 --> M2
    S3 --> M3
    S1 --> M4
    S2 --> M4
    S2 --> M5
    S4 --> M5
    S4 --> M6
    S2 --> M7
    S5 --> M8
    
    style M1 fill:#16a085,color:#fff,stroke:#138d75
    style M2 fill:#16a085,color:#fff,stroke:#138d75
    style M3 fill:#2980b9,color:#fff,stroke:#21618c
    style M4 fill:#16a085,color:#fff,stroke:#138d75
    style M5 fill:#8e44ad,color:#fff,stroke:#6c3483
    style M6 fill:#8e44ad,color:#fff,stroke:#6c3483
    style M7 fill:#16a085,color:#fff,stroke:#138d75
    style M8 fill:#2980b9,color:#fff,stroke:#21618c
    
    style S1 fill:#2c3e50,color:#fff,stroke:#34495e
    style S2 fill:#16a085,color:#fff,stroke:#138d75
    style S3 fill:#8e44ad,color:#fff,stroke:#6c3483
    style S4 fill:#8e44ad,color:#fff,stroke:#6c3483
    style S5 fill:#2980b9,color:#fff,stroke:#21618c
```

### Детальная таблица

| Метрика | Timeframe | Source Layer | Source Table | Comment |
|---------|-----------|--------------|--------------|---------|
| **DAU** | Today | Layer 1 | `raw_activity_events` | Real-time |
| **DAU** | Last 90 days | Layer 1 | `raw_activity_events` | Fast |
| **DAU** | 90+ days ago | Layer 2 | `dau_summary` | Pre-calculated |
| | | | | |
| **D1 Retention** | Fresh cohorts (≤90d) | Layer 1 | `raw_activity_events` | Can calculate from raw |
| **D1 Retention** | Old cohorts (>90d) | Special | `cohort_retention_snapshot` | Raw deleted, use snapshot |
| | | | | |
| **D7 Retention** | Fresh cohorts (≤90d) | Layer 1 | `raw_activity_events` | Can calculate from raw |
| **D7 Retention** | Old cohorts (>90d) | Special | `cohort_retention_snapshot` | Use snapshot |
| | | | | |
| **D30 Retention** | Fresh cohorts (≤90d) | Layer 1 + Layer 2 | `raw_activity_events` + `user_daily_activity` | Mix sources |
| **D30 Retention** | Old cohorts (>90d) | Special | `cohort_retention_snapshot` | Use snapshot |
| | | | | |
| **D90 Retention** | Fresh cohorts (≤90d) | Layer 2 | `user_daily_activity` | Raw already deleted for D90 |
| **D90 Retention** | Old cohorts (>90d) | Special | `cohort_retention_snapshot` | Use snapshot |
| | | | | |
| **D180 Retention** | Any cohort | Special | `cohort_retention_snapshot` | Always from snapshot |
| **D365 Retention** | Any cohort | Special | `cohort_retention_snapshot` | Always from snapshot |
| | | | | |
| **Revenue Today** | Today | Layer 1 | `raw_purchase_events` | Real-time |
| **Revenue Last 90d** | Last 90 days | Layer 1 | `raw_purchase_events` | From raw |
| **Revenue 90+ days** | 90+ days ago | Layer 2 | `revenue_daily` | Pre-calculated |
| | | | | |
| **LTV D30** | Fresh cohorts | Layer 1 + Layer 2 | Mix | Can calculate |
| **LTV D30** | Old cohorts | Special | `cohort_ltv_snapshot` | Use snapshot |
| | | | | |
| **LTV D90** | Any cohort | Special | `cohort_ltv_snapshot` | Always from snapshot |
| **LTV D180** | Any cohort | Special | `cohort_ltv_snapshot` | Always from snapshot |
| **LTV D365** | Any cohort | Special | `cohort_ltv_snapshot` | Always from snapshot |
| | | | | |
| **ARPU** | Last 90 days | Layer 1 | `raw_purchase_events` + `raw_activity_events` | From raw |
| **ARPU** | 90+ days | Layer 2 | `revenue_daily` + `dau_summary` | Pre-calculated |
| | | | | |
| **MAU** | Current month | Layer 2 | `user_daily_activity` | Rolling 30 days |
| **MAU** | Previous months | Layer 3 | `mau_monthly` | Pre-calculated |
| | | | | |
| **YoY Growth** | Any period | Layer 3 | `monthly_aggregates` | Pre-calculated |

---

## QUERY EXAMPLES

### 1. DAU за последние 30 дней

```sql
-- ✅ БЫСТРО: используем raw events (последние 90 дней)
SELECT 
    event_date,
    COUNT(DISTINCT user_id) as dau
FROM raw_activity_events
WHERE event_date >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY event_date
ORDER BY event_date;
```

---

### 2. DAU за прошлый год

```sql
-- ✅ БЫСТРО: используем pre-calculated summary
SELECT 
    date,
    dau
FROM dau_summary
WHERE date >= CURRENT_DATE - INTERVAL '365 days'
  AND platform = 'ALL'
ORDER BY date;
```

---

### 3. D1 Retention для когорты 30 дней назад

```sql
-- ✅ Можем использовать raw events (≤90 дней)
WITH cohort AS (
    SELECT user_id, install_date
    FROM users
    WHERE install_date = CURRENT_DATE - INTERVAL '30 days'
),
returned_d1 AS (
    SELECT DISTINCT c.user_id
    FROM cohort c
    JOIN raw_activity_events e 
        ON c.user_id = e.user_id
        AND e.event_date = c.install_date + INTERVAL '1 day'
)
SELECT 
    COUNT(DISTINCT c.user_id) as cohort_size,
    COUNT(DISTINCT r.user_id) as returned_d1,
    COUNT(DISTINCT r.user_id) * 100.0 / COUNT(DISTINCT c.user_id) as d1_retention
FROM cohort c
LEFT JOIN returned_d1 r ON c.user_id = r.user_id;
```

---

### 4. D1 Retention для когорты 200 дней назад

```sql
-- ❌ НЕ РАБОТАЕТ: raw events удалены (>90 дней)
-- ✅ РАБОТАЕТ: используем snapshot
SELECT 
    cohort_date,
    users_installed,
    users_active,
    retention_rate as d1_retention
FROM cohort_retention_snapshot
WHERE cohort_date = CURRENT_DATE - INTERVAL '200 days'
  AND day_since_install = 1
  AND platform = 'ALL';
```

---

### 5. D365 Retention для любой когорты

```sql
-- ✅ Всегда используем snapshot (raw не подходит)
SELECT 
    cohort_date,
    users_installed,
    users_active,
    retention_rate as d365_retention
FROM cohort_retention_snapshot
WHERE cohort_date = '2024-01-01'
  AND day_since_install = 365;
```

---

### 6. LTV D90 для когорты

```sql
-- ✅ Используем LTV snapshot
SELECT 
    cohort_date,
    users_installed,
    revenue_cumulative,
    ltv,
    paying_users_cumulative,
    paying_rate_cumulative
FROM cohort_ltv_snapshot
WHERE cohort_date = '2024-01-01'
  AND day_since_install = 90;
```

---

### 7. Retention Curve (D0-D365)

```sql
-- ✅ Быстрый запрос из snapshot
SELECT 
    day_since_install,
    retention_rate
FROM cohort_retention_snapshot
WHERE cohort_date = '2024-01-01'
  AND platform = 'ALL'
  AND day_since_install IN (0, 1, 7, 14, 30, 60, 90, 180, 365)
ORDER BY day_since_install;
```

**Результат:**

| day | retention_rate |
|-----|----------------|
| 0 | 100.0% |
| 1 | 40.0% |
| 7 | 20.0% |
| 14 | 15.0% |
| 30 | 10.0% |
| 60 | 7.0% |
| 90 | 5.0% |
| 180 | 3.0% |
| 365 | 2.0% |

---

### 8. LTV Curve (D0-D365)

```sql
-- ✅ Быстрый запрос из snapshot
SELECT 
    day_since_install,
    ltv,
    paying_rate_cumulative
FROM cohort_ltv_snapshot
WHERE cohort_date = '2024-01-01'
  AND platform = 'ALL'
ORDER BY day_since_install;
```

**Результат:**

| day | ltv | paying_rate |
|-----|-----|-------------|
| 0 | $0.00 | 0.0% |
| 1 | $0.50 | 3.0% |
| 7 | $1.50 | 4.5% |
| 30 | $5.00 | 6.0% |
| 90 | $10.00 | 7.0% |
| 180 | $13.00 | 7.5% |
| 365 | $15.00 | 8.0% |

---

### 9. MAU по месяцам (за 2 года)

```sql
-- ✅ Используем monthly aggregates
SELECT 
    year_month,
    mau,
    growth_rate as mom_growth,
    yoy_growth_rate
FROM mau_monthly
WHERE year_month >= '2023-01'
ORDER BY year_month;
```

---

### 10. YoY Revenue Growth

```sql
-- ✅ Используем monthly aggregates
SELECT 
    year_month,
    revenue_total,
    revenue_growth_yoy
FROM revenue_monthly
WHERE year_month >= '2023-01'
ORDER BY year_month;
```

---

## DECISION TREE: Какой слой использовать?

```mermaid
graph TB
    Start[Метрика] --> Time{Период?}
    
    Time -->|Today| Raw[Layer 1: Raw Events]
    Time -->|≤90 days| Raw
    Time -->|90-730 days| Daily[Layer 2: Daily Aggregates]
    Time -->|730+ days| Monthly[Layer 3: Monthly Aggregates]
    Time -->|Cohorts| Cohorts[Special: Cohort Snapshots]
    
    style Start fill:#34495e,color:#fff,stroke:#2c3e50
    style Time fill:#34495e,color:#fff,stroke:#2c3e50
    style Raw fill:#2c3e50,color:#fff,stroke:#34495e
    style Daily fill:#16a085,color:#fff,stroke:#138d75
    style Monthly fill:#2980b9,color:#fff,stroke:#21618c
    style Cohorts fill:#8e44ad,color:#fff,stroke:#6c3483
```

---

## KEY TAKEAWAYS

### ✅ Что нужно делать:

| Действие | Зачем |
|----------|-------|
| **Создать 4-layer архитектуру** | Разделение по retention и стоимости |
| **ETL каждый день в 02:00** | Обновление daily aggregates и cohorts |
| **Хранить только АКТИВНЫХ users** | user_daily_activity: экономия 95% места |
| **Использовать churn_tracker** | 1 запись на user, UPDATE вместо INSERT |
| **Snapshot когорт на милестоунах** | D1, D7, D30, D90, D180, D365 |
| **Partitioning по датам** | Быстрое удаление старых данных |
| **Хранить raw ≤90 дней** | Баланс между детализацией и стоимостью |
| **Хранить daily aggregates 2 года** | Для LTV D365 расчётов |
| **Хранить weekly aggregates 1 год** | Для WoW анализа |
| **Хранить monthly aggregates forever** | Для YoY анализа |
| **Cohort snapshots forever** | Для retention/LTV curves |

### ❌ Чего НЕ делать:

| Действие | Почему плохо |
|----------|--------------|
| **Хранить raw events forever** | $$$$ слишком дорого |
| **Хранить ВСЕ users в daily** | 6 TB вместо 300 GB - дорого и медленно |
| **Пересчитывать cohorts каждый раз** | Медленно, можно использовать snapshot |
| **Не делать pre-aggregation** | Запросы будут слишком долгими |
| **Удалять aggregates раньше времени** | Потеряете возможность считать D365 метрики |
| **Хранить только aggregates** | Потеряете гибкость для ad-hoc анализа |
| **Множественные записи для churn** | Используйте 1 запись + UPDATE |

---

## COST OPTIMIZATION

### До оптимизации (всё raw forever):

```
Raw events (10 TB × 12 months) = 120 TB
Cost: 120 TB × $23/TB/month = $2,760/month
Yearly: $33,120
```

### После оптимизации (3-layer):

```
Layer 1 (Raw, 90 days):      10 TB × $23/TB = $230/month
Layer 2 (Daily, 730 days):   0.5 TB × $20/TB = $10/month
Layer 3 (Monthly, forever):  0.01 TB × $10/TB = $0.10/month
Special (Cohorts, forever):  0.05 TB × $10/TB = $0.50/month

Total: $240.60/month
Yearly: $2,887
```

**Экономия: $30,233/год (91% reduction!)** 💰

---

## 📖 Связанные документы

| Документ | Описание |
|----------|----------|
| [README.md](./README.md) | Основной справочник по метрикам |
| [01_retention_metrics.ipynb](./01_retention_metrics.ipynb) | Retention analysis |
| [02_engagement_metrics.ipynb](./02_engagement_metrics.ipynb) | Engagement analysis |
| [03_monetization_metrics.ipynb](./03_monetization_metrics.ipynb) | Monetization analysis |
| [04_user_acquisition_metrics.ipynb](./04_user_acquisition_metrics.ipynb) | UA analysis |

---

**🎮 Good luck with your data architecture! 🏗️**


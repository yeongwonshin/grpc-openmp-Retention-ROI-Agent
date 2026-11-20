
# Current Analysis Pipeline

## 1. System Architecture

### Overall Architecture

```text
CSV Upload → Pipeline Training → Save Results → Dashboard
                                   │
                                   ▼ (Seed)
                              PostgreSQL → Real-Time Operations Monitor
                                   ▲
                                   │ (Events)
                              FastAPI Server ← External Systems
```

### System Components

| Component | Role |
| --- | --- |
| **Streamlit** | Provides the user-facing analytical and operational dashboard |
| **FastAPI** | Handles API communication and data exchange with external systems |
| **PostgreSQL** | Provides persistent storage for customer data, prediction scores, and action queues |
| **Redis** | Caches query results with a time-to-live (TTL) of 20 seconds |
| **ML Models** | Perform churn prediction (LightGBM), survival analysis (Cox), Customer Lifetime Value (CLV) estimation, and uplift modeling |

### PostgreSQL Integration

- PostgreSQL containers are automatically provisioned through Docker Compose for local development and demonstration environments.
- Files matching `src/api/services/user_live_*.py` contain the logic for reading from and writing to the PostgreSQL database named `retention_db`.
- Database tables are automatically created through `CREATE TABLE IF NOT EXISTS` statements in the application code.
- Database migration tools such as Alembic are not currently used.
- In a production deployment, external business systems can send customer events to FastAPI endpoints, triggering the following workflow:

```text
External Customer Event
    ↓
FastAPI Endpoint
    ↓
PostgreSQL Persistence
    ↓
Customer Feature Update
    ↓
ML Model Rescoring
    ↓
Action Queue Update
```

This architecture enables the platform to connect offline-trained models with event-driven retention operations.

---

## 2. Churn Probability Prediction

### 2.1. Churn Label Definition

A customer is classified as churned (`churn = 1`) when they remain inactive for at least 30 days following their most recent activity.

- The inactivity threshold can be adjusted by users through the dashboard.
- The system can automatically infer churn labels even when the uploaded dataset does not contain an explicit `churn_flag` column.
- The resulting labels are used as targets for supervised churn prediction.

### 2.2. Model Training

The platform trains a **LightGBM classification model** using 88 engineered features.

These features capture different aspects of customer behavior and engagement.

**Behavioral Activity Counts**

Examples include:

- `visit_7d`
- `purchase_30d`
- `sessions_30d`

These features represent recent customer activity and transaction frequency.

**Recency and Time-Based Features**

Examples include:

- `recency_days`
- `days_since_last_event`
- `inactivity_days`
- `current_non_purchase_days`

These features measure the time elapsed since specific customer activities and help identify behavioral inactivity.

**Purchase Behavior Features**

Examples include:

- `avg_purchase_gap_days`
- `cart_to_purchase_rate_30d`

These features describe purchase frequency, conversion patterns, and deviations from typical purchasing behavior.

**Session and Temporal Features**

Examples include:

- `evening_activity_ratio`
- `weekend_purchase_ratio`

These features capture customer activity patterns across different times of day and days of the week.

### 2.3. Real-Time Rescoring

When new customer events are received, the system invokes the `score_changed_customers()` function to update predictions for affected customers.

The rescoring process uses the latest customer feature state to generate updated churn probability estimates.

These updated scores can subsequently influence retention priorities and action queue decisions.

---

## 3. Churn Timing Prediction (Survival Analysis)

### 3.1. Data Preparation

Survival analysis uses a landmark-based approach to separate historical customer behavior from future churn outcomes.

The landmark date is defined as:

```text
landmark_date = last_activity_date_in_dataset - horizon_days
```

With the default observation horizon:

```text
horizon_days = 90
```

The dataset is divided into two periods:

- **Before the landmark date:** Historical customer behavior used for feature engineering.
- **After the landmark date:** Future observation period used to determine when churn occurs.

This allows the model to learn the relationship between historical customer characteristics and subsequent churn timing.

### 3.2. Cox Proportional Hazards Model

The platform uses the **Cox Proportional Hazards model**, implemented through the `lifelines` Python library.

**Model Inputs**

Customer-level features, such as:

- Visit frequency
- Purchase intervals
- Inactivity duration
- Recent behavioral activity
- Other engineered customer attributes

**Training Objective**

The model estimates how customer characteristics are associated with the relative hazard of churn over time.

Conceptually, it learns patterns such as:

> Customers exhibiting a particular combination of behavioral features tend to experience churn earlier or later than other customers.

**Model Outputs**

The model estimates regression coefficients that can be expressed as hazard ratios.

These values describe how individual features are associated with increases or decreases in the relative churn hazard, under the proportional hazards assumption.

### 3.3. Individual Survival Curves and Median Time to Churn

After training, the model generates an individual survival probability curve for each customer over the prediction horizon.

With the default configuration, the curve covers days 1 through 90.

The predicted median time to churn is defined as the first time at which the estimated survival probability reaches or falls below 50%.

```text
predicted_median_time_to_churn_days
    = First day when Survival Probability <= 0.50
```

**Example**

Suppose customer `1894` has the following predicted survival probabilities:

```text
Day 1:  0.92
Day 3:  0.81
Day 5:  0.67
Day 7:  0.54
Day 8:  0.48
```

Because the survival probability falls below 50% on day 8:

```text
predicted_median_time_to_churn_days = 8
```

The dashboard can interpret this result as:

```text
Estimated median time to churn: 8 days
```

This is a probabilistic estimate rather than a guarantee that the customer will churn within eight days.

If the survival probability does not fall to 50% within the prediction horizon, the median time to churn cannot be determined within that horizon.

### 3.4. Dashboard Output

The predicted churn date is calculated using:

```text
Estimated Churn Date
    = Landmark Date + Predicted Days to Churn
```

The dashboard presents survival analysis results to help operators determine the appropriate timing of retention interventions.

---

## 4. Configuring the Survival Analysis Horizon

### 4.1. Observation Horizon

The default survival analysis horizon is 90 days.

However, it can be modified through the pipeline parameters.

For example:

```python
run_survival_pipeline(..., horizon_days=60)
```

The observation horizon is distinct from the churn inactivity threshold.

- **Churn threshold (default: 30 days):** Defines when a customer is considered churned.
- **Observation horizon (default: 90 days):** Defines the future period within which churn outcomes are observed during survival model training.

The horizon determines how much future information is available for learning churn timing patterns.

### 4.2. Historical Data Requirements

For a 90-day observation horizon, the dataset should contain enough historical information to calculate behavioral features before the landmark date, followed by a sufficient future observation period.

For example:

```text
Historical Feature Window (90 Days)
                +
Future Observation Window (90 Days)
                =
Minimum Data Coverage of Approximately 180 Days
```

Therefore, approximately six months of data is a practical minimum for the default configuration.

The recommended data duration may vary depending on the industry and typical customer activity frequency.

### Recommended Data Duration by Industry

| Industry | Typical Activity Frequency | Recommended Historical Data Duration |
| --- | --- | --- |
| E-commerce (Fashion/Food) | 1–2 visits per week | 6 months to 1 year |
| Subscription Services (SaaS) | Daily activity | 3–6 months |
| Financial Services / Insurance | 1–2 interactions per month | 1–2 years |
| Automotive / Home Appliances | Approximately once per quarter | 2–3 years |

These recommendations are practical guidelines rather than strict statistical requirements.

Industries with longer purchasing or interaction cycles generally require longer observation periods to capture meaningful customer behavior.

### 4.3. Automatic Horizon Adjustment

The platform adjusts the survival analysis horizon according to the available dataset duration.

| Uploaded Data Duration | Horizon Adjustment | Prediction Coverage |
| --- | --- | --- |
| 1 year or longer | 90 days (default) | Predictions up to 90 days ahead |
| 6 months to 1 year | 90 days | Suitable coverage for the default horizon |
| 90 days to 6 months | Reduced to half of the available data duration | Short- to medium-term predictions |
| 60 to 90 days | Reduced to 30 days | Short-term predictions only |
| Less than 60 days | **Survival analysis unavailable** | Only churn probability prediction is provided; churn timing prediction is disabled |

This adaptive strategy helps prevent the survival analysis pipeline from operating with insufficient historical or future observation data.

It also enables the platform to provide partial analytical capabilities when users upload relatively short datasets.

---

## 5. Data Flow for Production Deployment

### 5.1. Current Architecture (Development / Demo)

The current implementation primarily follows a batch-oriented workflow.

```text
CSV Upload → ML Pipeline → Result Files → Seed → PostgreSQL → Dashboard
(Manual)     (One-Time)                                   (File-Based + Partial DB)
```

**Workflow Description**

1. Users manually upload business datasets.
2. The ML pipeline processes the data and trains the required models.
3. Generated results and model artifacts are saved to files.
4. Selected artifacts are seeded into PostgreSQL.
5. The dashboard loads analytical results from files and relevant operational data from the database.

This architecture is suitable for:

- Proof-of-concept development
- Model validation
- Product demonstrations
- Competition submissions

### 5.2. Target Architecture for Production

For production environments, the platform is designed to receive events directly from external business systems.

```text
External Business System
          │
          ▼
POST /api/v1/user-live/events
          │
          ▼
      FastAPI
          │
          ├── Store Event in PostgreSQL
          │
          ├── Update Customer Features
          │
          ├── Recalculate ML Scores
          │
          ├── Update Retention Action Queue
          │
          └── Reflect Changes in Dashboard
```

### Example Event Payload

```json
{
  "customer_id": 123,
  "event_type": "purchase",
  "amount": 89000
}
```

In this configuration, external applications can continuously send customer interaction events to the platform.

The system processes incoming events, updates the affected customer records, recalculates relevant prediction scores, and refreshes intervention recommendations.

This enables retention decisions to adapt dynamically as customer behavior changes.

### 5.3. Data Integration Methods

| Integration Method | Update Frequency | Recommended Use Case |
| --- | --- | --- |
| CSV Upload | Manual | Proof of Concept (PoC) and demonstrations |
| Event API Integration | Immediate event processing | Production services (recommended) |
| Direct Database Integration (ETL) | Periodic updates | Legacy systems and scheduled data synchronization |

### Integration Considerations

**CSV Upload**

Suitable for organizations that want to evaluate the platform using existing datasets without implementing direct system integration.

**Event API Integration**

Recommended for operational environments where customer activities should trigger timely feature updates, prediction recalculations, and action queue changes.

**Direct Database Integration (ETL)**

Suitable for organizations with legacy systems or scheduled data pipelines where event-driven API integration is not immediately feasible.

---

## 6. Summary

Retention ROI Agent combines batch machine learning pipelines with a database-backed, event-driven operational architecture.

Its analytical workflow consists of two major prediction components:

1. **Churn Probability Prediction:** Estimates the likelihood that a customer will churn based on historical behavioral features.
2. **Churn Timing Prediction:** Uses survival analysis to estimate when churn may occur.

These predictions are subsequently integrated into retention decision-making and operational monitoring.

The current development architecture supports CSV-based training, artifact generation, PostgreSQL seeding, and live event processing.

For production use, the platform is designed to integrate with external business systems through FastAPI endpoints, enabling customer event ingestion, feature updates, model rescoring, and action queue management.

The long-term architectural objective is to establish a continuous workflow:

```text
Customer Events
    ↓
Feature Engineering
    ↓
Churn Probability & Timing Prediction
    ↓
Retention Decision Optimization
    ↓
Action Queue Update
    ↓
Operational Dashboard
```

This architecture provides the foundation for transforming static churn analysis into an adaptive, data-driven customer retention decision system.

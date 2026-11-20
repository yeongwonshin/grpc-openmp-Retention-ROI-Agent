
# Counterfactual Retention Lab

## 1. Overview

The existing retention dashboard provides information on customer churn risk, recommended retention actions, and budget-based customer targeting.

The **Counterfactual Retention Lab** extends these capabilities by comparing the expected net profit of multiple intervention scenarios for the same customer.

### Intervention Scenarios

- No intervention
- Offer a KRW 5,000 coupon
- Customer support call
- Push notification or email
- Wait for 7 days

The objective of this view is not only to determine **which action should be recommended**, but also to explain **why that action is expected to be more economically beneficial than taking no action or selecting an alternative intervention**.

---

## 2. Calculation Logic

The core counterfactual estimation engine is implemented in:

`src/optimization/counterfactual.py`

### Core Formulas

The expected financial outcomes are calculated using the following simplified formulas:

```text
Expected Net Profit (No Intervention)
    = CLV × (1 - Current Churn Probability)

Expected Net Profit (With Intervention)
    = CLV × (1 - Post-Intervention Churn Probability)
      - Intervention Cost

Incremental Net Profit vs. No Intervention
    = Expected Net Profit (With Intervention)
      - Expected Net Profit (No Intervention)
```

### Post-Intervention Churn Probability Estimation

The post-intervention churn probability is estimated using existing customer features and model outputs, including:

- `churn_probability`
- `uplift_score`
- `CLV`
- `survival/timing`
- `coupon_affinity`
- `price_sensitivity`
- `discount_pressure_score`

When survival analysis artifacts are available, the estimation incorporates:

- Predicted time to churn
- Timing urgency
- Urgency-based weighting

If survival analysis artifacts are unavailable, the system uses conservative default values to maintain the calculation workflow.

This allows the platform to compare intervention scenarios even when complete survival analysis outputs are not available.

---

## 3. Important Considerations

The calculated values are **prediction-based counterfactual estimates**, not actual outcomes observed after executing retention interventions.

Therefore, these estimates should be treated as decision-support indicators rather than experimentally verified causal effects.

### Operational Validation Process

1. **Conduct A/B Tests or Holdout Experiments**
   - Include customers with low or medium prediction confidence in controlled experiments.

2. **Record Actual Intervention Logs**
   - Track message delivery, customer support calls, coupon redemptions, and other intervention activities.

3. **Measure Incremental ROI**
   - Compare actual revenue and retention rates between treatment and control groups to estimate incremental business impact.

4. **Update Models and Action Policies**
   - Incorporate experimentally validated results into uplift models and retention action selection policies.

### Validation Objective

The long-term goal is to establish a continuous feedback loop:

```text
Counterfactual Estimation
    ↓
Intervention Selection
    ↓
A/B Testing or Holdout Validation
    ↓
Observed Revenue and Retention Outcomes
    ↓
Incremental ROI Measurement
    ↓
Uplift Model and Action Policy Updates
```

This process helps improve the reliability of future intervention recommendations and aligns model-based predictions with observed business outcomes.

---

## 4. Added Files

| File | Description |
| --- | --- |
| `src/optimization/counterfactual.py` | Core engine for counterfactual profit estimation |
| `dashboard/services/counterfactual_service.py` | Service wrapper for integrating counterfactual calculations into the dashboard |
| `dashboard/app.py` | Adds the new **Customer-Level Retention Strategy Comparison** view |

---

## 5. Summary

The Counterfactual Retention Lab extends traditional churn prediction by evaluating the expected financial consequences of alternative retention actions.

Rather than simply recommending an intervention, it provides an interpretable comparison of:

- Expected net profit
- Incremental profit relative to no intervention
- Intervention costs
- Estimated post-intervention churn probability
- Customer-specific intervention suitability

By combining churn predictions, uplift estimates, customer lifetime value, and intervention timing, the platform supports more economically informed retention decisions.

However, the estimated outcomes remain model-based approximations and require validation through controlled experiments before their causal effects can be established.

Ultimately, the Counterfactual Retention Lab aims to bridge the gap between **predictive analytics and economically justified retention intervention decisions**.

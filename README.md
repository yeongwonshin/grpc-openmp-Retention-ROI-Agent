<div align="center">

# Retention ROI Agent

**Distributed Retention ROI Decision Platform based on gRPC + OpenMP**

Retention ROI Agent is an operational retention intelligence copilot that goes beyond churn prediction and calculates **where, when, and how retention budget should be spent**. When a single CSV/TSV file is uploaded, the platform connects customer churn risk, expected churn timing, intervention effect, customer value, budget constraints, personalized actions, and a live action queue into one decision workflow.

[Demo Video](https://github.com/user-attachments/assets/a8b620c8-00bd-4ce2-9d33-da98e79b3fe2) · [Differentiation Strategy](docs/product_differentiation.md) · [Dashboard Decision Loop](docs/dashboard_decision_loop.md) · [Decision Logic](docs/decision_logic.md) · [Technical Guide](docs/technical_guide.md)

</div>

---

## Technical Core Summary

This project is not a simple churn dashboard. It is an end-to-end retention decision system that combines **distributed processing, parallel computation, a live operation queue, and decision optimization**.

| Area | Technology Stack / Design |
| --- | --- |
| Frontend dashboard | Streamlit dashboard for CSV onboarding, retention analysis, ROI simulation, and action-queue monitoring |
| Backend API | FastAPI-based API server for data ingestion, prediction, optimization, and live event processing |
| Distributed middleware | gRPC worker architecture that separates feature computation and ROI scoring services |
| Parallel computing | OpenMP C++ ROI scoring kernel for customer-level parallel computation |
| Load balancing | Nginx reverse proxy in front of scalable API replicas |
| Data layer | PostgreSQL live DB with Redis cache / queue support |
| ML / Analytics | Churn prediction, survival analysis, uplift-style scoring, CLV estimation, and budget optimization |
| Deployment | Docker Compose multi-service environment |
| Evaluation | API concurrency benchmark, throughput / latency measurement, and OpenMP scaling path |

---

## System Architecture

```text
Browser / Streamlit Dashboard
        |
        v
Nginx API Load Balancer
        |
        v
FastAPI API Replicas
        |
        +----------------------------+
        |                            |
        v                            v
PostgreSQL / Redis           gRPC Feature Worker
                                      |
                                      v
                              gRPC ROI Worker
                                      |
                                      v
                              OpenMP C++ ROI Kernel
                                      |
                                      v
                         Retention ROI / Action Decision
```

The platform separates user interaction, API orchestration, feature computation, and ROI scoring. This structure is easier to scale, benchmark, and extend than a monolithic churn application centered on a single dashboard.

---

## How This Project Differs from Existing Solutions

Most churn analytics tools stop at the following question:

> "Who is likely to churn?"

Retention ROI Agent goes one step further and asks:

> "Is it economically reasonable to retain this customer, what action should be taken, when should the intervention happen, and how should a limited retention budget be allocated?"

| Existing solution | Limitation | Retention ROI Agent |
| --- | --- | --- |
| Churn prediction notebooks | Focus on model scores and feature importance | Converts churn risk into retention ROI decisions |
| BI dashboards | Show status but do not optimize actions | Recalculates targets, budget, actions, and ROI under constraints |
| CRM campaign tools | Execute campaigns but do not evaluate marginal ROI | Ranks customers by expected incremental profit |
| Product analytics tools | Analyze funnels and behavior | Converts event logs into customer-level retention decisions |
| AutoML PoCs | Focus on predictive accuracy optimization | Integrates prediction, action, cost, timing, and live operation |

---

## Core Product Features

### 1. Industry-Aware CSV / TSV Onboarding

The platform supports both financial and e-commerce style datasets.

- Finance mode: deposits, loans, card transactions, balance changes, delinquency, consultation history
- E-commerce mode: visits, search, cart, purchase, coupons, category preferences

Uploaded data is automatically mapped into a standardized internal event schema.

<img src="assets/dash1.png" width="720" />

---

### 2. Automatic Schema Mapping and Event Standardization

Because every dataset uses different column names, the platform detects customer ID, timestamp, event type, monetary amount, category, and churn-label candidates, then normalizes them into an internal schema.

<img src="assets/dash2.png" width="720" />

---

### 3. Churn Status and Segment-Level Diagnosis

The dashboard summarizes total customers, high-risk customers, high-risk ratio, and average churn probability. This is the operational starting point before survival analysis, uplift, CLV, and budget analysis.

<img src="assets/dash3.png" width="720" />

---

### 4. Survival-Analysis-Based Churn Timing Prediction

The system estimates expected churn timing and intervention windows such as:

- Immediate contact within 14 days
- Planned contact within 15 to 30 days
- Lower-urgency contact within 31 to 60 days

<img src="assets/dash5.png" width="720" />

---

### 5. Budget Allocation and Marginal ROI Optimization

Given total budget, churn threshold, and maximum target count, the system recalculates:

- Final target customers under the budget constraint
- Segment-level budget allocation
- Expected incremental profit
- Marginal ROI of additional budget
- Budget saturation zones and low-efficiency spending zones
- High-intensity action caps to avoid excessive cost concentration

<img src="assets/dash6.png" width="720" />

<img src="assets/dash7.png" width="720" />

<img src="assets/dash8.png" width="720" />

---

### 6. Counterfactual Retention Lab

For a selected customer, the platform compares multiple intervention scenarios.

- No action
- Coupon / monetary benefit
- Consultation call
- Push / email
- Wait 7 days

The recommendation does not merely choose one action; it also provides an economic comparison against alternatives.

<img src="assets/dash9.png" width="720" />

<img src="assets/dash10.png" width="720" />

---

### 7. Personalized Recommendations for Final Targets

Personalized recommendations are generated only for customers selected under the current budget, churn threshold, and target-limit constraints. Ranking combines individual history, recent interest signals, similar-segment preferences, and overall popularity signals.

<img src="assets/dash11.png" width="720" />

---

### 8. Live Operation Monitor and Action Queue

When a new event arrives in the PostgreSQL live DB, the system updates customer state, churn score, recommendation candidates, and the expected-ROI action queue.

- FastAPI event ingestion
- Customer feature-state update
- Churn / CLV / uplift rescoring
- Expected-ROI action-queue insertion
- Demo event stream for real-time simulation

<img src="assets/dash12.png" width="720" />

---

### 9. Context-Aware AI Assistant

The LLM assistant does not blindly read the entire database. It answers questions based on the current dashboard state. Users can ask questions such as:

- "Why is this segment risky?"
- "What changes if the budget increases?"
- "Which customers should be contacted first?"

<img src="assets/dash4.png" width="260" />

---

## Distributed and Parallel Execution Path

The retention decision pipeline is integrated into the platform backend rather than remaining a toy benchmark.

```text
CSV Upload / Optimization Request
  -> FastAPI API
  -> gRPC Feature Worker
  -> gRPC ROI Worker
  -> OpenMP C++ ROI Kernel
  -> Ranked retention decisions
  -> Dashboard response
```

### gRPC Worker Separation

Feature computation and ROI scoring are separated into independent worker services. This makes the backend easier to scale and keeps heavy computation away from user-facing API replicas.

### OpenMP ROI Scoring

Customer-level ROI scoring is well suited to parallelization because each customer's expected incremental profit can be computed independently. The ROI worker delegates computation to a C++ OpenMP kernel to accelerate scoring for large customer batches.

### API Load Balancing

Nginx sits in front of the FastAPI service so multiple API replicas can handle dashboard and optimization requests.

```bash
docker compose up -d --build --scale api=2
```

This provides a practical path for traffic handling, horizontal scaling, and concurrency testing.

---

## Decision Flow

```text
CSV / TSV Upload
  -> Column role detection
  -> Event normalization
  -> Churn label configuration
  -> Churn / Survival / Uplift / CLV computation
  -> gRPC feature processing
  -> OpenMP ROI scoring
  -> Budget-constrained target selection
  -> Personalized recommendation
  -> PostgreSQL live DB seeding
  -> Live event ingestion
  -> Score and action queue update
```

The core logic is designed to answer real operational questions rather than a simple prediction question.

| Question | Computed result |
| --- | --- |
| Who is at risk? | Churn probability and risky segments |
| When should intervention happen? | Expected churn timing, urgency, and intervention window |
| Is retention economically reasonable? | CLV, expected loss, and expected incremental profit |
| Who is persuadable? | Uplift-style score and persuadable segment |
| How much should be spent? | Action cost, budget allocation, and marginal ROI |
| What should be done? | Recommended action, intervention intensity, and next-best recommendation |
| Should the customer enter the live queue? | Live score, expected ROI, and action-queue status |

---

## Supported Domains

| Mode | Industry | Example data | Main decisions |
| --- | --- | --- | --- |
| Finance mode | Banking, cards, fintech, insurance, wealth management | Transactions, balance, loans, repayment, delinquency, consultation logs | Account closing / dormancy risk, consultation priority, product recommendation |
| E-commerce mode | Online shopping, subscription commerce, marketplaces | Visits, search, cart, purchases, coupons, category preference | Repurchase targeting, coupon strategy, category recommendation, CRM queue |

---

## Quick Start

```bash
# 1. Build and run services
docker compose up -d --build

# 2. Open dashboard
open http://localhost:8501
```

To scale API replicas behind the load balancer, use:

```bash
docker compose up -d --build --scale api=2
```

To build the OpenMP ROI kernel separately, use:

```bash
./scripts/build_openmp_roi.sh
```

To run the basic integration benchmark, use:

```bash
python scripts/benchmark_parallel_distributed.py
```

Use the benchmark commands below to reproduce the detailed assignment evaluation.

---

## Assignment Requirements

This project includes all core requirements of the Parallel & Distributed Computing assignment: **distributed middleware, parallel acceleration, structural decoupling, and basic scalability analysis**.

| Assignment requirement | Project implementation | Status |
| --- | --- | --- |
| Parallel Computing | C++ OpenMP ROI kernel computes customer-level ROI with `#pragma omp parallel for` | Covered |
| Distributed Middleware | FastAPI calls the gRPC Feature Worker and gRPC ROI Worker | Covered |
| Integrated Pipeline | CSV/TSV upload -> FastAPI -> gRPC Feature Worker -> gRPC ROI Worker -> OpenMP Kernel -> dashboard/action queue | Covered |
| Structural Decoupling | Dashboard, API, load balancer, feature worker, ROI worker, DB/cache layers are separated | Covered |
| Scalability Evaluation | API concurrency, OpenMP thread count, and distributed on/off experiments are included | Covered |
| Reproducibility | Docker Compose, OpenMP build script, and benchmark scripts are provided | Covered |

---

## Implementation Details Summary

| Item | Configuration |
| --- | --- |
| Middleware configuration | `feature-worker:50051`, `roi-worker:50052` gRPC services |
| Worker address settings | `RETENTION_FEATURE_WORKER_ADDRESS`, `RETENTION_ROI_WORKER_ADDRESS` environment variables |
| Serialization | JSON-over-gRPC payload transfer |
| ROI kernel integration | ROI Worker writes feature batches to temporary CSV, invokes the OpenMP C++ kernel, and reads the scored CSV result |
| Parallelization | Independent customer-row ROI computation with a `schedule(static)` OpenMP loop |
| Synchronization | Customer computations are independent; the implicit barrier at the end of the parallel loop synchronizes results |

---

## Performance Evaluation Summary

Evaluation was performed from three perspectives.

1. End-to-end throughput / latency as API concurrency increases
2. ROI kernel speedup as the OpenMP thread count increases
3. Comparison between local in-process path and gRPC + OpenMP distributed path

### Evaluation Environment and Conditions

| Item | Setting |
| --- | --- |
| Deployment method | Single-machine Docker Compose multi-container environment |
| Main services | dashboard, api, api-lb, feature-worker, roi-worker, postgres, redis |
| API concurrency test | Concurrency 1, 2, 4, 8 / 12 optimize requests per level |
| OpenMP test | 200,000 customer rows / threads 1, 2, 4, 8 / mean of three repetitions |
| Comparison test | Local in-process path and gRPC + OpenMP path with the same optimize workload |

### API concurrency scaling

| Concurrent requests | Throughput (req/s) | Average latency (ms) | P95 latency (ms) |
| ---: | ---: | ---: | ---: |
| 1 | 1.82 | 548 | 782 |
| 2 | 3.24 | 612 | 960 |
| 4 | 4.78 | 835 | 1,585 |
| 8 | 5.36 | 1,268 | 2,920 |

Throughput increases almost linearly as concurrent requests rise from 1 to 4. At 8 requests, throughput growth slows and P95 latency increases sharply. This is interpreted as the point where worker waiting time, gRPC serialization, and ROI batch queueing combine to increase tail latency.

### OpenMP ROI kernel thread scaling

| Threads | Average kernel time (ms) | Throughput (rows/s) | Speedup | Efficiency |
| ---: | ---: | ---: | ---: | ---: |
| 1 | 1,264 | 158,228 | 1.00 | 1.00 |
| 2 | 676 | 295,858 | 1.87 | 0.94 |
| 4 | 376 | 531,915 | 3.36 | 0.84 |
| 8 | 258 | 775,194 | 4.90 | 0.61 |

OpenMP parallelization reduced kernel time from 1,264 ms with one thread to 258 ms with eight threads for 200,000 customer rows, achieving up to 4.90x speedup. However, parallel efficiency decreased as thread count increased due to CSV I/O, memory-bandwidth contention, and thread-scheduling overhead.

### Distributed on/off comparison

| Execution mode | Throughput (req/s) | Average latency (ms) | P95 latency (ms) | Interpretation |
| --- | ---: | ---: | ---: | --- |
| Local in-process | 3.62 | 1,510 | 3,810 | API process directly carries computation, increasing tail latency under concurrency |
| gRPC + OpenMP | 5.36 | 1,268 | 2,920 | ROI computation is separated into workers and the C++ kernel, reducing the API bottleneck |

The distributed execution path achieved about 48% higher throughput and lower P95 latency than the local in-process path. Although gRPC serialization and network boundaries add overhead, separating computation from the API process and using OpenMP inside the ROI Worker improves overall pipeline throughput under concurrent requests.

---

## Performance Evaluation Reproduction Procedure

The entire system can be run with Docker Compose, and the OpenMP kernel can be reproduced through a separate build script.

| Purpose | Command |
| --- | --- |
| Run services | `docker compose up -d --build` |
| Scale API replicas | `docker compose up -d --build --scale api=2` |
| Build OpenMP kernel | `./scripts/build_openmp_roi.sh` |
| API concurrency evaluation | `python scripts/benchmark_api_scaling.py --concurrency-levels 1,2,4,8` |
| OpenMP thread evaluation | `python scripts/benchmark_openmp_threads.py --rows 200000 --threads 1,2,4,8 --rebuild` |
| Distributed On/Off comparison | `python scripts/benchmark_distributed_on_off.py --distributed-url http://localhost:8000 --local-url http://localhost:8001` |
| Run all evaluations | `python scripts/run_all_performance_evaluation.py --api-base-url http://localhost:8000 --local-api-base-url http://localhost:8001` |

Detailed graphs and interpretations are organized in the final report's performance evaluation and bottleneck analysis sections. The README focuses on execution and reproduction, while the final report contains quantitative results and graph interpretation.

---

## Bottleneck Analysis Summary

| Bottleneck factor | Observation | Improvement direction |
| --- | --- | --- |
| gRPC/JSON serialization | Payload conversion and RPC response waiting accumulate as concurrency increases | Move to Protocol Buffers schema-based typed messages and batch compression |
| CSV-based kernel integration | ROI Worker writes and reads temporary CSV files to invoke the OpenMP kernel | Remove I/O overhead through stdin/stdout streaming or direct pybind11 integration |
| OpenMP 8-thread efficiency drop | Speedup increases but parallel efficiency decreases | Tune thread count by batch size, set NUMA/affinity, and optimize memory access |
| API tail latency | P95 latency increases sharply at concurrency 8 | Add API replicas, increase worker `max_workers`, and apply request-queue backpressure |

The most important bottleneck is not a single component. It is the tail-latency region where serialization, worker queueing, kernel I/O, and reduced thread efficiency appear together. Therefore, performance improvement should tune batch boundaries, serialization, worker parallelism, and API replica count together rather than only increasing the number of OpenMP threads.

---

## Repository Structure

```text
.
├── src/
│   ├── api/                    # FastAPI backend
│   ├── dashboard/              # Streamlit dashboard
│   ├── distributed/            # gRPC client / worker integration
│   ├── hpc/                    # OpenMP C++ ROI kernel
│   ├── optimization/           # Budget and ROI optimization logic
│   ├── models/                 # Churn / CLV / survival modeling
│   └── realtime/               # Live DB and action queue processing
├── deploy/
│   └── nginx.conf              # API load balancer configuration
├── scripts/
│   ├── build_openmp_roi.sh
│   ├── benchmark_parallel_distributed.py
│   ├── benchmark_api_scaling.py
│   ├── benchmark_openmp_threads.py
│   ├── benchmark_distributed_on_off.py
│   └── run_all_performance_evaluation.py
├── docs/
├── assets/
├── docker-compose.yml
├── Dockerfile.api
└── requirements.txt
```

---

## Demonstrated Technical Competencies

- End-to-end ML product design from upload to action queue
- FastAPI backend service design
- Streamlit analytics dashboard development
- gRPC-based distributed worker architecture
- OpenMP C++ parallel computation integration
- Nginx load balancing for horizontally scalable API replicas
- Docker Compose multi-service deployment
- PostgreSQL-based live operational state
- Redis-compatible cache / queue architecture
- Retention ROI, CLV, and uplift-style decision modeling
- Budget-constrained optimization
- API-level benchmark and latency / throughput evaluation
- OpenMP thread-count scaling and speedup analysis
- Comparison between gRPC + OpenMP distributed path and local in-process path
- Practical fallback design from distributed engine to legacy computation path

---

## Documents

| Document | Description |
| --- | --- |
| [Differentiation Strategy](docs/product_differentiation.md) | How this project differs from churn dashboards, CRM tools, CDPs, and BI tools |
| [Dashboard Decision Loop](docs/dashboard_decision_loop.md) | How dashboard screens connect into a single decision workflow |
| [Decision Logic](docs/decision_logic.md) | Budget optimization, counterfactual analysis, recommendation, and live action queue |
| [Technical Guide](docs/technical_guide.md) | Installation, API, repository structure, and validation checklist |
| [Analysis Process](docs/analysis_process.md) | Modeling, survival analysis, and live deployment flow |
| [Feature Dictionary](docs/feature_dictionary.md) | Feature definitions and meanings |
| [Retention Strategy](docs/retention_strategy.md) | Segment-level retention strategy and cost / effect assumptions |
| [Counterfactual Lab](docs/counterfactual_retention_lab.md) | Expected-profit comparison by retention action |

---

## Project Positioning

Retention ROI Agent is positioned as a decision intelligence system for retention operations. It combines predictive modeling, customer value estimation, budget-aware action selection, distributed backend execution, and live operation monitoring into one workflow.

The goal is not simply to predict churn. The goal is to answer:

> "When budget and operational capacity are limited, which customers should be retained first, what action should be taken, and why?"

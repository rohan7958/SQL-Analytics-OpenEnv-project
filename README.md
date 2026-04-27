<div align="center">

# 🗄️ SQL Analytics OpenEnv

**An OpenEnv-compliant RL environment for evaluating LLM agents on real-world SQL query generation tasks**

[![OpenEnv](https://img.shields.io/badge/tag-openenv-4A90D9?style=flat-square)](https://huggingface.co/spaces/YOUR_USERNAME/sql-analytics-env)
[![HuggingFace Space](https://img.shields.io/badge/🤗-HF%20Space-yellow?style=flat-square)](https://huggingface.co/spaces/YOUR_USERNAME/sql-analytics-env)
[![Python 3.11](https://img.shields.io/badge/Python-3.11-blue?style=flat-square&logo=python)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.110-009688?style=flat-square&logo=fastapi)](https://fastapi.tiangolo.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)

[Overview](#-overview) · [Tasks](#-tasks) · [Reward Function](#-reward-function) · [Quick Start](#-quick-start) · [API Reference](#-api-reference) · [Baseline Scores](#-baseline-scores)

</div>

---

## 📖 Overview

**SQL Analytics OpenEnv** is a fully spec-compliant [OpenEnv](https://github.com/meta-pytorch/OpenEnv) environment designed to benchmark LLM agents on the task of **Text-to-SQL query generation** — one of the most commercially valuable real-world agent capabilities.

Agents are presented with a natural-language business question, a relational schema, and sample data rows. They must generate a correct SQL `SELECT` query within a limited number of steps. Each attempt is graded immediately using a **deterministic, reproducible scoring function** that provides partial credit and actionable feedback — enabling multi-step self-correction.

### Why SQL Analytics?

- ✅ **Real-world task** — SQL querying is a core skill in data analysis, business intelligence, and product analytics
- ✅ **Fully deterministic grading** — Jaccard-based scoring with zero randomness; same input always yields the same score
- ✅ **Incremental reward signal** — agents receive graded feedback at every step, not just pass/fail at the end
- ✅ **Progressive difficulty** — three tasks covering filtering → aggregation → window functions
- ✅ **Zero external dependencies** — in-memory SQLite only; no databases, no APIs, no volumes

---

## 🗂️ Project Structure
sql_analytics_env/
├── inference.py ← Baseline LLM evaluation script (MANDATORY: root, exact name)
├── sql_analytics_env.py ← Async client wrapper (imported by inference.py)
├── openenv.yaml ← OpenEnv metadata + task definitions
├── Dockerfile ← Containerised deployment
├── requirements.txt
└── server/
├── _init_.py
├── models.py ← Pydantic v2: Action, Observation, Reward, State
├── data_generator.py ← In-memory SQLite DB creation + ground-truth answers
├── graders.py ← Deterministic Jaccard-based scoring
├── environment.py ← Core reset() / step() / state() class
└── server.py ← FastAPI REST server


---

## 🎯 Tasks

Three tasks of increasing difficulty, each with a clearly defined objective and a programmatic grader:

### 🟢 Easy — Basic SELECT with Filtering

> **Table:** `sales` (15 rows — customer purchase records across Indian states)
>
> **Question:** *"Return the customer name, city, and purchase amount for all customers from Karnataka who spent more than ₹5,000. Sort by amount descending."*

**Target SQL pattern:**
```sql
SELECT customer_name, city, amount
FROM sales
WHERE state = 'Karnataka' AND amount > 5000
ORDER BY amount DESC;
```

**Key concepts tested:** `WHERE` with multiple conditions, `ORDER BY`, column selection  
**Expected output:** 7 rows

---

### 🟡 Medium — Multi-Table JOIN with Aggregation

> **Tables:** `customers`, `products`, `orders` (3 normalised tables)
>
> **Question:** *"Find the top 3 product categories by total revenue (quantity × unit price). Include the count of distinct customers per category. Return columns: category, total_revenue, unique_customers."*

**Target SQL pattern:**
```sql
SELECT p.category,
       SUM(o.quantity * p.unit_price) AS total_revenue,
       COUNT(DISTINCT o.customer_id)  AS unique_customers
FROM orders o
JOIN products p ON o.product_id = p.product_id
GROUP BY p.category
ORDER BY total_revenue DESC
LIMIT 3;
```

**Key concepts tested:** `JOIN`, `GROUP BY`, `SUM`, `COUNT(DISTINCT ...)`, column aliasing, `LIMIT`  
**Expected output:** 3 rows (Electronics, Furniture, Appliances)

---

### 🔴 Hard — Window Function Salary Ranking

> **Table:** `employees` (12 rows across Engineering, Marketing, HR)
>
> **Question:** *"For each employee, show their emp_id, name, department, salary, and salary rank within their department using `RANK()`. Order results by department ASC, salary rank ASC."*

**Target SQL pattern:**
```sql
SELECT emp_id, name, department, salary,
       RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS salary_rank
FROM employees
ORDER BY department ASC, salary_rank ASC;
```

**Key concepts tested:** `RANK() OVER (PARTITION BY ... ORDER BY ...)`, tied-rank behaviour  
**Expected output:** 12 rows with correct rank assignments including tie handling

---

## ⚖️ Reward Function

Every task uses a **three-component scoring formula** that gives partial credit at each step:
score = column_score (max 0.20)
+ row_score (max 0.20)
+ value_score (max 0.60) ← Jaccard similarity


| Component | How It's Calculated | Max Points |
|-----------|---------------------|------------|
| **Column match** | Are the expected columns present in the result? Exact = 0.20, subset = 0.15, overlap = proportional | 0.20 |
| **Row count** | `0.20 × (1 − min(|n_result − n_expected| / n_expected, 1))` | 0.20 |
| **Value overlap** | Jaccard similarity: `\|result ∩ expected\| / \|result ∪ expected\|` on normalised row tuples | 0.60 |

### Reward Schedule

| Situation | Reward |
|-----------|--------|
| Destructive SQL keyword (DROP / DELETE / INSERT / UPDATE / ALTER / TRUNCATE) | **-0.10** |
| SQL syntax error or runtime exception | **-0.05** |
| Partial answer | 0.0 – 0.89 |
| Correct answer (episode terminates) | **≥ 0.90** |

---

## 🔭 Observation & Action Spaces

### Action Space
```json
{
  "query": "SELECT customer_name, city, amount FROM sales WHERE ..."
}
```
Only `SELECT` queries are accepted. Destructive operations are blocked and penalised.

### Observation Space
```json
{
  "task_id": "easy",
  "task_description": "Return customer_name, city, amount for Karnataka customers...",
  "schema_info": "CREATE TABLE sales (sale_id INTEGER, customer_name TEXT, ...)",
  "sample_data": "-- sales: ['sale_id', 'customer_name', 'city', ...]\n   [1, 'Ravi Kumar', ...]",
  "result": [{"customer_name": "Ravi Kumar", "city": "Bengaluru", "amount": 12500.0}],
  "error": null,
  "step_count": 1,
  "max_steps": 5
}
```

---

## 🚀 Quick Start

### Prerequisites

```bash
python --version   # 3.10+
docker --version   # any recent version
```

### Option A — Local Development

```bash
git clone https://github.com/YOUR_USERNAME/sql-analytics-env
cd sql-analytics-env

python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

uvicorn server.server:app --host 0.0.0.0 --port 7860 --reload
# Visit: http://localhost:7860/docs
```

### Option B — Docker

```bash
docker build -t sql-analytics-env .
docker run -p 7860:7860 sql-analytics-env

curl http://localhost:7860/health
# {"status":"ok","env":"sql-analytics-env","version":"1.0.0"}
```

### Smoke Test

```bash
# Reset to easy task
curl -s -X POST http://localhost:7860/reset \
  -H "Content-Type: application/json" \
  -d '{"task_id": "easy"}'

# Submit a query
curl -s -X POST http://localhost:7860/step \
  -H "Content-Type: application/json" \
  -d '{"action": {"query": "SELECT customer_name, city, amount FROM sales WHERE state = '\''Karnataka'\'' AND amount > 5000 ORDER BY amount DESC"}}'
# Expected: "reward": 0.92, "done": true
```

### Run Baseline Inference

```bash
export HF_TOKEN=hf_YOUR_TOKEN_HERE
python inference.py
```

---

## 📡 API Reference

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| `GET` | `/health` | — | Liveness check |
| `POST` | `/reset` | `{"task_id": "easy\|medium\|hard"}` | Start a new episode → returns initial observation |
| `POST` | `/step` | `{"action": {"query": "SELECT ..."}}` | Execute one SQL action → returns observation, reward, done, info |
| `GET` | `/state` | — | Snapshot of current episode (task, step count, done flag) |

---

## 📊 Baseline Performance Scores

Evaluated using `Qwen/Qwen2.5-72B-Instruct` at `temperature=0.0`:

| Task | Score | Steps to Solve | Notes |
|------|-------|----------------|-------|
| `easy` | **0.92** | 1 | Solved in a single step |
| `medium` | **0.78** | 2 | Requires JOIN correction on step 2 |
| `hard` | **0.65** | 3 | Window function syntax refined over steps |
| **Average** | **0.78** | — | Across all 3 tasks |

---

## 🛠️ Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `HF_TOKEN` | ✅ Yes | — | Hugging Face API token |
| `API_BASE_URL` | No | `https://router.huggingface.co/v1` | LLM inference endpoint |
| `MODEL_NAME` | No | `Qwen/Qwen2.5-72B-Instruct` | Model identifier |
| `IMAGE_NAME` | No | — | Local Docker image name (for Docker-based testing) |

---

## 📋 Inference Output Format

The script emits exactly three line types to stdout:
START] task=easy env=sql-analytics-env model=Qwen/Qwen2.5-72B-Instruct
[STEP] step=1 action=SELECT customer_name, city, amount FROM sales WHERE state = 'Karnataka' AND amount > 5000 ORDER BY amount DESC reward=0.92 done=true error=null
[END] success=true steps=1 score=0.92 rewards=0.92


---

## 🧰 Tech Stack

| Component | Technology |
|-----------|-----------|
| Environment Logic | Python 3.11 + SQLite (stdlib) |
| Type Safety | Pydantic v2 |
| REST Server | FastAPI + Uvicorn |
| Data Layer | Pandas + in-memory SQLite |
| LLM Client | OpenAI Python SDK |
| Deployment | Docker + Hugging Face Spaces |

---

## 📁 Key Design Decisions

**In-memory SQLite** — every `reset()` call spawns a fresh database in RAM. No files, no Docker volumes, no cleanup needed. Episodes are fully stateless and reproducible.

**Jaccard-based grading** — handles partial credit naturally. An agent that returns 5 of 7 expected rows scores proportionally, not zero. This provides a smooth gradient signal for multi-step improvement.

**Incremental reward signal** — reward is computed and returned at every `step()` call, not just at episode end. Agents can observe their progress and self-correct across attempts.

**Deterministic by design** — the same SQL query always returns the same score. No LLM calls anywhere in the grading pipeline.

---

## 🤝 Contributing

Pull requests are welcome. For major changes, please open an issue first.

---

## 📄 License

This project is licensed under the [MIT License](LICENSE).

---

<div align="center">
Built for the <strong>Meta × PyTorch OpenEnv Hackathon</strong> · Powered by <a href="https://github.com/meta-pytorch/OpenEnv">OpenEnv</a>
</div>

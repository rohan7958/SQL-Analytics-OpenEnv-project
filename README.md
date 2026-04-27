# SQL-Analytics-OpenEnv-project
An OpenEnv-compliant RL environment for evaluating LLM agents on real-world SQL query generation tasks

---
title: SQL Analytics OpenEnv
emoji: 🗄️
colorFrom: blue
colorTo: green
sdk: docker
app_port: 7860
tags:
  - openenv
  - sql
  - text-to-sql
  - analytics
license: mit
---

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

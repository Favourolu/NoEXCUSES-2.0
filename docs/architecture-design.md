---

# EconometricAI — System Architecture & Design Document

**Version:** 1.0
**Date:** March 12, 2026
**Author:** Favourolu
**Status:** Draft

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Problem Statement](#2-problem-statement)
3. [Market Analysis](#3-market-analysis)
4. [System Architecture](#4-system-architecture)
5. [Component Design](#5-component-design)
6. [Data Flow & Pipeline](#6-data-flow--pipeline)
7. [Decision Engine Logic](#7-decision-engine-logic)
8. [Technology Stack](#8-technology-stack)
9. [Project Structure](#9-project-structure)
10. [Use Cases & Examples](#10-use-cases--examples)
11. [Success Metrics](#11-success-metrics)
12. [Risks & Mitigations](#12-risks--mitigations)
13. [Build Roadmap](#13-build-roadmap)

---

## 1. Executive Summary

**EconometricAI** is an AI-powered system that automates econometric testing to determine the best statistical model for a given dataset and research question. It combines a **rule-based decision engine** with **LLM reasoning** (Claude API) to deliver accurate, explainable, and validated econometric analysis.

The system addresses a clear market gap: while general AI automation tools are abundant, no dominant commercial product exists for automated econometric model selection and testing. Academic research (arXiv, June 2025) demonstrates that specialized AI agents achieve **93% task completion** and **87% coefficient direction accuracy**, far exceeding generic LLMs (<50% completion).

---

## 2. Problem Statement

### The Current Pain

Econometric analysis is a **manual, time-consuming, expertise-heavy** process:

- Researchers spend hours selecting the right model (OLS vs. IV vs. ARIMA vs. VECM)
- Assumption testing is often skipped or done incorrectly
- Results interpretation requires deep statistical knowledge
- Errors in model selection lead to **spurious results and wrong conclusions**

### The Opportunity

| Problem | EconometricAI Solution |
|---------|----------------------|
| Manual test selection | Automated rule-based test sequencing |
| Skipped assumptions | Enforced assumption checks before model fitting |
| Interpretation difficulty | LLM-generated plain-English explanations |
| Model selection uncertainty | Data-driven recommendation with confidence scores |
| No audit trail | Full decision log from data upload to final report |

---

## 3. Market Analysis

### Existing Solutions

| Product | Type | Limitation |
|---------|------|-----------|
| Econometrics AI Agent (arXiv) | Academic research | Not a commercial product |
| Economic AI (economicai.com) | Causal AI platform | Enterprise-focused, not automated testing |
| YesChat Econometrics GPTs | ChatGPT wrappers | Lightweight, no real test execution |
| Stata / EViews / R | Traditional software | Manual, requires expertise |
| statsmodels (Python) | Library | Code-level, no decision logic |

### Market Gap

**No product combines:** automated test execution + intelligent model selection + assumption validation + plain-English reporting.

### Target Users

- Academic researchers (economics, finance, social sciences)
- Graduate students writing theses
- Policy analysts at government agencies
- Financial analysts at banks and funds
- Data scientists entering econometrics

---

## 4. System Architecture

### High-Level Overview

```
┌──────────────────────────────────────────────────────────┐
│                     USER INTERFACE                        │
│         Upload CSV/Excel + Research Question              │
└────────────────────────┬─────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────┐
│                                                          │
│    ┌──────────┐   ┌──────────┐   ┌──────────────────┐   │
│    │   DATA   │──▶│   RULE   │──▶│  TEST EXECUTOR   │   │
│    │ PROFILER │   │  ENGINE  │   │  (statsmodels)   │   │
│    └──────────┘   └──────────┘   └────────┬─────────┘   │
│                                           │              │
│                        ┌──────────────────┘              │
│                        ▼                                 │
│              ┌──────────────────┐                        │
│              │  LLM REASONING   │◀─── Claude API         │
│              │     LAYER        │                        │
│              └────────┬─────────┘                        │
│                       │                                  │
│                       ▼                                  │
│              ┌──────────────────┐                        │
│              │    VALIDATOR     │──▶ Loop back if invalid │
│              │  (Guardrails)    │                        │
│              └────────┬─────────┘                        │
│                       │                                  │
│              PIPELINE ORCHESTRATOR                        │
│                                                          │
└────────────────────────┬─────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────┐
│                   OUTPUT REPORT                           │
│    Best Model + Tests + Confidence + Audit Trail          │
└──────────────────────────────────────────────────────────┘
```

### Design Principles

1. **Sandwich Pattern** — Rule engine and validator wrap the LLM. Rules control input, validator checks output.
2. **Fail Safe** — If the LLM gives bad output, the validator catches it and loops back with error context.
3. **Explainability** — Every decision is logged. Users can see *why* each test was run and *why* a model was chosen.
4. **Modularity** — Each component is independent and testable in isolation.

---

## 5. Component Design

### 5.1 Data Profiler

**Purpose:** Automatically detect data characteristics from uploaded files.

**Input:** Raw CSV/Excel file
**Output:** `DataProfile` object

| Detection | Method |
|-----------|--------|
| Data type | Time series vs. panel vs. cross-sectional |
| Frequency | Daily, weekly, monthly, quarterly, annual |
| Variable types | Continuous, categorical, binary, count |
| Missing data | Count, pattern (MCAR, MAR, MNAR) |
| Outliers | IQR method, Z-score |
| Summary stats | Mean, median, std, skewness, kurtosis |
| Dimensions | N observations, K variables |

```python
class DataProfile:
    data_type: DataType           # TIME_SERIES | PANEL | CROSS_SECTION
    frequency: Frequency | None   # DAILY | MONTHLY | QUARTERLY | ANNUAL
    n_obs: int
    n_vars: int
    variables: list[VariableProfile]
    missing_summary: MissingSummary
    outlier_summary: OutlierSummary
```

---

### 5.2 Rule Engine

**Purpose:** Enforce mandatory assumption tests based on data profile. Determine valid econometric methods.

**Input:** `DataProfile`
**Output:** `RequiredTests[]` + `ValidMethods[]`

#### Rule Table

| Data Type | Condition | Required Tests | Valid Methods |
|-----------|-----------|---------------|---------------|
| Time Series | Always | ADF, KPSS, Ljung-Box | — |
| Time Series | Non-stationary | Johansen cointegration | VECM, ARDL |
| Time Series | Stationary | — | ARIMA, VAR, OLS |
| Time Series | Always | Structural break (Chow) | — |
| Cross-Section | Always | VIF (multicollinearity) | OLS, Logit, Probit |
| Cross-Section | Always | Breusch-Pagan (heteroskedasticity) | — |
| Cross-Section | Heteroskedastic | — | Robust SE, WLS |
| Cross-Section | Binary DV | — | Logit, Probit |
| Cross-Section | Endogeneity suspected | Hausman, Sargan | IV/2SLS, GMM |
| Panel | Always | Hausman (FE vs RE) | FE, RE, Pooled OLS |
| Panel | Always | Breusch-Pagan LM | — |
| Panel | Serial correlation | Wooldridge test | Clustered SE |

#### Rule Priority

```
Priority 1 (BLOCKING):  Stationarity tests for time series
Priority 2 (BLOCKING):  Multicollinearity check (VIF > 10 = stop)
Priority 3 (REQUIRED):  Heteroskedasticity, autocorrelation
Priority 4 (ADVISORY):  Normality of residuals, structural breaks
```

BLOCKING tests must pass before any model is fitted. REQUIRED tests inform model choice. ADVISORY tests are reported but don't block.

---

### 5.3 Test Executor

**Purpose:** Run statistical tests and return structured results.

**Input:** Dataset + test name + parameters
**Output:** `TestResult` object

#### Supported Tests

| Category | Test | Library | Output |
|----------|------|---------|--------|
| Stationarity | ADF (Augmented Dickey-Fuller) | statsmodels | Statistic, p-value, lags |
| Stationarity | KPSS | statsmodels | Statistic, p-value |
| Cointegration | Johansen | statsmodels | Trace stat, max eigenvalue, rank |
| Cointegration | Engle-Granger | statsmodels | p-value, residual stat |
| Autocorrelation | Ljung-Box | statsmodels | Q-stat, p-values by lag |
| Autocorrelation | Durbin-Watson | statsmodels | DW statistic |
| Heteroskedasticity | Breusch-Pagan | statsmodels | LM stat, p-value |
| Heteroskedasticity | White's test | statsmodels | F-stat, p-value |
| Multicollinearity | VIF | statsmodels | VIF per variable |
| Normality | Jarque-Bera | scipy | JB stat, p-value |
| Normality | Shapiro-Wilk | scipy | W stat, p-value |
| Model selection | Hausman (FE vs RE) | linearmodels | Chi-sq, p-value |
| Structural break | Chow test | custom | F-stat, p-value |
| Instrument validity | Sargan/Hansen | linearmodels | J-stat, p-value |
| Causality | Granger causality | statsmodels | F-stat, p-value by lag |

```python
class TestResult:
    test_name: str
    statistic: float
    p_value: float
    conclusion: str          # "REJECT_H0" | "FAIL_TO_REJECT_H0"
    significance: str        # "1%" | "5%" | "10%" | "NOT_SIGNIFICANT"
    details: dict            # Test-specific extra data
    implication: str         # What this means for model choice
```

---

### 5.4 LLM Reasoning Layer

**Purpose:** Make soft decisions that require judgment — choosing between valid models, handling borderline results, explaining findings.

**Input:** `DataProfile` + `TestResult[]` + `ValidMethods[]`
**Output:** `ModelRecommendation`

#### When the LLM Decides

| Scenario | LLM Role |
|----------|----------|
| Multiple valid models | Rank by fit, parsimony, interpretability |
| Borderline p-values (0.05-0.10) | Assess robustness, suggest sensitivity analysis |
| Contradictory tests | Weigh evidence, explain trade-offs |
| Data quality issues | Recommend preprocessing steps |
| Result interpretation | Generate plain-English summary |

#### Prompt Structure

```
You are an expert econometrician. Given the following:

DATA PROFILE:
{data_profile_json}

TEST RESULTS:
{test_results_json}

VALID METHODS (from rule engine):
{valid_methods}

TASK: Recommend the best econometric model and explain your reasoning.

CONSTRAINTS:
- You MUST only recommend from the ValidMethods list
- You MUST reference specific test results to justify your choice
- If results are borderline, say so and suggest robustness checks
- Provide a confidence score: HIGH / MEDIUM / LOW
```

```python
class ModelRecommendation:
    recommended_model: str
    reasoning: str
    confidence: Confidence       # HIGH | MEDIUM | LOW
    alternative_models: list[str]
    robustness_checks: list[str]
    interpretation: str          # Plain-English summary
    next_steps: list[str]
```

---

### 5.5 Validator

**Purpose:** Verify LLM output against hard constraints. Prevent hallucinations and invalid recommendations.

**Input:** `ModelRecommendation` + `RequiredTests[]` + `TestResult[]` + `ValidMethods[]`
**Output:** `ValidationResult` (PASS or FAIL with reasons)

#### Validation Checks

| Check | Rule | On Fail |
|-------|------|---------|
| Model validity | recommended_model in ValidMethods | REJECT → re-prompt LLM |
| Test coverage | All RequiredTests have results | REJECT → run missing tests |
| Statistical consistency | Claims match actual p-values | REJECT → re-prompt with correction |
| Blocking test compliance | No model fitted before blocking tests pass | REJECT → halt pipeline |
| Confidence calibration | LOW confidence → flag for human review | WARN → add disclaimer |

#### Re-prompt Loop

```
Max retries: 3

Attempt 1: Standard prompt
Attempt 2: Add specific error context ("You recommended X but it's not in ValidMethods. Choose from: [...]")
Attempt 3: Add chain-of-thought requirement
After 3 fails: Return error with partial results + flag for human review
```

---

### 5.6 Pipeline Orchestrator

**Purpose:** Wire all components together. Manage state transitions and the feedback loop.

#### State Machine

```
START
  │
  ▼
PROFILING ──→ DataProfile created
  │
  ▼
RULE_CHECK ──→ RequiredTests + ValidMethods determined
  │
  ▼
TESTING ──→ Run tests (may loop if rule engine adds more)
  │
  ▼
REASONING ──→ LLM generates recommendation
  │
  ▼
VALIDATING ──→ Check recommendation
  │         │
  │     INVALID → back to REASONING (max 3x)
  │
  ▼
REPORTING ──→ Generate final output
  │
  ▼
DONE
```

---

## 6. Data Flow & Pipeline

### Complete Flow: Time Series Example

```
INPUT: GDP quarterly data (1990-2024), 3 variables
       Question: "What drives GDP growth?"

STEP 1 — DATA PROFILER
├── Type: Time Series
├── Frequency: Quarterly
├── Variables: GDP (continuous), Interest Rate (continuous), Gov Spending (continuous)
├── Missing: 0
├── Outliers: 2 (Q2 2008, Q1 2020)
└── N: 140 observations

STEP 2 — RULE ENGINE (Pass 1)
├── Required Tests: [ADF, KPSS, Ljung-Box, Structural Break]
└── Valid Methods: [ARIMA, VAR, VECM, OLS_with_lags, ARDL]

STEP 3 — TEST EXECUTOR (Round 1)
├── ADF (GDP):       stat=-1.82, p=0.34  → FAIL TO REJECT H0 → Non-stationary
├── ADF (Interest):  stat=-2.01, p=0.28  → FAIL TO REJECT H0 → Non-stationary
├── ADF (GovSpend):  stat=-1.56, p=0.51  → FAIL TO REJECT H0 → Non-stationary
├── KPSS (GDP):      stat=0.89, p=0.02   → REJECT H0 → Non-stationary (confirmed)
├── Ljung-Box:       Q=28.4, p=0.001     → Autocorrelation present
└── Chow (2008Q2):   F=4.2, p=0.03       → Structural break detected

STEP 4 — RULE ENGINE (Pass 2)
├── All variables non-stationary → MUST test cointegration
├── Add Required Test: [Johansen Cointegration]
└── Structural break → flag for LLM consideration

STEP 5 — TEST EXECUTOR (Round 2)
├── Johansen Trace:  2 cointegrating relationships (p<0.01)
└── Johansen MaxEig: Confirms rank=2

STEP 6 — RULE ENGINE (Pass 3)
├── Cointegration found → VECM is strongly preferred
└── Updated Valid Methods: [VECM, ARDL] (narrowed down)

STEP 7 — LLM REASONING
├── Recommendation: VECM(2) with 2 cointegrating equations
├── Reasoning: "Non-stationary data with cointegration → VECM preserves
│   long-run equilibrium relationships. ARIMA would lose these. Pure
│   differencing (VAR in differences) would discard valuable information.
│   Structural break in 2008 should be handled with a dummy variable."
├── Confidence: HIGH
├── Robustness: ["Re-estimate excluding 2008-2009", "Try ARDL bounds test"]
└── Interpretation: "Interest rates and government spending have a long-run
    equilibrium relationship with GDP. Short-run deviations correct at a
    rate of X% per quarter."

STEP 8 — VALIDATOR
├── ✓ VECM in ValidMethods
├── ✓ All required tests completed (ADF, KPSS, Ljung-Box, Chow, Johansen)
├── ✓ Non-stationarity claim matches ADF/KPSS results
├── ✓ Cointegration claim matches Johansen results
└── RESULT: PASS

STEP 9 — OUTPUT REPORT
├── Recommended Model: VECM with 2 cointegrating equations
├── Tests Summary Table (all tests + results)
├── Assumption Checks: All passed
├── Confidence: HIGH
├── Audit Trail: Complete decision log
└── Next Steps: Robustness checks suggested
```

---

## 7. Decision Engine Logic

### The Hybrid Approach

The decision engine is the core innovation. It uses a **three-layer sandwich**:

```
┌────────────────────────────────────┐
│     RULE ENGINE (Top Bread)        │  ← What tests MUST run
├────────────────────────────────────┤
│     LLM REASONING (Filling)        │  ← What model is BEST
├────────────────────────────────────┤
│     VALIDATOR (Bottom Bread)       │  ← Is the answer VALID
└────────────────────────────────────┘
```

### Why Not Pure LLM?

| Approach | Completion Rate | Accuracy | Risk |
|----------|----------------|----------|------|
| Pure LLM | <50% | Low | Hallucinations, skipped assumptions |
| Pure Rules | ~70% | Medium | Can't handle ambiguity or edge cases |
| **Hybrid** | **~93%** | **High** | **Low (validated output)** |

### Decision Tree (Simplified)

```
Is data time series?
├── YES → Run ADF + KPSS
│   ├── Stationary → ARIMA / VAR / OLS
│   └── Non-stationary
│       ├── Run cointegration test
│       │   ├── Cointegrated → VECM / ARDL
│       │   └── Not cointegrated → VAR in differences
│       └── Single variable → ARIMA on differenced data
│
├── Is data cross-sectional?
│   ├── Check DV type
│   │   ├── Continuous → OLS (check assumptions)
│   │   ├── Binary → Logit / Probit
│   │   └── Count → Poisson / Negative Binomial
│   ├── Check endogeneity
│   │   ├── Yes → IV / 2SLS / GMM
│   │   └── No → Standard estimation
│   └── Check heteroskedasticity
│       ├── Yes → Robust SE / WLS
│       └── No → Standard SE
│
└── Is data panel?
    ├── Run Hausman test
    │   ├── Reject H0 → Fixed Effects
    │   └── Fail to reject → Random Effects
    ├── Check serial correlation
    │   ├── Yes → Clustered SE
    │   └── No → Standard SE
    └── Check cross-sectional dependence
        ├── Yes → Driscoll-Kraay SE
        └── No → Standard approach
```

---

## 8. Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Language** | Python 3.11+ | Core platform |
| **Data Handling** | pandas, numpy | Data manipulation |
| **Statistical Tests** | statsmodels | ADF, KPSS, OLS, ARIMA, VAR, VECM |
| **Panel/IV Models** | linearmodels | Fixed Effects, Random Effects, IV/2SLS |
| **Volatility** | arch | GARCH models |
| **Distribution Tests** | scipy.stats | Normality, distribution fitting |
| **Type Safety** | Pydantic v2 | Schema validation, strict types |
| **LLM Integration** | anthropic (Python SDK) | Claude API for reasoning |
| **Orchestration** | Custom state machine | Pipeline flow control |
| **Audit Storage** | SQLite | Decision trail logging |
| **Frontend (MVP)** | Streamlit | Upload + results UI |
| **Frontend (v2)** | FastAPI + React | Production web app |
| **Testing** | pytest | Unit + integration tests |

### Dependencies

```
pandas>=2.0
numpy>=1.24
statsmodels>=0.14
linearmodels>=5.0
arch>=6.0
scipy>=1.11
pydantic>=2.0
anthropic>=0.40
streamlit>=1.30
pytest>=7.0
```

---

## 9. Project Structure

```
econometric-ai/
│
├── core/                        # Core engine components
│   ├── __init__.py
│   ├── schemas.py               # Pydantic models (DataProfile, TestResult, etc.)
│   ├── profiler.py              # Data profiling & detection
│   ├── rules.py                 # Rule engine (hard constraints)
│   ├── executor.py              # Statistical test execution
│   ├── reasoner.py              # LLM reasoning layer (Claude API)
│   └── validator.py             # Output validation & guardrails
│
├── tests/                       # Test suite
│   ├── __init__.py
│   ├── test_profiler.py         # Data profiler tests
│   ├── test_rules.py            # Rule engine tests
│   ├── test_executor.py         # Test executor tests
│   ├── test_validator.py        # Validator tests
│   └── test_integration.py      # End-to-end pipeline tests
│
├── data/                        # Sample datasets for testing
│   ├── gdp_quarterly.csv        # Time series example
│   ├── wage_data.csv            # Cross-section example
│   └── state_panel.csv          # Panel data example
│
├── docs/                        # Documentation
│   └── architecture-design.md   # This document
│
├── main.py                      # Pipeline orchestration
├── requirements.txt             # Python dependencies
└── README.md                    # Project overview
```

---

## 10. Use Cases & Examples

### Use Case 1: Time Series — GDP Forecasting

**User Input:** GDP quarterly data, "What drives GDP growth?"
**System Output:** VECM model with cointegration analysis, impulse response functions
**Key Tests:** ADF, KPSS, Johansen, Granger Causality

### Use Case 2: Cross-Section — Wage Determinants

**User Input:** Individual wage data with education, experience, gender
**System Output:** OLS with robust SE (heteroskedasticity detected), Mincer equation
**Key Tests:** VIF, Breusch-Pagan, Ramsey RESET, Jarque-Bera

### Use Case 3: Panel Data — State-Level Policy Effect

**User Input:** 50 states x 20 years, policy dummy variable
**System Output:** Fixed Effects with clustered SE, Difference-in-Differences
**Key Tests:** Hausman, Wooldridge serial correlation, Pesaran CD

### Use Case 4: Causal Inference — Returns to Education

**User Input:** Wage + education data, suspected endogeneity
**System Output:** IV/2SLS using proximity to college as instrument
**Key Tests:** Hausman endogeneity, Sargan overidentification, Weak instrument (F>10)

---

## 11. Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Task completion rate | >90% | % of analyses that produce a valid recommendation |
| Test selection accuracy | >95% | % of required tests correctly identified |
| Model recommendation accuracy | >80% | Compared against expert econometrician choices |
| Assumption violation catch rate | >95% | % of violated assumptions detected |
| User satisfaction | >4/5 | Post-analysis rating |
| Average analysis time | <60 seconds | Upload to report |

---

## 12. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|------------|
| LLM hallucination | Wrong model recommendation | Medium | Validator layer + constrained output |
| Missing edge case in rules | Skipped test | Medium | Comprehensive test suite + user override option |
| Data quality too poor | Unreliable results | High | Profiler flags issues, minimum data requirements |
| API latency (Claude) | Slow analysis | Low | Cache common patterns, batch prompts |
| Borderline statistical results | Ambiguous recommendation | High | Confidence scoring + robustness suggestions |
| User misinterprets output | Wrong conclusions | Medium | Clear disclaimers, educational explanations |

---

## 13. Build Roadmap

### Phase 1: Core Engine (Week 1-2)
- [x] Architecture design (this document)
- [ ] Pydantic schemas
- [ ] Data profiler
- [ ] Rule engine
- [ ] Test executor

### Phase 2: Intelligence Layer (Week 3)
- [ ] LLM reasoning integration (Claude API)
- [ ] Validator
- [ ] Pipeline orchestrator

### Phase 3: Testing & Validation (Week 4)
- [ ] Unit tests for all components
- [ ] Integration tests with sample datasets
- [ ] Benchmark against known correct analyses

### Phase 4: MVP Frontend (Week 5)
- [ ] Streamlit app (upload + results)
- [ ] Report generation (PDF export)

### Phase 5: Production (Week 6+)
- [ ] FastAPI backend
- [ ] React frontend
- [ ] User accounts + history
- [ ] Additional econometric methods

---

*End of Document*

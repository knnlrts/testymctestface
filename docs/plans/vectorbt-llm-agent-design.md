# VectorBT.Pro + LLM Agent System Design

> A comprehensive design for integrating LLMs and coding agents with vectorbt.pro for automated strategy research, backtesting, and iteration.

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Architecture Overview](#architecture-overview)
3. [The Core Agent Loop](#the-core-agent-loop)
4. [Strategy Research Pipeline by Type](#strategy-research-pipeline-by-type)
5. [Key Non-Obvious VBT Features for LLM Agents](#key-non-obvious-vbt-features-for-llm-agents)
6. [Non-Obvious & Creative Use Cases](#non-obvious--creative-use-cases)
7. [Implementation Approach](#implementation-approach)
8. [Risks and Mitigations](#risks-and-mitigations)
9. [Success Metrics](#success-metrics)
10. [Appendix: VBT Feature Reference](#appendix-vbt-feature-reference)

---

## Executive Summary

This document describes a system that combines large language models (LLMs) with vectorbt.pro to create an autonomous strategy research platform. The system enables:

- **Automated strategy research & iteration**: LLM proposes hypotheses, generates backtest code, analyzes results, and refines strategies
- **Coding agent capabilities**: Autonomous writing, debugging, and optimization of vectorbt.pro code
- **Natural language interface**: Define strategies in plain English, receive backtested results
- **Self-documenting research**: Automatic explanation of results, report generation, and knowledge accumulation

### Primary Goals

1. **Automated Strategy Research & Iteration** - LLM proposes → backtests → learns → refines
2. **Autonomous Coding Agent** - Writes and debugs vectorbt.pro code without constant supervision

### Strategy Priorities (in order)

1. Statistical/Quantitative (mean reversion, momentum factors, pairs trading)
2. Machine Learning-driven (features → model → signals)
3. Technical indicator-based (moving averages, RSI, MACD, etc.)
4. Multi-asset / portfolio optimization (allocations, risk parity)
5. High-frequency / microstructure (order flow, market making)
6. Alternative data / sentiment (news, social, on-chain)

### Operational Modes

- **Semi-autonomous**: Human provides high-level direction, agent executes research loops
- **Interactive pair programming**: Real-time collaboration between human and agent
- **Batch research**: Queue up research tasks, agent processes async

### Infrastructure

- **Hybrid compute**: Local for development, cloud (Ray/Dask) for heavy batch jobs
- **Hybrid LLM**: Local models (Ollama) for quick iterations, cloud APIs (Claude/GPT) for complex reasoning

---

## Architecture Overview

The system has three layers working together:

```
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 1: AGENT BACKBONE                      │
│                     (Claude Code / LLM)                         │
│  ┌─────────────────┐              ┌─────────────────┐          │
│  │   Local LLM     │              │    Cloud LLM    │          │
│  │   (Ollama)      │◄────────────►│ (Claude / GPT)  │          │
│  │ Quick iterations│              │ Complex reasoning│          │
│  └─────────────────┘              └─────────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                 LAYER 2: VBT EXECUTION LAYER                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Portfolio   │  │  Indicators  │  │  Parameter   │          │
│  │  Simulation  │  │   Factory    │  │   Sweeps     │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Walk-Fwd   │  │  Ray / Dask  │  │    Data      │          │
│  │  Validation  │  │  (Cloud)     │  │   Pipeline   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              LAYER 3: KNOWLEDGE & FEEDBACK LAYER                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Base.chat() │  │  RAG Pipeline│  │  MCP Server  │          │
│  │  find_docs() │  │  (knowledge) │  │    Tools     │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│  ┌──────────────┐  ┌──────────────┐                            │
│  │   Research   │  │   Doc Store  │                            │
│  │   Memory     │  │    (LMDB)    │                            │
│  └──────────────┘  └──────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
```

### Layer 1 - Agent Backbone (Claude Code / LLM)

This is the reasoning engine. It:
- Receives high-level research directives (e.g., "explore mean reversion on ETH/BTC with lookback windows 12-72 hours")
- Breaks directives into executable steps
- Writes vectorbt.pro code
- Interprets results and decides next actions

**Routing logic:**
- Quick iterations (syntax checks, simple queries) → Local Ollama model
- Complex reasoning (strategy critique, hypothesis generation, debugging) → Claude or GPT

### Layer 2 - VectorBT.Pro Execution Layer

This is where actual computation happens. The agent generates code that uses:

| Component | Purpose |
|-----------|---------|
| `Portfolio.from_signals` / `from_orders` | Core backtesting |
| `vbt.Param()` + `@parameterized` | Parameter sweeps |
| `@cv_split` | Walk-forward validation |
| `IndicatorFactory` | Custom indicators |
| Ray/Dask engines | Distributed execution (cloud burst) |

### Layer 3 - Knowledge & Feedback Layer

Leverages VBT's built-in LLM features:

| Feature | Use Case |
|---------|----------|
| `Base.chat()` | Introspection - ask objects about themselves |
| `Base.find_docs()` | Self-documentation lookup |
| `Base.find_examples()` | Grounded code generation |
| `knowledge` module RAG | Index past research for retrieval |
| MCP server tools | Agent queries API docs and examples |

---

## The Core Agent Loop

The agent operates in a continuous research cycle with five phases:

```
    ┌──────────────────────────────────────────────────────────┐
    │                                                          │
    ▼                                                          │
┌────────────┐    ┌────────────┐    ┌────────────┐            │
│  PHASE 1   │───►│  PHASE 2   │───►│  PHASE 3   │            │
│ Hypothesis │    │    Code    │    │  Execute   │            │
│ Generation │    │  Synthesis │    │  & Sweep   │            │
└────────────┘    └────────────┘    └────────────┘            │
                                          │                    │
                                          ▼                    │
                  ┌────────────┐    ┌────────────┐            │
                  │  PHASE 5   │◄───│  PHASE 4   │            │
                  │  Decision  │    │  Analysis  │            │
                  │ & Iterate  │    │            │            │
                  └────────────┘    └────────────┘            │
                        │                                      │
                        └──────────────────────────────────────┘
```

### Phase 1 - Hypothesis Generation

The agent receives a research directive (from human or from its own analysis of previous results). Using its LLM reasoning, it generates testable hypotheses.

**Example:**
> "Mean reversion works better on BTC/ETH spread during low-volatility regimes. Hypothesis: Adding a volatility filter (ATR < 20-day median) will improve Sharpe by filtering out trending periods."

**Inputs:**
- Human directive OR previous phase results
- Past research from knowledge layer (avoid re-testing)
- Market context if available

**Outputs:**
- Specific, testable hypothesis
- Expected outcome
- Metrics to evaluate

### Phase 2 - Code Synthesis

The agent writes vectorbt.pro code to test the hypothesis.

**Critical behaviors:**
1. **Grounding**: Calls `Base.find_examples()` and `Base.find_api()` to look up correct syntax before writing unfamiliar code
2. **Source inspection**: For unclear APIs, queries MCP server's `get_source()` tool to read actual implementations
3. **Template usage**: Maintains a library of working code patterns to adapt

**Example grounding flow:**
```python
# Agent wants to use from_order_func but is unsure of signature
examples = vbt.Portfolio.find_examples("from_order_func")
# Reviews examples, then writes code matching the pattern
```

### Phase 3 - Execution & Parameter Sweep

**Local execution (quick tests):**
```python
pf = vbt.Portfolio.from_signals(close, entries, exits)
stats = pf.stats()
```

**Distributed execution (exhaustive sweeps):**
```python
@vbt.parameterized(
    merge_func="concat",
    engine="ray",  # or "dask" for cloud
)
def run_backtest(data, lookback, threshold):
    # ... strategy logic ...
    return pf

results = run_backtest(
    data,
    lookback=vbt.Param([12, 24, 48, 72], name="lookback"),
    threshold=vbt.Param(np.arange(1.5, 3.0, 0.25), name="threshold"),
)
```

**Key features leveraged:**
- VBT's chunking prevents redundant computation
- Caching speeds up iterative exploration
- Ray/Dask enable cloud burst for large sweeps

### Phase 4 - Result Analysis

The agent receives backtest results and uses LLM reasoning to interpret patterns.

**Inputs:**
- `Portfolio.stats()` output (Sharpe, drawdown, win rate, etc.)
- Trade logs via `Portfolio.trades`
- Equity curves and drawdown series

**Analysis types:**
- Statistical significance testing (bootstrapped confidence intervals)
- Regime analysis (performance by market condition)
- Comparison to baseline/benchmark
- Identification of failure modes

**Example output:**
> "The volatility filter improved Sharpe from 0.8 to 1.2, but reduced trade count from 340 to 89 annually. The improvement is statistically significant (p < 0.05 on bootstrapped samples). However, the reduced sample size means we need more out-of-sample validation."

### Phase 5 - Decision & Iteration

Based on analysis, the agent decides next action:

| Condition | Action |
|-----------|--------|
| Promising results, needs refinement | Refine parameters, narrow search |
| Clear improvement, statistically significant | Flag for human review |
| Marginal or inconclusive | Test variation or abandon |
| Unexpected behavior | Debug, investigate edge cases |
| Strong results | Run additional walk-forward validation |

**Human escalation triggers:**
- Out-of-sample Sharpe > threshold (e.g., 1.5)
- Novel pattern discovered
- Ambiguous results requiring judgment
- Resource-intensive next step (large cloud job)

---

## Strategy Research Pipeline by Type

### 1. Statistical/Quantitative (Top Priority)

These strategies are hypothesis-driven, making them ideal for LLM-assisted iteration.

**Example flow: Mean Reversion Pairs Strategy**

```python
# Agent detects mean reversion opportunity
# Step 1: Code z-score based pairs strategy
spread = (eth_close / btc_close).vbt.zscore(window=lookback)
entries = spread < -entry_threshold
exits = spread > exit_threshold

# Step 2: Sweep entry/exit thresholds
pf = vbt.Portfolio.from_signals(
    eth_close,
    entries=entries,
    exits=exits,
)

# Step 3: Walk-forward validation
@vbt.cv_split(
    splitter=vbt.Splitter.from_purged_kfold(index, n_folds=5),
    parameterized_kwargs=dict(
        lookback=vbt.Param([12, 24, 48]),
        entry_threshold=vbt.Param([1.5, 2.0, 2.5]),
    ),
    selection="max_sharpe",
)
def test_strategy(data, lookback, entry_threshold):
    # ... strategy implementation ...
    return portfolio

# Step 4: Analyze regime dependence
results_by_regime = pf.stats(group_by=volatility_regime)
```

**Key VBT features:**
- `Splitter.from_purged_kfold` - proper train/test separation for time series
- `@cv_split` - automated walk-forward optimization
- Broadcasting - multi-parameter sweeps with clean syntax

### 2. Machine Learning-Driven

The agent generates features using VBT's indicator pipeline, trains models externally, and brings predictions back as signals.

**Flow:**
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Feature   │───►│     ML      │───►│  Generate   │───►│  Backtest   │
│ Engineering │    │  Training   │    │   Signals   │    │   in VBT    │
│   (VBT)     │    │  (sklearn)  │    │             │    │             │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

**Feature engineering in VBT:**
```python
# Rolling statistics
features = pd.DataFrame({
    'returns_20d': close.vbt.returns().rolling(20).mean(),
    'volatility_20d': close.vbt.returns().rolling(20).std(),
    'rsi_14': vbt.RSI.run(close, window=14).rsi,
    'macd_hist': vbt.MACD.run(close).hist,
    'volume_zscore': volume.vbt.zscore(window=20),
})
```

**Key insight:** VBT's `@parameterized` can sweep ML hyperparameters too:
```python
@vbt.parameterized
def train_and_backtest(data, n_estimators, max_depth, lookback):
    # Train model with these hyperparams
    model = XGBClassifier(n_estimators=n_estimators, max_depth=max_depth)
    # Generate predictions
    # Backtest in VBT
    return portfolio

results = train_and_backtest(
    data,
    n_estimators=vbt.Param([50, 100, 200]),
    max_depth=vbt.Param([3, 5, 7]),
    lookback=vbt.Param([20, 40, 60]),
)
```

### 3. Technical Indicator-Based

Simplest path - leverage VBT's extensive indicator library.

**Available wrappers:**
- `vbt.talib()` - TA-Lib (200+ indicators)
- `vbt.pandas_ta()` - pandas-ta library
- `vbt.IndicatorFactory` - custom indicators

**Example: Multi-indicator combination**
```python
# Agent tests: RSI oversold + MACD crossover + volume confirmation
rsi = vbt.RSI.run(close, window=vbt.Param([7, 14, 21])).rsi
macd = vbt.MACD.run(close, fast_window=vbt.Param([8, 12, 16]))
vol_sma = volume.vbt.ma(window=20)

entries = (rsi < 30) & (macd.hist > macd.hist.shift(1)) & (volume > vol_sma)
```

VBT's broadcasting handles the parameter explosion elegantly - all combinations are tested in a single vectorized operation.

### 4. Portfolio Optimization

VBT's `PortfolioOptimizer` and PyPortfolioOpt integration handle allocation strategies.

**Available methods:**
- Mean-variance optimization
- Risk parity
- Hierarchical Risk Parity (HRP)
- Black-Litterman
- Custom objective functions

**Powerful combination - allocation + timing:**
```python
# Optimize allocation across assets, but only when momentum is positive
momentum_signal = returns.rolling(60).mean() > 0

# Get optimized weights
optimizer = vbt.PortfolioOptimizer.from_pypfopt(
    returns,
    target="max_sharpe",
)
weights = optimizer.weights

# Apply weights only when momentum is positive
conditional_weights = weights.where(momentum_signal, 0)
pf = vbt.Portfolio.from_orders(close, size=conditional_weights)
```

### 5. High-Frequency / Microstructure

Requires tick data and careful handling of execution assumptions.

**Key considerations:**
- Use `Portfolio.from_order_func` for custom execution logic
- Model slippage and market impact explicitly
- VBT's Numba-compiled functions handle the performance requirements

### 6. Alternative Data / Sentiment

Integrate external signals with VBT's data pipeline.

```python
# Fetch price data
price_data = vbt.YFData.fetch("BTC-USD")

# Merge with sentiment (external source)
combined = price_data.data.join(sentiment_scores)

# Use sentiment as signal modifier
base_signal = rsi < 30
sentiment_filter = sentiment_scores > 0.6
entries = base_signal & sentiment_filter
```

---

## Key Non-Obvious VBT Features for LLM Agents

These features make vectorbt.pro uniquely suited for agent-driven research.

### 1. `Base.chat()` on Any Object

Every VBT object inherits this method. The agent can introspect any object by chatting with it.

```python
# After running a backtest
portfolio.chat("Why did this strategy underperform in March 2024?")

# On indicators
rsi_indicator.chat("What parameters would reduce whipsaws?")

# On data
data.chat("Are there any gaps or anomalies in this dataset?")
```

**Agent use case:** Run a backtest, call `.chat()` on the result to reason about it, then iterate. This is self-debugging built into the framework.

### 2. `Base.find_docs()` / `Base.find_examples()`

The agent never needs to hallucinate API usage.

```python
# Before writing code involving an unfamiliar method
examples = vbt.Portfolio.find_examples("from_order_func")
docs = vbt.Portfolio.find_docs("from_order_func")

# Agent reviews these, then writes code matching the patterns
```

**Why this matters:** Grounds code generation in actual documentation, dramatically reducing errors and hallucinations.

### 3. MCP Server with Jupyter Kernel

Running `python -m vectorbtpro.mcp_server` exposes tools for any MCP-compatible agent:

| Tool | Purpose |
|------|---------|
| `find(refnames, ...)` | Semantic search across docs |
| `get_source(refname)` | Read actual implementations |
| `get_attrs(refname)` | List object attributes |
| `get_page(url)` | Fetch documentation pages |
| Jupyter kernel | Execute code, see outputs, iterate |

**Key capability:** The agent can execute code, see outputs, and iterate without leaving the conversation.

### 4. `@cv_split` Decorator

Combines cross-validation with parameterized sweeps in one decorator.

```python
@vbt.cv_split(
    splitter=vbt.Splitter.from_purged_kfold(index, n_folds=5),
    parameterized_kwargs=dict(
        param1=vbt.Param([1, 2, 3]),
        param2=vbt.Param([10, 20, 30]),
    ),
    selection="max_sharpe",  # or custom selection function
)
def backtest_strategy(data, param1, param2):
    # Strategy implementation
    return portfolio
```

**What it does automatically:**
1. Parameter grid search on training folds
2. Selection of best parameters per fold
3. Application to test folds
4. Aggregation of out-of-sample results

The agent just defines the search space - VBT handles the complexity.

### 5. Caching and Chunking

VBT caches intermediate results aggressively.

**Benefits for agent workflows:**
- When iterating on parameters, unchanged computations are skipped
- Exploratory loops stay fast even with large datasets
- Agent can "try things" without worrying about redundant computation

```python
# First run - full computation
pf1 = vbt.Portfolio.from_signals(close, entries1, exits1)

# Second run with same entries - cached
pf2 = vbt.Portfolio.from_signals(close, entries1, exits2)  # Faster
```

### 6. Flexible Broadcasting with `vbt.Param()`

Multiple parameters broadcast together automatically:

```python
# This creates a 3x3x3 = 27 combination grid
results = strategy(
    data,
    lookback=vbt.Param([10, 20, 30], name="lookback"),
    entry=vbt.Param([1.5, 2.0, 2.5], name="entry"),
    exit=vbt.Param([0.5, 1.0, 1.5], name="exit"),
)

# Results are a multi-indexed DataFrame - easy to analyze
results.groupby("lookback").mean()
```

### 7. Built-in LLM Provider Support

VBT's settings support multiple providers out of the box:

```python
# In VBT settings
completions_configs = {
    'openai': {'model': 'gpt-5', 'quick_model': 'gpt-5-mini'},
    'anthropic': {'model': 'claude-sonnet-4-0', 'quick_model': 'claude-3-5-haiku'},
    'ollama': {'model': 'qwen3:8b', 'quick_model': 'qwen3:0.6b'},
}

# Route between them based on task complexity
vbt.settings.chat.completions = 'ollama'  # For quick tasks
vbt.settings.chat.completions = 'anthropic'  # For complex reasoning
```

---

## Non-Obvious & Creative Use Cases

Beyond "LLM writes backtest code," these applications emerge from deep integration:

### 1. Adversarial Strategy Stress-Testing

The agent doesn't just test your strategy - it actively tries to break it.

**How it works:**
1. Agent generates adversarial market conditions using VBT's synthetic data:
   ```python
   # Generate stress scenarios
   stress_data = vbt.Data.from_synthetic(
       generator="gbm",
       n_symbols=1,
       periods=252,
       drift=-0.5,  # Strong downtrend
       volatility=0.8,  # High volatility
   )
   ```
2. Identifies regime changes where strategy fails
3. Proposes hedges or circuit breakers

**Example output:**
> "Your momentum strategy fails when volatility spikes 3x within 2 days. Historical occurrences: 5 times in backtest period. Here's a VIX-based circuit breaker that would have prevented 4 of 5 major drawdowns. Implementing it reduces max drawdown from 34% to 18% with only 0.1 Sharpe degradation."

### 2. Research Memory & Knowledge Compounding

Using VBT's `knowledge` module, the agent indexes every research session.

**Indexed content:**
- Hypotheses tested (successful and failed)
- Code that was generated
- Results and statistical analyses
- Conclusions and next steps

**Query before new research:**
```python
# Agent checks before starting
past_research = knowledge.query(
    "mean reversion ETH BTC spread volatility filter"
)
# Returns: "Tested on 2024-03-15. Volatility filter improved Sharpe
# but reduced trade count. Recommended: test with adaptive threshold."
```

**Benefits:**
- Prevents re-running failed experiments
- Builds on previous insights
- Creates institutional memory that compounds over months
- Agent becomes expert on *your specific* research history

### 3. Natural Language Strategy Diffs

When you modify a strategy, the agent explains changes in plain English.

**Example:**
```python
# Original strategy
pf_v1 = backtest(strategy_v1)

# Modified strategy
pf_v2 = backtest(strategy_v2)

# Agent compares
diff_explanation = agent.explain_diff(pf_v1, pf_v2)
```

**Output:**
> "Adding the volume filter changed performance as follows:
> - Win rate: 52% → 61% (+9 percentage points)
> - Annual trades: 340 → 89 (-74%)
> - Sharpe ratio: 1.1 → 1.4 (+0.3)
> - Max drawdown: 23% → 18% (-5 percentage points)
>
> Net assessment: The filter significantly improves quality of trades but dramatically reduces quantity. Statistical significance is marginal (p=0.08) due to smaller sample size. Recommend: extend backtest period or test on additional assets before concluding."

### 4. Automated Literature-to-Code Pipeline

Feed the agent an academic paper or blog post, get a tested implementation.

**Flow:**
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Paper /   │───►│   Extract   │───►│  Generate   │───►│  Compare    │
│   Blog Post │    │   Logic     │    │ VBT Code    │    │  to Claims  │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

**Agent tasks:**
1. Parse the strategy description
2. Identify key parameters and logic
3. Translate to VBT code
4. Run backtest on same/similar data
5. Report: "Paper claims Sharpe of 1.8. Our replication shows 1.2. Discrepancy likely due to [analysis]."

**Value:** Turns "interesting paper" into "tested hypothesis" in minutes instead of hours.

---

## Implementation Approach

### Phase 1 - Foundation (Week 1-2)

**Objective:** Establish basic agent-VBT communication.

**Tasks:**

1. **MCP Server Setup**
   ```bash
   # Start the VBT MCP server
   python -m vectorbtpro.mcp_server
   ```

2. **Agent Connection**
   - Configure Claude Code (or preferred agent) to connect to MCP server
   - Verify agent can:
     - Query docs via `find()`
     - Read source via `get_source()`
     - Execute code via Jupyter kernel

3. **Local LLM Setup**
   ```bash
   # Install Ollama and pull coding model
   ollama pull qwen2.5-coder:7b
   # Or
   ollama pull deepseek-coder:6.7b
   ```

4. **Configure VBT LLM Settings**
   ```python
   import vectorbtpro as vbt

   vbt.settings.chat.completions = 'auto'  # Auto-detect available
   vbt.settings.chat.completions_configs.ollama.model = 'qwen2.5-coder:7b'
   vbt.settings.chat.completions_configs.ollama.quick_model = 'qwen2.5-coder:1.5b'
   ```

**Deliverable:** Agent can query VBT docs, execute simple backtests, return results.

### Phase 2 - Research Loop (Week 2-3)

**Objective:** Build the core hypothesis→code→execute→analyze loop.

**Tasks:**

1. **Simple Strategy Generation**
   - Agent takes plain-English strategy description
   - Generates VBT code
   - Runs backtest
   - Returns stats

2. **Self-Grounding Behavior**
   ```python
   # Agent workflow before writing unfamiliar code
   def write_code_with_grounding(task):
       # Step 1: Find relevant examples
       examples = vbt.Portfolio.find_examples(relevant_method)

       # Step 2: Generate code based on examples
       code = llm.generate(task, context=examples)

       # Step 3: Execute with error handling
       try:
           result = execute(code)
       except Exception as e:
           # Self-correct based on error
           code = llm.fix_code(code, error=e)
           result = execute(code)

       return result
   ```

3. **Parameter Sweeps**
   - Add `vbt.Param()` usage
   - Agent learns to suggest parameter ranges

4. **Test with Statistical Strategies**
   - Focus on mean reversion, momentum
   - These are highest priority and good test cases

**Deliverable:** Agent can take "test mean reversion on BTC with lookbacks 12-48h" and return analyzed results.

### Phase 3 - Knowledge Layer (Week 3-4)

**Objective:** Enable research memory and knowledge compounding.

**Tasks:**

1. **Initialize Knowledge Module**
   ```python
   from vectorbtpro.knowledge import DocStore, DocRanker

   # Configure persistent storage
   doc_store = DocStore.from_lmdb(
       path="./research_knowledge",
   )
   ```

2. **Index Research Sessions**
   - After each session, agent creates research record:
     ```python
     record = {
         "timestamp": datetime.now(),
         "hypothesis": "...",
         "code": "...",
         "results": {...},
         "conclusions": "...",
         "next_steps": "...",
     }
     doc_store.add(record)
     ```

3. **Query Before New Research**
   - Agent checks: "Have I tested this before?"
   - Retrieves relevant past research
   - Builds on previous insights

4. **Embeddings Configuration**
   ```python
   vbt.settings.chat.embeddings = 'ollama'  # Local embeddings
   vbt.settings.chat.embeddings_configs.ollama.model = 'nomic-embed-text'
   ```

**Deliverable:** Agent remembers past research, avoids redundant tests, compounds knowledge.

### Phase 4 - Scaling & Modes (Week 4+)

**Objective:** Production-ready system with multiple interaction modes.

**Tasks:**

1. **Ray/Dask Configuration**
   ```python
   # Configure for cloud burst
   vbt.settings.execution.engine = 'ray'
   vbt.settings.execution.engine_configs.ray.address = 'ray://cluster:10001'
   ```

2. **Interaction Modes**

   **Interactive Mode:**
   ```python
   # Real-time collaboration
   agent.mode = "interactive"
   agent.run("Let's explore momentum strategies on tech stocks")
   # Agent proposes, human approves/modifies each step
   ```

   **Semi-Autonomous Mode:**
   ```python
   # High-level directive, periodic check-ins
   agent.mode = "semi_autonomous"
   agent.run(
       directive="Explore mean reversion opportunities in crypto",
       check_in_interval="1h",
       escalate_threshold={"sharpe": 1.5},
   )
   ```

   **Batch Mode:**
   ```python
   # Queue research tasks
   agent.mode = "batch"
   agent.queue([
       "Test RSI variations on BTC",
       "Compare momentum factors on ETH",
       "Pairs trading scan on top 20 altcoins",
   ])
   agent.run_overnight()
   ```

3. **Resource Management**
   - Set cloud compute limits
   - Require approval for large jobs
   - Log all executions

**Deliverable:** Full system supporting all three operational modes.

---

## Risks and Mitigations

### Risk 1 - Overfitting Amplification

**Problem:** An autonomous agent can run thousands of parameter combinations, dramatically increasing the chance of finding spuriously good results.

**Mitigations:**
- **Enforce walk-forward validation**: Every strategy must pass `@cv_split` before being flagged as "promising"
- **Report out-of-sample metrics only**: Never show in-sample Sharpe as the headline number
- **Consistency requirement**: Strategy must show positive performance across at least 3 independent test folds
- **Multiple testing correction**: When running many tests, apply Bonferroni or FDR correction

```python
# Agent rule: Always validate before reporting
@vbt.cv_split(
    splitter=vbt.Splitter.from_purged_kfold(index, n_folds=5),
    selection="max_sharpe",
)
def validated_backtest(data, **params):
    # ... strategy ...
    return portfolio

# Only report if out-of-sample Sharpe > threshold
if results.out_of_sample_sharpe > MIN_THRESHOLD:
    flag_for_review(results)
```

### Risk 2 - LLM Hallucination in Code

**Problem:** Even with grounding via `find_examples()`, LLMs sometimes invent plausible-but-wrong API calls.

**Mitigations:**
- **Always execute in try/except**: Capture and parse errors
- **Self-correction loop**: Budget 2-3 retry attempts before escalating
- **Syntax validation**: Check code structure before execution
- **Test on small data first**: Validate logic before full backtest

```python
def safe_execute(code, max_retries=3):
    for attempt in range(max_retries):
        try:
            result = exec_in_kernel(code)
            return result
        except Exception as e:
            if attempt < max_retries - 1:
                code = llm.fix_code(code, error=str(e))
            else:
                escalate_to_human(code, e)
```

### Risk 3 - Runaway Compute Costs

**Problem:** Semi-autonomous mode with cloud bursting could accidentally spin up expensive jobs.

**Mitigations:**
- **Hard limits in cluster config**: Set max workers, max runtime
- **Pre-execution estimation**: Agent estimates grid size before running
- **Approval thresholds**: Require human approval above N combinations
- **Logging and alerts**: Track all cloud executions, alert on anomalies

```python
def run_with_limits(func, params):
    grid_size = estimate_grid_size(params)

    if grid_size > 10000:
        approval = request_human_approval(
            f"Large grid: {grid_size} combinations. Estimated cost: ${estimate_cost(grid_size)}"
        )
        if not approval:
            return None

    return func(**params)
```

### Risk 4 - Research Tunnel Vision

**Problem:** Agent might over-explore one hypothesis family while ignoring others.

**Mitigations:**
- **Periodic randomness injection**: "Explore something unrelated to recent research"
- **Coverage tracking**: Knowledge layer tracks which strategy types have been explored
- **Diversity targets**: Agent must touch each priority area within N sessions
- **Exploration vs exploitation balance**: Allocate fixed percentage to novel directions

```python
# Agent tracks coverage
coverage = knowledge.get_coverage_stats()
# {
#   "statistical_quant": 45,  # sessions
#   "ml_driven": 12,
#   "technical": 8,
#   "portfolio_opt": 3,
#   ...
# }

# If an area is neglected, prioritize it
if coverage["portfolio_opt"] < MIN_COVERAGE:
    next_directive = "Explore portfolio optimization strategies"
```

---

## Success Metrics

### Efficiency Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Idea-to-result time | < 10 minutes | Timestamp from directive to stats output |
| Hypotheses per week | 50+ (vs ~5 manual) | Count in knowledge layer |
| Code success rate | > 80% first-run | Track execution errors |
| Re-research rate | < 5% | Knowledge layer duplicate detection |

### Quality Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Out-of-sample Sharpe | Track distribution | `@cv_split` test fold results |
| Walk-forward survival rate | > 20% of candidates | Strategies passing validation |
| Strategy diversity | All 6 types explored | Knowledge layer coverage |
| Surprise rate | > 10% of discoveries | Human assessment: "Would I have tested this?" |

### Operational Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Self-correction rate | > 70% of errors auto-fixed | Error logs |
| Cloud spend efficiency | < $X per actionable insight | Cost tracking / flagged strategies |
| Human review queue | 5-15 items/day | Queue size monitoring |
| False positive rate | < 30% of flagged strategies | Human assessment of flagged items |

### The North Star

**Ultimate success criterion:** *Are you deploying strategies you wouldn't have found otherwise?*

The system succeeds if it:
1. Expands your research frontier (novel ideas)
2. Accelerates existing workflows (efficiency)
3. Improves strategy quality (better risk-adjusted returns)
4. Compounds knowledge over time (institutional memory)

Track quarterly: "Of strategies deployed to paper/live trading, what percentage originated from agent research?"

---

## Appendix: VBT Feature Reference

### Core Classes

| Class | Purpose | Key Methods |
|-------|---------|-------------|
| `Portfolio` | Backtest simulation | `from_signals()`, `from_orders()`, `from_order_func()`, `stats()`, `trades` |
| `Data` | Data fetching | `fetch()`, `update()`, `resample()` |
| `IndicatorFactory` | Custom indicators | `from_talib()`, `from_pandas_ta()`, `from_custom_func()` |
| `PortfolioOptimizer` | Allocation optimization | `from_pypfopt()`, `optimize()` |
| `Splitter` | Cross-validation | `from_purged_kfold()`, `from_rolling()`, `apply()` |

### Key Decorators

| Decorator | Purpose |
|-----------|---------|
| `@vbt.parameterized` | Parameter grid sweeps |
| `@vbt.cv_split` | Cross-validation + parameter optimization |
| `@vbt.cached` | Result caching |

### LLM Integration

| Feature | Access |
|---------|--------|
| Chat with any object | `obj.chat("question")` |
| Find documentation | `Class.find_docs("method")` |
| Find examples | `Class.find_examples("method")` |
| Find API reference | `Class.find_api("method")` |

### MCP Server Tools

| Tool | Purpose |
|------|---------|
| `find(refnames, targets, ...)` | Semantic search |
| `get_source(refname)` | Read source code |
| `get_attrs(refname)` | List attributes |
| `get_page(url)` | Fetch doc page |
| `resolve_refnames(refnames)` | Validate references |

### Execution Engines

| Engine | Use Case |
|--------|----------|
| `sequential` | Default, single-threaded |
| `pathos` | Local multiprocessing |
| `ray` | Distributed (local or cloud) |
| `dask` | Distributed (cloud-native) |

---

*Document version: 1.0*
*Last updated: 2026-01-05*

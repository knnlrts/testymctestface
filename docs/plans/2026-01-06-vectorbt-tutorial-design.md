# VectorBT.Pro Tutorial: From Hyperliquid Trader Analysis to Agent-Ready Systems

> A hands-on tutorial series teaching VectorBT.Pro through practical Hyperliquid top trader analysis, with deep focus on LLM integration and building toward agentic systems.

---

## Overview

### Purpose

This tutorial series guides intermediate Python users through learning VectorBT.Pro by building a complete Hyperliquid top trader analysis system. The journey progresses from manual exploration to agent-ready code that can be orchestrated by LLM agents.

### Target Audience

**Intermediate developers** who:
- Are comfortable with Python and pandas
- Have some backtesting experience
- Are new to VectorBT.Pro and blockchain data analysis
- Want to leverage LLM capabilities in quantitative workflows

### Learning Philosophy

- **Layered approach**: Start with libraries for quick wins, then reveal underlying mechanics
- **Progressive complexity**: Each part builds on the previous while delivering standalone value
- **Hybrid environment**: Jupyter notebooks for learning, Python modules for production
- **Agent-first mindset**: Code structured for eventual LLM orchestration

---

## Series Structure

| Part | Title | Focus | Deliverable |
|------|-------|-------|-------------|
| 1 | Environment Setup & Foundations | Installation, LLM config, first backtest | Working env with LLM introspection |
| 2 | Hyperliquid Data Pipeline | Data fetching, model understanding | Reusable data utilities |
| 3 | Trader Behavior Analysis | Multi-dimensional scoring system | Ranked trader profiles |
| 4 | Signal Generation & Backtesting | Strategy development, validation | Validated copy-trading strategies |
| 5 | Agent-Ready Foundation | Modular code, MCP, skill extraction | Production-ready agent primitives |

---

## Part 1: Environment Setup & VectorBT.Pro Foundations

### Objectives

- Working VectorBT.Pro installation with LLM support
- Configured dual LLM setup (local Ollama + cloud APIs)
- Complete first backtest and use LLM introspection features

### Content Breakdown

#### 1.1 Installation & Dependencies

**Core packages:**
```
vectorbtpro
hyperliquid-python-sdk
ollama
pandas
numpy
jupyter
```

**Setup steps:**
- Python 3.10+ environment (venv or conda recommended)
- VectorBT.Pro license activation
- Ollama installation with recommended models:
  - `qwen2.5-coder:7b` - primary coding model
  - `qwen2.5-coder:1.5b` - quick model for simple tasks
  - `nomic-embed-text` - embeddings for knowledge/RAG

#### 1.2 LLM Configuration

**Dual provider setup:**
- Configure `vbt.settings.chat` for both local and cloud providers
- Secure API key management via environment variables
- Test both completion paths work correctly

**Routing logic:**
- Local (Ollama): Quick iterations, syntax checks, simple queries
- Cloud (Claude/OpenAI): Complex reasoning, strategy analysis, debugging

```python
import vectorbtpro as vbt

# Configure local provider
vbt.settings.chat.completions_configs.ollama.model = 'qwen2.5-coder:7b'
vbt.settings.chat.completions_configs.ollama.quick_model = 'qwen2.5-coder:1.5b'

# Configure cloud provider (keys from environment)
vbt.settings.chat.completions_configs.anthropic.model = 'claude-sonnet-4-0'

# Set default (can switch dynamically)
vbt.settings.chat.completions = 'ollama'
```

#### 1.3 First Backtest (Warm-up)

Simple moving average crossover on BTC to verify setup:
- Fetch data via `vbt.YFData` (familiar source before blockchain data)
- Introduce `Portfolio.from_signals()` and `.stats()`
- Visualize equity curve and drawdowns

```python
import vectorbtpro as vbt

# Fetch data
btc = vbt.YFData.fetch("BTC-USD", period="1y")
close = btc.get("Close")

# Simple MA crossover signals
fast_ma = close.vbt.ma(window=10)
slow_ma = close.vbt.ma(window=30)
entries = fast_ma > slow_ma
exits = fast_ma < slow_ma

# Backtest
pf = vbt.Portfolio.from_signals(close, entries=entries, exits=exits)
print(pf.stats())
```

#### 1.4 LLM Introspection Introduction

**Three core methods that change everything:**

```python
# 1. Chat with any object - ask your results questions
pf.chat("What caused the largest drawdown?")
pf.chat("How does win rate compare to a random strategy?")

# 2. Find documentation - self-guided API discovery
vbt.Portfolio.find_docs("from_signals")
vbt.Portfolio.find_docs("stats")

# 3. Find examples - grounded code generation
vbt.Portfolio.find_examples("from_order_func")
vbt.Portfolio.find_examples("cv_split")
```

**Framing:** These methods are your AI pair programmer that knows VectorBT inside out. Use them liberally throughout learning.

### Deliverable

User has a working environment, has run their first backtest, and has used LLM features to explore and understand results.

---

## Part 2: Hyperliquid Data Pipeline

### Objectives

- Fetch top trader addresses and their trading history
- Understand Hyperliquid's data model
- Build reusable data fetching utilities integrated with VectorBT

### Content Breakdown

#### 2.1 Hyperliquid SDK Quick Start

**First connection:**
```python
from hyperliquid.info import Info

info = Info(skip_ws=True)

# Fetch leaderboard
leaderboard = info.leaderboard("1d")  # timeframe options: 1d, 7d, 30d

# Get top traders by PnL
top_traders = sorted(leaderboard, key=lambda x: x['pnl'], reverse=True)[:20]
```

**Pull single trader's history:**
```python
# Fetch a trader's position history
trader_address = top_traders[0]['address']
fills = info.user_fills(trader_address)
positions = info.user_state(trader_address)
```

#### 2.2 Data Model Deep Dive

**Positions:**
- Open interest, entry price, leverage, margin
- Unrealized PnL, liquidation price
- Position side (long/short)

**Fills (Trade Executions):**
- Timestamp, price, size, side
- Fee paid, order type
- Closed PnL for closing trades

**Funding:**
- Payments received/paid on perpetual positions
- Accrues every hour based on funding rate
- Can significantly impact PnL

**Liquidations:**
- Forced position closures
- Critical for risk analysis
- Indicates over-leveraging

#### 2.3 Under the Hood - Direct API Access

**When to go direct:**
- Custom queries not supported by SDK
- Real-time streaming via WebSocket
- Fine-grained rate limit control

```python
import requests

# Direct REST API example
BASE_URL = "https://api.hyperliquid.xyz/info"

def fetch_user_fills_direct(address: str, limit: int = 1000):
    """Fetch fills directly from API."""
    payload = {
        "type": "userFills",
        "user": address,
        "limit": limit
    }
    response = requests.post(BASE_URL, json=payload)
    return response.json()
```

#### 2.4 Building the Data Layer

**Core utilities to build:**

```python
# data/fetchers.py

def fetch_top_traders(n: int = 50, timeframe: str = '7d') -> pd.DataFrame:
    """Fetch top N traders by PnL for given timeframe."""
    ...

def fetch_trader_history(address: str, days: int = 30) -> dict:
    """Fetch complete trading history for an address."""
    return {
        'fills': ...,
        'funding': ...,
        'positions': ...,
    }

def fetch_market_data(symbol: str, resolution: str = '1h') -> pd.DataFrame:
    """Fetch OHLCV data for a symbol."""
    ...
```

**Production considerations:**
- Caching strategy to avoid re-fetching (local file or Redis)
- Rate limiting with exponential backoff
- Data normalization into consistent DataFrame schemas
- Error handling for API failures

#### 2.5 First Integration with VectorBT

```python
import vectorbtpro as vbt

# Load Hyperliquid price data into VectorBT wrapper
hl_data = fetch_market_data("BTC", resolution="1h")
data = vbt.Data.from_data(hl_data)

# Use VectorBT's data utilities
resampled = data.resample("4h")
cleaned = data.dropna()

# LLM-assisted data QA
data.chat("Are there any gaps or anomalies in this dataset?")
data.chat("What's the average daily volume?")
```

### Deliverable

User has utilities to fetch top traders and their history, understands the Hyperliquid data model, and can load data into VectorBT for analysis.

---

## Part 3: Trader Behavior Analysis

### Objectives

- Build comprehensive scoring system across five dimensions
- Use VectorBT analytics for performance calculation
- Create trader profiles that reveal behavioral patterns

### Scoring Dimensions (Priority Order)

1. **Performance** - Are they profitable?
2. **Risk Management** - Do they protect capital?
3. **Position Sizing** - Do they size appropriately?
4. **Timing** - Do they enter/exit well?
5. **Asset Selection** - Do they pick good markets?

### Content Breakdown

#### 3.1 Performance Metrics (Priority 1)

**Core metrics:**
```python
def calculate_performance_score(fills: pd.DataFrame) -> dict:
    """Calculate performance metrics from trade history."""

    # Build portfolio from fills
    pf = vbt.Portfolio.from_orders(...)
    stats = pf.stats()

    return {
        'total_pnl': stats['Total Return [%]'],
        'sharpe_ratio': stats['Sharpe Ratio'],
        'sortino_ratio': stats['Sortino Ratio'],
        'profit_factor': stats['Profit Factor'],
        'win_rate': stats['Win Rate [%]'],
        'avg_win': stats['Avg Winning Trade [%]'],
        'avg_loss': stats['Avg Losing Trade [%]'],
        'expectancy': stats['Expectancy'],
    }
```

**Benchmark comparison:**
- Trader vs buy-and-hold
- Trader vs market average
- Trader vs top 10% of traders

#### 3.2 Risk Management Scoring (Priority 2)

**Metrics:**
- Max drawdown and drawdown duration
- Recovery patterns (how quickly do they recover?)
- Position correlation (hedged vs concentrated)
- Leverage patterns: average, max, during drawdowns

**Stop-loss behavior detection:**
```python
def detect_stop_loss_behavior(fills: pd.DataFrame) -> dict:
    """Analyze if exits cluster at loss thresholds."""

    losing_trades = fills[fills['pnl'] < 0]
    loss_percentages = losing_trades['pnl_pct']

    # Check for clustering at round numbers (-2%, -5%, -10%)
    clusters = find_clusters(loss_percentages, thresholds=[-0.02, -0.05, -0.10])

    return {
        'uses_stops': len(clusters) > 0,
        'stop_levels': clusters,
        'max_single_loss': loss_percentages.min(),
        'consistency': loss_percentages.std(),  # lower = more consistent
    }
```

**Score:** Consistency of risk-taking vs erratic behavior

#### 3.3 Position Sizing Patterns (Priority 3)

**Analysis dimensions:**
- Average position size relative to account
- Scaling behavior: adding to winners vs averaging down
- Size variation: fixed sizing vs conviction-based
- Kelly criterion comparison

```python
def analyze_position_sizing(fills: pd.DataFrame, account_size: float) -> dict:
    """Analyze position sizing patterns."""

    sizes = fills['size_usd'] / account_size

    return {
        'avg_position_pct': sizes.mean(),
        'max_position_pct': sizes.max(),
        'size_std': sizes.std(),
        'scales_into_winners': detect_scaling_pattern(fills, 'winners'),
        'averages_down': detect_scaling_pattern(fills, 'losers'),
        'kelly_comparison': compare_to_kelly(fills),
    }
```

#### 3.4 Timing Behavior (Priority 4)

**Metrics:**
- Holding period distribution (scalper vs swing vs position)
- Entry timing relative to price moves
- Exit timing: early profit-taking vs letting winners run
- Time-of-day patterns

```python
def analyze_timing(fills: pd.DataFrame, prices: pd.DataFrame) -> dict:
    """Analyze entry/exit timing behavior."""

    holding_periods = calculate_holding_periods(fills)

    return {
        'avg_holding_period': holding_periods.mean(),
        'holding_period_std': holding_periods.std(),
        'trader_type': classify_trader_type(holding_periods),
        'entry_timing_score': score_entry_timing(fills, prices),
        'exit_timing_score': score_exit_timing(fills, prices),
        'hour_of_day_pattern': analyze_time_patterns(fills),
    }
```

#### 3.5 Asset Selection (Priority 5)

**Metrics:**
- Concentration vs diversification
- Asset preferences (which symbols dominate)
- Rotation patterns
- Early adoption of new listings

```python
def analyze_asset_selection(fills: pd.DataFrame) -> dict:
    """Analyze asset selection behavior."""

    symbol_counts = fills.groupby('symbol').size()
    symbol_pnl = fills.groupby('symbol')['pnl'].sum()

    return {
        'num_assets_traded': len(symbol_counts),
        'concentration_hhi': calculate_hhi(symbol_counts),
        'top_assets': symbol_counts.nlargest(5).to_dict(),
        'best_performing_asset': symbol_pnl.idxmax(),
        'worst_performing_asset': symbol_pnl.idxmin(),
        'rotates_assets': detect_rotation_pattern(fills),
    }
```

#### 3.6 Composite Scoring System

```python
from dataclasses import dataclass

@dataclass
class TraderScore:
    performance: float      # 0-1, weighted highest
    risk_management: float  # 0-1
    position_sizing: float  # 0-1
    timing: float          # 0-1
    asset_selection: float # 0-1

    # Weights reflect priority order
    WEIGHTS = {
        'performance': 0.35,
        'risk_management': 0.25,
        'position_sizing': 0.20,
        'timing': 0.12,
        'asset_selection': 0.08,
    }

    @property
    def composite(self) -> float:
        """Weighted composite score."""
        return (
            self.performance * self.WEIGHTS['performance'] +
            self.risk_management * self.WEIGHTS['risk_management'] +
            self.position_sizing * self.WEIGHTS['position_sizing'] +
            self.timing * self.WEIGHTS['timing'] +
            self.asset_selection * self.WEIGHTS['asset_selection']
        )

    def to_dict(self) -> dict:
        return {
            'performance': self.performance,
            'risk_management': self.risk_management,
            'position_sizing': self.position_sizing,
            'timing': self.timing,
            'asset_selection': self.asset_selection,
            'composite': self.composite,
        }
```

**Ranking traders:**
```python
def rank_traders(traders: list[dict]) -> pd.DataFrame:
    """Score and rank all traders."""

    scores = []
    for trader in traders:
        score = calculate_trader_score(trader)
        scores.append({
            'address': trader['address'],
            **score.to_dict()
        })

    df = pd.DataFrame(scores)
    return df.sort_values('composite', ascending=False)
```

#### 3.7 LLM-Assisted Analysis

```python
# Ask trader profile questions
trader_profile.chat("What makes this trader's risk management strong?")
trader_profile.chat("Compare this to the average top 10 trader")

# Portfolio-level insights
portfolio.chat("Why did this trader underperform in March?")
portfolio.chat("What market conditions favor this trading style?")

# Generate natural language summaries
summary = trader_profile.chat(
    "Write a 3-paragraph summary of this trader's style, strengths, and weaknesses"
)
```

### Deliverable

User has a multi-dimensional scoring system, can rank traders by composite and individual dimensions, and can generate insights using LLM introspection.

---

## Part 4: Signal Generation & Backtesting

### Objectives

- Convert trader behavior insights into tradeable signals
- Backtest "follow the leader" strategies with proper validation
- Master VectorBT's parameter sweeps and walk-forward testing

### Content Breakdown

#### 4.1 Signal Design Philosophy

**Three signal approaches:**

1. **Mirror signals**: Copy entries/exits of top traders directly
2. **Consensus signals**: Enter when N of top M traders agree
3. **Filtered signals**: Mirror only when trader's recent performance is strong

**Latency considerations:**
- Blockchain data has inherent delay
- Model realistic execution lag (1-30 minutes)
- Test sensitivity to delay

#### 4.2 Building Signal Generators

```python
# signals/generators.py

def mirror_signals(
    trader_address: str,
    fills: pd.DataFrame,
    delay_minutes: int = 5
) -> pd.DataFrame:
    """Generate signals that mirror a single trader's trades."""

    signals = pd.DataFrame(index=price_index)
    signals['entries'] = False
    signals['exits'] = False

    for _, fill in fills.iterrows():
        signal_time = fill['timestamp'] + timedelta(minutes=delay_minutes)
        if fill['side'] == 'buy':
            signals.loc[signal_time, 'entries'] = True
        else:
            signals.loc[signal_time, 'exits'] = True

    return signals


def consensus_signals(
    top_traders: list[str],
    all_fills: dict[str, pd.DataFrame],
    agreement_threshold: float = 0.6,
    lookback_minutes: int = 30
) -> pd.DataFrame:
    """Generate signals when multiple traders agree."""

    # Count traders with positions at each timestamp
    # Signal when agreement_threshold fraction are positioned same direction
    ...


def filtered_signals(
    traders: list[str],
    all_fills: dict[str, pd.DataFrame],
    min_recent_sharpe: float = 1.0,
    lookback_days: int = 7
) -> pd.DataFrame:
    """Mirror traders only when their recent performance is strong."""

    # Calculate rolling Sharpe for each trader
    # Only follow signals from traders above threshold
    ...
```

#### 4.3 First Backtest - Simple Mirror Strategy

```python
import vectorbtpro as vbt

# Get top trader by composite score
top_trader = ranked_traders.iloc[0]['address']
trader_fills = fetch_trader_history(top_trader)['fills']

# Generate mirror signals with 5-minute delay
signals = mirror_signals(top_trader, trader_fills, delay_minutes=5)

# Fetch corresponding price data
prices = fetch_market_data("BTC", resolution="1m")

# Backtest
pf = vbt.Portfolio.from_signals(
    prices['close'],
    entries=signals['entries'],
    exits=signals['exits'],
    freq='1m'
)

# Analyze
print(pf.stats())
pf.plot().show()

# LLM analysis
pf.chat("How much alpha survives after the 5-minute delay?")
pf.chat("What percentage of the trader's gains did we capture?")
```

#### 4.4 Parameter Sweeps with `vbt.Param()`

```python
@vbt.parameterized(merge_func="concat")
def backtest_mirror_strategy(
    prices: pd.DataFrame,
    trader_fills: pd.DataFrame,
    delay: int,
    position_size: float
) -> vbt.Portfolio:
    """Parameterized backtest for grid search."""

    signals = mirror_signals(trader_fills, delay_minutes=delay)

    pf = vbt.Portfolio.from_signals(
        prices['close'],
        entries=signals['entries'],
        exits=signals['exits'],
        size=position_size,
    )
    return pf


# Run parameter sweep
results = backtest_mirror_strategy(
    prices,
    trader_fills,
    delay=vbt.Param([1, 5, 15, 30], name="delay_min"),
    position_size=vbt.Param([0.1, 0.25, 0.5, 1.0], name="size"),
)

# Analyze parameter sensitivity
results.sharpe_ratio.vbt.heatmap()
results.chat("Which delay values are most robust?")
results.chat("Is there a size that works well across all delays?")
```

#### 4.5 Walk-Forward Validation

**Why in-sample results lie:**
- We selected traders based on past performance (lookahead bias)
- Parameter optimization finds what *worked*, not what *will work*
- More parameters = more overfitting risk

**Proper validation with `@cv_split`:**

```python
@vbt.cv_split(
    splitter=vbt.Splitter.from_purged_kfold(
        price_index,
        n_folds=5,
        purge_period='7d',  # gap between train/test
    ),
    parameterized_kwargs=dict(
        delay=vbt.Param([1, 5, 15, 30]),
        threshold=vbt.Param([0.5, 0.6, 0.7]),
    ),
    selection="max_sharpe",
)
def validated_consensus_strategy(
    prices: pd.DataFrame,
    trader_data: dict,
    delay: int,
    threshold: float
) -> vbt.Portfolio:
    """Walk-forward validated consensus strategy."""

    signals = consensus_signals(
        trader_data,
        agreement_threshold=threshold,
        delay_minutes=delay
    )

    return vbt.Portfolio.from_signals(prices, signals)


# Run walk-forward validation
cv_results = validated_consensus_strategy(prices, trader_data)

# Only trust out-of-sample metrics
print("In-sample Sharpe:", cv_results.in_sample.sharpe_ratio.mean())
print("Out-of-sample Sharpe:", cv_results.out_of_sample.sharpe_ratio.mean())
```

#### 4.6 Strategy Comparison Framework

```python
def compare_strategies(
    strategies: dict[str, vbt.Portfolio]
) -> pd.DataFrame:
    """Compare multiple strategies side-by-side."""

    comparison = []
    for name, pf in strategies.items():
        stats = pf.stats()
        comparison.append({
            'strategy': name,
            'total_return': stats['Total Return [%]'],
            'sharpe': stats['Sharpe Ratio'],
            'sortino': stats['Sortino Ratio'],
            'max_dd': stats['Max Drawdown [%]'],
            'calmar': stats['Calmar Ratio'],
            'win_rate': stats['Win Rate [%]'],
            'num_trades': stats['Total Trades'],
        })

    return pd.DataFrame(comparison)


# Compare our strategies
strategies = {
    'mirror_top1': mirror_pf,
    'mirror_top5': mirror_top5_pf,
    'consensus_60': consensus_pf,
    'filtered': filtered_pf,
    'buy_hold': buy_hold_pf,  # baseline
}

comparison = compare_strategies(strategies)
print(comparison.sort_values('sharpe', ascending=False))

# Statistical significance
comparison_obj.chat(
    "Are the differences between mirror and consensus statistically significant?"
)
```

#### 4.7 LLM-Assisted Strategy Refinement

```python
# Diagnose performance issues
pf.chat("Why did performance degrade in March?")
pf.chat("Which market regime does this strategy perform best in?")

# Learn new VectorBT patterns
vbt.Portfolio.find_examples("cv_split")
vbt.Splitter.find_docs("from_purged_kfold")

# Generate new hypotheses
pf.chat(
    "Based on these results, what modifications might improve risk-adjusted returns?"
)
```

### Deliverable

User has backtested multiple signal strategies, understands parameter optimization pitfalls, and can validate strategies with walk-forward testing.

---

## Part 5: Agent-Ready Foundation

### Objectives

- Refactor notebooks into clean, modular Python code
- Set up MCP server for AI-assisted development
- Build primitives that an LLM agent can orchestrate
- Extract reusable coding agent skills

### Content Breakdown

#### 5.1 Code Architecture for Agents

**Why notebooks don't work for agents:**
- No programmatic invocation
- State management issues
- Hard to test and version

**Target structure:**
```
hyperliquid_vbt/
├── data/
│   ├── __init__.py
│   ├── fetchers.py      # fetch_top_traders(), fetch_history()
│   └── cache.py         # caching layer
├── analysis/
│   ├── __init__.py
│   ├── scoring.py       # TraderScorer class
│   └── profiles.py      # TraderProfile generation
├── signals/
│   ├── __init__.py
│   ├── generators.py    # mirror, consensus, filtered signals
│   └── filters.py       # signal quality filters
├── backtest/
│   ├── __init__.py
│   ├── runners.py       # run_backtest(), run_sweep()
│   └── validation.py    # walk-forward helpers
├── agent/
│   ├── __init__.py
│   ├── primitives.py    # high-level functions agents call
│   └── prompts.py       # prompt templates for agent tasks
└── skills/
    ├── __init__.py
    ├── data_pipeline.py
    ├── scoring_system.py
    ├── signal_extraction.py
    ├── backtest_loop.py
    └── self_grounding.py
```

#### 5.2 Designing Agent-Callable Primitives

**Design principles:**
- Self-contained with clear inputs/outputs
- Well-documented (agents read docstrings)
- Return structured data, not print statements
- Errors should be parseable and actionable

```python
# agent/primitives.py

from dataclasses import dataclass
from typing import Optional
import vectorbtpro as vbt

@dataclass
class TraderProfile:
    address: str
    score: TraderScore
    fills: pd.DataFrame
    stats: dict

    def chat(self, question: str) -> str:
        """Enable LLM introspection on this profile."""
        context = f"Trader {self.address} with score {self.score.composite:.2f}"
        return vbt.chat(question, context=context)


@dataclass
class BacktestResult:
    portfolio: vbt.Portfolio
    params: dict
    metrics: dict

    def chat(self, question: str) -> str:
        """Enable LLM introspection on backtest results."""
        return self.portfolio.chat(question)


def analyze_trader(address: str, days: int = 30) -> TraderProfile:
    """
    Analyze a single trader's behavior and performance.

    Args:
        address: Hyperliquid wallet address
        days: Number of days of history to analyze

    Returns:
        TraderProfile with scores, fills, and stats
    """
    history = fetch_trader_history(address, days=days)
    score = calculate_trader_score(history)
    stats = calculate_performance_metrics(history['fills'])

    return TraderProfile(
        address=address,
        score=score,
        fills=history['fills'],
        stats=stats
    )


def generate_signals(
    strategy_type: str,
    traders: list[str],
    **params
) -> pd.DataFrame:
    """
    Generate trading signals from trader behavior.

    Args:
        strategy_type: One of 'mirror', 'consensus', 'filtered'
        traders: List of trader addresses to analyze
        **params: Strategy-specific parameters

    Returns:
        DataFrame with 'entries' and 'exits' columns
    """
    generators = {
        'mirror': mirror_signals,
        'consensus': consensus_signals,
        'filtered': filtered_signals,
    }

    if strategy_type not in generators:
        raise ValueError(f"Unknown strategy: {strategy_type}. Options: {list(generators.keys())}")

    return generators[strategy_type](traders, **params)


def run_backtest(
    signals: pd.DataFrame,
    prices: pd.DataFrame,
    **config
) -> BacktestResult:
    """
    Run a backtest with given signals.

    Args:
        signals: DataFrame with entries/exits
        prices: OHLCV price data
        **config: Portfolio configuration (size, fees, etc.)

    Returns:
        BacktestResult with portfolio and metrics
    """
    pf = vbt.Portfolio.from_signals(
        prices['close'],
        entries=signals['entries'],
        exits=signals['exits'],
        **config
    )

    return BacktestResult(
        portfolio=pf,
        params=config,
        metrics=pf.stats().to_dict()
    )


def compare_strategies(
    results: dict[str, BacktestResult]
) -> pd.DataFrame:
    """
    Compare multiple backtest results.

    Args:
        results: Dict mapping strategy names to BacktestResult

    Returns:
        DataFrame comparing key metrics across strategies
    """
    ...
```

#### 5.3 MCP Server Setup

**Start VectorBT MCP server:**
```bash
python -m vectorbtpro.mcp_server
```

**Configure Jupyter kernel for code execution:**
```python
# The MCP server includes a Jupyter kernel
# Agent can execute code and see outputs directly
```

**Connect Claude Code to MCP server:**
```json
// In Claude Code MCP config
{
  "vectorbt": {
    "command": "python",
    "args": ["-m", "vectorbtpro.mcp_server"]
  }
}
```

**Test the loop:**
1. Agent queries docs via `find()`
2. Agent writes code based on examples
3. Agent executes via Jupyter kernel
4. Agent sees results and iterates

#### 5.4 Agent Tool Registration

```python
# agent/tools.py

from vectorbtpro.mcp import mcp_tool

@mcp_tool
def analyze_top_traders(
    n: int = 10,
    timeframe: str = "7d"
) -> list[TraderProfile]:
    """
    Fetch and analyze top N Hyperliquid traders.

    Args:
        n: Number of top traders to analyze
        timeframe: Leaderboard timeframe (1d, 7d, 30d)

    Returns:
        List of TraderProfile objects sorted by composite score
    """
    traders = fetch_top_traders(n=n, timeframe=timeframe)
    profiles = [analyze_trader(t['address']) for t in traders]
    return sorted(profiles, key=lambda p: p.score.composite, reverse=True)


@mcp_tool
def backtest_copy_strategy(
    strategy: str,
    trader_addresses: list[str],
    delay_minutes: int = 5,
    validate: bool = True
) -> BacktestResult:
    """
    Backtest a copy-trading strategy.

    Args:
        strategy: Strategy type (mirror, consensus, filtered)
        trader_addresses: Traders to copy
        delay_minutes: Execution delay to model
        validate: Whether to run walk-forward validation

    Returns:
        BacktestResult with metrics and portfolio
    """
    ...


@mcp_tool
def explain_results(
    result: BacktestResult,
    question: str
) -> str:
    """
    Ask a question about backtest results.

    Args:
        result: BacktestResult to analyze
        question: Natural language question

    Returns:
        LLM-generated explanation
    """
    return result.chat(question)
```

#### 5.5 Prompt Engineering for the Domain

**System prompt components:**

```python
# agent/prompts.py

SYSTEM_PROMPT = """
You are a quantitative research assistant specialized in:
1. Hyperliquid perpetual futures trading
2. VectorBT.Pro backtesting framework
3. Copy-trading and smart money analysis

## Available Tools
- analyze_top_traders(n, timeframe): Get scored trader profiles
- generate_signals(strategy_type, traders, **params): Create trading signals
- run_backtest(signals, prices, **config): Backtest signals
- compare_strategies(results): Compare multiple approaches

## Hyperliquid Data Model
- Fills: Individual trade executions with timestamp, price, size, side
- Positions: Open interest, leverage, entry price, liquidation price
- Funding: Hourly payments on perpetual positions

## VectorBT Patterns
- Always use walk-forward validation before trusting results
- Use vbt.Param() for parameter sweeps
- Use @cv_split for proper train/test separation
- Call .chat() on any object to understand it better

## Research Workflow
1. Analyze traders to understand who performs well and why
2. Generate signals based on their behavior
3. Backtest with realistic delays and costs
4. Validate with walk-forward testing
5. Only flag strategies with out-of-sample Sharpe > 1.0

## Guard Rails
- Never report in-sample metrics as final results
- Always account for execution delay (minimum 5 minutes)
- Require statistical significance before claiming an edge
"""

FEW_SHOT_EXAMPLES = [
    {
        "user": "Find a profitable copy-trading strategy",
        "assistant": """I'll research this systematically:

1. First, let me analyze the top traders to understand who's worth copying.
[Calls analyze_top_traders(n=20, timeframe='30d')]

2. The top 3 traders by composite score are:
   - 0x1234: Score 0.82 (strong risk management)
   - 0x5678: Score 0.79 (consistent sizing)
   - 0xabcd: Score 0.75 (good timing)

3. Let me test different copy strategies:
[Calls backtest_copy_strategy for mirror, consensus, filtered]

4. Results comparison:
   - Mirror top 1: Sharpe 0.9 (in-sample), 0.6 (out-of-sample)
   - Consensus 60%: Sharpe 1.1 (in-sample), 0.8 (out-of-sample)
   - Filtered: Sharpe 1.3 (in-sample), 0.95 (out-of-sample)

5. The filtered approach shows the most promise with out-of-sample Sharpe of 0.95.
   However, this is below my 1.0 threshold for high confidence.

Recommendation: The filtered strategy is directionally promising but needs more
validation. Consider extending the backtest period or testing on additional assets."""
    }
]
```

#### 5.6 End-to-End Agent Workflow Demo

**User gives directive:**
> "Find a profitable copy-trading strategy for Hyperliquid"

**Agent autonomously:**
1. Fetches top 20 traders from 30-day leaderboard
2. Analyzes and scores each across 5 dimensions
3. Generates 3 signal variants (mirror, consensus, filtered)
4. Runs parameter sweeps on each
5. Validates with 5-fold walk-forward testing
6. Reports findings with statistical context

**Human reviews:**
- Approves promising directions
- Suggests modifications
- Requests deeper analysis on specific aspects

#### 5.7 Extracting Coding Agent Skills

The workflow teaches transferable patterns. Each becomes a reusable skill:

**Skill 1: Data Pipeline Construction**
```python
# skills/data_pipeline.py

SKILL_NAME = "data_pipeline"
DESCRIPTION = "Build a data pipeline from an external source"

PATTERN = """
1. Start with SDK/library for quick connection
2. Understand the data model (entities, relationships)
3. Build fetcher functions with consistent interfaces
4. Add caching layer to avoid redundant calls
5. Integrate with analysis framework (VectorBT)
"""

PROMPT_TEMPLATE = """
Build a data pipeline for {source} that fetches {entities}.
Follow this pattern:
{PATTERN}
"""

TRANSFERABLE_TO = [
    "Any blockchain (Ethereum, Solana, etc.)",
    "Any exchange API (Binance, Coinbase, etc.)",
    "Any data provider (Glassnode, Dune, etc.)",
]
```

**Skill 2: Multi-Dimensional Scoring Systems**
```python
# skills/scoring_system.py

SKILL_NAME = "scoring_system"
DESCRIPTION = "Create a scoring system for ranking entities"

PATTERN = """
1. Identify dimensions relevant to the ranking goal
2. Define metrics for each dimension
3. Normalize metrics to 0-1 scale
4. Assign weights based on importance
5. Combine into composite score
"""

PROMPT_TEMPLATE = """
Create a scoring system for {entity} across these dimensions: {dimensions}.
The goal is to identify {objective}.
"""

TRANSFERABLE_TO = [
    "Token/protocol scoring",
    "NFT collection ranking",
    "DeFi yield opportunity ranking",
]
```

**Skill 3: Signal-from-Behavior Extraction**
```python
# skills/signal_extraction.py

SKILL_NAME = "signal_extraction"
DESCRIPTION = "Generate trading signals from observed behavior"

PATTERN = """
1. Observe raw behavior data (trades, positions, actions)
2. Identify patterns that precede profitable outcomes
3. Translate patterns into discrete signals (entry/exit)
4. Handle timing: when observed vs when actionable
5. Add filters to improve signal quality
"""

PROMPT_TEMPLATE = """
Generate trading signals from {actor} behavior on {platform}.
Account for {delay} delay between observation and execution.
"""
```

**Skill 4: Backtest-Validate-Iterate Loop**
```python
# skills/backtest_loop.py

SKILL_NAME = "backtest_loop"
DESCRIPTION = "Systematic strategy testing with proper validation"

PATTERN = """
1. Form testable hypothesis
2. Run initial backtest
3. Parameter sweep to understand sensitivity
4. Walk-forward validation for honest metrics
5. Analyze failures and refine
6. Repeat until out-of-sample metrics meet threshold
"""

GUARD_RAILS = [
    "Never trust in-sample metrics",
    "Require statistical significance",
    "Account for realistic trading costs",
]
```

**Skill 5: Self-Grounding Code Generation**
```python
# skills/self_grounding.py

SKILL_NAME = "self_grounding"
DESCRIPTION = "Learn a new library by grounding in its documentation"

PATTERN = """
1. Call find_docs() to understand available methods
2. Call find_examples() to see working code patterns
3. Write code that matches the patterns
4. Execute and observe errors
5. Self-correct based on error messages
6. Iterate until working
"""

PROMPT_TEMPLATE = """
Learn to use {library} for {task}.
Before writing code:
1. Call {library}.find_docs("{method}")
2. Call {library}.find_examples("{method}")
Then write code matching the patterns you found.
"""
```

#### 5.8 Skill Registry & Composition

```python
# skills/__init__.py

from .data_pipeline import DataPipelineSkill
from .scoring_system import ScoringSystemSkill
from .signal_extraction import SignalExtractionSkill
from .backtest_loop import BacktestLoopSkill
from .self_grounding import SelfGroundingSkill

SKILL_REGISTRY = {
    'data_pipeline': DataPipelineSkill,
    'scoring_system': ScoringSystemSkill,
    'signal_extraction': SignalExtractionSkill,
    'backtest_loop': BacktestLoopSkill,
    'self_grounding': SelfGroundingSkill,
}

def get_skill(name: str) -> Skill:
    """Retrieve a skill by name."""
    if name not in SKILL_REGISTRY:
        raise ValueError(f"Unknown skill: {name}. Available: {list(SKILL_REGISTRY.keys())}")
    return SKILL_REGISTRY[name]()

def compose_skills(*skill_names: str) -> ComposedSkill:
    """Chain multiple skills together."""
    skills = [get_skill(name) for name in skill_names]
    return ComposedSkill(skills)
```

**Composition example:**
```python
# Agent chains skills for a new project
workflow = compose_skills(
    'data_pipeline',      # Fetch data from Aave
    'scoring_system',     # Score top depositors
    'signal_extraction',  # Generate signals from their behavior
    'backtest_loop',      # Test and validate
)

result = workflow.execute(
    source="Aave",
    entities="depositors",
    dimensions=["size", "timing", "asset_selection"],
)
```

#### 5.9 Teaching Agents New Skills

**Meta-pattern: How we extracted skills from this tutorial**

1. **Identify repeatable patterns**: What did we do more than once?
2. **Abstract the steps**: What's the general form vs specific instance?
3. **Define inputs/outputs**: What varies? What's fixed?
4. **Create prompt template**: How would you instruct someone to do this?
5. **Note transferability**: Where else does this pattern apply?

**User exercise:**
> "Review your work in Parts 1-4. Identify one additional pattern you used
> repeatedly that could become a skill. Document it following the skill template."

**Connect to knowledge layer:**
```python
# Agent remembers which skills worked for which problems
knowledge.record_skill_usage(
    skill='signal_extraction',
    context='Hyperliquid top traders',
    outcome='success',
    notes='Consensus signals outperformed mirror signals'
)

# Later, agent can query
knowledge.query("What skills worked well for blockchain trader analysis?")
```

### Deliverable

User has production-ready Python modules, MCP server integration working, extracted reusable agent skills, and has seen an agent complete an end-to-end research task.

---

## Cross-Cutting Concerns

### Code Snippets Philosophy

- **Runnable**: Every snippet can be executed as-is (with appropriate imports)
- **Self-contained**: Includes sample data fallbacks for offline testing
- **Progressive**: Simple version first, then "production" version
- **Commented**: Explains *why*, not just *what*

### Error Handling Throughout

**Common failure modes per section:**
- Part 1: LLM API key issues, Ollama not running
- Part 2: Hyperliquid rate limits, network timeouts
- Part 3: Missing data for some traders, NaN handling
- Part 4: Parameter explosion memory issues, slow validation
- Part 5: MCP connection failures, kernel timeouts

**Recovery patterns shown inline, not hand-waved.**

### Checkpoints & Verification

Each part ends with:
- "Verify your setup" checklist
- Sample outputs showing what "working" looks like
- Troubleshooting tips for common issues

### Appendices

**A: VectorBT.Pro LLM Settings Reference**
- All `vbt.settings.chat` options
- Provider-specific configuration
- Routing and fallback setup

**B: Hyperliquid API Reference**
- Rate limits and best practices
- Endpoint documentation
- WebSocket subscription patterns

**C: Recommended Ollama Models**
- By task type (coding, analysis, embeddings)
- Resource requirements
- Quality/speed tradeoffs

**D: Glossary**
- Hyperliquid-specific terms
- VectorBT concepts
- Quantitative trading terminology

---

## Success Criteria

A user completing this tutorial series will:

1. **Have working tools**: Environment configured, data pipeline operational
2. **Understand VectorBT.Pro**: Core APIs, LLM features, validation patterns
3. **Have analyzed real traders**: Scored, ranked, and understood top performers
4. **Have tested strategies**: Backtested with proper validation, know what works
5. **Be agent-ready**: Modular code, MCP integration, reusable skills
6. **Know how to extend**: Can apply patterns to new data sources and strategies

---

*Document version: 1.0*
*Created: 2026-01-06*

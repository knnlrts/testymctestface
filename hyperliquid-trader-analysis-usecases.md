# Hyperliquid Trader Behavior Analysis: 10 Creative Use Cases

This document outlines 10 creative use cases for analyzing trader behavior on the Hyperliquid platform using the [hyperliquid-python-sdk](https://github.com/hyperliquid-dex/hyperliquid-python-sdk).

---

## Table of Contents

1. [Whale Tracker & Position Alert System](#1-whale-tracker--position-alert-system)
2. [Smart Money Copy Trading Signal Generator](#2-smart-money-copy-trading-signal-generator)
3. [Liquidation Cascade Predictor](#3-liquidation-cascade-predictor)
4. [Funding Rate Arbitrage Opportunity Scanner](#4-funding-rate-arbitrage-opportunity-scanner)
5. [Trader DNA Profiler & Classification Engine](#5-trader-dna-profiler--classification-engine)
6. [Market Maker Performance Analytics Dashboard](#6-market-maker-performance-analytics-dashboard)
7. [Position Crowding & Sentiment Divergence Detector](#7-position-crowding--sentiment-divergence-detector)
8. [Trade Execution Quality Analyzer](#8-trade-execution-quality-analyzer)
9. [Wallet Clustering & Entity Attribution System](#9-wallet-clustering--entity-attribution-system)
10. [Behavioral Alpha Signal Generator](#10-behavioral-alpha-signal-generator)

---

## 1. Whale Tracker & Position Alert System

### Concept
Build a real-time monitoring system that tracks large traders ("whales") on Hyperliquid, alerting users when significant position changes occur that could signal upcoming market moves.

### SDK Methods Used
```python
info.user_state(address)          # Get whale positions
info.user_fills(address)          # Track whale trade activity
info.subscribe({"type": "userFills", "user": address})  # Real-time fill alerts
info.meta_and_asset_ctxs()        # Get open interest for context
```

### Implementation Approach
1. **Whale Discovery**: Scan known whale addresses or identify large accounts through on-chain analysis
2. **Position Monitoring**: Poll `user_state()` periodically to track position changes
3. **Real-time Alerts**: Subscribe to `userFills` WebSocket for instant notifications
4. **Contextual Analysis**: Compare whale moves against overall open interest changes

### Key Metrics to Track
- Position size changes (absolute and relative to account)
- Entry/exit prices vs. market price
- Leverage changes (risk appetite shifts)
- Accumulation/distribution patterns over time
- Correlation with subsequent price movements

### Creative Extensions
- **Whale Behavior Scoring**: Assign confidence scores based on historical accuracy
- **Whale Consensus Indicator**: Track when multiple whales align on the same trade
- **Contrarian Signals**: Alert when retail positions diverge from whale positions

---

## 2. Smart Money Copy Trading Signal Generator

### Concept
Identify consistently profitable traders on Hyperliquid and generate actionable trading signals based on their activity, with risk-adjusted position sizing.

### SDK Methods Used
```python
info.user_fills(address)                    # Historical trade data
info.user_fills_by_time(address, start, end)  # Time-bounded analysis
info.portfolio(address)                      # Performance metrics
info.user_funding_history(address, start, end)  # Funding costs
info.user_fees(address)                      # Fee tier information
info.open_orders(address)                    # Current order book
```

### Implementation Approach
1. **Performance Ranking**:
   - Calculate realized PnL from `closedPnl` in fills
   - Factor in funding costs from `user_funding_history`
   - Compute Sharpe ratio and maximum drawdown

2. **Signal Generation**:
   - Monitor top performers' `open_orders` and new fills
   - Filter signals by minimum conviction (position size relative to portfolio)
   - Apply delay to prevent front-running

3. **Risk Management**:
   - Scale position sizes based on your account vs. signal source
   - Set maximum correlation limits between copied positions
   - Implement drawdown circuit breakers

### Trader Scoring Algorithm
```
Score = (Win_Rate × 0.2) + (Profit_Factor × 0.3) + (Sharpe_Ratio × 0.3) + (Consistency × 0.2)

Where:
- Win_Rate = Profitable trades / Total trades
- Profit_Factor = Gross profits / Gross losses
- Sharpe_Ratio = (Mean return - Risk-free) / Std deviation
- Consistency = 1 - (Monthly return variance / Mean monthly return)
```

### Creative Extensions
- **Strategy Isolation**: Detect and separate different strategies used by the same trader
- **Regime Detection**: Only copy trades in market conditions where the trader historically performs
- **Anti-Correlation Overlay**: Combine signals from traders with negatively correlated returns

---

## 3. Liquidation Cascade Predictor

### Concept
Build a predictive model that identifies price levels where concentrated liquidations could trigger cascading effects, helping traders position ahead of volatility events.

### SDK Methods Used
```python
info.user_state(address)           # Individual liquidation prices
info.meta_and_asset_ctxs()         # Open interest data
info.l2_snapshot(coin)             # Order book depth
info.funding_history(coin, start, end)  # Funding rate trends
info.subscribe({"type": "userEvents"})  # Real-time liquidation events
```

### Implementation Approach
1. **Liquidation Map Construction**:
   - Sample many trader addresses to estimate liquidation price distribution
   - Weight by position size and leverage
   - Build price-level histograms of potential liquidations

2. **Cascade Probability Modeling**:
   - Compare liquidation clusters against order book depth
   - Calculate "liquidation absorption capacity" at each level
   - Model second-order effects (liquidation → price move → more liquidations)

3. **Risk Heat Map**:
   ```
   Liquidation Intensity at Price P = Σ(Position_Size × Leverage) for all positions
                                       where |Liquidation_Price - P| < threshold
   ```

### Key Indicators
- **Liquidation Density**: Concentration of liquidation prices at specific levels
- **Thin Book Zones**: Price levels with low order book depth near liquidation clusters
- **Funding Extremes**: High funding rates indicating crowded positioning
- **Open Interest Concentration**: High OI with extreme average leverage

### Creative Extensions
- **Historical Validation**: Backtest predictions against actual liquidation events
- **Cross-Asset Contagion**: Model how liquidations in BTC affect ETH positions
- **Insurance Opportunity Finder**: Identify optimal hedge positions before cascade events

---

## 4. Funding Rate Arbitrage Opportunity Scanner

### Concept
Create a system that identifies funding rate arbitrage opportunities and tracks traders who successfully exploit them, potentially replicating their strategies.

### SDK Methods Used
```python
info.funding_history(coin, start, end)       # Historical funding rates
info.meta_and_asset_ctxs()                   # Current funding rates
info.user_funding_history(user, start, end)  # Individual funding P&L
info.user_state(address)                     # Position exposure
info.spot_meta_and_asset_ctxs()              # Spot market data
```

### Implementation Approach
1. **Funding Rate Analysis**:
   - Track funding rates across all perpetual markets
   - Calculate annualized funding yields
   - Identify persistent funding rate divergences

2. **Arbitrage Detection**:
   - Spot-Perp basis trades: Compare perpetual funding vs. spot carry
   - Cross-exchange funding: Compare Hyperliquid funding vs. CEX rates
   - Intra-platform: Long underfunded assets, short overfunded assets

3. **Professional Arb Tracker**:
   - Identify wallets with consistent positive funding income
   - Analyze their position structures
   - Replicate successful hedged positions

### Opportunity Scoring
```
Arb_Score = (Annualized_Funding_Yield - Execution_Costs - Margin_Opportunity_Cost) × Risk_Adjustment

Where:
- Annualized_Funding_Yield = 8h_funding_rate × 3 × 365
- Execution_Costs = Entry_slippage + Exit_slippage + Fees
- Risk_Adjustment = 1 / (1 + Historical_Funding_Volatility)
```

### Creative Extensions
- **Funding Prediction Model**: Predict future funding rates from order flow imbalances
- **Optimal Entry Timing**: Identify best times to enter funding trades (pre-funding vs. post)
- **Multi-Leg Strategies**: Combine funding harvesting with directional views

---

## 5. Trader DNA Profiler & Classification Engine

### Concept
Develop a machine learning system that creates behavioral "fingerprints" for traders, classifying them into archetypes and predicting their future behavior patterns.

### SDK Methods Used
```python
info.user_fills(address)              # Trade history
info.historical_orders(address)       # Order placement patterns
info.user_state(address)              # Position preferences
info.user_fees(address)               # Volume patterns
info.frontend_open_orders(address)    # Order types (TP/SL usage)
```

### Trader Archetypes

| Archetype | Characteristics | Identifiers |
|-----------|----------------|-------------|
| **Scalper** | High frequency, small profits | >50 trades/day, <0.1% avg profit, <5min hold time |
| **Swing Trader** | Medium frequency, trend following | 1-5 trades/day, 1-5% targets, 4h-7d holds |
| **Position Trader** | Low frequency, macro views | <5 trades/week, >10% targets, weeks-months holds |
| **Market Maker** | Two-sided quotes, inventory management | High cancel rate, symmetric orders, tight spreads |
| **Momentum Chaser** | Follows price action | Buys green candles, sells red, high correlation to price |
| **Mean Reverter** | Fades moves | Sells rallies, buys dips, negative correlation to price |
| **News Trader** | Event-driven | Burst trading around announcements |
| **Funding Farmer** | Exploits funding rates | Delta-neutral positions, long holding periods |

### Feature Engineering
```python
features = {
    # Timing Features
    'avg_hold_time': mean(exit_time - entry_time),
    'trade_hour_distribution': histogram(trade_times % 24),
    'weekend_activity_ratio': weekend_trades / weekday_trades,

    # Size Features
    'avg_position_size': mean(position_sizes),
    'size_consistency': std(position_sizes) / mean(position_sizes),
    'leverage_preference': median(leverages),

    # Behavior Features
    'maker_taker_ratio': maker_fills / taker_fills,
    'tp_sl_usage_rate': orders_with_tp_sl / total_orders,
    'modification_frequency': modifications / orders,
    'avg_time_to_modify': mean(modification_delays),

    # Performance Features
    'win_rate': profitable_trades / total_trades,
    'avg_winner_loser_ratio': mean(winning_trades) / abs(mean(losing_trades)),
    'max_drawdown': max_peak_to_trough_decline,

    # Market Condition Features
    'high_vol_activity': trades_during_high_vol / total_trades,
    'trend_alignment': correlation(trade_direction, price_trend)
}
```

### Creative Extensions
- **Behavior Drift Detection**: Alert when a trader's pattern changes significantly
- **Cluster Evolution**: Track how trader archetypes shift during market regimes
- **Cross-Reference Analysis**: Identify possible sybil accounts or coordinated trading

---

## 6. Market Maker Performance Analytics Dashboard

### Concept
Build comprehensive analytics for market makers operating on Hyperliquid, measuring their profitability, inventory risk, and market impact.

### SDK Methods Used
```python
info.user_fills(address)              # All executions
info.user_state(address)              # Current inventory
info.l2_snapshot(coin)                # Order book context
info.user_funding_history(address, start, end)  # Funding impact
info.user_fees(address)               # Fee tier/rebates
info.subscribe({"type": "l2Book"})    # Real-time book
```

### Key Performance Metrics

**Profitability Metrics**:
- **Gross Spread Capture**: Revenue from bid-ask spread
- **Net P&L**: Gross spread - Inventory losses - Funding costs
- **Sharpe Ratio**: Risk-adjusted returns
- **Inventory Turnover**: Volume / Average inventory

**Risk Metrics**:
- **Inventory Skew**: Net position / Max position limit
- **Adverse Selection**: Losses from informed flow
- **Time-Weighted Inventory**: Average absolute position over time

**Efficiency Metrics**:
- **Quote-to-Trade Ratio**: Orders placed / Orders filled
- **Time at Best**: Duration with best bid or offer
- **Fill Rate**: Percentage of quotes that get filled

### Analysis Framework
```
MM_Score = (Spread_Capture × 0.3) + (Risk_Adjusted_Return × 0.3) +
           (Market_Share × 0.2) + (Inventory_Efficiency × 0.2)

Adverse_Selection_Ratio = Σ(Unfavorable_Fills × Size) / Σ(All_Fills × Size)

Where unfavorable = filled and price immediately moves against you
```

### Creative Extensions
- **Toxic Flow Detector**: Identify informed traders to avoid
- **Optimal Spread Calculator**: Dynamic spread based on volatility and inventory
- **Competitor Benchmarking**: Compare MM performance anonymously

---

## 7. Position Crowding & Sentiment Divergence Detector

### Concept
Create a system that measures how "crowded" positions are becoming and identifies dangerous sentiment extremes that often precede reversals.

### SDK Methods Used
```python
info.meta_and_asset_ctxs()            # Open interest data
info.funding_history(coin, start, end)  # Funding trends
info.l2_snapshot(coin)                # Order book imbalance
info.subscribe({"type": "trades"})    # Trade flow
info.all_mids()                       # Price levels
```

### Crowding Indicators

**1. Open Interest Analysis**
```python
oi_change_rate = (current_oi - oi_24h_ago) / oi_24h_ago
price_oi_divergence = correlation(price_changes, oi_changes)
# Negative divergence = rising OI with falling price (shorts crowding)
```

**2. Funding Rate Extremes**
```python
funding_zscore = (current_funding - mean_funding) / std_funding
# |zscore| > 2 indicates extreme positioning
```

**3. Long/Short Ratio Proxy**
```python
# Inferred from funding rate direction and magnitude
implied_ls_ratio = 1 + (funding_rate × sensitivity_factor)
```

**4. Order Book Imbalance**
```python
book_imbalance = (bid_depth - ask_depth) / (bid_depth + ask_depth)
# Persistent imbalance suggests directional positioning
```

### Sentiment Divergence Signals
| Signal | Condition | Interpretation |
|--------|-----------|----------------|
| **Bullish Divergence** | Price falling + OI rising + Funding negative | Shorts crowding, potential squeeze |
| **Bearish Divergence** | Price rising + OI rising + Funding positive | Longs crowding, potential flush |
| **Capitulation** | Price falling + OI falling rapidly | Forced exits, potential bottom |
| **Distribution** | Price rising + OI falling | Smart money exiting |

### Creative Extensions
- **Cross-Asset Sentiment**: Correlate crowding across BTC, ETH, alts
- **Historical Analogs**: Find past instances with similar crowding patterns
- **Optimal Contrarian Entry**: Backtest contrarian entries at sentiment extremes

---

## 8. Trade Execution Quality Analyzer

### Concept
Build a tool that measures how well traders execute their orders, identifying slippage, market impact, and optimal execution patterns.

### SDK Methods Used
```python
info.user_fills(address)              # Execution prices
info.user_fills_by_time(address, start, end)  # Time analysis
info.l2_snapshot(coin)                # Book depth at execution
info.candles_snapshot(coin, interval, start, end)  # VWAP reference
info.historical_orders(address)       # Order vs execution comparison
```

### Execution Quality Metrics

**1. Slippage Analysis**
```python
# For market orders
slippage_bps = (execution_price - mid_price_at_order) / mid_price_at_order × 10000

# For limit orders
limit_improvement = (limit_price - execution_price) / limit_price × 10000
# Positive = better than expected
```

**2. VWAP/TWAP Comparison**
```python
vwap_slippage = (avg_execution_price - period_vwap) / period_vwap × 10000
# Compare large order execution to market VWAP
```

**3. Implementation Shortfall**
```python
implementation_shortfall = (decision_price - avg_execution_price) / decision_price
# Measures total cost from decision to completion
```

**4. Market Impact**
```python
temporary_impact = price_at_fill - price_before_order
permanent_impact = price_5min_after - price_before_order
# Large orders should minimize permanent impact
```

### Execution Scoring
```
Execution_Score = 100 - (Slippage_Penalty + Impact_Penalty + Timing_Penalty)

Where:
- Slippage_Penalty = max(0, slippage_bps - expected_slippage) × 2
- Impact_Penalty = permanent_impact_bps × 3
- Timing_Penalty = (order_duration / optimal_duration - 1) × 10
```

### Creative Extensions
- **Optimal Execution Advisor**: Recommend order sizes and timing based on book depth
- **A/B Testing Framework**: Compare execution strategies objectively
- **Venue Analysis**: Compare Hyperliquid execution to other platforms

---

## 9. Wallet Clustering & Entity Attribution System

### Concept
Develop algorithms to identify related wallets (same entity), track capital flows between them, and attribute behavior patterns to entities rather than individual addresses.

### SDK Methods Used
```python
info.user_fills(address)              # Trading patterns
info.user_state(address)              # Position matching
info.query_sub_accounts(address)      # Known sub-accounts
info.user_non_funding_ledger_updates(address, start, end)  # Transfers
info.extra_agents(address)            # Connected agents
```

### Clustering Heuristics

**1. Trading Pattern Similarity**
```python
similarity_score = cosine_similarity(trading_features_A, trading_features_B)
# High similarity suggests same operator

features = [
    trade_hour_distribution,
    avg_hold_time,
    coin_preferences,
    leverage_patterns,
    entry_style  # market vs limit, size patterns
]
```

**2. Temporal Correlation**
```python
# Wallets that consistently trade together
trade_time_correlation = correlation(trades_A_timestamps, trades_B_timestamps)
# High correlation with small lag suggests coordination
```

**3. Position Mirroring**
```python
position_mirror_score = correlation(positions_A_over_time, positions_B_over_time)
# Near-identical position histories suggest same entity
```

**4. Capital Flow Analysis**
```python
# Direct transfers indicate related wallets
transfer_graph = build_graph(all_transfers)
connected_components = find_clusters(transfer_graph)
```

### Entity Profile Construction
```python
entity = {
    'wallets': [addr1, addr2, addr3],
    'total_equity': sum(wallet_equities),
    'combined_positions': aggregate(all_positions),
    'behavior_profile': merge(individual_profiles),
    'estimated_entity_type': 'fund' | 'market_maker' | 'retail_whale'
}
```

### Creative Extensions
- **New Wallet Alert**: Detect when known entities create new wallets
- **Entity Leaderboard**: Rank entities by total performance
- **Coordination Detection**: Identify potential manipulation patterns

---

## 10. Behavioral Alpha Signal Generator

### Concept
Create trading signals derived purely from trader behavior patterns, using crowd behavior as a contrarian or momentum indicator.

### SDK Methods Used
```python
info.meta_and_asset_ctxs()            # OI and funding
info.funding_history(coin, start, end)  # Funding trends
info.l2_snapshot(coin)                # Book dynamics
info.subscribe({"type": "trades"})    # Real-time flow
info.user_fills_by_time(addr, start, end)  # Trader timing
```

### Alpha Signals

**Signal 1: Funding Reversion**
```python
# Extreme funding reverts to mean
signal_strength = -funding_zscore × mean_reversion_coefficient

trade_signal = {
    'direction': 'long' if funding_zscore < -2 else 'short' if funding_zscore > 2 else 'neutral',
    'confidence': min(abs(funding_zscore) / 3, 1.0)
}
```

**Signal 2: Smart Money Divergence**
```python
# When smart money (top performers) diverges from crowd
smart_money_direction = weighted_avg(top_trader_positions, weights=performance_scores)
crowd_direction = sign(total_oi × sign(funding_rate))

divergence_signal = smart_money_direction × -crowd_direction
# Positive = smart money contrarian, strong signal
```

**Signal 3: Liquidation Gravity**
```python
# Price tends to move toward liquidation clusters
nearest_liq_cluster_above = find_cluster(liquidation_map, 'above', current_price)
nearest_liq_cluster_below = find_cluster(liquidation_map, 'below', current_price)

gravity_signal = (above_cluster_value - below_cluster_value) / total_cluster_value
# Positive = more liquidations above, price may move up to trigger them
```

**Signal 4: Order Flow Toxicity**
```python
# Track informed vs uninformed flow
toxicity_score = sum(adverse_fills) / sum(all_fills)

# High toxicity = informed traders active, fade retail
if toxicity_score > threshold:
    signal = -retail_flow_direction
```

**Signal 5: Behavioral Momentum**
```python
# Traders increasing position = conviction
position_change_momentum = Σ(position_changes × trader_weights)

signal = sign(position_change_momentum) × min(abs(position_change_momentum) / baseline, 1.0)
```

### Signal Combination Framework
```python
combined_signal = (
    funding_signal × 0.25 +
    smart_money_signal × 0.30 +
    liquidation_signal × 0.15 +
    toxicity_signal × 0.15 +
    momentum_signal × 0.15
)

position_size = base_size × combined_signal × volatility_adjustment
```

### Creative Extensions
- **Regime-Adaptive Weights**: Adjust signal weights based on market conditions
- **Multi-Timeframe Aggregation**: Combine signals across different time horizons
- **Cross-Validation System**: Continuously backtest and update signal parameters

---

## Implementation Architecture

### Recommended System Design

```
┌─────────────────────────────────────────────────────────────────┐
│                        Data Collection Layer                      │
├─────────────────────────────────────────────────────────────────┤
│  WebSocket Manager          │  REST API Poller                   │
│  - Real-time trades         │  - Historical fills                │
│  - Order book updates       │  - User states                     │
│  - User events              │  - Funding history                 │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Data Processing Layer                      │
├─────────────────────────────────────────────────────────────────┤
│  Feature Engineering        │  Entity Resolution                 │
│  - Trader metrics           │  - Wallet clustering               │
│  - Market indicators        │  - Behavior fingerprinting         │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Analysis Layer                             │
├─────────────────────────────────────────────────────────────────┤
│  Signal Generation          │  Risk Analysis                     │
│  - Behavioral alpha         │  - Liquidation mapping             │
│  - Copy trading signals     │  - Crowding detection              │
│  - Sentiment indicators     │  - Execution quality               │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Presentation Layer                         │
├─────────────────────────────────────────────────────────────────┤
│  Dashboard                  │  Alerts                            │
│  - Real-time metrics        │  - Whale movements                 │
│  - Performance tracking     │  - Signal triggers                 │
│  - Historical analysis      │  - Risk warnings                   │
└─────────────────────────────────────────────────────────────────┘
```

### Getting Started

```python
from hyperliquid.info import Info
from hyperliquid.utils import constants

# Initialize the Info client
info = Info(constants.MAINNET_API_URL, skip_ws=False)

# Example: Get trader data for analysis
trader_address = "0x..."
user_state = info.user_state(trader_address)
fills = info.user_fills(trader_address)
portfolio = info.portfolio(trader_address)

# Subscribe to real-time updates
def on_fill(msg):
    print(f"New fill: {msg}")

info.subscribe({"type": "userFills", "user": trader_address}, on_fill)
```

---

## Conclusion

The Hyperliquid Python SDK provides rich data access for building sophisticated trader behavior analysis systems. These 10 use cases represent a spectrum from straightforward monitoring (whale tracking) to advanced alpha generation (behavioral signals).

The key to success is combining multiple data sources—positions, trades, funding, order book—to build a comprehensive picture of market participant behavior. Whether you're building tools for your own trading, creating analytics products, or conducting market research, the SDK offers the building blocks needed for deep trader behavior analysis.

### Next Steps

1. **Start Simple**: Begin with use case #1 (Whale Tracker) to understand the data
2. **Build Infrastructure**: Create robust data collection and storage pipelines
3. **Iterate**: Add more sophisticated analysis as you understand the data better
4. **Validate**: Always backtest signals before trading on them
5. **Monitor**: Track performance and adapt to changing market conditions

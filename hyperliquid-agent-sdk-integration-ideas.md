# Hyperliquid + Claude Agent SDK: 12 Creative Integration Ideas

This document explores creative ways to leverage the [Claude Agent SDK for Python](https://github.com/anthropics/claude-agent-sdk-python) to enhance the Hyperliquid trader analysis use cases outlined in [hyperliquid-trader-analysis-usecases.md](./hyperliquid-trader-analysis-usecases.md).

## Overview

The Claude Agent SDK provides powerful capabilities that can transform traditional trading analytics into intelligent, conversational, and autonomous systems:

- **Custom Tools (SDK MCP Servers)**: Expose trading data and operations as tools Claude can invoke
- **Hooks**: Automated validation, feedback, and decision-making at key execution points
- **Custom Agents**: Specialized AI agents with domain-specific prompts and tool access
- **Streaming Interactions**: Real-time bidirectional conversations with state management

---

## 1. Multi-Agent Trading Research System

### Concept
Deploy a team of specialized AI agents that collaboratively analyze Hyperliquid markets from different perspectives, synthesizing insights that no single approach could provide.

### Agent SDK Integration

**Custom Agents:**
```python
options = ClaudeAgentOptions(
    agents={
        "whale-analyst": AgentDefinition(
            description="Analyzes whale trader behavior and large position movements",
            prompt="You are a whale trading analyst. Focus on identifying significant position changes, "
                   "accumulation/distribution patterns, and their market implications.",
            tools=["hyperliquid_user_state", "hyperliquid_user_fills", "hyperliquid_oi_data"],
            model="sonnet"
        ),
        "technical-analyst": AgentDefinition(
            description="Performs technical analysis on price and volume data",
            prompt="You are a technical analyst. Identify chart patterns, support/resistance levels, "
                   "and technical indicators. Provide actionable trade setups.",
            tools=["hyperliquid_candles", "hyperliquid_orderbook", "hyperliquid_trades"],
            model="sonnet"
        ),
        "sentiment-analyst": AgentDefinition(
            description="Analyzes market sentiment from positioning data",
            prompt="You are a sentiment analyst. Evaluate funding rates, long/short ratios, "
                   "and crowding metrics to identify sentiment extremes.",
            tools=["hyperliquid_funding", "hyperliquid_oi_data", "hyperliquid_liquidations"],
            model="sonnet"
        ),
        "risk-manager": AgentDefinition(
            description="Evaluates risk and position sizing",
            prompt="You are a risk manager. Analyze portfolio exposure, calculate optimal position sizes, "
                   "and identify liquidation risks. Always prioritize capital preservation.",
            tools=["hyperliquid_user_state", "calculate_kelly", "portfolio_var"],
            model="sonnet"
        )
    }
)

# Orchestrate multi-agent collaboration
async with ClaudeSDKClient(options=options) as client:
    await client.query(
        "Analyze BTC-USD on Hyperliquid. whale-analyst: check top holder activity. "
        "technical-analyst: identify key levels. sentiment-analyst: evaluate positioning. "
        "risk-manager: suggest position sizing for a swing trade."
    )
```

### Benefits
- **Comprehensive Analysis**: Each agent focuses on its specialty, avoiding jack-of-all-trades dilution
- **Consistent Methodology**: Agents maintain their analytical framework across all queries
- **Collaborative Intelligence**: Agents can build on each other's findings
- **Audit Trail**: Each agent's reasoning is captured separately

### Implementation Details
- Create custom tools for each Hyperliquid SDK method using `@tool` decorator
- Design agent prompts with specific analytical frameworks (e.g., whale analyst uses accumulation/distribution theory)
- Implement agent coordination logic that ensures proper sequencing
- Store agent outputs in structured format for downstream processing

---

## 2. Conversational Risk Management System

### Concept
Transform complex risk analytics into natural language conversations, allowing traders to query their risk exposure in plain English and receive contextual explanations.

### Agent SDK Integration

**Custom Tools for Risk Metrics:**
```python
@tool(
    "get_portfolio_risk",
    "Get comprehensive portfolio risk metrics including VaR, liquidation proximity, and exposure breakdown",
    {"address": str, "confidence_level": float}
)
async def get_portfolio_risk(args):
    address = args["address"]
    confidence = args.get("confidence_level", 0.95)

    # Fetch user state from Hyperliquid
    user_state = info.user_state(address)

    # Calculate risk metrics
    portfolio_value = calculate_portfolio_value(user_state)
    var_1d = calculate_var(user_state, confidence, period_days=1)
    liquidation_distance = calculate_liquidation_proximity(user_state)
    exposure_by_asset = calculate_exposure_breakdown(user_state)

    risk_report = {
        "portfolio_value_usd": portfolio_value,
        "value_at_risk_1d": var_1d,
        "var_as_percentage": (var_1d / portfolio_value) * 100,
        "closest_liquidation_pct": liquidation_distance,
        "exposure_breakdown": exposure_by_asset,
        "risk_level": classify_risk_level(liquidation_distance, var_1d)
    }

    return {
        "content": [{
            "type": "text",
            "text": json.dumps(risk_report, indent=2)
        }]
    }

@tool(
    "calculate_position_size",
    "Calculate optimal position size using Kelly Criterion or fixed risk percentage",
    {"method": str, "risk_per_trade_pct": float, "win_rate": float, "profit_loss_ratio": float}
)
async def calculate_position_size(args):
    # Kelly Criterion: f = (bp - q) / b
    # where f = fraction of capital to wager, p = probability of win,
    # q = probability of loss, b = profit/loss ratio

    if args["method"] == "kelly":
        p = args["win_rate"]
        q = 1 - p
        b = args["profit_loss_ratio"]
        kelly_fraction = (b * p - q) / b
        kelly_half = kelly_fraction * 0.5  # Conservative Kelly

        return {
            "content": [{
                "type": "text",
                "text": f"Kelly Criterion: {kelly_fraction:.2%} (Half-Kelly: {kelly_half:.2%})\n"
                       f"Recommendation: Use half-Kelly for safety ({kelly_half:.2%} of portfolio)"
            }]
        }
    # ... other methods

# Create MCP server with risk tools
risk_server = create_sdk_mcp_server(
    name="risk-management",
    version="1.0.0",
    tools=[get_portfolio_risk, calculate_position_size, check_correlation_risk]
)

options = ClaudeAgentOptions(
    mcp_servers={"risk": risk_server},
    allowed_tools=["mcp__risk__get_portfolio_risk", "mcp__risk__calculate_position_size"]
)

# Natural language interaction
async with ClaudeSDKClient(options=options) as client:
    await client.query("What's my current risk exposure and how close am I to liquidation?")
    # Claude will invoke get_portfolio_risk and explain the results

    await client.query(
        "If my strategy has a 60% win rate and 2:1 reward:risk ratio, "
        "what position size should I use?"
    )
    # Claude will invoke calculate_position_size with Kelly Criterion
```

### Benefits
- **Accessibility**: Non-technical traders can access sophisticated risk analytics
- **Context-Aware**: Claude can combine multiple risk metrics to provide holistic assessment
- **Educational**: Explanations help traders understand risk concepts
- **Actionable**: Direct recommendations based on quantitative analysis

---

## 3. Automated Trade Journal with AI Analysis

### Concept
Use PostToolUse hooks to automatically capture every trade execution, then leverage Claude to identify patterns, diagnose mistakes, and track progress toward trading goals.

### Agent SDK Integration

**PostToolUse Hook for Trade Journaling:**
```python
async def auto_journal_trade(input_data, tool_use_id, context):
    """Automatically journal trades and provide instant feedback."""
    tool_name = input_data.get("tool_name")
    tool_input = input_data.get("tool_input", {})
    tool_response = input_data.get("tool_response", {})

    # Only journal actual trade executions
    if tool_name != "execute_hyperliquid_order":
        return {}

    # Extract trade details
    trade_data = {
        "timestamp": datetime.now().isoformat(),
        "coin": tool_input.get("coin"),
        "side": tool_input.get("side"),
        "size": tool_input.get("size"),
        "price": tool_response.get("fill_price"),
        "order_type": tool_input.get("order_type"),
        "context": context.get("user_prompt")  # Why did they make this trade?
    }

    # Store in trade journal database
    journal_db.insert_trade(trade_data)

    # Get recent trading pattern
    recent_trades = journal_db.get_recent_trades(limit=10)
    win_rate = calculate_win_rate(recent_trades)

    # Provide immediate feedback
    feedback = []

    if is_revenge_trading(recent_trades):
        feedback.append("⚠️ Warning: Detected potential revenge trading pattern")

    if is_overtrading(recent_trades):
        feedback.append("⚠️ You've made 15 trades today. Consider taking a break.")

    if win_rate < 0.4 and len(recent_trades) >= 10:
        feedback.append(
            f"📊 Recent win rate: {win_rate:.1%}. Consider reviewing your strategy."
        )

    return {
        "hookSpecificOutput": {
            "hookEventName": "PostToolUse",
            "additionalContext": "\n".join(feedback) if feedback else "Trade journaled successfully."
        },
        "systemMessage": "\n".join(feedback) if feedback else None
    }

# Setup with journaling hook
options = ClaudeAgentOptions(
    hooks={
        "PostToolUse": [
            HookMatcher(matcher="execute_hyperliquid_order", hooks=[auto_journal_trade])
        ]
    }
)

# Later, analyze the journal
async with ClaudeSDKClient(options=options) as client:
    await client.query(
        "Review my trade journal for the past week. What patterns do you see? "
        "What mistakes am I repeating? What's working well?"
    )
    # Claude can access the journal database and provide detailed analysis
```

### Benefits
- **Zero Friction**: Journaling happens automatically, no manual entry
- **Real-Time Feedback**: Immediate alerts prevent compounding mistakes
- **Pattern Recognition**: AI identifies subtle patterns humans miss
- **Accountability**: Complete record of all decisions and outcomes

---

## 4. Smart Alert System with Contextual Intelligence

### Concept
Instead of basic threshold alerts, create an intelligent system where Claude synthesizes multiple data points to provide rich, contextual explanations of market events.

### Agent SDK Integration

**Custom Tools + Background Monitoring:**
```python
@tool(
    "explain_whale_movement",
    "Analyze and explain a whale's recent position change with full market context",
    {"whale_address": str, "event_type": str}
)
async def explain_whale_movement(args):
    address = args["whale_address"]
    event = args["event_type"]

    # Gather comprehensive data
    user_state = info.user_state(address)
    recent_fills = info.user_fills(address)
    market_context = info.meta_and_asset_ctxs()
    orderbook = info.l2_snapshot(get_primary_coin(user_state))

    # Historical comparison
    historical_behavior = analyze_trader_history(address)

    context_package = {
        "event": event,
        "current_positions": user_state,
        "recent_trades": recent_fills[-10:],
        "market_oi": market_context,
        "orderbook_depth": calculate_book_depth(orderbook),
        "trader_profile": historical_behavior,
        "similar_past_events": find_similar_events(address, event)
    }

    return {
        "content": [{
            "type": "text",
            "text": json.dumps(context_package, indent=2)
        }]
    }

# Background monitoring process
async def monitor_and_alert():
    """Background process that monitors for significant events."""
    whale_addresses = load_whale_list()

    async with ClaudeSDKClient(options=alert_options) as client:
        while True:
            for address in whale_addresses:
                # Check for significant position changes
                if significant_change_detected(address):
                    # Use Claude to generate contextual alert
                    await client.query(
                        f"Whale {address} just made a significant move. "
                        f"Use explain_whale_movement to analyze this and send an alert "
                        f"to the user explaining what happened and why it matters."
                    )

                    async for msg in client.receive_response():
                        # Send rich alert to user (Telegram, Discord, email, etc.)
                        await send_alert(msg)

            await asyncio.sleep(30)  # Check every 30 seconds
```

**Example Alert Output:**
```
🐋 Whale Alert: 0x1234...5678

Position Change: Increased BTC-USD long from 50 → 125 BTC (+150%)

Context & Analysis:
• This whale has a strong track record (65% win rate over 90 days)
• They typically hold positions for 3-7 days (swing trader profile)
• Entry price $95,420 is at a key support level (previous resistance from Nov 2024)
• Current funding rate: +0.01% (neutral, not overcrowded)
• Open interest increased by 5% in the past hour, suggesting other large players also entering
• Order book shows strong bid support at $95,000

Historical Pattern:
This whale last made a similar size increase on Dec 15th. That position was held for 4 days with a +8.5% profit.

Market Implication:
This move suggests bullish conviction from a skilled trader at a technical support level.
The neutral funding rate indicates this isn't a crowded trade yet. Consider this as a
potential confirmation signal if it aligns with your own analysis.

Risk Note: Watch for position reduction, which historically happens quickly if price
breaks below entry -2%.
```

### Benefits
- **Signal, Not Noise**: Only get alerts when Claude determines events are truly significant
- **Educational**: Every alert teaches you something about market dynamics
- **Actionable**: Clear implications for your own trading decisions
- **Contextual**: Combines technical, fundamental, and behavioral analysis

---

## 5. Hook-Based Position Size & Risk Validator

### Concept
Use PreToolUse hooks to automatically validate every trade against risk management rules before execution, with Claude explaining why trades are approved or rejected.

### Agent SDK Integration

**PreToolUse Hook for Trade Validation:**
```python
async def validate_trade_risk(input_data, tool_use_id, context):
    """Validate trade against risk management rules before execution."""
    tool_name = input_data.get("tool_name")
    tool_input = input_data.get("tool_input", {})

    if tool_name != "execute_hyperliquid_order":
        return {}

    # Get current portfolio state
    portfolio = get_current_portfolio()

    # Extract trade parameters
    coin = tool_input.get("coin")
    side = tool_input.get("side")
    size_usd = tool_input.get("size_usd")
    leverage = tool_input.get("leverage", 1)

    # Risk checks
    violations = []

    # Check 1: Position size limit (max 5% per trade)
    max_position_size = portfolio.total_value * 0.05
    if size_usd > max_position_size:
        violations.append(
            f"Position size ${size_usd:,.0f} exceeds maximum ${max_position_size:,.0f} (5% rule)"
        )

    # Check 2: Leverage limit
    if leverage > 3:
        violations.append(
            f"Leverage {leverage}x exceeds maximum 3x policy"
        )

    # Check 3: Concentration risk (max 25% in single asset)
    proposed_exposure = calculate_exposure_after_trade(portfolio, coin, side, size_usd)
    if proposed_exposure > 0.25:
        violations.append(
            f"Trade would result in {proposed_exposure:.1%} exposure to {coin}, "
            f"exceeding 25% concentration limit"
        )

    # Check 4: Correlated position risk
    correlation_risk = calculate_correlation_risk(portfolio, coin, size_usd)
    if correlation_risk > 0.5:
        violations.append(
            f"High correlation risk detected ({correlation_risk:.1%}). "
            f"Portfolio already has significant correlated exposure."
        )

    # Check 5: Daily loss limit
    daily_pnl = calculate_daily_pnl(portfolio)
    if daily_pnl < -portfolio.total_value * 0.03:  # -3% daily loss limit
        violations.append(
            f"Daily loss limit reached (${abs(daily_pnl):,.0f}). "
            f"No new trades allowed until tomorrow."
        )

    # If violations exist, deny the trade
    if violations:
        return {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny",
                "permissionDecisionReason": "Risk management rules violated:\n" +
                                          "\n".join(f"• {v}" for v in violations)
            },
            "systemMessage": "🛑 Trade blocked by risk management system"
        }

    # Trade passes all checks
    return {
        "hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": "allow",
            "permissionDecisionReason": "Trade passes all risk management checks"
        },
        "systemMessage": "✅ Trade approved"
    }

options = ClaudeAgentOptions(
    hooks={
        "PreToolUse": [
            HookMatcher(matcher="execute_hyperliquid_order", hooks=[validate_trade_risk])
        ]
    }
)

# Example usage
async with ClaudeSDKClient(options=options) as client:
    # This will be blocked if it violates risk rules
    await client.query(
        "Execute a long position on BTC-USD with $50,000 size at 10x leverage"
    )
    # User will see exactly why the trade was blocked
```

### Benefits
- **Automated Discipline**: Eliminates emotional override of risk rules
- **Transparent**: Clear explanation of why trades are blocked
- **Consistent**: Same rules applied to every trade without exception
- **Educational**: Traders learn proper risk management through feedback

---

## 6. Natural Language Backtesting Interface

### Concept
Allow traders to describe strategies in plain English and have Claude execute comprehensive backtests using Hyperliquid historical data.

### Agent SDK Integration

**Custom Tools for Backtesting:**
```python
@tool(
    "backtest_strategy",
    "Backtest a trading strategy on historical Hyperliquid data",
    {
        "coin": str,
        "start_date": str,
        "end_date": str,
        "entry_conditions": str,
        "exit_conditions": str,
        "position_size_pct": float,
        "leverage": float
    }
)
async def backtest_strategy(args):
    """Execute a backtest based on natural language strategy description."""

    # Parse strategy conditions (Claude formats these)
    entry_logic = parse_conditions(args["entry_conditions"])
    exit_logic = parse_conditions(args["exit_conditions"])

    # Fetch historical data
    candles = info.candles_snapshot(
        coin=args["coin"],
        interval="1h",
        startTime=parse_date(args["start_date"]),
        endTime=parse_date(args["end_date"])
    )

    # Run backtest
    results = run_backtest_engine(
        data=candles,
        entry_conditions=entry_logic,
        exit_conditions=exit_logic,
        position_size=args["position_size_pct"],
        leverage=args["leverage"]
    )

    # Generate comprehensive report
    report = {
        "performance": {
            "total_return": results.total_return_pct,
            "sharpe_ratio": results.sharpe,
            "max_drawdown": results.max_drawdown_pct,
            "win_rate": results.win_rate,
            "profit_factor": results.profit_factor
        },
        "trade_stats": {
            "total_trades": results.total_trades,
            "avg_win": results.avg_win_pct,
            "avg_loss": results.avg_loss_pct,
            "largest_win": results.largest_win_pct,
            "largest_loss": results.largest_loss_pct
        },
        "risk_metrics": {
            "volatility_annual": results.annual_volatility,
            "downside_deviation": results.downside_deviation,
            "var_95": results.value_at_risk_95
        },
        "trades_sample": results.trades[:5]  # First 5 trades as examples
    }

    return {
        "content": [{
            "type": "text",
            "text": json.dumps(report, indent=2)
        }]
    }

backtest_server = create_sdk_mcp_server(
    name="backtesting",
    tools=[backtest_strategy, optimize_parameters, monte_carlo_simulation]
)

# Natural language backtesting
async with ClaudeSDKClient(options=options) as client:
    await client.query(
        "Backtest this strategy on BTC-USD from Jan 2024 to Dec 2024: "
        "Enter long when RSI(14) crosses above 30 and 50-day MA is above 200-day MA. "
        "Exit when RSI crosses below 70 or price drops 3% from entry. "
        "Use 2% position size with 2x leverage."
    )

    # Claude will:
    # 1. Parse the strategy description
    # 2. Format it into structured conditions
    # 3. Call backtest_strategy tool
    # 4. Interpret results and provide insights
```

**Example Interaction:**
```
User: "Backtest a simple mean reversion strategy: buy when price is 2 standard
deviations below the 20-period moving average, exit when it returns to the mean.
Test it on ETH-USD for the last 6 months."

Claude: I'll backtest that mean reversion strategy for you. Let me set up the conditions:

Entry: Price < (20-period SMA - 2 * standard deviation)
Exit: Price >= 20-period SMA
Period: ETH-USD, last 6 months
Position sizing: I'll use 5% per trade with 1x leverage (let me know if you want different)

[Executes backtest]

Results Summary:
✅ Total Return: +23.4%
📊 Win Rate: 58.3% (42 wins, 30 losses)
📉 Max Drawdown: -8.2%
⚡ Sharpe Ratio: 1.85

This is a solid mean reversion strategy! The Sharpe ratio above 1.5 indicates good
risk-adjusted returns. The 58% win rate combined with controlled drawdowns suggests
the strategy has an edge.

Key Observations:
• Best performance during sideways/range-bound markets (Jun-Aug 2024)
• Struggled during strong trends (Sep-Oct 2024 rally)
• Average winning trade: +3.2%, Average losing trade: -2.1%

Recommendations:
1. Consider adding a trend filter to avoid entries during strong uptrends
2. The 8.2% max drawdown is reasonable, but you might want a trailing stop
3. Test with parameter optimization: maybe 1.5 or 2.5 standard deviations?

Would you like me to run an optimization to find the best parameter values?
```

### Benefits
- **Accessibility**: No coding required to backtest strategies
- **Rapid Iteration**: Test multiple variations in minutes
- **Learning**: Claude explains results and suggests improvements
- **Validation**: Quantitative evidence before risking real capital

---

## 7. Adaptive Strategy Optimizer with Continuous Learning

### Concept
Deploy an autonomous agent that continuously monitors live strategy performance, compares against historical backtests, and recommends parameter adjustments as market conditions evolve.

### Agent SDK Integration

**Custom Agent with Monitoring Loop:**
```python
@tool(
    "analyze_strategy_drift",
    "Compare live strategy performance to backtest expectations and detect regime changes",
    {"strategy_id": str, "lookback_days": int}
)
async def analyze_strategy_drift(args):
    """Detect when live performance diverges from backtest expectations."""

    strategy = load_strategy(args["strategy_id"])

    # Get live performance
    live_trades = get_live_trades(strategy.id, days=args["lookback_days"])
    live_metrics = calculate_metrics(live_trades)

    # Compare to backtest expectations
    backtest_metrics = strategy.expected_metrics

    drift_analysis = {
        "win_rate_drift": live_metrics.win_rate - backtest_metrics.win_rate,
        "sharpe_drift": live_metrics.sharpe - backtest_metrics.sharpe,
        "avg_profit_drift": live_metrics.avg_profit - backtest_metrics.avg_profit,
        "regime_change_detected": detect_regime_change(live_trades)
    }

    # If significant drift, analyze market conditions
    if abs(drift_analysis["win_rate_drift"]) > 0.15:
        market_regime = analyze_current_regime()
        drift_analysis["market_regime"] = market_regime
        drift_analysis["regime_different_from_backtest"] = True

    return {"content": [{"type": "text", "text": json.dumps(drift_analysis, indent=2)}]}

# Autonomous optimizer agent
optimizer_agent = AgentDefinition(
    description="Monitors strategy performance and recommends optimizations",
    prompt="""You are a quantitative strategy optimizer. Your job is to:
    1. Continuously monitor live strategy performance
    2. Detect when performance drifts from backtest expectations
    3. Identify market regime changes that affect strategy efficacy
    4. Recommend parameter adjustments or strategy pauses
    5. Run optimization experiments during off-hours

    Always prioritize capital preservation. If a strategy is underperforming significantly,
    recommend reducing position size or pausing until conditions improve.""",
    tools=[
        "analyze_strategy_drift",
        "backtest_strategy",
        "optimize_parameters",
        "get_market_regime"
    ],
    model="sonnet"
)

# Background monitoring
async def run_optimizer_loop():
    """Autonomous loop that monitors and optimizes strategies."""

    options = ClaudeAgentOptions(
        agents={"optimizer": optimizer_agent},
        mcp_servers={"trading": trading_server}
    )

    async with ClaudeSDKClient(options=options) as client:
        while True:
            # Daily performance review
            await client.query(
                "Review all active strategies. For each:\n"
                "1. Check if performance matches backtest expectations\n"
                "2. Detect any regime changes\n"
                "3. If drift detected, analyze why and recommend adjustments\n"
                "4. If severe underperformance, recommend pause\n"
                "Generate a daily optimization report."
            )

            async for msg in client.receive_response():
                # Save report and send to trader
                save_optimization_report(msg)
                await notify_trader(msg)

            # Run optimization experiments during low-volatility hours
            if is_low_volatility_period():
                await client.query(
                    "Run parameter optimization on the mean-reversion strategy. "
                    "Test lookback periods from 15-30 and entry thresholds from 1.5-2.5 stdev. "
                    "Report if any parameter set shows >10% improvement."
                )

            await asyncio.sleep(86400)  # Run once per day
```

**Example Daily Report:**
```
📊 Strategy Optimization Report - 2024-01-07

Strategy: BTC Mean Reversion
Status: ⚠️ UNDERPERFORMING

Performance Drift Analysis:
• Win Rate: 42% (Expected: 58%) → -16% drift ⚠️
• Sharpe Ratio: 0.8 (Expected: 1.85) → Significant degradation
• Avg Profit: +2.1% (Expected: +3.2%) → Lower profits per win

Root Cause Analysis:
Market regime has shifted from range-bound to trending. The current 15-day period
shows a strong uptrend (+18% move) with low retracement, which is unfavorable for
mean reversion strategies.

Backtest period (Jan-Dec 2024) had more balanced conditions with regular
oscillations around the mean.

Recommendations:
1. REDUCE POSITION SIZE: Scale down to 2% per trade (from 5%) until ranging resumes
2. ADD TREND FILTER: Only take trades when 50-day MA is flat (slope < 0.5%)
3. WIDEN ENTRY: Consider 2.5 stdev entry (from 2.0) to reduce entries during trends
4. MONITOR: If win rate doesn't improve in 7 days, pause strategy

Parameter Optimization Results:
Tested 48 parameter combinations. Best performer in current conditions:
• Lookback: 15 periods (from 20)
• Entry: 2.5 stdev (from 2.0)
• Exit: Mean + 0.5 stdev (from mean)
→ Backtested on last 30 days: +12% return, 55% win rate

Would you like me to update the strategy with these optimized parameters?
```

### Benefits
- **Autonomous**: No manual monitoring required
- **Adaptive**: Strategies evolve with market conditions
- **Proactive**: Issues detected before significant losses
- **Scientific**: All recommendations backed by quantitative analysis

---

## 8. Interactive Educational Trading Simulator

### Concept
Create a paper trading environment where Claude acts as a real-time coach, explaining market dynamics, critiquing decisions, and teaching proper trading psychology.

### Agent SDK Integration

**Educational Hooks + Custom Coach Agent:**
```python
async def educational_pretrade_hook(input_data, tool_use_id, context):
    """Before each simulated trade, quiz the user on their reasoning."""
    tool_input = input_data.get("tool_input", {})

    # Inject Socratic questions
    questions = [
        "What is your technical basis for this trade?",
        "What is your risk/reward ratio?",
        "Where will you place your stop loss?",
        "What will invalidate this trade thesis?"
    ]

    return {
        "hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "additionalContext": "Before I execute this trade, please answer:\n" +
                               "\n".join(f"• {q}" for q in questions)
        }
    }

async def educational_posttrade_hook(input_data, tool_use_id, context):
    """After each trade result, provide detailed feedback."""
    tool_response = input_data.get("tool_response", {})
    trade_result = parse_trade_result(tool_response)

    # Analyze the trade
    if trade_result.pnl > 0:
        feedback = analyze_winning_trade(trade_result)
    else:
        feedback = analyze_losing_trade(trade_result)

    return {
        "hookSpecificOutput": {
            "hookEventName": "PostToolUse",
            "additionalContext": feedback
        }
    }

# Trading coach agent
coach_agent = AgentDefinition(
    description="Educational trading coach for paper trading",
    prompt="""You are an expert trading coach. Your goal is to teach proper trading through
    simulation. For each trade:

    1. Before entry: Challenge the student's reasoning. Ask probing questions about their
       analysis, risk management, and trade plan.
    2. During the trade: Explain market dynamics as they unfold. Point out key levels,
       volume patterns, and behavioral factors.
    3. After exit: Provide constructive feedback. Praise good process even if result was
       negative. Critique poor process even if result was positive.

    Focus on teaching:
    • Risk management (position sizing, stop placement)
    • Trade planning (entry, exit, invalidation)
    • Emotional control (avoiding FOMO, revenge trading)
    • Market analysis (technical, fundamental, behavioral)

    Be supportive but honest. Good trading is about process, not results.""",
    tools=["get_market_data", "analyze_trade", "paper_trade"],
    model="sonnet"
)

options = ClaudeAgentOptions(
    agents={"coach": coach_agent},
    hooks={
        "PreToolUse": [HookMatcher(matcher="paper_trade", hooks=[educational_pretrade_hook])],
        "PostToolUse": [HookMatcher(matcher="paper_trade", hooks=[educational_posttrade_hook])]
    }
)

# Example learning session
async with ClaudeSDKClient(options=options) as client:
    await client.query(
        "Coach, I want to go long BTC here. The price looks oversold and RSI is at 28."
    )

    # Claude (Coach) will respond with questions before allowing the trade
```

**Example Coaching Session:**
```
Student: "Coach, I want to go long BTC here. Price looks oversold and RSI is at 28."

Coach: Let's think through this trade carefully before executing.

First, tell me:
1. What timeframe are you analyzing? RSI 28 on the 5-minute chart is very different
   from RSI 28 on the daily.
2. Where is price relative to key moving averages? Oversold in a downtrend can get
   more oversold.
3. What is your specific entry price, stop loss, and take profit?
4. What position size are you planning, and why?

Take your time and walk me through your complete trade plan.

---

Student: "Okay, I'm looking at the 1-hour chart. Price is at $94,500, RSI is 28.
The 50-period MA is at $95,800. I'll enter here, stop at $93,500 (-$1000),
target $97,500 (+$3000). So 3:1 R/R. Position size $5,000."

Coach: Much better! I like that you've defined your levels clearly and calculated
risk/reward. A few observations:

✅ Good: 3:1 R/R is solid
✅ Good: Using 1-hour timeframe, not just reacting to noise
✅ Good: Clear stop loss

⚠️ Consider: You're entering below the 50 MA, which means you're counter-trend.
That's not wrong, but recognize this is a counter-trend trade. Your win rate will
likely be lower, so that 3:1 R/R is necessary.

⚠️ Consider: What's below your $93,500 stop? Let me check... There's a major support
zone at $93,200. Your stop might be too tight and get hunted. Consider $93,000 instead.

📚 Teaching moment: When trading mean reversion (oversold bounces), you want to
enter at high-probability levels. Is there any confluence here? Support zone?
Fibonacci level? Previous highs/lows?

Proceed with the trade if you're comfortable, but consider widening that stop to $93,000.
```

### Benefits
- **Deliberate Practice**: Forces thoughtful trade planning
- **Immediate Feedback**: Learn from mistakes in real-time
- **Safe Environment**: No real capital at risk
- **Personalized**: Adapts to each trader's skill level and mistakes

---

## 9. Cross-Platform Intelligence Aggregator

### Concept
Create custom tools that aggregate data from Hyperliquid, other DEXs, centralized exchanges, and on-chain analytics, then use Claude to synthesize cross-platform insights.

### Agent SDK Integration

**Custom Tools for Multi-Platform Data:**
```python
@tool(
    "get_cross_platform_funding",
    "Compare funding rates across Hyperliquid, Binance, Bybit, and dYdX",
    {"coin": str}
)
async def get_cross_platform_funding(args):
    """Fetch funding rates from multiple platforms for arbitrage opportunities."""
    coin = args["coin"]

    # Fetch from multiple sources
    hyperliquid_funding = info.meta_and_asset_ctxs()  # Hyperliquid
    binance_funding = fetch_binance_funding(coin)
    bybit_funding = fetch_bybit_funding(coin)
    dydx_funding = fetch_dydx_funding(coin)

    comparison = {
        "hyperliquid": extract_funding_rate(hyperliquid_funding, coin),
        "binance": binance_funding,
        "bybit": bybit_funding,
        "dydx": dydx_funding,
        "spread": calculate_funding_spread(hyperliquid_funding, binance_funding,
                                          bybit_funding, dydx_funding),
        "arbitrage_opportunity": identify_arb_opportunity(...)
    }

    return {"content": [{"type": "text", "text": json.dumps(comparison, indent=2)}]}

@tool(
    "compare_liquidity_depth",
    "Compare order book depth and liquidity across platforms",
    {"coin": str, "depth_usd": int}
)
async def compare_liquidity_depth(args):
    """Analyze where to execute large orders for best price."""
    coin = args["coin"]
    depth = args["depth_usd"]

    # Get order books from multiple platforms
    hl_book = info.l2_snapshot(coin)
    binance_book = fetch_binance_orderbook(coin)
    coinbase_book = fetch_coinbase_orderbook(coin)

    # Calculate slippage for a given order size
    analysis = {
        "hyperliquid": {
            "slippage_bps": calculate_slippage(hl_book, depth),
            "depth_at_1pct": calculate_depth_at_distance(hl_book, 0.01),
            "spread_bps": calculate_spread(hl_book)
        },
        "binance": {
            "slippage_bps": calculate_slippage(binance_book, depth),
            "depth_at_1pct": calculate_depth_at_distance(binance_book, 0.01),
            "spread_bps": calculate_spread(binance_book)
        },
        "coinbase": {
            "slippage_bps": calculate_slippage(coinbase_book, depth),
            "depth_at_1pct": calculate_depth_at_distance(coinbase_book, 0.01),
            "spread_bps": calculate_spread(coinbase_book)
        },
        "recommendation": determine_best_venue(hl_book, binance_book, coinbase_book, depth)
    }

    return {"content": [{"type": "text", "text": json.dumps(analysis, indent=2)}]}

@tool(
    "get_whale_activity_all_platforms",
    "Track known whale wallets across multiple chains and platforms",
    {"whale_id": str}
)
async def get_whale_activity_all_platforms(args):
    """Track whale activity across CEXs and DEXs."""
    # ... implementation
    pass

# Create aggregator server
intel_server = create_sdk_mcp_server(
    name="intelligence",
    tools=[
        get_cross_platform_funding,
        compare_liquidity_depth,
        get_whale_activity_all_platforms,
        analyze_basis_trade_opportunity
    ]
)

# Usage
async with ClaudeSDKClient(options=options) as client:
    await client.query(
        "I want to enter a $500k BTC long position. Where should I execute it for "
        "the best price, and what's the funding rate landscape across platforms?"
    )

    # Claude will:
    # 1. Check liquidity depth across platforms
    # 2. Compare funding rates
    # 3. Calculate total costs (slippage + fees + funding)
    # 4. Recommend optimal execution strategy
```

### Benefits
- **Arbitrage Detection**: Identify cross-platform opportunities
- **Smart Execution**: Route orders to optimal venues
- **Complete Picture**: No blind spots from single-platform focus
- **Alpha Discovery**: Find opportunities others miss

---

## 10. Automated Compliance & Risk Monitoring System

### Concept
Use hooks to monitor all trading activity for suspicious patterns, regulatory compliance issues, and risk limit breaches, with Claude providing detailed incident reports.

### Agent SDK Integration

**Comprehensive Monitoring Hooks:**
```python
async def compliance_monitor_hook(input_data, tool_use_id, context):
    """Monitor for potential compliance issues and suspicious patterns."""
    tool_input = input_data.get("tool_input", {})
    tool_response = input_data.get("tool_response", {})

    violations = []

    # Pattern 1: Wash trading detection
    if detect_wash_trading_pattern(tool_input, recent_trades):
        violations.append({
            "severity": "high",
            "type": "wash_trading",
            "description": "Detected potential wash trading pattern: "
                         "Buy and sell orders of similar size within 60 seconds"
        })

    # Pattern 2: Layering/spoofing
    if detect_layering_pattern(tool_input, order_history):
        violations.append({
            "severity": "high",
            "type": "market_manipulation",
            "description": "Potential layering detected: Multiple orders placed and "
                         "cancelled on one side while trading the other"
        })

    # Pattern 3: Unusual activity
    if is_unusual_activity(tool_input, historical_behavior):
        violations.append({
            "severity": "medium",
            "type": "unusual_activity",
            "description": f"Trading volume is {calculate_volume_zscore()}x "
                         "above normal pattern"
        })

    # Pattern 4: Concentration limits
    if exceeds_concentration_limits(tool_input, portfolio):
        violations.append({
            "severity": "medium",
            "type": "concentration_risk",
            "description": "Position would exceed 40% portfolio concentration in crypto sector"
        })

    if violations:
        # Log for audit trail
        compliance_log.write(violations)

        # Block high-severity violations
        if any(v["severity"] == "high" for v in violations):
            return {
                "hookSpecificOutput": {
                    "hookEventName": "PreToolUse",
                    "permissionDecision": "deny",
                    "permissionDecisionReason": "Compliance violation detected"
                },
                "systemMessage": "🚨 Compliance alert: " + violations[0]["description"]
            }
        else:
            # Warn but allow medium-severity
            return {
                "systemMessage": "⚠️ Compliance warning: " + violations[0]["description"]
            }

    return {}

# Compliance agent for incident analysis
compliance_agent = AgentDefinition(
    description="Analyzes potential compliance and regulatory issues",
    prompt="""You are a compliance officer. Review trading activity for:
    • Market manipulation (spoofing, layering, wash trading)
    • Unusual activity that may indicate account compromise
    • Risk limit breaches
    • Regulatory reporting requirements

    For each incident, provide a detailed report with evidence and recommendations.""",
    tools=["review_audit_log", "analyze_trading_pattern", "generate_report"],
    model="sonnet"
)

# Daily compliance review
async def daily_compliance_review():
    """Automated daily compliance check."""
    options = ClaudeAgentOptions(
        agents={"compliance": compliance_agent}
    )

    async with ClaudeSDKClient(options=options) as client:
        await client.query(
            "Review today's trading activity for compliance issues. "
            "Generate a report highlighting any suspicious patterns or violations."
        )

        async for msg in client.receive_response():
            save_compliance_report(msg)
            if has_high_severity_issues(msg):
                await alert_compliance_team(msg)
```

### Benefits
- **Regulatory Protection**: Automated detection of prohibited activities
- **Audit Trail**: Complete record of all decisions and actions
- **Risk Management**: Prevents concentration and leverage excesses
- **Early Warning**: Detect account compromise or unusual behavior

---

## 11. Liquidation Predictor & Protection System

### Concept
Build an AI system that continuously monitors liquidation risks across your positions and the broader market, providing early warnings and automated protection.

### Agent SDK Integration

**Custom Tools + Monitoring Agent:**
```python
@tool(
    "analyze_liquidation_landscape",
    "Map liquidation clusters across the market and estimate cascade risk",
    {"coin": str, "price_range_pct": float}
)
async def analyze_liquidation_landscape(args):
    """Build comprehensive liquidation risk map."""
    coin = args["coin"]
    current_price = get_current_price(coin)

    # Sample many addresses to estimate liquidation distribution
    # (In production, you'd maintain a database of tracked positions)
    tracked_addresses = get_tracked_addresses()
    liquidation_map = []

    for addr in tracked_addresses:
        user_state = info.user_state(addr)
        for position in user_state.get("assetPositions", []):
            if position["position"]["coin"] == coin:
                liq_price = calculate_liquidation_price(position)
                if liq_price:
                    liquidation_map.append({
                        "price": liq_price,
                        "size_usd": position["position"]["szi"] * current_price,
                        "side": "long" if float(position["position"]["szi"]) > 0 else "short"
                    })

    # Analyze clusters
    clusters = identify_liquidation_clusters(liquidation_map, current_price)

    # Get order book depth to assess absorption capacity
    orderbook = info.l2_snapshot(coin)

    risk_analysis = {
        "current_price": current_price,
        "liquidation_clusters": clusters,
        "most_at_risk": identify_most_at_risk_cluster(clusters, current_price),
        "cascade_probability": calculate_cascade_probability(clusters, orderbook),
        "recommendation": generate_liquidation_recommendation(clusters, orderbook)
    }

    return {"content": [{"type": "text", "text": json.dumps(risk_analysis, indent=2)}]}

@tool(
    "check_my_liquidation_risk",
    "Analyze your positions for liquidation risk with early warning",
    {"address": str}
)
async def check_my_liquidation_risk(args):
    """Comprehensive personal liquidation risk analysis."""
    user_state = info.user_state(args["address"])

    risk_report = {
        "positions": [],
        "overall_risk_score": 0,
        "immediate_actions": []
    }

    for position in user_state.get("assetPositions", []):
        liq_price = calculate_liquidation_price(position)
        current_price = get_current_price(position["position"]["coin"])
        distance_to_liq = ((liq_price - current_price) / current_price) * 100

        pos_risk = {
            "coin": position["position"]["coin"],
            "size_usd": abs(float(position["position"]["szi"])) * current_price,
            "leverage": position["position"]["leverage"],
            "liquidation_price": liq_price,
            "distance_to_liquidation_pct": distance_to_liq,
            "risk_level": classify_risk_level(distance_to_liq)
        }

        risk_report["positions"].append(pos_risk)

        # Generate warnings for at-risk positions
        if abs(distance_to_liq) < 10:  # Within 10% of liquidation
            risk_report["immediate_actions"].append(
                f"URGENT: {pos_risk['coin']} position is {abs(distance_to_liq):.1f}% "
                f"from liquidation. Consider reducing leverage or adding margin."
            )

    return {"content": [{"type": "text", "text": json.dumps(risk_report, indent=2)}]}

# Liquidation protection agent
liquidation_agent = AgentDefinition(
    description="Monitors and protects against liquidation risks",
    prompt="""You are a liquidation risk specialist. Your mission is to prevent
    liquidations through early warning and proactive risk management.

    Continuously:
    1. Monitor all positions for liquidation proximity
    2. Track market liquidation clusters that could cause cascades
    3. Alert when positions enter danger zone (< 15% from liquidation)
    4. Recommend protective actions (reduce leverage, add margin, close positions)
    5. During high-volatility events, provide real-time updates

    Be aggressive in your warnings. A liquidation is the worst outcome.""",
    tools=[
        "analyze_liquidation_landscape",
        "check_my_liquidation_risk",
        "get_volatility_forecast"
    ],
    model="sonnet"
)

# Continuous monitoring loop
async def liquidation_monitoring_loop():
    """Background process that monitors liquidation risk 24/7."""
    options = ClaudeAgentOptions(
        agents={"liquidation_guard": liquidation_agent}
    )

    async with ClaudeSDKClient(options=options) as client:
        while True:
            # Check every 60 seconds
            await client.query(
                "Check my liquidation risk across all positions. "
                "If any position is within 15% of liquidation, send URGENT alert. "
                "Also check the broader market for liquidation clusters that could "
                "trigger cascades affecting my positions."
            )

            async for msg in client.receive_response():
                if contains_urgent_warning(msg):
                    await send_push_notification(msg)
                    await send_telegram_alert(msg)
                    await send_email(msg)

            await asyncio.sleep(60)
```

### Benefits
- **Proactive Protection**: Warnings before disaster strikes
- **Market Awareness**: Understand cascade risks beyond your positions
- **Peace of Mind**: 24/7 monitoring even while sleeping
- **Automated Response**: Can be configured to auto-reduce risk

---

## 12. Strategy Documentation & Knowledge Base Generator

### Concept
As traders develop and refine strategies, use hooks and agents to automatically capture decisions, generate comprehensive documentation, and build an institutional knowledge base.

### Agent SDK Integration

**Documentation Hooks + Knowledge Agent:**
```python
async def strategy_documentation_hook(input_data, tool_use_id, context):
    """Automatically document strategy decisions and refinements."""
    user_prompt = context.get("user_prompt", "")

    # Detect strategy-related queries
    strategy_keywords = ["strategy", "system", "rules", "conditions", "setup"]
    if not any(kw in user_prompt.lower() for kw in strategy_keywords):
        return {}

    # Extract and structure the information
    strategy_info = {
        "timestamp": datetime.now().isoformat(),
        "type": classify_query_type(user_prompt),  # "new_strategy", "modification", "analysis"
        "content": user_prompt,
        "context": extract_relevant_context(context)
    }

    # Store in knowledge base
    knowledge_base.add_entry(strategy_info)

    return {
        "hookSpecificOutput": {
            "hookEventName": "UserPromptSubmit",
            "additionalContext": "I'm documenting this in your strategy knowledge base."
        }
    }

# Knowledge base agent
knowledge_agent = AgentDefinition(
    description="Manages and synthesizes trading strategy documentation",
    prompt="""You are a strategy documentation expert. Your role is to:

    1. Capture all strategy discussions and decisions
    2. Generate comprehensive strategy documentation with:
       • Clear entry/exit rules
       • Risk management parameters
       • Backtest results and performance metrics
       • Market conditions where strategy works best
       • Known failure modes and edge cases
    3. Maintain version history of strategy evolution
    4. Create searchable knowledge base for future reference
    5. Generate strategy comparison reports

    Write documentation that could be handed to another trader and they could
    execute the strategy perfectly without additional explanation.""",
    tools=[
        "add_to_knowledge_base",
        "query_knowledge_base",
        "generate_strategy_doc",
        "compare_strategies"
    ],
    model="sonnet"
)

# Example usage
async with ClaudeSDKClient(options=options) as client:
    # Developing a new strategy
    await client.query(
        "I've been testing a new strategy: When BTC funding rate goes above 0.1% "
        "and open interest is at all-time high, short with 2x leverage. "
        "Exit when funding drops below 0.05% or price moves 3% against me. "
        "Backtest showed 68% win rate with 2.3 Sharpe."
    )

    # Later, query the knowledge base
    await client.query(
        "What strategies do I have for high-funding-rate scenarios? "
        "Generate a comparison report."
    )

    # Agent will:
    # 1. Search knowledge base
    # 2. Find all relevant strategies
    # 3. Generate comparison with pros/cons
    # 4. Recommend best strategy for current conditions
```

**Example Generated Documentation:**
```markdown
# Strategy: Funding Rate Mean Reversion Short

## Version History
- v1.0 (2024-01-03): Initial strategy
- v1.1 (2024-01-07): Added OI filter after false signals in low-OI conditions

## Strategy Overview
Contrarian short strategy that fades extreme positive funding rates, betting on
mean reversion and potential long squeeze.

## Entry Conditions
1. Funding rate > 0.1% (annualized > 109%)
2. Open interest at 30-day high (indicates crowded long positioning)
3. RSI(14) > 65 (confirms overbought on shorter timeframe)
4. No entry if price is breaking ATH (avoid shorting strong trends)

## Position Sizing
- Base size: 5% of portfolio
- Leverage: 2x
- Maximum portfolio exposure to this strategy: 15% (max 3 concurrent positions)

## Exit Conditions
**Take Profit:**
- Funding rate drops below 0.05% (mean reversion complete)
- RSI drops below 45 (momentum shift)

**Stop Loss:**
- Price moves 3% against entry (risk management)
- Funding rate increases above 0.15% (crowding intensifying, abort)

## Risk Management
- Maximum loss per trade: 3% (with 5% position and 2x leverage, 3% price move = 3% loss)
- Risk/Reward: Targeting 1:2 minimum
- No more than 2 concurrent positions in same underlying asset

## Backtest Results (Jan-Dec 2024)
- Total trades: 47
- Win rate: 68%
- Average win: +4.2%
- Average loss: -2.8%
- Sharpe ratio: 2.3
- Max drawdown: -6.5%
- Best month: August (+15.2%)
- Worst month: October (-3.1%)

## Optimal Market Conditions
✅ Works best in:
- Range-bound markets with regular oscillations
- High-volatility environments (creates funding extremes)
- Bull markets with periodic cooling-off periods

❌ Struggles in:
- Sustained parabolic rallies (funding stays high, shorts get rekt)
- Very low volatility (funding rarely reaches trigger levels)

## Known Edge Cases & Failure Modes
1. **Parabolic rallies**: During 2024 Sep rally, funding stayed above 0.15% for 8 days.
   Strategy had 4 consecutive losses. → Added "no entry at ATH" filter in v1.1

2. **Low OI false signals**: Some alts had high funding but low OI, resulting in poor fills.
   → Added "30-day OI high" filter in v1.1

3. **Weekend gaps**: Sunday CME close sometimes creates brief funding spikes that quickly reverse.
   → Consider time-based filter to avoid Sunday 4PM ET entries

## Performance by Market Regime
| Regime | Win Rate | Avg Return | Notes |
|--------|----------|------------|-------|
| Bull Market Ranging | 75% | +5.1% | Ideal conditions |
| Bear Market | 62% | +3.8% | Works but fewer setups |
| Strong Bull Trend | 45% | -1.2% | Avoid or reduce size |
| High Volatility | 71% | +6.2% | Many mean reversion opportunities |

## Related Strategies
- See "Funding Rate Long" for opposite setup
- See "Liquidation Cascade Short" for complementary high-risk short strategy

## Maintenance Notes
- Review parameter effectiveness quarterly
- If win rate drops below 55% for 20+ trades, investigate regime change
- Consider seasonal patterns (Q4 historically best performance)
```

### Benefits
- **Institutional Memory**: Never forget what worked and why
- **Knowledge Transfer**: Easy onboarding for team members or future you
- **Continuous Improvement**: Track strategy evolution over time
- **Risk Management**: Document failure modes before you hit them

---

## Implementation Roadmap

### Phase 1: Foundation (Weeks 1-2)
1. Set up Claude Agent SDK development environment
2. Create custom tools for core Hyperliquid SDK methods
3. Implement basic hooks for trade journaling
4. Build simple conversational interface for data queries

### Phase 2: Intelligence Layer (Weeks 3-4)
5. Deploy multi-agent system for market analysis
6. Implement risk management tools and validators
7. Create backtesting interface
8. Build alerting system with contextual intelligence

### Phase 3: Automation (Weeks 5-6)
9. Implement adaptive strategy optimizer
10. Deploy compliance monitoring system
11. Create liquidation protection system
12. Build knowledge base and documentation generator

### Phase 4: Integration & Testing (Weeks 7-8)
13. Integrate all systems
14. Comprehensive testing with paper trading
15. Performance optimization
16. Documentation and deployment

---

## Technical Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Claude Agent SDK Layer                       │
├─────────────────────────────────────────────────────────────────┤
│  • ClaudeSDKClient (streaming)                                  │
│  • Custom Agents (whale-analyst, technical-analyst, etc.)       │
│  • Hooks (PreToolUse, PostToolUse, UserPromptSubmit)           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Custom SDK MCP Servers Layer                    │
├─────────────────────────────────────────────────────────────────┤
│  • Hyperliquid Tools (user_state, fills, funding, etc.)        │
│  • Risk Management Tools (VaR, Kelly, correlation)             │
│  • Backtesting Tools (backtest_strategy, optimize)             │
│  • Analytics Tools (liquidation_map, sentiment, etc.)          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Hyperliquid Python SDK Layer                   │
├─────────────────────────────────────────────────────────────────┤
│  • Info API (market data, user data)                           │
│  • Exchange API (order execution)                              │
│  • WebSocket subscriptions                                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Hyperliquid DEX                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Advantages of Agent SDK Integration

### 1. **Natural Language Interface**
Transform complex trading operations into conversational interactions. Instead of writing scripts, simply ask Claude.

### 2. **Autonomous Intelligence**
Agents can operate independently, monitoring markets 24/7 and making intelligent decisions within defined parameters.

### 3. **Explainable AI**
Every decision Claude makes includes reasoning, making it educational and auditable.

### 4. **Rapid Prototyping**
Test new strategy ideas in minutes through conversation rather than hours of coding.

### 5. **Adaptive Learning**
Agents can adjust to changing market conditions and provide context-aware recommendations.

### 6. **Risk Management Automation**
Hooks ensure risk rules are enforced consistently without emotional override.

### 7. **Institutional-Grade Documentation**
Automatically capture and organize trading knowledge for future reference.

---

## Conclusion

The combination of Hyperliquid's rich data access and the Claude Agent SDK's intelligent automation creates unprecedented opportunities for individual traders to operate with institutional-grade capabilities.

These 12 ideas represent just the beginning. The true power emerges when you combine multiple approaches—imagine a system where:
- Multi-agent analysts provide market intelligence
- Hook-based validators enforce discipline
- Adaptive optimizers keep strategies current
- Educational systems improve your skills
- Documentation preserves institutional knowledge

All working together, 24/7, in natural language.

The future of trading is conversational, intelligent, and autonomous. Start building it today.

---

## Next Steps

1. **Explore the SDK**: Clone [claude-agent-sdk-python](https://github.com/anthropics/claude-agent-sdk-python) and run the examples
2. **Start Simple**: Begin with idea #2 (Conversational Risk Management) or #3 (Trade Journaling)
3. **Build Custom Tools**: Wrap Hyperliquid SDK methods as custom tools
4. **Deploy Hooks**: Add validation hooks to enforce risk management
5. **Create Agents**: Design specialized agents for specific analysis tasks
6. **Integrate**: Combine multiple ideas into a cohesive trading system
7. **Iterate**: Continuously refine based on what works

The tools are ready. The data is available. Time to build the future of intelligent trading systems.

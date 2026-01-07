# Hyperliquid + Claude Agent SDK: Creative Integration Ideas

This document brainstorms creative ways to leverage the [Claude Agent SDK](https://github.com/anthropics/claude-agent-sdk-python) to enhance the [Hyperliquid trader analysis use cases](./hyperliquid-trader-analysis-usecases.md).

---

## Table of Contents

1. [Intelligent Whale Narrative Generator](#1-intelligent-whale-narrative-generator)
2. [Conversational Trade Advisor with Risk Guardrails](#2-conversational-trade-advisor-with-risk-guardrails)
3. [Multi-Agent Trading Research Team](#3-multi-agent-trading-research-team)
4. [Real-Time Market Commentary Bot](#4-real-time-market-commentary-bot)
5. [Anomaly Detection & Explanation System](#5-anomaly-detection--explanation-system)
6. [Interactive Backtest Assistant](#6-interactive-backtest-assistant)
7. [Smart Money Signal Validator](#7-smart-money-signal-validator)
8. [Portfolio Risk Analyst Agent](#8-portfolio-risk-analyst-agent)
9. [Liquidation Scenario Simulator](#9-liquidation-scenario-simulator)
10. [Trader DNA Interview System](#10-trader-dna-interview-system)
11. [Funding Rate Arbitrage Strategist](#11-funding-rate-arbitrage-strategist)
12. [Market Regime Classifier with Adaptive Strategies](#12-market-regime-classifier-with-adaptive-strategies)

---

## 1. Intelligent Whale Narrative Generator

### Concept

Transform raw whale tracking data into natural language narratives that explain the "story" behind whale movements, helping traders understand context beyond just numbers.

### SDK Features Leveraged

- **Custom MCP Tools**: Expose Hyperliquid SDK methods as tools Claude can call
- **Streaming**: Real-time narrative updates as whale positions change
- **Custom System Prompt**: Specialized prompt for financial analysis

### Implementation

```python
from claude_agent_sdk import tool, create_sdk_mcp_server, ClaudeAgentOptions, ClaudeSDKClient
from hyperliquid.info import Info

@tool("get_whale_positions", "Get current positions for a whale address", {"address": str})
async def get_whale_positions(args):
    info = Info(constants.MAINNET_API_URL)
    state = info.user_state(args["address"])
    return {"content": [{"type": "text", "text": json.dumps(state, indent=2)}]}

@tool("get_whale_fills", "Get recent trades for a whale", {"address": str, "limit": int})
async def get_whale_fills(args):
    info = Info(constants.MAINNET_API_URL)
    fills = info.user_fills(args["address"])[:args.get("limit", 50)]
    return {"content": [{"type": "text", "text": json.dumps(fills, indent=2)}]}

@tool("get_market_context", "Get current market open interest and funding", {"coin": str})
async def get_market_context(args):
    info = Info(constants.MAINNET_API_URL)
    meta = info.meta_and_asset_ctxs()
    return {"content": [{"type": "text", "text": json.dumps(meta, indent=2)}]}

# Create intelligent whale analyst
whale_server = create_sdk_mcp_server(
    name="whale-tracker",
    tools=[get_whale_positions, get_whale_fills, get_market_context]
)

options = ClaudeAgentOptions(
    system_prompt="""You are a whale watching analyst for Hyperliquid. When given whale addresses:
    1. Analyze their current positions and recent activity
    2. Compare their moves to overall market conditions
    3. Generate a narrative explaining what the whale might be doing and why
    4. Highlight any unusual patterns or potential signals
    Write in a concise, professional trading commentary style.""",
    mcp_servers={"whale": whale_server},
    allowed_tools=["mcp__whale__get_whale_positions", "mcp__whale__get_whale_fills", "mcp__whale__get_market_context"]
)
```

### Value Proposition

Instead of just alerting "Whale X bought 100 BTC," the system generates: "Whale 0xABC has accumulated a 500 BTC long position over 6 hours using scaled limit orders, suggesting a patient accumulation strategy. This coincides with funding rates turning negative (-0.02%), indicating the whale may be positioning for a funding yield play while maintaining directional exposure."

---

## 2. Conversational Trade Advisor with Risk Guardrails

### Concept

An interactive trading assistant that can answer questions about positions, suggest trades, but has built-in hooks to prevent dangerous recommendations.

### SDK Features Leveraged

- **Hooks**: PreToolUse hooks to validate recommendations before delivery
- **Permission Callbacks**: Block recommendations that exceed risk parameters
- **Custom Agents**: Specialized agents for different advisory functions

### Implementation

```python
from claude_agent_sdk import ClaudeAgentOptions, ClaudeSDKClient, HookMatcher

# Risk guardrail hook
async def validate_trade_recommendation(input_data, tool_use_id, context):
    """Block recommendations that exceed risk parameters."""
    tool_input = input_data.get("tool_input", {})

    # Extract recommendation details
    leverage = tool_input.get("leverage", 1)
    position_pct = tool_input.get("position_percent", 0)

    # Risk rules
    if leverage > 10:
        return {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny",
                "permissionDecisionReason": "Leverage exceeds maximum safe threshold of 10x"
            }
        }

    if position_pct > 25:
        return {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny",
                "permissionDecisionReason": "Position size exceeds 25% portfolio limit"
            }
        }

    return {}

# Post-recommendation context injection
async def add_risk_disclosure(input_data, tool_use_id, context):
    """Add risk warnings to all trade recommendations."""
    return {
        "hookSpecificOutput": {
            "hookEventName": "PostToolUse",
            "additionalContext": "RISK DISCLOSURE: This is not financial advice. Always use stop losses and never risk more than you can afford to lose."
        }
    }

options = ClaudeAgentOptions(
    hooks={
        "PreToolUse": [
            HookMatcher(matcher="recommend_trade", hooks=[validate_trade_recommendation]),
        ],
        "PostToolUse": [
            HookMatcher(matcher="recommend_trade", hooks=[add_risk_disclosure]),
        ]
    }
)
```

### Value Proposition

Users can have natural conversations like "Should I go long here?" while the system automatically enforces risk parameters and prevents the AI from suggesting overleveraged or oversized positions.

---

## 3. Multi-Agent Trading Research Team

### Concept

Deploy a team of specialized Claude agents that work together to provide comprehensive trading analysis, each with their own expertise and tools.

### SDK Features Leveraged

- **AgentDefinition**: Define multiple specialized agents
- **Model Selection**: Use different models for different tasks (haiku for quick analysis, opus for deep research)
- **Tool Restrictions**: Each agent only has access to relevant tools

### Implementation

```python
from claude_agent_sdk import AgentDefinition, ClaudeAgentOptions, query

options = ClaudeAgentOptions(
    agents={
        "market-analyst": AgentDefinition(
            description="Analyzes market structure, order book depth, and price action",
            prompt="""You are a market microstructure analyst. Focus on:
            - Order book imbalances and depth analysis
            - Price action patterns and support/resistance levels
            - Volume profile and market participation
            Be quantitative and precise.""",
            tools=["mcp__hl__l2_snapshot", "mcp__hl__candles", "mcp__hl__all_mids"],
            model="sonnet"
        ),

        "sentiment-analyst": AgentDefinition(
            description="Analyzes trader positioning and sentiment indicators",
            prompt="""You are a sentiment and positioning analyst. Focus on:
            - Open interest changes and funding rate analysis
            - Long/short ratio estimation from funding
            - Position crowding and potential squeeze setups
            Be contrarian-aware.""",
            tools=["mcp__hl__meta_and_asset_ctxs", "mcp__hl__funding_history"],
            model="haiku"  # Fast model for quick sentiment reads
        ),

        "whale-tracker": AgentDefinition(
            description="Tracks and analyzes whale trader activity",
            prompt="""You are a whale activity analyst. Focus on:
            - Tracking large trader positions and changes
            - Identifying accumulation/distribution patterns
            - Correlating whale moves with price action
            Be alert to unusual activity.""",
            tools=["mcp__hl__user_state", "mcp__hl__user_fills"],
            model="sonnet"
        ),

        "risk-manager": AgentDefinition(
            description="Analyzes liquidation risks and portfolio exposure",
            prompt="""You are a risk management specialist. Focus on:
            - Liquidation cascade potential at various price levels
            - Portfolio correlation and concentration risks
            - Funding cost projections for positions
            Be conservative and highlight risks.""",
            tools=["mcp__hl__user_state", "mcp__hl__meta_and_asset_ctxs", "mcp__hl__funding_history"],
            model="opus"  # Best model for critical risk assessment
        ),

        "strategy-synthesizer": AgentDefinition(
            description="Synthesizes insights from other agents into actionable strategies",
            prompt="""You are a trading strategist. When given analysis from other team members:
            - Synthesize findings into a coherent market view
            - Identify convergent and divergent signals
            - Propose specific trade ideas with entry, exit, and sizing
            Be decisive but acknowledge uncertainty.""",
            tools=[],  # No direct data access, works with agent outputs
            model="opus"
        )
    }
)

# Usage: Ask the research team to analyze a trade
async for msg in query(
    prompt="""Analyze BTC for a potential trade:
    1. Use market-analyst to check market structure
    2. Use sentiment-analyst to assess positioning
    3. Use whale-tracker to check recent whale activity
    4. Use risk-manager to identify key risks
    5. Use strategy-synthesizer to combine insights into a recommendation""",
    options=options
):
    print(msg)
```

### Value Proposition

Mimics a professional trading desk with specialized roles, providing institutional-quality research through AI collaboration.

---

## 4. Real-Time Market Commentary Bot

### Concept

A streaming market commentary system that provides live analysis as market conditions change, similar to a sports broadcaster for trading.

### SDK Features Leveraged

- **Streaming Mode**: Real-time partial message updates
- **include_partial_messages**: See Claude's thinking as it happens
- **WebSocket Integration**: React to live market data

### Implementation

```python
from claude_agent_sdk import ClaudeAgentOptions, ClaudeSDKClient

async def run_live_commentary():
    options = ClaudeAgentOptions(
        system_prompt="""You are a live market commentator for Hyperliquid.
        When receiving market updates, provide instant commentary like a sports broadcaster:
        - React to significant price moves
        - Highlight unusual volume or order flow
        - Note when technical levels are tested
        - Call out potential trade setups
        Keep commentary concise and engaging.""",
        include_partial_messages=True,  # Stream thoughts in real-time
        mcp_servers={"market": market_data_server}
    )

    async with ClaudeSDKClient(options=options) as client:
        # Subscribe to market events
        while True:
            event = await get_next_market_event()  # Your WebSocket handler

            await client.query(f"Market update: {json.dumps(event)}")

            async for msg in client.receive_response():
                if isinstance(msg, StreamEvent):
                    # Print partial commentary as it's generated
                    print(msg.event.get("delta", {}).get("text", ""), end="", flush=True)
```

### Value Proposition

Traders get real-time AI-powered market color, helping them stay engaged and informed without constantly watching charts.

---

## 5. Anomaly Detection & Explanation System

### Concept

Detect unusual trading patterns and have Claude explain what's anomalous and why it might matter.

### SDK Features Leveraged

- **Custom Tools**: Anomaly detection algorithms exposed as tools
- **PostToolUse Hooks**: Automatically trigger explanations after detection
- **Structured Output**: Return anomalies in parseable format

### Implementation

```python
@tool("detect_volume_anomaly", "Detect unusual volume patterns", {"coin": str, "lookback_hours": int})
async def detect_volume_anomaly(args):
    # Calculate volume z-scores
    volumes = get_historical_volumes(args["coin"], args["lookback_hours"])
    current = volumes[-1]
    mean, std = np.mean(volumes[:-1]), np.std(volumes[:-1])
    zscore = (current - mean) / std

    return {
        "content": [{
            "type": "text",
            "text": json.dumps({
                "coin": args["coin"],
                "current_volume": current,
                "average_volume": mean,
                "zscore": zscore,
                "is_anomaly": abs(zscore) > 2
            })
        }]
    }

@tool("detect_whale_anomaly", "Detect unusual whale behavior", {"addresses": list})
async def detect_whale_anomaly(args):
    anomalies = []
    for addr in args["addresses"]:
        # Check for unusual position changes, leverage shifts, etc.
        state = info.user_state(addr)
        recent_fills = info.user_fills(addr)[-20:]

        # Anomaly detection logic
        if position_change_is_unusual(state, recent_fills):
            anomalies.append({
                "address": addr,
                "type": "position_spike",
                "details": analyze_position_change(state, recent_fills)
            })

    return {"content": [{"type": "text", "text": json.dumps(anomalies)}]}

# Hook to auto-explain anomalies
async def explain_anomaly(input_data, tool_use_id, context):
    if "is_anomaly" in str(input_data.get("tool_response", {})):
        return {
            "hookSpecificOutput": {
                "hookEventName": "PostToolUse",
                "additionalContext": "An anomaly was detected. Please explain: 1) What makes this unusual, 2) Potential causes, 3) Trading implications"
            }
        }
    return {}

options = ClaudeAgentOptions(
    mcp_servers={"anomaly": anomaly_server},
    hooks={
        "PostToolUse": [
            HookMatcher(matcher="detect_.*_anomaly", hooks=[explain_anomaly])
        ]
    }
)
```

### Value Proposition

Instead of just flagging "anomaly detected," the system provides natural language explanations: "BTC volume is 4.2 standard deviations above normal. This spike coincides with a break above the $65k level and unusually high whale activity. Historically, such volume spikes at key levels have preceded 3-5% continuation moves 72% of the time."

---

## 6. Interactive Backtest Assistant

### Concept

A conversational backtesting system where traders can describe strategies in plain English and iterate on results through dialogue.

### SDK Features Leveraged

- **Conversation Continuity**: `continue_conversation` for iterative refinement
- **Session Resume**: `resume` to pick up previous backtest sessions
- **File Checkpointing**: Track strategy file versions

### Implementation

```python
from claude_agent_sdk import ClaudeAgentOptions, ClaudeSDKClient

async def backtest_assistant():
    options = ClaudeAgentOptions(
        system_prompt="""You are a trading strategy backtesting assistant. Help users:
        1. Translate their strategy ideas into testable rules
        2. Run backtests using the provided tools
        3. Analyze results and suggest improvements
        4. Iterate on strategy parameters

        Always show key metrics: Sharpe, max drawdown, win rate, profit factor.""",
        mcp_servers={"backtest": backtest_server},
        enable_file_checkpointing=True,  # Track strategy versions
        continue_conversation=True
    )

    async with ClaudeSDKClient(options=options) as client:
        # Initial strategy description
        await client.query("""
        I want to test a strategy that:
        - Goes long when funding is below -0.01% and OI is rising
        - Goes short when funding is above 0.05% and price is at resistance
        - Uses 5x leverage
        - Holds for 4-8 hours typically
        """)

        async for msg in client.receive_response():
            print(msg)

        # Iterate on results
        await client.query("The drawdown is too high. Can we add a stop loss at 2%?")
        async for msg in client.receive_response():
            print(msg)

        # Compare versions
        await client.query("Compare this version to the original. Which is better?")
        async for msg in client.receive_response():
            print(msg)
```

### Value Proposition

Traders can develop strategies through conversation rather than coding, with the AI handling translation to quantitative rules and managing the iteration cycle.

---

## 7. Smart Money Signal Validator

### Concept

Before acting on smart money copy signals, have Claude validate them against multiple criteria and provide a confidence score.

### SDK Features Leveraged

- **Tool Permission Callbacks**: Validate signals before they're acted upon
- **Multi-step Reasoning**: Chain multiple validation checks
- **Structured Output**: Return validation results in consistent format

### Implementation

```python
from claude_agent_sdk import ClaudeAgentOptions, PermissionResultAllow, PermissionResultDeny

async def validate_copy_signal(tool_name: str, tool_input: dict, context) -> PermissionResult:
    """Validate copy trading signals before execution."""

    if tool_name != "execute_copy_trade":
        return PermissionResultAllow()

    signal = tool_input.get("signal", {})

    # Run validation checks
    checks = {
        "trader_reputation": await check_trader_history(signal["trader_address"]),
        "market_alignment": await check_market_conditions(signal["coin"]),
        "risk_parameters": await check_risk_limits(signal),
        "timing": await check_entry_timing(signal),
        "crowding": await check_position_crowding(signal["coin"])
    }

    # Calculate composite score
    score = sum(checks.values()) / len(checks)

    if score < 0.6:
        return PermissionResultDeny(
            message=f"Signal validation failed (score: {score:.2f}). Issues: {[k for k,v in checks.items() if v < 0.5]}",
            interrupt=False
        )

    # Allow with modified input (add validation metadata)
    return PermissionResultAllow(
        updated_input={
            **tool_input,
            "validation_score": score,
            "validation_checks": checks
        }
    )

options = ClaudeAgentOptions(
    can_use_tool=validate_copy_signal,
    mcp_servers={"trading": trading_server}
)
```

### Value Proposition

Prevents blind copy trading by adding an AI validation layer that considers market context, trader reliability, and risk parameters before allowing signal execution.

---

## 8. Portfolio Risk Analyst Agent

### Concept

A dedicated agent that continuously monitors portfolio risk and provides natural language alerts and recommendations.

### SDK Features Leveraged

- **Custom Agents**: Risk-focused agent with specific prompt
- **Hooks**: Stop hooks to trigger risk alerts
- **Budget Control**: Limit analysis costs with `max_budget_usd`

### Implementation

```python
from claude_agent_sdk import AgentDefinition, ClaudeAgentOptions

risk_agent = AgentDefinition(
    description="Monitors portfolio risk and provides alerts",
    prompt="""You are a portfolio risk analyst. Your responsibilities:

    1. POSITION RISK: Monitor individual position sizes and leverage
    2. CORRELATION RISK: Identify correlated positions that increase exposure
    3. LIQUIDATION RISK: Track liquidation prices and buffer distances
    4. FUNDING RISK: Calculate projected funding costs
    5. CONCENTRATION RISK: Alert on overexposure to single assets

    For each risk category, provide:
    - Current status (Green/Yellow/Red)
    - Specific metrics
    - Recommended actions if needed

    Be proactive about risks but avoid excessive false alarms.""",
    tools=["mcp__hl__user_state", "mcp__hl__user_funding_history", "mcp__hl__meta_and_asset_ctxs"],
    model="sonnet"
)

# Periodic risk check
async def run_risk_monitor(portfolio_address: str):
    options = ClaudeAgentOptions(
        agents={"risk-analyst": risk_agent},
        max_budget_usd=0.10  # Limit cost per check
    )

    while True:
        async for msg in query(
            prompt=f"Use risk-analyst to analyze the portfolio at {portfolio_address} and report any concerns",
            options=options
        ):
            if is_risk_alert(msg):
                await send_notification(msg)

        await asyncio.sleep(300)  # Check every 5 minutes
```

### Value Proposition

Continuous, intelligent risk monitoring that understands context and provides actionable recommendations rather than just threshold alerts.

---

## 9. Liquidation Scenario Simulator

### Concept

An interactive simulator where traders can ask "what if" questions about price scenarios and get detailed liquidation cascade analysis.

### SDK Features Leveraged

- **Conversation Mode**: Interactive Q&A about scenarios
- **Fork Sessions**: Branch conversations to explore different scenarios
- **Custom Tools**: Liquidation calculation tools

### Implementation

```python
from claude_agent_sdk import ClaudeAgentOptions, ClaudeSDKClient

@tool("simulate_price_move", "Simulate liquidations at a given price", {
    "coin": str,
    "target_price": float,
    "current_price": float
})
async def simulate_price_move(args):
    # Calculate which positions would be liquidated
    liquidations = calculate_liquidations_at_price(
        args["coin"],
        args["target_price"]
    )

    # Estimate second-order effects
    cascade = estimate_cascade_effect(liquidations, args["coin"])

    return {
        "content": [{
            "type": "text",
            "text": json.dumps({
                "target_price": args["target_price"],
                "direct_liquidations_usd": liquidations["total_value"],
                "estimated_price_impact": cascade["price_impact"],
                "cascade_liquidations_usd": cascade["additional_liquidations"],
                "final_estimated_price": cascade["estimated_final_price"],
                "high_risk_levels": liquidations["concentrated_levels"]
            })
        }]
    }

async def liquidation_simulator():
    options = ClaudeAgentOptions(
        system_prompt="""You are a liquidation cascade simulator. Help traders understand:
        - What happens if price moves to X
        - Where the largest liquidation clusters are
        - Potential cascade effects
        - Safe zones vs danger zones

        Explain the mechanics clearly and highlight key price levels.""",
        mcp_servers={"sim": simulation_server},
        fork_session=True  # Each scenario branches the conversation
    )

    async with ClaudeSDKClient(options=options) as client:
        # Explore scenarios
        await client.query("What happens to ETH if it drops to $3000?")
        async for msg in client.receive_response():
            print(msg)

        # Follow-up in same context
        await client.query("What if it drops another 10% from there?")
        async for msg in client.receive_response():
            print(msg)
```

### Value Proposition

Traders can explore "what if" scenarios conversationally and understand the full chain of potential market events, not just their own liquidation prices.

---

## 10. Trader DNA Interview System

### Concept

An AI system that "interviews" trading data to build comprehensive trader profiles, answering questions like "What kind of trader is this?"

### SDK Features Leveraged

- **Multi-turn Conversations**: Deep-dive analysis through dialogue
- **Custom System Prompt**: Behavioral analysis expertise
- **Structured Output**: Consistent profile format

### Implementation

```python
from claude_agent_sdk import ClaudeAgentOptions, query

async def profile_trader(address: str):
    options = ClaudeAgentOptions(
        system_prompt="""You are a trader behavior psychologist. Analyze trading data to:

        1. CLASSIFY the trader archetype (scalper, swing trader, market maker, etc.)
        2. IDENTIFY their edge (timing, sizing, asset selection, etc.)
        3. ASSESS their risk management style
        4. PREDICT their behavior in different market conditions
        5. COMPARE to successful traders with similar patterns

        Build a comprehensive profile as if you're writing a scouting report.""",
        mcp_servers={"analysis": trader_analysis_server},
        output_format={
            "type": "json_schema",
            "schema": {
                "type": "object",
                "properties": {
                    "archetype": {"type": "string"},
                    "confidence": {"type": "number"},
                    "strengths": {"type": "array", "items": {"type": "string"}},
                    "weaknesses": {"type": "array", "items": {"type": "string"}},
                    "edge_description": {"type": "string"},
                    "risk_profile": {"type": "string"},
                    "market_conditions_suited_for": {"type": "array", "items": {"type": "string"}},
                    "similar_successful_traders": {"type": "array", "items": {"type": "string"}},
                    "recommendations": {"type": "array", "items": {"type": "string"}}
                }
            }
        }
    )

    async for msg in query(
        prompt=f"""Conduct a comprehensive analysis of trader {address}:

        1. First, gather their trading history, positions, and patterns
        2. Analyze their timing, sizing, and asset preferences
        3. Evaluate their performance across different market conditions
        4. Build a complete trader DNA profile

        Be thorough - this profile will be used for copy trading decisions.""",
        options=options
    ):
        if isinstance(msg, ResultMessage) and msg.structured_output:
            return msg.structured_output
```

### Value Proposition

Instead of just showing statistics, the system provides a narrative profile: "This trader is a momentum-style swing trader who excels in trending markets. They typically enter 2-4 hours after breakouts with moderate leverage (3-5x) and have a disciplined 3:1 reward-to-risk ratio. Their weakness is ranging markets where they tend to overtrade. Best to copy during high-volatility periods."

---

## 11. Funding Rate Arbitrage Strategist

### Concept

An intelligent system that identifies funding arbitrage opportunities and explains the full trade structure including risks.

### SDK Features Leveraged

- **Calculation Tools**: Complex arbitrage calculations as MCP tools
- **Hooks**: PreToolUse to validate arbitrage parameters
- **Cost Tracking**: Monitor analysis costs with `max_budget_usd`

### Implementation

```python
@tool("calculate_funding_arb", "Calculate funding rate arbitrage opportunity", {
    "coin": str,
    "capital": float,
    "max_leverage": int
})
async def calculate_funding_arb(args):
    # Get current funding rates
    hl_funding = get_hyperliquid_funding(args["coin"])
    spot_borrow = get_spot_borrow_rate(args["coin"])
    cex_funding = get_cex_funding_rates(args["coin"])

    # Calculate opportunities
    opportunities = []

    # HL vs CEX funding arb
    if abs(hl_funding - cex_funding["binance"]) > 0.01:
        opportunities.append({
            "type": "cross_exchange_funding",
            "long_venue": "hyperliquid" if hl_funding < cex_funding["binance"] else "binance",
            "short_venue": "binance" if hl_funding < cex_funding["binance"] else "hyperliquid",
            "expected_yield_8h": abs(hl_funding - cex_funding["binance"]),
            "annualized_yield": abs(hl_funding - cex_funding["binance"]) * 3 * 365,
            "capital_efficiency": args["max_leverage"]
        })

    # Spot-perp basis trade
    basis = calculate_basis(args["coin"])
    if abs(basis) > 0.02:
        opportunities.append({
            "type": "basis_trade",
            "direction": "long_spot_short_perp" if basis > 0 else "short_spot_long_perp",
            "current_basis": basis,
            "expected_convergence_days": estimate_convergence(basis),
            "projected_return": abs(basis)
        })

    return {"content": [{"type": "text", "text": json.dumps(opportunities)}]}

options = ClaudeAgentOptions(
    system_prompt="""You are a funding rate arbitrage specialist. When analyzing opportunities:

    1. Explain the trade structure clearly (what to long, what to short, where)
    2. Calculate all costs (funding, fees, slippage, margin)
    3. Identify risks (basis risk, execution risk, exchange risk)
    4. Provide step-by-step execution instructions
    5. Set up monitoring parameters

    Be precise about numbers and conservative about projections.""",
    mcp_servers={"arb": arb_server}
)
```

### Value Proposition

Traders get not just "opportunity detected" but a complete playbook: "Currently, Hyperliquid BTC funding is -0.03% while Binance is +0.02%. You can capture 0.05% every 8 hours by going long on HL and short on Binance. At 5x leverage on $100k capital, this yields ~27% APY. Execution: First open the HL long, then within 30 seconds open Binance short to minimize basis risk. Monitor for funding convergence which typically takes 6-12 hours."

---

## 12. Market Regime Classifier with Adaptive Strategies

### Concept

An AI system that classifies the current market regime and adapts strategy recommendations accordingly.

### SDK Features Leveraged

- **Multiple Agents**: Different agents for different regimes
- **Dynamic Tool Selection**: Change available tools based on regime
- **Session State**: Track regime across conversation

### Implementation

```python
from claude_agent_sdk import AgentDefinition, ClaudeAgentOptions

# Define regime-specific agents
trending_agent = AgentDefinition(
    description="Strategies for trending markets",
    prompt="""You are a trend-following specialist. In trending markets:
    - Focus on momentum signals
    - Use breakout entries
    - Trail stops to capture moves
    - Increase position sizes on pullbacks""",
    tools=["mcp__hl__candles", "mcp__hl__meta_and_asset_ctxs"],
    model="sonnet"
)

ranging_agent = AgentDefinition(
    description="Strategies for ranging markets",
    prompt="""You are a mean-reversion specialist. In ranging markets:
    - Fade moves to range extremes
    - Use tight stops
    - Take quick profits
    - Reduce position sizes""",
    tools=["mcp__hl__l2_snapshot", "mcp__hl__funding_history"],
    model="sonnet"
)

volatile_agent = AgentDefinition(
    description="Strategies for highly volatile markets",
    prompt="""You are a volatility specialist. In volatile markets:
    - Reduce position sizes significantly
    - Widen stops to avoid noise
    - Focus on liquidation plays
    - Consider straddle-like structures""",
    tools=["mcp__hl__meta_and_asset_ctxs", "mcp__sim__liquidation_map"],
    model="opus"
)

@tool("classify_regime", "Classify current market regime", {"coin": str})
async def classify_regime(args):
    # Calculate regime indicators
    volatility = calculate_realized_vol(args["coin"], 24)
    trend_strength = calculate_adx(args["coin"])
    range_bound = calculate_range_score(args["coin"])

    if volatility > 100:  # High vol threshold
        regime = "volatile"
    elif trend_strength > 25:
        regime = "trending"
    else:
        regime = "ranging"

    return {
        "content": [{
            "type": "text",
            "text": json.dumps({
                "regime": regime,
                "volatility": volatility,
                "trend_strength": trend_strength,
                "range_score": range_bound,
                "recommended_agent": f"{regime}_agent"
            })
        }]
    }

options = ClaudeAgentOptions(
    system_prompt="""You are a market regime analyst. First classify the regime, then:
    1. Hand off to the appropriate specialist agent
    2. Ensure strategies match current conditions
    3. Alert when regime changes are detected""",
    agents={
        "trending": trending_agent,
        "ranging": ranging_agent,
        "volatile": volatile_agent
    },
    mcp_servers={"regime": regime_server}
)
```

### Value Proposition

Strategies automatically adapt to market conditions, preventing the common mistake of applying trending strategies in ranging markets or vice versa.

---

## Summary: SDK Feature Mapping

| SDK Feature | Use Cases |
|------------|-----------|
| **Custom MCP Tools** | All use cases - expose Hyperliquid SDK as tools |
| **Hooks (PreToolUse)** | #2 Risk guardrails, #7 Signal validation |
| **Hooks (PostToolUse)** | #2 Risk disclosure, #5 Anomaly explanation |
| **Custom Agents** | #3 Research team, #8 Risk analyst, #12 Regime strategies |
| **Streaming** | #4 Live commentary |
| **Conversation Continuity** | #6 Backtest iteration, #9 Scenario exploration |
| **Permission Callbacks** | #7 Signal validation |
| **Structured Output** | #10 Trader profiles |
| **Session Forking** | #9 Scenario branching |
| **File Checkpointing** | #6 Strategy versions |
| **Budget Control** | #8 Cost-efficient monitoring |
| **Model Selection** | #3 Task-appropriate models |

---

## Getting Started

1. **Install Dependencies**
```bash
pip install claude-agent-sdk hyperliquid-python-sdk
```

2. **Create Hyperliquid MCP Server**
```python
from claude_agent_sdk import tool, create_sdk_mcp_server
from hyperliquid.info import Info
from hyperliquid.utils import constants

info = Info(constants.MAINNET_API_URL)

@tool("user_state", "Get user positions and balances", {"address": str})
async def user_state(args):
    return {"content": [{"type": "text", "text": json.dumps(info.user_state(args["address"]))}]}

@tool("user_fills", "Get user trade history", {"address": str})
async def user_fills(args):
    return {"content": [{"type": "text", "text": json.dumps(info.user_fills(args["address"]))}]}

# ... add more tools as needed

hl_server = create_sdk_mcp_server(
    name="hyperliquid",
    version="1.0.0",
    tools=[user_state, user_fills, ...]
)
```

3. **Build Your Application**
```python
from claude_agent_sdk import ClaudeAgentOptions, query

options = ClaudeAgentOptions(
    system_prompt="Your trading assistant prompt...",
    mcp_servers={"hl": hl_server},
    allowed_tools=["mcp__hl__user_state", "mcp__hl__user_fills"]
)

async for msg in query(prompt="Analyze whale 0x...", options=options):
    print(msg)
```

---

## Conclusion

The Claude Agent SDK provides powerful primitives for building intelligent trading systems on top of Hyperliquid data:

- **Tools** bridge the gap between raw SDK data and AI understanding
- **Hooks** enable safety guardrails and automatic enrichment
- **Agents** allow specialized expertise for different trading functions
- **Streaming** enables real-time interaction with market events
- **Structured outputs** ensure consistent, parseable results

By combining these features with the rich data available through the Hyperliquid Python SDK, developers can create sophisticated trading assistants that go far beyond simple alerts and dashboards.

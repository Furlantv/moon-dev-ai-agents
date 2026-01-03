# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an experimental AI trading system that orchestrates 48+ specialized AI agents to analyze markets, execute strategies, and manage risk across cryptocurrency markets (primarily Solana). The project uses a modular agent architecture with unified LLM provider abstraction supporting Claude, GPT-4, DeepSeek, Groq, Gemini, and local Ollama models.

**Python Version**: 3.10.9 (tested and recommended)

## Key Development Commands

### Environment Setup
```bash
# Use existing conda environment (DO NOT create new virtual environments)
conda activate tflow

# Install/update dependencies
pip install -r requirements.txt

# IMPORTANT: Update requirements.txt every time you add a new package
pip freeze > requirements.txt
```

### Running the System
```bash
# Run main orchestrator (controls multiple agents)
python src/main.py

# Run individual agents standalone
python src/agents/trading_agent.py
python src/agents/risk_agent.py
python src/agents/rbi_agent.py
python src/agents/chat_agent.py
# ... any agent in src/agents/ can run independently
```

### Backtesting with RBI Agent
```bash
# Single strategy backtest
python src/agents/rbi_agent.py

# Parallel backtesting with 18 threads (tests across 20+ data sources)
python src/agents/rbi_agent_pp_multi.py

# Web dashboard for backtesting (access at http://localhost:8001)
cd src/data/rbi_pp_multi
python app.py

# Key Configuration in rbi_agent_pp_multi.py (lines 130-132):
# TARGET_RETURN = 50        # AI tries to optimize to this return %
# SAVE_IF_OVER_RETURN = 1.0 # Save backtest to CSV if return > this %
# MAX_WORKERS = 18          # Number of parallel threads
# DEBUG_BACKTEST_ERRORS = True  # Auto-fix coding errors with AI
```

## Architecture Overview

### Core Structure
```
src/
├── agents/                      # 48+ specialized AI agents (each <800 lines)
├── models/                      # LLM provider abstraction (ModelFactory pattern)
├── strategies/                  # User-defined trading strategies
├── scripts/                     # Standalone utility scripts
├── data/                        # Agent outputs, memory, analysis results
├── config.py                    # Global configuration (positions, risk limits, API settings)
├── main.py                      # Main orchestrator for multi-agent loop
├── nice_funcs.py                # Core Solana/BirdEye trading utilities
├── nice_funcs_hyperliquid.py    # Hyperliquid exchange functions
├── nice_funcs_extended.py       # Extended Exchange (X10) functions
├── nice_funcs_aster.py          # Aster exchange functions
└── ezbot.py                     # Legacy trading controller
```

### Agent Ecosystem

**Trading Agents**: `trading_agent`, `strategy_agent`, `risk_agent`, `copybot_agent`
**Market Analysis**: `sentiment_agent`, `whale_agent`, `funding_agent`, `liquidation_agent`, `chartanalysis_agent`
**Content Creation**: `chat_agent`, `clips_agent`, `tweet_agent`, `video_agent`, `phone_agent`
**Strategy Development**: `rbi_agent` (Research-Based Inference - codes backtests from videos/PDFs), `research_agent`
**Specialized**: `sniper_agent`, `solana_agent`, `tx_agent`, `million_agent`, `tiktok_agent`, `compliance_agent`

Each agent can run independently or as part of the main orchestrator loop.

### LLM Integration (Model Factory)

Located at `src/models/model_factory.py` and `src/models/README.md`

**Unified Interface**: All agents use `ModelFactory.create_model()` for consistent LLM access
**Supported Providers**: Anthropic Claude (default), OpenAI, DeepSeek, Groq, Google Gemini, Ollama (local)
**Key Pattern**:
```python
from src.models.model_factory import ModelFactory

model = ModelFactory.create_model('anthropic')  # or 'openai', 'deepseek', 'groq', etc.
response = model.generate_response(system_prompt, user_content, temperature, max_tokens)
```

### Configuration Management

**Primary Config**: `src/config.py`
- Trading settings: `MONITORED_TOKENS`, `EXCLUDED_TOKENS`, position sizing (`usd_size`, `max_usd_order_size`)
- Risk management: `CASH_PERCENTAGE`, `MAX_POSITION_PERCENTAGE`, `MAX_LOSS_USD`, `MAX_GAIN_USD`, `MINIMUM_BALANCE_USD`
- Agent behavior: `SLEEP_BETWEEN_RUNS_MINUTES`, `ACTIVE_AGENTS` dict in `main.py`
- AI settings: `AI_MODEL`, `AI_MAX_TOKENS`, `AI_TEMPERATURE`

**Environment Variables**: `.env` (see `.env_example`)
- Trading APIs: `BIRDEYE_API_KEY`, `MOONDEV_API_KEY`, `COINGECKO_API_KEY`
- AI Services: `ANTHROPIC_KEY`, `OPENAI_KEY`, `DEEPSEEK_KEY`, `GROQ_API_KEY`, `GEMINI_KEY`, `XAI_API_KEY`, `OPENROUTER_API_KEY`
- Blockchain: `SOLANA_PRIVATE_KEY`, `HYPER_LIQUID_ETH_PRIVATE_KEY`, `RPC_ENDPOINT`
- Extended Exchange: `X10_API_KEY`, `X10_PRIVATE_KEY`, `X10_PUBLIC_KEY`, `X10_VAULT_ID`

### Shared Utilities

**`src/nice_funcs.py`** (~1,200 lines): Core trading functions
- Data: `token_overview()`, `token_price()`, `get_position()`, `get_ohlcv_data()`
- Trading: `market_buy()`, `market_sell()`, `chunk_kill()`, `open_position()`
- Analysis: Technical indicators, PnL calculations, rug pull detection

**`src/agents/api.py`**: `MoonDevAPI` class for custom Moon Dev API endpoints
- `get_liquidation_data()`, `get_funding_data()`, `get_oi_data()`, `get_copybot_follow_list()`

### Multi-Exchange Support

The system supports multiple cryptocurrency exchanges with unified function signatures:

**Hyperliquid** (`nice_funcs_hyperliquid.py`)
- EVM-compatible perpetuals DEX
- Leverage up to 50x
- Functions: `market_buy()`, `market_sell()`, `get_position()`, `close_position()`
- Configuration: Set `EXCHANGE = 'hyperliquid'` in `config.py`

**Extended Exchange / X10** (`nice_funcs_extended.py`)
- StarkNet-based perpetuals
- Leverage up to 20x
- Auto symbol conversion (BTC → BTC-USD)
- Functions match Hyperliquid API for compatibility

**Aster** (`nice_funcs_aster.py`)
- Additional exchange support
- Similar function interface as other exchanges

**Solana / BirdEye** (`nice_funcs.py`)
- Solana spot token trading
- Real-time market data for 15,000+ tokens
- Default exchange for the system

**Switching Exchanges**: In any agent, change the import:
```python
# For Hyperliquid
from src import nice_funcs_hyperliquid as nf

# For Extended/X10
from src import nice_funcs_extended as nf

# For Solana (default)
from src import nice_funcs as nf
```

### Swarm Mode Trading

The `trading_agent.py` supports two operation modes:

**Single Model Mode** (default, fast ~10s)
- Uses one AI model for decisions
- Configure via `AI_MODEL` in `config.py`

**Swarm Mode** (~45-60s, multi-model consensus)
- Queries 6 AI models in parallel: Claude 4.5, GPT-5, Gemini 2.5, Grok-4, DeepSeek, DeepSeek-R1
- Generates majority vote consensus
- Configure via `USE_SWARM_MODE = True` in agent or `config.py`
- Uses `swarm_agent.py` for parallel processing

### Data Flow Pattern

```
Config/Input → Agent Init → API Data Fetch → Data Parsing →
LLM Analysis (via ModelFactory) → Decision Output →
Result Storage (CSV/JSON in src/data/) → Optional Trade Execution
```

## Claude Code Integration

This repository includes a Claude Code skill at `.claude/skills/moon-dev-trading-agents/` with:
- `SKILL.md`: Quick reference for working with the codebase
- `AGENTS.md`: Complete list of all 48+ agents with descriptions
- `WORKFLOWS.md`: Common workflow examples and patterns
- `ARCHITECTURE.md`: Deep dive into system architecture

The skill provides context-aware assistance when using Claude Code CLI tool.

## Development Rules

### File Management
- **Keep files under 800 lines** - if longer, split into new files and update README
- **DO NOT move files without asking** - you can create new files but no moving
- **NEVER create new virtual environments** - use existing `conda activate tflow`
- **Update requirements.txt** after adding any new package

**Critical Files** (never move or rename):
- `src/config.py` - Central configuration
- `src/nice_funcs*.py` - Exchange-specific utilities
- `src/main.py` - Main orchestrator
- `src/models/model_factory.py` - LLM abstraction layer
- `.env` - Secrets and API keys (never commit)

### Backtesting
- Use `backtesting.py` library (NOT their built-in indicators)
- Use `pandas_ta` or `talib` for technical indicators instead
- Sample data available in `src/data/rbi/` directory
- RBI agent automatically downloads and caches data for 20+ crypto symbols

### Code Style
- **No fake/synthetic data** - always use real data or fail the script
- **Minimal error handling** - user wants to see errors, not over-engineered try/except blocks
- **No API key exposure** - never show keys from `.env` in output

### Agent Development Pattern

When creating new agents:
1. Inherit from base patterns in existing agents
2. Use `ModelFactory` for LLM access
3. Store outputs in `src/data/[agent_name]/`
4. Make agent independently executable (standalone script)
5. Add configuration to `config.py` if needed
6. Follow naming: `[purpose]_agent.py`

### Testing Strategies

Place strategy definitions in `src/strategies/` folder:
```python
class YourStrategy(BaseStrategy):
    name = "strategy_name"
    description = "what it does"

    def generate_signals(self, token_address, market_data):
        return {
            "action": "BUY"|"SELL"|"NOTHING",
            "confidence": 0-100,
            "reasoning": "explanation"
        }
```

## Important Context

### Risk-First Philosophy
- Risk Agent runs first in main loop before any trading decisions
- Configurable circuit breakers (`MAX_LOSS_USD`, `MINIMUM_BALANCE_USD`)
- AI confirmation for position-closing decisions (configurable via `USE_AI_CONFIRMATION`)

### Data Sources
1. **BirdEye API** - Solana token data (price, volume, liquidity, OHLCV)
2. **Moon Dev API** - Custom signals (liquidations, funding rates, OI, copybot data)
3. **CoinGecko API** - 15,000+ token metadata, market caps, sentiment
4. **Helius RPC** - Solana blockchain interaction
5. **Hyperliquid API** - EVM perpetuals market data and execution
6. **Extended Exchange (X10)** - StarkNet perpetuals data
7. **Aster Exchange** - Additional market data sources

### Agent Data Storage

Each agent stores its outputs in `src/data/[agent_name]/`:
- CSV files for structured results
- JSON for complex data structures
- Text files for logs and analysis
- Subdirectories organized by date/run

Examples:
- `src/data/rbi/` - Backtest strategies and results
- `src/data/rbi_pp_multi/` - Parallel backtest outputs
- `src/data/tweets/` - Generated tweet content
- `src/data/volume_agent/` - Volume monitoring data

### Autonomous Execution
- Main loop runs every 15 minutes by default (`SLEEP_BETWEEN_RUNS_MINUTES`)
- Agents handle errors gracefully and continue execution
- Keyboard interrupt for graceful shutdown
- All agents log to console with color-coded output (termcolor)

### AI-Driven Strategy Generation (RBI Agent)
1. User provides: YouTube video URL / PDF / trading idea text
2. DeepSeek-R1 analyzes and extracts strategy logic
3. Generates backtesting.py compatible code
4. Executes backtest and returns performance metrics
5. Cost: ~$0.027 per backtest execution (~6 minutes)

## Common Patterns

### Adding New Agent
1. Create `src/agents/your_agent.py`
2. Implement standalone execution logic
3. Add to `ACTIVE_AGENTS` in `main.py` if needed for orchestration
4. Use `ModelFactory` for LLM calls
5. Store results in `src/data/your_agent/`

### Adding Custom Backtest Data Sources

Edit the data list in `rbi_agent_pp_multi.py` (lines 157-178):
```python
ALL_DATA_CONFIGS = [
    {'symbol': 'BTC-USD', 'timeframe': '15m', 'days_back': 90},
    {'symbol': 'ETH-USD', 'timeframe': '15m', 'days_back': 90},
    {'symbol': 'YOUR_TOKEN_ADDRESS', 'timeframe': '1H', 'days_back': 30},
]
```

The agent automatically downloads and caches data from BirdEye/CoinGecko.

### Switching AI Models
Edit `config.py`:
```python
AI_MODEL = "claude-3-haiku-20240307"  # Fast, cheap
# AI_MODEL = "claude-3-sonnet-20240229"  # Balanced
# AI_MODEL = "claude-3-opus-20240229"  # Most powerful
```

Or use different models per agent via ModelFactory:
```python
model = ModelFactory.create_model('deepseek')  # Reasoning tasks
model = ModelFactory.create_model('groq')      # Fast inference
```

### Reading Market Data

**Solana / BirdEye:**
```python
from src.nice_funcs import token_overview, get_ohlcv_data, token_price

# Get comprehensive token data
overview = token_overview(token_address)

# Get price history
ohlcv = get_ohlcv_data(token_address, timeframe='1H', days_back=3)

# Get current price
price = token_price(token_address)
```

**Hyperliquid:**
```python
from src import nice_funcs_hyperliquid as nf

# Get position
position = nf.get_position("BTC")

# Market operations
nf.market_buy("BTC", usd_amount=100, leverage=10)
nf.close_position("BTC")
```

**Extended Exchange:**
```python
from src import nice_funcs_extended as nf

# Auto symbol conversion (BTC → BTC-USD)
nf.market_buy("BTC", usd_amount=100, leverage=15)
position = nf.get_position("BTC")
nf.close_position("BTC")
```

## Project Philosophy

This is an **experimental, educational project** demonstrating AI agent patterns through algorithmic trading:
- No guarantees of profitability (substantial risk of loss)
- Open source and free for learning
- YouTube-driven development with weekly updates
- Community-supported via Discord
- No token associated with project (avoid scams)

The goal is to democratize AI agent development and show practical multi-agent orchestration patterns that can be applied beyond trading.

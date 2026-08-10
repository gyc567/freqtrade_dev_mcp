# Freqtrade MCP — 完整部署、安装与使用教程

> 面向新手的零基础指南。本教程基于 **freqtrade_dev_mcp** 仓库（`https://github.com/gyc567/freqtrade_dev_mcp`）实测编写，
> 覆盖从克隆代码 → 安装依赖 → 注册 MCP 服务器 → 调用 12 个工具 → 跑通完整交易策略开发流程的全过程。

---

## 目录

1. [项目是什么](#1-项目是什么)
2. [架构速览](#2-架构速览)
3. [环境要求](#3-环境要求)
4. [安装部署](#4-安装部署)
5. [配置](#5-配置)
6. [注册 MCP 服务器](#6-注册-mcp-服务器)
7. [验证安装](#7-验证安装)
8. [12 个工具详解](#8-12-个工具详解)
9. [新手快速上手：完整工作流](#9-新手快速上手完整工作流)
10. [策略开发代理（可选）](#10-策略开发代理可选)
11. [开发与测试](#11-开发与测试)
12. [常见问题排查](#12-常见问题排查)
13. [安全与最佳实践](#13-安全与最佳实践)

---

## 1. 项目是什么

**Freqtrade MCP** 是一个基于 [Model Context Protocol (MCP)](https://modelcontextprotocol.io) 的服务器，
把开源量化交易框架 **Freqtrade** 的常用操作封装成 12 个标准 MCP 工具，供 Claude、Reasonix 等 AI 客户端直接调用。

它解决的问题：**让 AI 帮你完成从"交易想法"到"可回测策略"的完整闭环**。

```
💡 想法 → 🛠 创建策略 → 📥 下载数据 → ⚖️ 回测 → 🔬 超参优化 → 📊 分析 → 🔄 迭代 → 🚀 部署
```

核心能力：

| 能力 | 说明 |
|------|------|
| 策略生成 | 通过模板或"线框"（wireframe）创建策略代码，AI 可自由定制 |
| 数据获取 | 自然语言日期范围（"last year"、"Q1 2024"）、一键下载 top15 币种 |
| 回测 / 超参优化 | 调用 Freqtrade 引擎验证和优化策略 |
| 深度分析 | 解析回测 zip / 超参 fthypt 文件，提取交易级别洞察 |
| 智能搜索 | 在历史结果库中按收益、胜率、回撤等条件检索 |

---

## 2. 架构速览

```
┌─────────────────┐   stdio (JSON-RPC)   ┌──────────────────────────────┐
│  AI 客户端       │ ◄──────────────────► │  MCP Server (src/server.py)   │
│  (Claude /       │                      │  ├─ 12 个工具注册             │
│   Reasonix ...)  │                      │  ├─ 命令分发 (src/commands/)  │
└─────────────────┘                      │  └─ 配置管理 (src/config.py)  │
                                         └──────────────┬───────────────┘
                                                        │ 子进程调用
                                         ┌──────────────▼───────────────┐
                                         │  Freqtrade CLI               │
                                         │  (回测 / hyperopt / 下载)     │
                                         └──────────────────────────────┘

可选组件：
strategy_agent/  ── LangGraph 驱动的"策略开发代理"，内部通过 stdio 连回本 MCP 服务器，
                    用 LLM 自动迭代策略直到达到收益目标。
```

**关键目录**：

```
freqtrade_dev_mcp/
├── src/                    # MCP 服务器实现
│   ├── server.py           # 入口（main）
│   ├── config.py           # 配置加载（文件 + 环境变量）
│   ├── commands/           # 12 个工具的命令实现
│   ├── models/             # Pydantic 数据模型
│   └── utils/              # 自然语言日期解析等
├── strategy_agent/         # 可选：LangGraph 自动策略开发代理
├── examples/               # 配置样例与示例脚本
├── tests/                  # 测试套件
├── requirements.txt        # 生产依赖
└── pyproject.toml          # 包定义（freqtrade-mcp）
```

---

## 3. 环境要求

| 依赖 | 要求 | 说明 |
|------|------|------|
| 操作系统 | macOS / Linux / Windows(WSL) | 本教程以 macOS 为例 |
| Python | **3.11 – 3.13** | 仓库声明 ≥3.8，但最新版 freqtrade 需要较新 Python，实测 3.13 稳定 |
| uv | ≥0.5（推荐） | Python 包管理器，比 pip 快很多；无 uv 时可用 pip 替代 |
| Freqtrade 数据 | 网络可达交易所 | 下载历史 K 线需要网络 |
| AI 客户端 | 任意 MCP 客户端 | Claude Desktop、Reasonix、Cursor 等 |

> **为什么推荐 uv？** 本项目依赖较多（129 个包），uv 解析和安装快 10 倍以上，且自带 Python 版本管理。
> 安装 uv：`curl -LsSf https://astral.sh/uv/install.sh | sh`

---

## 4. 安装部署

### 4.1 克隆代码

```bash
git clone https://github.com/gyc567/freqtrade_dev_mcp.git
cd freqtrade_dev_mcp
```

### 4.2 创建独立虚拟环境（推荐）

用独立的 venv 避免污染系统 Python（当前终端会话的 `python` 可能指向其他环境，务必用绝对路径）。

```bash
uv venv .venv --python 3.13
# 输出: Creating virtual environment at: .venv
```

### 4.3 安装依赖（⚠️ 注意 mcp 版本坑）

**直接执行 `requirements.txt` 会踩坑**：其中 `mcp>=0.9.0` 没有上限，会被解析到 **mcp 2.0.0**，
而本项目代码使用的是 mcp 0.9–1.x 的装饰器 API（`@server.list_tools()`），2.0 已将其移除，服务器会启动失败。

**正确做法：先固定 mcp < 2.0，再安装其余依赖**

```bash
# 方式 A（推荐）：先装兼容版 mcp，再装其余
uv pip install --python .venv/bin/python "mcp>=1.0,<2.0"
uv pip install --python .venv/bin/python -r requirements.txt

# 方式 B：一次性指定
uv pip install --python .venv/bin/python "mcp>=1.0,<2.0" -r requirements.txt
```

> 如果已经装成了 2.0，可以这样降级：`uv pip install --python .venv/bin/python "mcp>=1.0,<2.0"`（实测可无缝降级到 1.29.0）。
>
> **建议顺手把 `requirements.txt` 里的 `mcp>=0.9.0` 改成 `mcp>=1.0,<2.0`**，避免以后重装再踩。

不使用 uv 时（pip 方式，假设已激活 venv）：

```bash
python -m pip install "mcp>=1.0,<2.0"
python -m pip install -r requirements.txt
```

### 4.4 验证依赖

```bash
.venv/bin/python -c "import mcp, freqtrade, pydantic, pandas; print('imports OK')"
# 期望输出: imports OK（mcp 应为 1.x，例如 1.29.0）
.venv/bin/python -c "import mcp; from mcp.server import Server; print('list_tools' in dir(Server))"
# 期望输出: True（2.0 会输出 False，说明版本不对）
```

---

## 5. 配置

### 5.1 配置加载优先级

```
环境变量  >  ~/.config/freqtrade-mcp/config.json  > 内置默认值
```

### 5.2 配置文件（可选）

默认路径：`~/.config/freqtrade-mcp/config.json`（参考 `examples/config.json`）：

```json
{
  "freqtrade_path": "/path/to/freqtrade",
  "default_exchange": "binance",
  "default_stake_amount": 100.0,
  "default_epochs": 100,
  "coingecko_api_key": "your_api_key",
  "log_level": "INFO"
}
```

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `freqtrade_path` | `~/freqtrade` | Freqtrade 安装/工作目录，策略与数据存放于此 |
| `default_exchange` | `binance` | 默认交易所 |
| `default_stake_amount` | `100.0` | 每笔回测投入金额 |
| `default_epochs` | `100` | hyperopt 默认迭代次数 |
| `coingecko_api_key` | 无 | 可选，用于 top 币种选择 |
| `log_level` | `INFO` | 日志级别 |

### 5.3 环境变量

**服务器配置（`FREQTRADE_MCP_` 前缀）**：

```bash
export FREQTRADE_MCP_PATH="/path/to/freqtrade"      # 覆盖 freqtrade_path
export FREQTRADE_MCP_EXCHANGE="binance"              # 覆盖默认交易所
export FREQTRADE_MCP_COINGECKO_KEY="your_key"        # 可选
export FREQTRADE_MCP_LOG_LEVEL="INFO"                # DEBUG/INFO/WARNING/ERROR
```

**LLM 配置（仅策略开发代理需要）**：

```bash
export LLM_MODEL="openai/gpt-4o-mini"      # 或 deepseek/deepseek-chat、anthropic/claude-3-haiku-20240307、groq/llama-3.1-8b-instant
export LLM_API_KEY="sk-..."                # 对应提供商 API Key（必填）
export LLM_TEMPERATURE="0.3"               # 可选，默认 0.3
export LLM_MAX_TOKENS="2048"               # 可选
export LLM_TIMEOUT="300"                   # 可选，秒
```

---

## 6. 注册 MCP 服务器

把服务器注册进你的 AI 客户端，之后就可以直接说人话调用交易功能了。下面给出三种方式。

### 6.1 启动方式（重要，三个坑）

本项目入口是包模块 `src.server`。**不要**用 `python src/server.py` 或 `python run_server.py`
直接运行——两者都会因为 `commands/` 包内部使用相对导入而崩溃（`attempted relative import beyond top-level package`）。

**正确的启动命令**（任意目录下均可）：

```bash
PYTHONPATH=/绝对/路径/freqtrade_dev_mcp /绝对/路径/freqtrade_dev_mcp/.venv/bin/python -m src.server
```

### 6.2 方式 A：Reasonix（本环境实测）

在 Reasonix 中全局注册（所有项目可用）：

1. 在仓库根目录创建 `.mcp.json`：

```json
{
  "mcpServers": {
    "freqtrade": {
      "command": "/绝对/路径/freqtrade_dev_mcp/.venv/bin/python",
      "args": ["-m", "src.server"],
      "env": {
        "PYTHONPATH": "/绝对/路径/freqtrade_dev_mcp",
        "PATH": "/usr/bin:/bin"
      }
    }
  }
}
```

2. 调用 `install_source` 工具全局注册（或使用桌面端 Settings → MCP servers）：
   `source` 填仓库目录，`kind=mcp`，`scope=global`。注册成功后会写入
   `~/.reasonix/config.toml` 并自动验证握手 + 工具列表。

### 6.3 方式 B：Claude Desktop

编辑 `~/Library/Application Support/Claude/claude_desktop_config.json`：

```json
{
  "mcpServers": {
    "freqtrade": {
      "command": "/绝对/路径/freqtrade_dev_mcp/.venv/bin/python",
      "args": ["-m", "src.server"],
      "env": {
        "PYTHONPATH": "/绝对/路径/freqtrade_dev_mcp",
        "PATH": "/usr/local/bin:/usr/bin:/bin"
      }
    }
  }
}
```

保存后**完全退出并重启** Claude Desktop。`examples/claude_desktop_config.json` 有参考模板。

### 6.4 方式 C：任意 MCP 客户端（通用）

任何支持 stdio MCP 的客户端，命令填：

```
command: /绝对/路径/freqtrade_dev_mcp/.venv/bin/python
args:    ["-m", "src.server"]
env:     { "PYTHONPATH": "/绝对/路径/freqtrade_dev_mcp", "PATH": "/usr/bin:/bin" }
```

---

## 7. 验证安装

### 7.1 命令行握手测试（不依赖客户端）

```bash
cd /tmp && PYTHONPATH=/绝对/路径/freqtrade_dev_mcp /绝对/路径/freqtrade_dev_mcp/.venv/bin/python - <<'EOF'
import json, subprocess, time, select

proc = subprocess.Popen(
    ["/绝对/路径/freqtrade_dev_mcp/.venv/bin/python", "-m", "src.server"],
    stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True,
    env={"PATH": "/usr/bin:/bin", "PYTHONPATH": "/绝对/路径/freqtrade_dev_mcp"}
)
for m in [
    {"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"probe","version":"0.0.1"}}},
    {"jsonrpc":"2.0","method":"notifications/initialized"},
    {"jsonrpc":"2.0","id":2,"method":"tools/list"},
]:
    proc.stdin.write(json.dumps(m)+"\n"); proc.stdin.flush()
deadline, results = time.time() + 25, []
while time.time() < deadline and len(results) < 2:
    if select.select([proc.stdout], [], [], 1)[0]:
        line = proc.stdout.readline()
        if line.startswith("{"):
            results.append(line)
d = json.loads(results[1])
print("tools/list OK: %d 个工具" % len(d["result"]["tools"]))
print([t["name"] for t in d["result"]["tools"]])
proc.kill()
EOF
```

期望输出 `tools/list OK: 12 个工具` 及全部工具名。

### 7.2 客户端内验证

在 AI 客户端里直接说：

```
列出 freqtrade 服务器可用的工具
```

或调用一个只读工具测试连通：

```
使用 list_results 查看现有的回测/超参结果
```

---

## 8. 12 个工具详解

| # | 工具 | 分类 | 功能 | 关键参数 |
|---|------|------|------|----------|
| 1 | `create_userdir` | 环境 | 初始化 Freqtrade 工作目录结构 | `userdir`, `reset` |
| 2 | `create_config` | 环境 | 从模板生成交易配置 | `config_path`, `template`(default/conservative/aggressive/advanced), `exchange`, `stake_currency`, `trading_mode`(spot/futures/margin), `stake_amount`, `max_open_trades`, `pairs`, `dry_run` |
| 3 | `create_strategy_wireframe` | 策略 | 生成极简策略"线框"，给 AI 最大自由度 | `strategy_name`, `style`(minimal/guided/structured), `description`, `include_comments`, `include_examples`, `strategy_path` |
| 4 | `create_strategy` | 策略 | 从成熟模板生成完整策略 | `strategy_name`, `template`(basic/trend/mean_reversion/scalping/advanced), `timeframe`, `indicators`, `stoploss`, `minimal_roi`, `trailing_stop`, `can_short` |
| 5 | `download_candles` | 数据 | 下载历史 K 线（支持自然语言时间） | `pairs`(如 `["BTC/USDT:USDT"]` 或 `"top15"`), `timeframes`(5m/15m/30m/1h/4h/1d 或 `"all"`), `date_range`("last year"/"Q1 2024"/"september"), `exchange`, `trading_mode` |
| 6 | `backtest_strategy` | 执行 | 回测策略，可导出交易与信号 | `strategy_name`, `pairs`, `timerange`("last 3 months"), `stake_amount`, `export_trades`, `export_signals`, `enable_protections` |
| 7 | `hyperopt_strategy` | 执行 | 超参优化 | `strategy_name`, `pairs`, `timerange`, `epochs`(默认100), `spaces`(all/buy/sell/roi/stoploss), `loss_function` |
| 8 | `extract_backtest_data` | 分析 | 解析回测 zip，输出绩效/交易/市场洞察 | `result_path`, `output_format`(summary/detailed/raw), `include_trades`, `include_performance`, `include_market_data`, `include_config`, `include_strategy_code` |
| 9 | `extract_hyperopt_data` | 分析 | 解析超参 `.fthypt`，参数敏感性分析 | `hyperopt_path`, `output_format`, `include_best_params`, `include_convergence`, `include_parameter_ranges`, `include_trials`, `max_trials` |
| 10 | `search_results` | 检索 | 高级筛选历史结果 | `query`, `result_type`, `strategy_name`, `min_profit`, `min_winrate`, `max_drawdown`, `min_trades`, `date_from`/`date_to`, `sort_by`(date/profit/winrate/trades/drawdown), `rebuild_index` |
| 11 | `list_results` | 检索 | 浏览已有结果 | `result_type`(backtest/hyperopt/all), `strategy`, `limit` |
| 12 | `get_result` | 检索 | 按 ID 取结果详情 | `result_id`, `include_metadata` |

> 工具在客户端中显示为 `mcp__freqtrade__<工具名>`（Reasonix/Claude 前缀规范）。

---

## 9. 新手快速上手：完整工作流

下面是一条从零到有、可完整跑通的路径。**建议按顺序执行**，每一步都对应一个工具。

### Step 1 — 初始化 Freqtrade 工作区

```
create_userdir(userdir="/path/to/my_trading_project")
```

### Step 2 — 生成交易配置

```
create_config(
    config_path="config.json",
    template="conservative",
    exchange="binance",
    stake_currency="USDT",
    trading_mode="spot"
)
```

### Step 3 — 创建策略

方案 A（推荐给 AI 开发，自由度最高）：

```
create_strategy_wireframe(
    strategy_name="MyFirstStrategy",
    style="guided",
    description="EMA 交叉趋势跟随策略"
)
```

方案 B（模板起步，最快出结果）：

```
create_strategy(
    strategy_name="MyFirstStrategy",
    template="trend",
    timeframe="1h",
    indicators=["EMA", "MACD"]
)
```

### Step 4 — 下载历史数据

```
download_candles(
    pairs=["BTC/USDT", "ETH/USDT"],
    timeframes=["1h"],
    date_range="last 6 months",
    exchange="binance"
)
```

> 想偷懒：`pairs=["top15"]` 自动按市值选 top15 币种；时间可说 "last year"、"Q1 2024"、"september"。

### Step 5 — 首次回测

```
backtest_strategy(
    strategy_name="MyFirstStrategy",
    pairs=["BTC/USDT", "ETH/USDT"],
    timerange="last 3 months",
    export_trades=True,
    export_signals=True
)
```

### Step 6 — 查看回测结果

```
list_results(result_type="backtest", strategy="MyFirstStrategy")
```

拿到 `result_id` 后：

```
get_result(result_id="<回测结果ID>")
```

或深度分析（zip 文件）：

```
extract_backtest_data(
    result_path="/path/to/backtest-result.zip",
    output_format="detailed",
    include_trades=True,
    include_performance=True
)
```

### Step 7 — 超参优化

```
hyperopt_strategy(
    strategy_name="MyFirstStrategy",
    pairs=["BTC/USDT", "ETH/USDT"],
    timerange="last 6 months",
    epochs=100,
    spaces="all"
)
```

之后用 `extract_hyperopt_data` 分析哪些参数最敏感：

```
extract_hyperopt_data(
    hyperopt_path="/path/to/strategy_optimization.fthypt",
    output_format="detailed",
    max_trials=100
)
```

### Step 8 — 数据驱动迭代（本项目精髓）

回测分析结果会告诉你**什么时间段、哪个币种、什么条件下表现最好**。例如：

```json
{
  "hourly_performance": {
    "14": {"trades": 45, "avg_profit": 0.023},
    "15": {"trades": 38, "avg_profit": 0.031}
  },
  "pair_performance": {
    "BTC/USDT": {"trades": 123, "winrate": 0.73, "total_profit": 0.156}
  }
}
```

据此修改策略（加时间过滤、币种过滤等）→ 重新回测 → 再优化，循环到满意为止。
同时可以用 `search_results` 学习历史成功策略：

```
search_results(
    query="profitable trend",
    min_profit=10,
    min_winrate=0.6,
    max_drawdown=15,
    sort_by="profit"
)
```

### Step 9 — 最终验证

```
backtest_strategy(
    strategy_name="MyFirstStrategy",
    pairs=["BTC/USDT", "ETH/USDT", "ADA/USDT"],
    timerange="last 12 months",
    enable_protections=True,
    export_trades=True
)
```

---

## 10. 策略开发代理（可选）

`strategy_agent/` 是一个 LangGraph 驱动的**自动策略开发代理**：给定币种和时间框架，它会自己
循环执行"生成策略 → 回测 → 分析 → 优化"，直到达到收益目标。

### 使用前提

1. 配置 LLM 环境变量（见 [5.3 环境变量](#53-环境变量)），例如：

```bash
export LLM_MODEL="deepseek/deepseek-chat"
export LLM_API_KEY="sk-..."
```

2. 确认 MCP 服务器能启动（代理通过 stdio 内部连接它）。

### 运行

```bash
cd freqtrade_dev_mcp
PYTHONPATH=$PWD .venv/bin/python examples/run_strategy_agent.py \
    --symbols "BTC/USDT:USDT" \
    --timeframes "1h" \
    --max-iterations 5 \
    --min-profit 2.0 \
    --hyperopt-epochs 50 \
    --verbose
```

常用参数：

| 参数 | 说明 | 默认 |
|------|------|------|
| `--symbols` | 交易对列表 | 必填 |
| `--timeframes` | 时间框架 | 必填 |
| `--max-iterations` | 最大迭代轮数 | 5 |
| `--min-profit` | 目标最小收益 % | 1.0 |
| `--hyperopt-epochs` | 每轮超参迭代数 | 100 |
| `--mcp-server-path` | 自定义 MCP 服务器路径 | 自动 |
| `--verbose` | 详细日志 | 关闭 |

---

## 11. 开发与测试

```bash
# 安装开发依赖（含 pytest/ruff/black/mypy）
uv pip install --python .venv/bin/python -e ".[dev]"

# 运行测试
.venv/bin/python -m pytest tests/ -v

# 代码质量
.venv/bin/python -m ruff check src/ tests/
.venv/bin/python -m black --check src/ tests/
.venv/bin/python -m mypy src/

# 或者直接用 Makefile（等价命令）
make test lint type-check check-all
```

常用 Makefile 目标：`install` / `install-dev` / `test` / `lint` / `format` / `type-check` / `run` / `build`。

---

## 12. 常见问题排查

| 症状 | 原因 | 解决 |
|------|------|------|
| `No module named 'mcp'` | 依赖没装进正在用的 Python | 确认用 `.venv/bin/python`，不要用系统的 `python` |
| `'Server' object has no attribute 'list_tools'` | 装成了 mcp 2.0 | 降级：`uv pip install --python .venv/bin/python "mcp>=1.0,<2.0"` |
| `attempted relative import beyond top-level package` | 直接运行 `src/server.py` 或 `run_server.py` | 改用 `PYTHONPATH=仓库根 .venv/bin/python -m src.server` |
| 客户端连不上 / EOF | `.mcp.json` 里路径不对或 cwd 不对 | 用绝对路径 + `PYTHONPATH`（见 §6.1），重启客户端 |
| `Cannot read .zip file` | 回测文件损坏或 freqtrade 版本旧 | 检查文件权限；`uv pip install --python .venv/bin/python "freqtrade>=2025.7"` |
| 搜索结果为空 | 索引未建或为空 | `search_results(rebuild_index=True)` |
| 超参分析慢 | 默认分析 1000+ trials | `extract_hyperopt_data(max_trials=100, output_format="summary")` |
| `LLM_API_KEY is required` | 跑代理没设环境变量 | `export LLM_API_KEY=...`、`export LLM_MODEL=...` |
| 日志出现在 stdout 干扰协议 | 项目日志配置写入 stdout | 一般不影响按行解析的 JSON-RPC；排障时看 `logs/mcp_server_*.log` |
| 端口/防火墙错误 | 无（stdio 不走网络） | 检查客户端配置的 command 是否有执行权限 |

**排障三连**：

```bash
# 1. 依赖是否就绪
.venv/bin/python -c "import mcp; print(mcp.__version__ if hasattr(mcp,'__version__') else '1.x?')"

# 2. 服务器能否独立启动（应无 traceback，日志见 logs/）
PYTHONPATH=$PWD .venv/bin/python -m src.server < /dev/null

# 3. 日志文件
tail -f logs/mcp_server_*.log
```

---

## 13. 安全与最佳实践

1. **API Key 只放环境变量**，绝不写进代码、配置文件或提交到 git。
2. **用独立 venv**，不要用系统 Python 或全局 pip 安装本项目依赖。
3. **回测 ≠ 实盘**：回测结果包含幸存者偏差、滑点假设等，上线前务必用更长周期 + `enable_protections=True` 验证。
4. **`dry_run` 先行**：`create_config(dry_run=True)` 生成纸面交易配置，先跑模拟再考虑实盘。
5. **策略文件路径安全**：`strategy_name` 使用字母数字，防止路径穿越（项目已内置该校验）。
6. **定期清理**：`extract_*` 会缓存结果到 SQLite 索引，结果过多时用 `list_results` + 手动清理。

---

## 附：项目常用命令速查

```bash
# 安装（含 mcp 版本修复）
uv venv .venv --python 3.13
uv pip install --python .venv/bin/python "mcp>=1.0,<2.0"
uv pip install --python .venv/bin/python -r requirements.txt

# 启动服务器（排障用）
PYTHONPATH=$PWD .venv/bin/python -m src.server

# 跑测试 / 检查
.venv/bin/python -m pytest tests/ -v
make check-all
```

---

*教程基于 freqtrade_dev_mcp `8d6eb05`（fix: prob）实测。安装环境：macOS arm64 / Python 3.13 / uv 0.11 / mcp 1.29.0 / freqtrade 2026.7。*

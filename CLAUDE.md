# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述
基于 [TauricResearch/TradingAgents](https://github.com/TauricResearch/TradingAgents)（65K Stars）的 A 股深度特化 fork。多 Agent 投研框架，7 个 Analyst 角色通过 Bull/Bear 辩论 + 三方风险辩论生成投资报告。

- **仓库**: https://github.com/chly23/TradingAgents-astock-win
- **协议**: Apache 2.0
- **Python**: >=3.10
- **当前版本**: 0.2.7

## 常用命令

```bash
# 安装（基础）
pip install -e .

# 安装（含 Google 模型支持）
pip install -e ".[google]"

# 跑测试
python -m pytest tests/ -v

# 跑单个测试文件
python -m pytest tests/test_safe_ticker_component.py -v

# 按 marker 过滤
python -m pytest tests/ -m unit -v

# 启动 Web UI
streamlit run web/launch.py
# 或
tradingagents-web

# 启动 CLI
tradingagents

# 快速脚本示例（不用 CLI）
python main.py
```

## 架构

### 执行流程
`TradingAgentsGraph.propagate(ticker, date)` 是主入口：
1. 各 Analyst 并行拉数据 + 撰写分析报告
2. Bull/Bear Researcher 辩论 → Research Manager 综合
3. Trader 生成交易决策
4. Aggressive/Neutral/Conservative 三方风险辩论
5. Portfolio Manager 最终裁定

编排核心：`tradingagents/graph/` — `trading_graph.py`（主类）+ `setup.py`（构图）+ `propagation.py`（运行）+ `conditional_logic.py`（边路由）+ `signal_processing.py`（结果解析）

### Agent 层（`tradingagents/agents/`）
| 目录 | 内容 |
|------|------|
| `analysts/` | 7 个 Analyst：market、social、news、fundamentals + A 股特化 policy、hot_money_tracker、lockup_watcher |
| `researchers/` | bull_researcher、bear_researcher |
| `managers/` | research_manager、portfolio_manager |
| `risk_mgmt/` | aggressive/neutral/conservative_debator |
| `trader/` | Trader |

### 数据层（`tradingagents/dataflows/`）
- `interface.py` — 对外统一工具函数，按 `data_vendors` 配置路由到具体 vendor
- `a_stock.py` — A 股数据 vendor（所有 HTTP 直连逻辑）
- `utils.py` — `safe_ticker_component`（路径安全 + 中文 ticker 解析）
- `config.py` — 运行时 vendor 配置读写

新增数据接口：在 `a_stock.py` 实现函数 → 在 `interface.py` 注册 → 在 `agent_utils.py` 暴露工具 → 在对应 Analyst 的 prompt 中引用。

### 数据来源
| 来源 | 协议/域名 | 数据 |
|------|-----------|------|
| mootdx | TCP 7709 | OHLCV K线、财务快照、F10 文本 |
| 腾讯财经 | qt.gtimg.cn | PE/PB/市值/换手率 |
| 东方财富 datacenter | datacenter-web.eastmoney | 龙虎榜、限售解禁、板块行情 |
| 东方财富 push2 | push2.eastmoney | 实时行情、资金流(分钟+日级) |
| 东方财富 np-weblist | eastmoney | 滚动新闻 |
| 新浪财经 | money.finance.sina | K线历史、财报三表 |
| 同花顺 | 10jqka.com.cn | EPS 一致预期、热股题材 |
| 财联社 | cls.cn | 全球财经快讯 |
| 百度股市通 | gushitong.baidu | 概念板块归属 |

### LLM 支持
OpenAI 兼容（统一用 `OpenAIClient`）：`openai`、`xai`、`deepseek`、`qwen`（dashscope）、`glm`（zhipu）、`ollama`、`openrouter`、`minimax`

独立客户端：`anthropic`、`google`（需 `[google]` 可选依赖）、`azure`

### 中文股票名解析链路
用户/LLM 输入 → `safe_ticker_component` 检测中文 → `resolve_ticker()` → `_build_name_code_map()`（mootdx 全市场映射，缓存）→ 返回 6 位代码

### 配置（`tradingagents/default_config.py`）
关键字段：`llm_provider`、`deep_think_llm`、`quick_think_llm`、`data_vendors`（vendor 路由）、`output_language`（默认 Chinese）、`checkpoint_enabled`（LangGraph 断点续跑）。
结果缓存默认落 `~/.tradingagents/`，可用环境变量 `TRADINGAGENTS_RESULTS_DIR` / `TRADINGAGENTS_CACHE_DIR` 覆盖。

## 开发规范
- 改动前先跑 `python -m pytest tests/ -v` 确保不破坏现有测试
- `safe_ticker_component`（`dataflows/utils.py`）是路径安全边界，任何绕过校验的改动必须慎重评估
- 数据层新增接口遵循 `interface.py` 的 vendor 路由模式（见上方"新增数据接口"步骤）
- deepseek-v4-flash 等模型在 tool call 时可能返回中文股票名，`safe_ticker_component` 已兜底转码，调试时注意区分

## 已知问题

### 依赖冲突
mootdx 锁死 httpx==0.25.2，与 langchain-google-genai 的 httpx>=0.28.1 冲突。已将 google-genai 移至可选依赖 `[google]`；`pip install -e .` 不再冲突。

### 待处理
- PR #18（hejingchi）：start_date 功能 + 主题切换 + Windows 字体。不建议直接 merge（与 v0.2.6 冲突），start_date 功能值得后续自行实现。

## Issue 归档
所有 GitHub Issue 的详细记录在 `issues/` 文件夹，包含问题描述、根因分析、修复方案和当前状态。

## 相关项目
- 上游 [TauricResearch/TradingAgents](https://github.com/TauricResearch/TradingAgents) — 原版框架

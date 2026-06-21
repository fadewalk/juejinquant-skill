# juejinquant-skill

> **WorkBuddy 上首个把"掘金量化 GM SDK"做成 AI Agent 可调用技能的开源项目** —— 自然语言生成策略 + 完整 API 字典 + 28 条实战踩坑经验，三件套打包给你。

[![License: Personal Use](https://img.shields.io/badge/License-Personal_Use-blue.svg)](#-许可证)
[![Python](https://img.shields.io/badge/Python-3.7%2B-blue.svg)](https://www.python.org/)
[![GM SDK](https://img.shields.io/badge/GM_SDK-v3.0.0-orange.svg)](https://www.myquant.cn/)
[![WorkBuddy Skill](https://img.shields.io/badge/WorkBuddy-Skill-purple.svg)](https://www.codebuddy.cn/)
[![Maintained](https://img.shields.io/badge/Maintained-yes-green.svg)]()

[掘金量化官网](https://www.myquant.cn/) · [API 文档](https://www.myquant.cn/docs/) · [WorkBuddy](https://www.codebuddy.cn/) · [报告问题](https://github.com/fadewalk/juejinquant-skill/issues)

---

## ✨ 项目特色

### 1️⃣ 意图路由 —— 按"你想干什么"加载最少上下文

`SKILL.md` 内置意图路由表，用户问"帮我写个 XX 策略"、"XX API 怎么用"、"代码报错了"会被分别路由到对应的 `examples-*.md` / 章节 / `pitfalls.md`，**AI Agent 加载的 token 减少 60-80%**。

### 2️⃣ 28 条实战踩坑经验（`pitfalls.md`）

从真实回测里提炼的"防弹"指南，覆盖：

- `cost_price=0`、`bar.get()` 返回 `None`、`history_n` 失效
- `order_target_*` 用 `position_side`，`order_volume` 用 `position_effect`
- 市价单资金不足变 `Cancelling` 而非 `Rejected`
- 回测状态码有 `10`（内部中间态）需兼容
- ROE 字段是 `roe_weight_avg`、流通股本是 `circ_shr`
- 18:30 前只能回测到上一交易日、5 开头 ETF = 沪市、1 开头 = 深市

### 3️⃣ 6 大策略模板 —— 可直接运行

| 策略类型 | references | scripts |
|---------|-----------|---------|
| 双均线（基础+高级） | `examples-dual-ma.md` | `strategy_ma_cross.py`、`strategy_xinyisheng_ma.py` |
| 海龟 CTA 趋势跟踪 | `examples-cta-turtle.md` | — |
| 配对交易（统计套利） | `examples-pair-trading.md` | — |
| 多因子选股 | `examples-multi-factor.md` | — |
| 风险管理 | `examples-risk-mgmt.md` | — |
| 行业轮动 / ETF 动量 | `examples-industry-rotation.md` | `strategy_etf_rotation.py`、`strategy_etf_momentum*.py`、`strategy_high_dividend.py` |

### 4️⃣ 完整 API 字典 —— 27 篇 references

覆盖 GM SDK 全部章节：行情订阅、订单/账户/持仓、融资融券、算法单（TWAP/VWAP/Iceberg）、可转债、L2 数据、付费数据 API、字段定义、枚举常量、`context` 对象。付费数据 API（16-21）单独成章，区分基础和高级接口，避免误用付费功能被扣费。

### 5️⃣ 强制 `strategy_id` + 一键运行

```bash
python scripts/run_strategy.py \
    --strategy scripts/strategy_ma_cross.py \
    --strategy-id my_ma_strategy_v1 \
    --mode backtest \
    --token YOUR_TOKEN
```

回测结果**自动持久化**到掘金终端后台（https://www.myquant.cn → 策略列表），可复盘完整的收益曲线、夏普、回撤、持仓明细。

### 6️⃣ 13 个可运行脚本 —— 开箱即用

`scripts/` 下含 5 类策略模板 + 4 个 API 烟测脚本（账户/全部/付费/财务/新 API）+ 事件回调测试 + `check_import.py` 环境检测 + `demo_import.py` 最小 demo。新手第一天就能跑通。

---

## 🚀 快速开始

### 1. 安装 WorkBuddy Skill

```bash
# 用户级（推荐）
cp -r juejinquant ~/.workbuddy/skills/

# 项目级
cp -r juejinquant <your-project>/.workbuddy/skills/
```

### 2. 安装掘金量化 SDK

```bash
pip install gm.api
```

### 3. 设置 Token（去掘金终端申请）

```python
# 进入掘金终端 → 策略 → 新建策略 → 复制 strategy_id 和 token
# 或参考：https://www.myquant.cn/docs/
```

### 4. 跑第一个策略

```bash
# 环境检测
python scripts/check_import.py

# 跑双均线回测
python scripts/run_strategy.py \
    --strategy scripts/strategy_ma_cross.py \
    --strategy-id my_first_strategy \
    --mode backtest \
    --token YOUR_GM_TOKEN
```

回测完成后登录 https://www.myquant.cn → 策略列表 → 查看完整绩效。

### 5. 在 WorkBuddy 中使用

直接用自然语言：

> "帮我写一个 A 股 ETF 动量轮动策略，10 个交易日调仓一次"

> "GM SDK 里 order_target_volume 怎么用？"

> "我的 history_n 一直返回 None，怎么排查？"

Skill 会自动路由到对应的章节或踩坑库，**无需手动翻文档**。

---

## 📁 项目结构

```
juejinquant/
├── SKILL.md                          # 主技能说明（必读，含意图路由表）
├── README.md                         # 本文件
├── agents/
│   └── openai.yaml                   # OpenAI 兼容 agent 配置
├── assets/                           # 资源占位目录
├── references/                       # 27 篇 GM SDK API 参考文档
│   ├── 01-quick-start.md             # 快速开始
│   ├── 02-core-functions.md          # 核心函数
│   ├── 03-subscribe-events.md        # 行情订阅与事件
│   ├── 04-market-data.md             # 行情数据
│   ├── 05-l2-data.md                 # L2 行情
│   ├── 06-symbol-info.md             # 代码信息
│   ├── 07-trading-dates.md           # 交易日历
│   ├── 08-order-api.md               # 订单 API
│   ├── 09-algo-order.md              # 算法单（TWAP/VWAP/Iceberg）
│   ├── 10-account-query.md           # 账户查询
│   ├── 11-bond-convertible.md        # 可转债
│   ├── 12-data-objects.md            # 数据对象
│   ├── 13-enums.md                   # 枚举常量
│   ├── 14-context.md                 # context 对象
│   ├── 15-user-guide.md              # 用户指南 + FAQ
│   ├── 16-premium-data-apis.md       # 付费数据 API（综合）
│   ├── 17-financial-data-fields.md   # 财务数据字段
│   ├── 18-stock-premium-apis.md      # 股票增值 API
│   ├── 18-user-guide.md              # 用户指南（备份）
│   ├── 19-fund-premium-apis.md       # 基金增值 API
│   ├── 20-cb-premium-apis.md         # 可转债增值 API
│   ├── 21-futures-premium-apis.md    # 期货增值 API
│   ├── examples-cta-turtle.md        # 海龟策略示例
│   ├── examples-dual-ma.md           # 双均线示例
│   ├── examples-industry-rotation.md # 行业轮动示例
│   ├── examples-multi-factor.md      # 多因子选股示例
│   ├── examples-pair-trading.md      # 配对交易示例
│   ├── examples-risk-mgmt.md         # 风险管理示例
│   ├── pitfalls.md                   # 28 条实战踩坑
│   └── quick-reference.md            # API 速查卡
└── scripts/                          # 可直接运行的策略 / 测试脚本
    ├── check_import.py               # 环境检测（SDK 装没装、token 通不通）
    ├── demo_import.py                # 最小可运行 demo
    ├── run_strategy.py               # 一键运行器
    ├── strategy_ma_cross.py          # 双均线
    ├── strategy_xinyisheng_ma.py     # 新易盛均线策略
    ├── strategy_etf_rotation.py      # ETF 轮动
    ├── strategy_etf_momentum.py      # ETF 动量
    ├── strategy_etf_momentum_rotation.py  # ETF 动量轮动
    ├── strategy_high_dividend.py     # 高股息策略
    ├── strategy_event_callbacks_test.py  # 事件回调全套测试
    ├── test_account_apis.py          # 账户 API 烟测
    ├── test_all_apis.py              # 全部 API 烟测
    ├── test_all_premium_apis.py      # 增值 API 烟测
    ├── test_financial_apis.py        # 财务 API 烟测
    └── test_new_apis.py              # 新 API 烟测
```

---

## 🎯 适用场景

- ✅ **A 股 / 期货 / 期权 / ETF / 可转债** 量化策略开发
- ✅ 历史行情**回测**、实时行情**订阅**、模拟盘 / 实盘**交易**
- ✅ **事件驱动**编程（`subscribe` / `schedule` / `on_tick` / `on_bar` / `on_order_status`）
- ✅ 订单 / 账户 / 持仓查询，**算法单**（TWAP / VWAP / Iceberg）
- ✅ 用 AI Agent（WorkBuddy / Claude / GPT）做**自然语言→策略**生成
- ✅ L2 行情、融资融券、可转债、付费数据 API 高级用法

---

## 🆚 与官方文档的差异化

| 维度 | 官方文档 | juejinquant-skill |
|------|---------|-------------------|
| 索引方式 | 按 API 函数字母序 | 按**用户意图**路由 |
| 实战经验 | 无 | **28 条踩坑库** |
| 代码示例 | 零散、签名级 | **6 大完整可运行模板** |
| 排错支持 | 论坛零散问答 | `pitfalls.md` 系统化收录 |
| AI Agent 适配 | 未设计 | **专为 WorkBuddy 优化** |

---

## 👥 适用人群

- ✅ 用自然语言做 A 股 / 期货 / ETF / 可转债策略的**研究员与量化开发者**
- ✅ 想用 AI Agent（WorkBuddy / Claude / GPT）做量化策略生成的**团队负责人**
- ✅ 需要查 GM SDK API 但又怕踩坑的**中高级用户**
- ✅ 想把 AI Agent 接入掘金终端的**个人投资者**

---

## 🤝 贡献

欢迎提 Issue / Pull Request 补充：

- 新的策略模板（CTA / 套利 / 期权希腊字母 / 机器学习预测等）
- 新的踩坑案例（在 `pitfalls.md` 里追加）
- 新的 references 章节翻译或补充

---

## 📊 仓库数据

- **47 个源文件** + README.md + .gitignore + assets/
- **27 篇 references** + 6 篇 examples + 13 个 scripts
- **版本**：v3.0.0（架构级重构，与 gm-quant v2.1.0 / myquant v1.2.0 深度融合）

---

## 📜 许可证

**仅供个人学习与研究使用。** 掘金量化 SDK 版权归原作者所有，禁止用于商业用途。

本技能涉及的 GM SDK API 名称、参数定义、调用方式均以 https://www.myquant.cn/docs/ 为准；如官方文档更新与本仓库内容冲突，以官方文档为准。

---

## 🔗 相关链接

- 掘金量化官网：https://www.myquant.cn/
- 掘金量化 API 文档：https://www.myquant.cn/docs/
- WorkBuddy 官网：https://www.codebuddy.cn/
- 上游 gm-quant v2.1.0
- 上游 myquant v1.2.0
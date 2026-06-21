# juejinquant-skill

掘金量化 (juejinquant / MyQuant / GoldMiner) WorkBuddy Skill — 自然语言策略生成 + 完整 GM SDK API 参考。

## 目录结构

```
juejinquant/
├── SKILL.md                # 主技能说明（必读）
├── agents/openai.yaml      # OpenAI 兼容 agent 配置
├── assets/                 # 资源占位目录
├── references/             # 18+ 篇 GM SDK API 参考文档
│   ├── 01-quick-start.md
│   ├── 02-core-functions.md
│   ├── 03-subscribe-events.md
│   ├── 04-market-data.md
│   ├── 05-l2-data.md
│   ├── 06-symbol-info.md
│   ├── 07-trading-dates.md
│   ├── 08-order-api.md
│   ├── 09-algo-order.md
│   ├── 10-account-query.md
│   ├── 11-bond-convertible.md
│   ├── 12-data-objects.md
│   ├── 13-enums.md
│   ├── 14-context.md
│   ├── 15-user-guide.md
│   ├── 16-premium-data-apis.md
│   ├── 17-financial-data-fields.md
│   ├── 18-stock-premium-apis.md
│   ├── 19-fund-premium-apis.md
│   ├── 20-cb-premium-apis.md
│   ├── 21-futures-premium-apis.md
│   ├── examples-cta-turtle.md
│   ├── examples-dual-ma.md
│   ├── examples-industry-rotation.md
│   ├── examples-multi-factor.md
│   ├── examples-pair-trading.md
│   ├── examples-risk-mgmt.md
│   ├── pitfalls.md
│   └── quick-reference.md
└── scripts/                # 可直接运行的策略 / 测试脚本
    ├── check_import.py
    ├── demo_import.py
    ├── run_strategy.py
    ├── strategy_etf_momentum.py
    ├── strategy_etf_momentum_rotation.py
    ├── strategy_etf_rotation.py
    ├── strategy_event_callbacks_test.py
    ├── strategy_high_dividend.py
    ├── strategy_ma_cross.py
    ├── strategy_xinyisheng_ma.py
    ├── test_account_apis.py
    ├── test_all_apis.py
    ├── test_all_premium_apis.py
    ├── test_financial_apis.py
    └── test_new_apis.py
```

## 适用场景

- A 股 / 期货 / 期权 / ETF / 可转债 量化策略开发
- 历史行情回测、实时订阅、模拟盘 / 实盘交易
- 事件驱动编程（subscribe / schedule / on_tick / on_bar / on_order_status）
- 订单 / 账户 / 持仓查询，算法单（TWAP / VWAP / Iceberg）

## 安装到 WorkBuddy

```bash
# 用户级（推荐）
cp -r juejinquant ~/.workbuddy/skills/

# 项目级
cp -r juejinquant <your-project>/.workbuddy/skills/
```

## 许可

仅供个人学习与研究使用。掘金量化 SDK 版权归原作者所有。
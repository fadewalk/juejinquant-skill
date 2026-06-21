---
name: juejinquant
description: >
  掘金量化（juejinquant / MyQuant / GoldMiner）统一技能 — 自然语言策略生成 + 完整 SDK API 参考。
  涵盖 order_volume、subscribe、history、set_token、history_n、current、schedule、algo_order、
  stk_get_fundamentals、stk_get_industry_constituents、bnd_get_analysis、fnd_get_etf_constituents、
  fut_get_contract_info 等全部 API。当用户提到掘金量化、gm、gm.api、juejinquant、MyQuant、
  量化策略、回测、订阅行情、历史行情、下单、委托、持仓、定时任务、财务数据、PE/PB、成分股、
  行业分类、可转债、ETF、期货合约、A 股回测、CTA、配对交易、多因子选股、行业轮动时自动加载。
  集成 gm-quant 的自然语言策略引擎 + myquant 的 API 字典 + 28 条实战踩坑经验。
version: 3.0.0
homepage: https://www.myquant.cn
---

# 掘金量化（juejinquant）统一技能

## 定位

你是**掘金量化平台的统一策略助手**。融合两个原始技能：

- **gm-quant**：自然语言 → 策略生成 + 一键运行
- **myquant**：完整 SDK API 参考 + 教程式示例

用户用中文描述需求，你负责：
1. **理解意图** → 区分是"查 API"还是"生成策略"还是"排查问题"
2. **路由到对应 references** → 加载最少必要的上下文
3. **生成代码 / 给出方案** → 输出可直接运行的完整策略 `.py` 文件
4. **执行运行** → 调用 `scripts/run_strategy.py` 一键启动回测或实盘

## 核心原则

1. **必须先 set_token**：纯数据查询（非策略 run）场景下，代码开头必须调用 `set_token('your_token')`。
2. **symbol 格式**：`交易所代码.证券代码`，如 `SHSE.600000`、`SZSE.000001`，**严格区分大小写**。
3. **gm 包通过掘金终端连接**：终端必须保持打开，否则接口会超时或报错。
4. **两种模式**：`MODE_LIVE=1`（实时/仿真）、`MODE_BACKTEST=2`（回测）；`run()` 函数启动策略。
5. **数据查询不需要 run**：仅用 `set_token` 后直接调用数据函数即可。
6. **A 股 100 股整数倍**：`order_value` 会自动取整；`order_volume` 必须是 100 的倍数。

## 🚀 用户工作流（自然语言→运行）

### 第 0 步：路由判断

根据用户意图，加载对应的 references：

| 用户意图 | 加载 references |
|---------|----------------|
| "帮我写个 XX 策略" | `examples-*.md`（找对应类型）+ `08-order-api.md` + `pitfalls.md` |
| "XX API 怎么用" | 对应章节（如查 history → `04-market-data.md`） |
| "代码报错了" | `pitfalls.md` + `15-user-guide.md` |
| "怎么查财务数据" | `16-premium-data-apis.md` + `17-financial-data-fields.md` |
| "回测结果不对" | `pitfalls.md` 第 12、15、25 条 |

### 第 1 步：确认 Strategy ID（生成策略时必填！）

**每次生成策略前，必须向用户索要 `strategy_id`。**

`strategy_id` 是策略在掘金终端中的唯一标识。填写后：
- 回测结果**持久化**到掘金终端后台
- 用户登录 [掘金终端网页](https://www.myquant.cn) → 策略列表 → 查看完整的**绩效分析图表**
  （收益曲线、回撤分析、夏普比率、持仓明细等）

> **交互方式**：如果用户没有主动提供 strategy_id，在生成代码前询问：
> "请给我一个 **strategy_id**（英文/数字/下划线），用于在掘金终端标识这个策略。
> 填完后你可以在终端网页上看到绩效分析图表。例如：`ma_cross_600519`、`momentum_v1`"

| 场景 | 处理方式 |
|------|---------|
| 用户提供了 strategy_id | 直接使用 |
| 用户没提供 | **必须追问**，不能自己编造一个默认值后静默使用 |
| 用户说"随便起一个" | 根据策略特征起一个有意义的名字（如 `dual_ma_kweichow`） |

### 第 2 步：理解用户意图（生成策略时）

当用户用自然语言描述策略时，按以下维度提取信息：

| 维度 | 需确认的信息 | 默认值（如未明确说明） |
|------|-------------|---------------------|
| **strategy_id** | 策略在掘金终端的标识（**必须用户提供**） | 无默认，必须询问 |
| **标的池** | 哪些股票/指数/期货？ | 沪深300成分股 |
| **时间频率** | 日线/分钟线/tick？ | 日线 `1d` |
| **买入信号** | 什么条件买入？（均线/指标/事件） | 必须明确，不能猜测 |
| **卖出信号** | 什么条件卖出？ | 必须明确，不能猜测 |
| **仓位管理** | 全仓/固定金额/比例/等权 | 等权分配 |
| **止损止盈** | 有无？阈值多少？ | 无 |
| **回测区间** | 开始~结束日期 | 最近1年 |
| **初始资金** | 多少钱？ | 100万 |
| **运行模式** | 回测还是实盘？ | 先回测 |

> ⚠️ **如果用户描述模糊（如"帮我做个赚钱的策略"），必须追问具体条件后再生成代码。**

### 第 3 步：生成策略文件

使用下方**标准策略模板**生成完整 `.py` 文件，保存到用户的输出目录：

```python
"""
策略名称：{name}
策略描述：{description}
生成时间：{date}
"""

import sys, os, io
sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8', errors='replace')

from gm.api import *

# ============================================================
# 配置区 —— 用户可通过修改此处调整策略参数
# ============================================================
SYMBOLS = 'SHSE.600519,SZSE.000001'      # 标的（逗号分隔）
FREQUENCY = '1d'                          # K线周期：1d/60s/300s/tick
COUNT = 20                                # 订阅K线数量（context.data滑窗大小）

# 交易参数
ORDER_TYPE = OrderType_Market              # 下单方式：Market(市价) / Limit(限价)
POSITION_PCT = 0.2                        # 单只股票仓位占比（0~1）

# 回测参数
BACKTEST_START = '2024-01-02 09:30:00'
BACKTEST_END   = '2025-12-31 15:30:00'
INITIAL_CASH   = 1000000                  # 初始资金
COMMISSION     = 0.00025                  # 手续费率
SLIPPAGE       = 0.001                    # 滑点


# ============================================================
# 策略逻辑
# ============================================================

def init(context):
    """初始化：订阅行情"""
    log.info(f'策略启动 | 标的:{SYMBOLS} | 周期:{FREQUENCY}')
    subscribe(symbols=SYMBOLS, frequency=FREQUENCY, count=COUNT)
    context.last_signal = {}


def on_bar(context, bars):
    """每根K线触发"""
    for bar in bars:
        symbol = bar['symbol']
        try:
            _handle_bar(context, symbol)
        except Exception as e:
            log.error(f'处理{symbol}异常: {e}')


def on_tick(context, tick):
    """tick级别回调"""
    pass


def _handle_bar(context, symbol):
    """单只标的策略逻辑"""
    data = context.data(symbol=symbol, frequency=FREQUENCY, count=COUNT)
    if data is None or len(data) < COUNT:
        return

    all_positions = get_position()
    position = None
    if all_positions:
        for p in all_positions:
            sym = p.get('symbol') if isinstance(p, dict) else (p.symbol if hasattr(p, 'symbol') else None)
            if sym == symbol:
                position = p
                break

    # 【策略核心】在此处实现买卖信号
    close = data['close'].tolist()
    ma_short = sum(close[-5:]) / 5
    ma_long  = sum(close[-20:]) / 20
    prev_ma5 = sum(close[-6:-1]) / 5 if len(close) >= 6 else ma_short
    prev_ma20 = sum(close[-26:-6]) / 20 if len(close) >= 27 else ma_long

    buy_signal  = (prev_ma5 <= prev_ma20) and (ma_short > ma_long)
    sell_signal = (prev_ma5 >= prev_ma20) and (ma_short < ma_long)

    current_price = close[-1]
    cash_info = get_cash()

    if buy_signal and not position:
        available = cash_info.available
        order_value = available * POSITION_PCT
        if order_value > 10000:
            order_value(symbol, order_value,
                        side=OrderSide_Buy,
                        position_effect=PositionEffect_Open,
                        order_type=ORDER_TYPE)
            print(f'[买入] {symbol} 价格={current_price:.2f}')

    elif sell_signal and position:
        order_target_volume(symbol, 0,
                            position_side=PositionSide_Long,
                            order_type=ORDER_TYPE)
        print(f'[卖出] {symbol} 价格={current_price:.2f}')


def handle_error(context, error_code, error_msg, **kwargs):
    """错误处理"""
    log.error(f'策略异常 [{error_code}]: {error_msg}')


if __name__ == '__main__':
    TOKEN = os.environ.get('GM_TOKEN', '') or ''
    MODE = os.environ.get('GM_RUN_MODE', 'backtest')
    STRATEGY_ID = os.environ.get('GM_STRATEGY_ID', '') or 'my_strategy'
    START = os.environ.get('GM_BACKTEST_START', BACKTEST_START)
    END = os.environ.get('GM_BACKTEST_END', BACKTEST_END)
    CASH = float(os.environ.get('GM_INITIAL_CASH', str(INITIAL_CASH)))

    mode = MODE_LIVE if MODE.lower() in ('live', 'realtime') else MODE_BACKTEST

    run(
        strategy_id=STRATEGY_ID,
        filename=__file__[:__file__.rfind('.')] if '.' in __file__ else __file__,
        mode=mode,
        token=TOKEN,
        backtest_start_time=START,
        backtest_end_time=END,
        backtest_initial_cash=CASH,
        backtest_commission_ratio=COMMISSION,
        backtest_slippage_ratio=SLIPPAGE,
        backtest_adjust=ADJUST_PREV,
    )
```

### 第 4 步：执行策略

```bash
python scripts/run_strategy.py --strategy <策略文件路径> --strategy-id <你的策略ID> [--mode backtest|live] [--token YOUR_TOKEN]
```

> **`--strategy-id` 必填**：填写后回测结果会持久化到掘金终端，登录终端网页即可查看绩效分析图表。

## 📚 References 索引（按使用场景路由）

### 快速入门 & 核心概念

| 文件 | 内容 | 何时加载 |
|------|------|---------|
| `references/01-quick-start.md` | 快速开始、策略架构、运行模式 | 用户刚接触掘金 |
| `references/02-core-functions.md` | `run`、`set_token`、`stop`、`schedule`、`timer` | 查核心函数签名 |
| `references/14-context.md` | context 对象详解 | 需要用 context.xxx 时 |
| `references/quick-reference.md` | 最常用 API 速查卡 | 用户想快速上手 |

### 行情订阅 & 数据查询

| 文件 | 内容 | 何时加载 |
|------|------|---------|
| `references/03-subscribe-events.md` | `subscribe`、`unsubscribe`、`on_tick`、`on_bar`、`on_l2*` | 订阅行情时 |
| `references/04-market-data.md` | `current`、`last_tick`、`current_price`、`history`、`history_n`、`context.data` | 查历史/实时数据 |
| `references/05-l2-data.md` | L2 行情查询接口（付费） | 用户需要 L2 数据 |
| `references/06-symbol-info.md` | 标的信息查询 API | 查合约/股票基础信息 |
| `references/07-trading-dates.md` | 交易日历 API | 需要交易日判断 |

### 交易 & 账户

| 文件 | 内容 | 何时加载 |
|------|------|---------|
| `references/08-order-api.md` | 下单 API 全集 | 下单/撤单时 |
| `references/09-algo-order.md` | 算法单 API | TWAP/VWAP 算法单 |
| `references/10-account-query.md` | 账户/持仓/委托查询 | 查账户状态 |
| `references/11-bond-convertible.md` | 可转债交易 API | 可转债策略 |

### 数据对象 & 枚举

| 文件 | 内容 | 何时加载 |
|------|------|---------|
| `references/12-data-objects.md` | 数据对象字段定义 | 解析返回数据时 |
| `references/13-enums.md` | 枚举常量速查 | 写代码时查枚举值 |

### 高级数据 API（付费）

| 文件 | 内容 | 何时加载 |
|------|------|---------|
| `references/16-premium-data-apis.md` | 增值数据 API 速查合集 | 查财务/行业/资金流 |
| `references/17-financial-data-fields.md` | 财务数据字段定义 | 解析财务数据时 |
| `references/18-stock-premium-apis.md` | 股票增值数据完整文档 | 行业/板块/分红/龙虎榜 |
| `references/19-fund-premium-apis.md` | 基金增值数据 | ETF 成分股/规模 |
| `references/20-cb-premium-apis.md` | 可转债增值数据 | 转股价/赎回/回售 |
| `references/21-futures-premium-apis.md` | 期货增值数据 | 成交持仓排名/仓单 |

### 完整策略示例（6 大类型）

| 文件 | 内容 | 何时加载 |
|------|------|---------|
| `references/examples-dual-ma.md` | 双均线策略（基础 + 高级两种风格） | 用户要均线策略 |
| `references/examples-cta-turtle.md` | 海龟交易法（CTA 期货） | 用户要趋势跟踪 |
| `references/examples-pair-trading.md` | 配对交易（统计套利） | 用户要均值回归 |
| `references/examples-multi-factor.md` | 多因子选股 | 用户要选股策略 |
| `references/examples-risk-mgmt.md` | 通用风险管理模块 | 需要风控逻辑 |
| `references/examples-industry-rotation.md` | 行业轮动 + 纯数据研究 | 用户要 ETF 轮动 |

### 实战经验

| 文件 | 内容 | 何时加载 |
|------|------|---------|
| `references/15-user-guide.md` | 用户指南（常见问题） | 用户问通用问题 |
| `references/pitfalls.md` | **28 条实测踩坑经验** | **生成代码前必读！遇到报错时必读！** |

## 🔥 交易 API 补充

### 融资融券（信用交易）

融资融券交易需在信用账户下操作，使用 `credit_` 前缀函数：

```python
credit_buying_on_margin(symbol, volume, price=0, order_type=OrderType_Market, position_effect=PositionEffect_Open)
credit_short_selling(symbol, volume, price=0, order_type=OrderType_Market, position_effect=PositionEffect_Open)
credit_buying_on_repayment(symbol, volume, price=0, order_type=OrderType_Market)
credit_selling_on_repayment(symbol, volume, price=0, order_type=OrderType_Market)
credit_direct_repayment(amount)
credit_direct_return_securities(symbol, volume)
credit_get_collateral_instruments()
credit_get_borrowable_instruments()
credit_get_collateral_ratio(symbol)
credit_get_concentrate_limit(symbol)
credit_get_margin_ratio(symbol)
credit_get_max_volume(symbol, side)
```

> ⚠️ 融资融券功能需要开通信用账户，且掘金终端需切换到信用交易模式。

### 批量下单

```python
orders = [
    {'symbol': 'SHSE.600519', 'volume': 100, 'side': OrderSide_Buy,
     'position_effect': PositionEffect_Open, 'order_type': OrderType_Market},
    {'symbol': 'SZSE.000001', 'volume': 200, 'side': OrderSide_Buy,
     'position_effect': PositionEffect_Open, 'order_type': OrderType_Market},
]
order_batch(orders)
```

### 撤单

```python
order_cancel(cl_ord_id)         # cl_ord_id 从 get_orders() 或 on_order_status 回调中获取
order_cancel_all()              # 撤销全部未成交委托
get_unfinished_orders()         # 查询未成交委托
```

> ⚠️ `order_cancel` 的参数是 `cl_ord_id`（客户端订单ID），**不是** `order_id`。

### 特殊交易函数

```python
ipo_buy(symbol, volume, price=0, order_type=OrderType_Limit)               # 新股申购
fund_etf_buy(symbol, volume, price=0, order_type=OrderType_Market)         # ETF 申购
fund_etf_redemption(symbol, volume, price=0, order_type=OrderType_Market)  # ETF 赎回
fund_subscribing(symbol, volume, price=0)                                  # 场外基金认购
fund_buy(symbol, volume, price=0)                                          # 场外基金申购
fund_redemption(symbol, volume, price=0)                                   # 场外基金赎回
bond_reverse_repurchase_agreement(symbol, volume, price=0, order_type=OrderType_Limit)  # 国债逆回购
```

> ⚠️ 国债逆回购的 `volume` 单位是**张**（1张=1000元面值），`price` 是年化利率。

### 动态参数（终端 UI 可调）

```python
def init(context):
    add_parameter(key='ma_short', value=5, min=1, max=100, step=1, name='短期均线周期')
    add_parameter(key='ma_long', value=20, min=1, max=200, step=1, name='长期均线周期')

def on_parameter(context, parameter):
    key = parameter['key']
    value = parameter['value']
    if key == 'ma_short':
        context.ma_short_period = value

def on_bar(context, bars):
    ma_short = get_parameter(key='ma_short')
```

### 连接事件

适用于实盘/仿真模式：

```python
def on_market_data_connected(context):    log.info('行情服务已连接')
def on_market_data_disconnected(context): log.info('行情服务已断开')
def on_trade_data_connected(context):     log.info('交易服务已连接')
def on_trade_data_disconnected(context):  log.info('交易服务已断开')
```

## 🏛️ 交易所代码表

| 代码 | 交易所 | 示例 |
|------|--------|------|
| SHSE | 上海证券交易所 | `SHSE.600000` |
| SZSE | 深圳证券交易所 | `SZSE.000001` |
| CFFEX | 中国金融期货交易所 | `CFFEX.IF2506` |
| SHFE | 上海期货交易所 | `SHFE.ag2506` |
| DCE | 大连商品交易所 | `DCE.m2509` |
| CZCE | 郑州商品交易所 | `CZCE.CF501` |
| INE | 上海国际能源交易中心 | `INE.sc2506` |
| GFEX | 广州期货交易所 | `GFEX.si2508` |

## 🔢 枚举常量表（速查）

```python
# 订单状态 OrderStatus
OrderStatus_New = 1                # 新建
OrderStatus_PartiallyFilled = 3    # 部分成交
OrderStatus_Filled = 4             # 全部成交
OrderStatus_Canceled = 5           # 已撤
OrderStatus_Rejected = 7           # 拒绝
OrderStatus_Cancelling = 8         # 待撤
# 注意：回测中可能出现未记录状态码 10（内部中间态），需兼容处理

# 订单类型 OrderType
OrderType_Market = 1               # 市价单
OrderType_Limit = 2                # 限价单

# 买卖方向 OrderSide
OrderSide_Buy = 1
OrderSide_Sell = 2

# 开平仓 PositionEffect（order_volume 用）
PositionEffect_Open = 1            # 开仓
PositionEffect_Close = 2           # 平仓
PositionEffect_CloseToday = 3      # 平今
PositionEffect_CloseYesterday = 4  # 平昨

# 持仓方向 PositionSide（order_target_* 用）
PositionSide_Long = 1
PositionSide_Short = 2

# 复权方式 AdjustType
ADJUST_NONE = 0                    # 不复权
ADJUST_PREV = 1                    # 前复权（回测常用）
ADJUST_POST = 2                    # 后复权

# 运行模式
MODE_LIVE = 1                      # 实时/仿真
MODE_BACKTEST = 2                  # 回测
```

## ⏱️ 数据频率与运行模式

**K线频率（frequency）**：

| 值 | 说明 |
|----|------|
| `tick` | 逐笔 |
| `60s` | 1分钟 |
| `300s` | 5分钟 |
| `900s` | 15分钟 |
| `1800s` | 30分钟 |
| `3600s` | 1小时 |
| `1d` | 日线 |

**运行模式**：

| 值 | 说明 |
|----|------|
| `MODE_LIVE = 1` | 实时/仿真模式，行情实时推送 |
| `MODE_BACKTEST = 2` | 回测模式，数据按时间序列回放 |

## 🆕 新股申购 & 分红 & L2

### 新股申购

```python
ipo_get_quota(exchange='SHSE')                                  # 申购额度
ipo_get_instruments(trade_date='2025-01-15')                    # 可申购列表
ipo_get_match_number(symbol='SHSE.688001')                      # 配号
ipo_get_lot_info(symbol='SHSE.688001')                          # 中签信息
```

### 分红数据

```python
get_dividend(symbol='SHSE.600519', start_date='2020-01-01', end_date='2025-12-31', df=True)
# 字段：ex_date(除权日), record_date(登记日), pay_date(发放日),
#       cash_div(每股派息), bonus_share_r(送股比例), transfer_share_r(转增比例)
```

### L2 历史数据查询

```python
get_history_l2_transaction(symbol, start_time, end_time, fields=None, df=True)  # 逐笔成交
get_history_l2_order(symbol, start_time, end_time, fields=None, df=True)       # 逐笔委托
get_history_l2_queue(symbol, start_time, end_time, fields=None, df=True)       # 买卖盘口
```

> ⚠️ L2 数据接口为付费功能。

## 🔄 API 参数补充说明

### current() 回测 vs 实盘字段差异

```python
data = current(symbols='SHSE.600519,SZSE.000001')
# 通用字段：symbol, open, high, low, close, volume, amount, frequency, timestamp
# 实盘额外字段：bid_price/bid_volume(买价买量), ask_price/ask_volume(卖价卖量),
#               last_price(最新价), num_trades(成交笔数)
```

> ⚠️ 回测模式下 `current()` 只能查询已订阅标的，实盘模式可查询任意标的。

**⚠️ current() 实时模式调用频次限制（2026-05-19 起）**：
- 5分钟内最多调用 **100 次**
- 24小时内最多调用 **1000 次**
- **2026-06-01 起**：单次查询标的数量上限调整为 **50 个**

### last_tick / current_price — 轻量替代 current

```python
last_tick(symbols, fields="", include_call_auction=False)
current_price(symbols)
```

不受 `current()` 调用频次限制。`last_tick` 输入的 `symbols` **必须先通过 `subscribe` 订阅 tick**。

### fut_get_continuous_contracts — 查询连续合约

```python
fut_get_continuous_contracts(csymbol, start_date="", end_date="")
```

**csymbol 后缀规则**：

| 后缀 | 含义 | 示例 |
|:----:|------|------|
| 无 | 主力连续 | `CFFEX.IM` |
| 22 | 次主力连续 | `CFFEX.IM22` |
| 00 | 当月连续 | `CFFEX.IM00` |
| 01 | 下月连续 | `CFFEX.IM01` |
| 02 | 下季连续 | `CFFEX.IM02` |
| 99 | 加权指数 | `CFFEX.IM99` |

### history() 补充参数

```python
history(symbol, frequency, start_time=None, end_time=None, count=None,
        fields=None, skip_suspended=True, fill_missing=None, df=True)
# skip_suspended=True（默认）：跳过停牌日
# fill_missing='pre'/'post'：前值/后值填充缺失
```

### subscribe() wait_group 参数

```python
subscribe(symbols='SHSE.600519,SZSE.000001', frequency='60s', count=20, wait_group=True)
# 多标的同步：所有标的 bar 都到达后才触发一次 on_bar
```

### order_value / order_percent / order_target_percent

```python
order_value(symbol='SHSE.600519', value=50000, side=OrderSide_Buy,
            position_effect=PositionEffect_Open, order_type=OrderType_Market)

order_percent(symbol='SHSE.600519', percent=0.1, side=OrderSide_Buy,
              position_effect=PositionEffect_Open, order_type=OrderType_Market)

order_target_percent(symbol='SHSE.512000', percent=0.15,
                     position_side=PositionSide_Long, order_type=OrderType_Market)
# ETF 调仓推荐用 order_target_percent，更简洁
```

### 成交回报查询

```python
get_execution_reports(cl_ord_id=None, symbol=None, start_time=None,
                      end_time=None, position_side=None, limit=None, df=True)
```

## ⚠️ 必读：踩坑经验速览（完整版见 pitfalls.md）

**生成代码前必看，遇到报错先查这里！**

1. `run()` 参数是 `strategy_id` + `filename`（模块名，不是路径）
2. 用 `print()` 而非 `log.info()`
3. `context.data()` 返回 **DataFrame**，用 `data['close'].tolist()`
4. `get_position()` 不支持 `symbol=` 参数
5. `order_target_volume()` 用 `position_side`（不是 `position_effect`）
6. `order_volume()` 用 `position_effect`
7. Windows 脚本开头必须加 stdout 编码处理
8. A 股 T+1：当天买入不能当天卖出
9. `cl_ord_id` 不是下单函数的参数
10. 回测状态码有 `10`（内部中间态）需兼容
11. `on_execution_report` 回测 `exec_type` 是数字（如 15）
12. 回测中资金不足 → Cancelling 而非 Rejected
13. order 对象同时支持 dict 和属性访问
14. execrpt 对象同上
15. 市价单资金不足会变 Cancelling
16. `get_constituents` 没有 `df` 参数
17. 财务数据 `fields` 必填且 ≤20 个
18. ROE 字段是 `roe_weight_avg`（不是 `roe_waa`）
19. 流通股本是 `circ_shr`（不是 `float_shr`）
20. `eps_dil2`（deriv）vs `eps_dil`（prime）
21. `_pt` 后缀 = 截面，`date/trade_date` 参数
22. 期货/基金/可转债增值数据需付费
23. `stk_get_fundamentals_*_pt` 的 `date` 是发布日期
24. `*_pt` 函数的 `trade_date` 是交易日
25. 18:30 前只能回测到上一交易日
26. 实时模式没交易的排查：定时任务时间 / 期货主连 / 日线推送
27. `order_volume` 叫 `position_effect`，`order_target_*` 叫 `position_side`
28. 5开头 ETF = 沪市（SHSE），1开头 = 深市（SZSE）

## 🚀 一键运行

```bash
# 1. 检查依赖
python scripts/check_import.py

# 2. 跑策略（必须带 --strategy-id，回测结果会持久化到终端）
python scripts/run_strategy.py \
    --strategy scripts/strategy_ma_cross.py \
    --strategy-id my_ma_strategy_v1 \
    --mode backtest \
    --token YOUR_TOKEN
```

## 社区与支持

- 📖 [掘金量化官方文档](https://www.myquant.cn/docs/python/python_overview)
- 💬 [掘金量化社区](https://www.myquant.cn/community)
- 🌐 [掘金终端网页](https://www.myquant.cn)（登录后查看策略绩效图表）

---

> 💡 **融合说明**：本技能由 gm-quant v2.1.0 + myquant v1.2.0 融合而来，保留全部 API 参考、6 大策略示例、28 条踩坑经验。版本号跳到 v3.0.0 表示重大架构变更。
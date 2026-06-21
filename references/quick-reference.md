# 掘金量化快速参考卡

> 来源：myquant (大佬量化) QUICK_REFERENCE.md — 浓缩最常用的 API 速查。

## 初始化与认证

```python
from gm.api import *
set_token("your_token_here")
```

## 订阅行情

```python
# 订阅日K线
subscribe(symbols="SHSE.600000", frequency="1d")
# 订阅分钟K线
subscribe(symbols="SHSE.600000", frequency="60s")
# 订阅多只股票
subscribe(symbols="SHSE.600000,SZSE.000001", frequency="1d")
# 订阅 Tick
subscribe(symbols="SHSE.600000", frequency="tick")
```

## 行情事件回调

```python
def on_bar(context, bars):
    """K线回调"""
    bar = bars[0]
    print(f"代码={bar.symbol}, 收盘={bar.close}, 成交量={bar.volume}")

def on_tick(context, tick):
    """Tick回调"""
    print(f"代码={tick.symbol}, 最新价={tick.price}")
```

## 下单操作

```python
# 按数量下单（买入100股）
order_volume(symbol="SHSE.600000", volume=100,
             side=OrderSide_Buy, order_type=OrderType_Market,
             position_effect=PositionEffect_Open)

# 按比例下单（用30%资金买入）
order_percent(symbol="SHSE.600000", percent=0.3,
              side=OrderSide_Buy, order_type=OrderType_Market,
              position_effect=PositionEffect_Open)

# 目标仓位调整（调整到50%仓位）
order_target_percent(symbol="SHSE.600000", percent=0.5,
                     order_type=OrderType_Market, position_effect=PositionEffect_Open)

# 清仓
order_target_percent(symbol="SHSE.600000", percent=0,
                     order_type=OrderType_Market, position_effect=PositionEffect_Close)
```

## 历史数据查询

```python
# 查询最近N条数据
df = history_n(symbol="SHSE.600000", frequency="1d", count=20,
               fields="open,high,low,close,volume")

# 查询指定时间段
df = history(symbol="SHSE.600000", frequency="1d",
             start_time="2024-01-01", end_time="2024-12-31",
             fields="open,high,low,close,volume")
```

## 持仓查询

```python
def on_bar(context, bars):
    account = context.account()

    # 查询所有持仓
    positions = account.positions()
    for pos in positions:
        print(f"{pos.symbol}: 数量={pos.volume}, 可用={pos.available}, 成本={pos.vwap}")

    # 查询单只股票持仓
    pos = account.position(symbol="SHSE.600000", side=PositionSide_Long)
    if pos:
        print(f"持仓数量: {pos.volume}")
```

## 代码格式说明

| 交易所 | 代码前缀 | 示例 |
|--------|---------|------|
| 上交所 | `SHSE` | `SHSE.600000` |
| 深交所 | `SZSE` | `SZSE.000001` |
| 中金所 | `CFFEX` | `CFFEX.IF2506` |
| 上期所 | `SHFE` | `SHFE.rb2501` |
| 大商所 | `DCE` | `DCE.m2405` |
| 郑商所 | `CZCE` | `CZCE.CF405` |
| 能源中心 | `INE` | `INE.sc2407` |
| 广期所 | `GFEX` | `GFEX.si2407` |

## 更多资源

- [掘金量化官方文档](https://www.myquant.cn/docs/python/python_overview)
- [掘金量化社区](https://www.myquant.cn/community)
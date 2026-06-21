# 双均线策略完整示例

经典 MA5/MA20 金叉死叉策略，覆盖最常见的入门级量化场景。

## 策略逻辑

- **买入信号**：MA5 上穿 MA20（金叉）
- **卖出信号**：MA5 下穿 MA20（死叉）
- **仓位**：单标的 90% 仓位

---

## 示例 1：基础版（gm-quant 风格）

```python
from gm.api import *

def init(context):
    subscribe(symbols='SHSE.600000', frequency='1d', count=21)
    context.symbol = 'SHSE.600000'

def on_bar(context, bars):
    hist = context.data(symbol=context.symbol, frequency='1d', count=20)
    if hist is None or len(hist) < 20:
        return
    closes = hist['close'].tolist()  # DataFrame 风格

    ma5 = sum(closes[-5:]) / 5
    ma20 = sum(closes) / 20
    price = bars[0]['close']

    # 查询持仓
    all_positions = get_position()
    has_position = any(
        (p.get('symbol') if isinstance(p, dict) else getattr(p, 'symbol', None)) == context.symbol
        for p in (all_positions or [])
    )

    # 金叉买入
    if ma5 > ma20 and not has_position:
        order_value(symbol=context.symbol, value=100000,
                    side=OrderSide_Buy, order_type=OrderType_Market,
                    position_effect=PositionEffect_Open)
        print(f'[买入] {context.symbol} 价格={price:.2f}')

    # 死叉卖出
    elif ma5 < ma20 and has_position:
        order_target_volume(symbol=context.symbol, volume=0,
                            position_side=PositionSide_Long,
                            order_type=OrderType_Market)
        print(f'[卖出] {context.symbol} 价格={price:.2f}')

if __name__ == '__main__':
    run(strategy_id='ma_cross_basic',
        filename='main.py',
        mode=MODE_BACKTEST,
        token='your_token',
        backtest_start_time='2024-01-01 09:30:00',
        backtest_end_time='2024-12-31 15:00:00',
        backtest_initial_cash=1000000,
        backtest_commission_ratio=0.0003,
        backtest_slippage_ratio=0.001,
        backtest_adjust=ADJUST_PREV)
```

---

## 示例 2：context.account() 风格（myquant 风格）

```python
from gm.api import *

def init(context):
    subscribe(symbols='SHSE.600000', frequency='1d', count=21)
    context.symbol = 'SHSE.600000'

def on_bar(context, bars):
    hist = context.data(symbol=context.symbol, frequency='1d', count=20)
    closes = [bar['close'] for bar in hist]

    if len(closes) < 20:
        return

    ma5 = sum(closes[-5:]) / 5
    ma20 = sum(closes) / 20
    price = bars[0]['close']

    # context.account() 风格查询持仓
    pos = context.account().position(symbol=context.symbol, side=PositionSide_Long)

    if ma5 > ma20 and (pos is None or pos['volume'] == 0):
        order_percent(symbol=context.symbol, percent=0.9,
                      side=OrderSide_Buy, order_type=OrderType_Limit,
                      position_effect=PositionEffect_Open, price=price)
        print(f'金叉买入 {context.symbol} 价格={price:.2f}')
    elif ma5 < ma20 and pos is not None and pos['volume'] > 0:
        order_close_all()
        print(f'死叉卖出 {context.symbol} 价格={price:.2f}')

def on_order_status(context, order):
    if order['status'] == OrderStatus_Filled:
        print(f"Executed: {order['symbol']} volume={order['filled_volume']} avg_price={order['filled_vwap']}")

def on_backtest_finished(context, indicator):
    print(f"Backtest completed: return={indicator['pnl_ratio']:.2%}, "
          f"Sharpe ratio={indicator['sharpe_ratio']:.2f}, "
          f"max drawdown={indicator['max_drawdown']:.2%}")

if __name__ == '__main__':
    run(strategy_id='ma_cross_account',
        filename='main.py',
        mode=MODE_BACKTEST,
        token='your_token',
        backtest_start_time='2023-01-01 09:30:00',
        backtest_end_time='2024-06-30 15:00:00',
        backtest_initial_cash=1000000,
        backtest_commission_ratio=0.0003,
        backtest_slippage_ratio=0.001,
        backtest_adjust=ADJUST_PREV)
```

---

## 关键要点

| 项 | 注意事项 |
|----|---------|
| 持仓查询 | `get_position()` 不支持 `symbol` 参数，需遍历查找 |
| 卖出函数 | `order_target_volume(symbol, 0, position_side=PositionSide_Long)` |
| 数据结构 | `context.data()` 返回 DataFrame，用 `.tolist()` 访问 |
| A 股最小 | 100 股整数倍（`order_value` 自动取整） |
| T+1 | 当天买入次日才能卖出 |
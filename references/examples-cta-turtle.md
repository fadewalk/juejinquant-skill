# CTA 海龟交易法完整示例

经典的趋势跟踪策略，适用于期货市场。N 日突破入场 + ATR 动态仓位。

## 策略逻辑

- **入场**：价格突破 N1 日最高价做多，跌破 N1 日最低价做空
- **出场**：反向突破 N2 日通道平仓
- **仓位**：基于 ATR 的风险百分比定价

---

## 完整代码

```python
from gm.api import *
import numpy as np

def init(context):
    context.symbol = 'CFFEX.IF2506'   # 沪深300股指期货
    context.entry_period = 20          # 入场通道周期（20日高低点）
    context.exit_period = 10           # 出场通道周期（10日高低点）
    context.atr_period = 20            # ATR 计算周期
    context.risk_ratio = 0.01          # 单笔风险占比（总资金1%）

    subscribe(symbols=context.symbol, frequency='1d', count=context.entry_period + 1)

def on_bar(context, bars):
    hist = context.data(symbol=context.symbol, frequency='1d', count=context.entry_period)
    if hist is None or len(hist) < context.entry_period:
        return

    highs = hist['high'].tolist()
    lows = hist['low'].tolist()
    closes = hist['close'].tolist()

    # Donchian 通道
    entry_high = max(highs[-context.entry_period:])    # 20日最高（多头突破）
    entry_low = min(lows[-context.entry_period:])      # 20日最低（空头突破）
    exit_high = max(highs[-context.exit_period:])      # 10日最高（空头止损）
    exit_low = min(lows[-context.exit_period:])        # 10日最低（多头止损）

    # ATR（平均真实波幅）
    tr_list = []
    for i in range(1, len(closes)):
        tr = max(highs[i] - lows[i],
                 abs(highs[i] - closes[i-1]),
                 abs(lows[i] - closes[i-1]))
        tr_list.append(tr)
    atr = sum(tr_list[-context.atr_period:]) / context.atr_period if len(tr_list) >= context.atr_period else 0

    current_price = closes[-1]

    # 查询持仓
    all_positions = get_position() or []
    pos_long = next((p for p in all_positions
                     if (p.get('symbol') if isinstance(p, dict) else getattr(p, 'symbol', None)) == context.symbol
                     and (p.get('side') if isinstance(p, dict) else getattr(p, 'side', None)) == PositionSide_Long), None)
    pos_short = next((p for p in all_positions
                      if (p.get('symbol') if isinstance(p, dict) else getattr(p, 'symbol', None)) == context.symbol
                      and (p.get('side') if isinstance(p, dict) else getattr(p, 'side', None)) == PositionSide_Short), None)

    has_long = pos_long is not None and (pos_long.get('volume') if isinstance(pos_long, dict) else pos_long.volume) > 0
    has_short = pos_short is not None and (pos_short.get('volume') if isinstance(pos_short, dict) else pos_short.volume) > 0

    # 动态仓位（基于ATR）
    cash_info = get_cash()
    nav = cash_info.nav if hasattr(cash_info, 'nav') else cash_info.available
    if atr > 0:
        unit_size = max(int(nav * context.risk_ratio / atr / 300), 1)  # IF 合约乘数300
    else:
        unit_size = 1

    # 入场信号
    if current_price > entry_high and not has_long:
        if has_short:
            order_volume(symbol=context.symbol, volume=unit_size,
                         side=OrderSide_Buy, order_type=OrderType_Market,
                         position_effect=PositionEffect_Close)
        order_volume(symbol=context.symbol, volume=unit_size,
                     side=OrderSide_Buy, order_type=OrderType_Market,
                     position_effect=PositionEffect_Open)
        print(f'突破做多: 价格={current_price:.2f}, 上轨={entry_high:.2f}, 手数={unit_size}')

    elif current_price < entry_low and not has_short:
        if has_long:
            order_volume(symbol=context.symbol, volume=unit_size,
                         side=OrderSide_Sell, order_type=OrderType_Market,
                         position_effect=PositionEffect_Close)
        order_volume(symbol=context.symbol, volume=unit_size,
                     side=OrderSide_Sell, order_type=OrderType_Market,
                     position_effect=PositionEffect_Open)
        print(f'突破做空: 价格={current_price:.2f}, 下轨={entry_low:.2f}, 手数={unit_size}')

    # 出场信号
    elif has_long and current_price < exit_low:
        order_volume(symbol=context.symbol, volume=unit_size,
                     side=OrderSide_Sell, order_type=OrderType_Market,
                     position_effect=PositionEffect_Close)
        print(f'多头止损: 价格={current_price:.2f}, 下轨={exit_low:.2f}')

    elif has_short and current_price > exit_high:
        order_volume(symbol=context.symbol, volume=unit_size,
                     side=OrderSide_Buy, order_type=OrderType_Market,
                     position_effect=PositionEffect_Close)
        print(f'空头止损: 价格={current_price:.2f}, 上轨={exit_high:.2f}')

def on_backtest_finished(context, indicator):
    print(f'\n回测结果:')
    print(f"  收益率: {indicator['pnl_ratio']:.2%}")
    print(f"  年化: {indicator['pnl_ratio_annual']:.2%}")
    print(f"  夏普: {indicator['sharpe_ratio']:.2f}")
    print(f"  最大回撤: {indicator['max_drawdown']:.2%}")
    print(f"  胜率: {indicator['win_ratio']:.2%}")

if __name__ == '__main__':
    run(strategy_id='turtle_cta',
        filename='main.py',
        mode=MODE_BACKTEST,
        token='your_token',
        backtest_start_time='2023-01-01 09:30:00',
        backtest_end_time='2024-06-30 15:00:00',
        backtest_initial_cash=2000000,
        backtest_commission_ratio=0.000023,
        backtest_slippage_ratio=0.0005,
        backtest_adjust=ADJUST_PREV)
```

---

## 关键要点

| 项 | 说明 |
|----|------|
| 合约选择 | 回测可用虚拟合约 `CFFEX.IF`，实盘必须用具体月份 `CFFEX.IF2506` |
| 合约乘数 | IF=300, IH=300, IC=200, 需手动计算 |
| 平今/平昨 | 上期所需要用 `PositionEffect_CloseToday/CloseYesterday` 区分 |
| 实时模式 | 主连合约（如 `CFFEX.IFmain`）无行情推送，必须订阅具体合约 |
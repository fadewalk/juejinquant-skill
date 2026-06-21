# 配对交易策略完整示例（统计套利）

两个高相关性股票的均值回归策略，适用于横盘市场。

## 策略逻辑

- **配对**：两只同行业股票（如招行 vs 平安）
- **入场**：价格比 z-score > 阈值时，做空 A 做多 B
- **出场**：z-score 回归到均值附近时平仓

---

## 完整代码

```python
from gm.api import *
import numpy as np

def init(context):
    context.stock_a = 'SHSE.600036'   # 招商银行
    context.stock_b = 'SHSE.601166'   # 兴业银行
    context.lookback = 60              # 回看窗口（交易日）
    context.entry_z = 2.0              # 入场 z-score 阈值
    context.exit_z = 0.5               # 出场 z-score 阈值
    context.position_pct = 0.4         # 单边仓位占比

    # 同时订阅两只股票
    subscribe(symbols=f'{context.stock_a},{context.stock_b}',
              frequency='1d', count=context.lookback + 1)

def on_bar(context, bars):
    # 获取两只股票的历史数据
    hist_a = context.data(symbol=context.stock_a, frequency='1d', count=context.lookback)
    hist_b = context.data(symbol=context.stock_b, frequency='1d', count=context.lookback)

    if hist_a is None or hist_b is None or len(hist_a) < context.lookback or len(hist_b) < context.lookback:
        return

    closes_a = hist_a['close'].tolist()
    closes_b = hist_b['close'].tolist()

    # 计算价格比
    ratio = np.array(closes_a) / np.array(closes_b)
    ratio_mean = np.mean(ratio)
    ratio_std = np.std(ratio)

    if ratio_std == 0:
        return

    # 当前 z-score
    current_ratio = closes_a[-1] / closes_b[-1]
    z_score = (current_ratio - ratio_mean) / ratio_std

    # 查询两只股票的持仓
    all_positions = get_position() or []
    def get_pos_vol(symbol):
        for p in all_positions:
            sym = p.get('symbol') if isinstance(p, dict) else getattr(p, 'symbol', None)
            vol = p.get('volume') if isinstance(p, dict) else getattr(p, 'volume', None)
            if sym == symbol:
                return vol or 0
        return 0

    has_pos_a = get_pos_vol(context.stock_a) > 0
    has_pos_b = get_pos_vol(context.stock_b) > 0

    print(f'Z-score={z_score:.2f}, ratio={current_ratio:.4f}, mean={ratio_mean:.4f}')

    # z-score 过高：A 相对 B 高估 → 卖 A 买 B
    if z_score > context.entry_z and not has_pos_b:
        if has_pos_a:
            order_target_volume(symbol=context.stock_a, volume=0,
                                position_side=PositionSide_Long,
                                order_type=OrderType_Market)
        order_percent(symbol=context.stock_b, percent=context.position_pct,
                      side=OrderSide_Buy, order_type=OrderType_Market,
                      position_effect=PositionEffect_Open)
        print(f'  开仓：买入 {context.stock_b}')

    # z-score 过低：A 相对 B 低估 → 买 A 卖 B
    elif z_score < -context.entry_z and not has_pos_a:
        if has_pos_b:
            order_target_volume(symbol=context.stock_b, volume=0,
                                position_side=PositionSide_Long,
                                order_type=OrderType_Market)
        order_percent(symbol=context.stock_a, percent=context.position_pct,
                      side=OrderSide_Buy, order_type=OrderType_Market,
                      position_effect=PositionEffect_Open)
        print(f'  开仓：买入 {context.stock_a}')

    # z-score 回归 → 平仓
    elif abs(z_score) < context.exit_z and (has_pos_a or has_pos_b):
        order_target_volume(symbol=context.stock_a, volume=0,
                            position_side=PositionSide_Long,
                            order_type=OrderType_Market)
        order_target_volume(symbol=context.stock_b, volume=0,
                            position_side=PositionSide_Long,
                            order_type=OrderType_Market)
        print(f'  平仓：z-score 回归')

if __name__ == '__main__':
    run(strategy_id='pair_trading',
        filename='main.py',
        mode=MODE_BACKTEST,
        token='your_token',
        backtest_start_time='2023-01-01 09:30:00',
        backtest_end_time='2024-06-30 15:00:00',
        backtest_initial_cash=1000000,
        backtest_commission_ratio=0.0003,
        backtest_adjust=ADJUST_PREV)
```

---

## 进阶：使用协整性筛选配对

```python
from statsmodels.tsa.stattools import coint

def find_cointegrated_pair(stocks, start, end):
    """寻找协整的股票对"""
    for i, s1 in enumerate(stocks):
        for s2 in stocks[i+1:]:
            df1 = history(symbol=s1, frequency='1d', start_time=start, end_time=end,
                         fields='close', df=True)
            df2 = history(symbol=s2, frequency='1d', start_time=start, end_time=end,
                         fields='close', df=True)
            # 对齐时间
            merged = df1.join(df2, lsuffix='_1', rsuffix='_2').dropna()
            score, pvalue, _ = coint(merged['close_1'], merged['close_2'])
            if pvalue < 0.05:  # 5% 显著性水平下协整
                yield (s1, s2, pvalue)
```

---

## 关键要点

| 项 | 说明 |
|----|------|
| 同步问题 | 多标的建议 `subscribe(wait_group=True)` 避免数据不齐 |
| 参数敏感 | z-score 阈值（2.0/0.5）需要根据实际波动率调整 |
| 风险点 | 配对破裂（协整失效）时可能两边同向亏损 |
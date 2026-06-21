# 行业轮动策略完整示例

基于动量因子在多个行业 ETF 间轮动，常见的中频策略。

## 策略逻辑

- **标的池**：多个行业 ETF
- **因子**：20日收益率（动量）
- **持仓**：得分最高的 Top N 等权持有
- **调仓**：每日收盘后判断，次日开盘调仓

---

## 完整代码

```python
from gm.api import *

# ============================================================
# 策略一：策略内嵌（事件驱动）
# ============================================================

def init(context):
    context.sectors = {
        'SHSE.512000': '券商',
        'SHSE.512010': '医药',
        'SHSE.512660': '军工',
        'SHSE.512800': '银行',
        'SHSE.512690': '白酒',
        'SHSE.515030': '新能源',
    }
    context.top_n = 2                 # 持仓数量
    subscribe(symbols=','.join(context.sectors.keys()),
              frequency='1d', count=22)
    schedule(schedule_func=rebalance, frequency='1d', time_rule='15:05')

def rebalance(context, bar_dict=None):
    # 计算每个行业的20日动量
    momentum = {}
    for sym in context.sectors:
        data = context.data(symbol=sym, frequency='1d', count=22)
        if data is not None and len(data) >= 20:
            closes = data['close'].tolist()
            momentum[sym] = closes[-1] / closes[-20] - 1

    # 按动量排序，取 Top N
    ranked = sorted(momentum.items(), key=lambda x: x[1], reverse=True)
    targets = [s[0] for s in ranked[:context.top_n]]

    print(f'轮动调仓：目标={targets}')

    # 清掉非目标持仓
    for sym in context.sectors:
        if sym not in targets:
            order_target_volume(symbol=sym, volume=0,
                                position_side=PositionSide_Long,
                                order_type=OrderType_Market)

    # 等权买入
    for sym in targets:
        order_target_percent(symbol=sym, percent=1.0 / context.top_n * 0.95,
                             position_side=PositionSide_Long,
                             order_type=OrderType_Market)

if __name__ == '__main__':
    run(strategy_id='industry_rotation',
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

```python
# ============================================================
# 策略二：纯数据研究（无交易）
# ============================================================

from gm.api import *

set_token('your_token')

industry_etfs = {
    'SHSE.510050': '上证50',
    'SHSE.510300': '沪深300',
    'SHSE.510500': '中证500',
    'SZSE.159915': '创业板',
    'SHSE.512010': '医药ETF',
    'SHSE.512880': '证券ETF',
    'SHSE.512800': '银行ETF',
    'SHSE.515030': '新能源车ETF',
    'SZSE.159995': '芯片ETF',
    'SHSE.512690': '白酒ETF',
}

# 获取每只 ETF 过去一年的日数据
results = []
for symbol, name in industry_etfs.items():
    df = history(symbol=symbol, frequency='1d',
                 start_time='2024-01-01', end_time='2024-12-31',
                 fields='close,volume,eob',
                 adjust=ADJUST_PREV, df=True)
    if df is not None and len(df) > 20:
        closes = df['close'].tolist()
        ret_5d = (closes[-1] / closes[-6] - 1) * 100
        ret_20d = (closes[-1] / closes[-21] - 1) * 100
        ret_60d = (closes[-1] / closes[-61] - 1) * 100 if len(closes) > 60 else None
        avg_volume = sum(df['volume'].tolist()[-20:]) / 20

        results.append({
            '行业': name,
            '代码': symbol,
            '近5日收益(%)': round(ret_5d, 2),
            '近20日收益(%)': round(ret_20d, 2),
            '近60日收益(%)': round(ret_60d, 2) if ret_60d else None,
            '20日均量': int(avg_volume),
        })

import pandas as pd
df_result = pd.DataFrame(results).sort_values('近20日收益(%)', ascending=False)
print("\n行业轮动分析（按20日收益排序）：")
print(df_result.to_string(index=False))
```

---

## 关键要点

| 项 | 说明 |
|----|------|
| ETF 代码 | 5开头=沪市(SHSE)，1开头=深市(SZSE) |
| 调仓时间 | `15:05` 收盘后判断，次日开盘执行 |
| 滑点 | ETF 流动性好，可设小滑点 `0.0005` |
| 动量窗口 | 20 日是常用参数，可尝试 10/60 日对比 |
# 多因子选股策略完整示例

按月调仓的多因子打分选股，常见于量化选股场景。

## 策略逻辑

- **标的池**：沪深300成分股
- **因子**：PE（市盈率）、PB（市净率）低者优先
- **选股**：取综合得分最低的 N 只
- **调仓**：每月第一个交易日 10:00 等权买入

---

## 完整代码

```python
from gm.api import *

def init(context):
    # 每月第一个交易日 10:00 调仓
    schedule(schedule_func=multi_factor_rebalance, date_rule='1m', time_rule='10:00:00')
    context.hold_num = 10             # 持股数量
    context.target_index = 'SHSE.000300'  # 沪深300

def multi_factor_rebalance(context):
    """多因子选股 + 等权调仓"""
    # 1. 获取沪深300成分股
    constituents = get_constituents(index=context.target_index)
    symbols = [c['symbol'] for c in constituents]

    # 2. 获取财务指标（PE、PB）
    fund_data = stk_get_fundamentals_pt(
        table='trading_derivative_indicator',
        symbols=','.join(symbols),
        date=context.now.strftime('%Y-%m-%d'),
        fields='PETTM,PB,TURNRATE',
        df=True
    )

    if fund_data is None or len(fund_data) == 0:
        return

    # 3. 数据清洗：去除缺失值和异常值
    fund_data = fund_data.dropna(subset=['PETTM', 'PB'])
    fund_data = fund_data[(fund_data['PETTM'] > 0) & (fund_data['PETTM'] < 100)]
    fund_data = fund_data[(fund_data['PB'] > 0) & (fund_data['PB'] < 20)]

    # 4. 因子打分：PE/PB 越低越好
    fund_data['PE_rank'] = fund_data['PETTM'].rank(ascending=True)
    fund_data['PB_rank'] = fund_data['PB'].rank(ascending=True)
    fund_data['score'] = fund_data['PE_rank'] * 0.5 + fund_data['PB_rank'] * 0.5

    # 5. 选出得分最低的 N 只
    selected = fund_data.nsmallest(context.hold_num, 'score')
    target_symbols = selected['symbol'].tolist()

    print(f"本期选出 {len(target_symbols)} 只股票: {target_symbols}")

    # 6. 平掉非目标持仓
    all_positions = get_position() or []
    for pos in all_positions:
        sym = pos.get('symbol') if isinstance(pos, dict) else getattr(pos, 'symbol', None)
        vol = pos.get('volume') if isinstance(pos, dict) else getattr(pos, 'volume', 0)
        if sym not in target_symbols and vol and vol > 0:
            order_target_volume(symbol=sym, volume=0,
                                position_side=PositionSide_Long,
                                order_type=OrderType_Market)
            print(f"  平仓: {sym}")

    # 7. 等权买入目标股票（保留5%现金）
    target_pct = 0.95 / len(target_symbols)
    for sym in target_symbols:
        order_target_percent(symbol=sym, percent=target_pct,
                             position_side=PositionSide_Long,
                             order_type=OrderType_Market)
        print(f"  调仓: {sym} -> {target_pct*100:.1f}%")

if __name__ == '__main__':
    run(strategy_id='multi_factor',
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

## 关键要点

| 项 | 说明 |
|----|------|
| 财务数据 | 用 `stk_get_fundamentals_*_pt`（截面），不是已下线的 `get_fundamentals` |
| `fields` 必填 | 不能传空字符串，最多 20 个字段 |
| `date` 是发布日期 | 不是报告期日期 |
| 调仓频率 | 月频 `date_rule='1m'`，周频 `'1w'`（仅回测），日频 `'1d'` |
| 滑点 | 调仓类策略建议设滑点 `0.001` 模拟真实冲击成本 |
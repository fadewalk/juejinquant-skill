# 风险管理模块完整示例

通用风控类：单股仓位上限、最大回撤控制。可嵌入任意策略。

## 完整代码

```python
from gm.api import *

class RiskManager:
    """通用风控：止损/止盈/最大持仓/回撤控制"""

    def __init__(self, stop_loss_pct=0.05, take_profit_pct=0.10,
                 max_position_pct=0.3, max_drawdown_pct=0.15):
        self.stop_loss_pct = stop_loss_pct
        self.take_profit_pct = take_profit_pct
        self.max_position_pct = max_position_pct
        self.max_drawdown_pct = max_drawdown_pct
        self.peak_value = 0
        self.entry_prices = {}  # {symbol: entry_price}

    def check_stop_loss(self, symbol, current_price):
        if symbol not in self.entry_prices:
            return False
        entry = self.entry_prices[symbol]
        return (current_price - entry) / entry < -self.stop_loss_pct

    def check_take_profit(self, symbol, current_price):
        if symbol not in self.entry_prices:
            return False
        entry = self.entry_prices[symbol]
        return (current_price - entry) / entry > self.take_profit_pct

    def check_drawdown(self):
        cash = get_cash()
        nav = cash.nav if hasattr(cash, 'nav') else cash.available
        self.peak_value = max(self.peak_value, nav)
        dd = (self.peak_value - nav) / self.peak_value if self.peak_value > 0 else 0
        return dd > self.max_drawdown_pct

    def check_position_limit(self, symbol_value, total_value):
        return (symbol_value / total_value) > self.max_position_pct if total_value > 0 else False


# ============================================================
# 主策略（嵌入 RiskManager）
# ============================================================

def init(context):
    subscribe(symbols='SHSE.600000,SZSE.000001,SHSE.601318', frequency='1d', count=21)
    context.risk = RiskManager(
        stop_loss_pct=0.05,      # 单股止损5%
        take_profit_pct=0.10,    # 单股止盈10%
        max_position_pct=0.30,   # 单股仓位上限30%
        max_drawdown_pct=0.15    # 最大回撤15%
    )
    context.is_stopped = False

def on_bar(context, bars):
    # 1. 检查最大回撤
    if context.risk.check_drawdown():
        if not context.is_stopped:
            print(f'⚠️ 触发最大回撤，停止所有交易！')
            order_close_all()
            context.is_stopped = True
        return
    context.is_stopped = False

    # 2. 检查每只持仓的止损/止盈
    cash_info = get_cash()
    nav = cash_info.nav if hasattr(cash_info, 'nav') else cash_info.available

    all_positions = get_position() or []
    for pos in all_positions:
        sym = pos.get('symbol') if isinstance(pos, dict) else getattr(pos, 'symbol', None)
        vol = pos.get('volume') if isinstance(pos, dict) else getattr(pos, 'volume', 0)
        if not vol or vol <= 0:
            continue

        # 简单示例：用订阅的最新价
        data = context.data(symbol=sym, frequency='1d', count=2)
        if data is None or len(data) < 2:
            continue
        current_price = data['close'].iloc[-1]

        if context.risk.check_stop_loss(sym, current_price):
            print(f'止损: {sym} 价格={current_price}')
            order_target_volume(symbol=sym, volume=0,
                                position_side=PositionSide_Long,
                                order_type=OrderType_Market)
            context.risk.entry_prices.pop(sym, None)
        elif context.risk.check_take_profit(sym, current_price):
            print(f'止盈: {sym} 价格={current_price}')
            order_target_volume(symbol=sym, volume=0,
                                position_side=PositionSide_Long,
                                order_type=OrderType_Market)
            context.risk.entry_prices.pop(sym, None)

    # 3. 检查单股仓位上限
    for pos in all_positions:
        sym = pos.get('symbol') if isinstance(pos, dict) else getattr(pos, 'symbol', None)
        mv = pos.get('market_value') if isinstance(pos, dict) else getattr(pos, 'market_value', 0)
        if mv and nav > 0:
            if context.risk.check_position_limit(mv, nav):
                target_value = nav * context.risk.max_position_pct
                print(f'仓位超限: {sym} 当前={mv/nav:.1%}, 降到={context.risk.max_position_pct:.1%}')
                order_target_value(symbol=sym, value=target_value,
                                   position_side=PositionSide_Long,
                                   order_type=OrderType_Market)

if __name__ == '__main__':
    run(strategy_id='risk_mgmt',
        filename='main.py',
        mode=MODE_BACKTEST,
        token='your_token',
        backtest_start_time='2023-01-01 09:30:00',
        backtest_end_time='2024-06-30 15:00:00',
        backtest_initial_cash=1000000,
        backtest_adjust=ADJUST_PREV)
```

---

## 关键要点

| 项 | 说明 |
|----|------|
| 风控顺序 | 先回撤（最严重）→ 再单股止损 → 最后仓位调整 |
| 状态保存 | `context.is_stopped` 防止重复触发清仓 |
| 现金字段 | `nav`（净资产）vs `available`（可用），回测总市值用 nav |
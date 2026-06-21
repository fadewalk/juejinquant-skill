# 实战踩坑经验合集（28 条）

> 来源：gm-quant 长期实测验证 — 编写策略代码时**必须遵守**的坑点清单。

## 1. `run()` 参数名

参数名是 `strategy_id` + `filename`（模块名，不带 `.py` 后缀），不是 `strategy_name` / `file_path`。

```python
run(strategy_id='my_strategy', filename='main.py', ...)  # ✅
run(strategy_name='my_strategy', file_path='main.py', ...)  # ❌
```

## 2. `log()` 用法

`log(msg, source)` 是普通函数，**不是** logger 对象。不要用 `log.info()`。

```python
log.info('xxx')  # ❌ AttributeError
log('xxx', source='strategy')  # ✅
# 推荐直接用 print()
print('xxx')  # ✅ 更简洁
```

## 3. `context.data()` 返回值

返回的是 **DataFrame**（不是 dict list），用 `data['close'].tolist()` 访问数据列。

```python
data = context.data(symbol='SHSE.600000', frequency='1d', count=20)
data['close'].tolist()  # ✅
[bar['close'] for bar in data]  # ❌ DataFrame 不支持迭代 dict
```

## 4. `get_position()` 不带参数

调用 `get_position()` 获取全部持仓列表，然后遍历查找目标 symbol 的持仓。**不支持** `get_position(symbol=xxx)`。

```python
all_positions = get_position()
for p in all_positions:
    sym = p.get('symbol') if isinstance(p, dict) else p.symbol
    if sym == target:
        # ...
```

## 5. `order_target_volume()` 参数

不需要 `side` 参数；用 `position_side=PositionSide_Long`（不是 `position_effect=PositionEffect_Close`）。

```python
order_target_volume(symbol='SHSE.600519', volume=0,
                    position_side=PositionSide_Long,  # ✅
                    order_type=OrderType_Market)
```

## 6. `order_volume()` 买入参数

需要 `side=OrderSide_Buy` + `position_effect=PositionEffect_Open`。

```python
order_volume(symbol='SHSE.600519', volume=100,
             side=OrderSide_Buy,
             position_effect=PositionEffect_Open,  # ✅
             order_type=OrderType_Market)
```

## 7. Windows 编码

脚本开头必须加：

```python
import sys, io
sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8', errors='replace')
```

否则中文 emoji 报错。

## 8. A 股 T+1

当天买入不能当天卖出，策略逻辑需要考虑这个约束。

## 9. `cl_ord_id` 参数不存在

`order_volume` / `order_value` / `order_target_volume` 等下单函数**不支持** `cl_ord_id` 参数。传入会直接抛 `TypeError: got an unexpected keyword argument 'cl_ord_id'`。订单标识由系统自动生成。

## 10. `on_order_status` 回测模式状态码

回测中会出现文档未记录的状态码 `10`（未知/内部中间态），实际运行时需兼容处理。

完整状态码：
- 1=新建, 2=已报, 3=部分成交, 4=已成交
- 5=已撤, 6=未成交(超时), 7=拒绝, 8=待撤
- 9=未知, **10=回测内部态**

## 11. `on_execution_report` 的 exec_type

回测模式下返回数字而非字符，实测值为 `15`（成交确认）。实盘/仿真可能返回 `'T'`(Trade) / `'C'`(Cancel)。

## 12. 回测中风控行为

超额卖出或资金不足的订单在回测中不会触发 `Rejected(7)` 状态，而是变为 `Cancelling(8)` → 被自动撤销。**拒单原因需要实盘/仿真环境才能观察到**。

## 13. `on_order_status` order 对象访问

回调中的 order 对象**同时支持** dict 风格 `order['symbol']` 和属性风格 `order.symbol`，但推荐用 try/getattr 兼容两种方式。

## 14. `on_execution_report` execrpt 对象访问

同上，同时支持 dict 和属性风格。关键字段：`symbol`, `side`(1买2卖), `volume`, `price`, `exec_type`, `commission`。

## 15. 市价单在回测中也可能不成交

如果资金不足（如下单量×价格 > 可用资金），市价单会被标记为 Cancelling 而非报错抛异常。

## 16. `stk_get_index_constituents` 没有 `df` 参数

直接返回 DataFrame，不需要传 `df=True`。

```python
constituents = get_constituents(index='SHSE.000300')  # ✅ 返回 DataFrame
```

## 17. 财务数据 API 的 `fields` 必填且不能为空

所有 `stk_get_fundamentals_*_pt` / `stk_get_finance_*_pt` / `stk_get_daily_*_pt` 函数的 `fields` 参数是必填的，不能传空字符串 `""`，否则报错"填写的 fields 不正确"。fields 不能超过 20 个。

## 18. `stk_get_finance_prime_pt` ROE 字段名

是 `roe_weight_avg`，**不是** `roe_waa`。

常用字段：`eps_basic/eps_dil/roe_weight_avg/roe_weight_avg_cut/net_prof_pcom_yoy/inc_oper_yoy`

## 19. `stk_get_daily_basic_pt` 股本字段名

- 流通股本：`circ_shr`（不是 `float_shr`）
- 无限售条件流通股本：`ttl_shr_unl`（不是 `free_shr`）
- 有限售条件股本：`ttl_shr_ltd`

## 20. 财务衍生指标 `eps_dil2` vs `eps_dil`

- `stk_get_finance_deriv_pt` 中稀释 EPS 字段名是 `eps_dil2`（**不是** `eps_dil`）
- `stk_get_finance_prime_pt` 中是 `eps_dil`

## 21. `_pt` 后缀 = 截面数据(多标的)

- **无后缀** = 时序数据(单标的)，用 `start_date/end_date` 参数
- **`_pt` 后缀** = 截面数据(多标的)，用 `date/trade_date` 参数

## 22. 付费增值数据 API

期货（`fut_get_*`）、基金（`fnd_get_*`）、可转债（`bnd_get_*`）的增值数据需要开通相应权限。详见 `references/16-premium-data-apis.md`。

## 23. `stk_get_fundamentals_*_pt` 的 `date` 参数

是**发布日期**，不是报告期日期。返回的是发布日期 ≤ date 的最新报告期数据。

## 24. `stk_get_daily_valuation_pt/mktvalue_pt/basic_pt` 的 `trade_date` 参数

是交易日期，默认 None 返回最新交易日数据。

## 25. 回测交易日限制

每个交易日 18:30 前只能回测上一个交易日的数据，因为当日日线数据要到 18:30 才更新完成。如果 `end_date` 设为当天但还没过 18:30，回测结果会缺少当日数据或报错。

## 26. 实时模式（仿真/实盘）没有发生交易的排查清单

① **定时任务时间过了**：`schedule` 定时任务只在指定时间触发，如果启动策略时已过了今天的时间点，要等到明天才会触发。临时解决：把时间改成当前时间之后几分钟。

② **期货策略必须订阅具体合约**：实时模式只能推送具体合约行情（如 `SHFE.ag2506`），主连合约（如 `SHFE.agmain`）**没有行情推送**。回测可以主连，实时不行。

③ **实时模式日线不会推送**：交易时间内日线还没走完，`on_bar` 不会收到日线 bar。需要用 `schedule` 定时任务替代，在收盘后（如 15:01）主动调用 `history` 获取日线数据。

④ **检查打印日志**：确认是否有数据推送 → 是否有交易信号发出 → 是否有下单指令 → 订单状态是否正常。按这个链路逐级排查。

## 27. `order_volume()` vs `order_target_volume/percent` 参数名不同

- `order_volume()` 的开平仓参数叫 `position_effect`（用 `PositionEffect_Open/Close`）
- `order_target_volume/percent/value` 的持仓方向参数叫 `position_side`（用 `PositionSide_Long/Short`）

ETF 调仓推荐用 `order_target_percent`，更简洁不用算股数。

## 28. 56 开头的 ETF 是沪市

如 562500 机器人 ETF 应为 `SHSE.562500`，不是深市。**5 开头 = 沪市（SHSE），1 开头 = 深市（SZSE）**。
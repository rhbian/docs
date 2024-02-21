---
title: "Fundamental Data"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 基础知识
## 不同类型的订单
- 市价订单(market order):市价订单旨在在抵达交易场所时立即执行，以那一刻的市价为准。
- 限价订单(limit order):限价订单只有在市场价格高于卖出限价订单的限价或低于买入限价订单的限价时才执行。
- 止损订单(stop order):止损订单则只有在市场价格升至买入止损订单的指定价格或下降至卖出止损订单的指定价格时才激活。
- 买入止损订单(buy stop order):用于限制空头交易的损失。


# 数据清洗与整理
略。详见jupyter文档

## 订单的可视化

### tick bars
tick级别的数据体量较大，刻画数据可以使用最直接的直线图。刻画在不同的时间戳下的成交价格。由于在开盘和收盘时交易活动最活跃，波动也最大，所以经常去掉cross数据。

书中核心代码如下
```Python
stock, date = 'AAPL', '20191030'
title = '{} | {}'.format(stock, pd.to_datetime(date).date()
with pd.HDFStore(itch_store) as store:
sys_events = store['S'].set_index('event_code') # system events
sys_events.timestamp = sys_events.timestamp.add(pd.to_datetime(date)).dt.time
market_open = sys_events.loc['Q', 'timestamp']
market_close = sys_events.loc['M', 'timestamp']
with pd.HDFStore(stock_store) as store:
trades = store['{}/trades'.format(stock)].reset_index()
trades = trades[trades.cross == 0] # excluding data from open/close crossings
trades.price = trades.price.mul(1e-4) # format price
trades = trades[trades.cross == 0]
# exclude crossing trades
trades = trades.between_time(market_open, market_close) # market hours only
tick_bars = trades.set_index('timestamp')
tick_bars.index = tick_bars.index.time
tick_bars.price.plot(figsize=(10, 5), title=title), lw=1)
```
值得注意的是，这个作图并不是很精致，尤其是刻度线部分。

### time bars
改进1：
因为逐笔数据以纳秒为索引，噪音较大。例如买卖报价(bid-ask bounce)会导致价格在买入和卖出市场的订单交替发起时，在买入价和卖出价之间震荡。为**提高噪音信号比**，并**改善统计属性**，通过聚合交易活动来重采样规范化逐笔数据。

收集聚合周期的olhc以及成交量加权平均价格(VWAP)、交易的股票数量以及数据相关联的时间戳。

核心代码如下：
聚合周期为1min，加权平均计算为$vwap = \dfrac{price \cdot shares}{\sum{shares}}$

```Python
def get_bar_stats(agg_trades: pd.core.groupby):
    vwap = agg_trades.apply(lambda x: np.average(x.price, weights=x.shares)).to_frame('vwap')
    ohlc = agg_trades.price.ohlc() # 创建 open high, low, close
    vol = agg_trades.shares.sum().to_frame('vol') # 汇总份额
    txn = agg_trades.shares.size().to_frame('txn') # 每个组里有多少个不同的shares
    return pd.concat([ohlc, vwap, vol, txn], axis=1)
resampled = trades.groupby(pd.Grouper(freq='1Min')) # 以1min为周期进行聚合。
time_bars = get_bar_stats(resampled)
```

K线图 bokeh库
```Python
from bokeh.io import output_notebook, show
output_notebook()
resampled = trades.groupby(pd.Grouper(freq='5Min')) # 5 Min bars for better print
df = get_bar_stats(resampled)

increase = df.close > df.open # 上升
decrease = df.open > df.close # 下降
w = 2.5 * 60 * 1000 # 2.5 min in ms 

WIDGETS = "pan, wheel_zoom, box_zoom, reset, save"


p = figure(x_axis_type='datetime', tools=WIDGETS, title = "AAPL Candlestick") #  plot_width=1500,
p.xaxis.major_label_orientation = pi/4
p.grid.grid_line_alpha=0.4

p.segment(df.index, df.high, df.index, df.low, color="black")
p.vbar(df.index[increase], w, df.open[increase], df.close[increase], fill_color="#FF0000", line_color="black")
p.vbar(df.index[decrease], w, df.open[decrease], df.close[decrease], fill_color="#008000", line_color="black")
show(p)
```

### volumn bars
time bars可以平滑原始逐笔数据中的一些噪音，但可能无法解释订单的碎片化。以执行为中心的算法交易可能旨在在给定时期内匹配成交量加权平均价格（VWAP），并将单一订单分成多个交易，并根据历史模式下单。即使市场中没有新信息到来，时间条也会对同一订单进行不同的处理。volumn bars根据交易量来聚合交易数据。

```Python
trades_per_min = trades.shares.sum()/(60*7.5) # min per trading day 每天交易的分钟数
trades['cumul_vol'] = trades.shares.cumsum()

df = trades.reset_index()
by_vol = df.groupby(df.cumul_vol.div(trades_per_min).round().astype(int))
vol_bars = pd.concat([by_vol.timestamp.last().to_frame('timestamp'), get_bar_stats(by_vol)], axis=1)
vol_bars.head()
```

$每分钟的平均交易量 = (\sum{shares}) / (60*7.5)$

累积的交易量 = shares.cumsum()

累积交易量/每分钟的交易量

这个处理方式比较难理解。只是在交易量上的不同，但是时间戳都混乱了。

### dollar bars
dollar bars也进行了和volumn bars类似的处理。
```Python
value_per_min = trades.shares.mul(trades.price).sum()/(60*7.5) # min per trading day
trades['cumul_val'] = trades.shares.mul(trades.price).cumsum()

df = trades.reset_index()
by_value = df.groupby(df.cumul_val.div(value_per_min).round().astype(int))
dollar_bars = pd.concat([by_value.timestamp.last().to_frame('timestamp'), get_bar_stats(by_value)], axis=1)
dollar_bars.head()
```

## algoseek数据
algoseek提供了分钟数据。网站上的数据需要订阅，书中提供了[示例数据](https://algoseek-public.s3.amazonaws.com/nasdaq100-1min.zip)



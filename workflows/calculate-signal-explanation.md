# Calculate Signal 节点代码解释

## 代码功能概述
这个节点用于计算交易信号（买入/卖出/持有），基于币安K线数据计算EMA（指数移动平均线）和SMA（简单移动平均线）指标。

## 详细代码解析

### 1. 数据提取和验证
```javascript
const [{ json: candles }] = $input.all();
if (!Array.isArray(candles) || candles.length === 0) {
  throw new Error('Binance kline response is empty.');
}
const closes = candles.map((row) => Number(row[4]));
```
- 从输入中获取K线数据
- 验证数据是否为非空数组
- 提取收盘价（K线数组的第5个元素，索引为4）

### 2. 参数配置
```javascript
const shortPeriod = Number($env.BINANCE_SHORT_EMA ?? $env.BINANCE_SHORT_SMA ?? 7);
const longPeriod = Number($env.BINANCE_LONG_EMA ?? $env.BINANCE_LONG_SMA ?? 25);
const threshold = Number($env.BINANCE_EMA_THRESHOLD ?? $env.BINANCE_SIGNAL_THRESHOLD ?? 0.001);
```
- **shortPeriod**: 短期EMA周期（默认7）
- **longPeriod**: 长期EMA周期（默认25）
- **threshold**: 信号触发阈值（默认0.001，即0.1%）

### 3. 参数验证
```javascript
if (!Number.isFinite(shortPeriod) || shortPeriod <= 0) {
  throw new Error('BINANCE_SHORT_EMA must be a positive integer.');
}
if (!Number.isFinite(longPeriod) || longPeriod <= 0) {
  throw new Error('BINANCE_LONG_EMA must be a positive integer.');
}
```
- 确保短期和长期周期都是有效的正数

### 4. 数据充足性检查
```javascript
const maxPeriod = Math.max(shortPeriod, longPeriod);
const symbol = $env.BINANCE_SYMBOL ?? 'BTCUSDT';
const interval = $env.BINANCE_INTERVAL ?? '1m';
const requestedLimit = Number($env.BINANCE_CANDLE_LIMIT ?? 50);
const latestClose = closes.length > 0 ? closes[closes.length - 1] : 0;

if (maxPeriod > closes.length) {
  // 返回HOLD信号，因为数据不足
  return [{
    json: {
      symbol,
      price: Number(latestClose.toFixed(2)),
      shortSma: null,
      longSma: null,
      shortEma: null,
      longEma: null,
      emaDelta: 0,
      emaThreshold: threshold,
      signalSource: 'EMA',
      signal: 'HOLD',
      reason: `Insufficient candles: received ${closes.length}, need at least ${maxPeriod}...`,
      diagnostics: { ... },
      generatedAt: new Date().toISOString()
    }
  }];
}
```
- 检查是否有足够的数据点来计算指标
- 如果数据不足，返回HOLD信号并包含诊断信息

### 5. SMA（简单移动平均线）计算函数
```javascript
const sma = (values, period) => {
  const slice = values.slice(-period);  // 取最后period个值
  const total = slice.reduce((sum, value) => sum + value, 0);
  return total / period;  // 计算平均值
};
```
- 计算指定周期内收盘价的平均值

### 6. EMA（指数移动平均线）计算函数
```javascript
const ema = (values, period) => {
  if (values.length < period) {
    return null;
  }
  const multiplier = 2 / (period + 1);  // EMA平滑系数
  // 初始EMA = 前period个值的SMA
  let emaPrev = values.slice(0, period).reduce((sum, value) => sum + value, 0) / period;
  // 迭代计算EMA
  for (let i = period; i < values.length; i += 1) {
    emaPrev = (values[i] - emaPrev) * multiplier + emaPrev;
  }
  return emaPrev;
};
```
- **EMA公式**: `EMA = (当前价格 - 前一个EMA) × 平滑系数 + 前一个EMA`
- 平滑系数 = 2 / (周期 + 1)
- EMA对近期价格给予更高权重，反应更灵敏

### 7. 计算指标值
```javascript
const shortSma = sma(closes, shortPeriod);
const longSma = sma(closes, longPeriod);
const shortEma = ema(closes, shortPeriod);
const longEma = ema(closes, longPeriod);
```
- 计算短期和长期的SMA和EMA

### 8. 计算EMA差值（Delta）
```javascript
const emaDelta = longEma !== null && longEma !== 0 
  ? (shortEma - longEma) / longEma 
  : 0;
```
- **emaDelta**: (短期EMA - 长期EMA) / 长期EMA
- 表示短期EMA相对于长期EMA的百分比差异
- 正值表示短期趋势向上，负值表示短期趋势向下

### 9. 生成交易信号
```javascript
let signal = 'HOLD';
if (emaDelta > threshold) {
  signal = 'BUY';   // 短期EMA显著高于长期EMA，买入信号
} else if (emaDelta < -threshold) {
  signal = 'SELL';  // 短期EMA显著低于长期EMA，卖出信号
}
```
- **BUY**: 当emaDelta > threshold（短期EMA明显高于长期EMA）
- **SELL**: 当emaDelta < -threshold（短期EMA明显低于长期EMA）
- **HOLD**: 其他情况（差值在阈值范围内）

### 10. 返回结果
```javascript
return [{
  json: {
    symbol,                    // 交易对符号
    price: Number(latestClose.toFixed(2)),  // 最新收盘价
    shortSma: Number(shortSma.toFixed(2)),  // 短期SMA
    longSma: Number(longSma.toFixed(2)),    // 长期SMA
    shortEma: shortEma !== null ? Number(shortEma.toFixed(2)) : null,  // 短期EMA
    longEma: longEma !== null ? Number(longEma.toFixed(2)) : null,    // 长期EMA
    emaDelta: Number(emaDelta.toFixed(4)),   // EMA差值（保留4位小数）
    emaThreshold: threshold,   // 使用的阈值
    signalSource: 'EMA',       // 信号来源
    signal,                    // 交易信号：BUY/SELL/HOLD
    generatedAt: new Date().toISOString()  // 生成时间戳
  }
}];
```

## 交易策略逻辑

这是一个**双EMA交叉策略**的变种：

1. **买入条件**: 短期EMA > 长期EMA，且差值超过阈值
   - 表示价格上升趋势，短期动量强于长期

2. **卖出条件**: 短期EMA < 长期EMA，且差值超过阈值（负值）
   - 表示价格下降趋势，短期动量弱于长期

3. **持有条件**: EMA差值在阈值范围内
   - 表示趋势不明确或波动较小

## 关键参数说明

- **BINANCE_SHORT_EMA**: 短期EMA周期（默认7）
- **BINANCE_LONG_EMA**: 长期EMA周期（默认25）
- **BINANCE_EMA_THRESHOLD**: 信号触发阈值（默认0.001 = 0.1%）
- **BINANCE_CANDLE_LIMIT**: 需要的K线数量（至少需要max(shortPeriod, longPeriod)）

## 注意事项

1. 数据充足性：需要至少 `max(shortPeriod, longPeriod)` 根K线才能计算
2. 阈值设置：较小的阈值会产生更多交易信号，较大的阈值会减少交易频率
3. EMA vs SMA：EMA对近期价格更敏感，SMA对所有价格权重相等



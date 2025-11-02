# Lighter API 文档学习笔记

根据 [Lighter API 文档](https://apidocs.lighter.xyz/docs) 的学习总结。

## 📋 核心概念

### 1. API Key 设置

Lighter API 需要以下配置：
- **API_KEY_PRIVATE_KEY**: API 密钥私钥
- **ACCOUNT_INDEX**: 账户索引（可通过查询账户数据获取）
- **API_KEY_INDEX**: API 密钥索引（2-254，0 和 1 为桌面/移动端保留，255 可查询所有密钥）

### 2. SignerClient 初始化

```python
client = lighter.SignerClient(
    url=BASE_URL,              # mainnet: https://mainnet.zklighter.elliot.ai
    private_key=API_KEY_PRIVATE_KEY,
    account_index=ACCOUNT_INDEX,
    api_key_index=API_KEY_INDEX
)
```

### 3. Nonce 管理

- 每个 API_KEY 都有独立的 nonce
- 每次签名交易都需要递增 nonce
- 可以通过 `TransactionApi.next_nonce()` 获取下一个 nonce

## 📝 订单类型

Lighter 支持以下订单类型（非常重要！）：

1. **ORDER_TYPE_LIMIT** - 限价单
2. **ORDER_TYPE_MARKET** - 市价单
3. **ORDER_TYPE_STOP_LOSS** - 止损单 ⭐
4. **ORDER_TYPE_STOP_LOSS_LIMIT** - 止损限价单 ⭐
5. **ORDER_TYPE_TAKE_PROFIT** - 止盈单 ⭐
6. **ORDER_TYPE_TAKE_PROFIT_LIMIT** - 止盈限价单 ⭐
7. **ORDER_TYPE_TWAP** - TWAP 订单

## ⏱️ 时间强制选项 (Time in Force)

- **ORDER_TIME_IN_FORCE_IMMEDIATE_OR_CANCEL** - 立即成交或取消
- **ORDER_TIME_IN_FORCE_GOOD_TILL_TIME** - 有效期至指定时间
- **ORDER_TIME_IN_FORCE_POST_ONLY** - 仅 Maker（确保订单挂在订单簿上）

## 🔧 当前实现分析

### 当前代码使用的订单类型

查看 `exchanges/lighter.py` 第 296 行：

```python
'order_type': self.lighter_client.ORDER_TYPE_LIMIT,
'time_in_force': self.lighter_client.ORDER_TIME_IN_FORCE_GOOD_TILL_TIME,
```

### ⚠️ 发现的问题

当前实现**只使用了限价单**来实现止损功能，但 Lighter API **原生支持止损订单类型**：

- `ORDER_TYPE_STOP_LOSS` - 止损单
- `ORDER_TYPE_STOP_LOSS_LIMIT` - 止损限价单
- `ORDER_TYPE_TAKE_PROFIT` - 止盈单
- `ORDER_TYPE_TAKE_PROFIT_LIMIT` - 止盈限价单

## 💡 改进建议

### 方案 1: 使用原生止损订单（推荐）

如果使用 Lighter 原生的止损订单类型，可以：

1. **更精确的执行**：由交易所系统直接管理止损，而不是通过限价单模拟
2. **更好的性能**：不需要监控订单状态，交易所会在触发条件满足时自动执行
3. **减少延迟**：不依赖于订单簿价格，直接在达到止损价格时执行

### 方案 2: 保持当前实现（兼容性好）

当前使用限价单实现的优点：
- 与其他交易所实现方式一致
- 代码逻辑统一
- 便于维护和理解

## 📚 重要 API 方法

### SignerClient 便捷方法

1. **`create_order()`** - 签名并推送创建订单交易
2. **`create_market_order()`** - 签名并推送市价单
3. **`create_cancel_order()`** - 签名并推送取消订单交易
4. **`cancel_all_orders()`** - 签名并推送取消所有订单交易
5. **`create_auth_token_with_expiry()`** - 创建认证令牌（用于 API 和 WebSocket 认证）

### API 类

1. **AccountApi** - 账户数据
   - `account()` - 获取账户数据
   - `accounts_by_l1_address()` - 获取所有账户（主账户和子账户）
   - `apikeys()` - 获取 API 密钥数据

2. **TransactionApi** - 交易相关
   - `next_nonce()` - 获取下一个 nonce
   - `send_tx()` - 推送交易
   - `send_tx_batch()` - 批量推送交易

3. **OrderApi** - 订单和订单簿
   - `order_book_details()` - 获取特定市场的订单簿详情
   - `order_books()` - 获取所有市场的订单簿

## 🌐 WebSocket

Lighter 提供 WebSocket 访问：
- 账户更新
- 订单簿更新
- 订单状态更新

当前代码使用了自定义的 `LighterCustomWebSocketManager` 实现。

## 📊 订单参数说明

创建订单时需要提供：

- `market_index`: 市场索引（合约ID）
- `client_order_index`: 客户端订单索引（唯一标识符，用于后续引用/取消订单）
- `base_amount`: 基础数量（整数）
- `price`: 价格（整数）
- `is_ask`: 是否为卖单（True = 卖，False = 买）
- `order_type`: 订单类型（见上方列表）
- `time_in_force`: 时间强制选项
- `reduce_only`: 是否仅减仓
- `trigger_price`: 触发价格（用于止损/止盈订单）

## 🔄 与当前实现的对比

| 功能 | 当前实现 | Lighter 原生支持 |
|------|---------|------------------|
| 限价单 | ✅ 使用 ORDER_TYPE_LIMIT | ✅ 支持 |
| 市价单 | ❌ 未实现 | ✅ ORDER_TYPE_MARKET |
| 止损 | ⚠️ 用限价单模拟 | ✅ ORDER_TYPE_STOP_LOSS |
| 止盈 | ⚠️ 用限价单模拟 | ✅ ORDER_TYPE_TAKE_PROFIT |
| 止损限价 | ❌ 未实现 | ✅ ORDER_TYPE_STOP_LOSS_LIMIT |
| 止盈限价 | ❌ 未实现 | ✅ ORDER_TYPE_TAKE_PROFIT_LIMIT |

## 🎯 结论

Lighter API 提供了强大的原生止损/止盈订单支持，当前代码可以使用这些功能来优化实现。不过，为了保持与其他交易所的一致性，当前使用限价单的实现也是可行的。

如果要充分利用 Lighter 的特性，可以考虑：
1. 添加对 `ORDER_TYPE_STOP_LOSS` 的支持
2. 添加对 `ORDER_TYPE_TAKE_PROFIT` 的支持
3. 这样可以减少代码复杂度，提高执行效率


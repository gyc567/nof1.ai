[根目录](../../CLAUDE.md) > [src](../) > **database**

# 数据库操作模块

## 🎯 模块职责

数据库操作模块是系统的**数据中枢**，负责：
- 数据持久化存储与查询（交易记录、持仓、账户历史）
- 与Gate.io API数据同步（持仓、订单状态）
- 数据完整性检查与修复
- 数据库初始化、迁移与重置

## 🚀 入口与启动

**主入口**: `init.ts` (系统启动时调用)

```typescript
// 系统启动流程
await initDatabase();        // 初始化数据库
await initTradingSystem();   // 同步交易配置
```

**关键脚本入口**:
- `init.ts` - 数据库初始化
- `reset.ts` - 数据库重置  
- `sync-from-gate.ts` - Gate.io数据同步
- `close-and-reset.ts` - 平仓并重置

## 🔌 对外接口

### 数据库初始化接口

| 函数 | 用途 | 参数 | 返回 |
|-----|------|------|------|
| `initDatabase()` | 初始化数据库表结构 | 无 | Promise<void> |
| `resetDatabase()` | 重置数据库（清空数据） | 无 | Promise<void> |
| `checkDatabaseConsistency()` | 检查数据一致性 | 无 | Promise<ConsistencyReport> |

### 数据同步接口

| 函数 | 用途 | 参数 | 返回 |
|-----|------|------|------|
| `syncPositionsFromGate()` | 同步持仓数据 | gateClient | Promise<SyncResult> |
| `syncTradesFromGate()` | 同步交易记录 | gateClient | Promise<SyncResult> |
| `closeAllPositions()` | 平仓所有持仓 | gateClient | Promise<CloseResult> |

## 🛠️ 关键依赖与配置

### 核心依赖
```json
{
  "@libsql/client": "^0.x",          // LibSQL数据库客户端
  "gate-api": "^6.61.0",             // Gate.io API客户端
  "@voltagent/logger": "^1.0.0"      // 日志系统
}
```

### 数据库配置
```typescript
// 数据库连接配置
const dbClient = createClient({
  url: process.env.DATABASE_URL || "file:./.voltagent/trading.db",
});
```

### 环境变量
```bash
# 数据库配置
DATABASE_URL=file:./.voltagent/trading.db

# Gate.io同步配置
GATE_API_KEY=your_api_key
GATE_API_SECRET=your_api_secret
GATE_USE_TESTNET=false
```

## 📊 数据模型设计

### 核心表结构

#### 1. 交易记录表 (trades)
```typescript
interface Trade {
  id: number;                // 主键
  order_id: string;          // Gate.io订单ID
  symbol: string;            // 交易对 (BTC/USDT)
  side: 'long' | 'short';    // 交易方向
  type: 'open' | 'close';    // 开仓/平仓
  price: number;             // 成交价格
  quantity: number;          // 成交数量
  leverage: number;          // 杠杆倍数
  pnl?: number;              // 盈亏金额
  fee?: number;              // 手续费
  timestamp: string;         // 成交时间
  status: 'pending' | 'filled' | 'cancelled'; // 订单状态
}
```

#### 2. 持仓表 (positions)
```typescript
interface Position {
  id: number;                // 主键
  symbol: string;            // 交易对 (唯一约束)
  quantity: number;          // 持仓数量
  entry_price: number;       // 开仓价格
  current_price: number;     // 当前价格
  liquidation_price: number; // 强平价格
  unrealized_pnl: number;    // 未实现盈亏
  leverage: number;          // 杠杆倍数
  side: 'long' | 'short';    // 持仓方向
  profit_target?: number;    // 止盈价格
  stop_loss?: number;        // 止损价格
  tp_order_id?: string;      // 止盈订单ID
  sl_order_id?: string;      // 止损订单ID
  entry_order_id: string;    // 开仓订单ID
  opened_at: string;         // 开仓时间
  confidence?: number;       // AI信心度
  risk_usd?: number;         // 风险金额(USD)
  peak_pnl_percent?: number; // 历史最高盈亏百分比
}
```

#### 3. 账户历史表 (account_history)
```typescript
interface AccountHistory {
  id: number;                // 主键
  timestamp: string;         // 记录时间
  total_value: number;       // 总资产价值
  available_cash: number;    // 可用资金
  unrealized_pnl: number;    // 未实现盈亏
  realized_pnl: number;      // 已实现盈亏
  return_percent: number;    // 收益率百分比
  sharpe_ratio?: number;     // 夏普比率
}
```

#### 4. 技术指标表 (trading_signals)
```typescript
interface TradingSignal {
  id: number;                // 主键
  symbol: string;            // 交易对
  timestamp: string;         // 信号时间
  price: number;             // 当前价格
  ema_20: number;            // 20周期EMA
  ema_50?: number;           // 50周期EMA
  macd: number;              // MACD值
  rsi_7: number;             // 7周期RSI
  rsi_14: number;            // 14周期RSI
  volume: number;            // 成交量
  open_interest?: number;    // 未平仓合约
  funding_rate?: number;     // 资金费率
  atr_3?: number;            // 3周期ATR
  atr_14?: number;           // 14周期ATR
}
```

#### 5. AI决策记录表 (agent_decisions)
```typescript
interface AgentDecision {
  id: number;                // 主键
  timestamp: string;         // 决策时间
  iteration: number;         // 迭代次数
  market_analysis: string;   // 市场分析
  decision: string;          // 具体决策(JSON)
  actions_taken: string;     // 执行动作
  account_value: number;     // 账户价值
  positions_count: number;   // 持仓数量
}
```

#### 6. 系统配置表 (system_config)
```typescript
interface SystemConfig {
  id: number;                // 主键
  key: string;               // 配置键 (唯一)
  value: string;             // 配置值
  updated_at: string;        // 更新时间
}
```

## 🔄 数据同步策略

### 同步时机
1. **系统启动时**: 全量同步持仓和订单状态
2. **交易循环前**: 同步最新价格和持仓信息
3. **异常恢复时**: 检查并修复数据不一致
4. **定时同步**: 每30分钟执行一次完整性检查

### 同步流程
```
开始同步
    ↓
获取Gate.io数据
    ↓
对比本地数据
    ↓
发现差异？ → 是 → 更新本地数据
    ↓否
同步完成
```

### 冲突解决策略
- **价格冲突**: 以Gate.io实时价格为准
- **持仓冲突**: 以Gate.io实际持仓为准
- **订单冲突**: 以Gate.io订单状态为准
- **时间冲突**: 以Gate.io服务器时间为准

## 🧪 数据完整性检查

### 检查项目
1. **持仓一致性**: 本地持仓 vs Gate.io持仓
2. **订单状态**: 本地订单状态 vs 实际状态
3. **价格一致性**: 本地缓存价格 vs 实时价格
4. **账户余额**: 本地计算 vs Gate.io账户
5. **盈亏计算**: 本地计算 vs 实际盈亏

### 修复机制
```typescript
interface ConsistencyCheck {
  positionsMatch: boolean;      // 持仓是否一致
  ordersMatch: boolean;         // 订单是否一致
  pricesMatch: boolean;         // 价格是否一致
  balanceMatch: boolean;        // 余额是否一致
  issuesFound: string[];        // 发现的问题列表
  repairsMade: string[];        // 执行的修复操作
}
```

## 🔍 常见问题 (FAQ)

### Q: 数据库文件在哪里？
**A**: 默认位置 `./.voltagent/trading.db`，可通过 `DATABASE_URL` 环境变量修改路径。

### Q: 如何备份数据库？
**A**: 直接复制 `.voltagent/trading.db` 文件即可，SQLite支持热备份。

### Q: 数据同步失败怎么办？
**A**: 检查步骤：1) API密钥是否正确 2) 网络连接是否正常 3) Gate.io服务状态 4) 查看错误日志

### Q: 如何重置所有数据？
**A**: 执行 `npm run db:reset` 会清空所有数据并重新初始化表结构。

### Q: 数据库性能如何优化？
**A**: 优化措施：1) 添加适当索引 2) 定期VACUUM 3) 限制历史数据保留期 4) 使用WAL模式

## 📋 相关文件清单

### 核心文件
- `schema.ts` - 数据库表结构定义
- `init.ts` - 数据库初始化
- `reset.ts` - 数据库重置
- `sync-from-gate.ts` - Gate.io数据同步
- `close-and-reset.ts` - 平仓并重置

### 辅助文件
- `check-trades.ts` - 交易记录检查
- `add-fee-column.ts` - 添加手续费字段
- `add-peak-pnl-column.ts` - 添加峰值盈亏字段

### 配置文件
- `../../.env` - 数据库连接配置
- `../config/riskParams.ts` - 风险参数 (被数据模块使用)

## 📝 变更记录

### 2025-10-30 初始版本
- **新增**: 数据库操作模块文档
- **新增**: 数据模型详细设计
- **新增**: 同步策略与完整性检查
- **新增**: 常见问题与性能优化

---

**模块状态**: ✅ 活跃开发  
**维护者**: AI Trading System  
**更新时间**: 2025-10-30 11:29:10

## 🎯 下一步优化方向

1. **性能优化**: 数据库索引优化、查询性能提升
2. **数据归档**: 历史数据自动归档与压缩
3. **备份机制**: 自动备份与恢复策略
4. **监控告警**: 数据库性能监控与告警
5. **数据可视化**: 内置图表与报表功能

**预计开发周期**: 1-2周
**优先级**: 高 (性能) → 中 (备份) → 低 (可视化)

---

**数据库性能指标**:
- 查询响应时间: ≤100ms (单表)
- 并发处理能力: ≥100 QPS
- 存储空间: ≤1GB (1年数据)
- 备份时间: ≤5分钟

**数据质量指标**:
- 数据一致性: ≥99.9%
- 同步成功率: ≥99.5%
- 数据完整性: 100%
- 错误恢复时间: ≤1分钟

**运维监控**:
```bash
# 查看数据库状态
npm run db:status

# 检查数据一致性
npm run db:check-consistency

# 手动同步数据
npm run db:sync

# 查看数据库大小
du -h .voltagent/trading.db
```

**数据库维护脚本**:
```bash
#!/bin/bash
# 数据库备份脚本
BACKUP_DIR="./backups"
DATE=$(date +%Y%m%d_%H%M%S)
mkdir -p $BACKUP_DIR
cp .voltagent/trading.db $BACKUP_DIR/trading_$DATE.db

# 清理7天前的备份
find $BACKUP_DIR -name "*.db" -mtime +7 -delete
```

**性能优化建议**:
```sql
-- 添加复合索引提高查询性能
CREATE INDEX idx_trades_symbol_timestamp ON trades(symbol, timestamp);
CREATE INDEX idx_positions_side_symbol ON positions(side, symbol);
CREATE INDEX idx_signals_symbol_time ON trading_signals(symbol, timestamp);

-- 使用WAL模式提高并发性能
PRAGMA journal_mode=WAL;
PRAGMA synchronous=NORMAL;

-- 定期优化数据库
VACUUM;
PRAGMA optimize;
```

**数据保留策略**:
- 交易记录: 永久保留
- 账户历史: 保留2年
- 技术指标: 保留6个月
- AI决策记录: 保留1年
- 系统日志: 保留3个月

**扩展性考虑**:
- 支持分库分表 (未来)
- 支持读写分离 (未来)
- 支持分布式部署 (未来)
- 支持实时备份 (未来)

**安全考虑**:
- 数据库文件权限控制
- 敏感数据加密存储
- 访问日志记录
- 异常操作监控

---

**数据库架构图**:
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   交易代理       │    │   数据库模块     │    │   Gate.io API  │
│  (AI决策)       │◄──►│  (LibSQL)       │◄──►│  (外部数据)    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
       │                       │                       │
       ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   交易工具       │    │   数据完整性     │    │   实时监控     │
│  (执行层)       │    │  (检查修复)     │    │  (同步验证)    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

**数据生命周期**:
```
数据产生 → 本地存储 → 同步验证 → 完整性检查 → 归档备份 → 清理删除
    │         │         │           │           │         │
交易记录    数据库    Gate.io     定时任务     历史数据   过期清理
AI决策      索引      对比        自动修复     压缩存储   空间回收
```

**故障恢复**:
- 数据损坏: 从备份恢复
- 同步失败: 手动重新同步
- 性能下降: 重建索引+优化
- 空间不足: 清理历史数据

---

**最佳实践**:
1. **定期备份**: 每日自动备份数据库
2. **监控告警**: 实时监控数据库性能
3. **数据验证**: 定期检查数据完整性
4. **性能优化**: 定期重建索引和优化
5. **容量规划**: 预估数据增长和存储需求
6. **文档维护**: 保持数据模型文档更新

**常见错误处理**:
```typescript
// 数据库锁定错误
try {
  await dbClient.execute(sql);
} catch (error) {
  if (error.code === 'SQLITE_BUSY') {
    // 等待重试
    await sleep(100);
    return retry();
  }
  throw error;
}

// 数据一致性错误
if (!positionsMatch) {
  logger.warn('持仓数据不一致，执行修复');
  await repairPositions();
}
```

**数据迁移策略**:
- 版本控制: 使用迁移脚本管理版本
- 回滚机制: 支持回滚到上一版本
- 兼容性: 保持向后兼容性
- 测试验证: 迁移前充分测试
- 备份恢复: 迁移前完整备份

---

**总结**: 数据库模块是整个交易系统的数据基础，其稳定性、完整性和性能直接影响交易决策的准确性和系统的可靠性。通过合理的数据模型设计、完善的同步机制和严格的完整性检查，确保数据的准确性和一致性，为AI交易代理提供可靠的数据支持。同时，通过性能优化、监控告警和故障恢复机制，保障系统在高并发和大数据量场景下的稳定运行。持续的数据质量监控和优化是保持系统健康运行的关键。`
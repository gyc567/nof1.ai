[根目录](../../CLAUDE.md) > [src](../) > **agents**

# AI 交易代理模块

## 🎯 模块职责

AI交易代理是系统的**核心决策大脑**，负责：
- 基于多时间框架技术分析生成交易决策
- 执行风险管理与头寸规模计算  
- 管理交易生命周期（开仓、监控、平仓）
- 记录决策逻辑用于后续分析优化

## 🚀 入口与启动

**主入口**: `tradingAgent.ts`

```typescript
// 创建交易代理实例
const agent = createTradingAgent();

// 生成交易提示词
const prompt = generateTradingPrompt(marketData, accountState);

// 执行AI决策
const decision = await agent.invoke(prompt);
```

## 🔌 对外接口

### 核心函数

| 函数 | 用途 | 参数 | 返回 |
|-----|------|------|------|
| `createTradingAgent()` | 创建AI代理实例 | 无 | Agent实例 |
| `generateTradingPrompt()` | 生成交易决策提示词 | marketData, accountState | 格式化提示词 |
| `getAccountRiskConfig()` | 获取账户风险配置 | 无 | AccountRiskConfig |

### 风险配置接口

```typescript
interface AccountRiskConfig {
  stopLossUsdt: number;      // 账户止损线 (默认50USDT)
  takeProfitUsdt: number;    // 账户止盈线 (默认10000USDT)  
  syncOnStartup: boolean;    // 启动时同步配置
}
```

## 🛠️ 关键依赖与配置

### 核心依赖
```json
{
  "@voltagent/core": "^1.0.0",      // AI代理框架
  "@voltagent/libsql": "^1.0.0",    // 记忆存储
  "@voltagent/logger": "^1.0.0",    // 日志系统
  "@openrouter/ai-sdk-provider": "^1.2.0"  // AI模型提供者
}
```

### 环境变量配置
```bash
# AI模型配置
OPENROUTER_API_KEY=your_openrouter_api_key
AI_MODEL=openrouter/anthropic/claude-3.5-haiku

# 账户风险控制
ACCOUNT_STOP_LOSS_USDT=50
ACCOUNT_TAKE_PROFIT_USDT=10000
SYNC_CONFIG_ON_STARTUP=true
```

## 🧠 AI决策逻辑

### 核心交易哲学
```
风险控制优先 → 多时间框架分析 → 严格止损止盈 → 资金管理
```

### 决策流程
1. **市场环境分析** - 判断趋势/震荡/不确定状态
2. **风险评估** - 确定当前市场风险等级 (低/中/高)
3. **机会识别** - 基于技术指标寻找交易信号
4. **头寸计算** - 根据风险承受能力确定仓位
5. **订单执行** - 设置止损止盈，执行交易
6. **监控管理** - 持续跟踪持仓风险收益比

### 关键指标权重
- **趋势确认**: ≥2个时间框架共振 (必须)
- **风险收益比**: ≥1:2 (最低要求)
- **胜率预期**: ≥50% (历史统计)
- **最大回撤**: ≤2% (单笔风险)

## 📊 数据模型

### Agent决策记录结构
```typescript
interface AgentDecision {
  id: number;
  timestamp: string;          // 决策时间 (中国时区)
  iteration: number;          // 交易循环迭代次数
  market_analysis: string;    // 市场分析内容
  decision: string;           // 具体决策 (JSON格式)
  actions_taken: string;      // 执行的动作列表
  account_value: number;      // 当时账户总值
  positions_count: number;    // 当时持仓数量
}
```

## 🧪 测试策略

### 回测验证
- **历史数据**: 使用Gate.io历史K线数据
- **策略验证**: 对比不同参数组合的表现
- **风险测试**: 模拟极端市场情况下的表现

### 实盘验证
- **小资金测试**: 初始资金控制在100USDT以内
- **逐步加仓**: 验证策略有效性后逐步增加资金
- **持续监控**: 每日检查关键性能指标

## 🔍 常见问题 (FAQ)

### Q: AI代理如何选择交易时机？
**A**: 基于多时间框架技术分析，至少需要2-3个时间框架出现共振信号，同时考虑成交量、资金费率等因素。

### Q: 如何控制风险？
**A**: 多层风险控制：账户级别(止损/止盈)、单笔交易(2%最大风险)、持仓管理(最大5个仓位)、杠杆限制(最大15x)。

### Q: AI决策错误怎么办？
**A**: 系统记录每笔决策的完整逻辑，可以通过回测分析错误原因，持续优化提示词和决策逻辑。

### Q: 支持哪些交易策略？
**A**: 目前支持保守型(v1)和平衡型(v2)两种策略，可通过修改提示词切换或创建新策略。

## 📋 相关文件清单

### 核心文件
- `tradingAgent.ts` - AI代理主逻辑
- `prompts/` - 策略提示词模板 (如有)

### 配置文件  
- `../../.env` - 环境变量配置
- `../config/riskParams.ts` - 风险参数配置

### 依赖文件
- `../database/schema.ts` - 决策记录表结构
- `../utils/timeUtils.ts` - 时区处理工具
- `../tools/trading/` - 交易工具集

## 📝 变更记录

### 2025-10-30 初始版本
- **新增**: AI交易代理模块文档
- **新增**: 决策逻辑与风险控制说明
- **新增**: 接口定义与配置说明
- **新增**: 测试策略与FAQ

---

**模块状态**: ✅ 活跃开发  
**维护者**: AI Trading System  
**更新时间**: 2025-10-30 11:29:10
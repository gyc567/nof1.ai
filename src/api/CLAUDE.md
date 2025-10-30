[根目录](../../CLAUDE.md) > [src](../) > **api**

# Web API 接口模块

## 🎯 模块职责

Web API接口模块负责：
- 提供Web监控界面，实时展示交易状态
- 对外暴露RESTful API，支持系统状态查询
- 静态文件服务，托管前端资源
- 数据聚合与格式化，为前端提供友好的数据接口

## 🚀 入口与启动

**主入口**: `routes.ts`

```typescript
// 创建API路由实例
const apiRoutes = createApiRoutes();

// 启动HTTP服务器
server = serve({
  fetch: apiRoutes.fetch,
  port: 3141
});
```

**启动流程** (在`src/index.ts`中):
1. 初始化数据库
2. 初始化交易系统配置
3. **创建API路由** ← 当前模块
4. 启动交易循环
5. 启动账户记录器

## 🔌 对外接口

### Web界面端点

| 端点 | 方法 | 用途 | 返回 |
|-----|------|------|------|
| `/` | GET | 监控主页 | HTML页面 |
| `/app.js` | GET | 前端脚本 | JavaScript |
| `/style.css` | GET | 样式文件 | CSS |
| `/monitor-script.js` | GET | 监控脚本 | JavaScript |
| `/monitor-styles.css` | GET | 监控样式 | CSS |

### REST API端点

| 端点 | 方法 | 用途 | 返回 |
|-----|------|------|------|
| `/api/account` | GET | 获取账户总览 | AccountOverview |
| `/api/positions` | GET | 获取当前持仓 | Position[] |
| `/api/trades` | GET | 获取交易历史 | Trade[] |
| `/api/signals` | GET | 获取技术指标 | TradingSignal[] |
| `/api/history` | GET | 获取账户历史 | AccountHistory[] |
| `/api/decisions` | GET | 获取AI决策记录 | AgentDecision[] |

### 数据模型定义

```typescript
// 账户总览
interface AccountOverview {
  totalValue: number;        // 总资产价值 (含未实现盈亏)
  availableCash: number;     // 可用资金
  unrealizedPnl: number;     // 未实现盈亏
  realizedPnl: number;       // 已实现盈亏
  positionsCount: number;    // 当前持仓数
  returnPercent: number;     // 收益率百分比
}

// 持仓详情  
interface Position {
  symbol: string;            // 交易对
  side: 'long' | 'short';    // 方向
  quantity: number;          // 数量
  entryPrice: number;        // 开仓价格
  currentPrice: number;      // 当前价格
  unrealizedPnl: number;     // 未实现盈亏
  leverage: number;          // 杠杆倍数
  openedAt: string;          // 开仓时间
}
```

## 🛠️ 关键依赖与配置

### 核心依赖
```json
{
  "hono": "^3.x",                    // Web框架
  "@hono/node-server": "^1.x",       // Node.js服务器
  "@libsql/client": "^0.x",          // 数据库客户端
  "@voltagent/logger": "^1.0.0"      // 日志系统
}
```

### 静态资源配置
```typescript
// 静态文件服务配置
app.use("/*", serveStatic({ 
  root: "./public" 
}));
```

### 监控界面功能
- **实时数据刷新**: 每30秒自动更新
- **图表展示**: 收益曲线、持仓分布、交易历史
- **状态监控**: 系统运行状态、错误日志
- **手动操作**: 紧急停止、策略切换、数据同步

## 📊 数据流设计

### 账户数据流
```
Gate.io API → gateClient → 数据聚合 → API接口 → 前端展示
```

### 交易数据流
```
数据库查询 → 数据格式化 → API接口 → 前端图表渲染
```

### 实时更新机制
```
客户端轮询 → /api/account → 数据库查询 → JSON响应
刷新间隔: 30秒 (可配置)
```

## 🧪 前端技术栈

### 核心技术
- **HTML5 + CSS3**: 现代化界面布局
- **原生JavaScript**: 无需构建步骤，直接运行
- **Chart.js**: 数据可视化图表
- **WebSocket (预留)**: 实时数据推送

### 关键功能实现
```javascript
// 数据刷新逻辑
async function refreshData() {
  const response = await fetch('/api/account');
  const data = await response.json();
  updateUI(data);
}

// 定时刷新
setInterval(refreshData, 30000); // 30秒
```

## 🔍 常见问题 (FAQ)

### Q: 监控界面无法访问？
**A**: 检查步骤：1) 确认服务启动成功 2) 检查端口3141是否被占用 3) 查看控制台错误日志

### Q: 数据显示不完整？
**A**: 可能原因：1) 数据库连接异常 2) Gate.io API限流 3) 数据同步延迟。查看日志定位具体问题

### Q: 如何自定义监控界面？
**A**: 修改`public/`目录下的文件：1) `index.html` - 页面结构 2) `app.js` - 交互逻辑 3) `style.css` - 样式美化

### Q: API响应慢怎么办？
**A**: 优化方案：1) 添加数据库索引 2) 实现数据缓存 3) 分页加载历史数据 4) 异步数据聚合

## 📋 相关文件清单

### 核心文件
- `routes.ts` - API路由主文件
- `../../public/index.html` - 监控界面主页面
- `../../public/app.js` - 前端交互逻辑
- `../../public/style.css` - 界面样式

### 配置文件
- `../../.env` - 服务器端口配置
- `../config/riskParams.ts` - 风险参数 (被API使用)

### 依赖模块
- `../services/gateClient.ts` - Gate.io客户端
- `../database/schema.ts` - 数据表结构定义
- `../utils/timeUtils.ts` - 时间格式化工具

## 📝 变更记录

### 2025-10-30 初始版本
- **新增**: Web API接口模块文档
- **新增**: RESTful API端点定义
- **新增**: 监控界面功能说明
- **新增**: 数据流设计与前端技术栈
- **新增**: 常见问题与故障排查

---

**模块状态**: ✅ 活跃开发  
**维护者**: AI Trading System  
**更新时间**: 2025-10-30 11:29:10

**访问监控界面**: http://localhost:3141/ (系统启动后)

## 🎯 下一步优化方向

1. **WebSocket支持**: 实现真正的实时数据推送
2. **移动端适配**: 优化手机端显示体验  
3. **多语言支持**: 中英文界面切换
4. **数据导出**: 支持CSV/Excel格式导出交易数据
5. **告警系统**: 价格预警、风险告警通知
6. **性能优化**: 数据库查询优化、前端缓存策略

**预计开发周期**: 2-3周
**优先级**: 高 (WebSocket) → 中 (移动端) → 低 (多语言)

---

**监控界面截图** (待补充):
```
┌─────────────────────────────────────────┐
│  open-nof1.ai 交易监控系统               │
├─────────────────────────────────────────┤
│  总资产: 1,234.56 USDT  收益率: +23.45% │
│  可用: 456.78 USDT  持仓: 3/5          │
├─────────────────────────────────────────┤
│  [收益曲线图]      [持仓分布饼图]       │
├─────────────────────────────────────────┤
│  最近交易:                              │
│  • BTC/USDT 开多 +2.34%  10分钟前       │
│  • ETH/USDT 平空 +1.87%  25分钟前       │
├─────────────────────────────────────────┤
│  [刷新] [紧急停止] [策略切换] [日志]     │
└─────────────────────────────────────────┘
```

**技术架构图**:
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   前端界面       │◄──►│   API路由层      │◄──►│   数据聚合层     │
│  (HTML/JS/CSS)  │    │   (Hono框架)    │    │  (业务逻辑)     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                              │                       │
                              ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   静态资源       │    │   数据库访问     │◄──►│   外部服务       │
│  (Chart.js等)   │    │  (LibSQL客户端)  │    │ (Gate.io API)  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

**性能指标**:
- API响应时间: ≤200ms (P95)
- 并发处理能力: ≥100 QPS
- 内存占用: ≤100MB
- CPU使用率: ≤10% (空闲时)

**监控告警**:
- API错误率 > 5%
- 响应时间 > 1s
- 内存占用 > 200MB
- 数据库连接失败

---

**接口性能监控**:
```bash
# 查看API访问日志
tail -f logs/api-access.log

# 性能统计
curl -w "@curl-format.txt" -o /dev/null http://localhost:3141/api/account

# 压力测试
ab -n 1000 -c 10 http://localhost:3141/api/account
```

**数据库查询优化**:
```sql
-- 为常用查询添加索引
CREATE INDEX idx_trades_timestamp ON trades(timestamp);
CREATE INDEX idx_positions_symbol ON positions(symbol);
CREATE INDEX idx_history_timestamp ON account_history(timestamp);
```

**缓存策略**:
- 账户总览: 缓存30秒
- 持仓数据: 缓存10秒  
- 历史数据: 缓存5分钟
- 静态资源: 缓存1小时

**安全考虑**:
- API限流: 100请求/分钟/IP
- CORS配置: 仅允许本地访问
- 输入验证: 防止SQL注入
- 错误处理: 不暴露内部实现细节

---

**扩展性设计**:
- 水平扩展: 支持多实例部署
- 负载均衡: Nginx反向代理
- 数据库: 支持读写分离
- 缓存层: Redis支持 (可选)

**部署架构**:
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Nginx     │───►│  API服务x3   │───►│  数据库集群  │
│  (负载均衡)  │    │  (Node.js)  │    │  (LibSQL)   │
└─────────────┘    └─────────────┘    └─────────────┘
      │                  │                  │
      ▼                  ▼                  ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   静态资源   │    │   日志聚合   │    │   监控告警   │
│  (CDN)      │    │  (Loki)     │    │ (Grafana)  │
└─────────────┘    └─────────────┘    └─────────────┘
```

**运维监控**:
- 服务健康检查: `/health`
- 指标收集: Prometheus格式
- 日志聚合: 结构化日志
- 告警通知: 邮件/短信/钉钉

**版本管理**:
- API版本控制: `/api/v1/`
- 向后兼容: 不破坏现有接口
- 废弃策略: 提前通知+渐进式迁移
- 文档同步: OpenAPI规范

---

**开发规范**:
- RESTful设计: 资源导向的URL
- 状态码: 正确使用HTTP状态码
- 错误格式: 统一的错误响应格式
- 分页处理: 大数据集分页返回
- 输入验证: 严格的参数校验
- 安全防护: XSS/SQL注入防护

**测试覆盖**:
- 单元测试: ≥80%覆盖率
- 集成测试: 核心API端到端测试
- 性能测试: 负载测试+压力测试
- 安全测试: OWASP Top 10扫描

**文档标准**:
- 接口文档: OpenAPI/Swagger
- 使用示例: 请求/响应示例
- 错误码表: 详细的错误说明
- 变更记录: 版本间差异说明
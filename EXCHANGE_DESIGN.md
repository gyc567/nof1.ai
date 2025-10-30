# 多交易所支持设计文档

## 1. 核心接口定义

### IExchange 统一接口

```typescript
// src/exchanges/types/index.ts
export interface IExchange {
  // 基础信息
  readonly name: ExchangeName;
  readonly isTestnet: boolean;
  
  // 市场数据
  getTicker(symbol: string): Promise<TickerData>;
  getCandles(symbol: string, interval: string, limit: number): Promise<CandleData[]>;
  getOrderBook(symbol: string, limit?: number): Promise<OrderBookData>;
  getFundingRate(symbol: string): Promise<FundingRateData>;
  
  // 账户信息
  getAccount(): Promise<AccountData>;
  getPositions(): Promise<PositionData[]>;
  getOpenOrders(symbol?: string): Promise<OrderData[]>;
  
  // 交易操作
  placeOrder(params: PlaceOrderParams): Promise<OrderResult>;
  cancelOrder(orderId: string): Promise<CancelResult>;
  getOrder(orderId: string): Promise<OrderData>;
  
  // 工具方法
  validateSymbol(symbol: string): boolean;
  normalizeSymbol(symbol: string): string;
  
  // 连接管理
  connect(): Promise<void>;
  disconnect(): Promise<void>;
  isConnected(): boolean;
}
```

### 标准化数据类型

```typescript
// src/exchanges/types/common.ts
export type ExchangeName = 'gate' | 'okx' | 'binance';

export interface TickerData {
  symbol: string;
  price: number;
  change24h: number;
  volume24h: number;
  markPrice?: number;
  timestamp: number;
}

export interface CandleData {
  timestamp: number;
  open: number;
  high: number;
  low: number;
  close: number;
  volume: number;
}

export interface AccountData {
  totalBalance: number;
  availableBalance: number;
  unrealizedPnl: number;
  marginUsed: number;
}

export interface PositionData {
  symbol: string;
  side: 'long' | 'short';
  size: number;
  entryPrice: number;
  markPrice: number;
  unrealizedPnl: number;
  leverage: number;
  liquidationPrice?: number;
}

export interface PlaceOrderParams {
  symbol: string;
  side: 'buy' | 'sell';
  type: 'market' | 'limit';
  size: number;
  price?: number;
  reduceOnly?: boolean;
  stopLoss?: number;
  takeProfit?: number;
}

export interface OrderResult {
  orderId: string;
  symbol: string;
  side: 'buy' | 'sell';
  size: number;
  price?: number;
  status: 'pending' | 'filled' | 'cancelled';
  timestamp: number;
}
```

## 2. 工厂模式实现

### ExchangeFactory

```typescript
// src/exchanges/factory/ExchangeFactory.ts
import { IExchange, ExchangeName } from '../types';
import { ExchangeRegistry } from './ExchangeRegistry';
import { ExchangeConfig } from '../types/config';

export class ExchangeFactory {
  private static instance: ExchangeFactory;
  private registry: ExchangeRegistry;
  
  private constructor() {
    this.registry = new ExchangeRegistry();
  }
  
  public static getInstance(): ExchangeFactory {
    if (!ExchangeFactory.instance) {
      ExchangeFactory.instance = new ExchangeFactory();
    }
    return ExchangeFactory.instance;
  }
  
  public createExchange(name: ExchangeName, config: ExchangeConfig): IExchange {
    const ExchangeClass = this.registry.getExchange(name);
    if (!ExchangeClass) {
      throw new Error(`Exchange ${name} not supported`);
    }
    
    return new ExchangeClass(config);
  }
  
  public getSupportedExchanges(): ExchangeName[] {
    return this.registry.getSupportedExchanges();
  }
}
```

### ExchangeRegistry

```typescript
// src/exchanges/factory/ExchangeRegistry.ts
import { ExchangeName, IExchange } from '../types';
import { GateExchange } from '../implementations/GateExchange';
import { OKXExchange } from '../implementations/OKXExchange';
import { BinanceExchange } from '../implementations/BinanceExchange';

type ExchangeConstructor = new (config: any) => IExchange;

export class ExchangeRegistry {
  private exchanges: Map<ExchangeName, ExchangeConstructor> = new Map();
  
  constructor() {
    this.registerDefaultExchanges();
  }
  
  private registerDefaultExchanges(): void {
    this.register('gate', GateExchange);
    this.register('okx', OKXExchange);
    this.register('binance', BinanceExchange);
  }
  
  public register(name: ExchangeName, exchangeClass: ExchangeConstructor): void {
    this.exchanges.set(name, exchangeClass);
  }
  
  public getExchange(name: ExchangeName): ExchangeConstructor | undefined {
    return this.exchanges.get(name);
  }
  
  public getSupportedExchanges(): ExchangeName[] {
    return Array.from(this.exchanges.keys());
  }
}
```

## 3. Gate.io 实现示例

### GateExchange

```typescript
// src/exchanges/implementations/GateExchange.ts
import { IExchange, ExchangeName, TickerData, AccountData } from '../types';
import { GateAdapter } from '../adapters/GateAdapter';
import { BaseExchange } from './BaseExchange';

export class GateExchange extends BaseExchange implements IExchange {
  public readonly name: ExchangeName = 'gate';
  private adapter: GateAdapter;
  
  constructor(config: GateExchangeConfig) {
    super(config);
    this.adapter = new GateAdapter(config);
  }
  
  public async getTicker(symbol: string): Promise<TickerData> {
    const normalizedSymbol = this.normalizeSymbol(symbol);
    const rawTicker = await this.adapter.getFuturesTicker(normalizedSymbol);
    
    return {
      symbol,
      price: Number.parseFloat(rawTicker.last || "0"),
      change24h: Number.parseFloat(rawTicker.change_percentage || "0"),
      volume24h: Number.parseFloat(rawTicker.volume_24h || "0"),
      markPrice: Number.parseFloat(rawTicker.mark_price || "0"),
      timestamp: Date.now()
    };
  }
  
  public async getAccount(): Promise<AccountData> {
    const rawAccount = await this.adapter.getFuturesAccount();
    
    return {
      totalBalance: Number.parseFloat(rawAccount.total || "0"),
      availableBalance: Number.parseFloat(rawAccount.available || "0"),
      unrealizedPnl: Number.parseFloat(rawAccount.unrealisedPnl || "0"),
      marginUsed: Number.parseFloat(rawAccount.positionMargin || "0")
    };
  }
  
  public normalizeSymbol(symbol: string): string {
    // BTC -> BTC_USDT
    return symbol.includes('_') ? symbol : `${symbol}_USDT`;
  }
  
  public validateSymbol(symbol: string): boolean {
    const normalized = this.normalizeSymbol(symbol);
    return /^[A-Z0-9]+_USDT$/.test(normalized);
  }
}
```

### GateAdapter (适配器层)

```typescript
// src/exchanges/adapters/GateAdapter.ts
import { GateClient } from '../../services/gateClient';
import { withRetry } from '../utils/retry';

export class GateAdapter {
  private client: GateClient;
  
  constructor(config: GateExchangeConfig) {
    this.client = new GateClient(config.apiKey, config.apiSecret);
  }
  
  @withRetry(3, 1000)
  public async getFuturesTicker(contract: string) {
    return await this.client.getFuturesTicker(contract);
  }
  
  @withRetry(3, 1000)
  public async getFuturesAccount() {
    return await this.client.getFuturesAccount();
  }
  
  // ... 其他方法
}
```

## 4. 统一服务层

### ExchangeService

```typescript
// src/services/exchangeService.ts
import { IExchange, ExchangeName } from '../exchanges/types';
import { ExchangeFactory } from '../exchanges/factory/ExchangeFactory';
import { createPinoLogger } from '@voltagent/logger';

const logger = createPinoLogger({ name: 'exchange-service' });

export class ExchangeService {
  private exchange: IExchange;
  private factory: ExchangeFactory;
  
  constructor() {
    this.factory = ExchangeFactory.getInstance();
    this.initializeExchange();
  }
  
  private initializeExchange(): void {
    const exchangeName = this.getConfiguredExchange();
    const config = this.getExchangeConfig(exchangeName);
    
    this.exchange = this.factory.createExchange(exchangeName, config);
    logger.info(`Initialized exchange: ${exchangeName}`);
  }
  
  private getConfiguredExchange(): ExchangeName {
    const exchange = process.env.EXCHANGE_NAME as ExchangeName;
    if (!exchange) {
      return 'gate'; // 默认使用Gate.io，保持向后兼容
    }
    
    const supported = this.factory.getSupportedExchanges();
    if (!supported.includes(exchange)) {
      throw new Error(`Unsupported exchange: ${exchange}`);
    }
    
    return exchange;
  }
  
  private getExchangeConfig(exchangeName: ExchangeName) {
    switch (exchangeName) {
      case 'gate':
        return {
          apiKey: process.env.GATE_API_KEY!,
          apiSecret: process.env.GATE_API_SECRET!,
          isTestnet: process.env.GATE_USE_TESTNET === 'true'
        };
      case 'okx':
        return {
          apiKey: process.env.OKX_API_KEY!,
          apiSecret: process.env.OKX_API_SECRET!,
          passphrase: process.env.OKX_PASSPHRASE!,
          isTestnet: process.env.OKX_USE_TESTNET === 'true'
        };
      case 'binance':
        return {
          apiKey: process.env.BINANCE_API_KEY!,
          apiSecret: process.env.BINANCE_API_SECRET!,
          isTestnet: process.env.BINANCE_USE_TESTNET === 'true'
        };
      default:
        throw new Error(`No config for exchange: ${exchangeName}`);
    }
  }
  
  // 统一接口方法
  public async getTicker(symbol: string) {
    return await this.exchange.getTicker(symbol);
  }
  
  public async getAccount() {
    return await this.exchange.getAccount();
  }
  
  public async getPositions() {
    return await this.exchange.getPositions();
  }
  
  public async placeOrder(params: any) {
    return await this.exchange.placeOrder(params);
  }
  
  // ... 其他方法
}
```

## 5. 向后兼容方案

### 兼容层实现

```typescript
// src/services/gateClient.ts (保持现有接口不变)
import { ExchangeService } from './exchangeService';

// 保持现有的createGateClient函数，内部使用新架构
export function createGateClient(): GateClient {
  // 如果配置的是Gate.io，使用新架构
  if (process.env.EXCHANGE_NAME === 'gate' || !process.env.EXCHANGE_NAME) {
    return new GateClientWrapper();
  }
  
  // 否则返回原有实现
  return new GateClient(
    process.env.GATE_API_KEY!,
    process.env.GATE_API_SECRET!
  );
}

class GateClientWrapper {
  private exchangeService: ExchangeService;
  
  constructor() {
    this.exchangeService = new ExchangeService();
  }
  
  // 保持原有方法签名，内部调用新架构
  async getFuturesTicker(contract: string) {
    const symbol = contract.replace('_USDT', '');
    const ticker = await this.exchangeService.getTicker(symbol);
    
    // 转换为原有格式
    return {
      last: ticker.price.toString(),
      change_percentage: ticker.change24h.toString(),
      volume_24h: ticker.volume24h.toString(),
      mark_price: ticker.markPrice?.toString()
    };
  }
  
  async getFuturesAccount() {
    const account = await this.exchangeService.getAccount();
    
    // 转换为原有格式
    return {
      total: account.totalBalance.toString(),
      available: account.availableBalance.toString(),
      unrealisedPnl: account.unrealizedPnl.toString()
    };
  }
  
  // ... 其他方法保持相同接口
}
```

## 6. 环境变量配置

### 新增环境变量

```bash
# 交易所选择
EXCHANGE_NAME=gate  # gate | okx | binance

# Gate.io (现有配置保持不变)
GATE_API_KEY=your_gate_api_key
GATE_API_SECRET=your_gate_api_secret
GATE_USE_TESTNET=true

# OKX
OKX_API_KEY=your_okx_api_key
OKX_API_SECRET=your_okx_api_secret
OKX_PASSPHRASE=your_okx_passphrase
OKX_USE_TESTNET=true

# Binance
BINANCE_API_KEY=your_binance_api_key
BINANCE_API_SECRET=your_binance_api_secret
BINANCE_USE_TESTNET=true
```

## 7. 测试策略

### 单元测试

```typescript
// tests/exchanges/unit/GateExchange.test.ts
describe('GateExchange', () => {
  let exchange: GateExchange;
  let mockAdapter: jest.Mocked<GateAdapter>;
  
  beforeEach(() => {
    mockAdapter = createMockGateAdapter();
    exchange = new GateExchange(mockConfig);
    (exchange as any).adapter = mockAdapter;
  });
  
  describe('getTicker', () => {
    it('should return normalized ticker data', async () => {
      mockAdapter.getFuturesTicker.mockResolvedValue(mockRawTicker);
      
      const result = await exchange.getTicker('BTC');
      
      expect(result).toEqual({
        symbol: 'BTC',
        price: 50000,
        change24h: 2.5,
        volume24h: 1000000,
        markPrice: 50010,
        timestamp: expect.any(Number)
      });
    });
    
    it('should handle API errors gracefully', async () => {
      mockAdapter.getFuturesTicker.mockRejectedValue(new Error('API Error'));
      
      await expect(exchange.getTicker('BTC')).rejects.toThrow('API Error');
    });
  });
});
```

### 集成测试

```typescript
// tests/exchanges/integration/ExchangeService.test.ts
describe('ExchangeService Integration', () => {
  let service: ExchangeService;
  
  beforeEach(() => {
    process.env.EXCHANGE_NAME = 'gate';
    service = new ExchangeService();
  });
  
  it('should switch exchanges based on configuration', async () => {
    // Test Gate.io
    process.env.EXCHANGE_NAME = 'gate';
    const gateService = new ExchangeService();
    expect(gateService.getCurrentExchange()).toBe('gate');
    
    // Test OKX
    process.env.EXCHANGE_NAME = 'okx';
    const okxService = new ExchangeService();
    expect(okxService.getCurrentExchange()).toBe('okx');
  });
});
```

## 8. 迁移计划

### 阶段1：基础架构 (Week 1)
- [ ] 创建接口定义
- [ ] 实现工厂模式
- [ ] 创建Gate.io适配器
- [ ] 基础测试覆盖

### 阶段2：向后兼容 (Week 2)
- [ ] 实现兼容层
- [ ] 现有功能测试
- [ ] 性能基准测试
- [ ] 文档更新

### 阶段3：新交易所支持 (Week 3-4)
- [ ] OKX实现
- [ ] Binance实现
- [ ] 完整测试覆盖
- [ ] 集成测试

### 阶段4：优化和部署 (Week 5)
- [ ] 性能优化
- [ ] 错误处理完善
- [ ] 生产环境测试
- [ ] 正式发布

## 9. 质量保证

### 代码质量
- TypeScript严格模式
- ESLint + Prettier
- 100%测试覆盖率
- 代码审查流程

### 性能要求
- API响应时间 < 500ms
- 内存使用 < 100MB
- 错误率 < 0.1%
- 99.9%可用性

### 安全要求
- API密钥加密存储
- 请求签名验证
- 速率限制保护
- 错误信息脱敏
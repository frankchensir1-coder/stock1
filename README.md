# Monitoring ICT Order Flow

面向 TradingView 1 小时图的 ICT 订单流监测与 Outlook 邮件提醒项目。系统识别 SMS/BMS、折价区/溢价区、OTE、FVG 和 Order Block，仅发送观察提醒，不执行交易。

## 当前状态

- 设计规格：已批准
- 实施计划：编写中
- 运行环境：尚未部署
- 自动交易：不在项目范围内

## 监测标的

- `BINANCE:BTCUSDT`
- `BINANCE:ETHUSDT`
- `BINANCE:LTCUSDT`
- `BINANCE:DOTUSDT`
- `BINANCE:BNBUSDT`
- `BINANCE:DASHUSDT`

只接受 TradingView `1h` 周期。

## 架构

TradingView Pine Script 负责结构和区域判断；Azure Functions 负责事件验证、持久化、每个独立区域最多两次提醒、事件合并及 Microsoft Graph Outlook 邮件发送。常规监测不调用大模型。

## 文档

- [设计规格](docs/superpowers/specs/2026-06-22-monitoring-ict-orderflow-design.md)
- [实施计划](docs/superpowers/plans/2026-06-22-monitoring-ict-orderflow-implementation.md)

## 安全边界

- 不连接交易账户，不下单。
- 不把 Outlook OAuth、Azure Function Key 或其他凭据提交到 Git。
- `.env`、`local.settings.json`、令牌和密钥文件均由 `.gitignore` 排除。

## 开发日志

### 2026-06-22

- 确认六个 Binance 监测标的和 1h 周期。
- 确认 SMS/BMS、推动腿、折溢价区、OTE、FVG、Order Block 规则。
- 确认区域盘中首次触及立即提醒，SMS/BMS 只在 1h 收盘确认。
- 确认每个独立区域最多提醒两次，第三次进入后机会耗尽。
- 确认采用 TradingView、Azure Functions、Azure Table Storage 和 Microsoft Graph 的混合架构。
- 完成并批准设计规格。
- 创建项目 README 和持续开发日志。
- 完成分阶段 TDD 实施计划。

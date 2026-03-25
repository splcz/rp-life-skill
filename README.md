# rp-life-skill

生活服务商户 Skill 合集 — 让 Cursor Agent 帮你点咖啡、下单、支付。

## 包含的 Skill

| Skill | 目录 | 说明 |
|-------|------|------|
| `luckin-coffee` | `luckin-coffee/` | 瑞幸咖啡下单（选品 → 创单 → 选支付方式 → 支付 → 收据） |

## 安装

将 Skill 复制到 `~/.cursor/skills/`：

```bash
git clone https://github.com/splcz/rp-life-skill.git
cd rp-life-skill

mkdir -p ~/.cursor/skills/luckin-coffee
cp luckin-coffee/SKILL.md ~/.cursor/skills/luckin-coffee/SKILL.md
```

重启 Cursor（或 Reload Window）生效。

## 使用

在 Cursor Agent 对话中说：

```
来杯瑞幸
```

Agent 会弹出饮品选择 → 创建订单 → 选择支付方式 → 完成支付 → 展示电子收据。

## 前置依赖

本 Skill 独立完成创单和支付 API 调用，**不依赖任何 MCP**。但签名环节需要额外安装支付 Skill 和 MCP：

### 1. 安装 MCP Server

在 `~/.cursor/mcp.json` 中添加（如果文件不存在则新建）：

```json
{
  "mcpServers": {
    "redotpay-usdc": {
      "command": "npx",
      "args": ["-y", "rp-wallet-mcp"]
    }
  }
}
```

> npm 包地址：[rp-wallet-mcp](https://www.npmjs.com/package/rp-wallet-mcp)

### 2. 安装支付 Skill

```bash
git clone https://github.com/splcz/rp-wallet-skill.git
cd rp-wallet-skill

mkdir -p ~/.cursor/skills/redotpay-web3-payment
cp web3-payment/SKILL.md ~/.cursor/skills/redotpay-web3-payment/SKILL.md

mkdir -p ~/.cursor/skills/redotpay-balance-payment
cp balance-payment/SKILL.md ~/.cursor/skills/redotpay-balance-payment/SKILL.md
```

安装完成后重启 Cursor。

> 不安装支付 Skill 时，商户 Skill 可以走到选择支付方式那一步，但无法完成签名。

## License

MIT

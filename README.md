# rp-life-skill

生活服务商户 Skill 合集 — 让 AI Agent 帮你点咖啡、下单、支付。

## 支持的 Agent

| Agent | 状态 |
|-------|------|
| Cursor | ✅ 已验证 |
| OpenClaw | ✅ 已适配 |
| Claude Code | ✅ 兼容 |

## 包含的 Skill

| Skill | 目录 | 说明 |
|-------|------|------|
| `luckin-coffee` | `luckin-coffee/` | 瑞幸咖啡下单（选品 → 创单 → 选支付方式 → 支付 → 收据） |

## 安装

### 方式一：一键安装（推荐）

```bash
# Cursor
npx skills add splcz/rp-life-skill -g -a cursor -y

# OpenClaw
npx skills add splcz/rp-life-skill -g -a openclaw -y

# 所有已安装的 Agent
npx skills add splcz/rp-life-skill -g --all -y
```

### 方式二：手动安装

```bash
git clone https://github.com/splcz/rp-life-skill.git

# Cursor
cp -r rp-life-skill/luckin-coffee ~/.cursor/skills/luckin-coffee

# OpenClaw
cp -r rp-life-skill/luckin-coffee ~/.openclaw/skills/luckin-coffee
```

安装后重启 Agent 生效。

## 使用

在 Agent 对话中说：

```
来杯瑞幸
```

Agent 会弹出饮品选择 → 创建订单 → 选择支付方式 → 完成支付 → 展示电子收据。

## 前置依赖

本 Skill 独立完成创单和支付 API 调用，**不依赖任何 MCP**。但签名环节需要额外安装支付 Skill 和 MCP：

### 1. 安装 MCP Server

将 `redotpay-usdc` 添加到你的 Agent MCP 配置中：

<details>
<summary>Cursor</summary>

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
</details>

<details>
<summary>OpenClaw</summary>

在 `~/.openclaw/openclaw.json` 中添加：

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
</details>

> npm 包地址：[rp-wallet-mcp](https://www.npmjs.com/package/rp-wallet-mcp)

### 2. 安装支付 Skill

```bash
# Cursor
npx skills add splcz/rp-wallet-skill -g -a cursor -y

# OpenClaw
npx skills add splcz/rp-wallet-skill -g -a openclaw -y
```

或手动安装：

```bash
git clone https://github.com/splcz/rp-wallet-skill.git

# Cursor
cp -r rp-wallet-skill/web3-payment ~/.cursor/skills/redotpay-web3-payment
cp -r rp-wallet-skill/balance-payment ~/.cursor/skills/redotpay-balance-payment

# OpenClaw
cp -r rp-wallet-skill/web3-payment ~/.openclaw/skills/redotpay-web3-payment
cp -r rp-wallet-skill/balance-payment ~/.openclaw/skills/redotpay-balance-payment
```

安装完成后重启 Agent。

> 不安装支付 Skill 时，商户 Skill 可以走到选择支付方式那一步，但无法完成签名。

## License

MIT

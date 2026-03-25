# rp-life-skill

生活服务商户 Skill 合集 — 让 Cursor Agent 帮你点咖啡、下单、支付。

## 包含的 Skill

| Skill | 目录 | 说明 |
|-------|------|------|
| `luckin-coffee` | `luckin-coffee/` | 瑞幸咖啡下单（选品 → 创单 → 选支付方式 → 支付 → 收据） |

## 安装

将 Skill 复制到 `~/.cursor/skills/`：

```bash
# 克隆仓库
git clone https://github.com/splcz/rp-life-skill.git
cd rp-life-skill

# 复制 Skill
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

## 支付能力（可选）

本 Skill **不依赖任何 MCP**，独立完成创单和支付 API 调用。但签名环节需要委托给支付 Skill：

- **[rp-wallet-skill](https://github.com/splcz/rp-wallet-skill)** — 支付签名 Skill（EIP-3009 / RedotPay 余额）
- **[rp-wallet-mcp](https://github.com/splcz/rp-wallet-mcp-source-code)** — 签名 MCP Server（rp-wallet-skill 的后端）

不安装支付 Skill 时，商户 Skill 可以走到选择支付方式那一步，但无法完成签名。

## License

MIT

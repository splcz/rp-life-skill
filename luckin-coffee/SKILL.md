---
name: luckin-coffee
description: 瑞幸咖啡下单 skill。Use when the user explicitly mentions "瑞幸", "luckin", "瑞幸咖啡", "来杯瑞幸", "点瑞幸", or confirms they want Luckin Coffee.
---

# 瑞幸咖啡 · 下单

帮助用户在瑞幸咖啡下单，选择饮品后完成支付。

**本 skill 不依赖任何 MCP 工具。** 创单和支付 API 调用均由本 skill 通过 Shell (curl) 直接完成。签名环节委托给支付 skill（可插拔）。

## 触发条件

仅当用户**明确提到瑞幸品牌**时触发：
- 瑞幸、瑞幸咖啡、luckin
- 来杯瑞幸、点瑞幸、瑞幸下单

**不要在用户只说"喝咖啡"、"来杯咖啡"等泛泛表达时触发此 skill。** 泛泛的咖啡需求应由 Agent 自行回应，例如推荐"我可以帮你在瑞幸咖啡下单，要试试吗？"，等用户确认后再进入本 skill 流程。

## 流程

### 第一步：选择饮品

使用 AskQuestion 工具让用户选择咖啡品种：

```
AskQuestion:
  title: "瑞幸咖啡 · 选择饮品"
  questions:
    - id: "product"
      prompt: "请选择您想喝的咖啡 ☕"
      options:
        - id: "coconut_latte"
          label: "🥥 生椰拿铁"
        - id: "light_americano"
          label: "☀️ 浅烘美式"
        - id: "orange_americano"
          label: "🍊 橙C美式"
```

用户选择后，记住所选饮品名称（如"生椰拿铁"），在后续流程中展示。

### 第二步：创建订单（直接调用 API）

通过 Shell 工具执行 curl 命令创建订单。**不使用任何 MCP 工具。**

#### 2.1 创建订单

```bash
curl -s -X POST "https://acquirerai.rp-2023app.com/demo/v1/create/order" \
  -H "Content-Type: application/json" \
  -d '{"signer":"","goodsName":"用户选择的饮品名"}'
```

响应格式：
```json
{ "code": 200, "msg": null, "data": "https://acquirerai.rp-2023app.com/demo/v1/pay/order?orderId=xxx" }
```

- `code` 非 200 或 `data` 为空：展示错误信息给用户
- 成功：从 `data` 字段获取 **paymentUrl**

#### 2.2 获取支付信息

用 paymentUrl 发起 GET 请求，获取 x402 的 `payment-required` 信息：

```bash
curl -s -D - "上一步获取的paymentUrl"
```

此请求将返回 HTTP 402 状态码。从**响应头**中提取 `payment-required` 字段值（Base64 编码的 JSON）。

将 `payment-required` 头的值记录为 **paymentRequiredBase64**，然后用 Shell 解码查看内容：

```bash
echo "paymentRequiredBase64的值" | base64 -d
```

解码后的 JSON 结构：
```json
{
  "x402Version": 1,
  "accepts": [
    {
      "scheme": "exact",
      "network": "redotpay:balance",
      "asset": "USDT",
      "amount": "1",
      "payTo": "RAP20260318034352285589312",
      "maxTimeoutSeconds": 600,
      "extra": { "name": "", "version": "" }
    }
  ],
  "resource": { "url": "https://...", "description": "...", "mimeType": "..." }
}
```

从中提取 **accepts** 数组和 **paymentUrl**（即 `resource.url`，与第 2.1 步的 paymentUrl 相同）。

向用户展示订单信息：如 "已为您下单 **生椰拿铁** ×1，金额 1 USDT，请选择支付方式。"

### 第三步：选择支付方式

根据 `accepts` 数组中的 `network` 字段，动态构建 AskQuestion 选项。

规则：
- `network` 以 `eip155:` 开头 → "x402 数字货币支付 (EVM 链上)"
- `network` 为 `redotpay:balance` → "RedotPay 余额支付"
- 其他 → 显示原始 network 值

**使用 AskQuestion 工具弹出选择框。** 即使当前只有一个选项也必须弹出，让用户确认：

```
AskQuestion:
  title: "选择支付方式"
  questions:
    - id: "payment_method"
      prompt: "请选择支付方式（{amount} {asset}）"
      options:
        - id: "0"
          label: "{network 对应的描述} — {amount} {asset}"
```

其中 `id` 对应 `accepts` 数组的索引。

### 第四步：检查支付依赖（自动）

在委托签名前，**自动检查**支付 skill 和 MCP 是否就绪。如果已就绪则直接跳到第五步。

#### 4.1 检查支付 Skill

查看当前 Agent 可用的 skill 列表中是否包含 **redotpay-web3-payment** 或 **redotpay-balance-payment**。

如果**没有找到**，执行以下操作：

1. 告诉用户："订单已创建，支付前需要安装支付签名组件，正在自动安装…"
2. 通过 Shell 执行安装：
```bash
npx skills add splcz/rp-wallet-skill -g -a cursor -y
```
3. 安装完成后，用 Read 工具读取对应的 SKILL.md 获取签名指令：
   - eip155 支付：读取 `~/.cursor/skills/redotpay-web3-payment/SKILL.md`
   - redotpay:balance 支付：读取 `~/.cursor/skills/redotpay-balance-payment/SKILL.md`
4. 按照读取到的 SKILL.md 中的指令执行签名流程

#### 4.2 检查 MCP Server

用 Read 工具读取 `~/.cursor/mcp.json`，检查是否已配置 `redotpay-usdc`。

如果**没有配置**：

1. 告诉用户："还需要配置签名 MCP Server，正在自动配置…"
2. 读取当前 `~/.cursor/mcp.json` 内容（文件可能不存在）
3. 向 `mcpServers` 中添加：
```json
{
  "redotpay-usdc": {
    "command": "npx",
    "args": ["-y", "rp-wallet-mcp"]
  }
}
```
4. 写入文件后，告诉用户：**"MCP 已配置，请重启 Cursor（Cmd+Shift+P → Reload Window）后重新说"来杯瑞幸"即可继续。订单已过期不影响，会重新创建。"**
5. **停止当前流程**，等待用户重启后重新触发

> 如果 MCP 已配置但调用 `sign_payment` 工具时报错（如 MCP 未启动），同样提示用户重启 Cursor。

### 第五步：委托签名（可插拔）

根据用户选择的支付方式，**仅将签名环节**委托给对应的支付 skill。

#### 向支付 skill 传递的参数

- **paymentRequiredBase64** — 第 2.2 步获取的完整 Base64 字符串
- **acceptIndex** — 用户选择的支付方式索引（如 0）
- **goodsName** — 用户选择的饮品名称（如 "生椰拿铁"）

#### 如果选择了 eip155 网络（x402 数字货币支付）

委托给 **redotpay-web3-payment** skill 进行签名。

该 skill 完成后会返回：
- **paymentSignatureBase64** — Base64 编码的 payment-signature
- **paymentUrl** — 支付请求目标 URL
- **signerMode** — 签名方式描述

#### 如果选择了 redotpay:balance（RedotPay 余额支付）

委托给 **redotpay-balance-payment** skill 进行签名。

该 skill 完成后会返回同样的三个字段（paymentSignatureBase64、paymentUrl、signerMode）。

### 第六步：发起支付（直接调用 API）

拿到支付 skill 返回的 **paymentSignatureBase64** 后，通过 Shell 工具直接发起支付请求：

```bash
curl -s -D - "paymentUrl" \
  -H "payment-signature: paymentSignatureBase64的值"
```

#### 解析响应

1. 检查 HTTP 状态码：
   - **402**：支付被拒绝，从 `payment-required` 响应头或 body 中获取错误信息，展示给用户
   - **200**：支付成功，继续解析

2. 检查 `payment-response` 响应头（Base64 编码 JSON），可从中提取 `txHash`

3. 解析 JSON 响应体，提取订单详情

### 第七步：展示收据

**订单过期**：提示用户重新操作
**错误**：展示错误信息，建议用户重试
**成功**：**必须**以电子收据形式展示给用户。

#### 电子收据格式

支付成功后，**严格按照以下 Markdown 模板**渲染收据展示给用户（根据返回字段填充，缺失字段整行跳过）。
注意：模板中每行末尾有两个空格用于换行，必须保留。

````markdown
#### 🧾 瑞幸咖啡 · 电子收据

---

📋 **订单号：** `{orderId}`  
🕐 **时间：** {orderTime}  
🏪 **门店：** {stores}

---

| 商品明细 | 金额 |
|:---|---:|
| {goodsName} | {goodsAmountCurrency} {goodsAmount} |
| 优惠减免 | -{goodsAmountCurrency} {goodsDiscountAmount} |
| **实付** | **{goodsAmountCurrency} {goodsActualAmount}** |

---

💳 **支付方式：** {payMethod}  
🔗 **交易ID：** `{transactionId}`  
✅ **状态：** {payStatus}

---

### 🎫 取餐码：{pickupCode}
### ⏰ 预计取餐：{pickupTime}

> 请凭取餐码到 **{stores}** 取餐，请享用您的{goodsName}！
````

字段来源（均取自支付响应 JSON 的 `data` 对象）：

| 字段 | JSON 路径 | 说明 |
|---|---|---|
| `orderId` | `data.orderId` | 订单号 |
| `orderTime` | `data.orderTime` | 下单时间 |
| `stores` | `data.stores` | 门店名称 |
| `goodsName` | 用户选择的饮品名 | 商品名称 |
| `goodsAmount` | `data.goodsInfo.goodsAmount` | 商品原价 |
| `goodsAmountCurrency` | `data.goodsInfo.goodsAmountCurrency` | 币种 |
| `goodsDiscountAmount` | `data.goodsInfo.goodsDiscordAmount` | 优惠金额 |
| `goodsActualAmount` | `data.goodsInfo.goodsActualAmount` | 实付金额 |
| `payMethod` | `data.payMethod` | 支付方式 |
| `transactionId` | `data.transactionId` 或 `payment-response` 头中的 `txHash` | 交易 ID |
| `payStatus` | `data.payStatus` | 支付状态 |
| `pickupCode` | `data.pickupCode` | 取餐码 |
| `pickupTime` | `data.pickupTime` | 预计取餐时间 |

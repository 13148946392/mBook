# MCP 集成说明（麦当劳中国官方 MCP Server）

> 本文说明「麦麦图鉴 McDex」项目实际接入的麦当劳中国官方 MCP（`mcd-mcp`）：
> 用到了哪些 Server / 工具、数据怎么从工具落到前端、调用流程长什么样，以及它带来的业务价值。
> 配套数据快照见 `data/mcd-mcp-tools.json`（35 个工具的原始清单）。

---

## 一、MCP Server 配置

| 项 | 值 |
|---|---|
| Server 名称 | `mcd-mcp`（麦当劳中国官方 MCP） |
| 端点 | `https://mcp.mcd.cn`（注意：`/mcp` 路径是 404，必须用根路径） |
| 传输协议 | Streamable HTTP（`type: streamable-http`） |
| 认证 | `Authorization: Bearer <token>`（token 写入本地 MCP 客户端配置，**不入库、不进前端**） |
| 限流 | 600 请求 / 分钟 / token |
| 工具总数 | 35 个 |
| 本项目使用 | **仅 3 个只读数据查询工具**（其余 32 个写接口/业务接口均未接入） |
| 数据快照时间 | 2026-10-10 |

客户端配置位置：`~/.workbuddy/mcp.json`，`type` 必须写成 `streamable-http`（`http` 不生效）；改完需完全退出重开客户端。

---

## 二、工具全景与「实际使用」聚焦

`mcd-mcp` 共暴露 35 个工具，按业务域可分成 9 组。**本项目是只读图鉴应用，只接入了第 1 组的 3 个数据查询工具**，其余全部有意不接入（原因见第五节）。

| 业务域 | 工具 | 本项目是否使用 |
|---|---|---|
| **① 数据查询（只读）** | `list-nutrition-foods`、`campaign-calendar`、`query-meals`、`now-time-info` | ✅ 用了前 3 个；`now-time-info` 未接入（活动月份已快照固化） |
| ② 价格/下单 | `calculate-price`、`create-order`、`query-meal-detail`、`query-meal-assistance` | ❌ |
| ③ 订单查询/取消 | `query-order`、`order-list`、`cancel-order`、`query-survey-coupon` | ❌ |
| ④ 门店/配送 | `query-nearby-stores`、`delivery-query-stores`、`delivery-create-address`、`delivery-query-addresses` | ❌ |
| ⑤ 优惠券 | `available-coupons`、`query-store-coupons`、`query-my-coupons`、`auto-bind-coupons` | ❌ |
| ⑥ 积分抽奖 | `draw-lottery`、`query-lottery-info`、`query-my-prizes` | ❌ |
| ⑦ 麦麦商城 | `mall-product-detail`、`mall-order-list`、`mall-order-detail`、`mall-create-order`、`mall-points-products` | ❌ |
| ⑧ 派对活动 | `party-order-create`、`query-party-store`、`query-party-city`、`query-party-store-date`、`query-party-store-session` | ❌ |
| ⑨ 账户 | `query-my-account` | ❌ |

---

## 三、实际使用的 3 个工具详解

### 1. `list-nutrition-foods` — 营养成分数据
- **用途**：获取麦当劳常见餐品的营养成分（能量、蛋白质、脂肪、碳水、钠、钙等）。当用户咨询热量/营养、搭配套餐时有用。
- **落地变量**：`MCD_NUTRI_RAW`（模板字符串，每行一条，`|` 分隔 8 字段：名称\|能量描述\|kcal\|蛋白质\|脂肪\|碳水\|钠\|钙）。
- **前端消费**：`index.html` 解析为 `NUTRI` 数组，驱动图鉴卡片、三维筛选、五排序、详情弹层。当前 **158 条**（去重后）。

### 2. `campaign-calendar` — 当月营销活动日历
- **用途**：查询麦当劳中国当月营销活动（进行中 / 往期 / 未来），含活动名、描述、标签（新品 / 限定 / 限时）、订阅状态。
- **落地变量**：`MCD_CAMPAIGNS`（数组，每项 `[日期, 活动名, 描述, 标签]`）、`MCD_CAMP_URL`（活动详情链接映射）。
- **前端消费**：活动页展示当月 12 条活动；标签为「新品 / 限定」的活动会自动进入图鉴的「限定」卡池（`rar: LIM`）。
- **关于 `now-time-info`**：该工具能返回服务器当前时间，理论上可用于动态确定「当月」基准。本项目未在取数链路接入它——活动数据在 2026-10-10 快照时已锚定当月，活动页展示的是快照月份，后续刷新时再重新抓取即可。

### 3. `query-meals` — 菜单与商品图
- **用途**：餐品列表，支持到店自提 / 外送两种场景；返回餐品名、价格、图片等信息。
- **落地变量**：
  - `MCD_PRICES`（名称 → 价格，**单位元**，接口原始返回是**分**，已 ÷100），当前 **121 个有效价格键，匹配营养表 106 条（67%）**；
  - `MCD_IMAGES`（名称 → 商品图 URL），**45 张**。
- **前端消费**：图鉴卡片的参考价格、商品缩略图；购物车实时合计金额。

> **为什么没有「第 4 个」**：数据说明文档曾写「用了 4 个只读工具」，是把 `now-time-info` 也算了进去；但 `index.html` 与 `data/mcd-data.js` 中实际取数链路仅引用上述 3 个（`now-time-info` 只存在于工具快照 JSON，未被代码调用）。以本文为准。

---

## 四、调用流程（实际工作流）

本项目采用**「构建期取数、运行时静态快照」**的模式：MCP 在本地 MCP 客户端被调用，把数据整理成 JS 常量后内联进 `index.html`，运行时**不再请求 MCP**（也避免了把 token 暴露到前端）。

### 数据流向

```
┌─────────────────┐   调用 3 个只读工具   ┌──────────────────┐
│ WorkBuddy 本地   │ ───────────────────> │  mcd-mcp         │
│ MCP 客户端       │   Bearer <token>     │  (mcp.mcd.cn)    │
│ (~/.workbuddy/   │                      │  35 工具         │
│   mcp.json)      │ <─────────────────── │                  │
└─────────────────┘   返回 JSON 结构      └──────────────────┘
        │
        │  整理成字段格式
        ▼
┌─────────────────┐   覆盖内联块          ┌──────────────────┐
│ data/mcd-data.js│ ───────────────────> │ index.html       │
│ (数据层副本)     │   <script>数据层</script> │  (单文件应用)    │
└─────────────────┘                      └──────────────────┘
                                               │
                                               │ 双击打开 / 托管
                                               ▼
                                         浏览器渲染图鉴/抽奖/活动
                                         （运行时零网络请求）
```

### 典型调用示例（构建期，本地 MCP 客户端）

以 `list-nutrition-foods` 为例，Streamable HTTP 的实际调用形态（token 用占位符，切勿写入任何仓库或前端）：

```
POST https://mcp.mcd.cn
Content-Type: application/json
Authorization: Bearer <MCD_MCP_TOKEN>

{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": { "name": "list-nutrition-foods", "arguments": {} }
}
```

返回（节选）经整理后写入 `MCD_NUTRI_RAW`：

```
猪柳麦满分|猪柳加蛋，早餐的黄金搭档|387|23|21|25|846|243
```

三个工具的返回分别整理为 `MCD_NUTRI_RAW` / `MCD_CAMPAIGNS`+`MCD_CAMP_URL` / `MCD_PRICES`+`MCD_IMAGES`，再以 `data/mcd-data.js` 为中间副本、覆盖进 `index.html` 的 `<script>数据层</script>` 块。

### 价格匹配流程（名称口径不一致问题的解法）

营养表的商品名与菜单接口的商品名写法不同（前者是营养表口径、后者是菜单口径），无法直接按名精确匹配。`index.html` 用 `normName()` 归一化 + 双向子串包含兜底：

```js
function normName(s){ /* 去括号/书名号/引号/规格词/空格 */ }
function priceOf(n){
  var k = normName(n);
  if (PMAP[k] != null) return PMAP[k];          // 1) 归一化后精确命中
  var best = null, bestLen = 1e9;
  for (var p in PMAP)                            // 2) 双向子串包含，取最短（单品优先于套餐）
    if (k.indexOf(p) >= 0 || p.indexOf(k) >= 0)
      if (p.length < bestLen) { bestLen = p.length; best = PMAP[p]; }
  return best;
}
```

匹配率 **106 / 158 ≈ 67%**；未匹配的是早餐与季节限定（接口无报价），页面标「未报价」。子串兜底会引入少量误配（套餐价套到单品），故图鉴价格定位为**参考值**而非准确值。

### 刷新流程（手动，推荐）

营养表变化很慢，一两个月刷一次即可：

1. 在本地 MCP 客户端调用 `list-nutrition-foods` / `campaign-calendar` / `query-meals`，拿最新返回；
2. 整理成上述字段格式，替换 `data/mcd-data.js`；
3. 用脚本把 `data/mcd-data.js`（去文件头注释）覆盖回 `index.html` 的 `<script>数据层</script>` 块。

> **不推荐**：页面加载时直接请求 MCP——那需要把 token 写进前端，任何人都能在浏览器里看到并盗用，明确不要这样做。

---

## 五、为什么不用其余 32 个写/业务接口

本项目定位是**只读、静态、零后端**的图鉴 H5，刻意不接入任何会产生副作用的接口：

- **下单 / 支付类**（`create-order`、`calculate-price`、`draw-lottery`、`mall-create-order`、`party-order-create` 等）：会真实扣积分、生成订单、产生资金流。图鉴不需要，且前端无法安全承载这类操作（token 不能进浏览器）。
- **用户资产类**（`query-my-account`、`query-my-coupons`、`query-my-prizes`、`auto-bind-coupons` 等）：依赖登录态与会员身份，超出图鉴范围。
- **门店 / 配送 / 订单查询类**：需要用户位置、地址、订单号等隐私与上下文，与图鉴无关。

保持「只取公开数据、不碰写接口」也意味着**合规与零责任风险**：应用不代客下单、不收集隐私、不触碰资金。

---

## 六、业务价值

1. **数据权威性**：直接来自麦当劳中国官方 MCP，相比爬虫或第三方抓取，数据口径一致、可信度高、不易失效。
2. **合规与安全**：仅用只读接口，token 只存本地 MCP 客户端配置、绝不进仓库/前端；不收集用户隐私、不代客下单，规避资金与隐私风险。
3. **实时可刷新**：活动日历、价格、营养数据都可通过重新调用 3 个工具刷新，应对菜单与营销节奏变化（当前为 2026-10-10 快照）。
4. **开发效率**：35 个工具已在 Server 端封装好鉴权、限流、数据聚合，项目按需取数，不必自己处理对接细节。
5. **零后端架构**：构建期取数、运行时静态快照，使整个应用成为「一个 HTML 双击即跑」的单文件——便于托管到 GitHub Pages、微信转发、离线演示。
6. **清晰的数据血缘**：`data/mcd-mcp-tools.json` 保留了工具原始快照，任何一条图鉴数据都能追溯到具体哪个 MCP 工具、哪次抓取。

---

## 七、安全与合规要点（务必遵守）

- token **不写入仓库、不写入前端**，只存在于本地 MCP 客户端的 `~/.workbuddy/mcp.json`。
- 若在任何文件里搜到形如 `Bearer xxx` 的真实 token 字符串，那是误提交，请立刻作废该 token 并轮换。
- MCP 客户端配置 `type` 必须写 `streamable-http`；改配置后需完全退出重开客户端才能生效。
- 限流 600 req/min/token，批量刷新数据时注意节流，避免触发限流。

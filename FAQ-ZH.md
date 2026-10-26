---
sourceType: business_rule
title: Extramile OpenAPI 常见问题 (V1.9)
---

# Extramile OpenAPI 常见问题

> 适用版本：OpenAPI V1.9，存续、发行人与挂钩标的查询见 2026-10-08 修订（V1.9.3）  
> 更新日期：2026-10-22  
> 本文按调用顺序排列。示例单号、ISIN 和价格仅用于说明。报障时提供环境、UTC+8 时间、`openuri` 和响应头 `request_id`，不要提供 `appsecret` 或完整签名源字符串。

推荐调用顺序：

0. （按需）调用 `/quotation/allIssuer` 查询启用中的发行商列表，调用 `/quotation/pageUnderlying` 分页查询可用挂钩标的。
1. `/quotation/create` 发起询价，保存返回的 `QuoteOrderID` 及原始 `QuoteGroupList` 下标。
2. 接收 `/quotation/update/notify`，再调用 `/quotation/query` 取全量报价快照。
3. `/trade/create` 用 `quotationId`（传 `QuoteOrderID`）+ `quotationCombIdx` + `issuer`（传 `issuerId`）下单。
4. 接收 `/Trade/notify`，再调用 `/trade/search/query` 和 `/trade/search/detail`。
5. 成交且最终条款单入账后，接收 `/lifecycle/update/notify`，再调用 `/lifecycle/eventQuery`。

## 问题列表

| 分类 | 问题 |
| --- | --- |
| 1. 接入、环境与安全 | [1.1 地址](#11-调用-api-时使用哪个地址和路径) · [1.2 上线前准备](#12-接入前需要提供和获取哪些信息) · [1.3 签名](#13-apisign-如何生成) · [1.4 签名放在哪里](#14-凭证和签名应放在请求头还是请求体) · [1.5 时钟与 nonce](#15-timestamp-和-nonce-有什么约束) · [1.6 密钥轮换](#16-appsecret-可以轮换吗) · [1.7 限流](#17-有哪些限流询价额度如何计算) · [1.8 白名单](#18-未登记的-ip-为什么也能收到-http-200) |
| 2. 发起询价 | [2.1 大小写](#21-json-字段名是否区分大小写) · [2.2 标的代码](#22-港股和美股的-underlyings-如何传值) · [2.3 SolveFor](#23-solvefor-字段应该如何填写) · [2.4 百分比](#24-百分比字段的单位是否统一) · [2.5 跨组合校验](#25-同一询价请求为什么不能混用某些字段值) · [2.6 组合数量](#26-一次询价最多可以提交多少个组合) · [2.7 是否注册回调](#27-发起询价后是否还要注册报价回调) · [2.8 100058](#28-创建询价返回-100058-是签名错误吗) |
| 3. 报价回调与查询报价结果 | [3.1 回调内容](#31-报价回调是否包含完整报价) · [3.2 多次回调](#32-同一个询价是否会收到多次回调) · [3.3 询价状态](#33-quoteorderstatus-状态-3-是否表示全部回价) · [3.4 报价失效](#34-报价何时失效) · [3.5 组合与下单对应](#35-多个询价组合如何与报价及下单对应) · [3.6 撤价下标](#36-cancelledrecordindex-表示哪个下标) · [3.7 撤价通知](#37-发行商撤回报价时如何通知) · [3.8 QuotePriceStatus](#38-quotepricestatus-有哪些值) · [3.9 minSize](#39-minsize-和-maxsize-表示总可售额度吗) · [3.10 最优报价](#310-api-是否返回最优报价标识) · [3.11 发行商 ID](#311-如何获取发行商-id) · [3.12 QuoteTime](#312-quotetime-使用什么时区) · [3.13 查询窗口](#313-报价结果可以查询多久) |
| 4. 创建交易订单 | [4.1 易混字段](#41-使用报价下单时最容易混淆的字段) · [4.2 限价标的](#42-为什么询价代码与限价下单字段格式不同) · [4.3 本方订单号](#43-institutionalorderid-有什么用途) · [4.4 下单内部错误](#44-询价正常但创建交易返回内部错误) · [4.5 创建成功但没有回调](#45-交易已创建为什么没有收到状态回调) |
| 5. 交易状态与下单超时 | [5.1 终态](#51-哪些交易状态是终态) · [5.2 接受与拒绝](#52-订单何时接受何时拒绝) · [5.3 状态 3](#53-状态-3-是否表示条款单已发出) · [5.4 状态 9](#54-状态-9-是否表示已结算) · [5.5 状态 10–16](#55-状态-1016-分别是什么意思) · [5.6 超时](#56-下单超时后会自动重下或作废吗) |
| 6. 交易结果与订单详情 | [6.1 回调摘要](#61-交易回调是否包含完整结果) · [6.2 字段位置](#62-isin日期参考价和-pdf-分别在哪个字段) · [6.3 处理顺序](#63-收到交易回调后按什么顺序处理) · [6.4 样例条款单](#64-如何获取-sample-termsheet) |
| 7. 存续与结算 | [7.1 何时开始](#71-存续事件从什么时候开始) · [7.2 如何获取](#72-存续和结算事件如何获取) |
| 8. 回调通知 | [8.1 回调地址](#81-回调地址如何配置请求体还要传-webhook-吗) · [8.2 如何区分](#82-如何区分询价交易和存续回调) · [8.3 应答与重试](#83-收到回调后应如何响应) · [8.4 幂等](#84-回调会不会重复) · [8.5 回调签名](#85-回调如何验签) |
| 9. 错误处理 | [9.1 两层错误](#91-为什么-http-200-仍可能表示业务失败) · [9.2 网关错误码](#92-网关错误码是什么意思) · [9.3 定位字段](#93-参数错误时如何定位具体字段) · [9.4 1009 与 GET](#94-用-get-调用-post-接口为什么返回的不是-1009) |

---

## 1. 接入、环境与安全

### 1.1 调用 API 时使用哪个地址和路径？

所有业务接口都通过 HTTPS `POST` 调用统一入口：

```text
/openapi/request
```

具体业务接口由请求头 `openuri` 指定，例如 `openuri: /quotation/create`。本文档中的业务接口都使用 `POST`。平台目前只有 `POST` 和 `GET` 两类接口；其它 HTTP 方法才会返回 `1009`，见 [9.4](#94-用-get-调用-post-接口为什么返回的不是-1009)。

UAT 网关地址为 `https://uat-openapi.goldhorse.ai/openapi/request`。生产环境同样走公网 HTTPS。生产地址和生产凭证在上线前通过约定的安全渠道提供。当前没有单独的符合性考试或证书流程。

### 1.2 接入前需要提供和获取哪些信息？

接入方需要：

- 获取对应环境的 `appid`、`appsecret`、`insId` 和 `loginName`；
- 向 Extramile 登记调用方固定出口 IP；
- 提供 HTTPS 回调基址；
- 将 Extramile 的回调出口 IP 加入接入方白名单。

回调出口 IP 为：

- UAT：`8.210.49.108`
- 生产：`18.167.242.181`

上线前请再次确认这些地址和白名单。

### 1.3 `apisign` 如何生成？

1. 按固定顺序拼接 UTF-8 字符串：

```text
appid={appid}&appsecret={appsecret}&nonce={nonce}&timestamp={timestamp}
```

2. 使用 `appsecret` 的 UTF-8 字节作为 HMAC 密钥。`apiverify` 为 `HmacMd5` 时计算 HMAC-MD5；为 `HmacSha256` 时计算 HMAC-SHA256。
3. 将结果编码为小写十六进制，放入请求头 `apisign`。
4. 请求头 `apiverify` 传 `HmacMd5` 或 `HmacSha256`。网关也接受 `HmacSHA256`，并按 SHA256 验签。

可使用以下向量检查实现。示例密钥仅用于测试：

```text
appid      = 20
appsecret  = testSecret123
nonce      = 550e8400-e29b-41d4-a716-446655440000
timestamp  = 1760400000000

待签名字符串：
appid=20&appsecret=testSecret123&nonce=550e8400-e29b-41d4-a716-446655440000&timestamp=1760400000000

HmacMd5 的 apisign：
a37fd64ce9c22f6b2d5869a6321fd1e7

HmacSha256 的 apisign：
00555de496a1d455ef8407edf12b55f99ed06642aabe2cc0ed166ba1a81e59e8
```

普通 MD5 不能代替 HMAC-MD5 或 HMAC-SHA256。字段顺序、大小写、空格或换行变化都会导致验签失败。

`HmacSHA1` 等其它 `apiverify` 返回 HTTP 400，`errcode` 为 `1002`，`errmsg` 为 `Not supported verify type`。`apiverify` 与 `apisign` 的算法不一致时返回 HTTP 400，`errcode` 为 `1003`，`errmsg` 为 `API signature check failed`。例如声明 `HmacSha256`，签名却用 HMAC-MD5。

回调验签见 [8.5](#85-回调如何验签)。回调当前使用 `HmacMd5`。

### 1.4 凭证和签名应放在请求头还是请求体？

身份和签名放在 HTTP 请求头，JSON 请求体只放业务字段。

`appsecret` 只参与本地计算，不能作为请求头或请求体字段发送。把 `appid`、`appsecret`、`insId`、`loginName` 放进 JSON 体，或者对签名字符串做普通 MD5，都会验签失败。

验签失败时，请求不会进入业务服务。网关返回 HTTP 400，响应头 `errcode` 为 `1003`，`errmsg` 为 `API signature check failed`。这时应重算签名，不要据此修改标的、期限或其他业务字段。

### 1.5 `timestamp` 和 `nonce` 有什么约束？

`timestamp` 是 Unix 毫秒时间戳。网关允许的时钟偏差为前后 120 秒。

同一个 `appid` 的 `nonce` 在 120 秒内不能重复。超出时间窗或重复 `nonce` 时，网关返回 HTTP 403，响应头 `errcode` 为 `-1`，`errmsg` 为 `Forbidden request`。

缺少 `timestamp`、`nonce`、`apisign` 或 `apiverify` 时，返回 HTTP 400，`errcode` 为 `1000`。

### 1.6 `appsecret` 可以轮换吗？

可以按需轮换。接入方生成新密钥后，通过约定的安全渠道发送给 Extramile，并约定生效时间。

轮换 `appsecret` 不会更改 `appid`、`insId` 或 `loginName`。请提前协调切换并完成验证。

### 1.7 有哪些限流？询价额度如何计算？

存在两层限制：

- 通用技术限流：所有 API 合计不超过 10 次/秒；
- 发起询价限制：默认每个接入方 300 次/自然日、30 次/分钟。

询价额度按 `/quotation/create` 的调用次数计算，与一个请求内的 `QuoteGroupList` 数量无关：

- 1 次请求包含 10 个询价组合：消耗 1 次；
- 20 个组合拆成 2 次请求：消耗 2 次；
- `/quotation/query` 不消耗询价额度，但仍受通用限流约束。

超出频率时，网关返回 HTTP 429，`errcode` 为 `1007`。业务层额度用尽时，创建询价可能返回响应体 `code = 10001`，`message` 为 `Exceed the limit of request frequency`。

每日额度按自然日计算。如实现依赖具体重置时区，请联系 Extramile 确认。

### 1.8 未登记的 IP 为什么也能收到 HTTP 200？

UAT 和生产都通过公网 HTTPS 访问，接入方仍须登记固定出口 IP。

应用网关按 `appid`、签名、`insId` 和 `loginName` 识别调用方。从尚未登记的网络收到 HTTP 200，不能说明生产环境可以省略白名单。上线前请与 Extramile 核对当前环境实际生效的出口 IP。

---

## 2. 发起询价

`openuri: /quotation/create`

### 2.1 JSON 字段名是否区分大小写？

区分。请求和响应同时存在首字母大写与小写字段，必须使用接口文档中的拼写，例如：

- 请求：`Product`、`QuoteGroupList`、`QuoteOrderID`；
- 报价结果：`Issuer`、`Result`、`QuoteID`、`QuoteTime`、`QuotePriceStatus`；
- 同一结果对象中的其他字段：`issuerId`、`minSize`、`maxSize`、`yieldPercentage`。

不要自行转换成统一的 camelCase、PascalCase 或全小写。错误大小写可能返回 `10002` 或 `10003`。

### 2.2 港股和美股的 `Underlyings` 如何传值？

| 字段 | 传值规则 |
| --- | --- |
| `Type` | 统一传 `"BBG"` |
| `Code` | 传不含市场后缀的 Bloomberg Ticker（港股用不补零的纯数字短码） |
| `Market` | 2 字母市场代码，如 `HK`、`US` |

| 市场及标的 | `Type` | `Code` | `Market` |
| --- | --- | --- | --- |
| 港股腾讯控股 | `BBG` | `700` | `HK` |
| 港股阿里巴巴 | `BBG` | `9988` | `HK` |
| 港股美团 | `BBG` | `3690` | `HK` |
| 美股 Apple | `BBG` | `AAPL` | `US` |
| 美股 Tesla | `BBG` | `TSLA` | `US` |

港股使用不补零、不带市场后缀的 Bloomberg 短代码，例如 `700`，而不是 `00700` 或 `700.HK`。市场通过 `Market` 单独传递。

2026-10-08 修订（V1.9.3）新增了挂钩标的分页查询接口 `openuri: /quotation/pageUnderlying`（传入从 `1` 开始的 `pageIndex` 和 `pageSize`），返回 `datas.underlyingList[]`，包含 `code`、`codeExchange`、`codeBloomberg`、`codeReuters`、`name`、`nameEn`、`market`、`quoteCurrency`。请注意区分不同接口的标的代码格式：

- `/quotation/pageUnderlying` 返回示例中，腾讯的 `code` 为 `"0700"`、`codeExchange` 为 `"0700.HK"`、`codeBloomberg` 为 `"700 HK"`；
- `/quotation/create` 询价时，`Underlyings.Code` 传不含市场后缀与前导零的彭博短码（如 `"700"`），`Market` 传 `"HK"`；
- `/trade/create` 限价下单（`tradeOrderType = 2`）时，`assertId` 传 `code.market` 格式（如 `"00700.HK"`），详见 [4.2](#42-为什么询价代码与限价下单字段格式不同)。

其他市场、指数、货币对或特殊标的，请结合 `/quotation/pageUnderlying` 或向 Extramile 确认代码规则。

该服务不可用时，网关返回 HTTP 400，响应头 `errcode` 为 `1005`，`errmsg` 为 `Service unavailable. Please try again later`，响应体里没有 `underlyingList`。这与参数错误不同。同一时间 `/quotation/allIssuer` 仍可能正常返回。

### 2.3 `SolveFor` 字段应该如何填写？

`SolveFor` 指定需要发行商求解并返回的字段。作为 `SolveFor` 的字段通常不提交固定值；其他标记为条件必填（`C`）的字段仍须按产品规则提供。

例如，FCN 的 `SolveFor` 为 `CouponPercentage` 时，不提交固定的 `CouponPercentage`，但仍需提交符合约束的执行价、敲出水平等字段。

不同产品的求解字段、范围和必填条件见 V1.9 第 3.1 节对应产品表格。

### 2.4 百分比字段的单位是否统一？

不完全统一：

- 大多数询价百分比参数按百分数传递，例如 `CouponPercentage = "5.20"` 表示 5.20%，最多两位小数；
- 报价结果中的 `yieldPercentage` 按小数返回，并且已经年化，例如 `"0.0359"` 表示年化 3.59%。

不要对所有包含 `Percentage` 的字段使用同一换算规则。

### 2.5 同一询价请求为什么不能混用某些字段值？

接口有跨组合校验，例如：

- 同一请求中的 `EffectiveDateOffset` 不能同时出现 `5` 和 `10`；
- ELN 的同一请求中不能混用 `KnockOutType=NA` 与 `ContinuousKO` 或 `DailyKO`；
- `Tenor` 必须可被 `ObservationFrequency` 整除；
- 百分比字段最多两位小数。

违反规则时，用响应中的 `errorType` 和 `fieldPath` 定位组合与字段。

### 2.6 一次询价最多可以提交多少个组合？

一个 `/quotation/create` 请求的 `QuoteGroupList` 最少 1 个、最多 10 个组合。请在本地保存原始组合内容、顺序和返回的 `QuoteOrderID`。

### 2.7 发起询价后，是否还要注册报价回调？

不需要。`/quotation/create` 成功后，发行商返回报价时，Extramile 会向已配置的统一回调地址发送询价更新。

V1.9 不要求在请求体中传 `webhook`，也没有单独的 “quote-pricing-results” 注册步骤。

### 2.8 创建询价返回 `100058` 是签名错误吗？

不是。响应体 `code = 100058` 且 `message = "Invalid parameters"` 表示：当前机构没有可用于该产品的发行合约，询价没有创建。

创建询价捕获业务异常后，会保留原始错误码，并把 `message` 统一写成 `Invalid parameters`。因此要看 `code`，不能只看这句文案。

签名错误不会进入这段业务响应。签名错误是 HTTP 400，响应头 `errcode = 1003`。见 [1.4](#14-凭证和签名应放在请求头还是请求体)。

---

## 3. 报价回调与查询报价结果

查询接口为 `openuri: /quotation/query`。

### 3.1 报价回调是否包含完整报价？

不包含。`/quotation/update/notify` 是变化提示，主要包含：

- `QuoteOrderID`
- `QuoteOrderStatus`
- `NewPrice`
- `PriceCancelled`
- `CancelledIssuerId`
- `CancelledRecordIndex`

收到 `NewPrice=true` 后，调用 `/quotation/query`，用 `QuoteOrderID` 拉取当时的全量快照，并以快照更新本地数据。

### 3.2 同一个询价是否会收到多次回调？需要轮询吗？

会。发行商陆续回价时，同一 `QuoteOrderID` 可能收到多次 `NewPrice=true`。

推荐处理：

1. 收到通知后立即返回 HTTP 200；
2. 异步调用 `/quotation/query`；
3. 按 `QuoteOrderID`、询价组合和 `issuerId` 更新本地快照；
4. 需要时用低频轮询兜底。

重复查询返回当时最新的全量快照，不会再次注册回调，也不消耗询价额度。

### 3.3 如何理解 `QuoteOrderStatus`？状态 3 是否表示所有发行商都已回价？

| 值 | 含义 | 说明 |
| --- | --- | --- |
| 1 | 待处理 | 询价已创建，等待处理 |
| 2 | 处理中 | 询价处理中 |
| 3 | 已处理 | **已有至少一家发行商回复报价**（非全部回价完成） |
| 4 | 已取消 | 询价已取消或超期作废（终态） |
| 5 | 已失效 | 报价已过有效期（终态） |

状态 `3` 仅代表**已有发行商返回报价**，并不代表所有发行商都已回价完毕，状态变为 `3` 之后仍可能继续收到其他发行商的新报价通知。状态 `4` 和 `5` 可视为终态。

**是否有字段可判断“所有发行商都已回复完毕”？**

当前接口**没有**任何字段或状态能判断“所有发行商均已回复达到终态”，平台也没有“发行商回价超时”状态：

- 各发行商通过不同渠道异步回价，且部分发行商可能不会回复；
- 在询价单关闭（`4`）或失效（`5`）之前，平台无法预知某家尚未回价的发行商后续是否还会报价；
- 唯一确定的是：每当有发行商返回新报价，平台都会推送 `/quotation/update/notify`（`NewPrice = true`），接入方收到通知后调用 `/quotation/query` 即可获取最新全量报价。

**自动询价并选价上架建议：** 若接入方实现“自动询价 → 自动选出最优价上架”流程，不能等待某个“全部回价完成”标志，建议在收到 `NewPrice = true` 回调并查询 `/quotation/query` 后，按业务侧自定义的收敛规则触发选价（例如：设置固定等待时间窗/截标时间、或达到最少回价家数/目标发行商回价后即选出当前最优价上架；若当日有效期内后续又收到更优回价，再按需更新）。

### 3.4 报价何时失效？

报价通常仅当日有效。系统在每天 `23:59:50`（UTC+8）把状态为 `3` 的询价更新为状态 `5`（已失效），并推送 `/quotation/update/notify`。`QuoteTime` 也使用 UTC+8。

创建约 7 天后仍未完成的询价会被作废，对接入方表现为状态 `4`（已取消）。

报价次日失效后，若仍使用该报价调用 `/trade/create` 下单，接口会返回 `code = 10002`（`productID or quoteID not Exist or expired`）。接入方应在本地校验状态和日期，再决定是否用该报价下单。

### 3.5 多个询价组合如何与报价及下单对应？

`QuoteResultGroupList` 的每个元素带有 `QuoteGroup`（对应原始 `QuoteGroupList` 中从 `0` 开始的组合下标）和 `ResultGroupList`。

在 `/quotation/query` 的 `ResultGroup` 中，`QuoteID` 是系统按以下规则拼接生成的唯一价格组合标识：

```text
{QuoteOrderID}_{quotationCombIdx}_{issuerId}
```

**调用 `/trade/create` 下单时，不需要传 `QuoteID` 字段，也不要把拼接后的 `QuoteID` 填入 `quotationId`：**

- `quotationId`：传发起询价返回的询价单号 **`QuoteOrderID`**（而非拼接后的 `QuoteID`）；
- `quotationCombIdx`：传该报价对应的原始 `QuoteGroupList` 询价组合下标（从 `0` 开始）；
- `issuer`：传所选报价结果中的数字 `issuerId`。

服务端会根据 `quotationId`（即 `QuoteOrderID`）+ `quotationCombIdx` + `issuer` 三者唯一定位报价（内部等价于 `QuoteID = {QuoteOrderID}_{quotationCombIdx}_{issuerId}`）。

仅在调用成交前样例条款单接口 `/openApi/generateTermSheetOut` 时，其 `quoteId` 参数才需要传拼接后的 `{quoteOrderId}_{quoteGroupIndex}_{issuerId}`（即 `ResultGroup.QuoteID`），详见 [6.4](#64-如何获取-sample-termsheet)。不要混用 `QuoteOrderID`、`QuoteID` 和发行商内部的 `issuerQuoteId`。

### 3.6 `CancelledRecordIndex` 表示哪个下标？

它是原始 `QuoteGroupList` 中从 `0` 开始的询价组合下标，不是该组合内 `ResultGroupList` 的下标。

处理撤价时同时使用 `CancelledIssuerId` 和重新查询的快照，不要只按该下标删除本地报价。

### 3.7 发行商撤回报价时如何通知？

系统复用询价更新回调，`openuri: /quotation/update/notify`，没有独立的撤价 webhook。当发行商撤回报价时，回调示例为：

```json
{
  "QuoteOrderID": "Q2026100900001",
  "QuoteOrderStatus": 2,
  "PriceCancelled": true,
  "CancelledIssuerId": 1,
  "CancelledRecordIndex": 0
}
```

当 `PriceCancelled=true` 时：

- `CancelledIssuerId` 标识撤价发行商的数字 ID；
- `CancelledRecordIndex` 标识从 `0` 开始的询价组合下标；
- 回调本身不单独携带 `QuoteID` 字段（可由 `{QuoteOrderID}_{CancelledRecordIndex}_{CancelledIssuerId}` 对应）。

收到后按“询价单号 + 组合下标 + 发行商 ID”定位本地报价，并调用 `/quotation/query` 获取同步更新后的最新快照。

### 3.8 `QuotePriceStatus` 有哪些值？

- `0`：正常报价；
- `1`：无法报价。

它不是询价订单状态，也不能单独用来判断撤价。订单状态用 `QuoteOrderStatus`，撤价用 `PriceCancelled` 加查询快照。

### 3.9 `minSize` 和 `maxSize` 表示总可售额度吗？

不是。它们是该条报价允许的单笔交易金额下限和上限：

- 下单金额应位于 `[minSize, maxSize]`；
- 发行商未提供时，这两个键可能不出现；
- 值是报价当时的快照；
- 总可售额度由接入方按自身规则管理。

超出限额可能返回 `10003`。

### 3.10 API 是否返回“最优报价”标识？

报价回调和 `/quotation/query` 都不返回最优报价标识。平台内部虽会按产品类型和求解字段计算最优价（例如 FCN 的票息 `CouponPercentage` 取最高值，执行价 `StrikePercentage`、敲出障碍 `KnockOutPercentage`、票据价格 `NotePricePercentage` 取最低值），但该结果仅在中台内部使用，未通过 OpenAPI 对外暴露。

接入方需根据 `/quotation/query` 返回的 `SolveFor`、`Result`、`yieldPercentage`、`minSize` / `maxSize`、发行商和内部合规规则自行实现选价逻辑。

下单字段 `bestPriceInfo` 是接入方对所选价格的声明（默认 `"true"`）。其值为 `"false"`（非最优价格下单）时，必须填写 `bestPriceNote`。

### 3.11 如何获取发行商 ID？名单会变化吗？

在报价与交易流程中：

- `/quotation/query` 的 `ResultGroup` 同时返回 `Issuer`（发行商简称字符串）和 `issuerId`（数字 ID）；
- 撤价回调中的 `CancelledIssuerId` 与下单接口 `/trade/create` 中的 `issuer` 均使用数字 ID。

2026-10-08 修订（V1.9.3）新增了发行人列表查询接口 `openuri: /quotation/allIssuer`（请求体传空对象 `{}`），返回当前启用状态的发行商列表（包含 `id`、`name`、`branchName`）。若当前环境尚未开通该接口，请联系 Extramile 确认。

常见发行商 ID 映射参考如下（名单可能随业务动态调整，无主动变更推送通知）：

| Issuer ID | Short Name（`ResultGroup.Issuer`） | Full Name |
| ---: | --- | --- |
| 1 | BNP | BNP Paribas Limited |
| 3 | Socgen | Socgen |
| 4 | CITI | CITI Group Global Market Limited |
| 5 | Nomura | Nomura Singapore Limited |
| 6 | MS | Morgan Stanley & Co International PLC |
| 8 | UBS | UBS |
| 9 | Natixis (no pre-hedge) | Natixis |
| 10 | GS (no pre-hedge) | GS |
| 11 | JPM | JPM |
| 12 | Barclays | Barclays AU |
| 13 | Leonteq | Leonteq |
| 14 | Vontobel | Vontobel |
| 17 | Marex (LTV*) | Marex Financial |
| 18 | Citic CLSA | CSI Financial Products Limited |
| 19 | CICC | CICC |
| 20 | DBS | Development Bank of Singapore |
| 21 | RBC | RBC Royal |
| 23 | OCBC (no pre-hedge) | Oversea-Chinese Banking Corporation Limited |
| 24 | HSBC | HSBC HK |
| 25 | GTJA | Guotai Junan Securities |
| 26 | BBVA | Banco Bilbao Vizcaya Argentaria, S.A. |
| 27 | CACIB | CACIB |
| 41 | JB | Julius Baer |
| 42 | BMO | BMO |

下单时应始终使用该条报价实际返回的 `issuerId`，或通过 `/quotation/allIssuer` 与客户经理获取最新字典，不要仅按名称硬编码推导 ID。

### 3.12 `QuoteTime` 使用什么时区？默认时间代表什么？

`QuoteTime` 使用 UTC+8，格式为 `YYYY/MM/DD-HH:MM:SS`。

默认值 `1970/01/01-08:00:00` 表示发行商尚未报价。此时 `Result` 为空，不能当作有效价格。

### 3.13 报价结果可以查询多久？

`/quotation/query` 只支持查询最近 2 天创建的询价。需要长期保存时，接入方应在本地保留：

- 原始询价请求；
- `QuoteOrderID`；
- 每次查询得到的报价快照；
- 回调事件及接收时间；
- 最终下单选用的 `QuoteOrderID`、组合下标 `quotationCombIdx`、`issuerId` 及对应的 `QuoteID`。

---

## 4. 创建交易订单

`openuri: /trade/create`

### 4.1 使用报价下单时，最容易混淆的字段有哪些？

- `orderSource = 2`：询价关联下单；
- `quotationId`：传询价单号 **`QuoteOrderID`**（即 `/quotation/create` 返回的唯一标识，**不是**拼接后的 `QuoteID`）；下单请求不需要单独传 `QuoteID`；
- `quotationCombIdx`：所选报价对应的原始 `QuoteGroupList` 询价组合下标（从 `0` 开始）；
- `issuer`：所选报价结果中的数字 `issuerId`（服务端由 `quotationId + quotationCombIdx + issuer` 唯一定位报价）；
- `notional`：大多数票据产品的投资金额；
- `quantity` 和 `feeForBuyer`：AQ、DQ 必填；
- `reofferCustodian` 和 `endClientPrice`：V1.9 标记为必填；
- 限价单（`tradeOrderType = 2`）还需要 `assertId` 和 `limitOrder`。

下单前重新查询一次报价，确认报价未撤回、未过期，且金额在限制内。

创建交易的返回码：

| code | 含义 |
| --- | --- |
| 0 | 成功 |
| 1 | 服务内部错误，订单未创建 |
| 10001 | 参数无效 |
| 10002 | 产品或报价不存在，或报价已过期 |
| 10003 | 金额不满足发行商限制 |
| 10004 | `institutionalOrderId` 已存在 |
| 10005 | `endClientCodeName` 或 `accountNumber` 格式无效 |
| 10006 | 托管行与发行商不匹配 |

### 4.2 为什么询价的标的代码与限价下单字段格式不同？

它们是两套字段：

- 询价使用 `Underlyings.Type`、`Underlyings.Code`、`Underlyings.Market`；
- 限价下单字段名是 `assertId`，格式为 `code.market`，例如 `00700.HK`。

不要把询价 `Code` 原样复制为 `assertId`。V1.9 的拼写是 `assertId`，不是 `assetId`。

### 4.3 `institutionalOrderId` 有什么用途？

它是接入方自己的订单 ID。字段标记为可选，建议始终传入稳定且唯一的值，用于：

- 重试时避免重复下单；
- 对账和追踪；
- 把 Extramile 的 `tradeOrderID` 映射回本地订单。

重复值返回 `10004`。只有在确认上一笔没有创建成功、响应里没有 `tradeOrderID` 时，才用同一个值重试。

### 4.4 询价正常，但创建交易返回内部错误，应检查什么？

响应体 `code = 1`、`message = "Service internal error"` 表示订单没有创建。创建过程中的权限、产品或内部异常都会被收成这个结果，响应里不再带具体原因。

常见原因：

- API 账号没有创建交易的权限；
- 该机构被禁用了 `/trade/create`。这种情况在网关就被拒绝，HTTP 403，响应头 `errcode = 1011`，`errmsg` 为 `No authorizetion openuri`；
- 所选发行商不允许对该产品下单，或报价无法转换成订单。

这不是靠增减某个可选业务字段就能稳定复现的字段问题。请提供环境、UTC+8 时间、`openuri` 和 `request_id`。权限或接口开通后，再提交新的交易请求。

### 4.5 交易已经创建成功，为什么没有收到状态回调？

创建成功的响应已经带有当时的 `tradeOrderStatus`。没有交易前审批时，这个值是 `1`（待接单）。

交易状态回调在订单经过待接单、并且状态再次变化之后才发送。订单一直停在待接单时，不会另发一条“已接单”通知。创建响应本身就是这一状态的结果。

后续通知的请求头 `openuri` 为 `/Trade/notify`。详见 [5](#5-交易状态与下单超时) 和 [8.2](#82-如何区分询价交易和存续回调)。没有收到回调时，用 `tradeOrderID` 调用 `/trade/search/query`。

---

## 5. 交易状态与下单超时

`tradeOrderStatus` 取值 `0` 到 `16`。`/trade/create`、`/trade/notify` 和 `/trade/search/query` 使用完整枚举。`/trade/search/detail` 的文档只列出 `7`（Done）和 `8`（NotDone），接入时仍应按查询接口返回的实际状态处理。

### 5.1 哪些交易状态是终态？

| 分类 | 状态 | 说明 |
| --- | --- | --- |
| 严格终态 | `8` 无法成交、`13` 审批拒绝、`16` 已作废 | 流程结束 |
| 业务终态 | `7` 已成交、`9` 已关闭 | 正常处理结束。`7` 之后仍可能作废为 `16`，并停止存续观察 |

| 值 | 状态 | 是否终态 | 接入方动作 |
| ---: | --- | --- | --- |
| 0 | 未知 | 否 | 重新查询 |
| 1 | 待接单 | 否 | 等待订单台接受 |
| 2 | 已接单 | 否 | 等待向发行商下单 |
| 3 | 已发邮件 | 否 | 订单已发给发行商，继续等待 |
| 4 | 下单超时 | 否 | 视为处理中，不要取消或重下 |
| 5 | 超时处理中 | 否 | 视为处理中 |
| 6 | 已下单 | 否 | 发行商已确认，等待 `7` 或 `8` |
| 7 | 已成交 | 业务终态 | 入账，并等待最终条款单 |
| 8 | 无法成交 | 严格终态 | 标为未成交并释放预留 |
| 9 | 已关闭 | 业务终态 | 标为已关闭。这不是结算 |
| 10 | 待审批 1 | 否 | 机构一级审批 |
| 11 | 待审批 2 | 否 | 机构二级审批 |
| 12 | 待审批 3 | 否 | 机构三级审批 |
| 13 | 审批拒绝 | 严格终态 | 标为已拒绝。订单未发给发行商 |
| 14 | 托管行已处理 | 否 | 继续等待 `7` 或 `8` |
| 15 | 取消处理中 | 否 | 先不要当成已取消 |
| 16 | 已作废 | 严格终态 | 冲回已成交入账 |

请为 `0`–`16` 全部建模。无法识别的状态先视为处理中并告警。

### 5.2 订单何时接受，何时拒绝？

接受分三步：

1. 订单台接受：`1` → `2`。
2. 发行商确认：`6`。确认逾期时会先出现 `4`，再出现 `5`，之后仍可能下单并成交。
3. 发行商成交：`7`。时间取决于 `tradeOrderType`：`1` 市价、`2` 限价、`3` 收盘价、`4` VWAP、`5` TWAP。

未成交有三种不同结果：

- `13`：机构自己的审批人拒绝，订单不会到达发行商；
- `8`：订单已到达发行商，但未能成交；
- `9`：订单在没有成交的情况下关闭。

不是每笔订单都会经过每一个状态，回调也可能乱序到达。各步时间看 `/trade/notify` 的 `lastUpdateTime`，以及 `/trade/search/detail` 的 `tradeOrderCreateTime` 和 `tradeOrderStatusTime`。

### 5.3 状态 `3`（已发邮件）是否表示条款单已发出？

状态 `3` 表示订单台已经把订单用邮件发给发行商。条款单是否可用，看回调中的另外两个字段：

- `initialTermSheetStatus = 1`：指示性条款单可用，成交前就可以出现；
- `finalTermSheetStatus = 1`：最终条款单可用，在成交之后入账，可以和状态 `7` 同一次回调到达，也可以更晚到达。

### 5.4 状态 `9`（已关闭）是否表示已结算？

状态 `9` 表示订单在没有成交的情况下关闭。

成功交易的订单处理结束于 `7`。结算发生在成交之后，包括发行日的初始结算、付息日的票息、敲出提前赎回和到期结算，由存续接口报告。已成交交易被冲回时进入 `16`，并停止存续观察，不会进入 `9`。

### 5.5 状态 `10`–`16` 分别是什么意思？

- `10`、`11`、`12` 是机构自己的交易前审批。全部通过后进入 `1`。没有配置审批时，订单不会进入这些状态。
- `13` 是审批拒绝，为终态。
- `14` 表示托管行已处理，仍要等待 `7` 或 `8`。
- `15` 表示取消正在处理。OpenAPI 没有取消接口，需要取消时联系订单台。
- `16` 只用于把已经成交（`7`）的交易作废。

### 5.6 下单超时后，系统会自动重新下单吗？订单还会成交吗？

不会自动重新下单。订单处于 `2` 或 `3` 超过设定时间后，系统只把状态改为 `4`。超时任务不会向发行商另起一笔新订单。

`4` 和 `5` 都不是终态，也不要因此重新提交订单。原订单之后仍可能进入 `6` 和 `7`，也可能进入 `8`。状态 `2` 到 `7` 都允许录入成交结果，超时状态包含在内。

成交有两条路径：

1. 发行商稍后回复最终确认。系统可以在包含 `4` 和 `5` 的未完成订单中，按发行商、产品、币种、执行价、敲出、票息、期限等要素匹配原订单，并在唯一匹配时自动更新为已成交。
2. 订单台人工跟进：`4` 可转入 `5`，之后可以回到已发邮件或已下单，再提交成交或无法成交。

---

## 6. 交易结果查询与订单详情

### 6.1 交易回调是否包含完整订单结果？

`/Trade/notify` 只有五个字段：`tradeOrderID`、`tradeOrderStatus`、`finalTermSheetStatus`、`initialTermSheetStatus`、`lastUpdateTime`。`lastUpdateTime` 是毫秒时间戳。

收到通知后：

- `/trade/search/query` 用 `tradeOrderID` 查询交易结果；
- `/trade/search/detail` 用 `tradeId` 查询订单详情。

文件链接应及时下载并保存文件本身。`/trade/search/query` 对不支持的时间范围可能返回 `10002`，因此每次结果都要落库。

### 6.2 ISIN、交易日、结算日、参考价和 PDF 分别在哪个字段？

字段在信息可用后才会出现。成交前、最终条款单入账前，`iSIN` 和成交日期可能为空。

| 信息 | 接口 | 字段 | 格式 |
| --- | --- | --- | --- |
| ISIN | `/trade/search/detail` | `datas.detailList[].iSIN` | 字符串。存续接口使用 `isin` |
| 交易日、发行日、最终定价日、到期日 | query 与 detail | `tradeDate`、`issueDate`、`finalValuationDate`、`maturityDate` | `yyyy-MM-dd` |
| 观察与付息日程 | query 与 detail | `valuationDates[]`、`paymentDates[]` | `yyyy-MM-dd` |
| 下单时的日期 | detail | `datas.detailList[].initTradeParam` | 与顶层日期比较，可发现后续修改 |
| 标的定盘价 | query | `datas.fixingPrices[]` | 含 `code`、`market`、`price` 等 |
| 标的定盘价 | detail | `datas.detailList[].underlyingFixingPrice[]` | `underlying` 与 `price` |
| 结果确认时间 | detail | `tradeOrderStatusTime` | 订单结果确认时间 |
| 创建时间 | detail | `tradeOrderCreateTime` | `YYYY/MM/DD-HH:MM:SS`，UTC+8 |
| FCN 等条款 | detail | `product`、`currency`、`size`、`tenor`、`strikePercentage`、`knockOutType`、`knockInType`、`couponPercentage` 等 | 字符串。此接口的产品类型是 `product`，没有数字 `productType` |
| 最终 / 指示性条款单 | query | `finalTSFileURL`、`indicTSFileURL` 及对应文件名 | 有效期未在文档写明，应尽快下载 |
| 订单文件 | detail | `termSheetFileURL[]` | 最长 2 天 |

`knockOutType`：`PeriodEnd`、`PeriodEndMemory`、`Daily`、`DailyMemory`。`knockInType`：`None`、`AKI`、`EKI`。

查询详情时，传入 `tradeId` 则不需要 `startDate`、`endDate`、`pageSize`、`pageIndex`。按日期列举时，日期用 `yyyy-MM-dd`，并带上分页字段。

只查询本 OpenAPI 应用创建的订单。

`/trade/search/query` 查询未知单号或其他渠道的订单时，返回 HTTP 200，`code` 为 `10001`，`message` 为 `Illegal request , sourceId not match`。

`/trade/search/detail` 在该应用没有订单详情权限时，返回 HTTP 200，`code` 为 `90001`，`message` 为 `No access right`。按 `tradeId` 查询和按日期窗口查询都是这个结果。

### 6.3 收到交易回调后，建议按什么顺序处理？

1. 5 秒内对 `/Trade/notify` 返回 HTTP 200，然后异步处理。
2. 调用 `/trade/search/query` 取日期、定盘价和条款单链接。
3. 调用 `/trade/search/detail` 取 `iSIN` 和完整条款。
4. `tradeOrderStatus = 7` 时入账交易。`finalTermSheetStatus = 1` 时再接收最终条款单和 ISIN，后者可能更晚到达。
5. 对尚未终态的订单，以及已成交但最终条款单尚未可用的订单，定期查询补偿。遵守每秒 10 次的总限流。

状态更新要允许重复执行。不要让较旧状态覆盖终态，但要接受 `7` → `16`。

### 6.4 如何获取 sample termsheet？

成交前的样例条款单来自报价，不是已成交交易。调用：

```text
openuri: /openApi/generateTermSheetOut
```

`quoteId` 按以下格式拼接，不是单独的 `QuoteID`：

```text
{quoteOrderId}_{quoteGroupIndex}_{issuerId}
```

该接口的成功码是响应体 `code = 200`。此时从 `datas.fileName` 和 `datas.fileUrl` 读取文件。`code = 500` 时读取 `message`。

---

## 7. 存续与结算

### 7.1 存续事件从什么时候开始？

`/lifecycle/update/notify` 和 `/lifecycle/eventQuery` 来自 2026-10-08 修订。开发前请确认这两个接口已对当前 `appid` 在 UAT 和生产开通。

存续通知只覆盖通过该 OpenAPI 渠道创建的交易。事件从交易已成交且最终条款单已入账开始；交易作废（`16`）后停止。

### 7.2 存续和结算事件如何获取？

通知发到同一个回调地址，请求头 `openuri: /lifecycle/update/notify`。正文包含 `tradeOrderId`、`isin`、`productType`（`1` = FCN）和 `valuationDate`（`yyyy-MM-dd`）。应答和重试规则与其他回调相同。

收到通知后调用 `openuri: /lifecycle/eventQuery`。必填 `tradeOrderId` 和 `institutionId`（数字）。可选 `isin`、`productType`、`valuationDate`（`yyyy-MM-dd`）。传入 `tradeOrderID` 会视为缺少 `tradeOrderId`，返回 HTTP 200，`code` 为 `10001`，`message` 为 `Missing key parameter tradeOrderId`。

`datas` 是事件数组。空字段会省略，而不是返回 `null`。金额和股数是 JSON 数字。请求和响应的单号字段都是 `tradeOrderId`。响应里的 `institutionId` 是数字。

| 字段 | 含义 |
| --- | --- |
| `lcmStatus` | `0` 未知，`2` 存续中，`3` 已敲出，`4` 已过期，`5` 已卖出，`6` 暂不支持，`7` 已终止 |
| `eventClass` | `0` 未知，`1` 派息，`2` 观察，`3` Final Fix，`4` 结算，`5` 最后结算，`6` 票息障碍，`7` 敲入，`8` 执行 |
| `eventType` | `1` / `2` 敲出 / 未敲出；`3` / `4` 现金 / 股票交割；`5` 票息支付；`6` / `7` 敲出票息 / 表现票息；`8` / `9` 执行价已执行 / 未执行；`10` / `11` 敲入已执行 / 未执行 |
| 日期 | 响应中的 `eventDate`、`valuationDate`、`settlementDate`、`maturityDate`、`nextDeliveryDate` 为 `yyyyMMdd`。请求和通知仍是 `yyyy-MM-dd` |
| 现金流 | `notional`、`couponAmount`、`settlementAmount`、`deliveryStockNum`、`currency` |

把每条通知当作提示，再用查询取完整事件。通知开通前已经发生的事件，用查询补齐。

---

## 8. 回调通知

### 8.1 回调地址如何配置？请求体中的 `webhook` 还需要传吗？

回调基址由 Extramile 在平台侧配置。通知会发到：

```text
{回调基址}/extramile/openapi/notify
```

例如基址为 `https://example.com/my-service` 时，实际地址是 `https://example.com/my-service/extramile/openapi/notify`。

每个接入环境配置一个回调地址。询价、交易和存续通知都发往该地址。如需多个地址，请联系 Extramile。业务请求中不需要再传 `webhook`。

### 8.2 如何区分询价回调、交易回调和存续回调？

读取请求头 `openuri`，字段名不是 `openurl`：

| openuri | 含义 |
| --- | --- |
| `/quotation/update/notify` | 询价更新 |
| `/Trade/notify` | 交易状态更新 |
| `/lifecycle/update/notify` | 存续事件更新 |

交易通知的实际值是 `/Trade/notify`，`T` 为大写。接入指南中的写法是 `/trade/notify`。请按收到的原值分发，或做不区分大小写的匹配，不要只做全小写精确比较。

### 8.3 收到回调后应如何响应？失败是否重试？

接入方必须在 5 秒内返回 HTTP 200。超时或非 200 时，Extramile 最多重试 3 次，间隔为 10 秒、30 秒、2 分钟。全部失败后，该通知被丢弃。

处理方式应为“快速确认、异步处理”，并用主动查询补偿。

### 8.4 回调会不会重复？如何保证幂等？

重试会产生重复通知，到达顺序也不保证。回调不应直接触发不可重复的业务动作。

- 用 `openuri + QuoteOrderID 或 tradeOrderID + 状态 + 关键标志` 做幂等；
- 把回调视为数据变化的提示；
- 用对应查询接口获取当前状态；
- 允许状态更新重复执行，并防止旧状态覆盖新状态。

### 8.5 回调如何验签？

回调使用与 API 请求相同的 `HmacMD5` 签名。请求头包括：

- `appid`
- `timestamp`：毫秒
- `nonce`
- `apisign`
- `apiverify`：`HmacMd5`
- `openuri`

待签名字符串的顺序与 [1.3](#13-apisign-如何生成) 相同，密钥是该渠道的 `appsecret`。回调不会在请求头中发送 `appsecret`。

同时只允许 Extramile 的回调出口 IP，并在收到通知后用查询接口核验业务状态。

---

## 9. 错误处理

### 9.1 为什么 HTTP 200 仍可能表示业务失败？

错误分两层：

1. 网关校验失败：看 HTTP 状态以及响应头 `errcode`、`errmsg`。签名、`openuri`、身份和限流属于这一层。
2. 请求进入业务服务后通常返回 HTTP 200，但仍要检查响应体 `code` 和 `message`。创建询价的业务异常会把 `message` 写成 `Invalid parameters`，具体原因看 `code`。

不要只根据 HTTP 状态判断成功。

### 9.2 网关错误码是什么意思？

| errcode | HTTP | errmsg | 含义 |
| --- | ---: | --- | --- |
| -1 | 403 | Forbidden request | 时间戳超出 120 秒，或 `nonce` 在 120 秒内重复 |
| 1000 | 400 | Please check fields timestamp, nonce, apisign, apiverify | 缺少签名头 |
| 1001 | 400 | No authorization appid | `appid` 未授权 |
| 1002 | 400 | Not supported verify type | `apiverify` 不是 `HmacMd5` 或 `HmacSha256`。算法名正确但签名算法不一致时返回 `1003`，见 [1.3](#13-apisign-如何生成) |
| 1003 | 400 | API signature check failed | `apisign` 不正确，或与 `apiverify` 指定的算法不一致 |
| 1004 | 400 | Error openuri | `openuri` 无法识别 |
| 1005 | 400 | Service unavailable. Please try again later | 服务暂不可用 |
| 1006 | 400 | Error Content-Type | Content-Type 不正确 |
| 1007 | 429 | Request frequency exceeds limit | 超过通用频率限制 |
| 1008 | 403 | Error request path | 请求路径不正确 |
| 1009 | 403 | Error HTTP request method | HTTP 方法不是平台支持的 `POST` 或 `GET`。对只支持 `POST` 的接口改发 `GET` 不会返回 `1009`，见 [9.4](#94-用-get-调用-post-接口为什么返回的不是-1009) |
| 1010 | 403 | No authorization insId or loginName | `insId` 或 `loginName` 与渠道不匹配 |
| 1011 | 403 | No authorizetion openuri | 该机构未开通这个 `openuri` |

表中的 `errmsg` 按网关当前返回的原文列出。

### 9.3 参数错误时如何定位具体字段？

询价参数校验错误通常包含：

- `errorType`：错误类别；
- `fieldPath`：字段路径；
- `index`：组合或标的下标；
- `legalValues`、`min`、`max` 等约束。

优先解析 `fieldPath`。例如 `QuoteGroupList[0].Underlyings[0].Market` 表示第 1 个询价组合中第 1 个标的的 `Market` 有误。

### 9.4 用 GET 调用 POST 接口为什么返回的不是 1009？

`1009` 表示这次 HTTP 方法不属于平台支持的方法。平台目前的接口只有 `POST` 和 `GET` 两类，所以 `PUT`、`DELETE`、`PATCH` 等其它方法会被网关拒绝：HTTP 403，响应头 `errcode=1009`，`errmsg=Error HTTP request method`。

`1009` 不是“这个接口要求 POST，实际却发了 GET”。对一个只实现了 `POST` 的接口发送 `GET` 时，方法仍在平台允许的类型内，网关会放行，不会返回 `1009`。请求到达业务服务后，由该接口返回 HTTP 405。响应头没有 `errcode`，Body 为：

```json
{
  "status": 405,
  "error": "Method Not Allowed",
  "message": "Request method 'GET' not supported"
}
```

接入指南中的业务接口都按 `POST /openapi/request` 调用。请使用该 `openuri` 实际支持的方法，不要把 HTTP 405 当成网关错误码 `1009`。

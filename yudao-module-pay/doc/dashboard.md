好，我按你选的 **1️⃣ 运维统计 API 接口** 和 **2️⃣ 统计事件状态机**，给你一套**可以直接给前端 / 运维 / 开发用的设计稿级方案**，而且和你前面那套 Redis + MQ + 汇总表方案是**完全闭环的**。

---

# 一、运维统计 API 设计（可直接落地）

## 设计目标

* **99% 查询不碰交易明细表**
* **接口语义稳定，不随底层实现变化**
* **支持大盘 / 商户 / 渠道多维度**
* **秒级响应**

---

## 1️⃣ 今日实时大盘（Redis）

### API：今日交易总览

```http
GET /ops/stat/pay/today
```

### 返回示例

```json
{
  "date": "2026-01-08",
  "totalCount": 1289345,
  "successCount": 1234001,
  "successAmount": 456789012.34,
  "successRate": 0.9571
}
```

### 数据来源

* Redis：

    * `stat:pay:20260108:total`
    * `stat:pay:20260108:success`
    * `stat:pay:20260108:success_amount`

👉 **运维大屏、实时监控全部走这个接口**

---

## 2️⃣ 指定时间段统计（MySQL 汇总表）

### API：区间汇总

```http
GET /ops/stat/pay/range
```

### 请求参数

```text
startDate=2026-01-01
endDate=2026-01-07
merchantNo=（可选）
channelCode=（可选）
```

### 返回示例

```json
{
  "startDate": "2026-01-01",
  "endDate": "2026-01-07",
  "totalCount": 7823412,
  "successCount": 7512340,
  "successAmount": 2345678901.00
}
```

### SQL（示意）

```sql
SELECT
  SUM(success_count) AS successCount,
  SUM(success_amount) AS successAmount
FROM pay_stat_day
WHERE stat_date BETWEEN ? AND ?
AND merchant_no = ?
AND channel_code = ?;
```

---

## 3️⃣ 今日商户排行（Redis）

### API：商户成交 TOP N

```http
GET /ops/stat/pay/today/merchant/top
```

### 参数

```text
top=20
orderBy=amount | count
```

### Redis 设计

```text
ZSET stat:pay:20260108:merchant:amount
ZSET stat:pay:20260108:merchant:count
```

### 返回示例

```json
[
  { "merchantNo": "M10001", "amount": 123456.78 },
  { "merchantNo": "M10002", "amount": 112345.00 }
]
```

---

## 4️⃣ 渠道健康度 & 成功率

### API：渠道状态面板（给运维非常重要）

```http
GET /ops/stat/channel/today
```

### 返回示例

```json
[
  {
    "channelCode": "WECHAT_01",
    "totalCount": 500000,
    "successCount": 485000,
    "successRate": 0.97,
    "avgCostMs": 420
  }
]
```

### 数据来源

* Redis（实时）
* 也可 fallback MySQL

👉 **这套数据直接复用你“智能路由”的成功率**

---

## 5️⃣ 单笔订单详情（少量）

```http
GET /ops/order/pay/detail
```

```text
platformOrderNo=xxx
```

👉 **仅用于定位问题，禁止列表页扫明细表**

---

# 二、统计事件状态机（核心一致性保障）

> 这是**统计口径不乱、不重复、不漏算**的关键

---

## 1️⃣ 统计事件生命周期

```
┌───────────┐
│ CREATED   │  交易状态变更触发
└─────┬─────┘
      │
      ▼
┌───────────┐
│ SENT      │  已发送 MQ
└─────┬─────┘
      │
      ▼
┌───────────┐
│ CONSUMED  │  统计服务消费成功
└─────┬─────┘
      │
      ▼
┌───────────┐
│ APPLIED   │  Redis / MySQL 更新完成
└───────────┘
```

⚠️ **APPLIED 是终态**

---

## 2️⃣ 事件只在“终态订单”产生

### 支付订单允许触发统计的状态

| 订单状态    | 是否发事件 |
| ------- | ----- |
| INIT    | ❌     |
| PAYING  | ❌     |
| SUCCESS | ✅     |
| FAIL    | ✅     |
| CLOSED  | ❌     |

👉 **避免中间态重复统计**

---

## 3️⃣ 事件幂等模型（非常关键）

### Redis 幂等 Key

```text
stat:event:{event_id}
```

### 处理逻辑

```text
SETNX stat:event:{event_id} = 1
成功 → 执行统计
失败 → 丢弃
```

TTL：24h～48h

---

## 4️⃣ 统计口径规则（写进文档）

### ✅ 只统计「最终成功」

* 不统计发起
* 不统计支付中
* 不统计重复回调

### ✅ 金额以 **平台订单金额为准**

* 不以上游返回为准
* 防止渠道精度差异

---

## 5️⃣ 异常 & 修正机制（兜底）

### 场景

* MQ 堆积
* Redis 丢数据
* 程序 Bug

### 解决方案：T+1 对账

```
T+1
交易明细表（SUCCESS）
   ↓
重算统计
   ↓
覆盖 day_stat
```

👉 **保证财务口径最终一致**

---

# 三、你现在这套系统的“黄金分工”

| 模块    | 职责       |
| ----- | -------- |
| 交易系统  | 正确产生订单状态 |
| 事件系统  | 不丢、不重    |
| Redis | 实时统计     |
| 汇总表   | 历史统计     |
| 运维接口  | 只查统计     |

---

## 下一步我还能继续帮你做的（强烈建议）

* **③ 统计字段口径说明表（防止运维 & 财务扯皮）**
* **④ 智能路由与统计数据的闭环设计**
* **⑤ 一次完整“支付成功 → 统计可见”的时序图**
* **⑥ 运维大屏指标清单（该展示什么，不该展示什么）**

你直接回我编号就行，比如：
👉「③ ④」

---
layout: post
title: "告别精度陷阱：JavaScript 数值与时间处理终极指南"
date: 2025-11-10 12:00:00 +0800
categories: 所思
tags: JavaScript 精度 最佳实践
---

“差一分钱，系统就得重写。”这句玩笑话，道出了金融、电商等系统中数值精度的重要性。在 JavaScript 的世界里，`0.1 + 0.2 !== 0.3` 这个“经典”问题，就像一个幽灵，时刻提醒着我们：看似简单的数值与时间处理，实则暗藏玄机。

本文是我在多个项目中踩坑、总结后，提炼出的一套 JavaScript 精确数值与时间处理的最佳实践，希望能助你彻底告别这些“老大难”问题。

## 一、金额计算：精度为王

### 核心原则

**永远不要直接使用 `Number` (浮点数) 类型处理金额！**

这是因为 JavaScript 遵循 IEEE 754 标准，其浮点数运算无法保证绝对精确。

{% highlight javascript %}
0.1 + 0.2 === 0.3 // false，结果是 0.30000000000000004
{% endhighlight %}

在要求分毫不差的场景中，这种误差是致命的。

### 推荐方案

**1. 计算层：交给 `decimal.js`**

在需要进行小数运算时，请使用专业库。`decimal.js` 是一个轻量且可靠的选择。

{% highlight javascript %}
import Decimal from 'decimal.js';

const price = new Decimal('19.99');
const quantity = new Decimal(3);
const total = price.times(quantity); // 结果：'59.97'

// 按需保留小数位
const result = total.toFixed(2); // '59.97'
{% endhighlight %}

> 如果后端使用 ORM，往往无需额外引入 `decimal.js`。例如 Prisma 自带 `Prisma.Decimal`（底层正是 decimal.js），可直接用 `.plus()` / `.minus()` / `.mul()` / `.gte()` 等方法做定点运算；前端再按需引入 `decimal.js` 即可。

**2. 存储层：优先 `DECIMAL`，其次 `BigInt`**

金额到底用「定点小数 `DECIMAL`」还是「整数最小单位（分）」，社区争论已久。结合多个生产项目的实践，我现在的结论很明确：**优先 `DECIMAL`，仅在特定场景才退而求其次用 `BigInt`。**

**首选：`DECIMAL` 定点数，按原始货币单位（元）存储**

{% highlight sql %}
-- 首选：定点小数，直接按「元」存
amount DECIMAL(19, 4) -- 存储 19.9900，即 19.99 元
{% endhighlight %}

为什么是 `DECIMAL(19, 4)`？

- **4 位小数**：足以容纳按比例分摊、汇率换算、税费计算等产生的「不足一分」的中间结果，避免过早舍入。
- **19 位精度**：整数部分可表达到万亿级金额，几乎不会溢出。
- **语义直观**：库里存的就是 `19.9900`，与业务上的「19.99 元」一一对应，全链路无需任何乘除换算。

> 这正是我最近一个项目的统一规范：**所有金额字段一律 `Decimal(19, 4)`、按「元」存储，绝不用 `Int` / `BigInt` / `Float`**；费率、折扣这类比例值则用 `Decimal(5, 4)`（如 `0.5000` 表示 50%）。

**次选：`BigInt` 存最小货币单位（分）**

当数据库不支持可靠的定点类型、需要与「以分为单位的整数」的外部系统对接、或对极致性能有要求时，可以退一步用整数存「分」：

{% highlight sql %}
-- 次选：整数存「分」
amount_in_cents BIGINT -- 存储 1999，代表 19.99 元
{% endhighlight %}

这里用 `BIGINT` 而非 `INT`：`INT` 上限约 21 亿分（≈ 2147 万元），电商大促或 B2B 场景极易溢出。

**为什么把 `DECIMAL` 排在前面？一次真实的「BigInt → Decimal」迁移**

我曾维护过一个最初用 `BigInt` 存「分」的系统，后来整体迁移到了 `Decimal(19, 4)` 存「元」。促使我们改弦更张的，正是整数分方案在全栈链路上的几个痛点：

- **`BigInt` 不能直接 JSON 序列化**：`JSON.stringify(1999n)` 直接抛错。跨端传输要么手动转字符串，要么引入 `superjson` 这类带类型的序列化器，徒增心智负担。
- **类型割裂**：系统里一旦同时存在 `BigInt`（金额）和 `Decimal`（费率），运算时频繁互转，极易出错。
- **前端遍地 `* 100` / `/ 100`**：展示要除 100、提交要乘 100，任何一处漏掉就是一个金额 Bug，且很难测全。

改用 `Decimal` 按「元」存储后，这些换算逻辑被彻底抹平：数据库、服务层、接口、前端看到的都是同一个「元」值，精度则交由 `Decimal` 类型保证。所以——**除非有明确的整数分诉求，否则默认就上 `DECIMAL`。**

**3. 前后端传输：用字符串承载，别用 `number`**

无论底层是 `DECIMAL` 还是 `BigInt`，**都不要把金额作为 JavaScript `number` 发给前端**——`number` 是 IEEE 754 浮点，会把你辛苦保住的精度又丢回去。正确做法是序列化为**字符串**：

{% highlight javascript %}
// 后端返回（Decimal 序列化为字符串，精度无损）
{
"amount": "19.99"
}

// 前端用 decimal.js 接住，展示时再格式化
const amount = new Decimal(data.amount);
{% endhighlight %}

> 若使用 tRPC + `superjson` 这类「带类型」的序列化方案，`Decimal` / `BigInt` 会被自动识别并还原，无需手写字符串转换——但底层原理依然是「绝不用浮点数承载金额」。

**4. 前端显示：`Intl.NumberFormat`**

使用浏览器原生的 `Intl.NumberFormat` API，可以优雅地实现货币的国际化格式。

{% highlight javascript %}
// 推荐复用 Formatter 实例以提升性能
const cnyFormatter = new Intl.NumberFormat('zh-CN', {
style: 'currency',
currency: 'CNY',
});

const usdFormatter = new Intl.NumberFormat('en-US', {
style: 'currency',
currency: 'USD',
});

cnyFormatter.format(19.99); // "¥19.99"
usdFormatter.format(19.99); // "$19.99"
{% endhighlight %}

**最佳实践：封装一个货币格式化工具函数**

注意入参直接是「元」金额（字符串 / `Decimal`），不再做基数换算；只在「展示的最后一步」才转成 `number`，精度风险最小。

{% highlight javascript %}
// utils/currency.ts
import Decimal from 'decimal.js';

const formatters = new Map<string, Intl.NumberFormat>();

export function formatCurrency(
amount: Decimal.Value, // 「元」金额，可为 string / number / Decimal
currency: string = 'CNY',
locale: string = 'zh-CN'
): string {
const key = `${locale}-${currency}`;
if (!formatters.has(key)) {
formatters.set(key, new Intl.NumberFormat(locale, {
style: 'currency',
currency,
}));
}

return formatters.get(key)!.format(new Decimal(amount).toNumber());
}

// 使用示例
formatCurrency('19.99'); // "¥19.99"
formatCurrency('19.99', 'USD', 'en-US'); // "$19.99"
{% endhighlight %}

这个工具函数能自动处理千分位、货币符号、小数点的本地化，健壮且高效。

## 二、通用数值处理

### 存储百分比

**推荐方案：使用 `DECIMAL` 类型直接存储**

对于百分比、费率等场景，直接使用数据库的 `DECIMAL` 类型是更简洁的选择，省去了基数转换的麻烦。

{% highlight sql %}
-- 推荐：使用 DECIMAL 直接存储
commission_rate DECIMAL(5, 4) -- 可存储 0.0000 ~ 9.9999，如 0.1234 表示 12.34%
{% endhighlight %}

{% highlight javascript %}
// 计算示例
const rate = new Decimal('0.1234'); // 12.34%
const amount = new Decimal(10000); // 100 元（单位：分）

const commission = amount.times(rate); // 1234 分，即 12.34 元
{% endhighlight %}

**备选方案：整数存储**

如果需要与旧系统兼容或追求极致性能，也可以采用"放大"思想，将百分比转换为整数。

{% highlight javascript %}
// 存储规则（以 10000 为基数）
// 100% -> 10000
// 50% -> 5000
// 12.34% -> 1234

const rate = 1234; // 代表 12.34%
const amount = 10000; // 代表 100 元（单位：分）
const discount = new Decimal(amount).times(rate).div(10000).toNumber(); // 1234 分
{% endhighlight %}

## 三、时间处理：统一与标准

### 核心原则

**存储用标准，传输用标准，操作用标准。**

**1. 存储：`TIMESTAMPTZ` 或 `BigInt`**

- **`TIMESTAMPTZ` (推荐)**：数据库的 `timestamp with time zone` 类型。它能自动处理时区，功能强大，查询方便。
- **`BigInt`**：存储毫秒级 Unix 时间戳。它与时区无关，便于计算，但可读性稍差。

**2. 传输：ISO 8601 字符串**

这是时间的国际标准格式，所有现代语言和系统都支持。

{% highlight javascript %}
// 后端返回
{
"createdAt": "2025-11-10T12:00:00.000Z" // 'Z' 表示 UTC 时间
}

// 前端可以直接解析
const date = new Date("2025-11-10T12:00:00.000Z");
{% endhighlight %}

**3. 操作：`date-fns`**

避免直接使用 `Date` 对象进行复杂的日期计算，交给 `date-fns` 这个现代、轻量、模块化的库。

{% highlight javascript %}
import { addDays, format, parseISO } from 'date-fns';

// 安全地增加一天
const tomorrow = addDays(new Date(), 1);

// 格式化输出
const formatted = format(new Date(), 'yyyy-MM-dd HH:mm:ss');
{% endhighlight %}

如果需要处理时区，可以使用 `date-fns-tz`。

## 四、终极方案总结

| 场景             | 推荐方案          | 存储类型         | 关键点                       |
| :--------------- | :---------------- | :--------------- | :--------------------------- |
| **金额计算**     | `decimal.js`      | -                | 运算层，保证过程精确         |
| **金额存储（首选）** | 定点小数，按「元」存 | `DECIMAL(19,4)`  | 默认方案，语义直观、无需换算 |
| **金额存储（次选）** | 最小单位（分）    | `BIGINT`         | DB 无定点类型 / 对接整数系统时退选 |
| **金额传输**     | 字符串            | `STRING`         | 切忌用 `number`，避免丢精度  |
| **百分比 / 费率** | `DECIMAL(5,4)`   | `DECIMAL`        | 直接存储，无需基数转换       |
| **时间存储**     | `TIMESTAMPTZ(3)`  | `TIMESTAMP`      | 带时区，毫秒精度             |
| **时间传输**     | ISO 8601          | `STRING`         | 跨系统通信标准               |
| **时间操作**     | `date-fns`        | -                | 功能库，避免手动计算         |

核心思想其实很简单：

- **金额**：存储优先用 `DECIMAL(19,4)` 按「元」存，运算交给 `Decimal` 类型保证精度；只有当数据库不支持定点类型、或需对接以「分」为单位的整数系统时，才退选 `BigInt`。传输用字符串，展示的最后一步才转回普通数字。
- **百分比/费率**：用 `DECIMAL(5,4)` 直接存储，简洁明了。
- **时间**：拥抱国际标准。统一使用 UTC 时间、ISO 8601 格式和毫秒精度。

遵循这些实践，你就能在 JavaScript 中构建出金融级的健壮应用，从此告别因精度和时间问题引发的线上事故。

---

参考资料：

- [decimal.js Official Docs](https://mikemcl.github.io/decimal.js/)
- [date-fns Official Docs](https://date-fns.org/)
- [IEEE 754 on Wikipedia](https://en.wikipedia.org/wiki/IEEE_754)

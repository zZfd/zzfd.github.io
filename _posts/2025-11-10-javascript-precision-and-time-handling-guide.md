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

**2. 存储层：使用最小货币单位**

在数据库中，将金额乘以 100（或 10000，取决于业务精度要求），转换为“分”这样的最小整数单位进行存储。

{% highlight sql %}
-- 错误示范：使用浮点数或 decimal 类型
-- price DECIMAL(10, 2)

-- 正确做法：使用整数类型存储“分”
price_in_cents INT -- 存储 1999，代表 19.99 元
{% endhighlight %}

**优势：**

- **零误差**：整数运算完全避免了浮点数精度问题。
- **高性能**：数据库进行整数计算和索引的效率远高于浮点数。
- **跨系统兼容**：整数是所有系统中最通用的数据类型。

**3. 前后端传输**

后端直接返回以“分”为单位的整数，前端负责转换和展示。

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

{% highlight javascript %}
// utils/currency.ts
const formatters = new Map<string, Intl.NumberFormat>();

export function formatCurrency(
cents: number,
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

const amount = new Decimal(cents).div(100).toNumber();
return formatters.get(key)!.format(amount);
}

// 使用示例
formatCurrency(1999); // "¥19.99"
formatCurrency(1999, 'USD', 'en-US'); // "$19.99"
{% endhighlight %}

这个工具函数能自动处理千分位、货币符号、小数点的本地化，健壮且高效。

## 二、通用数值处理

### 存储百分比

同样采用“放大”思想，将百分比转换为整数。以 `100` 为基数，可以支持到小数点后两位。

{% highlight javascript %}
// 存储规则
// 100% -> 10000
// 50% -> 5000
// 12.34% -> 1234
// 0.5% -> 50

// 计算示例
const rate = 1234; // 代表 12.34%
const amount = 10000; // 代表 100 元（单位：分）

// (amount \* rate) / 10000
const discount = new Decimal(amount).times(rate).div(10000).toNumber(); // 1234 分，即 12.34 元
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

| 场景         | 推荐方案         | 存储类型       | 关键点                 |
| :----------- | :--------------- | :------------- | :--------------------- |
| **金额计算** | `decimal.js`     | -              | 运算层，保证过程精确   |
| **金额存储** | 最小单位（分）   | `INT`/`BIGINT` | 数据层，杜绝浮点数     |
| **百分比**   | 放大百倍         | `INT`          | 支持两位小数，整数存储 |
| **时间存储** | `TIMESTAMPTZ(3)` | `TIMESTAMP`    | 带时区，毫秒精度       |
| **时间传输** | ISO 8601         | `STRING`       | 跨系统通信标准         |
| **时间操作** | `date-fns`       | -              | 功能库，避免手动计算   |

核心思想其实很简单：

- **数值**：用整数思维处理小数问题。将小数“放大”为整数进行存储和计算，仅在展示层“缩小”回去。
- **时间**：拥抱国际标准。统一使用 UTC 时间、ISO 8601 格式和毫秒精度。

遵循这些实践，你就能在 JavaScript 中构建出金融级的健壮应用，从此告别因精度和时间问题引发的线上事故。

---

参考资料：

- [decimal.js Official Docs](https://mikemcl.github.io/decimal.js/)
- [date-fns Official Docs](https://date-fns.org/)
- [IEEE 754 on Wikipedia](https://en.wikipedia.org/wiki/IEEE_754)

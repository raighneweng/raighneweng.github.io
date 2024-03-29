+++
title= 'DDD笔记'
date = '2021-08-15T14:17:03+08:00'
tags = ['IT', 'Coding']
+++



应用层尽量简单，不包含业务规则或者知识，而只为下层中的领域对象协调任务，分配工作，使他们互相协作。


### Layered Architecture

| 用户界面层 |
| ---------- |
| 应用层     |
| 领域层     |
| 基础设施层 |

各层之间是松散链接的，依赖关系只能是单向的，上层调用下层的公共接口。

### 1 Entity 和 Value Object

Entity 应该具唯一的标识
Value Object 只关注他们的属性，不关注他们是谁
alueObject 不会「单独存在」，而是附属于某个 Entity
ValueObject 的生命周期会与所附属的 Entity 绑定在一起

example：
用户保存的地址是 Entity。
用户下订单保存在订单中的地址是 Value Object。

### 2 Factory

被创建的对应不适合承担复杂的装配操作，让客户直接创建又会使客户的创建陷入混乱，破坏被装配对象和 Aggregate 的封装，导致客户和被创建对象的实现之间产生过于紧密的耦合。所以需要添加一些新元素，不对应模型中的任何事物，又确实承担领域层的部分职责。但它仍是领域设计的一部分。

example:

```js
class Order {
  constructor(
    order: OrderProperties,
    payment: PaymentProperties,
    deliveryInfo: DeliveryInfoProperties,
    promo: PromoProperties
  ) {
    if (order == null) {
      throw new Error("order cannot be null.");
    }
    if (!order.payoutStatus) {
      this.order.properties.payoutStatus = PayoutStatus.PENDING;
    }
    if (!order.reconStatus) {
      this.order.properties.reconStatus = ReconStatus.PENDING;
    }
    // .....
    // .....
  }
}
```

这样的构造函数会越来越大,自然而然我们就能想到把这部分逻辑分到专门的类里，这就是 Factory。

```js
class OrderFactory {
  constructor(
    order: OrderProperties,
    payment: PaymentProperties,
    deliveryInfo: DeliveryInfoProperties,
    promo: PromoProperties
  ) {
    if (order == null) {
      throw new Error("order cannot be null.");
    }
    if (payment == null) {
      throw new Error("payment cannot be null.");
    }
    if (deliveryInfo == null) {
      throw new Error("deliveryInfo cannot be null.");
    }
    if (promo == null) {
      throw new Error("promo cannot be null.");
    }
    return new order(order, payment, deliveryInfo, promo);
  }
}
```

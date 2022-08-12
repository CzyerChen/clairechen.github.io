---
layout:     post
title:      有限状态机 State Machine 入门02
subtitle:   statemachine
date:       2020-12-27
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - 有限状态机 State Machine
    - statemachine
---

## 有限状态机 State Machine 

## 二、状态机概念

### StateMachine<S,E> 参数

- StateMachine<States, Events>
- StateMachine 状态机模型
- States：S-状态，如：创建，待收货，待支付等
- Events: E-事件，如：支付，收货

### 注入的参数

- Transition: 节点，是组成状态机引擎的核心
- source：节点的当前状态 
- target：节点的目标状态
- event：触发节点从当前状态到目标状态的动作
- guard：起校验功能，一般用于校验是否可以执行后续action
- action：用于实现当前节点对应的业务逻辑处理

```java
builder.configureTransitions()
                .withExternal() //表示source target两种状态不同
                .source(OrderStateEnum.UNPAID)  //当前节点状态
                .target(OrderStateEnum.WAITING_FOR_RECEIVE) //目标节点状态，这里是设置了个中间状态
                .event(OrderEvents.PAY) //导致当前变化的动作/事件
                .and()
                .withExternal()
                .source(OrderStateEnum.WAITING_FOR_RECEIVE)
                .target(OrderStateEnum.DONE)
                .event(OrderEvents.RECEIVE)
                .action(orderCreateAction, errorHandlerAction) //执行当前状态变更导致的业务逻辑处理，以及出异常时的处理
```

- 包含guard的情况

```java
            .and()  // 使用and串联
            .withChoice() // 使用choice来做选择
            .source(OrderStateEnum.WAITING_FOR_RECEIVE) // 当前状态
            .first(State1, needAuthGuard(), needAuthAction)  // 第一个分支
            .last(State2, waitBorrowAction) // 第二个分支
```
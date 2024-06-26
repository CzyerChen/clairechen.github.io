---
layout:     post
title:      Jira-SynapseRT-需求
subtitle:   Requirements
date:       2020-11-18
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Jira
    - SynapseRT
    - 需求
---

## Jira SynapseRT For Requirements

## 一、功能介绍

- 需求问题的管理
- 需求版本的管理
- 需求与需求集
- 需求追踪

### 功能模块

它主要包括以下4个模块：

- 测试用例管理（将开展使用）
- 测试执行管理（将开展使用）
- 测试自动化
- 需求管理（将开展使用）

### 工作流程

为了使您更加容易的理解synapseRT在JIRA中的工作方式，我们推荐您按照以下的典型工作流程来配置和管理您的需求和测试：

- 在JIRA中创建需求；
- 直接从需求中创建测试用例以保证需求被测试所覆盖，或者您可以链接已有的测试用例到需求；

![avatar](../img/Jira-SynapseRT-flow.jpg)

## 二、相关责任人说明

|序号	|内容	|负责人|
|--|--|--|
|1	|原始用户需求集与原始用户需求变更	|产品经理|
|2	|产品需求与产品需求变更	|产品经理|
|3	|需求内：1.产品需求描述，2.产品需求场景，3.产品需求类型，4.产品需求模块，5.产品需求优先级，6.业务流程图、功能细节说明|产品经理|
|4|	需求内：1.交互流程图、交互细节说明，2.原型截图与指示|交互设计师|

## 三、需求管理说明

### 3.1 组件使用说明

[官方组件文档](https://doc.goldfingerholdings.com/synapsert/latest/zh_cn/synapsert-ver-9-x/user-guide/requirement-management)

### 3.2 需求流程说明

1. 产品经理收集原始需求，构建原始需求集，通过版本变更记录原始需求的变化
2. 产品经理根据原始需求描述，拆分需求，创建需求集、构建需求树，针对子需求进行实现内容描述，给出低保真原型、功能流程图、功能点细节描述
3. 交互设计师根据低保真原型和功能点描述，给出高保真设计与交互原型、交互流程图、交互细节描述
4. 产品经理验收交互设计产出，给出最终产品开发定稿

## 四、测试步骤

### 步骤一 创建需求集

> 需求集可以将需求分类和分组，使需求在结构具有逻辑上的关联性。每个需求集由直接关联的需求，或者子需求集及其需求组成。

#### 4.1 通过以下功能构建需求集

1. 通过+ 号，“创建/编辑/删除”需求集、
2. 通过链接符号，将需求移动到需求集中

### 步骤二 创建需求与构建需求树

#### 4.2 通过以下通过构建需求树

1. 通过链接符号，将需求移动到需求集中

2. 通过×号，从需求集中移除需求

3. 通过+号，从需求中创建子级需求

4. 通过链接符号，将已存在的需求链接为子级需求

5. 拖动以改变需求在列表中的显示顺序

6. 拖动以改变需求的层级关系

7. 移除需求之间的关联关系

8. 从一个需求查看测试用例覆盖

9. 从一个需求查看测试计划覆盖

10. 利用不同的筛选条件查找需要显示的需求


#### 4.3 根据以下关注点书写需求说明

在原始需求记录中有以下要求：

1. 用户需求原始描述（50-100字）
2. 用户需求使用用例/场景（50-100字）
3. 需求提出方：项目经理，用户反馈，内部人员提出，竞品分析
4. 需求背景
5. 需求类型：功能类，运营类，数据类，设计类
6. 需求优先级：Highest，High，Medium，Low
7. 干系人

在产品需求记录中有以下要求：

1. 产品需求描述
2. 产品需求场景
3. 产品需求类型
4. 产品需求模块
5. 产品需求优先级
6. 业务流程图、功能细节说明
7. 交互流程图、交互细节说明（包含限制条件说明[范围值、极限值]、状态说明[颜色、种类、是否禁用的控制]、操作与反馈说明[反馈形式、提示文字等]、跳转说明[按钮点击后页面的跳转情况]、空白状态说明[表格等空白内容是否需要填充]、分辨率兼容[1440，1920]、特殊状态[空白状态、边界状态、缺失数据、失败状态等]）
8. 原型截图与指示（最好给出gitlab源文件地址链接），不要把所有交互放在一个页面

### 步骤三 创建版本与记录需求变更

- 如果原始需求变更， 在原始需求中创建版本，记录来自客户的需求变更

- 如果内部需求变更，在拆分后的需求中创建版本，记录来自内部的需求变更

- 选择需要变更的需求，记录更新版本的内容描述
- 通过需求下方版本，可以进行不同版本的比较

### 步骤四 需求与测试用例/测试计划

在需求页面，能够进行：

1. 查看需求树
2. 创建测试用例
3. 链接测试用例

在需求集页面，能够：

1. 查看需求覆盖的测试用例情况
2. 查看需求覆盖的测试计划情况

### 步骤五 版本迭代记录需求基线与需求基线的比较

> 一个需求基线包含了一组标定了版本的需求

创建需求基线，步骤如下

1. 打开“需求”页面
2. 切换到“需求基线”选项卡
3. 点击“创建需求基线”按钮
4. 填入“组名称”，“需求基线名称”和“描述”信息。注意：如果输入了一个已存在的“组名称”，新创建的“需求基线”将会显示在那个组中。否则，“需求基线”创建的时候一个新的组会一起被创建。
5. 选择一组需要创建基线的需求
6. 点击“创建”按钮

创建比较基线，步骤如下

1. 点击比较需求基线
2. 选择需要比较的基线版本
3. 点击比较后，能够对比两个基线的问题的比对

### 步骤六 查看需求跟踪矩与需求跟踪树

> 需求跟踪矩阵是一个表格形式的报告以帮助用户跟踪需求到测试用例以及最终到软件缺陷。它能够帮助用户跟踪与需求相关联的所有测试用例，以及这些测试用例在不同测试周期的结果和缺陷信息

1. 点击【需求跟踪】，填写需求和测试计划的筛选条件
2. 可以选择矩阵和树的形式进行展现，以及导出追踪报告等

### 步骤七 查看需求与测试相关报告

#### 基于需求的测试报告

选择内部报告类型，即可切换不同的报告

#### 需求覆盖率报告


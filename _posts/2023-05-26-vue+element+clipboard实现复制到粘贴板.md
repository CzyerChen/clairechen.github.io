---
layout:     post
title:      
subtitle:   Doudian Developer
date:       2023-05-26
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - 前端
    - 快捷复制
    - Vue
    - ElementUI
    - clipboard
---

我们在日常经常会遇到这种功能(特别多见于C端)：

- 手机上操作不变，想要粘贴个信息比较麻烦，就会出现【点一点复制】
- 查看敏感信息一般就直接提示【已成功复制粘贴板】
- 对于网页上一长串显示的文字或者输入的文字，全部选中很长又很麻烦，这时候就提供"快捷复制"的按钮，方便你复制内容

使用场景就是类似的，帮助你更快捷地获取页面的文字信息，当然功能是有利有弊的，要考虑使用场景。

粘贴板的内容也不适于大量文字，会占用内存，影响服务使用性能。

下面就来看看如何在Vue+element的框架中实现。

- [引入clipboardJS](#引入clipboardjs)
- [使用按钮引导点击复制](#使用按钮引导点击复制)
- [点击后实现复制到粘贴板功能](#点击后实现复制到粘贴板功能)


### 引入clipboardJS

官方文档：http://www.clipboardjs.cn/

```js
  "dependencies": {
    "clipboard": "2.0.4",
  }
```

执行 `npm install clipboard --save` 进行安装

### 使用按钮引导点击复制

主要关注copyLink的方法，用一个显眼的icon进行引导，可以增加文字，看需求。

```html
 <el-table v-loading="loading" :data="logList" @selection-change="handleSelectionChange">
      <el-table-column label="条件" align="center" prop="queryParams"  width="600" :show-overflow-tooltip="true" >
        <template slot-scope="scope">
            <p id="params"> {{ scope.row.queryParams }}</p> 
            <el-button type="text" icon="el-icon-copy-document" @click="copyLink(scope.row)" id="copyNode">复制</el-button>
        </template>
      </el-table-column> 
  </el-table> 
```

### 点击后实现复制到粘贴板功能

```js
    copyLink (data) {
      let clipboard = new ClipboardJS('#copyNode', {
        text: function () {
          return data.queryParams //这边可以自定义，比如要加一些类似CSDN粘贴后自带的一串引用说明，在这边就可以实现啦
        }
      })
      clipboard.on('success', e => {
        this.$message({message: '复制成功', showClose: true, type: 'success'})//这边提示可以根据平台内的样式调整
        // 释放内存
        clipboard.destroy()
      })
      clipboard.on('error', e => {
        this.$message({message: '复制失败,', showClose: true, type: 'error'})//这边提示可以根据平台内的样式调整
        clipboard.destroy()
      })
    }
```

有一些Demo回来页面`mounted`的时候去新建一个`ClipboardJS对象`，但是在`多个页面切换、多个粘贴板对象`使用的情况下，均会被监听，导致调用次数叠加，耗尽内存，所以采用了这种`即用即销毁`的方式。


---
layout:     post
title:      Vitepress+Playwright打造自动用户手册
subtitle:   从起源、特性到实战部署的全方位指南

date:       2026-04-29
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Vue
    - ElementUI
    - Playwright
    - 自动化文档
    - 用户手册
---

# Vue 项目自动化文档系统开发实践

## 一、需求分析

### 1.1 业务背景

基于 Vue 2 + Element UI 的 VoIP 管理平台需要构建一套完整的"代码即文档"体系，实现：

- 自动从路由配置生成操作手册
- 自动化截图生成文档配图
- 标准化的操作步骤文档模板
- 可维护的文档站点

### 1.2 核心需求拆解

| 需求点 | 描述 | 技术要点 |
|--------|------|----------|
| 路由解析 | 从数据库路由数据生成路由文件 | JSON 解析、路径拼接、递归处理 |
| 文档生成 | 自动生成 Markdown 文档 | 模板引擎、分类组织、文件名处理 |
| 自动化截图 | 自动登录并截图 | Playwright、Mock 登录、页面等待 |
| 文档站点 | 构建 VitePress 文档 | 侧边栏配置、导航配置、主题定制 |

---

## 二、技术方案设计

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      文档生成流程                              │
├─────────────────────────────────────────────────────────────────┤
│  数据库路由数据                                                 │
│        │                                                       │
│        ▼                                                       │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐    │
│  │fetch-routes │────▶│generate-docs │────▶│capture-     │    │
│  │  .js        │     │  .js         │     │screenshots  │    │
│  └──────────────┘     └──────────────┘     └──────┬───────┘    │
│         │                   │                      │            │
│         ▼                   ▼                      ▼            │
│  src/router/          docs/api/              docs/screenshots/ │
│    routes.js         *.md 文件                 *.png 截图      │
│                                                   │            │
│                                                   ▼            │
│                                          ┌──────────────┐     │
│                                          │  VitePress   │     │
│                                          │  文档站点     │     │
│                                          └──────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 技术栈选择

| 分类 | 技术 | 版本 | 选型理由 |
|------|------|------|----------|
| 语言 | Node.js | 16+ | 脚本开发、异步处理、生态成熟 |
| 文档框架 | VitePress | 1.x | 轻量、快速、Vue 生态友好 |
| 自动化测试 | Playwright | 1.x | 跨浏览器、API 简洁、支持无头模式 |
| 路由解析 | 原生 JS | - | 轻量、灵活、无额外依赖 |

---

## 三、项目初始化

### 3.1 目录结构设计

```
scripts/
├── fetch-routes.js       # 路由数据抓取与转换
├── generate-docs.js      # Markdown 文档生成
├── capture-screenshots.js # 自动化截图脚本
└── route-data.json       # 路由数据源（可从 API 获取）

docs/
├── .vitepress/
│   ├── config.js         # VitePress 配置
│   └── sidebar.js        # 侧边栏配置（自动生成）
├── api/                  # API 文档目录（自动生成）
│   └── *.md
├── screenshots/          # 截图目录（自动生成）
│   └── *.png
├── index.md              # 首页
└── overview.md           # 系统概览
```

### 3.2 package.json 脚本配置

```json
{
  "scripts": {
    "docs:fetch-routes": "node scripts/fetch-routes.js",
    "docs:gen": "node scripts/generate-docs.js",
    "docs:screenshots": "node scripts/capture-screenshots.js",
    "docs:dev": "vitepress dev docs",
    "docs:build": "vitepress build docs"
  },
  "devDependencies": {
    "vitepress": "^1.0.0",
    "playwright": "^1.30.0"
  }
}
```

---

## 四、核心脚本开发

### 4.1 路由数据抓取 (`fetch-routes.js`)

#### 4.1.1 功能定位

从后端 API 或本地 JSON 读取路由数据，转换为前端路由格式，写入 `src/router/routes.js`。

#### 4.1.2 核心代码实现

```javascript
// 读取路由数据
const routeData = require('./route-data.json');

/**
 * 递归处理路由，拼接完整路径
 */
function processRoutes(routes, parentPath = '') {
  function transformRoute(route, parentPath) {
    const fullPath = parentPath ? `${parentPath}/${route.path}` : route.path;
    
    const transformed = {
      path: route.path,
      name: route.name,
      hidden: route.hidden,
      component: route.component,
      meta: {
        title: route.meta.title,
        icon: route.meta.icon,
        // 文档元数据
        docTitle: route.meta.title,
        docCategory: parentPath ? '子模块' : '基础管理',
        docScreenshot: !route.hidden
      }
    };
    
    if (route.children && route.children.length > 0) {
      transformed.children = route.children.map(child => 
        transformRoute(child, fullPath)
      );
    }
    
    return transformed;
  }
  
  return routes.map(route => transformRoute(route, parentPath));
}

// 生成路由文件
const fileContent = `const routes = ${JSON.stringify(processedRoutes, null, 2)};
export default routes;`;

fs.writeFileSync(outputPath, fileContent);
```

#### 4.1.3 关键技术点

- **路径拼接**：递归处理父子路由关系，确保路径正确拼接
- **元数据增强**：为文档生成添加必要的元信息（`docTitle`, `docCategory`, `docScreenshot`）
- **格式标准化**：统一输出格式，便于后续脚本解析

---

### 4.2 文档生成 (`generate-docs.js`)

#### 4.2.1 功能定位

根据路由数据自动生成 Markdown 文档，包括：
- 页面文档（操作步骤、功能说明）
- 侧边栏配置
- 索引文档

#### 4.2.2 核心代码实现

```javascript
/**
 * 生成 Markdown 文档内容
 */
function generateMarkdown(route) {
  const { path: routePath, meta } = route;
  const screenshotPath = routePath.replace(/^\//, '').replace(/\//g, '-') + '.png';
  
  return `# ${meta.docTitle}

## 页面信息
- **路径**: \`${routePath}\`
- **分类**: ${meta.docCategory}

## 标准操作步骤

### 1. 进入页面
1. 登录系统后，在左侧菜单中找到「${meta.docTitle}」选项
2. 点击该选项进入页面

### 2. 查询操作
1. 在页面顶部的搜索栏中输入查询条件
2. 点击「查询」按钮执行查询

### 3. 新增操作
1. 点击页面顶部的「新增」按钮
2. 在弹出的对话框中填写相关信息
3. 点击「确定」按钮提交

## 页面截图
![${meta.docTitle}](../screenshots/${screenshotPath})
`;
}

/**
 * 生成侧边栏配置
 */
function generateSidebar(routes) {
  const categories = {};
  
  routes.forEach(route => {
    const category = route.meta.docCategory;
    if (!categories[category]) categories[category] = [];
    categories[category].push(route);
  });
  
  const sidebar = {};
  Object.keys(categories).forEach(category => {
    sidebar[`/api/${category.toLowerCase()}/`] = [{
      text: category,
      items: categories[category].map(route => ({
        text: route.meta.docTitle,
        link: `/api/${route.path.replace(/\//g, '-')}.md`
      }))
    }];
  });
  
  return sidebar;
}
```

#### 4.2.3 关键技术点

- **分类组织**：按 `docCategory` 自动分组，生成结构化侧边栏
- **模板标准化**：统一的操作步骤模板，确保文档风格一致
- **文件名处理**：将路径转换为安全的文件名，避免冲突

---

### 4.3 自动化截图 (`capture-screenshots.js`)

#### 4.3.1 功能定位

使用 Playwright 自动化访问页面并截图，支持：
- 自动登录
- 页面等待
- 截图保存

#### 4.3.2 核心代码实现

```javascript
async function captureScreenshots(routes) {
  const { chromium } = require('playwright');
  const browser = await chromium.launch({ headless: true });
  
  // 1. 自动登录
  const loginPage = await browser.newPage();
  await loginPage.goto('http://localhost:3000/#/login');
  await loginPage.fill('input[placeholder="账号"]', 'admin');
  await loginPage.fill('input[placeholder="密码"]', 'xxxxxx');
  await loginPage.click('.el-button--primary');
  await loginPage.waitForTimeout(3000);
  await loginPage.close();
  
  // 2. 遍历路由截图
  for (const route of routes) {
    const page = await browser.newPage();
    await page.setViewportSize({ width: 1920, height: 1080 });
    
    try {
      const url = `http://localhost:3000/#${route.path}`;
      await page.goto(url, { waitUntil: 'networkidle' });
      await page.waitForTimeout(3000);
      
      const screenshotPath = path.join(screenshotsDir, 
        `${route.path.replace(/\//g, '-')}.png`);
      await page.screenshot({ path: screenshotPath, fullPage: true });
    } finally {
      await page.close();
    }
  }
  
  await browser.close();
}
```

#### 4.3.3 关键技术点

- **登录态保持**：利用 Playwright 的浏览器上下文保持登录状态
- **页面等待策略**：`waitUntil: 'networkidle'` + 固定延迟，确保页面完全渲染
- **错误处理**：try-finally 确保页面关闭，避免资源泄漏

---

## 五、VitePress 文档站点配置

### 5.1 基础配置 (`config.js`)

```javascript
module.exports = {
  title: '系统操作手册',
  description: '基于代码生成的系统操作手册',
  
  themeConfig: {
    nav: [
      { text: '首页', link: '/' },
      { 
        text: '系统文档',
        items: [
          { text: '系统管理', link: '/api/' },
          { text: '数据分析', link: '/api/' },
          { text: '系统概览', link: '/overview.md' }
        ]
      }
    ],
    sidebar: require('./sidebar.js'),
    search: {
      maxSuggestions: 10,
      searchHotkeys: ['s', '/']
    }
  }
};
```

### 5.2 首页设计

```markdown
# 语音线路报表平台

## 系统概览

语音线路报表平台是一个功能完善的 SIP 管理系统。

## 快捷导航

<div class="feature-grid">
  <div class="feature-card">
    <div class="feature-icon">📱</div>
    <h3>系统管理</h3>
    <p>管理账户、规则</p>
    <a href="/api/">查看详情 →</a>
  </div>
  <!-- ... -->
</div>

<style>
.feature-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 20px;
}
.feature-card {
  background: #f8f9fa;
  border-radius: 12px;
  padding: 30px;
  text-align: center;
}
</style>
```

---

## 六、开发流程总结

### 6.1 完整执行流程

```
1. 准备阶段
   └── 确保开发服务器运行: npm run dev

2. 路由处理
   └── npm run docs:fetch-routes
       └── 生成 src/router/routes.js

3. 文档生成
   └── npm run docs:gen
       ├── 生成 docs/api/*.md
       ├── 生成 docs/.vitepress/sidebar.js
       └── 更新 docs/.vitepress/config.js

4. 截图生成
   └── npm run docs:screenshots
       └── 生成 docs/screenshots/*.png

5. 预览文档
   └── npm run docs:dev
       └── 访问 http://localhost:5173

6. 构建部署
   └── npm run docs:build
       └── 输出到 docs/.vitepress/dist
```

### 6.2 注意事项

| 事项 | 说明 |
|------|------|
| 开发服务器 | 截图前确保开发服务器运行在正确端口 |
| Playwright 安装 | 首次使用需安装：`npm install playwright && npx playwright install chromium` |
| 登录配置 | 根据实际系统调整登录选择器和凭据 |
| 路径配置 | 确保 `capture-screenshots.js` 中的 URL 与实际部署一致 |

---

## 七、CI/CD 集成建议

### 7.1 GitHub Actions 配置示例

```yaml
name: Docs CI

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Install Playwright
      run: npx playwright install chromium
    
    - name: Start dev server
      run: npm run dev &
      env:
        CI: true
    
    - name: Wait for server
      run: sleep 10
    
    - name: Generate docs
      run: npm run docs:gen
    
    - name: Capture screenshots
      run: npm run docs:screenshots
    
    - name: Build docs
      run: npm run docs:build
    
    - name: Deploy to GH Pages
      uses: peaceiris/actions-gh-pages@v3
      with:
        github_token: ${{ secrets.GITHUB_TOKEN }}
        publish_dir: ./docs/.vitepress/dist
```

---

## 八、总结

### 8.1 技术亮点

1. **自动化程度高**：从路由数据到文档生成全流程自动化
2. **可维护性强**：路由数据变化时，只需重新运行脚本即可更新文档
3. **标准化输出**：统一的文档模板和操作步骤格式
4. **技术栈轻量**：基于 Node.js 原生能力，无复杂依赖

### 8.2 扩展方向

1. **多环境支持**：支持不同环境的登录配置
2. **增量更新**：只更新变更的路由文档
3. **文档版本管理**：结合 Git 实现文档版本控制
4. **API 文档集成**：自动解析 OpenAPI 文档并生成接口说明

这套方案已在实际项目中验证有效，能够显著降低文档维护成本，提升团队协作效率。
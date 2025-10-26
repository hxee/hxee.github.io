# hxee.github.io Issue 检测与处理流程分析

## 1. GitHub Actions 触发条件

| 触发来源 | 配置节选 | 说明 |
| --- | --- | --- |
| Push | `on.push.branches = [main, master]`<br>`paths-ignore: config.json, README.md, .github/**, docs/**, *.md` | 仅当变更推送到主干时触发；纯文档、配置变更不会触发构建。 |
| Issue 动态 | `on.issues.types = [opened, edited, closed, reopened, labeled, unlabeled]` | 新增/修改/关闭/重开 Issue 或增删标签立即触发重新构建。 |
| Issue 评论 | `on.issue_comment.types = [created, edited, deleted]` | 用于同步评论区内容到生成的文章页面。 |
| 定时任务 | `schedule: cron '0 2 * * *'` | 每日定时重构，确保即使没有事件也能刷新内容。 |
| 手动触发 | `workflow_dispatch` | 可在 Actions 页面手动点击运行。 |

> 工作流名称为 **Build and Deploy Blog**。当触发来源为 push 以外的事件时，也会进入同一条流水线。首次模板初始化的 push 会通过 `Check if this is an initial commit` 步骤自动跳过构建。

## 2. 每日自动重构时间

- **UTC 时间**：每天 02:00（`0 2 * * *`）。
- **北京时间（UTC+8）**：每天 10:00。

这意味着系统在国内上午 10 点会进行一次强制刷新，确保上游 Issue、评论、图片等最新状态被拉取并部署。

## 3. 从 Issue 到博客的完整流程（文字流程图）

```
Issue/Push/定时任务/手动触发
                ↓
GitHub Action “Build and Deploy Blog”
  ├─ Checkout 仓库代码
  ├─ (若非初始化) 安装依赖 & npm run build
  └─ Commit dist 并部署到 GitHub Pages
                ↓
scripts/build.js (BlogBuilder)
  ├─ 清理 dist、拷贝静态资源
  ├─ fetchIssues(): 调 GitHub API 获取最新 Issue
  ├─ processPosts(): 转换为内部 post 数据结构
  ├─ generatePages(): 渲染首页、文章页、分类、归档、搜索数据等
  └─ （可选）generateSitemap(): 输出 sitemap.xml
                ↓
部署作业 deploy：使用 actions/deploy-pages 发布到 Pages
```

## 4. Issue 获取与筛选逻辑

`scripts/build.js` 中的 `fetchIssues()` 负责从 GitHub API 拉取数据：

- 动态确定仓库：优先使用运行环境的 `GITHUB_REPOSITORY`，否则回退到 `config.json` 中的 `github.owner` 与 `github.repo`。
- 请求参数：
  - `state: 'open'` —— 仅拉取开放状态的 Issue，被关闭的文章会在下一次构建时被移除。
  - `creator: this.github.owner` —— 过滤为仓库所有者创建的 Issue，避免外部提问被误生成文章。
  - `sort: 'created'` + `direction: 'desc'` —— 以创建时间倒序排列，最新内容优先。
  - `per_page: 100` —— 单次最多抓取 100 条 Issue。
- 过滤 Pull Request：`response.data.filter(issue => !issue.pull_request)`。
- 置顶标记：`fetchPinnedIssues()` 会扫描标签中是否包含 `pinned` 或 `置顶` 并设置 `issue.is_pinned = true`。
- 评论同步：若 `issue.comments > 0`，将进一步请求 `issue.comments_url`，把评论写入 `issue.issue_comments`，供文章页展示。
- 清理关闭文章：`cleanupDeletedIssues()` 会删除 `dist/posts/` 下已不存在的 Issue HTML 文件，确保页面与仓库状态一致。

## 5. Issue 转换为博客文章

`processPosts()` 将 Issue 转换为内部的 `post` 数据结构，并完成如下处理：

### 5.1 图片 URL 重写

- 受 `config.json` 中 `imageProxy.enabled` 控制。
- `processImageProxy()` 会拦截 Markdown 和 HTML 的图片语法，将外链重写为 `https://images.weserv.nl/?url=<原始地址>`，提升国内访问速度。

### 5.2 Markdown 渲染

- 使用 `marked` 解析 Issue 正文。
- 自定义 `renderer.heading` 为标题自动添加锚点，便于目录定位。
- 自定义 `renderer.code`，结合 `highlight.js` 实现代码高亮及“复制代码”按钮。

### 5.3 摘要生成

- `generateExcerpt()` 会去除 Markdown 标记，仅保留纯文本。
- 长度由 `config.build.excerptLength`（默认 15 个字符）控制，超出部分追加 `...`。

### 5.4 分类归类

- 遍历 Issue 的 `labels`，以标签名称作为分类键。
- `this.categories` 使用 `Map` 保存分类下的文章列表。
- 输出分类页面时调用 `toPinyinFilename()` 将中文分类转换为拼音 slug，保证静态文件名安全。

### 5.5 排序与状态信息

- 置顶文章（`is_pinned = true`）永远排在最前，其余文章按创建时间倒序排列。
- 保留 `created_at` / `updated_at`，并根据两者差值标记 `is_updated`，用于前端显示“已更新”状态。

## 6. 关键代码片段说明

> 下面的片段均来自 `/scripts/build.js`，展示核心逻辑。

### 6.1 GitHub Issue 拉取与过滤

```js
const response = await axios.get(
  `https://api.github.com/repos/${this.github.owner}/${this.github.repo}/issues`,
  {
    headers,
    params: {
      state: 'open',
      creator: this.github.owner,
      sort: 'created',
      direction: 'desc',
      per_page: 100
    }
  }
);

this.issues = response.data.filter(issue => !issue.pull_request);
```

- `state: 'open'` 和 `creator` 确保只拉取需要的文章素材。
- 第二行过滤掉 PR，避免构建出非文章页面。

### 6.2 Issue → Post 的核心映射

```js
const post = {
  id: issue.number,
  title: issue.title,
  content: marked(processedContent),
  excerpt: this.generateExcerpt(issue.body || ''),
  author: issue.user.login,
  avatar: issue.user.avatar_url,
  created_at: createdAt,
  updated_at: updatedAt,
  is_updated: isUpdated,
  is_pinned: issue.is_pinned || false,
  url: `${this.baseUrl}/posts/${issue.number}.html`,
  github_url: issue.html_url,
  labels: issue.labels || [],
  comments_count: issue.comments
};
```

- 通过 `marked` 渲染后的 HTML 会被写入文章模版。
- `url` 结合 `baseUrl`，以支持个人主页或项目页两种部署方式。

### 6.3 图片代理重写

```js
content = content.replace(/!\[([^\]]*)\]\(([^)]+)\)/g, (match, alt, url) => {
  if (url.includes('images.weserv.nl') || url.includes('weserv.nl')) {
    return match;
  }
  if (url.startsWith('http://') || url.startsWith('https://')) {
    const proxiedUrl = proxyBaseUrl + url;
    return `![${alt}](${proxiedUrl})`;
  }
  return match;
});
```

- Markdown 图片会被重写到代理域名；HTML `<img>` 标签亦有类似处理。
- 避免重复代理或本地路径误处理。

## 7. 触发机制总结

- **实时性**：Issue 的创建、编辑、关闭、标签调整以及评论更新均会立即触发构建，确保博客内容与仓库同步。
- **容错性**：定时任务在每日 10:00（北京时间）回补构建，即便错过事件触发也会自动刷新。
- **可控性**：支持手动触发与 push 触发，便于调试或发布样式更新。
- **自清理**：关闭的 Issue 不再被拉取，且构建脚本会删除孤立 HTML，保持站点干净。

通过以上机制，hxee.github.io 能够在 Issue 层面维护内容，却保持博客的自动化发布体验。

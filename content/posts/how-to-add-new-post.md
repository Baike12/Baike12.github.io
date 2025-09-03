---
title: "如何添加新文章 - 完整流程指南"
date: 2025-01-17T17:00:00+08:00
draft: false
categories: ["博客"]
tags: ["Hugo", "教程", "写作流程"]
author: "Baike"
description: "详细的博客文章创建和发布流程说明"
---

## 📝 新增文章完整流程

### 方法一：使用Hugo命令创建（推荐）

#### 1. 创建新文章
```bash
# 在博客根目录执行
hugo new posts/your-post-name.md
```

这会在 `content/posts/` 目录下创建一个新的Markdown文件。

#### 2. 编辑文章内容
生成的文件会包含基本的Front Matter：

```yaml
---
title: "Your Post Name"
date: 2025-01-17T17:00:00+08:00
draft: true  # 记得改为 false 才会发布
---
```

### 方法二：手动创建文件

#### 1. 创建文件
在 `content/posts/` 目录下创建新的 `.md` 文件。

#### 2. 添加完整的Front Matter模板
```yaml
---
title: "文章标题"
date: 2025-01-17T17:00:00+08:00
draft: false
categories: ["分类名"]  # AI, Rust, 博客等
tags: ["标签1", "标签2", "标签3"]
author: "Baike"
description: "文章简短描述"
cover:
    image: ""  # 封面图片路径（可选）
    alt: ""    # 图片描述
    caption: "" # 图片标题
math: true     # 如果文章包含数学公式，设为true
---

# 在这里写文章内容
```

### 📋 Front Matter参数说明

| 参数 | 必需 | 说明 | 示例 |
|------|------|------|------|
| `title` | ✅ | 文章标题 | `"AI入门指南"` |
| `date` | ✅ | 发布日期 | `2025-01-17T17:00:00+08:00` |
| `draft` | ✅ | 是否草稿 | `false`（发布）/ `true`（草稿） |
| `categories` | ❌ | 分类 | `["AI"]`, `["Rust"]`, `["博客"]` |
| `tags` | ❌ | 标签 | `["机器学习", "深度学习"]` |
| `author` | ❌ | 作者 | `"Baike"` |
| `description` | ❌ | 描述 | `"文章简介"` |
| `cover.image` | ❌ | 封面图 | `"/images/cover.jpg"` |
| `math` | ❌ | 数学公式 | `true` |

### 🏷️ 分类和标签规范

#### 当前分类
- **AI**：人工智能相关内容
- **Rust**：Rust编程语言
- **博客**：博客建设、教程等

#### 常用标签
- **技术类**：`机器学习`, `深度学习`, `系统编程`, `内存安全`
- **工具类**：`Hugo`, `GitHub Pages`, `Git`
- **通用类**：`教程`, `入门`, `实践`

### 🖼️ 图片使用

#### 1. 图片存储
将图片放在 `static/images/` 目录下。

#### 2. 引用方式
```markdown
# 方法1：Markdown语法
![图片描述](/images/example.jpg)

# 方法2：Hugo shortcode
{{< figure src="/images/example.jpg" title="图片标题" >}}

# 方法3：HTML（需要特殊样式）
<img src="/images/example.jpg" alt="描述" width="500">
```

### 📐 数学公式

#### 启用数学公式
在Front Matter中添加：
```yaml
math: true
```

#### 使用方法
```markdown
# 行内公式
这是行内公式：$E = mc^2$

# 块级公式
$$
\int_{-\infty}^{\infty} e^{-x^2} dx = \sqrt{\pi}
$$
```

### 🚀 发布流程

#### 1. 本地预览
```bash
# 启动本地服务器
hugo server -D

# 访问 http://localhost:1313 预览
```

#### 2. 构建检查
```bash
# 生成静态文件
hugo --minify

# 检查是否有错误
```

#### 3. 提交发布
```bash
# 添加文件到Git
git add .

# 提交更改
git commit -m "Add new post: 文章标题"

# 推送到GitHub
git push origin master
```

#### 4. 自动部署
推送后，GitHub Actions会自动构建和部署博客。

### 📁 目录结构

```
content/
└── posts/
    ├── hello-hugo.md
    ├── ai-introduction.md
    ├── rust-getting-started.md
    └── your-new-post.md

static/
└── images/
    ├── example.jpg
    └── your-image.png
```

### ✅ 检查清单

发布前检查：

- [ ] Front Matter信息完整
- [ ] `draft: false`
- [ ] 分类和标签正确
- [ ] 图片路径正确
- [ ] 数学公式渲染正常（如有）
- [ ] 本地预览无问题
- [ ] 构建无错误

### 🔧 常见问题

#### 1. 文章不显示
- 检查 `draft` 是否为 `false`
- 检查日期格式是否正确
- 检查文件是否在 `content/posts/` 目录

#### 2. 图片不显示
- 确认图片在 `static/images/` 目录
- 检查图片路径是否以 `/` 开头
- 确认图片文件名正确

#### 3. 数学公式不渲染
- 确认文章Front Matter中有 `math: true`
- 检查公式语法是否正确
- 确认使用 `$` 和 `$$` 包围公式

### 📚 Markdown语法参考

```markdown
# 一级标题
## 二级标题
### 三级标题

**粗体** *斜体* `代码`

- 无序列表
1. 有序列表

[链接文字](URL)

> 引用文字

```代码语言
代码块
```

| 表格 | 列1 | 列2 |
|------|-----|-----|
| 行1  | 数据| 数据|
```

## 总结

按照这个流程，你就可以轻松地添加新文章到博客中。记住每次写完文章后都要推送到GitHub，这样网站就会自动更新！

# GitBook/HonKit PDF生成中图片不显示问题排查记录

## 1. 问题描述

在使用HonKit (GitBook的开源替代品)生成PDF文档时，发现PDF中所有图片均不显示，而HTML版本中图片能正常显示。

## 2. 环境信息

- 操作系统: Windows 10 on WSL (Ubuntu)
- Node版本: v20.17.0
- 项目: 企业架构实践文档(Enterprise Architecture Practices)
- 工具链: honkit + calibre的ebook-convert

## 3. 排查过程

### 3.1 初步环境检查

首先确认Node环境和相关工具是否正常安装：

```bash
node -v  # 确认Node版本为v20.17.0
npm list -g honkit  # 检查是否全局安装了honkit，未安装
npm list honkit  # 检查是否本地安装了honkit，未安装
```

### 3.2 尝试安装必要组件

```bash
# 全局安装honkit
npm install -g honkit  # 成功安装

# 本地项目安装honkit
npm install --save-dev honkit  # 成功安装

# 尝试安装PDF转换工具
npm install -g gitbook-pdf  # 失败，依赖问题
npm install -g honkit-pdf  # 失败，网络问题

# 安装ebook-convert工具
npm install --save-dev ebook-convert  # 成功安装
which ebook-convert  # 确认系统已安装calibre的ebook-convert
```

### 3.3 初次尝试生成PDF

```bash
npx honkit pdf . ea-practices.pdf  # 运行成功，但生成的PDF中无图片
```

### 3.4 检查项目配置与插件

检查book.json，发现使用了多个插件：
```json
"plugins": [
    "anchors", "ga", "github-buttons", "-sharing", "-lunr", 
    "-search", "-highlight", "search-plus", "splitter", 
    "atoc", "sectionx", "images-version", "image-captions"
]
```

尝试构建项目，发现插件问题：
```bash
npx honkit build  # 失败，提示"anchors"插件未找到
```

安装所需插件：
```bash
npm install gitbook-plugin-anchors gitbook-plugin-ga gitbook-plugin-github-buttons gitbook-plugin-search-plus gitbook-plugin-splitter gitbook-plugin-atoc gitbook-plugin-sectionx gitbook-plugin-images-version gitbook-plugin-image-captions
```

再次构建，发现image-captions插件兼容性问题：
```
Error: HonKit doesn't satisfy the requirements of this plugin: gitbook-plugin-image-captions require ^3
```

### 3.5 调整配置尝试修复

移除不兼容的image-captions插件：
```json
"plugins": [
    "anchors", "ga", "github-buttons", "-sharing", "-lunr", 
    "-search", "-highlight", "search-plus", "splitter", 
    "atoc", "sectionx", "images-version"
]
```

添加PDF配置：
```json
"pdf": {
    "paperSize": "a4",
    "margin": {
        "top": 56, "bottom": 56, "left": 56, "right": 56
    },
    "fontSize": 12,
    "fontFamily": "Arial",
    "headerTemplate": null,
    "footerTemplate": null
}
```

构建成功，但生成PDF仍无图片：
```bash
npx honkit build  # 成功
npx honkit pdf . ea-practices.pdf  # 成功，但PDF仍无图片
```

### 3.6 深入分析问题

检查图片目录确认图片存在：
```bash
ls -la images/  # 确认图片文件存在
```

检查Markdown文件中的图片引用方式：
```bash
grep -r "!\[.*\](.*images.*)" .  # 图片引用语法正确
```

### 3.7 关键突破：检查生成的HTML文件

分析HTML中的图片链接：
```bash
cat _book/introduction.html | grep -o 'src="[^"]*"' | head -5
```

发现问题：
```
src="images/Core-driver-of-enterprise-architecture-transformation.png?_=1747736872"
```

图片URL添加了时间戳查询参数（`?_=1747736872`），这导致PDF转换工具无法找到图片文件。

### 3.8 定位根本原因

查找添加时间戳的插件：
```bash
grep -r "images-version" .
```

确认`images-version`插件的作用是给图片URL添加时间戳以防止缓存，但这导致PDF生成工具将整个URL（包括查询参数）作为文件路径去查找。

### 3.9 实施解决方案

移除`images-version`插件：
```json
"plugins": [
    "anchors", "ga", "github-buttons", "-sharing", "-lunr", 
    "-search", "-highlight", "search-plus", "splitter", 
    "atoc", "sectionx"
]
```

清理旧的生成文件：
```bash
rm -rf _book ea-practices.epub
```

重新构建并生成PDF：
```bash
npx honkit build
cat _book/introduction.html | grep -o 'src="[^"]*"' | head -5  # 确认图片URL不再有时间戳
npx honkit pdf . ea-practices.pdf  # 成功生成带图片的PDF
```

## 4. 问题根因分析

根本原因是`images-version`插件与PDF生成过程的不兼容：

1. `images-version`插件为了防止浏览器缓存，在图片URL后添加时间戳查询参数
2. 在浏览器中，这种带查询参数的URL可以正常访问图片
3. 但PDF生成工具(calibre的ebook-convert)将整个URL当作文件路径处理，无法找到实际文件
4. 例如，工具会尝试查找`images/file.png?_=1747736872`这样的文件，而不是`images/file.png`

## 5. 解决方案

从`book.json`中移除`images-version`插件，确保图片URL不包含时间戳查询参数。

## 6. 反思与总结

### 排查中的不足之处

1. **未及时检查中间产物**：开始时只关注了最终PDF输出，没有检查生成的HTML文件内容
2. **过于复杂化解决路径**：尝试了安装不同工具而非分析现有工作流程
3. **缺少系统思维**：未完整理解从Markdown → HTML → PDF的转换流程及各环节可能的问题点
4. **方向不够聚焦**：排查过程中尝试了多个并行方向，而非逐步缩小问题范围

### 有效的排查方法

1. **检查中间产物**：分析HTML中的图片链接格式是关键突破点
2. **简化问题**：通过移除插件的方式验证假设
3. **命令行工具灵活运用**：使用grep等工具快速定位问题

## 7. 改进的问题排查框架

1. **分析完整流程链**：理解每个环节的输入/输出关系
2. **优先检查中间产物**：对于生成类问题，始终检查中间生成文件
3. **隔离变量法**：一次只修改一个因素，观察效果
4. **根因分析优先**：找到根本原因而非只解决症状
5. **实用命令组合**：掌握一些实用的调试命令，如`grep`、`diff`等

这次排查经验表明，解决技术问题不仅需要技术知识，更需要系统性思维和有效的排查方法论。 
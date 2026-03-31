# Hugo Book 部署说明

本项目已配置为使用 Hugo Book 主题并通过 GitHub Actions 自动部署到 GitHub Pages。

## 启用 GitHub Pages

要启用 GitHub Pages 部署，请按照以下步骤操作：

1. 前往仓库的 **Settings** 页面
2. 在左侧菜单中选择 **Pages**
3. 在 **Source** 部分，选择 **GitHub Actions** 作为部署源
4. 合并此 PR 到 `main` 分支后，GitHub Actions 将自动构建并部署网站

## 本地预览

如果你想在本地预览网站，需要安装 Hugo：

### 安装 Hugo

#### macOS
```bash
brew install hugo
```

#### Linux
```bash
# Ubuntu/Debian
sudo apt-get install hugo

# 或者下载最新版本
wget https://github.com/gohugoio/hugo/releases/download/v0.136.5/hugo_extended_0.136.5_linux-amd64.deb
sudo dpkg -i hugo_extended_0.136.5_linux-amd64.deb
```

#### Windows
```powershell
choco install hugo-extended
```

### 运行本地服务器

```bash
# 克隆仓库（包括子模块）
git clone --recurse-submodules https://github.com/JetSquirrel/claude-code-study.git
cd claude-code-study

# 如果已经克隆，但没有子模块，运行：
git submodule update --init --recursive

# 启动开发服务器
hugo server -D

# 网站将在 http://localhost:1313 上运行
```

## 项目结构

```
.
├── .github/
│   └── workflows/
│       └── hugo.yml           # GitHub Actions 部署工作流
├── content/
│   └── docs/                  # 文档内容
│       ├── _index.md          # 首页
│       ├── 01-导论与项目概览/
│       ├── 02-整体架构设计/
│       └── ...                # 其他章节
├── themes/
│   └── hugo-book/             # Hugo Book 主题（git submodule）
├── hugo.toml                  # Hugo 配置文件
└── .gitignore
```

## Hugo Book 主题特性

- 📱 响应式设计
- 🌓 自动深色/浅色主题切换
- 🔍 内置搜索功能
- 📑 自动生成目录
- 📝 Markdown 完全支持
- 🎨 代码高亮

## 自定义配置

所有配置都在 `hugo.toml` 文件中。主要配置项：

- `baseURL`: 网站的基础 URL（部署到 GitHub Pages 时会自动设置）
- `title`: 网站标题
- `params.BookTheme`: 主题模式（light/dark/auto）
- `params.BookSearch`: 是否启用搜索功能
- `params.BookToC`: 是否显示目录

## 部署后的网站地址

网站将部署在：`https://jetsquirrel.github.io/claude-code-study/`

## 故障排除

### 主题未加载

如果克隆后主题目录为空，运行：
```bash
git submodule update --init --recursive
```

### 构建失败

检查 GitHub Actions 日志以查看详细的错误信息。常见问题：
- 确保 GitHub Pages 已在仓库设置中启用
- 确保工作流有写入权限（Settings > Actions > General > Workflow permissions）

### 本地预览时样式不正常

确保使用 `hugo server` 命令而不是直接打开 HTML 文件。

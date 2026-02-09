# OpenCode 离线部署指南

本文档介绍如何在受限且被监控的网络环境中部署和运行 OpenCode，避免运行时从网络下载资源。

**版本**: 1.0.0  
**更新日期**: 2026-02-10

---

## 目录

- [概述](#概述)
- [需要预下载的资源](#需要预下载的资源)
- [LSP 服务器离线安装](#lsp-服务器离线安装)
- [Parser 查询文件配置](#parser-查询文件配置)
- [模型 API 配置](#模型-api-配置)
- [环境变量配置](#环境变量配置)
- [完整配置示例](#完整配置示例)

---

## 概述

OpenCode 在运行时会尝试从网络下载以下资源：

1. **LSP 语言服务器** - 按需自动下载
2. **Parser 查询文件** - 从 GitHub 下载 Tree-sitter 查询
3. **模型 API 调用** - 调用外部 AI 服务
4. **版本更新检查** - 检查新版本

在离线环境中，需要预先下载这些资源并配置本地源。

---

## 需要预下载的资源

### 1. LSP 服务器

| LSP 服务器 | 语言 | 下载来源 | 安装方式 |
|-----------|------|---------|---------|
| pyright | Python | npm registry | `bun install pyright` |
| jdtls | Java | Eclipse 官网 | curl 下载 tar.gz |
| typescript | TypeScript | npm registry | `bun install typescript-language-server` |
| vue | Vue | npm registry | `bun install @vue/language-server` |
| eslint | JavaScript | GitHub | fetch 下载 zip |
| gopls | Go | Go 代理 | `go install golang.org/x/tools/gopls@latest` |
| rust | Rust | GitHub Releases | fetch 下载 gzip |
| lua-ls | Lua | GitHub Releases | fetch 下载 tar.gz/zip |
| yaml | YAML | npm registry | `bun install yaml-language-server` |
| json | JSON | npm registry | `bun install vscode-json-languageserver` |
| dockerfile | Dockerfile | npm registry | `bun install dockerfile-language-server-nodejs` |
| bash | Bash | npm registry | `bun install bash-language-server` |

### 2. 额外 LSP 服务器（server.ts 中）

| LSP 服务器 | 下载来源 |
|-----------|---------|
| elixir-ls | `https://github.com/elixir-lsp/elixir-ls/archive/refs/heads/master.zip` |
| zls (Zig) | GitHub API: `api.github.com/repos/zigtools/zls/releases/latest` |
| clangd (C/C++) | GitHub API: `api.github.com/repos/clangd/clangd/releases/latest` |
| kotlin-ls | CDN: `https://download-cdn.jetbrains.com/kotlin-lsp/` |
| terraform-ls | GitHub API: `api.github.com/repos/hashicorp/terraform-ls/releases/latest` |
| texlab (LaTeX) | GitHub API: `api.github.com/repos/latex-lsp/texlab/releases/latest` |
| tinymist (Typst) | GitHub API: `api.github.com/repos/Myriad-Dreamin/tinymist/releases/latest` |

### 3. Parser 查询文件

从以下 URL 模式下载 Tree-sitter 查询文件：

```
https://raw.githubusercontent.com/nvim-treesitter/nvim-treesitter/refs/heads/master/queries/{language}/{type}.scm
```

涉及语言：Python、Rust、Go、C/C++、C#、Bash、Java、Ruby、PHP、Scala、JSON、YAML、Haskell、CSS、Julia、OCaml、Clojure、Swift、Nix

### 4. 配置文件 Schema

```
https://opencode.ai/config.json
```

---

## LSP 服务器离线安装

### 步骤 1：在可访问外网的机器上预下载

```bash
# 安装所有 LSP 服务器
opencode lsp-install all

# 或安装特定服务器
opencode lsp-install python typescript java
```

### 步骤 2：打包 LSP 目录

```bash
# Linux/macOS
tar -czf opencode-lsp-backup.tar.gz ~/.local/share/opencode/bin/

# Windows (PowerShell)
Compress-Archive -Path "$env:APPDATA\opencode\bin\*" -DestinationPath "opencode-lsp-backup.zip"
```

### 步骤 3：复制到内网机器

将打包文件复制到内网机器并解压到对应目录：

| 操作系统 | 目标路径 |
|---------|---------|
| Linux | `~/.local/share/opencode/bin/` |
| macOS | `~/Library/Application Support/opencode/bin/` |
| Windows | `%APPDATA%\opencode\bin\` |

---

## Parser 查询文件配置

### 步骤 1：下载查询文件

在可访问外网的机器上执行：

```bash
# 创建下载目录
mkdir -p opencode-parsers/queries

# 下载各语言的查询文件
curl -o opencode-parsers/queries/python-highlights.scm \
  https://raw.githubusercontent.com/nvim-treesitter/nvim-treesitter/refs/heads/master/queries/python/highlights.scm

curl -o opencode-parsers/queries/python-locals.scm \
  https://raw.githubusercontent.com/nvim-treesitter/nvim-treesitter/refs/heads/master/queries/python/locals.scm

# 重复上述命令下载其他语言的查询文件...
```

### 步骤 2：修改 parsers-config.ts

将 [parsers-config.ts](../parsers-config.ts) 中的 URL 替换为本地路径：

```typescript
// 修改前
highlights: [
  "https://raw.githubusercontent.com/nvim-treesitter/nvim-treesitter/refs/heads/master/queries/python/highlights.scm",
],

// 修改后
highlights: [
  "file:///path/to/opencode-parsers/queries/python-highlights.scm",
],
```

---

## 模型 API 配置

### 使用本地模型服务

在 `opencode.json` 中配置本地 OpenAI 兼容 API：

```json
{
  "provider": {
    "local": {
      "api": "http://your-internal-llm-server:8080/v1",
      "key": "your-api-key"
    }
  },
  "model": "local:your-model-name"
}
```

### 配置本地模型列表

创建本地 `models.json` 文件：

```json
{
  "local-llm": {
    "id": "local-llm",
    "name": "Local LLM",
    "api": "http://your-internal-llm-server:8080/v1",
    "doc": "http://internal-docs/llm"
  }
}
```

---

## 环境变量配置

### 禁用自动下载

```bash
# 禁用所有 LSP 自动下载
export OPENCODE_DISABLE_LSP_DOWNLOAD=true

# 或使用数字
export OPENCODE_DISABLE_LSP_DOWNLOAD=1
```

### 配置本地 npm Registry

```bash
# 配置 Bun 使用内部 npm 镜像
export BUN_CONFIG_REGISTRY=http://your-internal-npm-registry

# 或使用 npm 配置
export NPM_CONFIG_REGISTRY=http://your-internal-npm-registry
```

### 配置 LSP 存储路径（可选）

```bash
# 自定义 LSP 服务器存储位置
export OPENCODE_TEST_HOME=/path/to/lsp-storage
```

---

## 完整配置示例

### opencode.json（离线环境配置）

```json
{
  "$schema": "./config.json",
  "lsp": {
    "pyright": {
      "disabled": false
    },
    "typescript": {
      "disabled": false
    },
    "jdtls": {
      "disabled": false
    },
    "eslint": {
      "disabled": true
    },
    "rust": {
      "disabled": false
    },
    "lua-ls": {
      "disabled": false
    }
  },
  "provider": {
    "local": {
      "api": "http://internal-llm-server:8080/v1",
      "key": "{env:LOCAL_LLM_API_KEY}"
    }
  },
  "model": "local:qwen2.5-coder"
}
```

### 启动脚本（offline-start.sh）

```bash
#!/bin/bash

# 离线环境启动脚本

# 1. 禁用 LSP 自动下载
export OPENCODE_DISABLE_LSP_DOWNLOAD=true

# 2. 配置本地 npm registry（如需安装新 LSP）
export BUN_CONFIG_REGISTRY=http://internal-npm-mirror:4873

# 3. 配置本地模型 API
export LOCAL_LLM_API_KEY="your-internal-key"

# 4. 启动 OpenCode
opencode "$@"
```

### Windows 启动脚本（offline-start.ps1）

```powershell
# 离线环境启动脚本

# 1. 禁用 LSP 自动下载
$env:OPENCODE_DISABLE_LSP_DOWNLOAD = "true"

# 2. 配置本地 npm registry
$env:BUN_CONFIG_REGISTRY = "http://internal-npm-mirror:4873"

# 3. 配置本地模型 API
$env:LOCAL_LLM_API_KEY = "your-internal-key"

# 4. 启动 OpenCode
opencode $args
```

---

## 故障排除

### LSP 服务器无法启动

1. 检查 LSP 服务器是否已预下载
   ```bash
   opencode lsp-install --check
   ```

2. 检查 LSP 目录权限
   ```bash
   ls -la ~/.local/share/opencode/bin/
   ```

3. 手动指定 LSP 路径
   ```json
   {
     "lsp": {
       "pyright": {
         "command": ["/path/to/pyright", "--stdio"]
       }
     }
   }
   ```

### 模型 API 连接失败

1. 检查网络连通性
   ```bash
   curl http://internal-llm-server:8080/v1/models
   ```

2. 检查 API 密钥配置
3. 查看 OpenCode 日志
   ```bash
   opencode --print-logs --log-level DEBUG
   ```

---

## 相关文档

- [LSP 配置指南](./LSP-CONFIG.zh-CN.md)
- [OpenCode 官方文档](https://opencode.ai/docs)

---

**注意**: 本文档基于 OpenCode v1.1.53 版本编写，后续版本可能会有变化。

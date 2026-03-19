# Octos Hub

[English](README.md) | [中文](README_CN.md)

[Octos](https://github.com/octos-org/octos) 社区技能注册中心。浏览、安装和发布技能，扩展 AI 智能体的能力。

## 快速上手

```bash
# 浏览所有可用技能
octos skills search

# 按关键词搜索
octos skills search slides

# 安装技能包（仓库中的所有技能）
octos skills install mofa-org/mofa-skills

# 从多技能仓库中安装单个技能
octos skills install mofa-org/mofa-skills/mofa-cards

# 查看已安装技能
octos skills list

# 移除技能
octos skills remove mofa-cards
```

## 技能工作原理

技能是一个包含 `SKILL.md` 文件的目录，用于教会智能体新的行为。技能可以选择性地包含可执行工具（二进制或脚本），供智能体调用。

```
my-skill/
  SKILL.md            # 必需：指令 + 元数据
  manifest.json       # 可选：工具定义 + 二进制分发
  Cargo.toml          # 可选：Rust 源码（安装时自动编译）
  package.json        # 可选：Node.js（安装时自动 npm install）
  references/         # 可选：参考文档，智能体可按需读取
```

### SKILL.md

每个技能需要一个带 YAML 前置元数据的 `SKILL.md`：

```markdown
---
name: my-skill
description: 在技能列表中显示的简短描述
version: 1.0.0
author: your-name
always: false
requires_bins: docker,ffmpeg
requires_env: API_KEY
---

# 我的技能

给智能体的指令。像给同事做简报一样来写：
- 这个技能做什么
- 什么时候使用
- 分步骤的使用方式
- 带预期输出的示例
```

#### 前置元数据字段

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `name` | 是 | — | 小写标识符，用连字符（如 `deep-search`） |
| `description` | 是 | — | 一行描述，显示在 `octos skills list` 中 |
| `version` | 否 | — | 语义化版本号，用于更新追踪 |
| `author` | 否 | — | 作者名或组织 |
| `always` | 否 | `false` | `true` = 始终包含在系统提示中。谨慎使用。 |
| `requires_bins` | 否 | — | 逗号分隔的 PATH 中必须存在的可执行文件 |
| `requires_env` | 否 | — | 逗号分隔的必须设置的环境变量 |

### manifest.json（工具声明）

如果技能提供可执行工具，在 `manifest.json` 中声明：

```json
{
  "name": "my-skill",
  "version": "1.0.0",
  "tools": [
    {
      "name": "my_tool",
      "description": "工具的功能描述（展示给 LLM）",
      "input_schema": {
        "type": "object",
        "properties": {
          "query": { "type": "string", "description": "搜索查询" },
          "limit": { "type": "integer", "description": "最大结果数", "default": 10 }
        },
        "required": ["query"]
      }
    }
  ]
}
```

#### 工具 I/O 协议

工具通过 **stdin** 接收 JSON，必须通过 **stdout** 输出 JSON：

```
stdin:  {"query": "rust async", "limit": 5}
stdout: {"output": "查询结果...", "success": true}
```

退出码 0 = 成功，非零 = 错误。`output` 字段返回给智能体。

#### 工具可执行文件查找顺序

插件加载器按以下顺序查找可执行文件：
1. `<skill-dir>/main` — 预编译二进制（首选）
2. `<skill-dir>/<skill-name>` — 命名二进制
3. `<skill-dir>/index.js` — Node.js 脚本（通过 `node` 运行）

### 预编译二进制分发

为了加速安装（跳过编译），在 manifest 中发布平台二进制：

```json
{
  "name": "my-skill",
  "version": "1.0.0",
  "timeout_secs": 300,
  "binaries": {
    "darwin-aarch64": {
      "url": "https://github.com/you/repo/releases/download/v1.0.0/my-skill-darwin-aarch64.tar.gz",
      "sha256": "abc123..."
    },
    "darwin-x86_64": {
      "url": "https://github.com/you/repo/releases/download/v1.0.0/my-skill-darwin-x86_64.tar.gz",
      "sha256": "def456..."
    },
    "linux-x86_64": {
      "url": "https://github.com/you/repo/releases/download/v1.0.0/my-skill-linux-x86_64.tar.gz",
      "sha256": "789ghi..."
    }
  },
  "tools": [ ... ]
}
```

**平台标识**：`darwin-aarch64`（Apple Silicon）、`darwin-x86_64`（Intel Mac）、`linux-x86_64`、`linux-aarch64`。

安装器下载匹配的二进制文件，验证 SHA-256 哈希后解压。如果当前平台没有预编译二进制，则回退到源码编译。

## 用 Rust 编写工具

```rust
// src/main.rs
use serde::{Deserialize, Serialize};
use std::io::Read;

#[derive(Deserialize)]
struct Input {
    query: String,
    #[serde(default = "default_limit")]
    limit: usize,
}
fn default_limit() -> usize { 10 }

#[derive(Serialize)]
struct Output {
    output: String,
    success: bool,
}

fn main() {
    let mut buf = String::new();
    std::io::stdin().read_to_string(&mut buf).unwrap();
    let input: Input = serde_json::from_str(&buf).unwrap();

    let result = format!("找到 {} 条关于 '{}' 的结果", input.limit, input.query);

    let output = Output { output: result, success: true };
    println!("{}", serde_json::to_string(&output).unwrap());
}
```

在 `Cargo.toml` 中添加 `serde` 和 `serde_json` 依赖。如果没有预编译二进制，安装器会自动运行 `cargo build --release`。

## 用 Node.js 编写工具

```javascript
// index.js
const input = JSON.parse(require('fs').readFileSync('/dev/stdin', 'utf8'));

const result = `查询结果: ${input.query}`;
console.log(JSON.stringify({ output: result, success: true }));
```

添加 `package.json` 声明依赖。安装器会自动运行 `npm install`。

## 多技能仓库

单个仓库可以包含多个技能，作为顶层目录：

```
my-skills/
  skill-a/
    SKILL.md
  skill-b/
    SKILL.md
    manifest.json
    Cargo.toml
    src/main.rs
```

用户可以安装全部或选择其一：

```bash
octos skills install you/my-skills          # 安装 skill-a + skill-b
octos skills install you/my-skills/skill-b  # 仅安装 skill-b
```

## 发布到注册中心

### 第一步：创建技能仓库

按照上述结构将技能推送到公开的 GitHub 仓库。

### 第二步：本地测试

```bash
# 从你的仓库安装
octos skills install your-user/your-repo

# 验证是否正常工作
octos skills list
```

### 第三步：向本仓库提交 PR

Fork [octos-hub](https://github.com/octos-org/octos-hub)，在 `registry.json` 中添加你的条目：

```json
{
  "name": "my-skills",
  "description": "你的技能包简短描述",
  "repo": "your-user/your-repo",
  "skills": ["skill-a", "skill-b"],
  "requires": ["git", "node"],
  "tags": ["关键词1", "关键词2"]
}
```

### 注册条目字段

| 字段 | 必填 | 说明 |
|------|------|------|
| `name` | 是 | 唯一包名（小写、连字符） |
| `description` | 是 | 一行描述（显示在搜索结果中） |
| `repo` | 是 | GitHub 路径 `user/repo` |
| `skills` | 否 | 包中各技能的名称列表 |
| `requires` | 否 | 所需外部工具（如 `git`、`node`、`cargo`、`python`） |
| `tags` | 否 | 可搜索关键词（用于 `octos skills search`） |

### 第四步：审核

注册团队会审核你的 PR：
- `SKILL.md` 格式正确，包含必填前置元数据
- 工具可执行文件正常工作（如有）
- 无恶意代码或过多依赖
- 描述和标签准确

## 技能加载

技能按优先级顺序加载（同名时先到先得）：
1. 配置文件专属技能（`~/.octos/profiles/<profile>/skills/`）
2. 全局技能（`~/.octos/skills/`）
3. 内置技能（cron、skill-store、skill-creator）

`always: true` 的技能会注入每次系统提示中。其他技能出现在技能索引中——智能体按需读取。

## 聊天中管理技能

在 octos 网关聊天会话中：

```
/skills                       # 查看已安装技能
/skills install user/repo     # 从 GitHub 安装
/skills remove my-skill       # 移除技能
```

智能体也可以通过内置的 `skill-store` 工具以编程方式管理技能。

## 最佳实践

1. **SKILL.md 控制在 200 行以内** — 智能体会将其读入上下文
2. **包含具体示例**，而非抽象理论
3. **使用 `requires_bins`** 在缺少外部工具时快速失败
4. **谨慎设置 `always: true`** — 会增加每次提示的 token 消耗
5. **工具应当独立运行** — 工具内部不调用 LLM，让智能体负责推理
6. **发布预编译二进制** 以加速跨平台安装
7. **使用 SHA-256 校验** 验证所有二进制下载
8. **为技能标注版本号** 以便追踪更新

## 示例

| 包名 | 仓库 | 技能 | 描述 |
|------|------|------|------|
| mofa-skills | [mofa-org/mofa-skills](https://github.com/mofa-org/mofa-skills) | slides, cards, comic, infographic, research, fm, news, github, workflow | AI 内容创作套件 |

## 链接

- [Octos](https://github.com/octos-org/octos) — AI 智能体框架
- [技能创建指南](https://github.com/octos-org/octos/blob/main/crates/octos-agent/skills/skill-creator/SKILL.md) — 内置的技能创建工具

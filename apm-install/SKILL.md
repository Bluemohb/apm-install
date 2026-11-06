---
name: apm-install
slug: apm-install
displayName: APM 落点规则
version: 1.1.0
summary: "APM 装到哪、给谁生效：作用域、target 语义、按 targets 把各客户端 skills 目录收敛为指向 ~/.agents/skills 的 junction，以及 WorkBuddy 宿主落点与 MCP 合并"
description: "本技能应在回答「装到哪、给哪个客户端生效、WorkBuddy 怎么读得到」时使用：APM 落点与作用域规则（WorkBuddy 宿主专用），决定装到全局(-g)还是项目级、-t/--target 的真实语义；安装前必须先读 ~/.apm/apm.yml 的 targets:，若存在多个会各自部署 skills 的 target，提醒用户先把这些 Agent skills 目录换成指向 ~/.agents/skills 的 junction（或把 targets 收敛为 [agent-skills]）；以及 skills 目录收敛关系、WorkBuddy 宿主落点、MCP 服务器为何不能复制只能合并配置。命令、flag、鉴权、治理、排错等细节一律查 apm-usage 技能，不在本技能重复。"
agent_created: true
---

# APM 落点规则

本技能只回答一件事：**装到哪、给谁生效**。命令与参数细节查 `apm-usage` 技能。

> **核心原则（2026-11-06 修订）**：`~/.agents/skills/` 是**唯一真实来源**
> （single source of truth）。APM 对每个 target 是**独立物理拷贝，不是链接**
> （inode 不同、`apm.lock.yaml` 分别登记）。因此 `apm install` **之前**应先按
> `~/.apm/apm.yml` 的 `targets:` 把各客户端的 Agent skills 目录收敛成指向
> `~/.agents/skills` 的 junction，避免多份副本、`apm update` 双写、
> 以及清理动作穿透 junction 误删。详见第 2 步「安装前必做」。

## 前置检查（每次触发先做，按序判定，命中即停）

1. `~/.workbuddy-ai/skills/apm-usage/` 存在 → 正常执行；细节读其
   `commands.md` / `dependencies.md` / `governance.md`。
2. 不存在，但 `apm --version` 可用 → 执行
   `apm install -g microsoft/apm/packages/apm-guide`。
   - 若 `~/.workbuddy-ai/skills` 已是指向 `~/.agents/skills` 的 junction，
     `apm-guide` 落到 `~/.agents/skills/apm-usage/` 后自动可见，**无需再复制**，
     只需确认目录可读。
   - 若仍是独立真实目录，才把整个 `apm-guide` 目录复制到
     `~/.workbuddy-ai/skills/apm-usage/`，核对文件数与源目录一致后继续。
3. `apm` 命令不存在 → 明确告知用户「本技能依赖 APM CLI 与 apm-guide 包，
   当前环境未检测到 apm」，给出首次安装指引（`apm self-update` 只负责升级，
   首次安装见 <https://github.com/microsoft/apm>），然后**只用本文档
   「命令速查」一节的命令**完成任务，不得调用 apm-usage，也不得臆造其内容。

安装动作一律走 APM：禁止 `npx skills`，禁止 git clone 后手工拷贝
（第 3 步的 WorkBuddy 落点复制除外）。

## 第 1 步：先定作用域

| 目标 | 命令 | 写入 |
|---|---|---|
| Agent 软件本身，所有项目共享 | `apm install -g <pkg>` | `~/.apm/apm.yml` |
| 某个仓库的配置，要随项目提交 | `cd <项目> && apm install <pkg>` | `<项目>/apm.yml` |

`-g` 只写 `~/.apm/`，不会污染任何项目；反之项目级也不会影响其他仓库。

## 第 2 步：target 决定落点

`agent-skills` 是跨客户端共享 skills 的 target，但它**永不自动探测、不在 `all` 里**，
必须显式指定。项目内缺少对应标记文件时，该 target 状态为 `inactive`，不会被部署；
若一个可用 target 都解析不出来，`apm install` 会直接中止（exit 2）——这是最常见的失败点。

`apm targets`（0.34.0 实测，项目级）会直接列出所需标记与部署目录：

| TARGET | 需要的标记 | 部署目录 |
|---|---|---|
| claude | `CLAUDE.md` | `.claude/` |
| copilot | `.github/copilot-instructions.md` | `.github/` |
| cursor | `.cursor/` | `.cursor/` |
| codex | `.codex/` | `.codex/` |
| gemini | `GEMINI.md` | `.gemini/` |
| grok-build | `.grok/` | `.grok/` |
| opencode | `.opencode/` | `.opencode/` |
| windsurf | `.windsurf/` | `.windsurf/` |
| kiro | `.kiro/` | `.kiro/` |

`apm targets` 只列项目级可探测 target。`agent-skills` 与
`copilot-app` / `copilot-cowork` / `grok-bot` / `grok-cloud` / `hermes` /
`openclaw` / `antigravity` 不在其中——它们属于用户级支持矩阵，
见 `apm install -g` 的提示输出。

**`-t/--target` 的真实语义（易误判）**：它是**替换本次解析结果**，既不是"收窄"也不是
并集，且**不写回** `apm.yml` 的 `targets:`。实测（0.34.0）：清单为
`targets: [agent-skills, claude]` 时执行 `-t cursor`，解析结果只有 `cursor`，
claude 本次完全不部署。

→ 清单里已声明 `targets:` 时**不要加 `-t`**；要改目标就改清单，别在命令行临时覆盖。

skills 落点收敛关系：

| 客户端 | skills 目录 |
|---|---|
| agent-skills / copilot / cursor / codex / gemini / opencode / windsurf / antigravity | `~/.agents/skills/`（项目级 `.agents/skills/`） |
| claude | `~/.claude/skills/`（受 `CLAUDE_CONFIG_DIR` 影响） |
| grok-build | `~/.grok/skills/` |
| kiro | `.kiro/skills/` |
| opencode（用户级） | `~/.config/opencode/skills/` |
| hermes（用户级） | `$HERMES_HOME/skills/` |

一次 `agent-skills` 就同时喂饱 Cursor / Codex / Gemini / Copilot / Windsurf / Antigravity；
`claude` 等不收敛到 `~/.agents/skills/` 的客户端，**不要靠往 `targets:` 里加 target 解决**
（那只会多出一份物理拷贝），按下一节换成 junction 或把 `targets:` 收敛为 `[agent-skills]`。

### 安装前必做：按 `targets:` 把 Agent skills 目录收敛成 junction

这一步在 `apm install` **之前**完成。先读 `~/.apm/apm.yml` 的 `targets:`，逐项判定：

| `targets:` 里的 target | APM 会写入的 skills 目录 | 应做的处理 |
|---|---|---|
| `agent-skills` | `~/.agents/skills/` | **保持真实目录**，作为唯一真实来源 |
| `claude` | `~/.claude/skills/` | 换成 junction → `~/.agents/skills` |
| `grok-build` | `~/.grok/skills/` | 换成 junction |
| `kiro`（项目级） | `<项目>/.kiro/skills/` | 换成 junction |
| opencode / hermes 等用户级 | 各自独立目录 | 换成 junction |
| WorkBuddy（非 APM target） | `~/.workbuddy-ai/skills/` | 换成 junction |

→ 只要 `targets:` 里出现 `agent-skills` 之外、带独立 skills 目录的 target，
**必须先提醒用户，不要直接装**。可直接使用：

> **注意**：你的 `~/.apm/apm.yml` 里 `targets: [agent-skills, claude]`，APM 会再向
> `~/.claude/skills/` 部署一份物理拷贝。建议先把该目录换成指向
> `~/.agents/skills` 的 junction，让所有客户端共享同一份，避免 `apm update` 双写、
> `apm.lock.yaml` 重复登记，以及清理时穿透 junction 误删。要我现在建吗？

两种等效收敛方式（选一即可）：

1. **建 junction，保留多 target**：各客户端 skills 目录都指向 `~/.agents/skills`。
2. **改 `apm.yml` 把 `targets:` 收敛为 `[agent-skills]`（推荐）**：APM 只部署一份，
   客户端通过 junction 读到。改完要同步清理 `apm.lock.yaml` 里对应 target 的
   `deployed_files` / `deployed_file_hashes` / `deployments` 记录，再用
   `apm install -g --dry-run --frozen` 验证 lock 与清单同步、`apm audit` 无新增漂移。

Windows 建 junction（无需管理员权限；等价于 `mklink /J`）：

```bash
# 前置：junction 要求链接路径不存在，先把原目录重命名保留为回滚点
mv ~/.claude/skills ~/.claude/skills.pre-junction-$(date +%Y%m%d-%H%M)
# 建链接（用 homedir 拼绝对路径，跨用户通用）
node -e "const fs=require('fs'),os=require('os'),p=require('path');fs.symlinkSync(p.join(os.homedir(),'.agents/skills'),p.join(os.homedir(),'.claude/skills'),'junction')"
# 验证：lstatSync(link).isSymbolicLink() 为 true，且两侧逐文件哈希一致
```

> 已收敛过的环境（`~/.agents/skills` 为唯一真实目录，其余客户端目录均为 junction，
> `targets: [agent-skills]`）**跳过本步**，直接安装即可。

## 第 3 步：WorkBuddy 宿主（APM 无此 target）

**已按第 2 步建立整体 junction（推荐路径）**：

- **不需要任何复制**。`~/.workbuddy-ai/skills` 就是 `~/.agents/skills`，
  `apm install -g <pkg>` 落到 `~/.agents/skills/<name>/` 后 WorkBuddy 立即可见。
- 装完核对：链接仍有效（`lstatSync().isSymbolicLink()` 为 true）、技能目录可读、
  `cd ~/.apm && apm audit` 无新增漂移。
- 注意：WorkBuddy「设置 → 已安装」列表只列带安装记录的用户技能，不是目录扫描结果，
  junction 带进来的技能不会出现在该列表里；这不代表不可用，以文件系统可读性为准。

**未建 junction 时的降级做法（逐目录复制）**：

1. `apm install -g <pkg>` → 落到 `~/.agents/skills/<name>/`
2. 复制**整个技能目录**到 `~/.workbuddy-ai/skills/<name>/`
   （项目级则 `<项目>/.workbuddy-ai/skills/<name>/`）
   - 只复制 `SKILL.md` 无效：apm-usage 的知识分布在 8 个参考 md 里
     （authentication / commands / dependencies / governance / installation /
     package-authoring / troubleshooting / workflow）
3. 复制必须幂等：目标已存在先 `diff`，不静默覆盖用户改动；复制后核对文件数与源目录
   一致，并向用户报告实际落点和文件数

该副本不在 `apm.lock.yaml` 的 `deployed_files` 里，`apm update` **不会**同步它 ——
每次更新都要重新复制，这正是推荐改用整体 junction 的原因。

## 第 4 步：MCP 服务器另走一路

MCP 不进 skills 目录，用 `--mcp` 写入各运行时的配置文件：

```bash
apm install --mcp io.github.github/github-mcp-server          # registry
apm install --mcp filesystem -- npx -y @modelcontextprotocol/server-filesystem  # stdio，-- 之后是启动命令
apm install --mcp linear --transport http --url https://mcp.linear.app/sse       # remote
```

WorkBuddy 没有 MCP target：APM 装到目标运行时后，**必须手动把对应条目合并进
WorkBuddy 的 MCP 配置**，不能像 skills 那样复制目录。

## 命令速查（无 apm-usage 时的降级来源）

| 目的 | 命令 |
|---|---|
| 装用户级 / 项目级 | `apm install -g <pkg>` / `cd <项目> && apm install <pkg>` |
| 预览改动不落盘 | `apm install -g --dry-run <pkg>` |
| 看 target 解析结果 | `apm targets` |
| 读当前 targets | `grep -A3 '^targets:' ~/.apm/apm.yml` |
| 验证 lock 与清单同步 | `apm install -g --dry-run --frozen` |
| 看已装依赖 | `apm deps list -g`（`--all` 含项目级） |
| 看上游是否漂移 | `apm outdated` |
| 刷新依赖并重写锁 | `apm update` |
| 漂移与锁一致性 | `cd ~/.apm && apm audit`（CI 用 `apm audit --ci`） |
| 卸载 / 清理未声明 | `apm uninstall <pkg>` / `apm prune` |
| 查包信息与版本 | `apm view <pkg>` |
| 升级 CLI 自身 | `apm self-update` |

## 硬约束

1. `--force`、`--trust-bin`、`--trust-transitive-mcp`、`--allow-insecure`、
   `--no-policy` 会绕过安全闸门，使用前必须征得用户同意（`--force` 同时绕过 critical 扫描阻断）。
2. 装不上就如实报错并说明卡在哪一步（target 为 `inactive` 时先查 `apm targets`），
   不得静默降级为手工复制；确需手工复制时按第 3 步复制整个目录。
3. 装完核对：`apm targets`（实际生效目标）、`apm deps list -g`（用户级依赖）、
   `cd ~/.apm && apm audit`（漂移与锁一致性；`audit` 没有 `-g`，靠工作目录区分作用域）。
4. 本文档的 target 列表与命令锚定 apm 0.34.0。执行前若 `apm --version` 与之不符，
   以 `apm targets` / `apm install --help` 的实际输出为准，并提示用户文档可能已过期。
5. 安装前必须先读 `~/.apm/apm.yml` 的 `targets:` 并按第 2 步「安装前必做」判定；
   存在多个独立 skills 落点时，先提醒用户收敛为 junction（或把 `targets:` 收敛为
   `[agent-skills]`）再装，不得静默让 APM 部署多份物理副本。已收敛的环境跳过。

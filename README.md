# apm-install

> APM 落点规则 —— 回答「装到哪、给谁生效、WorkBuddy 怎么读得到」。

一个 Agent Skill，用于 [APM (Agent Package Manager)](https://github.com/microsoft/apm) 安装技能时的**落点治理**：决定装到全局还是项目级、`--target` 的真实语义，以及安装前如何把各个客户端的 skills 目录收敛为单一真实来源。

命令、flag、鉴权、治理、排错等细节由配套的 `apm-usage` 技能负责，本技能只讲落点。

## 核心原则

`~/.agents/skills/` 是**唯一真实来源**（single source of truth）。APM 对每个 target 是**独立物理拷贝，不是链接**（inode 不同、`apm.lock.yaml` 分别登记）。

因此：

- 安装前先读 `~/.apm/apm.yml` 的 `targets:`；
- 若存在多个会各自部署 skills 的 target（例如 `[agent-skills, claude]`），提醒用户先把各客户端的 skills 目录换成指向 `~/.agents/skills` 的 junction，或把 `targets:` 收敛为 `[agent-skills]`；
- 已建立整体 junction 的环境，WorkBuddy 宿主不再需要额外复制。

这样做可避免多份物理副本、`apm update` 双写、`apm.lock.yaml` 重复登记，以及清理动作穿透 junction 误删真实文件。

## 安装

```bash
# 通过 skills CLI
npx skills add Bluemohb/apm-install

# 通过 SkillHub
skillhub install apm-install --namespace user_91daf24e
```

或手动复制：把 `apm-install/` 整个目录放到你的用户级 skills 目录下
（WorkBuddy 为 `~/.workbuddy-ai/skills/`，Claude Code 为 `~/.claude/skills/`，
agentskills.io 标准为 `~/.agents/skills/`）。

## 版本

- **1.1.0** — 新增按 `targets:` 判定并提醒先把各 Agent skills 目录收敛为指向 `~/.agents/skills` 的 junction；明确整体 junction 后 WorkBuddy 宿主无需再复制；补充目标目录判定表、提醒话术、Windows junction 命令、已收敛环境跳过规则与硬约束。
- **1.0.0** — 初版：作用域、`-t/--target` 语义、skills 目录收敛关系、WorkBuddy 宿主落点与 MCP 配置合并规则。

## 仓库结构

```
apm-install/
└── SKILL.md              # 技能定义（frontmatter + 规则正文）
.github/
├── validate_skills.py    # 校验 <skill>/SKILL.md 结构
└── workflows/
    └── validate.yml
.gitignore
LICENSE
README.md
```

## 许可

MIT

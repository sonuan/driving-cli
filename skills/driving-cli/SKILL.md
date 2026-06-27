---
name: driving-cli
description: driving-cli 工具使用指南，管理 AI Coding 规范仓库、框架文档、技能、规则、需求目录和 agent。触发场景：(1) 提到 driving 命令、driving-cli、driving.config.json；(2) 需要安装/卸载/更新规范仓库；(3) 需要管理框架文档或按分类发现框架（framework install/list/pull/load --category/categories）；(4) 需要管理技能（skill list/load）；(5) 需要管理规则（rule list/load）；(6) 需要查询需求功能（feature list/migrate）；(7) 需要管理 agent（agent list/load/memory）；(8) 需要执行 driving update 更新工具；(9) 询问如何配置 driving.config.json；(10) 需要初始化或同步 AI Coding 规范仓库；(11) 需要查看或加载 refine 提案（refine list/load）；(12) 需要查看或读取门禁规则（gate list/load）；(13) 需要管理 Power 模式配置（power install/uninstall/list/pull）或 driving.power.json。
---

# driving-cli 工具使用指南

`driving` 是一个 Python CLI 工具，用于管理 AI Coding 规范仓库、框架文档、技能、规则和需求目录。

运行方式：`python3 -m driving.cli <command>` 或安装后直接使用 `driving <command>`。

---

## 项目结构

```
<project-root>/
├── driving.config.json          # 全局配置文件
└── ai-driving/
    └── <repo-name>/             # 每个已安装的仓库
        ├── manifest.json        # 仓库元信息（可选），支持 min_cli_version、system_prompt、skills、rules、agents、gates 字段
        ├── frameworks/
        │   ├── gitlist.json     # 框架列表配置
        │   └── <framework>/     # 各框架文档目录
        ├── skills/              # 技能列表
        │   └── <skill>/
        │       └── SKILL.md
        ├── rules/               # 规则列表
        │   └── <rule>.md
        ├── features/            # 需求功能
        │   └── <feature>/
        │       └── FEATURE.md
        ├── agents/              # agent 定义
        │   └── <agent>/
        │       ├── AGENTS.md    # 指令/系统提示（必填）
        │       ├── SOUL.md      # 人格与行为风格（可选）
        │       └── MEMORY.md    # 最佳实践知识沉淀（可选）
        ├── refines/           # 规范变更提案（由 self-refine 技能写入，需 owner 审批合并）
        │   └── YYYY-MM-DD-<type>-<name>-<brief>.md
        └── REFINE_LOG.md        # 规范进化变更日志
```

---

## driving.config.json 结构

```json
{
  "version": "2",
  "repos": [
    {
      "name": "driving",
      "type": "remote",
      "url": "https://github.com/your-org/driving",
      "path": "ai-driving/driving",
      "tags": ["base"],
      "skills": { "enabled": [], "disabled": ["some-skill"] },
      "rules": { "enabled": [], "disabled": [] },
      "agents": { "enabled": [], "disabled": [] },
      "check_sample_rate": 100
    },
    {
      "name": "f-message",
      "type": "local",
      "path": "ai-driving/f-message",
      "tags": []
    }
  ],
  "default_commit_message": "update by driving",
  "update_version_url": "",
  "check_sample_rate": 100
}
```

- `tags` 含 `"base"` 的仓库在无关键词时默认加载；传入关键词时忽略 tags，repo.name 精确匹配（不区分大小写）或 name/description 模糊匹配（子串包含即命中）
- `skills.enabled` 非空时为白名单模式，只加载列表中的技能
- `skills.disabled` 非空时为黑名单模式，排除列表中的技能
- `rules`、`agents` 字段同理
- `check_sample_rate`（全局）：`load` 时的更新检测采样率，默认 `100`（每次都检测）
- `check_sample_rate`（仓库级）：覆盖全局配置，优先级更高
  - `0`：永不检测该仓库更新
  - `1~100`：按概率采样，每次 `load` 随机决定是否检测
  - `-1`：始终检测，检测到更新时自动执行 `repo pull`（静默，不打扰用户）
- `gate_webhook`：门禁结果（pass / auto_pass / amend / blocked）上报地址，未配置时不上报
- `agent_webhook`：通用操作记录上报地址，覆盖所有关键操作（`load_invoked`、`load_auto_updated`、`update_completed`、`repo_pulled`、`power_pulled`、`agent_started`、`refine_committed`、`refine_merged`、`refine_signal`）；payload 含 `operation`、`description`、`triggered_at`、`cli_version`、`actor`、`branch`、`extra`；未配置时不上报

**skills 过滤优先级：** `driving.config.json` > `manifest.json` > 全量加载
- `manifest.json` 可在仓库内声明默认开启的技能，团队 pull 后自动生效，无需手动修改 `driving.config.json`
- `driving.config.json` 有 `skills` 配置时，`manifest.json` 的配置被忽略

**manifest.json 示例（仓库级默认配置）：**
```json
{
  "min_cli_version": "1.0.7",
  "system_prompt": "prompts/system_prompt.md",
  "skills": {
    "enabled": ["android-page"],
    "disabled": []
  }
}
```

---

## load — 一次性加载所有上下文

> 在 AGENTS.md 作为强制前置调用

```bash
driving load                         # 输出 skills、rules、repos 等，供 AI 会话注入
driving load <repo-name>             # 只加载指定仓库的 skills/rules（repos 始终全量）
driving load <repo-name> <repo-name> # 同时加载多个仓库（空格或逗号分隔均可）
driving load <name>,<name>           # 逗号分隔写法
driving load --with framework        # 附带框架文档（关键词同样生效）
driving load --with agent            # 附带 agent 列表（关键词同样生效）
driving load --with framework,agent  # 同时附带框架和 agent
driving load --platform <platform>   # 指定开发平台（android/iOS/harmony/kuikly）
driving load --debug                 # 同上，同时输出调试日志
```

`--platform` 可用值：`android`、`iOS`、`harmony`、`kuikly`

`load` 输出格式（所有字段均按需输出，为空时不出现）：
```json
{
  "cli_version": "1.0.6",
  "skills": [{"name": "...", "description": "...", "path": "..."}],
  "rules":  [{"name": "...", "description": "...", "path": "..."}],
  "repos":  [{"name": "...", "type": "...", "description": "...", "path": "..."}],
  "frameworks": [{"name": "...", "description": "...", "path": "..."}],
  "agents": [{"name": "...", "description": "...", "path": "..."}],
  "platform": "android",
  "system_prompt": "...",
  "user_prompt": "...",
  "notifications": "..."
}
```

- 不传参数时：`skills` / `rules` / `agents` / `repos` 只加载 `tags=base` 仓库的内容
- 传入关键词时：repo.name 精确匹配（不区分大小写）或 name/description 模糊匹配，只加载匹配仓库的 skills/rules/agents/repos
- `skills` / `rules` / `frameworks` / `agents`：列表为空时不输出该字段
- `repos`：列表为空时不输出；**带关键词时始终不输出**（关键词即 repo-name，无需重复返回）
- `system_prompt`：来自各仓库 `manifest.json` 指向的系统规则文件，全程生效，优先级高于用户指令；**带关键词时不输出**
- `user_prompt`：来自 `driving.config.json` 的 `user_prompt` 字段；**带关键词时不输出**
- `notifications`：CLI 根据运行时状态动态生成（版本不满足、CLI 有新版本、仓库有可更新内容）；**带关键词时不输出**

---

## agent — Agent 管理

每个 agent 存放在仓库的 `agents/<name>/` 目录，包含：
- `AGENTS.md`（必填）：YAML frontmatter + agent 指令/系统提示
- `SOUL.md`（可选）：人格、价值观、沟通风格
- `MEMORY.md`（可选）：最佳实践知识沉淀，随 git 同步，团队共享

**AGENTS.md frontmatter 字段：** `name`（必填）、`description`（必填）、`version`

```bash
driving agent list                              # 列出所有 agent（按仓库分组，显示启用状态）
driving agent list --repo <name>               # 只显示指定仓库的 agent
driving agent list --edit                      # 交互模式，勾选启用/禁用 agent（auto：自动选最短字段）
driving agent list --edit --mode enable        # 强制写 enabled 白名单
driving agent list --edit --mode disable       # 强制写 disabled 黑名单
driving agent load                        # 只加载 tags=base 的仓库 agent（JSON，供 AI 注入上下文）
driving agent load <keywords...>          # repo.name 精确匹配或 name/description 模糊匹配（不区分大小写，取并集）
driving agent memory get <name>           # 读取 MEMORY.md 内容
driving agent memory append <name> <content>        # 追加知识条目到 MEMORY.md
driving agent memory set <name> <content>           # 覆盖写入 MEMORY.md（会提示确认）
driving agent memory set <name> <content> --force   # 强制覆盖
driving agent memory clear <name>         # 清空 MEMORY.md

# 导出到外部 AI 工具（kiro 每次都覆盖复制；其他工具 macOS/Linux 使用软链接，Windows 使用复制；非 kiro 工具文件已存在时自动跳过）
driving agent export <name> --tool kiro          # → .kiro/agents/<name>.md（复制，需含 tools 字段）
driving agent export <name> --tool claude-code   # → .claude/agents/<name>.md（软链接/复制）
driving agent export <name> --tool cursor        # → .cursor/rules/<name>.mdc（软链接/复制，需含 alwaysApply 字段）
driving agent export <name> --tool windsurf      # → .windsurf/rules/<name>.md（软链接/复制，需含 trigger 字段）
driving agent export <name> --tool kiro --force  # 强制覆盖复制

# 上报子 agent 启动事件（由子 agent 在加载步骤第 0 步调用）
driving agent report <name> --path <feature-dir> --source "<触发来源描述>"
```

`agent load` 输出格式：
```json
[
  {
    "name": "android-review-workflow",
    "description": "Android 开发审查工作流",
    "version": "1.0.0",
    "path": "ai-driving/<repo>/agents/android-review-workflow/"
  }
]
```

关键词匹配规则：
- 不传关键词：只加载 `tags=base` 的仓库 agent
- 传关键词：repo.name 精确匹配（不区分大小写）或 agent.name/description 模糊匹配（子串包含即命中，取并集）
- 支持多个关键词：`driving agent load android ios`

**注意：** remote 仓库的 agent 记忆修改后需手动 `driving repo commit` + `driving repo push` 同步给团队。

---

## repo — 规范仓库管理

```bash
driving repo install --url <url>       # 安装远程仓库（Git submodule）
driving repo install --url <url> --branch main              # 安装时指定分支（推荐），安装后自动 checkout
driving repo install --url <url> --tag base --tag features   # 安装时指定标签（可多次）
driving repo install --url <url> --desc "描述"               # 安装时指定描述
driving repo install --url <url> --module "order:订单" --module "pay:支付"  # 安装时指定业务模块
driving repo install --local <path>    # 安装本地仓库（软链接，仅 macOS/Linux；Windows 不支持，请用 --local --name 创建空目录后手动复制文件）
driving repo install --local --name <name>  # 创建空本地仓库目录
driving repo install                   # 初始化所有未初始化的远程仓库
driving repo uninstall <name>          # 卸载仓库
driving repo list                      # 查看已安装仓库列表（JSON）
driving repo load [name...]            # 输出仓库列表（JSON，支持关键词过滤）
driving repo pull [name]               # 从远程拉取更新
driving repo commit [name] [message]   # 提交修改
driving repo push [name]               # 推送到远程
driving repo checkout <name> <branch>  # 切换仓库分支
```

**注意：**
- `repo install --url` 会将远程仓库作为 git submodule 添加到 `ai-driving/<name>/`
- `repo install --local <path>` 会在 `ai-driving/<name>/` 创建指向 `<path>` 的软链接（仅 macOS/Linux；Windows 不支持，改用 `--local --name` 创建空目录后手动复制文件）
- `--branch <branch>`：指定仓库分支，安装后自动 checkout；`driving repo install`（无参数）初始化时也会自动切换已配置的分支
- `--tag`：新增仓库标签，可多次指定；`--desc`：仓库描述（`--description` 简写）
- `--module <name:description>`：新增业务模块，可多次指定，供 `feature modules` 聚合使用
- `repo pull/commit/push` 不指定仓库名时对所有 remote 类型仓库执行，local 类型自动跳过
- `repo pull` 检测到仓库目录不存在或为空（submodule 未初始化）时，自动执行 `ensure_submodule_initialized` 完成初始化后即返回，无需手动执行 `repo install`
- `repo checkout` 必须指定仓库名和目标分支，切换前会自动 fetch 远程，存在未提交修改时拒绝切换；目录不存在或为空时同样自动初始化，初始化完成后继续执行分支切换

---

## power — Power 配置管理

Power 模式允许将多个目录下的 `driving.config.json` 合并使用，解决多分支场景下配置文件需要跨分支合并的问题。

**工作原理：** 在项目根目录创建 `driving.power.json`，列出多个包含 `driving.config.json` 的目录（本地或远程 git 仓库），driving-cli 运行时自动合并所有配置。不创建该文件则完全走传统模式，零感知。

```bash
driving power install                          # 无参数：初始化 driving.power.json 中所有未就绪的 power
driving power install --url <url>              # 安装远程 power（作为 git submodule）
driving power install --url <url> --name <n>   # 指定 power 名称
driving power install --url <url> --branch master  # 指定分支（推荐，避免分支无 driving.config.json）
driving power install --url <url> --force      # 强制重新安装
driving power install --name <n> --path <p>   # 注册本地目录为 power
driving power pull                             # 拉取所有远程 power 更新
driving power pull <name>                      # 拉取指定 power 更新
driving power list                             # 列出所有已配置的 power（JSON）
driving power uninstall <name>                 # 卸载一个 power（仅修改 driving.power.json，不删除目录）
```

**`power install --url` 安装逻辑（幂等）：**
1. 本地目录不存在 → clone + 注册
2. 本地目录存在但未注册 → 直接注册到 `driving.power.json`
3. 已注册但无 `driving.config.json` → 提示运行 `driving repo install --power <name>` 生成配置
4. 已完整安装 → 提示已存在，加 `--force` 可重新安装

**`power install`（无参数）初始化逻辑：**
- remote power，目录不存在 → `submodule update --init` 或 `submodule add`
- remote power，目录已初始化 → 跳过
- local power，目录存在 → 跳过
- local power，目录不存在 → warning 提示，跳过（本地目录需手动准备）

**合并规则：**
- `repos`：按 `name` 去重，先出现的 power 优先
- 单值字段（`gate_webhook`、`update_version_url` 等）：多个 power 中非空值必须相同，否则报错
- 某个 power 的 `driving.config.json` 不存在时自动跳过
- 所有 power 均无有效配置时，降级读取项目根目录的 `driving.config.json`

**`driving.power.json` 结构：**
```json
{
  "powers": [
    { 
      "name": "main",
      "type": "local",
      "path": "ai-driving/main-config",
      "url": null 
    },
    { 
      "name": "feature",
      "type": "remote", 
      "path": "ai-driving/feature-config", 
      "url": "https://git.xxx.com/feature-config.git",
      "branch": "master"
    }
  ]
}
```

- `url` 有值 → remote 类型（git submodule，支持 `power pull` 更新）
- `url` 无值 → local 类型（本地目录）
- `branch`（可选）：指定分支。`driving load` 时若 power 目录缺少 `driving.config.json`，自动执行 `git checkout <branch>` 切换到指定分支；未配置时缺少 `driving.config.json` 会输出警告提示配置该字段

---

## framework — 框架文档管理

```bash
driving framework list                        # 列出所有框架（JSON）
driving framework list <name>                 # 查看指定框架详情（含 sources 路径）
driving framework list --table                # 表格格式输出
driving framework install <name>              # 安装框架仓库（克隆到 submodules/）
driving framework checkout <name> <branch>    # 切换框架分支
driving framework pull <name>                 # 更新框架仓库
driving framework sources <name>              # 获取框架源码完整路径列表
driving framework load                        # 加载所有框架文档元信息（name/description/path/category）
driving framework load <keywords...>          # 按框架名或仓库名过滤（取并集）
driving framework load --category <name>      # 按分类过滤（如 ui-component，不区分大小写，可与关键词组合）
driving framework categories                  # 列出所有框架分类（name/description/count）
```

**框架名称格式：** 支持 `<repo-name>/<framework-name>` 格式解决同名冲突。

**框架分类（category）：** 框架可在 FRAMEWORK.md frontmatter 声明 `category`（如 `ui-component`），分类的描述配置在各仓库 `manifest.json` 的 `categories: [{name, description}]`。先用 `driving framework categories` 发现有哪些分类，再用 `driving framework load --category <name>` 取该类组件清单，最后 `driving framework load <name>` 读完整文档。

**gitlist.json 字段说明：**
- `project_name/__local__`：表示本地项目，不需要克隆，sources 路径相对于项目根目录
- `extends`：依赖的其他框架，install/sources 时自动处理扩展框架

---

## skill — 技能管理

```bash
driving skill list                   # 列出所有技能（按仓库分组，显示启用状态）
driving skill list --repo <name>     # 只显示指定仓库的技能
driving skill list --edit            # 交互模式，勾选启用/禁用技能（auto：自动选最短字段）
driving skill list --edit --mode enable   # 强制写 enabled 白名单
driving skill list --edit --mode disable  # 强制写 disabled 黑名单
driving skill load                   # 只加载 tags=base 的仓库技能（JSON，供 AI 注入上下文）
driving skill load <keywords...>     # base 仓库 + 匹配 repo.name 或 skill.name 的技能
```

关键词匹配规则：
- 不传关键词：只加载 `tags=base` 的仓库技能
- 传关键词：忽略 tags，repo.name 精确匹配（不区分大小写）或 skill.name/description 模糊匹配（子串包含即命中，取并集）
- 支持多个关键词：`driving skill load f-message f-qucall`

`skill load` 输出格式：
```json
[
  {
    "name": "skill-name",
    "description": "技能描述",
    "path": "ai-driving/<repo>/skills/<skill>/"
  }
]
```

**SKILL.md 格式要求：**
- 必须有 YAML frontmatter（`---` 包裹）
- 必须包含 `name` 和非空 `description` 字段
- description 为空的技能会被跳过

---

## rule — 规则管理

```bash
driving rule list                              # 列出所有规则（按仓库分组，显示启用状态）
driving rule list --repo <name>               # 只显示指定仓库的规则
driving rule list --edit                      # 交互模式，勾选启用/禁用规则（auto：自动选最短字段）
driving rule list --edit --mode enable        # 强制写 enabled 白名单
driving rule list --edit --mode disable       # 强制写 disabled 黑名单
driving rule load                    # 只加载 tags=base 的仓库规则（JSON，供 AI 注入上下文）
driving rule load <keywords...>      # base 仓库 + 匹配 repo.name 或 rule.name 的规则
```

`rule load` 输出格式：
```json
[
  {
    "name": "rule-name",
    "description": "规则描述",
    "path": "ai-driving/<repo>/rules/<rule>.md"
  }
]
```

**规则文件格式：** `.md` 文件，必须有 YAML frontmatter，包含 `name` 字段。

---

## feature — 需求功能管理

```bash
driving feature modules                       # 列出所有仓库的业务模块（JSON）
driving feature modules --features-only       # 只输出 tags 含 features 的仓库模块
driving feature list                          # 列出所有 features（从 modules 聚合路径遍历）
driving feature list --repo <name>            # 只扫描指定仓库
driving feature list --keywords game,list     # 关键词过滤（OR 关系）
driving feature list --keywords game --keywords list
driving feature list --detail                 # 输出完整字段
driving feature migrate --platform android    # 迁移全部 feature 目录（v1 → v2）
driving feature migrate --platform iOS --path <feature_dir>   # 指定目录迁移（可多次 --path）
driving feature migrate --platform android --exclude aidoc    # 跳过路径含 aidoc 的目录
driving feature migrate --platform android --include my-repo  # 只迁移路径含 my-repo 的目录
driving feature migrate --platform android --dry-run          # 预览迁移计划，不执行
driving feature migrate --platform android --owner zhangsan   # 自定义 owner 目录名（默认 owner-main）
driving feature migrate --platform android --yes              # 跳过确认直接执行
```

**`feature modules` 输出规则：**
- 仓库有 `modules`：每个 module 输出 `name`、`description`、`path`（`{repo.path}/{module.name}`）
- `tags` 含 `"features"` 且 `modules` 非空：只输出 module 条目，不追加 `features` 兜底
- 其余仓库（含 `tags=features` 但 `modules` 为空）：追加 `{repo.path}/features` 兜底条目
- `--features-only`：只保留 `tags` 含 `"features"` 的仓库条目，过滤其余仓库

**`feature list` 遍历逻辑：**
- 普通仓库：扫描 module path 下各子目录，查找 `FEATURE.md`（单层）
- `tags` 含 `"features"` 的仓库：深度递归扫描，兼容多层结构（如 `{module}/{年度-季度}/{feature}/FEATURE.md`），自动提取 `quarter` 字段

**FEATURE.md frontmatter 字段：** `name`（必填）、`title`、`description`、`status`、`priority`、`module`、`assignee`、`tags`、`urls`

**`feature list` 输出计算字段：** `path`（完整路径）、`repo`（模块名）、`quarter`（年度季度，如 `2026-Q2`，仅深度扫描时有值）

**`feature migrate` v1 → v2 迁移规则：**
- `docs/technical-design.md` / `impact-map.md` / `coding-plan.md` / `implementation-notes.md` / `gate-state.json` → `docs/{platform}/{owner}/`
- `review/`（feature 根）→ `docs/{platform}/{owner}/review/`
- `docs/ui-design/` → `ui-design/`（提升到 feature 根，跨平台共享）
- 迁移完成后自动写入 `format_version: v2` 到 `FEATURE.md` frontmatter，已迁移的目录下次运行自动跳过
- `--platform` 必填；`--owner` 默认 `owner-main`；`--path` 可多次指定；不传 `--path` 则扫描全部
- `--include`：只处理路径包含指定关键词的目录（可多次指定，OR 关系，大小写不敏感）
- `--exclude`：跳过路径包含指定关键词的目录（可多次指定，OR 关系，大小写不敏感）

## refine — Refine 提案管理

```bash
driving refine list                          # 列出所有仓库的 pending refine 提案（按类型分组）
driving refine list --type skill             # 只显示指定类型（skill/rule/agent/framework）
driving refine list --repo <name>            # 只显示指定仓库的 refine
driving refine load                          # 输出所有 pending refine 内容（JSON，供 AI 检索）
driving refine load <name...>                # 按文件名模糊匹配（包含即命中），支持多个，name可以是skill-name、rule-name、agent-name、framework-name
driving refine load --type rule              # 只加载指定类型的 refine
driving refine commit <repo> --file <path>   # 将 refine 文件提交到 git（add + commit + push）
driving refine merge <repo> --file <path>    # 合并收尾：追加 REFINE_LOG → 上报 webhook → 删除 refine 文件 → commit/push
driving refine merge <repo> --file <path> --changed-file <path>  # 指定实际修改的文件加入 commit（可多次指定，未传则提示确认是否跳过）
driving refine merge <repo> --file <path> --operator <name>      # 指定操作者写入 REFINE_LOG
driving refine merge <repo> --file <path> --trigger-source <src> # 本次合并操作的触发来源（gate/self/manual），用于 webhook 上报
driving refine merge <repo> --file <path> --trigger-reason "..."  # 本次合并操作的触发原因，用于 webhook 上报
driving refine merge <repo> --file <path> --no-push              # 只 commit，不 push

# 上报规范缺陷信号（由 self-refine 技能在 gate check 自检后调用，走 agent_webhook 静默上报）
driving refine report --source <gate|self|manual> --description "<描述>"               # 上报信号（必填 --source）
driving refine report --source self --stage <阶段名> --problem-type <类型> --description "<描述>" --evidence "<证据>"
driving refine report --source gate --gate-id <gate-id> --problem-type <类型> --description "<描述>"
```

`refine load` 输出格式：
```json
[
  {
    "name": "2026-04-10-skill-self-refine-references-lazy-load",
    "description": "在 refine-workflow.md 末尾补充 references 按需加载判断标准",
    "path": "ai-driving/driving/refines/2026-04-10-skill-self-refine-references-lazy-load.md"
  }
]
```

**Refine 文件格式：** `.md` 文件，必须有 YAML frontmatter，包含 `target_type`、`description`、`status` 字段。

---

## gate — 门禁规则管理

```bash
driving gate list                          # 以表格形式列出所有 gate（列：ID/Name/Type/Location/Repo）
driving gate list --json                   # 以 JSON 数组格式输出（每条记录含 id/name/type/location/repo）
driving gate load                          # 加载所有 gate 的完整内容（JSON）
driving gate load <gate-id>                # 加载指定 gate（大小写不敏感）
driving gate load <gate-id> <gate-id> ...  # 加载多个指定 gate
driving gate request <gate-id> --path <dir>                                       # 执行门禁请求（auto_pass → 交互选择）
driving gate request <gate-id> --path <dir> --platform <platform>                 # 指定平台（android/iOS/harmony/kuikly），需求/分工门禁的 gate-state.json / state.json 写入 {dir}/docs/{platform}/；技术实现门禁的 platform 带 owner 子路径（默认 android/owner-main，分工时如 android/owner-芒果），状态写入该负责人目录
driving gate request <gate-id> --path <dir> --platform <platform> --owner <owner> # 指定负责人（main/apple/owner-main），激活 $vars.owner_dir = {platform_dir}/owner-{owner}
driving gate request <gate-id> --path <dir> --context '{}'                        # 附带 JSON 上下文变量
driving gate request <gate-id> --path <dir> --platform <platform> --context '{}'  # 同时指定平台和上下文
driving gate request <gate-id> --path <dir> --dry-run                             # 仅预览模板，不执行交互
driving gate respond <gate-id> --path <dir> --action <操作名>                      # 非交互式提交门禁选择（配合非终端环境使用）
driving gate respond <gate-id> --path <dir> --action <操作名> --note ""            # 修改类操作附带说明
driving gate respond <gate-id> --path <dir> --platform <platform> --owner <owner> --action <操作名>  # 指定平台和负责人
driving gate status --path <dir>                                                  # 查看所有 gate 状态
driving gate status --path <dir> --platform <platform>                            # 查看指定平台的 gate 状态
driving gate status --path <dir> --platform <platform> --owner <owner>            # 同时指定负责人
driving gate status <gate-id> --path <dir> --platform <platform>                  # 查看指定平台的指定 gate 状态
driving gate history <gate-id> --path <dir> --platform <platform>                 # 查看指定 gate 历史记录
driving gate history <gate-id> --path <dir> --platform <platform> --owner <owner> # 同时指定负责人
driving gate check <gate-id>                                                       # 执行门禁前置自检，输出自检问卷供 AI 回顾本阶段执行情况
```

`gate load` 输出格式：
```json
{
  "system_prompt": "...",
  "gates": [
    {
      "id": "GATE-R1",
      "name": "工作区间确认",
      "type": "mandatory",
      "location": "xxx",
      "trigger": "...",
      "template": ["..."],
      "actions": { "确认": "...", "修改": "..." }
    }
  ]
}
```

- `system_prompt`：来自 `gates.json` 顶层字段，多仓库拼接，为空时不输出
- 任一 ID 找不到时，`gates` 返回空数组 `[]`，不报错退出
- 多仓库存在相同 ID 时，返回 `driving.config.json` 中排在最前的仓库的 gate，并输出警告

**gate 配置：** 仓库 `manifest.json` 中通过 `"gates": "rules/gates.json"` 指向门禁定义文件。无该字段或文件不存在时静默跳过，不影响其他命令。

**gate check 输出格式：**
```json
{
  "gate_id": "GATE-R5",
  "gate_name": "需求拆解文档确认",
  "user_prompt": "请回顾本阶段执行情况，如实填写以下字段（无异常填 null）。如果存在非 null 字段时，触发 `self-refine` 技能进行总结并上报。",
  "self_checks": {
    "inferred_steps": "本阶段有没有文档未覆盖、靠推断执行的步骤？(描述 or null)",
    "rule_conflicts": "本阶段有没有遇到规则冲突、不知道该听哪个？(描述 or null)",
    "doc_mismatch": "本阶段有没有按文档操作但结果不符？(描述 or null)"
  }
}
```

- `self_checks` 内容来自 `gates.json` 中该门禁的 `self_checks` 字段；未配置时使用默认三问
- gate-id 大小写不敏感；不存在时返回 `{"error": "..."}` 并退出

**gate 模板变量：** `gates.json` 中的 `template`、`actions.next`、`auto_pass.conditions` 字段支持以下变量：

| 变量 | 来源 | 说明 |
|------|------|------|
| `{{path}}` | `--path` 参数 | feature 目录路径 |
| `{{context.xxx}}` | `--context` JSON | 用户传入的上下文字段 |
| `{{state.xxx}}` | gate-state.json | 当前 gate 历史状态 |
| `{{$vars.platform_dir}}` | CLI 内部计算 | `{path}/docs/{platform}`（无 platform 时为 `{path}/docs`） |
| `{{$vars.owner_dir}}` | CLI 内部计算 | 传 `--owner` 时为 `{platform_dir}/owner-{owner}`（已含 `owner-` 前缀则直接使用）；未传时等于 `{platform_dir}` |

`{{$vars.xxx}}` 变量由 CLI 自动注入，路径结构变更时只需修改 CLI，无需改动 `gates.json`。完整变量列表可通过 `driving gate load` 输出的 `vars` 字段查看。

---

## update — 更新管理

```bash
driving update                       # 检查并安装更新
driving update --check               # 仅检查是否有新版本
driving update --force               # 强制重新安装
driving update -y                    # 跳过确认提示
driving update --url <url>           # 使用自定义 version.json URL（会保存到 config）
```

**平台差异：**
- macOS / Linux：安装到 `~/.driving-cli/driving`，无写权限时自动 fallback 到 `sudo`
- Windows：安装到 `~\.driving-cli\driving.exe`，权限不足时提示以管理员身份运行 PowerShell（不使用 `sudo`）；下载地址自动从 `version.json` 的 `download_url_windows` 字段读取，不存在则将 `download_url` 末尾 `/driving` 改为 `/driving.exe`

**Windows 首次安装（未通过 pip 安装时）：**
```powershell
# 方式一：pip 安装（推荐，需 Python 3.8+）
pip install git+https://github.com/sonuan/driving-cli-tool.git

# 方式二：预编译 .exe
irm https://raw.githubusercontent.com/sonuan/driving-cli-tool/main/install.ps1 | iex
```

---

## 常见工作流

### 初始化新项目
```bash
driving repo install --url https://github.com/your-org/driving
# 编辑 driving.config.json，给 driving 仓库加上 "tags": ["base"]
driving skill load   # 验证技能加载（只加载 base 仓库）
driving rule load    # 验证规则加载
```

### 按模块加载上下文
```bash
driving skill load                    # 只加载 tags=base 的仓库
driving skill load f-message          # 精确匹配 repo.name=f-message（加载该仓库全部技能）
driving skill load message qucall     # 模糊匹配 name/description 包含 "message" 或 "qucall"
driving skill load code-review        # 模糊匹配 skill.name 包含 "code-review"
```

### 查看并启用/禁用技能
```bash
driving skill list           # 查看当前状态
driving skill list --edit    # 交互式编辑（需要 prompt_toolkit）
```

### 查看框架源码路径（供 AI 读取源码）
```bash
driving framework sources ximage   # 获取 ximage 框架的完整源码路径
```

### 搜索需求功能
```bash
driving feature list --keywords 登录,注册   # 搜索包含登录或注册的需求
```

### 使用 agent
```bash
driving agent load                                          # 查看 base 仓库的 agent
driving agent load review                                   # 模糊匹配 name/description 包含 "review"
driving agent load driving                                  # 精确匹配 repo.name=driving（加载该仓库全部 agent）
driving agent memory get android-review-workflow             # 读取 agent 记忆
driving agent memory append android-review-workflow "用户偏好简洁风格"
driving repo commit my-local "update agent memory"         # 同步记忆到团队
```

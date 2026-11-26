# hxxAgentForge 第一期改造开发计划

**状态**：待确认（确认前不开始任何开发）
**依据**：`docs/design/AI-Agent-Forum-Phase1-Design.md`（v0.3）、`docs/html/agent-forum-prototype.html`
**底座**：`src/` = Discourse（Rails + Ember，要求 Ruby >= 3.4、PostgreSQL、Redis、Node/pnpm）

---

## 1. 现状调查结论

| 项 | 现状 | 结论 |
|----|------|------|
| PostgreSQL | 本机 16 服务运行中，5432 通；密码 ABC123### | 可用；需启用扩展 `hstore`、`pg_trgm`（`unaccent`），自带账号需有建库/建扩展权限 |
| Redis | 6379 通 | 可用；需确认版本 >= 6.2（Windows 版 Redis 常为 3.x/5.x，过旧需换） |
| Node | v24.14 + pnpm 存在 | 可用（Discourse 需要的版本以 `package.json` engines 为准，必要时用 fnm/nvm 切换） |
| Ruby | **未安装**（要求 >= 3.4） | 需安装 |
| WSL / Docker | **均未安装** | 见 §2 |
| 原型图 | 仅有一份前台原型（含"主人控制台"，无独立管理后台页面） | 管理后台按设计文档 §8、§9.4 推导，基于 Discourse admin 扩展，原型缺口列入 §7 待确认 |

## 2. 运行 / 编译环境方案（先自动解决）

Discourse **官方不支持原生 Windows**（依赖 fork、unicorn、若干 Linux 原生 gem 等）。方案按优先级：

1. **方案 A（推荐）：WSL2 + Ubuntu 内运行 Rails**
   - 自动步骤：`wsl --install`（需管理员权限，**可能需要重启一次**）→ 安装 rbenv/ruby-build + Ruby 3.4、libpq-dev、imagemagick 等 → `bundle install`、`pnpm install`。
   - 连接 Windows 本机 PG/Redis：WSL2 镜像网络模式（`networkingMode=mirrored`）或使用宿主机 IP；需让 PG `listen_addresses`/`pg_hba.conf` 允许 WSL 访问（会修改本机 PG 配置，先征得你同意）。
   - 源码仍在 `D:\...\src`，WSL 通过 `/mnt/d` 访问（性能偏慢，必要时同步到 WSL 内部磁盘运行）。
2. **方案 B：Docker Desktop**（同样需要 WSL2/虚拟化，且需安装 Docker），不优先。
3. **方案 C：原生 Windows（RubyInstaller + MSYS2）**：大量 gem 编译/运行失败风险，且需改动 Discourse 核心，**不建议**。

> 若方案 A 因无管理员权限、BIOS 虚拟化未开启等无法自动完成，我会停下并给出具体手动步骤建议。
> 磁盘 D: 剩余约 23 GB，够用（WSL 发行版 + gems + node_modules 约 8–10 GB）。

## 3. 总体架构

- **不侵入核心**：所有业务以新插件 `src/plugins/hxx-agentforge/`（Ruby 后端 + Ember 前端）实现；必要的主题/首页以插件内的自定义首页路由实现。
- 业务数据库表统一使用前缀 `hxx_`，通过插件迁移管理。
- 精简默认插件：关闭与第一期无关的插件（adplugin、patreon、zendesk、subscriptions、lti、login-with-amazon、apple/microsoft-auth 等），保留 solved、poll、reactions、topic-voting、templates、details 等有用项（通过 `plugin_enabled` 站点设置/配置默认值关闭，不删除代码）。

## 4. initdb 模块（本期最先交付，满足"启动自动建库与自动迁移"）

位置：`src/plugins/hxx-initdb/`（独立插件，**最先加载**，`config/initializers` 级别执行）。

功能：
1. **自动创建数据库**：启动时用管理连接（`postgres` 库）检查目标库是否存在，不存在则 `CREATE DATABASE`（UTF8）。
2. **自动启用扩展**：`hstore`、`pg_trgm`、`unaccent`（及按需 `vector`，缺失则降级并告警）。
3. **自动迁移**：库为空/有待执行迁移时自动执行 Discourse 核心迁移 + 所有插件迁移（等价 `db:migrate`），含 `hxx_*` 表。
4. **自动种子**：首次启动写入默认站点设置、分类（7 个板块）、标签、内容模板、道具初始数据、规则清单、初始管理员（密码来自环境变量）。
5. **并发与安全**：用 PG advisory lock 避免多进程同时迁移；失败则明确报错并阻止启动；全过程日志；幂等可重复执行。
6. **配置**：通过环境变量/`config/database.yml`（`HXX_DB_HOST/PORT/USER/PASSWORD/NAME`、`HXX_REDIS_URL`），密码不入库、不提交仓库（使用 `.env.local`，已加入 `.gitignore`）。
7. **开发/生产开关**：`HXX_INITDB_AUTO=1` 默认开启；提供 `rake hxx:initdb:status` 查看状态。

## 5. 业务模块拆分（均在 `hxx-agentforge` 插件内）

| 阶段 | 模块 | 内容 | 对应设计 |
|------|------|------|----------|
| M0 | 环境 + initdb | §2、§4；跑通 Discourse 空站点 | §9 |
| M1 | 身份与所有权 | `hxx_owners`（个人/组织）、`hxx_agents`（Agent 作为特殊用户 + 归属表）、权限边界（单笔上限、日上限、自动接单）；Agent 标识徽章；主人名下 Agent 列表跳转 | §2 |
| M2 | 能量账户 | `hxx_energy_accounts`、`hxx_energy_ledger`（只增不改的流水）、充值订单（第一期为"平台内直接充值"，对接 **模拟支付网关**，真实支付渠道待你确认）、新 Agent 起始能量、事务+行锁扣减、提现入口预留（状态字段） | §3 |
| M3 | 报价系统 | `hxx_quotes`、`hxx_quote_rounds`（还价记录，含 min/max 私密字段）、状态机（草稿→已发布→还价中→已成交→执行中→待确认→已完成/已取消/争议中→已裁定）、价格可见性（仅发布者+管理员）、成交时能量结算（强事务）、交付确认 | §4 |
| M4 | 风控 | 每日 0 元报价上限、每 Agent 每日任务上限（站点设置可调）、提醒/轻罚/中罚/重罚、扣能量、冻结/封禁、申诉；争议协商→平台裁定→一次申诉；全量审计日志 | §8 |
| M5 | 数字人与道具 | 主人分身（基础外形自定义，SVG 组合渲染）、3–5 个视觉道具、能量购买与穿戴、在帖子/报价单/个人页统一展示；预留 `hxx_scenes` 表 | §7 |
| M6 | 内容板块与模板 | 种子创建 7 个板块（前沿速递、工具与排行、评测与体验、真实案例、研究与实验、问题与解法、商业与报价）；评测/案例/前沿三套发帖模板（Discourse category template）；新内容高亮 | §5 |
| M7 | 首页与前台 UI | 按原型实现：推新区、推荐区、排行入口、测评中心大入口、试用入口、动态流（最新/热门/关注）、能量充值入口、热门报价、道具商城入口；主人控制台（Agent 管理、能量流水、充值）、报价详情/还价页、排行页、统计页；中/英 i18n、深浅主题切换 | §6、原型 |
| M8 | 管理后台 | Discourse admin 内新增：报价与争议裁定（可见完整价格区间）、能量流水审计、处罚/封禁管理、Agent 与主人管理、首页推荐位运营、道具管理、上限参数配置 | §8、§9.4 |
| M9 | Agent 接入 | 扩展 API Key 作用域：Agent 以所属主人身份发帖、报价、还价、交付；JSON 端点；MCP 接入点（基于 discourse-ai/现有能力评估） | §9.4 |
| M10 | 规则清单 | 注册时勾选同意规则（附录 §12）并记录版本 | §10 |
| M11 | 测试与交付 | 后端 RSpec（能量事务、报价状态机、价格可见性、限额）、核心前端测试、启动自检脚本、README/运行文档 | — |

## 6. 里程碑与顺序

M0 → M1 → M2 → M3 → M4（后端核心，可用 API 联调）→ M6 → M5 → M7 → M8 → M9/M10 → M11。
每个里程碑结束：能启动、迁移自动执行、给你可验证的页面/接口和简要说明。

## 7. 需要你确认的事项

1. **环境方案**：同意使用方案 A（安装 WSL2 + Ubuntu，需管理员权限、可能重启）吗？同意我修改本机 PG 的 `pg_hba.conf`/`listen_addresses`（仅必要时）吗？
2. **Redis 版本**：若本机 Redis 版本过旧，是否允许我在 WSL 内另起一个 Redis（仍用密码 ABC123###）？
3. **数据库名/账号**：默认使用超级用户 `postgres` + 库名 `hxx_agentforge`，是否可以？
4. **支付**：第一期充值先用"模拟支付"（点击即到账，便于演示），真实支付渠道后续接入，可以吗？
5. **管理后台**：原型中没有后台页面，是否同意直接使用 Discourse 管理后台风格扩展（不单独出原型）？
6. **精简插件**：§3 的关闭清单是否同意？
7. **语言**：默认语言中文，同时保留英文（原型有语言切换），可以吗？
8. **版本控制**：`src/` 目前不是 git 仓库，是否需要我在 `hxxAgentForge` 根目录 `git init` 便于追踪改动？

确认后（可直接回复"确认"，或指出需调整项），我将从 M0 开始。

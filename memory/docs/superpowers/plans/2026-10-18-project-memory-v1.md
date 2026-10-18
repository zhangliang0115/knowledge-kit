# 项目记忆助手 v1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为一个明确授权的 Claude Code 项目交付“脱敏会话 → 候选 → 人工批准 → 跨会话召回 → 撤回”的可追溯闭环，并完成 CLI MVP、Web 和完整 v1 验收。

**Architecture:** TypeScript CLI 与 loopback Web 共用单写者 Core，SQLite 是唯一权威源；文件采集、持久任务与模型请求分离。Hook 独立打开只读数据库，只检索并渲染已批准版本，不启动 Core 或模型。先完成可行性门槛，再实现 CLI 闭环，最后增加 Web、恢复和真实试点。

**Tech Stack:** Node.js 24 LTS、TypeScript、better-sqlite3（兼容性不通过时在 Task 1 决策替换）、SQLite WAL/FTS5 trigram、commander、Zod、Vitest、Playwright；Web 使用原生 HTTP 和静态 HTML/CSS/JS；不引入向量库或框架后台。

**Spec:** [落地执行方案](../../../MemHub-落地执行方案.md) 为设计主线；[验收标准](../../../MemHub-验收标准.md) 为验收口径；[调研报告](../../../MemHub-调研报告.md) 第 19–25 节仅解释背景，不覆盖前两份文件的修正。

## Global Constraints

- 单机、单用户；首轮试点仅启用一个明确授权的项目；Claude Code；先验收 macOS，其他平台后续评估。
- `kind` 在 v1 仅允许 `constraint`、`convention`、`decision`；scope 固定为 project。
- 自动提取候选，用户确认后生效；禁止模型自动提升为已批准记忆。
- SQLite 是唯一权威源，Markdown 只作导出；内容修改永远创建新 revision。
- 只访问用户显式配置的单一端点、发送前脱敏与最小化、发送内容可预览、密钥不进入日志和导出；不启用遥测。
- 启动补扫 + 文件事件唤醒 + 每 30 秒补扫；每文件每轮读取预算 4 MiB，完整单记录上限 8 MiB。
- 累计 5 个完整 turn、完整 turn 已等待 5 分钟、用户执行 `mem flush` 任一触发；单批最多 20 个 turn。
- Stage A 生成最多 10 个候选，允许空数组；并发抽取为 1。
- 网络故障含首次最多 3 次；输出结构修复最多重试 1 次；每日最多 200 个抽取批次，数据目录软上限 2 GiB，日志滚动上限 20 MiB。
- 默认最多 5 条完整约束/约定；预算目标为 800 个目标宿主模型 tokens，包含说明和调用指引；未知 tokenizer 只能报告估算。
- Hook p95 ≤300 ms、进程内截止 500 ms；本地搜索 p95 ≤200 ms；只读失败时空输出且不阻塞宿主。
- 首版安装显式匹配 `startup|resume|clear|compact`；fork 不自动继承接入，另行验收。
- 核心模块逐模块行覆盖率和分支覆盖率均 ≥80%；Precision ≥90%、约束/约定 Recall ≥80%、噪声率 ≤5%、重复率 ≤5%、可追溯率 100%、禁止生效事件 0 次。
- 质量集 30 个开发、20 个验证、50 个冻结验收；冻结配置运行 3 次，每次均过门槛；至少 20 个配对跨会话任务，启用后遵守至少 18/20。
- 至少三天影子模式、至少五个工作日批准后注入模式；不可用自动化瞬时执行冒充真实试点。

## Review Focus

以下五项是审查时须主动追踪的失败路径，已补入对应任务的测试要求：

1. 证据删除、撤权、治理变更与迟到模型结果交错：Task 8/11/12 必须校验版本和授权，不能仅比较正文哈希。
2. 同 inode 等长改写、已读前缀改写、缺宿主 ID 的重放：Task 6 保守隔离歧义，不按正文合并事件。
3. 宿主 Agent 伪造审批调用、确认凭证重放、只拒绝一个冲突就恢复旧规则：Task 4/12/16 验证全部前置条件，不能凭本机请求自动批准。
4. 旧数据库恢复后遗忘撤回、共享备份残留被清空项目：Task 18/19 先关闭注入，缺审计转待复核，删除受管恢复路径。
5. Hook 的同步 SQLite 等待超过 JS 定时器、配置修改竞态、未知 tokenizer 与长否定句：Task 14/15 测量真实子进程，保留全文或整条省略，精确卸载。

---

## 0. 当前事实、裁决与执行边界

2026-10-18 检查：当前目录 `/Users/zhangliang/workspace/demo/demo-claude-config/memory` 不是 Git 仓库，没有待扩展的应用源码。这次只生成实施计划；没有安装依赖、读取真实 transcript、配置免费 API、修改宿主设置或执行验收。

**实现根目录（计划值）：** `/Users/zhangliang/workspace/demo/demo-claude-config/memory/project-memory`。后文 `src/`、`tests/` 等均相对此根。Task 1 在这里建立独立仓库；后续通过 using-git-worktrees 技能创建或核实实现分支/worktree，不直接在 main 开发。现有三份文档保持原样，复制到实现仓库 `docs/spec/` 并记录 SHA-256；本计划复制到实现仓库相同的 `docs/superpowers/plans/` 路径。复制后 Spec 链接改指向 `../../spec/`，不得保留失效相对链接。

**本计划采用的冲突裁决，批准计划即确认这些解释：**

| 原文冲突/空缺 | 本计划裁决及影响 |
|---|---|
| 验收开头“运行不出公网”、AC-01“模型在本机可用”；方案模块表“本地模型”，13.2“阻断非回环后完整链路” | 采用新版免费远程 API 路线：本地操作离线可用，抽取断网 blocked；只允许单一获批端点。AC-38/39 是实际网络判据，不声称全离线。Task 1 在 spec 副本中同步勘误并留 ADR。 |
| 方案 T01 声称模型质量不达标立即暂停，但 50 条冻结集与生产管线在后期才形成 | G0 先用 30/20 集对 spike 实际 A/B 输出人工计分（沿用相同阈值，不能代替最终验收）；不达标不进入自动抽取主体实现。最终冻结 50 条、3 次运行在 Task 20。 |
| “人工批准”不能被同账号任意进程严格证明 | 仅承诺阻止常规 Agent CLI/IPC 调用；交互 TTY 或浏览器一次性确认，凭证绑定具体操作和 revision。伪造 PTY/读进程内存的同账号恶意进程不在边界内，文档不可声称强人机认证。 |
| project scope 下如何区分局部规则 | `scope='project'` 不变，增加有证据支持的 `appliesTo`（项目内路径集合，空数组代表全项目）；未明确的子范围不推断。是否冲突同时比较作用路径、事实时间。 |
| 运行默认值尚缺少指纹周期、租约、tokenizer 回退等 | 计划默认：前缀全量分块指纹核查 300 秒、补扫处理裕量 5 秒、租约 60 秒/心跳 15 秒、Hook 宿主 timeout 2 秒、无 tokenizer 回退总 UTF-8 字节 800。Task 1 在指定机器实测后冻结；更改须写 ADR 并先改测试，不能测完再定过线标准。 |
| 工程样本与真实质量标签混淆 | 合成 fixtures 验证事务与边界；模型质量由人工标签和真实 API 输出计分。用户未提供授权样本/平台时相应门槛 BLOCKED，不制造 PASS。 |

这是一个紧耦合闭环，不拆为三个独立产品。保留一份实施计划、三个阶段门槛；Web 和运维有独立任务，可在 CLI MVP 后延后，但不因此宣布 v1 完成。阶段编号沿用源文档 T01–T11，下面 Task 1–20 是更细的工程审查单元。

## 1. 文件结构和公共契约

| 文件/目录（拟创建） | 单一职责 / 拥有任务 |
|---|---|
| `package.json`, `package-lock.json`, `.node-version`, `tsconfig.json`, `vitest.config.ts`, `eslint.config.js`, `.gitignore` | 版本、命令、测试与忽略运行数据 / Task 1 |
| `docs/spec/*.md`, `docs/adr/001-stack.md`, `docs/adr/002-contracts.md`, `config/compatibility.json` | 设计快照、冲突裁决、机器/宿主/模型预算与版本 / Task 1–2 |
| `spikes/{sqlite,host,model}.ts` | 可删除的兼容性探针，不能被产品运行时导入 / Task 1–2 |
| `src/domain/{types,schemas,errors}.ts` | 跨模块 ID、数据类型、Zod schema、稳定原因码 / Task 3 |
| `src/storage/{database,migrate,projection}.ts`, `src/storage/migrations/001-initial.sql` | 数据库生命周期、迁移、事务内有效视图投影 / Task 3、12 |
| `src/core/{projects,writer,ipc,confirmation,service}.ts` | 路径授权、唯一写者、IPC、人工确认、业务编排 / Task 4、8 |
| `src/sources/claude/{adapter,identity,privacy}.ts` | 版本化解析、事件身份、持久化前脱敏 / Task 5 |
| `src/ingest/{reader,fingerprints,ledger,watcher,turns}.ts` | 增量读取、重写检测、事务账本、补扫、轮次 / Task 6–7 |
| `src/jobs/{queue,worker,budget}.ts` | 持久租约、调度和预算 / Task 8 |
| `src/extraction/{request,transport,candidates,review,pipeline}.ts`, `prompts/{generate-v1,review-v1}.md` | 请求最小化、端点限制、A/B、提交快照校验 / Task 9–11 |
| `src/governance/{review,revisions,conflicts,suppression,evidence}.ts` | 审批事务、版本、冲突、遗忘和证据失效 / Task 12 |
| `src/recall/{search,context,budget}.ts` | FTS 字面检索、状态联查、稳定上下文 / Task 13–14 |
| `src/integration/{hook-entry,hook-worker,settings,install}.ts` | 有截止的只读 Hook 和精确配置合并 / Task 15 |
| `src/cli/{main,commands,review}.ts` | `mem` 入口/命令映射/TTY 审阅；只委派 Core / Task 3–15 |
| `src/review/{server,security}.ts`, `src/review/public/{index.html,app.js,style.css}` | loopback 审阅与纯文本 UI / Task 16 |
| `src/ops/{doctor,export,logging,backup,restore,purge,launch-agent}.ts` | 诊断、导出、受管数据生命周期 / Task 17–19 |
| `tests/{fixtures,helpers,unit,integration,e2e,performance}/`, `evals/{dev,validation,acceptance}/` | 合成 fixtures、分层测试、人审标签及冻结数据 / 各任务 |
| `scripts/{check-datasets,eval,acceptance,performance}.ts`, `docs/{privacy,operations,compatibility,pilot}.md` | 数据防泄漏检查、报告和交付操作说明 / Task 2、20 |

**公共类型（Task 3 定义，后续不得另建同名不同结构）：**

- ID 均为不透明 string；时间均为 UTC epoch milliseconds，事实时间未知为 `null`；offset 为源文件字节位置，引文 `[start,end)` 为脱敏文本 Unicode code point 下标，不能混用 UTF-16 索引。
- `ProjectRef={projectId:string,generation:number,root:string}`；`Scope={scope:'project',appliesTo:string[]}`；服务端通过调用 cwd 的 realpath 和登记项重新解析身份。
- `EvidenceRef={messageId:string,revision:number,start:number,end:number,quote:string}`。
- `Message={eventId:string,revision:number,projectId:string,episodeId:string,sourceRecordId:string,blockIndex:number,role:string,author:'person'|'agent'|'tool'|'host'|'unknown',provenance:'direct'|'quoted'|'managed'|'uncertain',text:string,createdAt:number|null}`。
- `TurnRef={turnId:string,revision:number}`；`Batch={project:ProjectRef,turns:TurnRef[],messages:Message[],inputHash:string,pipelineVersion:string}`。
- `CandidateDraft={kind:'constraint'|'convention'|'decision',title:string,content:string,scope:Scope,validFrom:number|null,evidence:EvidenceRef[],reason:string,uncertainties:string[]}`。标题 1–120、正文 1–2000 code points、理由 ≤1000；证据 1–20 条；这些长度是本计划工程决定，Task 1 记录。
- `Verdict={action:'add'|'merge'|'supersede'|'conflict'|'discard',candidate:CandidateDraft,targetRevisionIds:string[],reasonCode:string}`。
- `ActiveMemory={id:string,revision:number,revisionId:string,governanceVersion:number,kind:CandidateDraft['kind'],title:string,content:string,scope:Scope,validFrom:number|null,important:boolean}`。
- `ReviewCommand={operation:'approve'|'reject'|'edit'|'resolve'|'revoke'|'manual',targetId:string,expectedRevision:number,expectedGovernanceVersion:number,payload:unknown}`；`ConfirmationProof` 为 Core 验证后生成的不透明句柄，不能由模型/调用者传 `actor='human'` 获得。
- `Result<T>={ok:true,value:T}|{ok:false,code:string}`；确认失败统一 `confirmation_required`，旧租约统一 `stale_lease`；状态冲突统一 `stale_revision`/`stale_evidence`/`stale_governance`/`project_revoked`；CLI 退出码成功 0、输入错误 2、状态冲突 3、暂时不可用 4、内部错误 1。Hook 另用 fail-open 的 0 退出与空上下文。
- `Db` 包装选定 SQLite driver；`ReadDb` 只读；所有持久写函数仅在持有 writer lease 的 Core 数据线程执行。测试直接调用写函数时仍必须通过测试 Core，不能绕过事务断言。

**测试配置：** Vitest 只收集 `tests/**/*.test.ts`，不收集 Playwright 的 `*.spec.ts`；性能与外网实机测试显式单独运行。Playwright 只收集 `tests/e2e/*.spec.ts`。验证阶段运行的 CLI E2E 使用 Vitest 子进程/PTY，Web E2E 使用 Playwright，不混用两套 runner。

**统一命令契约：** `npm run build` 编译到 `dist/` 并复制 Web 静态资源、SQL migrations 与 prompts；`npm run typecheck`、`npm run lint`；`npm test -- <file>` 等价 `vitest run <file>`；`npm run test:coverage` 逐模块核查；`npm run test:e2e` 执行 Playwright；`npm run mem -- ...` 调用构建后的 CLI。`--data-dir` 是全局隔离参数（默认 `~/.project-memory/`），所有测试必须显式指定临时目录。网络默认关闭，只有明确标记的实机探针和真实 eval 才可访问已确认端点。

**测试风格：** 下文代码是必须落地的测试名/断言，不要求逐字复制测试组织。`fixture(name)` 在 Task 2 返回显式、固定输入；`h` 为 Task 3 起按任务扩展的临时 Core 测试夹具，所有方法必须委派实际模块，不许 mock SQLite 或返回预制业务结论。生产函数签名以各 Task 的 Interfaces 为准。

## 2. 可行性阶段（G0）

### Task 1: 建立独立工程并验证本地存储与宿主能力（T01）

**Files:** Create 工程配置文件（见文件表）、`spikes/sqlite.ts`, `spikes/host.ts`, `docs/adr/001-stack.md`, `docs/adr/002-contracts.md`, `config/compatibility.json`, `tests/integration/compatibility.test.ts`。

**Interfaces:** Consumes 三份 spec；Produces `probeSqlite(dir:string):Promise<{wal:boolean,foreignKeys:boolean,trigram:boolean}>`、`validateCompatibility(value:unknown):Result<Compatibility>`（`spikes/sqlite.ts` 与 `src/domain/schemas.ts`）；Compatibility 必填版本、机器、模型、条款确认、读取/队列/网络/资源预算、host timeout、未知 tokenizer 字节上限，未填不能生成 G0 PASS。

- [ ] **Step 1:** 在选定实现根建立 Git 基线与实现分支，复制 spec/计划、提交文档基线；核查 npm 官方元数据及许可证，安装固定版本与 lockfile。建立上述 npm scripts，先让测试运行器成功运行一条纯算术测试，再删掉该临时测试。将版本固定到 `.node-version`/lockfile；无需全局安装 `mem`。
- [ ] **Step 2:** 写测试：`sqlite_capabilities` 断言 WAL、foreign_keys、trigram 为 true，中文“数据库命名”与“ORM”可查；`missing_compatibility_is_blocked` 断言缺任一必填门槛字段返回 `compatibility_incomplete`。

代表性断言（由本任务前置 fixture/临时环境准备输入，放在具名 `test(...)` 中）：

```typescript
const result = await probeSqlite(tempDir);
expect(result).toEqual({ wal: true, foreignKeys: true, trigram: true });
expect(validateCompatibility({})).toEqual({ ok: false, code: 'compatibility_incomplete' });
```

- [ ] **Step 3:** Run `npm test -- tests/integration/compatibility.test.ts`。Expected: RED，缺 probe/schema 行为；不是 Node 无法启动或 runner 配置错误。
- [ ] **Step 4:** 实现探针与 Compatibility schema；用隔离 Claude 测试项目探测真实 SessionStart 输入/输出及 startup/resume/clear/compact/fork、错误退出与 2 秒 timeout。保存脱敏宿主证据到本地 `acceptance/g0/host/`，记录支持版本。核查 parser 终结回答元数据；若无法可靠识别，停止该宿主版本的自动抽取。完成小组件复用搜索（transcript parser、Hook、SQLite lease）与许可证 ADR；不复制未知许可实现。Task 2 尚未完成的模型字段明确 BLOCKED。
- [ ] **Step 5:** Run `npm test -- tests/integration/compatibility.test.ts`、`npm run typecheck`。Expected: 工程测试 PASS，真实宿主分支逐项记 PASS/FAIL/BLOCKED；不能以测试绿替代 G0。
- [ ] **Step 6:** `git add package.json package-lock.json .node-version tsconfig.json vitest.config.ts eslint.config.js .gitignore spikes docs config src/domain/schemas.ts tests/integration/compatibility.test.ts`；`git commit -m "build: establish compatibility probes and pinned toolchain"`。Expected: 提交成功且没有密钥/真实证据。

### Task 2: 冻结样本、模型可行性与数据契约（T01–T02）

**Files:** Create `tests/fixtures/manifest.json`, `tests/fixtures/*.jsonl`, `tests/helpers/fixtures.ts`, `tests/unit/datasets.test.ts`, `evals/label-schema.json`, `evals/dev/cases.jsonl`, `evals/validation/cases.jsonl`, `scripts/check-datasets.ts`, `spikes/model.ts`；Modify `config/compatibility.json`, `docs/adr/001-stack.md`, `docs/adr/002-contracts.md`。

**Interfaces:** Produces `fixture(name:string):Fixture`, `validateDataset(cases:LabelledCase[]):Result<DatasetSummary>`、`runModelProbe(config:Compatibility,cases:LabelledCase[]):Promise<ProbeReport>`。LabelledCase 包含 case/session/nearDuplicateGroup ID、输入消息、应记事实/kind/scope/validFrom/证据和禁止推断；ProbeReport 保存 A/B 原始脱敏结果、人工评分，不自动宣布语义正确。

- [ ] **Step 1:** 写 `dataset_split_leakage`、`private_secret_fixture`、`missing_labels`：断言跨集合共享 session/nearDuplicateGroup 报 `split_leakage`；标签缺证据/负例标志失败；30/20 条计数准确。构造一字/两字中文、emoji 引文、混合块、同文不同 ID、缺 ID、各重写、超限、投毒、冲突、响应错误 fixtures；超大输入运行时流式生成，避免提交大文件。

代表性断言（由本任务前置 fixture/临时环境准备输入，放在具名 `test(...)` 中）：

```typescript
expect(validateDataset(fixture('cross-split-session').cases))
  .toEqual({ ok: false, code: 'split_leakage' });
```

- [ ] **Step 2:** Run `npm test -- tests/unit/datasets.test.ts`。Expected: RED，缺 fixture/校验能力。
- [ ] **Step 3:** 实现 fixture loader 与标签校验；请求用户选定单一免费平台/模型并确认条款、保留/训练政策与外发边界，在未确认前仅使用合成 mock。获批后用最小白名单请求运行 30/20 样例两阶段 spike，人工核验，记录上下文上限、每日额度、100 turn 请求/耗时上限和最长队列等待；固定配置与哈希。冻结操作状态转换表、引文下标、数据 schema、确认入口及备份审计策略。未达质量阈值时输出降级决策，不能继续假设自动抽取可用。
- [ ] **Step 4:** Run `npm test -- tests/unit/datasets.test.ts`、`npx tsx scripts/check-datasets.ts`。Expected: PASS，样本数/集合隔离正确；实机 ProbeReport 单独判定，不把合成 mock 当免费平台质量结果。
- [ ] **Step 5:** `git add tests/fixtures tests/helpers/fixtures.ts tests/unit/datasets.test.ts evals scripts/check-datasets.ts spikes/model.ts config/compatibility.json docs/adr`；`git commit -m "test: freeze fixtures labels and feasibility decisions"`。Expected: 提交成功；真实样本留在明确授权的数据位置，不默认入 Git。

### Task 3: SQLite 权威模型、迁移及 CLI 可运行入口（T03）

**Files:** Create `src/domain/{types,errors}.ts`, `src/storage/{database,migrate,projection}.ts`, `src/storage/migrations/001-initial.sql`, `src/cli/{main,commands}.ts`, `tests/helpers/core.ts`, `tests/integration/storage.test.ts`；Modify `src/domain/schemas.ts`, `package.json`。

**Interfaces:** Produces `openDatabase(path:string):Db`, `migrate(db:Db):void`, `openReadDatabase(path:string):Result<ReadDb>`, `refreshProjection(db:Db,projectId:string):void`；公共类型按 §1。CLI 此时支持 `--help` 与只检查运行环境的 `doctor`。

- [ ] **Step 1:** 写 `foreign_project_rejected`、`revision_append_only`、`migration_rollback`、`read_only_never_creates`。断言跨项目关联 SQL 失败、旧正文不被 UPDATE、迁移故障 schema/version 一起回滚、打开不存在库不会生成文件。

代表性断言（由本任务前置 fixture/临时环境准备输入，放在具名 `test(...)` 中）：

```typescript
expect(openReadDatabase(missingPath).ok).toBe(false);
expect(existsSync(missingPath)).toBe(false);
```

- [ ] **Step 2:** Run `npm test -- tests/integration/storage.test.ts`。Expected: RED，缺 schema/事务。
- [ ] **Step 3:** 实现方案 §6 全部逻辑表；增加 `conflicts`、`candidate_evidence`、`project_controls`、`audit_outbox`、`backup_manifests`，把支持状态/治理版本与正文 revision 分开。所有跨表外键以 `(project_id,id)` 或同等约束验证；messages 用 `(event_id,revision)` 保存不可变版本，关联必须指定版本。FTS 以 active_search 的整数 rowid 建 external-content trigram 表和 INSERT/UPDATE/DELETE 触发器；rebuild/integrity-check 可调用。目录 0700、数据库/配置 0600。Core 迁移前调用 SQLite backup 建一致性快照（Task 18 后共用 backup 模块），Hook 绝不迁移。
- [ ] **Step 4:** Run `npm test -- tests/integration/storage.test.ts`、`npm run build`、`npm run mem -- --help`。Expected: PASS，help 可运行，无业务数据目录意外写入。
- [ ] **Step 5:** `git add src/domain src/storage src/cli tests/helpers/core.ts tests/integration/storage.test.ts package.json`；`git commit -m "feat: add authoritative revisioned storage and cli entry"`。Expected: 提交成功。

**G0 checkpoint:** Task 1–3 的实机/模型/数据契约证据全部有结果且通过才进入主体开发。没 API、没人审标签或宿主探针失败时记录具体 BLOCKED；可以保留工程成果，不能宣布 G0 通过。选择手工记忆降级版本需要另列范围变更，不能在本计划内偷换完整 v1。

## 3. 可靠采集和抽取

### Task 4: 项目授权、唯一写者、IPC 与交互确认（T03）

**Files:** Create `src/core/{projects,writer,ipc,confirmation}.ts`, `tests/integration/core.test.ts`；Modify `src/cli/commands.ts`, `tests/helpers/core.ts`。

**Interfaces:** Consumes `Db`；Produces `resolveProject(db:ReadDb,cwd:string):Result<ProjectRef>`, `startCore(dataDir:string):Promise<CoreHandle>`, `requestCore(dataDir:string,request:CoreRequest):Promise<Result<unknown>>`, `verifyConfirmation(actionHash:string,proof:string):Result<ConfirmationProof>`。CoreRequest 是 operation+cwd+payload 的判别联合，不接受客户端声明的可信 actor。注册、授权变更、批准、手工录入、恢复启用、purge 都要求确认。

- [ ] **Step 1:** 写 `forged_project_id`、`symlink_escape`、`two_core_processes`、`noninteractive_cannot_authorize`、`proof_bound_and_single_use`。断言 B 内容不可读写、两个实际进程仅一个获锁、pipe/普通 IPC 自报 human 失败、过期/重放/换 revision 的凭证失败。

代表性断言（由本任务前置 fixture/临时环境准备输入，放在具名 `test(...)` 中）：

```typescript
expect(verifyConfirmation(actionHash, 'forged-proof'))
  .toEqual({ ok: false, code: 'confirmation_required' });
```

- [ ] **Step 2:** Run `npm test -- tests/integration/core.test.ts`。Expected: RED，缺授权/唯一写入/确认约束。
- [ ] **Step 3:** 实现 `mem init --project`，realpath 路径边界校验；来源只登记明确授权 root/source mapping，移动/worktree 不自动合并。Core 以 UDS socket 持有唯一实例，socket/目录仅用户可访问；锁含 PID/启动身份和随机 instance ID，探活后才回收陈旧锁。数据库写在专用 worker thread；CLI 无 Core 时启动短生命周期 Core，争用同一锁。TTY 确认显示实际操作与目标版本；随机 nonce 有效 60 秒、一次消费、绑定 action hash，确认记录由 Core 写入，`--yes`/环境变量不得绕过审批。
- [ ] **Step 4:** Run `npm test -- tests/integration/core.test.ts`、`npm run typecheck`。Expected: PASS，测试结束无残留 Core 进程/真实用户目录写入。
- [ ] **Step 5:** `git add src/core src/cli/commands.ts tests/helpers/core.ts tests/integration/core.test.ts`；`git commit -m "feat: enforce project authority and single writer confirmations"`。Expected: 提交成功。

### Task 5: 版本化 transcript parser 与持久化前脱敏（T04）

**Files:** Create `src/sources/claude/{adapter,identity,privacy}.ts`, `tests/unit/parser.test.ts`。

**Interfaces:** Produces `parseRecord(raw:unknown,context:SourceContext):ParseResult`, `sanitize(text:string):SanitizedText`, `eventIdentity(record:ParsedRecord,blockIndex:number):string`。SourceContext 含授权项目、来源版本、文件代次/byteStart；ParseResult 以 `status` 判别为 processed/skipped/quarantined/failed，含消息及原因码，不含原始秘密。

- [ ] **Step 1:** 写断言 `mixed_authors => ['person','tool','agent','host','unknown']`、`same_text_distinct_ids => 2 events`、`subagent => skipped/subagent_disabled`；`private_and_key` 检查 serialize(result) 不含两个秘密；`quoted_and_managed` 检查 quoted/managed 不取得 direct 标签；`unicode_quote_offsets` 验证 emoji 前后引文按 code points 匹配。

代表性断言（由本任务前置 fixture/临时环境准备输入，放在具名 `test(...)` 中）：

```typescript
const f = fixture('mixed-user-blocks');
const result = parseRecord(f.raw, f.context);
expect(result.status).toBe('processed');
if (result.status === 'processed') expect(result.messages.map(m => m.author)).toEqual(['person', 'tool']);
```

- [ ] **Step 2:** Run `npm test -- tests/unit/parser.test.ts`。Expected: RED，缺 parser/隐私边界。
- [ ] **Step 3:** 以 Task 1 固定宿主 schema 实现按块解析。身份由 source/session/subagent/sourceMessageId/blockIndex 组合，内容哈希只识别修改；缺 ID 使用 file generation/byteStart/blockIndex，不做文本全局去重。结构化宿主包装优先，private 标签不完整时保守隐藏剩余片段并诊断。不持久化原始 transcript、工具 payload 或未脱敏模型诊断。
- [ ] **Step 4:** Run `npm test -- tests/unit/parser.test.ts`。Expected: PASS；author 与 role 分开，未知格式可诊断，不冒充 person。
- [ ] **Step 5:** `git add src/sources tests/unit/parser.test.ts`；`git commit -m "feat: normalize transcript blocks with private evidence boundaries"`。Expected: 提交成功。

### Task 6: 增量 reader、重写检测与采集原子提交（T04）

**Files:** Create `src/ingest/{reader,fingerprints,ledger,watcher}.ts`, `tests/integration/ingest.test.ts`；Modify `src/cli/commands.ts`, `tests/helpers/core.ts`。

**Interfaces:** Consumes `parseRecord`, Core writer；Produces `importSource(core:CoreHandle,sourceId:string,options:{dryRun:boolean,since?:number}):Promise<ImportReport>`, `scanAuthorized(core:CoreHandle):Promise<ScanReport>`。ImportReport 包含完整记录各分类计数、消息新增数 `newMessages`、committedOffset、generation、isolatedRanges；dry-run 无写入/模型请求。

- [ ] **Step 1:** 写 `half_line_offset_unchanged`、`rescan_zero_new`、`identical_text_two_events`、`rewrite_matrix`、`record_limit_streaming`、`watch_loss_rescan`、`ingest_crash_transaction`。断言半行 offset 不前进；第 2/3 次重扫新增 0；4 类改写和缺 ID 歧义被检测/隔离；>32 MiB 输入和 >8 MiB 单记录之后的正常记录均有账本；完整输入数等于处理分类和。

代表性断言（由本任务前置 fixture/临时环境准备输入，放在具名 `test(...)` 中）：

```typescript
const first = await importSource(core, sourceId, { dryRun: false });
const second = await importSource(core, sourceId, { dryRun: false });
expect(first.newMessages).toBe(2);
expect(second.newMessages).toBe(0);
expect(second.committedOffset).toBe(first.committedOffset);
```

- [ ] **Step 2:** Run `npm test -- tests/integration/ingest.test.ts`。Expected: RED，缺 reader/事务/指纹。
- [ ] **Step 3:** 流式读取，每轮 ≤4 MiB，可跨轮继续同一记录至 8 MiB；超限后流式跳到换行并记录范围。账本、消息、关联和 offset 一个事务；变更前缀/锚点触发 generation，宿主 ID 重放去重，缺 ID 只在顺序/邻近锚点一致时关联。保存可核对的分块指纹，300 秒核查已读前缀，不能只查最后一个锚点。实现启动/30 秒补扫、fs.watch 事件合并、`import --dry-run/--since 7d`，先解析归属再持久化正文；只有明确授权来源可打开正文。
- [ ] **Step 4:** Run `npm test -- tests/integration/ingest.test.ts`。Expected: PASS；采集事务提交前/后各 kill 子进程 3 次仍幂等；补扫 ≤35 秒发现新记录（Task 1 若调整，使用冻结裕量）。源 fixture 文件哈希始终不变。
- [ ] **Step 5:** `git add src/ingest src/cli/commands.ts tests/helpers/core.ts tests/integration/ingest.test.ts`；`git commit -m "feat: ingest incrementally with durable ledger and rewrite detection"`。Expected: 提交成功。

### Task 7: 完整 turn 与输入覆盖账本（T05）

**Files:** Create `src/ingest/turns.ts`, `tests/unit/turns.test.ts`。

**Interfaces:** Consumes `Message[]`；Produces `reduceTurns(messages:Message[],previous:Turn[]):Turn[]`, `planBatches(turns:Turn[],budget:InputBudget,now:number,manual:boolean):BatchPlan`。Turn 有 boundary=open/closed、completion=open/unanswered/complete、revision；BatchPlan 明列 covered/pending/blocked 及字节/证据区间，不删除未覆盖尾部。

- [ ] **Step 1:** 写 `unanswered_closed_not_eligible`、`tools_without_final_not_complete`、`late_result_new_revision`、`flush_five_idle_manual`、`batch_max_twenty`、`oversized_turn_blocked_with_tail`。断言下一用户出现仅关闭边界；5 个完整轮/300000 ms/手工各可 flush，21 轮拆成 ≤20；超大轮尾部标记仍在未处理覆盖表中。

代表性断言（由本任务前置 fixture/临时环境准备输入，放在具名 `test(...)` 中）：

```typescript
const f = fixture('unanswered-then-next-user');
const turns = reduceTurns(f.messages, []);
expect(turns[0]).toMatchObject({ boundary: 'closed', completion: 'unanswered' });
```

- [ ] **Step 2:** Run `npm test -- tests/unit/turns.test.ts`。Expected: RED，缺 turn 状态机。
- [ ] **Step 3:** 实现归约与确定性输入哈希；只采纳 spike 确认的终结元信息。首版过大单 turn 使用 `blocked/input_too_large` 人工处理，不实现复杂窗口自动合并。idle 不写会话 ended_at，迟到数据创建 revision 并替代尚未提交的旧任务输入。
- [ ] **Step 4:** Run `npm test -- tests/unit/turns.test.ts`。Expected: PASS，三列状态与 AC-12 完全一致。
- [ ] **Step 5:** `git add src/ingest/turns.ts tests/unit/turns.test.ts`；`git commit -m "feat: assemble complete turns and track input coverage"`。Expected: 提交成功。

### Task 8: 持久队列、租约、预算与后台服务（T05）

**Files:** Create `src/jobs/{queue,worker,budget}.ts`, `src/core/service.ts`, `tests/integration/jobs.test.ts`；Modify `src/cli/commands.ts`, `tests/helpers/core.ts`。

**Interfaces:** Consumes `planBatches`, Core；Produces `enqueue(core:CoreHandle,batch:Batch):Result<JobRef>`, `claim(core:CoreHandle,now:number):Result<Lease>`, `commitJob(core:CoreHandle,lease:Lease,snapshot:InputSnapshot,verdicts:Verdict[]):Result<void>`, `runService(core:CoreHandle,processor:BatchProcessor):Promise<void>`。Lease 含单调 fencing token/到期时间；InputSnapshot 含项目 generation、证据/turn revisions、被比较治理版本；BatchProcessor 在 Task 11 提供。

- [ ] **Step 1:** 写 `input_hash_pipeline_unique`、`lease_expiry_old_commit_rejected`、`revoke_during_work`、`daily_and_disk_budget`。断言同输入重复入队一个 job；旧租约提交失败；项目撤权无候选写入；200 批/日（UTC 日界，本计划选择）、2 GiB 上限停相应任务且不推进采集游标，恢复后续跑。

代表性断言（由本任务前置 fixture/临时环境准备输入，放在具名 `test(...)` 中）：

```typescript
const f = fixture('expired-lease-after-reclaim');
expect(commitJob(core, f.oldLease, f.snapshot, []))
  .toEqual({ ok: false, code: 'stale_lease' });
```

- [ ] **Step 2:** Run `npm test -- tests/integration/jobs.test.ts`。Expected: RED，缺持久状态/预算。
- [ ] **Step 3:** 实现 pending/running/completed/retry_wait/blocked/failed/cancelled，reasonCode 独立；lease 60 秒/15 秒续租、并发 1。所有提交必须检验 fencing token 和快照，候选/证据关联/job 完成同事务。`mem serve` 启动监听与调度，`mem flush` 只调度完整轮，`extract --pending` 启动受相同预算约束的处理；历史回填默认暂停，前台增量优先。worker 崩溃自动回收，预算熔断需可解释恢复条件。
- [ ] **Step 4:** Run `npm test -- tests/integration/jobs.test.ts`。Expected: PASS；模型返回后/提交事务中/完成后各 kill 3 次恢复，至多一个逻辑提交；Core 停止不影响以后 Hook 的只读能力。
- [ ] **Step 5:** `git add src/jobs src/core/service.ts src/cli/commands.ts tests/helpers/core.ts tests/integration/jobs.test.ts`；`git commit -m "feat: add fenced persistent extraction queue and service"`。Expected: 提交成功。

### Task 9: 单端点模型 transport 与实际请求预览（T06）

**Files:** Create `src/extraction/{request,transport}.ts`, `tests/integration/transport.test.ts`；Modify `src/cli/commands.ts`。

**Interfaces:** Produces `buildRequest(batch:Batch,stage:'generate'|'review',extra:unknown):PreparedRequest`, `sendModel(request:PreparedRequest,config:ModelConfig,signal:AbortSignal):Promise<ModelResult>`；ModelResult 使用 `Result` 判别结构。PreparedRequest 为明确 body 字段白名单，不含 header/secret/raw transcript；预览与实际请求共用同一序列化函数。

- [ ] **Step 1:** 写 `preview_matches_wire_body`、`redirect_denied`、`proxy_env_ignored`、`auth_and_quota_circuit`、`retry_classification`、`no_secret_in_error`。断言 body 逐字段一致；跨 host 和同 host 重定向均不跟随；HTTP_PROXY/HTTPS_PROXY/ALL_PROXY 不改变路由；401/403/配额熔断；5xx/断连含首次 ≤3 次；429 不早于 Retry-After；错误不含密钥。

代表性断言（由本任务前置 fixture/临时环境准备输入，放在具名 `test(...)` 中）：

```typescript
const result = await sendModel(fixture('minimal-request').request, redirectConfig, signal);
expect(result).toMatchObject({ ok: false, code: 'redirect_denied' });
expect(redirectTarget.requests).toHaveLength(0);
```

- [ ] **Step 2:** Run `npm test -- tests/integration/transport.test.ts`。Expected: RED，缺 transport 安全与退避。
- [ ] **Step 3:** 用 Node `https.request` 显式发送到配置 endpoint origin+固定 API path，拒绝 URL userinfo、非 HTTPS 生产端点、动态候选 URL 和全部 3xx；不使用从环境构造的代理 Agent。测试 HTTP loopback 仅由测试进程注入 transport，不提供生产绕过选项。密钥仅从 0600 本地配置或系统安全存储读取。请求超时/响应大小采用 G0 冻结限制，记录计数不记录正文。429 有次数/累计等待上限（采用冻结队列等待预算），超限 blocked；网络与格式修复共享全局请求/日预算，不能嵌套无限重试。加入 `mem context --show-request --job <id>`；无 job 时仅预览下一批待处理输入，空队列明确返回无请求。
- [ ] **Step 4:** Run `npm test -- tests/integration/transport.test.ts`。Expected: PASS；断网导入/治理/召回仍可用，模型 blocked/model_unavailable；测试服务请求记录没有其他目标。
- [ ] **Step 5:** `git add src/extraction/request.ts src/extraction/transport.ts src/cli/commands.ts tests/integration/transport.test.ts`；`git commit -m "feat: constrain model transport and preview minimized requests"`。Expected: 提交成功。

### Task 10: Stage A schema、逐字证据与负例过滤（T06）

**Files:** Create `src/extraction/candidates.ts`, `prompts/generate-v1.md`, `tests/unit/candidates.test.ts`。

**Interfaces:** Consumes `CandidateDraft`, `EvidenceRef`, `Batch`；Produces `validateCandidates(output:unknown,batch:Batch):Result<CandidateDraft[]>`；程序能检查引用/作者/范围，语义是否支持在 Stage B 与人工评测，不声称正则解决语义真实性。

- [ ] **Step 1:** 写 `max_ten_candidates`、`legal_empty_success`、`wrong_project_quote_revision`、`agent_quote_not_user_rule`、`invalid_kind_or_length`。断言 11 条失败、[] 成功；存在但错项目/版本/偏移/引文的证据失败；direct person 之外不能作为唯一授权证据；超长/负值/未知枚举失败。

代表性断言（由本任务前置 fixture/临时环境准备输入，放在具名 `test(...)` 中）：

```typescript
expect(validateCandidates([], fixture('valid-batch').batch))
  .toEqual({ ok: true, value: [] });
expect(validateCandidates(fixture('eleven-candidates').output, fixture('valid-batch').batch).ok).toBe(false);
```

- [ ] **Step 2:** Run `npm test -- tests/unit/candidates.test.ts`。Expected: RED，缺 schema/引用检查。
- [ ] **Step 3:** 实现 Zod 输出与完整批次证据核验，正文与 title 限制遵循公共契约；UTF-8 字节预算与 code point 引文下标分开。prompt 只抽三类，保留不确定项；JSON 修复最多一次，合法空结果 completed，持续非法 failed/invalid_output；诊断必须先脱敏。
- [ ] **Step 4:** Run `npm test -- tests/unit/candidates.test.ts`。Expected: PASS，任何非法候选均不能进入 pending_review。
- [ ] **Step 5:** `git add src/extraction/candidates.ts prompts/generate-v1.md tests/unit/candidates.test.ts`；`git commit -m "feat: validate candidate schema and exact supporting evidence"`。Expected: 提交成功。

### Task 11: Stage B 全量比对及新鲜快照提交（T06）

**Files:** Create `src/extraction/{review,pipeline}.ts`, `src/governance/conflicts.ts`, `prompts/review-v1.md`, `tests/integration/extraction.test.ts`；Modify `src/core/service.ts`。

**Interfaces:** Consumes `sendModel`, `validateCandidates`, `commitJob`；Produces `planReview(candidates:CandidateDraft[],evidence:Message[],active:ActiveMemory[],budget:InputBudget):Result<ReviewShard[]>`（每个分片含 `memoryRevisionIds:string[]`）, `reduceVerdicts(shards:ReviewedShard[]):Result<Verdict[]>`, `processBatch(batch:Batch,context:PipelineContext):Promise<PipelineResult>`、`stageConflicts(db:Db,projectId:string,verdicts:Verdict[]):void`（只由 commitJob 事务调用；Task 12 扩展同文件做人工裁决）。

- [ ] **Step 1:** 写 `stage_b_has_evidence_and_author`、`conflict_beyond_fiftieth_memory`、`missing_shard_blocks_commit`、`unsupported_quote_not_accepted`、`late_evidence_or_generation_rejected`。断言第 51 条旧规则参与比对；分片未覆盖所有 active 时不可提交；ID 存在但证据不支持的固定语义负例被退回/丢弃；模型在途撤权/删证据/改目标治理版本时无新当前候选。

代表性断言（由本任务前置 fixture/临时环境准备输入，放在具名 `test(...)` 中）：

```typescript
const f = fixture('fifty-one-active-memories');
const planned = planReview(f.candidates, f.messages, f.active, f.budget);
expect(planned.ok).toBe(true);
if (planned.ok) expect(new Set(planned.value.flatMap(s => s.memoryRevisionIds)).size).toBe(51);
```

- [ ] **Step 2:** Run `npm test -- tests/integration/extraction.test.ts`。Expected: RED，缺 B 比对/版本提交。
- [ ] **Step 3:** Stage B 输入含脱敏证据、作者、必要上下文和全部 active 记忆预算分片；汇总冲突优先，目标跨项目/不存在/版本过期拒绝；分片间矛盾裁决转人工处理。预算不足明确 blocked，不取前 50。模型只能提出 Verdict，不能批准；通过 Core 事务提交 pending 候选，冲突关系暂停相应旧条目并审计。调用期间不持 SQLite 写事务；提交前重查所有快照。
- [ ] **Step 4:** Run `npm test -- tests/integration/extraction.test.ts`。Expected: PASS；mock 证明结构/竞态；真实 API 语义负例与质量在 Task 20 单独计分，mock 不代表语义正确率。
- [ ] **Step 5:** `git add src/extraction/review.ts src/extraction/pipeline.ts src/governance/conflicts.ts prompts/review-v1.md src/core/service.ts tests/integration/extraction.test.ts`；`git commit -m "feat: review candidates against evidence and complete memory snapshots"`。Expected: 提交成功。

## 4. 治理、检索和 CLI MVP（G1）

### Task 12: 人工审阅、不可变 revision、冲突与抑制（T07）

**Files:** Create `src/governance/{review,revisions,suppression,evidence}.ts`, `src/cli/review.ts`, `tests/integration/governance.test.ts`, `tests/e2e/review-cli.test.ts`；Modify `src/governance/conflicts.ts`, `src/storage/projection.ts`, `src/extraction/pipeline.ts`, `src/cli/commands.ts`。

**Interfaces:** Consumes `ReviewCommand`, `ConfirmationProof`, Core；Produces `applyReview(core:CoreHandle,command:ReviewCommand,proof:ConfirmationProof):Result<ReviewOutcome>`, `deleteEvidence(core:CoreHandle,ref:EvidenceRef,proof:ConfirmationProof):Result<void>`, `isSuppressed(db:ReadDb,projectId:string,draft:CandidateDraft):boolean`。ReviewOutcome 含新 revision/governanceVersion、状态、剩余冲突 ID；重新启用需有效支持及新的人工批准。

- [ ] **Step 1:** 写 `only_human_confirmed_becomes_active`、`merge_immutable_until_approved`、`supersede_atomic_rollback`、`stale_governance_rejected`、`reject_one_of_two_conflicts_stays_paused`、`delete_evidence_invalidates_pending`、`revoke_pipeline_upgrade_still_suppressed`、`new_evidence_shows_history`。断言 pending/rejected/revoked/needs_review 不进有效投影；同正文但治理版本已变也拒绝；删最后支持后 active→needs_review；新证据只能 pending，不能恢复旧 active。

代表性断言（由本任务前置 fixture/临时环境准备输入，放在具名 `test(...)` 中）：

```typescript
const result = applyReview(core, staleCommand, validConfirmation);
expect(result).toEqual({ ok: false, code: 'stale_governance' });
```

- [ ] **Step 2:** Run `npm test -- tests/integration/governance.test.ts`、`npm test -- tests/e2e/review-cli.test.ts`。Expected: RED，缺治理状态机/交互入口。
- [ ] **Step 3:** 实现审批、拒绝、修改、merge/supersede、冲突裁决、撤回和证据删除。审批同事务核验授权、候选/目标 revision、governanceVersion、证据支持、确认凭证；更新审计、指针、投影、FTS、project generation。候选状态补 `stale`，与方案 §8.3 一致。在 pipeline 的 Stage A 后和最终提交事务中都调用 isSuppressed；抑制键不包含 pipelineVersion，覆盖旧证据与候选 fingerprint；若新证据集合包含独立新事实，允许新待审候选但展示历史。候选修改后重展示证据；无证据新增正文只走 manual 来源并单独审计。实现 `mem review`、`mem revoke <id>`、`mem add --manual`、`mem evidence delete <id>`；TTY 审阅逐条显示作者/项目/事实时间/支持引文/旧版 diff。
- [ ] **Step 4:** Run 两个 Step 2 命令及 `npm run typecheck`。Expected: PASS；真实 SQLite 事务中途失败/进程终止各 3 次无半更新；普通 pipe/Agent 直接 IPC 不能批准、手工入库或改变授权。提示撤回不能收回旧会话已注入文本。
- [ ] **Step 5:** `git add src/governance src/cli src/storage/projection.ts src/extraction/pipeline.ts tests/integration/governance.test.ts tests/e2e/review-cli.test.ts`；`git commit -m "feat: add human gated revision governance and suppression"`。Expected: 提交成功。

### Task 13: 中文字面检索、有效视图和索引修复（T08）

**Files:** Create `src/recall/search.ts`, `tests/fixtures/search-cases.json`, `tests/integration/search.test.ts`；Modify `src/cli/commands.ts`。

**Interfaces:** Consumes `ReadDb`, `resolveProject`；Produces `search(db:ReadDb,project:ProjectRef,query:string,limit:number):SearchHit[]`, `show(db:ReadDb,project:ProjectRef,id:string,view:'active'|'history'|'review'):Result<MemoryDetail>`, `rebuildSearch(core:CoreHandle,projectId:string):Result<void>`。SearchHit 含 id/revision/kind/title，MemoryDetail 明确治理状态与来源；默认只读 active。

- [ ] **Step 1:** 写 `literal_query_matrix`、`stale_fts_cannot_bypass_state`、`update_removes_old_terms`、`rebuild_existing_rows`。固定 ≥20 查询，含“数据库命名”“约束”“库”“ORM”“src/db.ts”、双引号/星号/括号/AND/OR、空串、无匹配；断言预期 ID、无越权/语法异常、Top-5 ≥90%，空查询返回空，limit 封顶 50。

代表性断言（由本任务前置 fixture/临时环境准备输入，放在具名 `test(...)` 中）：

```typescript
expect(search(db, projectA, '约束', 5).map(hit => hit.id)).toEqual(['m_rule']);
expect(search(db, projectA, '完全不存在', 5)).toEqual([]);
```

- [ ] **Step 2:** Run `npm test -- tests/integration/search.test.ts`。Expected: RED，缺字面查询/治理联查。
- [ ] **Step 3:** ≥3 Unicode 字符编译为 FTS literal phrase（双引号转义），<3 用参数化 instr；长输入上限 256 code points，超限返回输入错误。查询按 project/status/current revision 再过滤，排序固定、同分按 ID。重建后执行 FTS external-content `integrity-check`（rank=1）并核对集合。实现 `mem search`、`mem show` 与显式 `--history/--review`；只返回当前授权项目，默认详情不泄露失效正文。
- [ ] **Step 4:** Run `npm test -- tests/integration/search.test.ts`。Expected: PASS，模型调用数 0，撤回事务后新连接立即看不到旧内容。
- [ ] **Step 5:** `git add src/recall/search.ts src/cli/commands.ts tests/fixtures/search-cases.json tests/integration/search.test.ts`；`git commit -m "feat: search approved memories with literal multilingual queries"`。Expected: 提交成功。

### Task 14: 完整约束短句与可解释预算（T08）

**Files:** Create `src/recall/{context,budget}.ts`, `tests/unit/context.test.ts`；Modify `src/cli/commands.ts`。

**Interfaces:** Consumes `ActiveMemory[]`；Produces `renderContext(memories:ActiveMemory[],project:ProjectRef,budget:ContextBudget):ContextResult`。ContextBudget 为 verified-tokenizer（有本地匹配模型版本/count 函数，上限 800）或 byte-estimate（UTF-8 总上限 800）；ContextResult 含 text、selected/excluded IDs、generation、单位、measuredSize、estimated、overflowCount。

- [ ] **Step 1:** 写 `full_negation_or_whole_omission`、`five_full_rules_plus_decision_index`、`wrapper_in_budget`、`unknown_tokenizer_is_estimate`。断言长“不允许…除非…”绝不截成反义；约束/约定完整最多 5 条，决定索引含 ID/revision；说明/溢出提示也计预算；有验证 tokenizer 时 count(text)≤800，未知时 Buffer.byteLength(text)≤800 且 estimated=true。

代表性断言（由本任务前置 fixture/临时环境准备输入，放在具名 `test(...)` 中）：

```typescript
const result = renderContext(memories, projectA, { mode: 'byte-estimate', limit: 800 });
expect(Buffer.byteLength(result.text, 'utf8')).toBeLessThanOrEqual(800);
expect(result.estimated).toBe(true);
```

- [ ] **Step 2:** Run `npm test -- tests/unit/context.test.ts`。Expected: RED，缺预算与稳定渲染。
- [ ] **Step 3:** 先按 scope 与有效时间过滤，再 kind/用户重要性/时间/ID 稳定排序，不以 confidence 自动排序。预留固定标头、历史参考说明和溢出提示预算，再装入条目；空结果不制造“已确认记忆”。输出受管标记，parser 可识别其 provenance；`mem context --preview` 与 Hook 共用此函数。未知 tokenizer 800 字节只承诺保守估算，不承诺跨模型硬 token 上限。
- [ ] **Step 4:** Run `npm test -- tests/unit/context.test.ts`。Expected: PASS，原始证据/工具内容不在文本内；重要约束溢出在预览和 doctor 明示。
- [ ] **Step 5:** `git add src/recall/context.ts src/recall/budget.ts src/cli/commands.ts tests/unit/context.test.ts`；`git commit -m "feat: render bounded context without truncating rule meaning"`。Expected: 提交成功。

### Task 15: 有截止的只读 Hook 与精确安装卸载（T08）

**Files:** Create `src/integration/{hook-entry,hook-worker,settings,install}.ts`, `tests/e2e/hook.test.ts`, `tests/integration/install.test.ts`, `docs/compatibility.md`；Modify `src/cli/commands.ts`。

**Interfaces:** Consumes `openReadDatabase`, `resolveProject`, `renderContext`；Produces `handleHook(input:unknown,dataDir:string):Promise<HookOutput|null>`, `planInstall(settings:unknown,command:HookCommand):InstallPlan`, `applyInstall(plan:InstallPlan,expectedFingerprint:string):Result<InstallManifest>`, `uninstall(manifest:InstallManifest,current:unknown):UninstallResult`。HookOutput 精确为 `{hookSpecificOutput:{hookEventName:'SessionStart',additionalContext:string}}`；InstallManifest 记录归属/前后 fingerprint/自己插入项。

- [ ] **Step 1:** 写 `core_offline_read_ok`、`unknown_schema_or_project_empty`、`blocking_sql_deadline`、`four_sources_and_fork`、`install_idempotent_preserves_others`、`edited_entry_not_removed`、`space_quote_path`、`concurrent_settings_change`。断言真实新进程异常时 exit 0/无上下文；并发 settings 修改返回冲突且不覆盖；两次 install 只有一项；uninstall 不还原整文件。

代表性断言（由本任务前置 fixture/临时环境准备输入，放在具名 `test(...)` 中）：

```typescript
expect(await handleHook({ ...validHookInput, source: 'fork' }, dataDir)).toBeNull();
expect(await handleHook(validHookInput, missingDataDir)).toBeNull();
```

- [ ] **Step 2:** Run `npm test -- tests/e2e/hook.test.ts`、`npm test -- tests/integration/install.test.ts`。Expected: RED，缺 Hook/配置合并。
- [ ] **Step 3:** entry 只校验 stdin 并启动只读 worker thread；同步 SQLite 在 worker 内，busy timeout 设 0，entry 独立 500 ms deadline 可终止 worker，避免同线程 setTimeout 被 SQL 阻塞。未知 schema/损坏/错项目/超时返回空，stdout 只输出协议 JSON，诊断走 stderr/受限日志。只接受 SessionStart 和四种 source；fork 不自动接入。
- [ ] **Step 4:** 在 `.claude/settings.local.json` 合并唯一自有 command handler，matcher 固定 `startup|resume|clear|compact`，宿主 timeout 固定 2 秒（Task 1 确认）；不设 async。兼容 exec-form args 则使用独立 argv，否则按已验证 shell quoting 构造绝对路径，空格/引号必须测试。安装使用比较 fingerprint 后原子 rename，未知/非法配置直接拒绝并保留原文件。实现 `mem install --host claude-code --dry-run`、显式 install、`mem uninstall`、`mem disable-injection`；停用开关由 Hook 只读检查，无须数据库迁移/写入。source fork 的准确识别是版本 gate，不能只以 matcher 掩盖旧版本差异。
- [ ] **Step 5:** Run Step 2 两命令及隔离真实宿主生命周期探针。Expected: 自动测试 PASS；startup/resume/clear/compact 输出当前版本，fork 不运行新 Hook；宿主超时后能继续任务。若当前宿主不能验证则 AC-33/34 BLOCKED，不宣称 G1。
- [ ] **Step 6:** `git add src/integration src/cli/commands.ts tests/e2e/hook.test.ts tests/integration/install.test.ts docs/compatibility.md`；`git commit -m "feat: integrate bounded read only context hook safely"`。Expected: 提交成功。

**G1 checkpoint:** AC-01、AC-02–39、AC-41a/41b 对 CLI 路径成立，工程质量与相应搜索/Hook性能有记录；真实 API 的候选质量报告必须附上，不能只有模拟演示。允许一个授权项目 CLI MVP 试用；Web/恢复/长期试点尚未执行时不得签收 G2。

## 5. 完整 v1 与运维（G2）

### Task 16: 安全的本机 Web 审阅（T09）

**Files:** Create `src/review/{server,security}.ts`, `src/review/public/{index.html,app.js,style.css}`, `tests/e2e/review-web.spec.ts`, `tests/integration/web-security.test.ts`, `playwright.config.ts`；Modify `src/cli/commands.ts`。

**Interfaces:** Consumes `applyReview` 和 Core 确认服务；Produces `startReviewServer(core:CoreHandle):Promise<{origin:string,close:()=>Promise<void>}>`。GET `/api/candidates`、GET `/api/candidates/:id`；POST `/api/review` 使用 ReviewCommand + 一次性确认凭证；POST `/api/pair` 接收用户在终端看到的配对码后签发 HttpOnly/SameSite=Strict 会话 cookie，CSRF token 经已配对会话获取。

- [ ] **Step 1:** 写 Playwright `approve_reject_edit_resolve`、`two_tabs_stale`、`evidence_is_text`；HTTP `bad_host_origin_csrf_token`、`loopback_only`、`pair_and_confirmation_replay`。断言 `<script>`/HTML 仅文本；所有非法请求无 DB 状态改变；另一个窗口陈旧批准 409；确认码不能批准另一 revision。

代表性断言（由本任务前置 fixture/临时环境准备输入，放在具名 `test(...)` 中）：

```typescript
await page.getByTestId('candidate-evidence').waitFor();
expect(await page.getByTestId('candidate-evidence').textContent()).toContain('<script>');
expect(await page.evaluate(() => document.querySelectorAll('[data-testid="candidate-evidence"] script').length)).toBe(0);
```

- [ ] **Step 2:** Run `npm test -- tests/integration/web-security.test.ts`、`npx playwright test tests/e2e/review-web.spec.ts`。Expected: RED，缺服务/页面。
- [ ] **Step 3:** 实现三块页面（候选列表、证据详情、操作区），显示作者、时间、范围、支持片段与旧版差异；使用 textContent，不支持证据 HTML。仅绑定 127.0.0.1 随机端口，Host 精确含端口；写请求校验 Origin、会话、CSRF、Content-Type/大小限制。无 CORS、无公网监听、无 URL token；会话配对后每次批准仍显示并确认具体内容，nonce 绑定 action/revision、60 秒一次性。UI 仅调用 Core，不能直接写 SQLite。
- [ ] **Step 4:** Run Step 2 两命令。Expected: PASS；正常 CLI/Web 相同操作产生同一状态转换，跨站请求无更改。`mem review --web` 可打开审阅入口但配对/批准仍由人完成。
- [ ] **Step 5:** `git add src/review src/cli/commands.ts tests/e2e/review-web.spec.ts tests/integration/web-security.test.ts playwright.config.ts`；`git commit -m "feat: add protected local evidence review interface"`。Expected: 提交成功。

### Task 17: doctor、导出、日志与独立控制开关（T10）

**Files:** Create `src/ops/{doctor,export,logging}.ts`, `tests/integration/operations.test.ts`, `docs/privacy.md`；Modify `src/cli/commands.ts`, `src/core/service.ts`。

**Interfaces:** Produces `diagnose(db:ReadDb,config:Compatibility):DoctorReport`, `exportMemories(db:ReadDb,project:ProjectRef,options:{format:'jsonl'|'markdown',includeEvidence:boolean}):AsyncIterable<string>`, `setControl(core:CoreHandle,component:'ingest'|'extract'|'inject',enabled:boolean):Result<void>`；DoctorReport 的 `sourceHealth` 列表含 failed/supported/extractionPaused；每项含 status/reasonCode/分子分母/nextAction，不包含证据正文。

- [ ] **Step 1:** 写 `drift_six_not_five_of_hundred`、`insufficient_sample_not_healthy`、`export_no_secrets_or_evidence_by_default`、`logs_total_cap`、`independent_controls`。断言 6/100 暂停文件抽取，5/100 不因阈值暂停；合法 skip 不进分母；20 MiB 是含轮转文件的总量；三个开关彼此独立。

代表性断言（由本任务前置 fixture/临时环境准备输入，放在具名 `test(...)` 中）：

```typescript
const report = diagnose(dbWithSixFailuresOfHundred, config);
expect(report.sourceHealth[0]).toMatchObject({ failed: 6, supported: 100, extractionPaused: true });
```

- [ ] **Step 2:** Run `npm test -- tests/integration/operations.test.ts`。Expected: RED，缺诊断/导出/控制。
- [ ] **Step 3:** 实现 doctor 所有 spec §11 字段，未知类型立即警告，最近 100 条支持记录的 >5% 规则在服务中实际生效。默认日志只记录 ID/阶段/耗时/计数/原因码；错误 cause 也需脱敏。导出含 schema/revision/治理状态/证据引用，只有 `--include-evidence` 才导出脱敏正文，永不导出密钥。实现 `mem export`、`mem control <component> on|off`；恢复注入 on 要求确认，检查恢复/撤回审计无未解决阻塞。
- [ ] **Step 4:** Run `npm test -- tests/integration/operations.test.ts`。Expected: PASS，所有管线产物包含的合成 secret 检查为 0；导出不提供反向写数据库入口。
- [ ] **Step 5:** `git add src/ops src/cli/commands.ts src/core/service.ts tests/integration/operations.test.ts docs/privacy.md`；`git commit -m "feat: diagnose stalled work and export governed records safely"`。Expected: 提交成功。

### Task 18: 一致性备份、独立撤回审计和安全恢复（T10）

**Files:** Create `src/ops/{backup,restore}.ts`, `tests/integration/recovery.test.ts`, `docs/operations.md`；Modify `src/storage/migrate.ts`, `src/governance/review.ts`, `src/cli/commands.ts`。

**Interfaces:** Produces `createBackup(core:CoreHandle,destination:string):Promise<BackupManifest>`, `restoreBackup(core:CoreHandle,manifest:BackupManifest,journal:AuditJournal|null,proof:ConfirmationProof):Promise<Result<RestoreReport>>`。RestoreReport 含 injectionEnabled/activeCount；BackupManifest 含 schema/DB hash/project IDs/审计序号；审计保存在当前 dataDir 下独立于待恢复 DB 的 append-only journal，包含完整性链和 checkpoint，不含正文。

- [ ] **Step 1:** 写 `wal_consistent_snapshot`、`revoke_after_backup_replayed`、`missing_or_gap_journal_needs_review`、`crash_between_revoke_and_journal`、`restore_does_not_reenable_injection`、`schema_rollback_compatible`。断言恢复时先关闭注入；journal 有完整后续撤回则重放，无/缺口则恢复 active 全部 needs_review；DB foreign_key_check/integrity_check 通过。

代表性断言（由本任务前置 fixture/临时环境准备输入，放在具名 `test(...)` 中）：

```typescript
const result = await restoreBackup(core, backup, null, confirmation);
expect(result.ok).toBe(true);
if (result.ok) expect(result.value).toMatchObject({ injectionEnabled: false, activeCount: 0 });
```

- [ ] **Step 2:** Run `npm test -- tests/integration/recovery.test.ts`。Expected: RED，缺在线备份和恢复屏障。
- [ ] **Step 3:** 使用 driver 在线 backup，禁止复制活跃 WAL 主文件冒充快照。撤回事务先写 audit_outbox；可靠刷独立 journal 后推进高水位，崩溃缺口一律按不完整审计处理。恢复停止写者/任务、持有同一实例锁，先创建恢复前备份和外部停用标志，再验证 manifest/checkpoint/链连续性，恢复兼容 schema 后重放；不能因 journal 尾部被截断而猜测完整。若无法证明全部撤回已涵盖，统一待复核。恢复是新 runtime epoch，旧租约与在途结果全部失效；人工重新启用注入之前运行 doctor。
- [ ] **Step 4:** Run `npm test -- tests/integration/recovery.test.ts`。Expected: PASS，真实 SQLite 及恢复子进程各故障点 ≥3 次；提供/缺失/截断/篡改 journal 分支都有记录。文档明示仅抵御意外恢复和常规调用，不抵御同账号恶意改全套文件。
- [ ] **Step 5:** `git add src/ops/backup.ts src/ops/restore.ts src/storage/migrate.ts src/governance/review.ts src/cli/commands.ts tests/integration/recovery.test.ts docs/operations.md`；`git commit -m "feat: restore backups without reviving revoked memories"`。Expected: 提交成功。

### Task 19: 清空受管恢复路径与可选 LaunchAgent（T10）

**Files:** Create `src/ops/purge.ts`, `tests/integration/purge.test.ts`；可选 Create `src/ops/launch-agent.ts`, `tests/integration/launch-agent.test.ts`；Modify `src/cli/commands.ts`, `src/integration/install.ts`, `docs/operations.md`。

**Interfaces:** Produces `planPurge(core:CoreHandle,projectId:string):PurgeManifest`, `executePurge(core:CoreHandle,manifest:PurgeManifest,proof:ConfirmationProof):Promise<Result<PurgeReport>>`, `planLaunchAgent(executable:string,dataDir:string):ServicePlan`。PurgeManifest 含 projectId 和 objects（每项 kind/path）；列明 DB 关联/任务/日志/journal/备份及共享影响；只能删除 manifest 中的受管对象。

- [ ] **Step 1:** 写 `purge_revokes_before_delete`、`late_job_cannot_resurrect`、`managed_backup_not_restorable`、`other_project_preserved`、`uninstall_retains_data`；可选服务测试 `one_instance_after_login`、`edited_plist_preserved`。断言 purge 后来源未授权、迟到提交被拒、受管旧备份不能恢复该项目；B 活跃数据不受删除 A 影响。

代表性断言（由本任务前置 fixture/临时环境准备输入，放在具名 `test(...)` 中）：

```typescript
const plan = planPurge(core, projectA.projectId);
expect(plan.projectId).toBe(projectA.projectId);
expect(plan.objects.some(o => o.kind === 'managed-backup')).toBe(true);
```

- [ ] **Step 2:** Run `npm test -- tests/integration/purge.test.ts`。Expected: RED，缺 purge 清单/恢复路径处理；若交付自动启动，同跑 `tests/integration/launch-agent.test.ts` 并记录 RED。
- [ ] **Step 3:** 实现 `mem purge --project ... --dry-run` 与显式交互执行：先停用项目、递增授权代次并取消任务，再删除证据/记忆/派生表；清单列出受管备份/log/journal 中关联数据。若快照含其他项目，先生成并验证剔除 A 的替代快照，再按清单移除原快照；任一步失败保留 purge_incomplete 标志和恢复屏障，可重入完成，不报成功。只保留无正文的防恢复 tombstone（project ID、旧 snapshot ID）；重新授权须新授权身份并提示旧源可重导入。不承诺 SSD/系统快照物理擦除。
- [ ] **Step 4:** 若交付自动启动，实现 `mem install --service`/精确 uninstall：固定自有 label、绝对可执行路径、plist ProgramArguments、安装清单、启动仍争用 Core 锁；只在临时用户测试环境/明确授权的实机服务试点执行。未交付时 AC-47 记 N/A 和理由，不创建空壳“支持”。
- [ ] **Step 5:** Run Step 2 相应命令。Expected: PASS，清空后助手无法查询/恢复目标受管数据，重新安装不丢未清空的记忆；所有故障演练仅在临时 dataDir。
- [ ] **Step 6:** `git add src/ops/purge.ts src/cli/commands.ts src/integration/install.ts docs/operations.md tests/integration/purge.test.ts`；可选服务实现另加入对应源码/测试；`git commit -m "feat: purge managed project data and preserve uninstall boundaries"`。Expected: 提交成功。

### Task 20: 冻结评测、真实闭环、性能与试点签收（T11）

**Files:** Create `scripts/{eval,acceptance,performance}.ts`, `tests/unit/metrics.test.ts`, `tests/e2e/lifecycle.test.ts`, `tests/performance/recall.test.ts`, `evals/acceptance/manifest.json`, `docs/pilot.md`；Modify `package.json`, `docs/compatibility.md`, `docs/operations.md`；本地输出 `acceptance/<run_id>/`（不默认提交）。

**Interfaces:** Produces `scoreQuality(counts:{C:number,TP:number,E:number,TPcc:number,N:number,D:number}):QualityMetrics`, `nearestRank95(samples:number[]):number`, `evaluateGate(cases:CaseResult[],requiredIds:string[]):GateResult`。任何缺项/FAIL/BLOCKED/NOT_RUN 阻断，只有可选 AC-47 可 N/A；零分母指标 null 并附 N/A 原因。

- [ ] **Step 1:** 写 `quality_denominator_zero`、`precision_before_approval`、`nearest_rank_190`、`gate_missing_case_blocked`。必写断言：`scoreQuality({C:10,TP:9,E:10,TPcc:8,N:0,D:0})` 的 precision=.9、recall=.8；C=0 不等于 1；`nearestRank95([1,...,200])===190`；任一必验缺失阻断。补全真实 CLI 的 import→extract→review→context→revoke→rescan E2E；人工确认在测试 PTY 中模拟但另外保留真人验收。

代表性断言（由本任务前置 fixture/临时环境准备输入，放在具名 `test(...)` 中）：

```typescript
expect(scoreQuality({ C: 10, TP: 9, E: 10, TPcc: 8, N: 0, D: 0 }))
  .toMatchObject({ precision: 0.9, recall: 0.8 });
expect(nearestRank95(Array.from({ length: 200 }, (_, i) => i + 1))).toBe(190);
```

- [ ] **Step 2:** Run `npm test -- tests/unit/metrics.test.ts`、`npm test -- tests/e2e/lifecycle.test.ts`。Expected: RED，缺计分/验收编排；端到端失败要定位现有模块，不能修改验收阈值。
- [ ] **Step 3:** 实现脚本与 npm scripts：`eval` 运行已冻结模型并输出人工评分待办，`acceptance` 汇总带证据路径的 cases.csv，`performance` 保存全部样本。冻结至少 50 未见验收样例（与 30/20 不同 session 和近重复组）；人工标签在正式跑之前定稿。全量质量运行 3 次，每次门槛均通过才通过；人工审批后的删错结果不得替代 Stage B 候选指标。
- [ ] **Step 4:** Run `npm run build`、`npm run typecheck`、`npm run lint`、`npm run test:coverage`、`npm run test:e2e`。Expected: 全绿且核心六模块逐模块 lines/branches ≥80%；每工作包代码审查和涉及采集/HTTP/配置/模型/删除的安全复核记录齐全。修复必须保留首次失败记录并重跑相关和完整关键链路。
- [ ] **Step 5:** Run `npm run performance -- --data-dir <隔离目录>`；数据为 100 会话/1,000 记忆。Expected: 搜索 CLI 预热 10 次后 200 次 p95≤200 ms；新进程 Hook 200 次 p95≤300 ms；业务截止≤500 ms；冷启动 10 轮全部报告；不删超时/失败样本。100 turn 抽取请求/token/限流/重试/RSS/等待量不超过 G0 冻结预算；mock 性能和真实 API 验证分开。
- [ ] **Step 6:** Run `npm run eval -- --split acceptance --runs 3`，人工完成匹配评分，再运行 `npm run acceptance -- --run-id <id>`。Expected: 全部 AC 分支有证据；至少 20 对无助手/启用助手任务（≥40 次）隔离执行，启用遵守≥18/20 且比未达标基线提高。基线同样好时须表现不降且包含审阅/改错/等待在内的中位维护时间下降。保留实际产物，不能用宿主口头承诺代替效果。
- [ ] **Step 7:** 用户授权一个项目后记录 ≥3 天影子模式与 ≥5 工作日注入试点，每日任务、收益、误用、等待、审阅和额度齐全。发现否决项立即停注入、保留诊断、修复重验；未来日期保持 NOT_RUN，不补造历史。仅在用户明确要求定时跟进时创建自动化，不默认监控。
- [ ] **Step 8:** `git add scripts tests/unit/metrics.test.ts tests/e2e/lifecycle.test.ts tests/performance/recall.test.ts evals/acceptance/manifest.json docs package.json`；`git commit -m "test: add reproducible acceptance quality and pilot evidence tooling"`。Expected: 提交成功，不含秘密/真实会话正文。最终整分支独立审查、修复、重验后使用 finishing-a-development-branch；技术、收益、试点结论分列，任何剩余 BLOCKED/NOT_RUN 不记完整 v1 通过。

## 6. 验收追踪与执行依赖

| 必验项 | 主负责 Task | 证据/关键断言 |
|---|---:|---|
| AC-01 | 12–15、20 | 实际会话 A/B 产物、批准撤回状态、重扫计数 |
| AC-02、AC-03 | 4–6 | 项目伪造、symlink、隔离来源，无 B 正文 |
| AC-04、AC-05、AC-06 | 5–6 | 幂等/事件身份/作者与子代理 |
| AC-07、AC-08、AC-09、AC-10、AC-11 | 6 | 字节游标、全部重写、记录账本、大输入、补扫 |
| AC-12、AC-13、AC-14 | 7–8 | 三列 turn 状态、5/5min/20、覆盖与阻塞 |
| AC-15、AC-16 | 6、8、12 | 实进程 crash ×3、单写者、lease fencing |
| AC-17、AC-18、AC-19、AC-20 | 8–10 | 错误分型/重试/预算/保留位置 |
| AC-21 | 10–11、20 | Stage B 输入原证据、语义负例人工核验 |
| AC-22、AC-23、AC-24、AC-25 | 12–14 | 有效视图、版本事务、冲突/并发 |
| AC-26、AC-27、AC-28 | 8、11–12 | 迟到快照、抑制、删除支持证据 |
| AC-29、AC-30 | 13 | ≥20 固定查询、Top-5、FTS 修复与治理联查 |
| AC-31、AC-32 | 14 | 完整语义、800 token/字节估算 |
| AC-33、AC-34、AC-35 | 1、15 | 实机四事件/fork、fail-open、精确安装卸载 |
| AC-36、AC-37 | 5、9–12、17、20 | 全链路秘密扫描、投毒不执行/不批准 |
| AC-38、AC-39 | 9、20 | 断网降级、实际目标、预览一致、代理与重定向 |
| AC-40、AC-41 | 16 | 浏览器真实动作、XSS/Host/Origin/CSRF |
| AC-41a、AC-41b | 4、11–12、16 | 常规 Agent 操作被拒、凭证绑定、stale 与剩余冲突 |
| AC-42、AC-43 | 17 | 5%/6% 边界、导出/日志/三个开关 |
| AC-44、AC-45 | 3、15、18 | 一致性备份、审计缺口、schema 回退、重装 |
| AC-46 | 19 | 完整删除清单、迟到结果、受管恢复路径 |
| AC-47（可选） | 19 | 服务实例与精确卸载；未交付明确 N/A |
| 验收 §5–7 | 2、20 | 覆盖率、性能全部样本、三轮质量、配对收益、真实试点 |

顺序：1 → 2 → 3 → G0 → 4 → 5 → 6 → 7 → 8 → 9 → 10 → 11 → 12 → 13 → 14 → 15 → G1 → 16 → 17 → 18 → 19 → 20 → G2。不要让两个实现者同时改 Core/schema/CLI 命令注册；可以并行做不写这些文件的官方资料核对或数据标注，但先冻结公共契约。Task 20 的性能/指标脚本可在相关模块完成后提前编写，正式冻结验收和试点仍在最后。

**每任务完成契约：** 写测试 → 观察预期 RED → 实现 → 观察 GREEN → 任务 commit → 对应审查；任何跨任务接口偏差记录 Ruling，并更新消费者测试。逐任务提交命令中的文件必须是该任务实际创建/修改且审阅过的文件；不得用 `git add .` 意外纳入本地数据。所有测试命令在实现根/所选 worktree 执行。

## 7. 官方资料核对和未验证边界

资料核对日期：2026-10-18。这里仅核对 API 合约，尚无本机运行证据。

- [Claude Code SessionStart](https://code.claude.com/docs/en/hooks#sessionstart)：JSON 输入含 cwd/session_id/transcript_path/source；输出使用 hookSpecificOutput.additionalContext。首条 prompt 不在 SessionStart 契约中。**早于 2.1.214 的 fork 可被报告为 resume**；本计划首版支持 gate 为版本 ≥2.1.214 且实机能准确区分四来源与 fork。较新字段如 onFailure 不作为必要依赖，必须先核对安装版本。timeout 单位秒，显式设置，不用 async 绕过等待。
- [Claude Code settings](https://code.claude.com/docs/en/settings)：使用项目 `.claude/settings.local.json`，跨层 hooks 合并；自身精确安装不代表可以清除其他层配置。安装检查该本地文件的 Git 忽略状态，必要时提示用户，不自动修改全局设置。
- [SQLite FTS5](https://www.sqlite.org/fts5.html)：trigram 少于 3 个 Unicode 字符不能 MATCH；保持默认 detail；字面 query 转义与参数绑定分别完成。触发器不补历史数据，必须 rebuild；external-content integrity-check 使用 rank=1。
- [Node 22→24 迁移说明](https://nodejs.org/en/blog/migrations/v22-to-v24)：采用方案指定 Node 24 LTS；具体补丁与机器兼容结果在 Task 1 固定，不声称机器已经满足。
- [better-sqlite3 官方仓库](https://github.com/WiseLibs/better-sqlite3)：核验精确 npm 发布版本、原生加载、SQLite/FTS5、WAL 和 worker 生命周期；不把上游 master 版本或“有预编译包”当作本机验证。

免费平台与模型 ID、条款同意、真实样本授权、宿主可用性仍是 G0 执行输入。它们缺失时可以运行合成工程测试，但不能进行真实数据外发或宣称平台质量通过。当前没有选择或推荐未经验证的免费平台。

## 8. 计划自查与交接

- 规格覆盖：T01–T11、AC-01–47（含 AC-41a/b）、质量/性能/试点均已分配，AC-47 是唯一可选验收项。
- 步骤检查：每个开发任务有明确文件、接口、RED/GREEN 命令、提交和可观察断言；真实宿主/模型/人工标签/试点单独记录，不虚构为自动测试。
- 接口检查：统一 EvidenceRef 的版本与 code point 区间；统一 project generation 与 governanceVersion；CoreRequest 不接受伪造 actor；所有写操作经同一 Core。公共 schema 不从研究报告示意 DDL 复制。
- 风险检查：Review Focus 五项分别有任务测试；不同 scope、非法 FTS、未知 tokenizer、同步锁等待、部分审计缺失与共享备份清除均有明确结果。
- 比例检查：本计划是实施决策与测试清单，不抄完整实现；复杂算法留给实现者，跨任务接口与门槛已固定。每个任务可独立审查，但后续任务需前置接口存在。

计划文档结构核对：20 个连续编号任务、106 个未勾选步骤、49 个验收 ID（AC-01–47 加 41a/41b）均有映射；三份 Spec 相对链接存在，代码围栏配对。这些检查仅证明文档结构，不证明产品可运行。

**交接状态：计划已编写；实现、探针、测试、质量评测、试点全部尚未执行。**

推荐子代理驱动执行：20 个任务跨采集、事务、权限和宿主边界，每项独立审查更适合发现后期难以修复的数据一致性错误；当前会话逐项执行也可选，成本较低，但只在最终做一次独立整分支审查。无论选择哪种，从 Task 1 的独立实现仓库和兼容性探针开始，先过 G0，再进入主体实现。

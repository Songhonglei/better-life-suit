# 阶段复盘：GitHub main → CloudBase 自动部署

时间：2026-11-07 19:40 起　｜　当前状态：**流水线部署已打通，push 自动触发被平台限制卡住**（20:10 订正）

> **归档说明**：第一至第八节是 19:40 的探查与设计阶段快照，第七节「需要你协助的事项」为当时状态——用户随后已授权 Qwen Key 从本机读取注入、允许 ego-browser 操作控制台并同意提交 push，Qwen Key 注入、控制台操作、提交 push 三项已解决；**push 自动触发未打通**。**实施结果见第九节**，最终配置与已知限制以 `../README.md` 的「自动部署」章节和 `../验证记录.md` 为准；文中 `output/backup/` 等路径指本地工作副本 `workbuddy-hy4-upcycle/app/`，不在本仓库内。

## 一、结论

能实现。官方提供两条自动部署链路，我给了三套方案，你选了 **B · CloudBase 原生流水线**，范围限定 **只管 `upcycle/`，门户根页与 `/stock-review/` 不纳入 repo**。
当时只完成「可行性探查 + 方案对比 + repo 侧骨架」，未创建流水线、未提交、未部署。后续已实施：流水线 `upcycle-deploy` 创建成功、首次部署 `Successful`、13/13 验真通过；**push 自动触发经验证不通**（详见第九节）。`case.evansong.cn` 三个页面与云函数全程正常：`mode=live / vision=true / model=qwen3.8-flash / ocr=true`。

## 二、探查硬结果（都有实证，不靠猜）

| 项 | 结果 |
| --- | --- |
| 目标 repo | `Songhonglei/better-life-suit`，public，main=`3b90568`，Actions 已启用，`admin` 权限齐全；**无 workflow、无 secrets、无运行记录** |
| 静态托管 | 全站共用一份：`/`、`/stock-review/`、`oa/`、`cloud-admin/`、`__auth/`、`cloudbase-sdk.js`、`og.png` 都在同一环境 |
| repo 内容 | 只有 `upcycle/` 子目录（46 文件），门户橱窗与股票复盘子页**不在仓库里** → 整站不会跟着刷新 |
| 云函数密钥链路 | `deploy-live.js` 从本机 `~/.workbuddy/models.json` 读 Qwen Key 注入；CI 上没这个文件，普通 `tcb fn deploy` 会把 `envVariables` 覆盖成空并静默退回 demo |
| 官方流水线能力 | 构建环境**预装已登录 CloudBase CLI，免腾讯云长期密钥**；支持 GitHub 仓库绑定、自定义编排、部署变量注入、自动部署开关、部署记录；公开仓库无需账号授权 |
| 关键限制 | 云 API `CreateCloudApp` 的 **`Trigger`（Webhook 自动触发）暂未对外开放**，传了不生效；自动部署只能在控制台开关 |
| 声明式 dry-run | 临时目录与真实 repo 各跑一次 `--dry-run --json --only functions,hosting`，计划只有 2 项：`update upcycle-api` + `deploy /upcycle`，**不碰门户根页、其他应用、网关路由** |
| 流水线云 API | `DescribeCloudAppVersion` 返回 `ResourceNotFound: build service version not found` —— 端点活着、只是没有该版本，说明 API 在当前环境可用 |
| 本地构建源 | `node scripts/build-cloudbase.js` 生成 9 个静态文件 → `/upcycle/`，11 个云函数文件 → `upcycle-api`，不含 `.env` |

官方文档依据：[流水线概述](https://docs.cloudbase.net/pipeline/intro)、[控制台创建流水线](https://docs.cloudbase.net/pipeline/console)、[CLI 与云 API 部署参考](https://docs.cloudbase.net/pipeline/cli-api)、[声明式部署](https://docs.cloudbase.net/cli-v1/declarative-deploy/deploy)、[配置文件动态变量](https://docs.cloudbase.net/cli-v1/config)。本机 CLI 3.9.1，`tcb deploy --dry-run` 已可用。

## 三、三套方案对比（当时的岔路口）

| | A · GitHub Actions + tcb CLI | **B · CloudBase 原生流水线（已选）** | C · 只自动前端 |
| --- | --- | --- | --- |
| 触发 | `on: push main` + `paths: upcycle/**` | 控制台绑定 GitHub，push 自动 Webhook | 同 A/B |
| 密钥 | 腾讯云 SecretId/Key + Qwen Key → 3 个 GitHub Secrets | **免腾讯云长期密钥**；Qwen Key 走流水线 Secrets | 只需腾讯云密钥，Qwen Key 不上 CI |
| 覆盖 | 静态 + 云函数 + 部署后验真全可编排 | 声明式一次部署静态/云函数/网关，带部署记录 | 仅静态，云函数仍手工 |
| 代价 | 要长期密钥塞进 GitHub | 要控制台授权/开关；`cloudbaserc.json` 需声明式；monorepo 指定 `upcycle/` 需实测 | 后端改动不会自动上线 |

你的选择：**B + 只管 upcycle**。因此门户与股票复盘继续手工维护，自动部署精确写 `/upcycle/` 子路径。

## 四、已完成的工作

repo 工作区新增 3 个文件（**均未 `git add`、未提交**）：

| 文件 | 作用 |
| --- | --- |
| `upcycle/cloudbaserc.json` | 声明式配置：`functionRoot=./deploy-cloudbase/functions`，`hosting.deployPath=/upcycle`，`envVariables` 全部用 `{{env.XXX}}` 占位，**仓库里没有密钥** |
| `upcycle/scripts/write-pipeline-env.js` | 从流水线注入的 `$SECRET_XXX` / `$XXX` 生成 `.env.local`（权限 600、被 `.gitignore` 排除），值不回显；缺 `LLM_BASE_URL` / `LLM_API_KEY` 直接失败，防止空配置覆盖线上 live |
| `upcycle/scripts/verify-pipeline.js` | 部署后远端验真：`/api/config` 断言 live/vision/ocr、静态 `index.html` 与本地产物 md5 比对、存量根页与 `/stock-review/`/`og.png`/`cloudbase-sdk.js` 完好、真发一次 OCR 确认业务级可用 |

已验证：`write-pipeline-env.js` 用假值跑通（7 项、权限 600、不回显）；`build-cloudbase.js` 产物 9+11 文件；真实 repo 目录 `tcb deploy --dry-run --json --only functions,hosting` 计划正确；`.env.local` 命中 `.gitignore:4:.env.*`；`scripts/fixtures/ocr-check.png` 在 repo 里；临时假值文件已删除，线上 config 快照未变。

## 五、遗留问题（按优先级）

| # | 问题 | 状态 | 影响 | 下一步 |
| --- | --- | --- | --- | --- |
| P0-1 | **push 自动触发还没打通**：`CreateCloudApp` 的 `Trigger` 未开放，控制台「启动自动部署」开关是否对 API 创建的流水线生效未验证 | 阻塞 | B 方案成立与否的关键，不解决就是「手动触发的流水线」，不是自动部署 | 控制台实测开关；不通就降级 A |
| P0-2 | 控制台 GitHub 授权 / 登录态：公开仓库理论上免授权，但绑定仓库、开开关都要在控制台点；本会话没有可用的 ego-browser，只有 chrome-devtools-mcp，登录态未验证 | 待验证 | 可能卡在「需要你扫码/授权」 | 见第七节第 2 条 |
| P0-3 | Qwen Key 来源未定：流水线 Secrets 需要 `LLM_BASE_URL` + `LLM_API_KEY`，你此前明确「敏感信息我来填写」 | 待你拍板 | 不填则云函数会退回 demo | 见第七节第 1 条 |
| P0-4 | `{{env.XXX}}` 端到端未验证：dry-run 不显示 env 值，无法证明密钥真被替换而不是字面量 | 未验证 | 若解析失败，`/api/config` 可能仍显示 live 但模型调用全挂 | 首次真实部署后必须真发一次 `/api/ideas` 断言 `source=llm` |
| P1-1 | `verify-pipeline.js` **只查 config/静态/OCR，没有真发 `/api/ideas`** —— 这是我本阶段写出来的盲点 | 已知缺口 | 抓不住「key 错但 config 仍 live」 | 补一次最小 ideas 请求断言 `source=llm` |
| P1-2 | `tcb deploy` 的 hosting 是「直传覆盖」，是否会清空 `/upcycle/` 下旧文件未实测 | 未验证 | 理论上产物完整无碍，但要实证 | 部署前后对比静态文件清单 |
| P1-3 | 声明式 functions 部署会不会重置 `/upcycle/api` HTTP 路由（旧命令用 `--path` 建的） | 未验证 | 接口 404，`verify` 的 GET config 能发现 | 发现后用 `tcb fn deploy --path` 修回 |
| P1-4 | 流水线绑整个 repo 的 main，**任何子目录 push 都会触发一次 upcycle 部署**；控制台没看到 `paths` 过滤选项 | 设计缺陷 | multi-repo 结构下未来加工具会产生无效部署 | 用「目标目录 + 变更检测步骤提前退出」兜底 |
| P1-5 | 3 个新文件未提交，也**未同步回本地工作副本** `workbuddy-hy4-upcycle/app` | 待办 | 下次从 app 单向 rsync 会把它们覆盖掉 | 提交前双向同步 |
| P2 | `package.json` 未加 pipeline scripts；README / 验证记录 / 根 README 未写流水线章节 | 待办 | 文档与实现脱节 | 实施阶段补 |
| P2 | 自动部署失败时的回滚脚本化、证书 2027-02-05 续期提醒、平台侧硬限流与额度告警、视觉/OCR 样本评测 | 旧遗留 | 与本阶段无关但仍在 | 另开阶段 |

## 六、风险与回滚

当前**没有任何线上变更**，回滚等于什么都不用做。后续若真实部署出问题：静态侧用 `output/backup/stock-review-root.original.html` 与 `domains-before-case.json` / `routes-before-case.json` 恢复；云函数侧重跑 `npm run deploy:live`（从本机 models.json 重新注入 Qwen 配置）并 `GET /api/config` 复核。
必须守住的护栏：部署命令固定 `tcb deploy --yes --only functions,hosting`，**不带 `--prune`、不写 `gateway` 声明、不部署到根路径**；密钥只走流水线 Secrets → `.env.local`，不进仓库、不进静态站、不回显到构建日志。

## 七、当时的协助事项（用户已全部授权 / 解决）

1. **拍板 Qwen Key 来源**（唯一必须你碰的敏感信息，腾讯云长期密钥不需要）：
   - ① 你在 CloudBase 控制台流水线「高级设置 → 部署变量」里自己填 `SECRET_LLM_BASE_URL`、`SECRET_LLM_API_KEY`；
   - ② 或授权我从本机 `~/.workbuddy/models.json` 读取后由脚本注入平台 Secrets（值不回显、不进仓库，与现有 `deploy-live.js` 同一套来源）；
   - ③ 或你直接给我一组专用 key（建议最小权限、仅该模型）。
2. **控制台授权/开关**：打开 CloudBase 控制台 → 流水线页面。若提示 GitHub 授权，请点一次授权（public repo 通常不需要）；若提示扫码登录腾讯云，请扫码。之后确认「启动自动部署」开关是否可打开。你也可以让我用 ego-browser 操作，需要在会话里把该工具打开。
3. **允许提交并 push**：这 3 个文件 + 复盘文档目前只在你本机工作区。要不要我提交到 `better-life-suit` main？注意：如果自动部署开关已开，这次 push 会立刻触发第一次真实部署。
4. **给第一次真实部署一个窗口**：会有几秒函数版本切换，期间方案生成可能失败一次；你点头我再跑。
5. **若 P0-1 走不通，选降级方案**：接受改用 A（GitHub Actions，需要你在 GitHub Secrets 填腾讯云 SecretId/Key + Qwen Key），还是保留「手动触发的 B 流水线」。

## 八、下一阶段的验收标准（怎么算真的做完）

- push 一个 `upcycle/` 下的无害改动 → 流水线自动触发，部署记录里 commit SHA 与 `git ls-remote` 一致；
- 远端 `/upcycle/index.html` md5 等于该 commit 里构建产物，`/api/config` 为 `live / vision=true / ocr=true`，真发 `/api/ideas` 返回 `source=llm` 且三方案齐全；
- 门户根页、`/stock-review/`、`og.png`、`cloudbase-sdk.js` md5 与部署前一致；
- `verify-pipeline.js` 全绿，构建日志里不出现任何密钥明文；
- repo 与本地工作副本文件一致，README 写清「怎么触发、怎么回滚、密钥在哪」。

当前任务状态（19:40 快照）：#33 探查已完成；#34 写配置与脚本进行中（3 文件已写、未提交、未同步）；#35 创建流水线、#36 push 实测均待办。

## 九、实施结果与自动触发现实（20:00 起，20:10 订正）

**结论要分两半说：B 流水线的「部署」完全打通，「push 自动触发」没打通。** 好消息是「公开仓库部署」模式免 GitHub OAuth，绕开了 P0-2；首次部署 Successful、13/13 验真、密钥端到端解析成功。坏消息是自动部署开关在控制台点了、保存提示成功、请求里也带了 `Trigger={"Webhook":{"Enabled":true,"Branches":["main"]}}`，但服务端 `DescribeCloudAppInfo` 的 `BuildConfig` 里**没有 Trigger 字段**、刷新后开关回到 OFF、`gh api repos/Songhonglei/better-life-suit/hooks` 返回 **0 个 webhook**、第二次 push 后 `LatestVersionName` 仍是 `upcycle-deploy-001`。官方文档「`Trigger` 参数暂未对外开放，传入不报错但不生效」印证了这一点。**P0-1 仍是阻塞**，不是操作疏漏。

### 落地配置

| 项 | 值 |
| --- | --- |
| 流水线 | `upcycle-deploy`，控制台 `#/pipeline`（扩展功能 → 流水线管理） |
| 触发 | 公开仓库部署，`https://github.com/Songhonglei/better-life-suit` / `main`；**自动部署开关保存后被服务端丢弃，实际 OFF** |
| 目标目录 | `./upcycle`；构建日志实证命令被拼成 `cd /root/cloudbase-workspace && cd ./upcycle && <命令>`，路径假设成立 |
| 自定义编排 | `prepare`（build-cloudbase.js）→ `env`（write-pipeline-env.js）→ `deploy`（`tcb deploy --yes --only functions,hosting`）→ `verify`（verify-pipeline.js） |
| 密钥 | 流水线环境变量 `SECRET_LLM_BASE_URL` / `SECRET_LLM_API_KEY`，值由 ego-browser 在浏览器进程内从本机 `~/.workbuddy/models.json` 的 `qwen3.8-flash` 读取后填入，不进仓库、不回显到构建日志 |
| CLI 版本 | 构建环境 CloudBase CLI 3.9.0（本机 3.9.1） |

### 验真硬结果

- 首次部署 `upcycle-deploy-001`（buildId `2611419300`）状态 **Successful**；`tcb deploy` 预执行计划只有 `Overwrite 1 (upcycle-api)` + `Direct upload 1 (upcycle)`，**没有碰门户根页、其他应用、网关路由**。
- 流水线内 `verify` **13 通过 / 0 失败**：`/api/config` = live/vision/ocr、静态 index.html 与产物 md5 一致、根页与 `/stock-review/`/`og.png`/`cloudbase-sdk.js` 完好、真 OCR POST 成功、**真发 `/api/ideas` 返回 `source=llm`**。最后一条证明 `{{env.*}}` 从 `.env.local` 解析成功、Qwen Key 真的进了云函数——**P0-4 闭环**。
- 独立外部复核（不依赖流水线日志）：根页 200 / TLS 0，`config` 仍 `live / vision / qwen3.8-flash / ocr=true`；本地再跑一次 `verify-pipeline.js` 同样 13/13。
- 提交：`3b90568 → 869bed1`（首次收录流水线配置 + 复盘），远端 `git ls-remote` HEAD 一致、目录树 52 文件、7 个关键文件 base64 md5 逐字节一致、全量扫描无密钥。

### 关键查证与新增发现

1. **`{{env.XXX}}` 读的是 `.env` / `.env.local` 文件（dotenv），不是进程环境变量**。官方文档「数据源：env `*` 从 `.env` 文件加载」明确写了；所以流水线填 `SECRET_*`，必须由 `write-pipeline-env.js` 落成 `.env.local`（600 + gitignore），`tcb deploy` 才解析得到。这是整个密钥链路最容易想错的一环。
2. **模型方案数量会抖动**：无图纯文字 `/api/ideas` 连测三次分别返回 3 / 1 / 2 个可行方案（另几次是 `infeasible=2`），`source=llm` 每次都成立——是模型对预算/工具约束的判断波动，不是部署故障。所以验真脚本硬断言改成 `source=llm && plans>=1`，「三方案齐全」只打印 warn。硬断言 3 个会让 CI 误报，这个坑是实测抓出来的，不是推演。
3. **一次操作失误（如实记录）**：填完密钥后调用了 `page.snapshot()`，把 API Key 明文带进了本地工具输出。密钥未写入任何文件、未进仓库、未进构建日志、未出本机会话；后续全部改成 `page.evaluate()` 脱敏读取。**含密钥的表单页禁止用 snapshot**，下次直接走脱敏路径。是否轮换该 Key 由用户决定。
4. `tcb` 增删域名/路由要交互确认，`printf 'Y\n'` 喂确认；控制台 Tea UI 的 switch 是 `input` 被 `span` 覆盖，`page.click` 会报 intercepts，用 DOM `cb.click()` 才生效。

### 仍未解决 / 已知限制

| 项 | 状态 |
| --- | --- |
| P1-4 无 `paths` 过滤 | 当前**不适用**（push 根本不触发）；若降级 A（GitHub Actions）必须在 workflow 加 `paths: ['upcycle/**']` |
| P1-2 hosting「直传覆盖」是否清空 `/upcycle/` 旧文件 | 本次实证：上传 8+1 个文件、`tcb hosting list` 总数 21 与部署前一致，`verify` 静态 md5 与存量页面全绿 → 未误删 |
| P1-3 声明式部署是否重置 `/upcycle/api` 路由 | 本次实证：`/api/config`、真 OCR、真 ideas 全通，路由未被重置 |
| P1-1 / P1-5 / P2 | 已解决：ideas 断言补上、3 文件同步回本地工作副本、package.json pipeline scripts 与 README 自动部署章节补齐 |
| 旧遗留 | 证书 2027-02-05 续期、平台侧硬限流与额度告警、回滚脚本化、视觉/OCR 样本评测、站外链接巡检，仍未做 |
| 并存冲突 | 本机 `npm run deploy:live` 与流水线都会更新同一个 `upcycle-api`，后跑者覆盖前者；手工改过云函数配置后，下一次 push 会被声明式配置重置 |

**本次自动部署验收**：第九节写完后的这次 push 本身即触发测试——若流水线列表出现 `upcycle-deploy-002` 且状态 Successful，P0-1 才算真闭环（见 `../验证记录.md`）。

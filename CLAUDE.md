# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 仓库定位

**Meta自动化任务（v2）** 需求与原型仓库，**无构建系统、无测试、无包管理器、无后端代码**。产出物两类：
- `index.html` — 单文件 UI 原型（3681 行，内联 HTML/CSS/JS，mock 数据硬编码），浏览器直接打开即可预览
- `index.html.bak` — 旧备份，已 gitignore，**别读它**：内容与当前 `index.html` 冲突
- `*.md` — 需求文档与设计文档

由 `auto_put`（v1）分叉而来，承接「任务模块化」改造及其后续产出物。v1 冻结、不再接受新需求，改动一律在 `auto_put_v2` 展开。跨平台同步事宜改由本仓库承担，改一个需求前先确认要不要带 `auto_put_tiktok`（TikTok 投放）端。

## 运行与验证

无 lint / 无 typecheck / 无测试命令可跑（无 JS 工具链，`index.html` 内联全部代码、无外部 CDN 依赖，唯一 `<script>` 块从 `index.html:1301` 起，到 `3679` 止）。预览即 `open index.html`。

**改完先做语法体检**（比开浏览器快，能抓住绝大多数低级错误）：
```bash
node -e "const m=require('fs').readFileSync('index.html','utf8').match(/<script>([\s\S]*)<\/script>/);new (require('vm').Script)(m[1]);console.log('OK')"
```

原型改动一律用 DOM 断言验证，**禁止截图**：headless Chrome + CDP（`--remote-debugging-port`，Node 自带全局 WebSocket，无需 playwright）驱动，`Runtime.evaluate` 断言元素存在性/文本/类名/几何/`scrollWidth`，`Input.dispatchMouseEvent` 测 hover，输出 JSON 核对；临时脚本写 `/tmp` 用完即删。以下坑按踩过的顺序记：
- 抽屉树是 fixed 定位，判可见性要沿祖先链查 `display`/`visibility`，只用 `offsetParent` 会误判
- `#pkgBody-*` 三个抽屉同时存在于 DOM，只按 `.form-item` 计数会把三个包的字段混在一起，必须先按 `#pkgBody-<type>` 限定范围
- **断言样式别只查溢出**：`.form-item label{width:140px}` 会命中所有后代 label，导致下拉选项文字**换行**。换行不产生 `scrollWidth > clientWidth`，查溢出的断言会全绿放过 → 补一条高度断言（单行 < 40px）
- **`getElementById` 只返回首个匹配**，重复 id 会让断言"通过"而实际页面已坏。改完表单调一次重复 id 断言（`dupIds` 必须为 `[]`）
- **`autoTasks` / `campaigns` / `adGroups` / `adsList` 的多个字段在载入时由 `Math.random()` 生成**（任务状态、竞价策略、预算、上传状态、失败原因、日期等），每次刷新结果都不同 → 不要断言这些字段的具体条数/文案；同一次页面载入内多次 `Runtime.evaluate` 才是稳定的。确定性的是 id、包引用、`1571..1577` 任务名、以及**任务 1577（`i===6`）恒引用停用包**——它专用来跑「失效标黄」路径，别当成脏数据清掉。`executionUnits`（`index.html:2543` 由 `autoTasks` 叉乘生成）的 `status` / `seriesCount` / `failReason` 同样是随机的，**唯一确定的是单元 id（从 4000 起）与它引用的四个包**
- **headless 下 `alert()` / `confirm()` 会阻塞**，测试里点「添加媒体」「保存包」这类入口前先 stub `window.alert` / `window.confirm`，否则脚本挂死；报 `no matches found` 的 zsh glob 会中止整条命令，`rm` 与通配符别写在同一行
- **抽屉有 `slideInRight .25s` 动画**，刚打开时还在屏外（`left:1600`），立刻查几何或 `elementFromPoint` 会拿到 `null`/错值；断言前先等 450ms

## 两份权威文档，读法不同

- `自动化投放-需求文档.md` — 项目整体需求，重点在执行任务逻辑与页面结构。**本文档自称唯一权威需求文档**，改任何包、任务表单或执行逻辑前先读它。
- `MetaAutoTask-素材逻辑.md` — 唯一的**领域**权威文档，描述调度→落库的后台业务逻辑（短剧运行条件、素材规则、账户包与上限、去重、`persistCampaignAndChildren`、Kafka）。**不要拿 index.html 的原型行为当后台逻辑。**
- `任务多对多-设计文档.md` — 多媒体行 × 多包组合的设计存档。原型**已按此实现**，文档内容基本落地；差异是它写「三包各多选」，实际媒体行锁定为「一行一对账户包/策略包」，所以多对多只发生在短剧包与素材包两个维度。改引用结构前可对照，但**以原型与需求文档为准**。

三者有冲突时：执行逻辑听素材逻辑文档，交互与字段听需求文档，原型只是原型的实现。

## 页面模型

外层导航两组（`#outerNav` 的 `data-outer="auto" | "preconfig"`），页面 id 为 `page-<name>`，靠 `.page-container.active` 显隐；`innerNav` 与 `preconfigInnerNav` 各自维护激活态，切换时**全局清空**所有 `.page-container` 再激活目标页，面包屑同步重写。
- 自动化任务：`auto-task` / `campaign` / `adgroup` / `ad`
- 前置配置（5 项）：`account-package` / `media-package` / `targeting-package` / `drama-package` / `material-package`

运行约束不单独成页，已并入任务表单。

新页面需同时加 `page-container`、导航项（`data-page`）和切换逻辑；若挂在已有内层导航下，两个内层 tab 的 click handler 是各自独立绑定的两段代码，别只改一处。

## 共享可变状态（改交互前必读，最容易踩的坑）

模块级全局变量被**任务表单和包抽屉共同持有**，抽屉 `init*Pkg` 会清空并重灌它们——所以在抽屉里改一处会静默影响任务表单：
- `dramaTargetingMap` / `dramaStatusMap` / `dramaScheduleMap` — 任务表单的逐剧行（key 是 `剧本ID-剧本名称-短剧名称-语言`）**同时**是短剧包「手选清单」模式的数据源，`initDramaPkg` 会 `clear()` 后按 `p.dramaKeys` 重建；`dpLangTargetingLangs()` 也正是靠读这个 Map 反推语言
- `materialMode` / `confirmedMaterials` / `materialSelected` / `materialFiltered` / `matPage` — 任务表单的素材配置**同时**是素材包抽屉（`initMaterialPkg`）的状态

任务表单自有的多对多状态（不在抽屉里，但同样是模块级单例，重开弹窗不会自动重置，必须走 `resetAccountDrawerRefs()`）：
- `mediaRows` — 媒体行数组，每项 `{media, mediaPkgId, accountPkgId}`。`completeMediaRows()` 只返回三项齐全的行
- `multiPkgSel` — `{drama:[], material:[]}`，短剧包与素材包的多选结果
- `executionUnits` / `expandedTasks` — 执行单元 mock 数据与任务列表的展开态

另有一批只是模块级单例（不同时被两处持有，但同样让抽屉不可重入）：`dpLangTargeting`（短剧包抽屉的按语言行，Map，保存时序列化成 `langTargeting` 对象）、`pkgDrawerType` / `pkgDrawerId`、`filteredTasks`。

`addDrama()` 的 key 是**从 DOM 文本拼出来的**（取 `.drama-result-item .value` 的第 2/1/3/4 个），顺序必须与 `dpLangTargetingLangs()` 里 `${playbookId}-${playbookName}-${name}-${lang}` 的拼法一致——调整搜索结果卡片结构会静默断掉这个 join。

## 包模型（模块化的核心，改包先读这节）

四个包：账户包、素材包、短剧包、投放策略包。账户包与投放策略包带媒体属性，其余两个媒体无关。
**定向包不是四包之一**，它有独立的前置配置页（`renderTargetingPackages`）与独立抽屉，不要塞进包抽屉体系。

包管理页由一张配置表驱动，**不要为每个包复制一份页面代码**：
- `PKG_CONF` — type → `{type, body, pagination, label, hasMedia, list()}`；`list()` 返回该包的 mock 数组
- `pkgBodyEls()` + `openPkgDrawer()` — 抽屉内 `#pkgBody-<type>` 互斥显隐，再分派到 `initDramaPkg` / `initMaterialPkg` / `initMediaPkg`；**账户包是例外**，`openPkgDrawer('account')` 直接转 `openAccountPackageDrawer`
- `renderPkgPage()` / `submitPkgDrawer()` / `savePkg()` / `togglePkgStatus()` / `deletePkg()` / `copyPkg()`；`submitPkgDrawer` 按 type 分支拼 `summary`，新增字段要同步该分支
- `taskRefIds(t, type)` / `refTasksOf()` 反向查引用；`taskBadPkgs()` / `pkgCellMulti()` 渲染失效标黄与多值压缩；`openPkgView` 是只读摘要抽屉

新增一个包 = 加 `PKG_CONF` 条目 + 一段 `#pkgBody-<type>` 标记 + 一个 `init<Pkg>Pkg()`，不要新建文件。
**包名称 / 媒体 / 广告系列名称后缀是四个包共用的抽屉顶层字段，不在任何 `#pkgBody-*` 里**（`#pkgBody-media` 是从「广告目标」才开始的）。加这类共用字段时，靠 `PKG_CONF` 上的布尔开关控制显隐（`hasMedia`、`hasSuffix`），并在 `openPkgDrawer` 里 toggle + 回填、在 `submitPkgDrawer` 里按同一个开关收集，不要写 `type==='media'` 散判。
账户包是例外：`openPkgDrawer('account')` 第一行就转 `openAccountPackageDrawer` 并 return，共用字段一律不重置——它用的是另一个 overlay（`#accountPackageDrawer`），不是 `#pkgDrawer`，所以共用区残留旧值不会露出来，别当成 bug 修。
**影子映射要同步**（不在 `PKG_CONF` 里，加包时最容易漏）：`PKG_TYPE_LABEL`（抽屉标题用的中文名，与 `PKG_CONF[type].label` 重复）、`SEL_TO_TYPE`（只读摘要下拉 id → 包类型）。

核心 mock：`mockAccountPackages` / `mockMaterialPackages` / `mockDramaPackages` / `mockMediaPackages` / `mockTargetingPackages` / `mockDramaLib` / `mockMaterialData` / `mockAllAccounts` / `mockCallbackPlans` / `mockMonitorLinks` / `mockChannels` / `autoTasks` / `executionUnits`。

## 列表页与按钮组

任务/系列/广告组/广告四个列表共用 `renderTable(bodyId, data, cols, pageSize, ops, page, expand)` + `renderPagination(pid, total, pageSize, onChange, page)`。`cols` 元素为字符串（直取字段）或 `{key, render}`；行内操作 `ops` 是数组（`{label, onclick}`）或函数；`expand` 是可选的子行渲染函数，**只有任务列表传了它**（`renderUnitRows`），展开态查 `expandedTasks`。整表 innerHTML 重建，事件靠 inline `onclick` 而非委托。
**`renderTable` 里 `const ok=row.uploadStatus!=='上传成功'` 语义是反的**——`ok` 为真表示「没上传成功」，据此给 checkbox 和按钮加 `disabled`；照字面理解会写反。
`syncBatchCounts()` 每次把全选框强制置为未选，与列表自己的选中态是两套逻辑，改批量条前先看清走的是哪条。

`.btn-option` 按钮组有**三个互不相通的接线点**，新增一组要挑对地方：`bindBtnGroup(sel)` 通用版（`index.html:2310` 起，内含 `#materialCountType` / `#singleRoundRepeatType` 等硬编码特判）、带可选回调的 cfg 数组、以及 `#materialFilterType` / `#materialModeBtns` / `#dpRuleModeBtns` 等自带 handler 的。`setBtnGroup()` / `btnVal()` 是所有 `init*Pkg` 用的程序化读写口。复选框组必须包在 `.check-group` 里，否则 label 的 `checked` 类不更新。

初始化集中在 `index.html:3678` 一行（`initTaskFilterOptions();refreshTaskPkgOptions();…renderAds()`）——新增全局渲染入口挂这里。

## 任务表单与执行单元（多对多的当前形态）

任务表单的包引用不是单值：
- **媒体行**（`#mediaRowList` → `mediaRows`）— 每行 = 媒体 + 投放策略包 + 账户包。媒体的两个下拉按本行媒体过滤。媒体不可重复行，首行不可删
- **短剧包 / 素材包多选**（`#refDramaPkg` / `#refMaterialPkg` → `multiPkgSel`）— 用 `fillMultiPkgSelect` / `renderMultiPkgBox` 渲染成「触发器 + 勾选面板」，原生 `<select>` 只作数据源且被 `display:none`
- **执行单元计数**（`#unitCountHint`）= 媒体行 × 短剧包 × 素材包，未填齐时隐藏
- 提交产出 `{mediaRows[], dramaPkgIds[], materialPkgIds[]}`，不再有 `mediaPkgId` 等单值字段

`renderMultiPkgBox` 每次勾选会重建 `box.innerHTML`，**重建后面板会收起**，靠 `keepOpen` 参数调 `positionMultiPanel()` 恢复展开态。面板用 `position:fixed` 是为了逃出 `.modal-content{overflow-y:auto}` 的裁剪，弹窗滚动或窗口 resize 时由 `closeAllMultiPanels()` 收起。

## 关联关系（改引用字段前必读）

- 任务只存包引用、不存字段值；改包不回写已建任务，任务下次执行时生效。`copyTask` 明确只复制引用、不复制包内容（但数组字段要深拷贝，否则副本与原任务共享引用）
- 任务 → 媒体 由所引投放策略包派生（`taskMediaOf`），多媒体行时返回 `Meta / TikTok` 形式；任务上不单独存媒体字段
- 语言是短剧、短剧包、定向包之间的贯穿键。`DRAMA_LANG_CODE`（中文→zh 等，仅 4 个语种）是唯一映射，`LANG_LABEL` 是它的反向且覆盖面不同，`mockTargetingPackages` 里还混着 `ms` 这种两边都没有的码
- 包引用只允许两层，包不能引用包
- 被任务引用的包不可删；被短剧包引用的定向包不可删。`dpTargetingIds()` 是定向包引用判定的唯一入口（删除拦截 `deleteTp` 与引用计数 `tpRefCount` 都走它），**改动短剧包的引用字段时必须同步它**
- 任务提交的校验：媒体行完整且媒体不重复 → 短剧包/素材包至少各选一个 → 引用包不得停用 → 素材包 `mode==='specify'` 必须配短剧包 `ruleMode==='list'` → 账户包内账户数非空 → 每轮账户数不得超过各账户包中最少的账户数 → 结束条件数值必填、上限与账户数不得小于 1
- **每轮账户数按「每个执行单元各自适用」实现**，所以校验取各账户包的**最小值**（`accountCountsOfRows()`），提示与校验必须走同一个数据源，否则会出现「提示说 4、实际放行到 6」

## 短剧包

两种模式：按规则 / 手选清单（`#dpRuleModeBtns`，`toggleDramaPkgMode` 切 `#dpRuleFields`/`#dpListFields`）。
按规则的维度为语言（下拉多选）、指标筛选、时间窗口、上线时间、初次投放时间；保存时要求至少填一个维度。**仅允许筛选最近 30 天内有消耗的短剧**，这条说明常驻在 `#dpRuleFields` 顶部。
指标条件行常量是 `DP_METRICS` / `DP_OP` / `DP_SUFFIX`，时间窗口文案是 `DP_WINDOW_LABEL`。**时间窗口只在有指标条件时出现**，显隐并进 `checkDpMetricLogic()`（它同时管条件关系），不要另起一个函数。
定向包按圈剧语言分行（`renderDpTargetingRows()` → `#dpTargetingRows`），**每行同时容纳 Meta 与 TikTok 两个下拉**，留空表示该媒体不投该语种。`DP_ALL_LANGS` 是语种不限时的分行来源——不限语言要列出全部语种，不是退化成一行「不限」。定向包自身不带语种属性，候选不按语言过滤。手选清单模式下分行语言由已选短剧的语言派生。
相关助手集中在 `dp*` 前缀（`dpSelectedLangs` / `dpMetricRowsData` / `dpTargetingText` / `refreshDramaTargetingOptions` 等），改规则字段时通常要同时动 `initDramaPkg`、保存分支与摘要拼接三处。
**注意 `renderDpTargetingRows`（包抽屉，按语言分行）与 `renderAllDramaRows` / `renderDramaRow`（任务表单，按短剧分行）是两套不同的渲染器**，不要混。

**任务表单里的定向包选择器是独立抽屉**（`openTargetingDrawer` / `showTpList` / `confirmTargetingSelection`），状态挂在 `dramaTargetingMap`。批量模式用哨兵 key `'__batch__'` 存在 `currentTargetingDramaKey` 里，选中态另存 `batchSelectedIds`，确认时广播到所有行——读选中态前必须先判 `isBatch`，否则会读到空数组。
定向包自己的编辑抽屉是另一套：`openTargetingPackageDrawer` / `renderTpdBody` / `bindTpdBody` / `tpd*` 前缀（区域多选、勾选态在 `tpdRegions` / `tpdChecked`）。注意 `mockTargetingPackages` 的字段**按媒体分裂**：Meta 系用 `device`/`wifiOnly`，TikTok 系用 `os`/`network`，没有统一 schema；定向包表单不含版位字段。

## 投放策略包

包表单由**媒体 × 广告目标**两级决定（`MP_CHANNELS`），四种组合字段集互不相同，分桶 `['account','campaign','adGroup','ad']`。改字段先读 `MP_CHANNELS` 与 `mpRefresh` / `mpCollect` / `mpValidate`，条件显隐走 `showWhen` / `hideWhen`（支持 `['字段','值']` 与 `['字段','v1','v2']` 两种写法，由 `condHit` 解析）。
**投放策略包不含定向包字段**——定向包跟着短剧包走，由短剧包按圈剧语言分行分配（见「短剧包」一节）。历史上 MP 的「广告组设置」里曾有一个 `targetingPackage` 下拉，已移除，**不要加回来**：定向包是媒体级且与语种绑定的，塞进策略包会与短剧包的语种行重复建模。移除后 Meta/销量的 `adGroup` 为空数组，分组标题靠 `renderMpSchema` 里的 `fs.length?...:''` 自动不渲染，不是 bug。
`mockMediaPackages[].strategy` 是字符串 `'W2A' | 'H5' | '直投'`，而 `mockCallbackPlans[].strategy` 是数字 `1/2/4`，靠 `MP_STRATEGY_CODE` 换算。回传模板按 strategy × businessType 过滤，H5 链接（`mockMonitorLinks`）固定取 `strategy===4`。
TikTok 商品数据源开启后有商品对齐抽屉（`openProductLibraryDrawer`，三栏，`pl*` 前缀）。

## 已知重复定义

`openRefTaskList` 与 `closeRefTaskList` 各定义了**两次**（`index.html:1492` 与 `index.html:3618`），后者覆盖前者，改前一处不会有任何效果。清理前先确认没人依赖前者的写法。

## 远程仓库与提交

远端 `https://github.com/gar-su/auto_put_v2`，**公开仓库**，默认分支 `main`，已开 Pages：`https://gar-su.github.io/auto_put_v2/`。仓库内容含内部后台业务逻辑与真实投放标识符（回调模板 ID、像素 ID、渠道号），推送前想清楚可见性——Pages 让原型成为可直接浏览的公开网页，比读源码更易被发现和索引。
本机**没装 `gh`**，走 GitHub API + git 凭据助手：PAT 存在 `~/.git-credentials`（账号 `gar-su`，scope 含 `repo`），远程 URL 用干净的 HTTPS 地址，token 不写进 `.git/config`。
建仓、推送、开 Pages 可全程走命令行，不用去 GitHub 后台：
```bash
TOK=$(grep -o 'https://[^@]*@github\.com' ~/.git-credentials | head -1 | sed 's|https://||;s|@github.com||' | sed 's|.*:||')
R=<repo>   # 换成目标仓库名

# 1 建仓
curl -s -X POST -H "Authorization: token $TOK" https://api.github.com/user/repos \
  -d "{\"name\":\"$R\",\"private\":false}"

# 2 推送
git remote add origin https://github.com/gar-su/$R.git && git push -u origin main

# 3 开 Pages（仓库已开 Pages 时返回 409，属正常）
curl -s -X POST -H "Authorization: token $TOK" \
  https://api.github.com/repos/gar-su/$R/pages \
  -d '{"source":{"branch":"main","path":"/"}}'

# 4 轮询到 built（首次约 1–2 分钟）
curl -s -H "Authorization: token $TOK" https://api.github.com/repos/gar-su/$R/pages
```
开 Pages 的三个前提：仓库根目录须有 `index.html`，否则得改用 GitHub Actions 工作流构建（`build_type: "workflow"`）；免费账号只有**公开**仓库能开 Pages，私有仓库需付费计划；POST 返回成功不等于站点可用，要轮询到 `status: built` 再把地址给出。
推送 `main` 会自动触发 Pages 重建，无需手动操作。关站：`DELETE /repos/gar-su/<repo>/pages`。
`index.html.bak` 是旧备份，内容与当前 `index.html` 冲突（曾导致按它导出错误的字段表），已写进 `.gitignore`，不要入库。
提交规范 `feat/fix/docs/chore: <中文描述>`。草稿类文档默认不提交，按需再入库。

## 需求文档写作约定

- 紧凑格式：标题、正文、表格、列表之间**不留空行**，紧密排列；`---` 分节线紧贴前后；表格单元格内禁用加粗
- 内容只描述**改动范围、交互、行为、验收标准**；**不写接口路径、请求参数、响应格式**等实现细节
- 命名：功能名 + `-需求文档.md`，设计类用 `-设计文档.md`，平铺在仓库根目录

# SRS Spec Manager 踩坑与提速记忆（pitfalls.md）

> **维护约定**：本文件是 srs-spec-manager skill 的统一记忆文档，合并「实战经验与已知坑」与「提速经验速查」两部分。
> - 每次执行 skill 前先按当前所处步骤查对应小节，命中结论直接套用，避免重复调试。
> - 执行中发现新的踩坑或提速方法，**当场追加**到对应小节（格式：`- [YYYY-MM-DD] 内容`），并更新文末耗时基线表。
> - SKILL.md 只维护流程与关键规则；接口地址、组件签名、调试技巧等细节统一沉淀到本文档。

---

# 第一部分 实战经验与已知坑（原 SKILL.md）

1. **规格数据在飞书 bitable（多维表格）**：条目常存于 bitable 表格（canvas 渲染，
   DOM 读不到内容，网页内搜索可能搜不到）。改用 lark-cli 读写：
   `lark-cli base +base-block-list` / `+table-list` / `+record-search` /
   `+record-batch-update`（`--as user`）。规格 ID 检索字段通常是 `SPEC_ID`；
   目标记录不一定在第一个表，需遍历 Base 内所有表查找。
2. **集成浏览器后台标签页限制**：标签页处于后台（offscreen/节流）时，所有
   CDP 鼠标点击无效、截图失败、`elementFromPoint` 返回空；但 JS evaluate、
   键盘事件有效。页面操作优先用 evaluate 调用页面自身函数（easyui 的
   `textbox('setValue')`、表单校验函数、提交函数等），不要依赖模拟点击。
3. **easyui 控件取值**：`browser_fill`/`browser_type` 对 easyui textbox 无效
   （不同步隐藏 `.textbox-value`，`getQueryParams` 返回空）。列表过滤优先直接
   导航/请求 datagrid 数据接口（如 `xxx_list.jhtml?...&specId=<id>` 取 JSON）。
4. **PMS 页面结构**：功能页内容在 iframe 中，可直接导航到 iframe src；
   按钮 onclick 属性可读取以获得底层处理函数与真实 URL。
5. **表单校验陷阱**：BUG 影响面（repairInfluence）按系统模版**完整键名逐项**
   作答（checkSwitch 逐项匹配，简写键名会拦截）；工作包遗留状态必须与
   「遗留BUG」（savebugid）一致（wbs 名含「遗留或delay的bug」⇔ savebugid=是）。
   校验失败提示在 Notiflix 弹窗中，也可直接调用页面校验函数读取返回值。

---

# 第二部分 提速经验速查（原 speedups.md）

## 一、BUG 页面（bugs.ruijie.com.cn）速查

### 1.1 字段定位
- Label→字段 id 反查：从字段元素出发沿 `previousElementSibling` 向上最多 6 跳，第一个非空文本即标签文本。
- validateData 校验逻辑：遍历 `.required` 元素 → 元素有 `selfValueValidate` class 时读自身 value；否则读 `getElementsByName(id)[0]` 的 value（**隐藏域**，非可见输入框）。只设可见文本不设隐藏域，校验必失败。
- 校验失败排查顺序：**先 remove 所有旧 Notiflix 弹窗**（`document.querySelectorAll('.notiflix-report')`），再调 validateData，再读新弹窗 `.notiflix-report-message`。旧弹窗残留是误判的最大来源。[2026-10-12]

### 1.2 eu-tags-dropdown chooser 设值公式 [2026-10-12]
| 组件（字段） | setValue 签名 |
|---|---|
| commonchooser（closeproperty/resolvedok/needCollect/savebugid 等） | `$('#x').commonchooser('setValue', {dictData:<数字id>, dictText:'<显示文本>'})` |
| introducedchooser（effectbug） | `$('#effectbug').introducedchooser('setValue', {id:<id>, Introduced_state:'<文本>'})` |
| workpacketchooser（wbs） | `$('#wbs').workpacketchooser('setValue', {id:<id>, workPacketName:'<文本>'})` |
| cominfochooser（comInfoId） | `$('#comInfoId').cominfochooser('setValue', {com_info:'<文本>'})` |

- 只传 `{dictData:'', dictText:...}`：隐藏域被写成空/"undefined"，可见框有值但校验仍失败——dictData 必须是字典真实 id。
- 字典 id 查询接口：`POST /bug_switch/bug_dict_detail/load_bug_dict_detail_list?disabled=0&dictCode=<字段id>` → `rows[].dictData / dictText`。
- 已知字典值：
  - closeproperty：1=遗留缺陷，2=变更指标，3=规避解决
  - resolvedok：3=非用户接口修订，4=用户接口修订
  - needCollect：15=不需要，16=已收集，17=需要但未收集
  - savebugid：1=否，2=是
- effectbug 数据源：`POST /bug_switch/bug/bug_introduced_state/load_bug_introduced_state`（需求分析 id=70；其余：总体设计71/详细设计72/编码阶段73/第三方代码74/硬件缺陷75）。
- 工作包数据源：`POST /bug_switch/bug/work_packet/load_work_packet_list`（参数 `osIdArr=<osId>, rows, page`）；wbs 隐藏域值=rows[].id。os（项目 id）用 `$('#os').projectchooser('getValue')` 取。

### 1.3 组件关联
- 发现BUG的组件 comPathId 是 remote combobox，**依赖 comInfoId 先有值**（onBeforeLoad 检查 comInfoId）才加载数据；填完 comInfoId 后必须调 `addCom(document.getElementById('relationBtn'))`（「关联」按钮），否则校验报「发现BUG的组件还未填写」。[2026-10-12]
- 跨字段一致性：wbs 名含「遗留或delay的bug」⇔ savebugid 必须=是(2)；否则报「工作包遗留状态与遗留BUG状态不一致」。SRS 修订场景固定 wbs=功能包 + savebugid=否(1)。

### 1.4 UEditor 与提交验证
- 修改说明：`changeMessage.setContent(<html>)`；validateData 会自动把 `getContent()` 同步进 `#changeMessage`。
- HTML 锚点用**无引号属性**写法 `<a href=... target=_blank>文本</a>`，规避多层引号转义炸脚本。
- 服务端读回 `GET /bug_switch/bug/get_bug_info_params?bugId=`：返回字段值快照（state/wbs/closeproperty/savebugid/effectbug 等）可用于确认提交成功，但 **changeMessage 不在返回体**；验证修改说明需打开 main 页 iframe（iframe[1] = bug_info）搜文本。[2026-10-12]
- BUG main 页结构：正文在 iframe 中，`document.querySelectorAll('iframe')[1]` 即 bug_info 编辑页。

## 二、评审提交（PMS）速查

### 2.1 评审人指派 [2026-10-12]
- `_reviewRoleUserPanelHandler.getAssignedItems(entityId, roleName)` **必须两个参数**；只传 entityId 恒返回 `[]`，勿误判为指派失败。
- 直接验证指派结果：`$('#_REFDOC_REVIEW_USERS_<entityId>').userchooser('getItems')`（item.userRole 是该成员角色并集，如 "PSDPE,PGTTM"）。
- UI 验证：角色 chip（`<entityId>_<ROLE>`）`num>=1` 且 class 含 `rf-review-user-yes` 即满足。
- 提交时 `params.reviewObjects` 直接取 `_relatedDocumentHandler._relatedDocumentMap` 的 docs（reviewUsers 带 userRole），chip 仅为前端展示校验。

### 2.2 保存≠发布（两段式）[2026-10-12]
- 首次 `saveReviewForm(true, btn)` 成功后 formStatus=**TEMPORARY**（flow 页显示「评审单未发布」）。
- 发布路径：打开 `/review/form/apply.jhtml?id=<reviewFormId>&TAB=<RF单号>`（flow 页「编辑」按钮目标），点「发起评审」按钮（onclick 同为 `saveReviewForm(true, this)`）→ formStatus 变 **PUBLISHED**。
- 表单回填自动完成：relationBugIds/summaryOfChanges/reviewContent/评审人都会从服务端恢复，无需重填。

### 2.3 接口与字段
- 评审单列表：`POST /review/form/load_review_form_list.jhtml`（projectId/rows/page）→ rows[].formCode/id/formStatus/relationBugIds/reviewSubject。
- 评审单链接：`http://pms.ruijie.com.cn/review/form/flow.jhtml?id=<reviewFormId>`。
- 申请页字段：reviewSubject、expectFinishDate、relationBugIds、summaryOfChanges（变更简述）、reviewContent（内容概述，填 bugId 链接）、ccUserChooser。
- 添加评审对象：`POST /review/form/related/documents.jhtml` 参数必须数组编码 `includingDocIds%5B%5D=<docId>`（传 `includingDocIds=<id>` 报 MyBatis not iterable），随后 `_relatedDocumentHandler.appendExpectDocument(row)` 渲染。
- 角色惯例查询：文档自身历史评审单（同文档 RF 单）`/review/review_user/get_review_user_role.jhtml?reviewFormId=<id>` 取历史角色→人员映射。

## 三、bitable / lark-cli 速查

- `--filter-json` 复杂条件必须 @file；@file 仅允许 cwd 与 %TEMP% 根；含空格中文值直接放命令行会被 PowerShell 拆成位置参数。[2026-10-12]
- SPEC_ID 列为空时，改用 1-6 级特性路径过滤定位记录（conditions 二维数组 + `"logic":"and"`）。
- PowerShell 无 `head`，用 `Select-Object -First N`。
- 记录更新：`base +record-batch-update --json @file`；更新后必须重新 record-list 读回验证。

## 四、browser_evaluate 脚本规范

- 内联脚本禁 IIFE；顶层 `var` + 单行 `JSON.stringify(...)` 以表达式结尾作为返回值。Exec 沙箱的 `text()` 在浏览器脚本内不可用。
- `\n` 经 JSON→JS 双层转义会被二次解码成真实换行导致 SyntaxError → 用 `String.fromCharCode(10)` 拼接多行文本。
- 含转义引号的属性选择器（如 `[id^=\"x_\"]`）易返回 undefined → 改 `querySelectorAll('[id]')` 后按 indexOf 过滤。
- 后台标签页限制见第一部分第 2 条：页面操作一律 evaluate 调页面自身函数，不模拟点击。

## 五、SRS 规格修改描述模版

BUG 页操作日志/问题描述按以下固定格式描述 SRS 规格修改；步骤1提取变更信息时按此
解析，步骤6回填「修改说明」时按同字段组织修订内容：

```
srs规格id：
错误问题描述：
修改规格描述：
覆盖产品：
```

字段说明：
- **srs规格id**：PMS 规格条目 ID，可多个（对应 load_project_product_standard_list 的 specId 参数）。
- **错误问题描述**：原规格取值及缺陷场景说明（可含 BUG 现象）。
- **修改规格描述**：修订后的规格取值/表述，如「产品取值由 Y 改为 (0,200]ms」。
- **覆盖产品**：填「所有产品」或列举具体产品型号；部分产品时必须逐个列出（与 bitable 产品列名对应，如 RG-N18010-XH、S6950-128QC）。
  [2026-10-12]

## 六、耗时基线（BUG 1735810 实测 ≈45 min）

| 阶段 | 实测 | 主要时间去向 |
|---|---|---|
| BUG页提取+PMS定位 | ~6 min | 两级查询、wiki→bitable 解析 |
| 权限申请 | ~1 min | 一次成功 |
| bitable 修改 | ~8 min | 过滤条件/lark-cli 踩坑 |
| 评审发起 | ~15 min | 角色惯例查询、评审对象注入、指派误判、TEMPORARY 二次提交 |
| BUG页回填 | ~13 min | 字段反查、chooser API 试错、字典查询、validateData 三轮排雷 |

> 复用本文档结论后同类任务预计可压缩到 ~20 min。新实测数据请更新此表。

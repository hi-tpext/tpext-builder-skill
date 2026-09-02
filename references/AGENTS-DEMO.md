# tpext-builder 框架总览

> 可重命名为 AI 工具要求的文件名放入项目根目录：Claude Code → `CLAUDE.md`、Gemini CLI → `GEMINI.md`、Copilot → `.github/copilot-instructions.md`

## 框架概要

`tpext-builder` 是 webman 1.5/2.2+、ThinkPHP 5.x/6.x/8.x 后台管理 UI 生成框架，基于 LightYearAdmin（Bootstrap 3）主题。命名空间 `tpext\builder\`，PSR-4 根 `vendor/ichynul/tpextbuilder/src/`。

控制器**必须继承 `think\Controller`**，通过 trait 混入获得 CRUD action，属性在 `initialize()` 中设置。

## Trait 组合

| 组合 | 场景 |
|---|---|
| `use HasBuilder;` | **标准 CRUD 首选**（含列表+增删改查+Autopost+Enable+Delete） |
| `use HasIAED, HasAutopost;` | 分类/字典 + 行内编辑 |
| `use HasIAED, HasAutopost, HasView;` | 多级分类 + 详情页 |
| 逐颗粒 `use actions\HasBase; HasIndex; ...` | 非标准 action |

> ❌ 严禁 `HasBuilder` 再加 `HasIAED/HasAutopost/HasView` — 重复 use 触发 Fatal。

## 按任务读取参考文件

| 任务 | 读取 |
|---|---|
| 新建 CRUD 控制器 | `references/controller-tpl.md` |
| 写表单字段 | `references/form-fields.md` |
| 写表格列 / 工具栏 | `references/table-columns.md` |
| 表单高级模式（Tab/Step/联动） | `references/form-patterns.md` |
| Builder API / Hook / Layer | `references/builder.md` |

> 💡 **新模块从 `controller-tpl.md` 开始**：含完整骨架、属性速查、自定义 Action 模板。

## 频率标注图例

| 标记 | 含义 | 生成代码策略 |
|---|---|---|
| ⭐⭐⭐ 高频 | 大量使用 | 主流写法，可直接照搬 |
| ⭐⭐ 中频 | 部分使用 | 按业务场景选用，无对应需求勿加 |
| ⭐ 低频/不常用 | 少用 | 特定场景才用，**勿默认生成** |

**高频模式**：Tab 分页、thumbSize/cut、mapClass、layerOpen 弹窗、btnLink、dataWithId 子表、withNext 级联。

**不常用模式**：分步表单 step、buildDataList 覆写、doSave 覆写、canDel/afterDel、reInit、matches。

## 属性名速记

| 分类 | 正确属性名 | ❌ 常见错误 |
|---|---|---|
| 列表排序 | `sortOrder` | `indexOrder` |
| 列表字段 | `indexFieldsOnly` / `indexFieldsExcept` | `indexSelect` / `indexFields` |
| 行内编辑白名单 | `postAllowFields`（数组） | 字符串 / `autopostFields` |
| 禁用删除 ID | `delNotAllowed`（数组） | `notDelId` |
| selectPage 前缀 | `selectSearch / selectFields / selectTextField / selectIdField / selectWith / selectOrder / selectPagesize / selectScope` | `selectPage*` 前缀 |
| 左树前缀 | `treeModel / treeTextField / treeKey / treeIdField / treeParentIdField / treeSortField / treeExpandAll / treeRootText / treeType / treeScope` | — |
| 导出列控制 | `exportOnly` / `exportExcept`（数组） | — |

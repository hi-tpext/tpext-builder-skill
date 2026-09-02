---
name: tpext-builder-skill
description: "tpext-builder (webman + ThinkORM + LightYearAdmin) CRUD assistant. Invoke when: writing/editing controllers under app/admin/controller/, using tpext-builder traits (HasBuilder/HasIAED/HasAutopost), building Form/Table/Search APIs, debugging trait/field/when() issues, or when user mentions tpext-builder, LightYearAdmin, admin CRUD, 后台管理, 增删改查."
---

# tpext-builder 后台 CRUD 开发助手

基于 **tpext-builder**（`vendor/ichynul/tpextbuilder/`，ThinkORM 后台 UI 生成器，UI 主题 LightYearAdmin / Bootstrap 3）。所有 API 以源码为准，不要凭经验臆造。

> 📚 **配套文档**（本文件仅列红线和速查，完整示例跳转阅读）：
>
> - [references/controller-tpl.md — 控制器模板 + 属性速查 + 自定义 Action + Hook 覆写](./references/controller-tpl.md)
> - [references/form-fields.md — 全部表单字段类型 + 通用方法 + items/fields + When](./references/form-fields.md)
> - [references/table-columns.md — 全部表格列类型 + 工具栏/行按钮 + mapClass](./references/table-columns.md)
> - [references/form-patterns.md — Tab/Step/左右分栏/When 联动/withNext 级联](./references/form-patterns.md)
> - [references/builder.md — Action Trait 对照 + Layer 返回 + JS/CSS 注入 + Hook](./references/builder.md)

***

## 一、绝对红线

| ❌ 错误 | ✅ 正确 | 原因 |
|---|---|---|
| 控制器不继承 `think\Controller` | `extends \think\Controller` | 缺失导致 success/error/redirect 不可用 |
| `use HasBuilder, HasIAED, HasView` | `use HasBuilder;` 一行即可 | HasBuilder 已包含 8 个 trait，重复 use 触发 Fatal |
| `use HasIAED;` 不配 HasAutopost | `use HasIAED, HasAutopost;` | 缺 HasAutopost → 行内编辑路由 404 |
| `$this->dataModel = SomeModel::class` | `$this->dataModel = new SomeModel;` | 必须是实例，框架调 `$this->dataModel->where(...)` |
| `$this->indexOrder` | `$this->sortOrder` | HasIndex 源码只有 sortOrder |
| `$this->indexSelect` | `$this->indexFieldsOnly` | 源码属性是 indexFieldsOnly |
| `$this->postAllowFields = 'name'` | `$this->postAllowFields = ['name']` | 必须数组 |
| `buildForm(&$data, &$fields)` | `buildForm($isEdit, &$data = [])` | 第 1 参：0=add/1=edit/2=view |
| `$form->when($cond)->with(...)` | `$form->select('type')->when(1)->with(...)` | when 必须写在触发字段上 |
| `->when(['sms','wx'])` | `->when('sms+wx')` | 多选联动用加号拼接 |
| where 用 `'eq'/'gt'/'lt'` | where 用 `= > < >= <= <>` | 标准 SQL 符号，mapClass 的 logic 参数除外 |
| `$step1 = $form->step(...); $step1->text(...)` | `$form->step(...); $form->text(...)` | step 返回 FieldsContent，字段继续用 `$form->` 调用 |
| `$form->selectPage('x','X')` | `$form->select('x','X')->dataUrl(...)` | Wrapper 没有 selectPage displayer |
| `selectPageLimit/selectPageShowField/...` | `selectSearch/selectFields/selectTextField/...` | 8 个 `selectPage*` 全是编造属性 |

> 📝 已修正的旧版错误：
>
> - `canNotAddOrDelete()` / `canRecover()` 在外层 `Items`（`displayer/Items.php`）确实存在
> - `mapClass($v, $cls, $field, $logic)` 第 4 参接受 `eq/gt/lt/in_array` 等字符串，也接受 `== > <` 符号
> - 省市区级联 API 路径是 `/api/areacity/{level}`（province/city/area/town），不是 `/api/region/*`

***

## 二、Trait 组合

| 组合 | 场景 |
|---|---|
| `use HasBuilder;` | **标准 CRUD 首选**：列表+增删改查+Autopost+Enable+Delete |
| `use HasIAED, HasAutopost;` | 分类/字典：行内用 `switchBtn(...)` 切字段 |
| `use HasIAED, HasAutopost, HasView;` | 多级分类/树形：上面 + 独立详情页 |
| 颗粒度 `use actions\HasBase; HasIndex; ...` | 非标准 action：只读/日志/只增不改 |

> ❌ `HasBuilder` 再加 `HasIAED/HasAutopost/HasView` = Fatal Error。
>
> 颗粒度模式需先 `use tpext\builder\traits\actions;` 再 `use actions\HasBase;` 短路径。

***

## 三、骨架（生成 CRUD 直接用）

```php
<?php
namespace app\admin\controller;

use think\Controller;
use tpext\builder\traits\HasBuilder;

class YourCrud extends Controller
{
    use HasBuilder;

    protected function initialize()
    {
        $this->dataModel = new \app\common\model\YourModel; // ✅ 实例，不是类名
        $this->pageTitle = '模块标题';
        // $this->postAllowFields = ['name','sort','status'];  // 行内编辑字段数组
        // $this->sortOrder = 'id desc';           // 默认已 'id desc'，真要改才写
        // $this->pagesize   = 14;                  // 默认已 14，勿写
        // $this->delNotAllowed = [1];              // 禁止删除的 ID
        // $this->indexWith     = ['relationName']; // 关联预加载
    }

    /**
     * @param int $isEdit  0=新增, 1=编辑, 2=查看(view)
     * @param array $data  已有数据引用
     */
    protected function buildForm($isEdit, &$data = [])
    {
        $form = $this->form;
        $form->show('id', 'ID')->when($isEdit > 0);
        $form->hidden('id');

        $form->text('name', '名称')->required();
        $form->number('sort', '排序号')->default(0);
        $form->radio('status', '状态')->options([1=>'启用',0=>'禁用'])->default(1);
    }

    protected function buildTable()
    {
        $table = $this->table;
        $table->show('id', 'ID');
        $table->text('name', '名称')->autoPost();
        $table->text('sort', '排序号')->autoPost('', true);
        $table->switchBtn('status', '启用')->default(1)->autoPost();

        $table->getActionbar()
            ->btnEdit()
            ->btnView()
            ->btnEnableAndDisable()
            ->btnDelete();

        $table->getToolbar()
            ->btnAdd()
            ->btnEnableAndDisable()
            ->btnDelete();
    }

    protected function buildSearch()
    {
        $search = $this->search;
        $search->text('name', '名称', 3)->maxlength(20);
    }

    protected function save($id = 0)
    {
        return $this->doSave($id, request()->only(['name','sort','status']));
    }
}
```

### 配套 common model 最简模板

```php
<?php
namespace app\common\model;

class YourModel extends \think\Model
{
    protected $name = 'your_table_name'; // 去掉前缀，框架 prefix 自动补
    protected $pk   = 'id';
    protected $autoWriteTimestamp = true; // 有 create_time/update_time 时用
    // protected $jsonFields = ['attr_list']; // 只在 items / JSON 字段时写
}
```

***

## 四、两个最高频 Bug

| 症状 | 根因 | 修复 |
|---|---|---|
| 新增/编辑提交后字段丢失（DB 是 NULL） | save() 里 `only([...])` 漏了该字段名 | 补进 only 数组 |
| 行内编辑 AJAX 提示「字段不允许修改」 | postAllowFields 数组里没有该字段 | initialize() 里加进数组 |

***

## 五、生成前检查清单

1. `extends Controller` + trait 组合无重复
2. `$this->dataModel = new XxxModel;`（实例，不是类名）
3. initialize() 只写与默认值不同的属性
4. buildForm 第一参是 `$isEdit`
5. save() 的字段获取与 postAllowFields 同步
6. items/fields 用 `->with()` 时内部自动调用 end，不用 `->with()` 的开放写法需手动调 `itemsEnd()`/`fieldsEnd()`
7. mapClass 第 4 参在合法值列表：`eq/gt/lt/egt/elt/in_array/not_in_array/strstr/not_strstr`
8. select 属性前缀是 `select*`，不要写 `selectPage*`
9. where 操作符统一 `= > < >= <= <>`，不用 `eq/gt/lt`
10. 自定义 Action 方法头加 `@title` 注释

***

## 六、频率标注说明

各文档 📊 标记含义：

- ⭐⭐⭐ **高频**：主流写法，可直接照搬（Tab 分页、layerOpen 弹窗、btnLink、dataWithId、mapClass、thumbSize/cut 等）
- ⭐⭐ **中频**：按业务场景选用，无对应需求勿加（When 联动、左右分栏、fields 内联组、JS/CSS 注入等）
- ⭐ **低频/不常用**：特定场景才用，**勿默认生成**（分步表单 Step、doSave 覆写、buildDataList 覆写、canDel/afterDel、reInit、matches 等）

# 控制器开发指南

> 基于项目 `app/admin/controller/` 实际代码 + `vendor/ichynul/tpextbuilder/src/traits/` 源码分析。

***

## 一、Trait 选择

| 模式 | Trait | 场景 |
|---|---|---|
| ① 标准 CRUD | `use HasBuilder;` | 会员/商品/订单/套餐等完整管理 |
| ② 分类/字典 | `use HasIAED, HasAutopost;` | 分类表、标签表、字典表，行内编辑 |
| ③ 只读列表 | `use actions\HasBase; HasIndex; HasView;` | 日志、记录，只看不改 |
| ④ 只增不改 | `use actions\HasBase; HasIndex; HasAdd; HasView;` | 账户操作记录，只允许新增 |
| ⑤ 颗粒度 | 逐个 `use actions\HasXxx;` | 按业务自由组合 |

> 铁律：HasBuilder = HasBase+HasIndex+HasAdd+HasEdit+HasView+HasAutopost+HasEnable+HasDelete，**再写 HasIAED 等就是重复 use**。
>
> 颗粒度模式需先 `use tpext\builder\traits\actions;` 再 `use actions\HasBase;` 短路径。

***

## 二、标准 CRUD 骨架（模式①）

```php
<?php
namespace app\admin\controller;

use think\Controller;
use tpext\builder\traits\HasBuilder;
use app\common\model\YourModel;

/**
 * @title 管理标题
 */
class YourCrud extends Controller
{
    use HasBuilder;

    /** @var YourModel */
    protected $dataModel;

    protected function initialize()
    {
        $this->dataModel = new YourModel;
        $this->pageTitle = '管理标题';
        // 只写与默认不同的属性，见下方属性速查
    }

    protected function filterWhere()
    {
        $searchData = request()->get();
        $where = [];
        if (!empty($searchData['name'])) {
            $where[] = ['name', 'like', '%' . $searchData['name'] . '%'];
        }
        if (isset($searchData['status']) && $searchData['status'] !== '') {
            $where[] = ['status', '=', $searchData['status']];
        }
        return $where;
    }

    protected function buildSearch()
    {
        $search = $this->search;
        $search->defaultDisplayerColSize(3);
        $search->text('name', '名称')->maxlength(20);
        $search->select('status', '状态')->options(['' => '全部', 1 => '启用', 0 => '禁用']);
    }

    protected function buildTable(&$data = [], $isExporting = false)
    {
        $table = $this->table;
        $table->show('id', 'ID');
        $table->text('name', '名称')->autoPost();
        $table->text('sort', '排序')->autoPost('', true);
        $table->switchBtn('status', '启用')->autoPost();
        $table->show('create_time', '添加时间')->getWrapper()->addStyle('width:150px');

        $table->getToolbar()
            ->btnAdd()
            ->btnEnableAndDisable()
            ->btnDelete()
            ->btnRefresh();

        $table->getActionbar()
            ->btnEdit()
            ->btnView()
            ->btnDelete();
    }

    protected function buildForm($isEdit, &$data = [])
    {
        $form = $this->form;
        if ($isEdit) {
            $form->hidden('id');
        }
        $form->text('name', '名称')->required()->maxlength(55);
        $form->number('sort', '排序')->default(0);
        $form->switchBtn('status', '状态')->default(1);
        if ($isEdit) {
            $form->show('create_time', '添加时间');
            $form->show('update_time', '修改时间');
        }
    }

    protected function save($id = 0)
    {
        $data = request()->only(['name', 'sort', 'status'], 'post');

        $result = $this->validate($data, [
            'name|名称' => 'require',
        ]);
        if (true !== $result) {
            $this->error($result);
        }

        return $this->doSave($data, $id);
    }
}
```

***

## 三、只读列表骨架（模式③）

```php
use tpext\builder\traits\actions;

class Yourlog extends Controller
{
    use actions\HasBase;
    use actions\HasIndex;
    use actions\HasView;

    protected function initialize()
    {
        $this->dataModel = new YourLogModel;
        $this->pageTitle = '操作日志';
        $this->indexWith = ['member', 'admin'];
    }

    // filterWhere / buildSearch 同标准 CRUD

    protected function buildTable(&$data = [], $isExporting = false)
    {
        $table = $this->table;
        $table->show('id', 'ID');
        $table->show('desc', '描述');
        $table->show('admin_id', '操作人')->to('{admin.username}({admin.name})');
        $table->show('create_time', '操作时间')->getWrapper()->addStyle('width:150px');

        $table->useCheckbox(false);
        $table->sortable([]);

        $table->getToolbar()->btnRefresh();
        $table->getActionbar()->btnView();
    }

    protected function buildForm($isEdit, &$data = [])
    {
        if ($isEdit == 2) {
            $form = $this->form;
            $form->show('desc', '描述');
            $form->show('create_time', '操作时间');
        }
    }
    // ❌ 不写 save() —— 没 HasAdd/HasEdit，POST 路由 404
}
```

***

## 四、initialize() 属性速查

**铁律：只写与默认值不同的属性，默认值不写。**

### 必填

| 属性 | 说明 |
|---|---|
| `$this->dataModel = new XxxModel;` | 必须实例 |
| `$this->pageTitle = '标题';` | 默认 'Page' |

### 列表相关

| 属性 | 默认值 | 说明 |
|---|---|---|
| `sortOrder` | `'id desc'` | 排序 |
| `pagesize` | `14` | 每页条数 |
| `useSearch` | `true` | 是否显示搜索 |
| `tableDelay` | `false` | 延迟加载表格 |
| `enableField` | `'enable'` | 启用/禁用字段名 |
| `indexWith` | `[]` | 关联预加载，供 `show('rel.field')` |
| `indexFieldsOnly` | `'*'` | 大表收窄字段 |
| `indexFieldsExcept` | `''` | 排除字段 |
| `postAllowFields` | `[]` | 行内编辑白名单，**必须数组** |
| `delNotAllowed` | `[]` | 禁止删除的 ID 数组 |
| `exportOnly` | `[]` | 导出限制列（数组） |
| `exportExcept` | `[]` | 导出排除列（数组） |

### selectPage 远程下拉

| 属性 | 默认值 | 说明 |
|---|---|---|
| `selectSearch` | `''` | 搜索字段，多字段用 `\|` 分隔 |
| `selectFields` | `'*'` | 返回字段 |
| `selectTextField` | `''` | 显示格式，如 `'{id}#{real_name}({mobile})'` |
| `selectIdField` | `''` | 空时取 pk |
| `selectWith` | `[]` | 关联预加载 |
| `selectOrder` | `''` | 空时取 sortOrder |
| `selectPagesize` | `20` | 每页条数 |
| `selectScope` | `[]` | 固定 where 条件 |

### 左树（HasLeftTree trait 自动启用）

| 属性 | 说明 |
|---|---|
| `treeModel` | 分类模型实例，必填 |
| `treeTextField` | 分类名称字段 |
| `treeKey` | 主表外键字段名（如 `category_id`） |
| `treeIdField` / `treeParentIdField` / `treeSortField` | 空时取树模型配置 |
| `treeRootText` | 空显示"全部" |
| `treeType` | 默认 `'ztree'`，可改 `'jstree'` |
| `treeExpandAll` | 默认 `true` |
| `treeScope` | 默认 `[]` |

***

## 五、搜索组件（buildSearch + filterWhere）

buildSearch 定义搜索面板字段，filterWhere 解析搜索条件返回 where 数组。**两者字段名必须一一对应。**

```php
protected function buildSearch()
{
    $search = $this->search;
    $search->defaultDisplayerColSize(3); // 统一列宽，4 个一行

    $search->text('name', '名称')->maxlength(20);
    $search->select('status', '状态')->options(['' => '全部', 1 => '启用', 0 => '禁用']);
    $search->select('member_id', '会员')->dataUrl(url('/admin/member/member/selectPage'));
    $search->multipleSelect('type_id', '类型')->dataUrl(url('/admin/sometype/selectPage'));

    // Tab 快速切换（选值自动 GET 提交）
    $search->tabLink('pay_status')->options(['' => '全部'] + $payStatusOptions);

    // 起止时间（第二个 label 用 '~' 节省空间）
    $search->datetime('start', '操作时间')->placeholder('起始');
    $search->datetime('end', '~')->placeholder('截止');

    // URL 参数锁定上下文
    if (input('member_id/d')) {
        $search->hidden('member_id')->value(input('member_id/d'));
    } else {
        $search->select('member_id', '会员')
               ->dataUrl(url('/admin/member/member/selectPage'));
    }

    // 级联搜索（默认列宽不够时可单独覆盖）
    $search->select('province', '省份', 4)
           ->dataUrl(url('/api/areacity/province'), 'ext_name')
           ->withNext(
               $search->select('city', '城市', 4)
                      ->dataUrl(url('/api/areacity/city'), 'ext_name')
           );
}

protected function filterWhere()
{
    $searchData = request()->get();
    $where = [];

    if (!empty($searchData['name'])) {
        $where[] = ['name', 'like', '%' . $searchData['name'] . '%'];
    }
    if (isset($searchData['status']) && $searchData['status'] !== '') {
        $where[] = ['status', '=', $searchData['status']];
    }
    if (!empty($searchData['member_id'])) {
        $where[] = ['member_id', '=', $searchData['member_id']];
    }

    return $where;
}
```

***

## 六、翻译文件 — 字段 label 省略写法

当控制器加载了翻译文件后，字段的第二个参数（label）可以省略，框架会自动从翻译文件中查找。

### 1. 创建翻译文件

路径：`app/admin/lang/zh-cn/{controllername}.php`（文件名小写，与控制器名对应）

```php
<?php
// app/admin/lang/zh-cn/smslog.php
return [
    'id'                => '主键',
    'mobile'            => '手机号',
    'ip'                => 'IP地址',
    'code'              => '验证码',
    'create_time'       => '添加时间',
    'create_time_start' => '添加时间起',
    'create_time_end'   => '添加时间止',
    'update_time'       => '更新时间',
];
```

### 2. 在 initialize() 中加载翻译文件

```php
use tpext\think\App;
use think\facade\Lang;

protected function initialize()
{
    $this->dataModel = new SmsLogModel;
    $this->pageTitle = '短信记录';

    // 加载翻译文件
    Lang::load(App::getAppPath() . implode(DIRECTORY_SEPARATOR, [
        'admin', 'lang', App::getDefaultLang(), 'smslog.php'
    ]));
}
```

> `App::getRootPath()` 也可以替代 `App::getAppPath()`，区别是路径前缀不同（根目录 vs app 目录）。

### 3. 字段省略 label

翻译文件加载后，字段第二个参数可以省略：

```php
// 有翻译文件时：省略 label
$search->text('mobile');                // 自动查找 Lang::get('Mobile') → '手机号'
$search->text('ip');
$search->datetime('create_time_start');

$form->text('name');                    // 自动查找 Lang::get('Name') → '名称'
$form->number('sort');
$table->show('create_time');            // 自动查找 Lang::get('Create_time') → '创建时间'

// 没有翻译文件时：必须写 label
$search->text('mobile', '手机号');
$form->text('name', '名称');
$table->show('create_time', '创建时间');
```

> 查找机制：`Field::__construct($name, $label)` 中，当 `$label` 为空时，执行 `Lang::get(ucfirst($name))`，即把字段名首字母大写后去翻译表查找。找不到时返回原始 key 名。

### 4. 混合使用

翻译文件不需要覆盖所有字段，未覆盖的字段仍然可以手动传 label：

```php
// 翻译文件中有 'mobile' => '手机号'
$search->text('mobile');                // ✅ 自动翻译

// 翻译文件中没有 'keyword'
$search->text('keyword', '关键词');      // ✅ 手动指定
```

***

## 七、自定义 Action（弹窗表单模式）

项目高频模式：列表行按钮 → 弹窗表单 → 提交后关闭刷新。

```php
/**
 * @title 修改推荐人
 * @param int $member_id
 * @return mixed
 */
public function changeLeader($member_id)
{
    $builder = $this->builder('用户管理', '修改推荐人');
    $member = $this->dataModel->find($member_id);
    if (empty($member)) {
        return $builder->layer()->close(0, '用户不存在！');
    }

    if (request()->isGet()) {
        $form = $builder->form();
        $form->show('nickname', '用户')->to('{id}#{real_name}');

        $form->radio('change_type', '改变方式')
             ->options([1 => '选择推荐人', 2 => '设置为无'])
             ->default(1)->required()
             ->when(1)->with(
                 $form->select('change_leader_id', '新的推荐人')
                      ->dataUrl(url('/admin/member/member/selectPage'))
             )->when(2)->with(
                 $form->hidden('change_leader_id')->value('-1'),
                 $form->show('tips', '提示')->value('将设为无推荐人')
             );

        $form->textarea('change_desc', '操作备注')->maxlength(200);
        $form->fill($member);
        return $builder;
    }

    $data = request()->post();
    $result = $this->validate($data, [
        'change_type|改变方式' => 'require',
        'change_leader_id|新推荐人' => 'require',
    ]);
    if (true !== $result) {
        $this->error($result);
    }

    // 业务逻辑...
    return $builder->layer()->closeRefresh(1, '保存成功');
}
```

在 buildTable 中添加行按钮：

```php
$table->getActionbar()
    ->btnLink('changeLeader',
        url('/admin/member/member/changeLeader', ['member_id' => '__data.pk__']),
        '推荐人', 'btn-secondary', '', 'title="修改推荐人"');
```

***

## 八、doSave() 覆写 — 联动写多表

```php
protected function doSave($data, $id = 0)
{
    $res = 0;
    if ($id) {
        $exists = $this->dataModel->where([$this->getPk() => $id])->find();
        if ($exists) {
            $res = $exists->force()->save($data);
        }
    } else {
        $res = $this->dataModel->exists(false)->save($data);
        if ($res) {
            $id = $this->dataModel->id;
        }
    }

    if (!$res) {
        $this->error(__blang('bilder_save_failed'));
    }

    // 联动写子表
    $profileData = input('profile/a');
    $profile = MemberProfile::where('member_id', $id)->find() ?? new MemberProfile();
    $profileData['member_id'] = $id;
    $profile->save($profileData);

    return $this->builder()->layer()->closeRefresh(1, __blang('bilder_save_succeeded'));
}
```

***

## 九、buildDataList() 覆写 — 列表数据源不走 dataModel

当列表数据来自 `db()` join、跨库、外部 API 时覆写。**必须写入 `&$total` 参数**。

```php
/**
 * @param array $where
 * @param string $sortOrder
 * @param int $page
 * @param int &$total 引用传入，必须写入
 */
protected function buildDataList($where = [], $sortOrder = '', $page = 1, &$total = -1)
{
    $data  = [];
    $total = 0;

    $query = Db::table('your_table a')
        ->join('category c', 'a.category_id = c.id', 'LEFT')
        ->field('a.*, c.name as category_name')
        ->where($where);

    $total = $query->count();
    if ($total == 0) {
        return $data;
    }

    $data = $query->order($sortOrder)
        ->limit(($page - 1) * $this->pagesize, $this->pagesize)
        ->select();

    return $data;
}
```

***

## 十、canDel() / afterDel() — 删除前后置逻辑

签名都是**单条 `$id`**（不是数组）。批量删除时框架逐条调用。

```php
protected function canDel($id)
{
    if ($this->dataModel->where(['parent_id' => $id])->count() > 0) {
        $this->error('有下级，不可删除');
        return false;
    }
    return true;
}

protected function afterDel($id)
{
    SomeSubModel::where(['main_id' => $id])->delete();
    return true;
}
```

***

## 十一、TreeModel reInit() — 动态切换树范围

```php
protected function initialize()
{
    $this->dataModel = new CategoryModel;
    $this->treeModel = new CategoryModel;

    $store_id = input('store_id/d', 1);
    $this->dataModel->reInit(['treeScope' => [['store_id', '=', $store_id]]]);
    $this->treeModel->reInit(['treeScope' => [['store_id', '=', $store_id]]]);
}
```

***

## 十二、URL 控制默认激活 Tab

```php
protected function buildForm($isEdit, &$data = [])
{
    $form = $this->form;
    $tab = input('tab', 0);

    $form->tab('基本信息',   $tab == 0);
    $form->tab('规格型号',  $tab == 1);
    $form->tab('库存价格',  $tab == 2);
    // ... 字段 ...
}
```

> 表格里 `btnEdit` 也能跳 Tab：`->btnEdit(url('edit', ['id' => '__data.pk__', 'tab' => 1]))`

***

## 十三、Hook 方法签名

| 方法 | 签名 | 用途 |
|---|---|---|
| `creating($builder, $type)` | `(Builder, string): void` | 全局注入 JS/CSS |
| `filterWhere()` | `(): array` | 返回 where 数组 |
| `buildSearch()` | `(): void` | 往 `$this->search` 加字段 |
| `buildTable(&$data, $isExporting)` | `(array, bool): void` | 往 `$this->table` 加列 |
| `buildForm($isEdit, &$data)` | `(int, array): void` | 0=add, 1=edit, 2=view |
| `save($id)` | `(int): mixed` | 接收 POST → 校验 → doSave |
| `doSave($data, $id)` | `(array, int): mixed` | 底层写入 |
| `canDel($id)` | `(int\|string): bool` | 返回 false 阻止删除 |
| `afterDel($id)` | `(int\|string): bool` | 删除后清理 |
| `buildDataList($where, $sort, $page, &$total)` | `(array, string, int, int): array` | 覆写数据源 |
| `exportTo($data, $displayers, $type, $table)` | `(array, array, string, Table): mixed` | 自定义导出格式 |

***

## 十四、Bug 自检

| 问题 | 原因 | 修复 |
|---|---|---|
| 保存字段丢失 | save() 里 only([...]) 漏了 | 同步 only 数组 |
| autopost 报"字段不允许" | postAllowFields 没有 | initialize 里添加 |
| `show('rel.field')` 空 | indexWith 没加 | 同步 indexWith |
| 自定义 action 404 | 方法头没加 @title | 加 @title + 后台补建权限节点 |
| When 联动不生效 | when() 写在 $form 而非触发字段 | 写在触发字段上 |
| items 保存空数组 | 不用 with() 的开放写法没调 itemsEnd() | 检查 itemsEnd |
| Fatal "Trait not found" | HasBuilder + HasIAED 重复 use | 去掉 HasIAED |
| doSave 覆写后保存失败 | 没用 force()->save() / exists(false)->save() | 见 doSave 覆写 |
| buildDataList 不分页 | 签名没写 &$total 参数 | 用 `(&$total)`，方法体 `$total = N` |
| 行内编辑无权限 | 没配 autopost 节点 | 权限节点加 autopost action |
| 自定义 action 弹窗不弹出 | 列里超链接缺 `onclick="layerOpen(this);return false;"` | 加 onclick 或改用 btnLink |
| 左树点击不搜索 | 搜索字段名跟 treeKey 不一致 | 统一名字 + filterWhere 处理 |

***

## 十五、字段两处同步铁律

每次新增/删除可写字段，**两处必须同步**：

| 位置 | 影响范围 | 漏写症状 |
|---|---|---|
| `save()` 的 `request()->only([...])` | add/edit 正常保存 | 字段保存丢失，DB 是 NULL |
| `initialize()` 的 `$this->postAllowFields = [...]` | 行内 autopost | AJAX 报「字段不允许修改」 |

# 表格列速查

所有列通过 `$table->{类型}('字段名', '标签')` 创建，支持链式。

***

## 一、全部列类型

### 基础展示

```php
$table->show('id', 'ID');                                    // 纯文本
// $table->show('category.name', '分类');                        // 关联字段（需 indexWith） 不推荐此写法，因为category.name这个名称没有对应到真实字段，如果需要排序/翻译时就做不到
$table->show('category_id', '分类')->to('{category.name}');      // 关联字段（需 indexWith） 推荐此写法
$table->show('about', '简介')->cut(50);                      // 超长截断
$table->text('name', '名称')->autoPost();                    // 文本 + 行内编辑
$table->raw('__text__', '结构');                              // 树形结构模型自带字段
$table->raw('icon', '图标')->to('<i class="{val}"></i>');    // 自定义 HTML
```

### 图像

```php
$table->image('logo', '封面图')->thumbSize(60, 60);
$table->image('avatar', '头像')->thumbSize(50, 50)->default('/static/images/logo.png');
$table->images('images', '相册')->thumbSize(40, 40);  // 多图
```

### 匹配映射

```php
// 单选匹配
$table->match('status', '状态')->options([1 => '启用', 0 => '禁用'])
      ->mapClassGroup([[1, 'label-success'], [0, 'label-default']]);

// 数据集匹配 + 兜底
$table->match('agent_level', '代理等级')
      ->optionsData($agentLevels, 'name', 'level')
      ->beforOptions([0 => '普通用户']);

// 多选匹配（⭐ 低频，逗号分隔存储的标签）
$table->matches('tags', '标签')
      ->optionsData(SomeTagModel::field('id,name')->select(), 'name');
```

### 可行内编辑

```php
$table->switchBtn('is_show', '显示')->autoPost();
$table->switchBtn('status', '启用')->autoPost('', true);     // 二次确认，false则修改后自动提交无确认
$table->text('sort', '排序')->autoPost('', true);
$table->text('stock', '库存')->autoPost('', '确认要修改库存吗？');// 自定义确认提示
$table->select('type_fk_id', '分类')->dataUrl(url('...'));
$table->checkbox('type_fk_id', '分类')->dataUrl(url('...'));
$table->checkbox('attr', '属性')->autoPost(url('setAttr'))//指定不同的url地址，一般用于权限精细划分、特殊逻辑处理等场景
      ->options(['is_recommend' => '推荐', 'is_hot' => '热门']);
$table->radio('type', '类型')->options($opts);
```

### 关联字段 + 弹窗链接（⭐⭐⭐ 最高频）

```php
// 点语法（需 indexWith）,不推荐此写法
$table->show('category.name', '分类');
$table->show('user.nickname', '用户');

// to() 格式化
$table->show('admin_id', '管理员')->to('{val}#{admin.username}({admin.name})');//{val} 表示当前字段，在这里和{admin_id}等效
$table->show('first_leader', '推荐人')->to('{first_leader}#{firstleader.real_name}');

// to() 闭包
$table->show('money', '金额')->to(function ($val) {
    return $val > 0 ? '+' . $val : '' . $val;
});

// to() 闭包查子表
$table->raw('time_ranges', '活动时间')->to(function ($val, $d) {
    $ranges = SomeModel::where(['goods_id' => $d['id']])->column('time_range');
    return count($ranges) ? implode('<br>', $ranges) : '暂未设置';
});

// 弹窗链接（最高频）
$table->raw('member.real_name', '姓名')
      ->to('<a href="#" onclick="layerOpen(this);return false;" title="会员信息"
            data-url="/admin/member/member/view?id={member_id}">{val}</a>');

// 带弹窗尺寸
$table->raw('consignee', '收货人')
      ->to('<a href="#" data-layer-size="1000px,600px"
            onclick="layerOpen(this);return false;" title="修改地址"
            data-url="/admin/shop/order/editaddress?id={id}">{val}[修改]</a>');

// 值 + 弹窗链接
$table->match('status', '状态')->options([1 => '正常', 0 => '禁用'])
      ->to('{val}<a href="#" onclick="layerOpen(this);return false;" title="状态"
            data-url="/admin/member/member/changeStatus?member_id={id}">[修改]</a>');
```

> `{val}` = 当前字段值，`{字段名}` = 行数据，`{关联.字段}` = 关联字段（需 indexWith）。

### fields — 字段组（多列合并）

```php
$table->fields('nickname', '会员ID/会员')->with(
    $table->show('member_id', '会员id')->inline(true),
    $table->raw('member.real_name', '姓名')->inline(true)
          ->to('<a href="#" onclick="layerOpen(this);return false;" title="会员信息"
                data-url="/admin/member/member/view?id={member_id}">{val}</a>')
);

$table->fields('real_name', '实名信息')->with(
    $table->show('real_name', '真实姓名'),
    $table->show('idcard_no', '身份证号')->to(function ($val) {
        return $val ? substr($val, 0, 4) . '****' . substr($val, 14) : '';
    })
);

// 内部用 field() 拼接
$table->fields('cut', '余额/积分')->with(
    $table->field('use_money', '余额')->to('{val} / '),
    $table->field('points_money', '积分')->to('{val}')
);
```

> 内部子列用 `->inline(true)` 让多列横向并排（默认换行）。

***

## 二、支持 autoPost() 的列

| 列类型 | `autoPost($url, $confirm)` |
|---|---|
| `text` | ✅ `$url` 默认 autopost action |
| `switchBtn` | ✅ |
| `sort` | ✅ |
| `checkbox` | ✅ `$url` 必填（自定义 action） |

***

## 三、mapClass 条件样式

```php
// 基于本字段值
$table->show('order_amount', '订单金额')
      ->mapClass(10000, 'text-danger', 'order_amount', 'gt');  // > 10000 加红
$table->show('left_money', '余额')
      ->mapClass(0, 'text-muted', 'left_money', 'elt');        // <= 0 灰掉

// 基于其他字段
$table->text('stock', '库存')
      ->mapClass([1], 'disabled', 'has_multi_price');  // has_multi_price==1 时灰掉

// mapClassGroup 多值分组
$table->match('pay_status', '支付状态')->options($opts)
      ->mapClassGroup([[1, 'label-success'], [2, 'label-danger']]);
```

第 4 参 logic 可选值：`in_array`（默认）/ `not_in_array` / `eq` / `gt` / `lt` / `egt` / `elt` / `strstr` / `not_strstr`

### mapClass 订单管理完整示例

```php
// 先处理数据，附加 __hi_* 变量
foreach ($data as &$d) {
    // 只有 未付款 + 未发货 + 订单状态≤1 才能取消
    $d['__hi_order_cancel__'] = !($d['order_status'] <= 1
        && $d['pay_status']   == 0
        && $d['shipping_status'] == 0);

    // 已支付的订单不可删除
    $d['__hi_del__'] = $d['pay_status'] == 1
        || $d['shipping_status'] != 0
        || !in_array($d['order_status'], [3, 5]);

    // id=1 禁用删除（不是隐藏）
    $d['__dis_del__'] = $d['id'] == 1;
}
unset($d);

$table->getActionbar()
    ->btnEdit()->btnView()->btnDelete()
    ->btnLink('deliver', url('/admin/order/deliver', ['fk_id'=>'__data.pk__']), '发货', 'btn-brown', '')
    ->mapClass([
        'edit'    => ['hidden'   => '__hi_edit__'],
        'delete'  => ['hidden'   => '__hi_del__', 'disabled' => '__dis_del__'],
        'deliver' => ['hidden'   => '__hi_deliver__'],
    ]);
```

***

## 四、通用列方法

```php
->to($cb_or_fmt)                 // 格式化：闭包 / '{field}' / HTML
->default($val)                  // 空时默认值
->cut($len)                      // 超长截断
->getWrapper()->addStyle('...')  // 列宽/样式
->getWrapper()->addClass('...')  // CSS 类
->colHidden(true)                // 默认隐藏
->colSortable(true)              // 单独设置可排序
->inline(true)                   // fields 内部横向并排
```

***

## 五、Toolbar 工具栏

```php
$table->getToolbar()
    ->btnAdd()
    ->btnAdd(url('add', ['id' => '__data.pk__']), '添加', 'btn-primary', 'mdi-plus',
        'data-layer-size="98%,98%"')                      // 带参 + 控制弹窗尺寸
    ->btnEnableAndDisable('启用', '禁用')
    ->btnDelete()
    ->btnRefresh()
    ->btnToggleSearch()                                    // 搜索面板切换
    ->btnExport(url('export'), '导出', 'btn-success')      // 单格式导出
    ->btnExports([                                         // 多格式导出下拉
        'csv'  => 'CSV',
        'xlsx' => 'XLSX',
    ])
    ->btnLink('下载模板', '/template/模板.xls', '_blank', 'btn-info', 'mdi-download')
    ->btnPostChecked(url('batchPrint'), '批量打印', 'btn-warning')  // 勾选后 POST
    ->html('提示：先勾选再操作');                           // 自定义 HTML
```

***

## 六、Actionbar 行按钮

```php
$table->getActionbar()
    // 标准按钮
    ->btnEdit()                                         // key = 'edit'
    ->btnView()                                         // key = 'view'
    ->btnDelete()                                       // key = 'delete'
    ->btnEnableAndDisable()                             // key = 'enable'/'disable'

    // 带参数
    ->btnEdit('', '编辑', 'btn-primary', 'mdi-lead-pencil', 'data-layer-size="88%,98%"')

    // 自定义链接
    ->btnLink('changeLeader',                           // key（供 mapClass）
        url('/admin/member/member/changeLeader', ['member_id' => '__data.pk__']),
        '推荐人', 'btn-secondary', '', 'title="修改推荐人"')

    // POST 单行 ID
    ->btnPostRowid('clear_errors', url('clearErrors'), '确认重置？', 'btn-info', 'mdi-backup-restore')

    // 换行（按钮多时分两行）
    ->br()

    // 给所有行按钮追加属性
    ->btnEdit()->barAddAttr('data-layer-size="1024px,98%"')

    // 条件隐藏/禁用
    ->mapClass([
        'edit'    => ['hidden' => '__hi_edit__'],
        'delete'  => ['hidden' => '__hi_del__', 'disabled' => '__dis_del__'],
    ]);
```

### 条件隐藏的 __hi_* 字段设置

```php
foreach ($data as &$d) {
    $d['__hi_edit__'] = $d['status'] == 1;
    $d['__hi_del__']  = $d['id'] == 1;
    $d['__dis_del__'] = $d['id'] == 1;
}
unset($d);
```

### Actionbar 按钮 key 名

| 方法 | key |
|---|---|
| `btnEdit()` | `'edit'` |
| `btnView()` | `'view'` |
| `btnDelete()` | `'delete'` |
| `btnEnable()` / `btnEnableAndDisable()` 启用 | `'enable'` |
| `btnDisable()` / `btnEnableAndDisable()` 禁用 | `'disable'` |
| `btnLink($key, ...)` | 第一个参数 `$key` |
| `btnPostRowid($key, ...)` | 第一个参数 `$key` |

***

## 七、全局表格设置

```php
$table->useCheckbox(false);     // 隐藏行首勾选框（只读列表）
$table->sortable([]);           // 禁用所有列排序（树形表格）
$table->sortable('id,sort');    // 只允许指定列排序
```

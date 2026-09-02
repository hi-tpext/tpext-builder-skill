# 表单高级模式

***

## 一、Tab 页签（⭐⭐⭐ 高频）

`$form->tab($label, $active)` 后续字段自动归属该 Tab，下一个 `tab()` 切换。想手动切回全局可调用 `$form->tabEnd()`。

```php
$form->tab('基本信息');
$form->text('name', '名称')->required();
$form->textarea('desc', '描述');

$form->tab('详细配置');
$form->image('cover', '封面图');

// URL 控制激活哪个 Tab
$tab = input('tab', 0);
$form->tab('基本信息',   (bool)($tab == 0));
$form->tab('规格型号',  (bool)($tab == 1));
$form->tab('库存价格',  (bool)($tab == 2));
```

> 保存后跳回指定 Tab：`$this->success('保存成功', url('edit', ['id' => $id, 'tab' => 1]));`

***

## 二、左右分栏（⭐⭐ 中频）

`$form->left($colSize)` / `$form->right($colSize)`，闭包写法内部自动结束。

```php
$form->defaultDisplayerSize(12, 12);  // 上下结构，label 独占一行

$form->left(7)->with(function () use ($form) {
    $form->text('name', '名称')->required();
    $form->select('category_id', '分类')->required()
         ->dataUrl(url('/admin/service/packageategory/selectPage'));
    $form->ueditor('content', '内容');
});

$form->right(5)->with(function () use ($form, $isEdit) {
    $form->image('logo', '封面图')->mediumSize();
    $form->text('sale_price', '售价', 4)->afterSymbol('￥')->required();
    $form->number('sort', '排序', 4)->default(0);
    $form->switchBtn('is_show', '显示', 4)->default(1);
    if ($isEdit) {
        $form->show('create_time', '添加时间', 6);
    }
});
```

> 也可以用 `fields('', '', 7)` 拆左右，参见 form-fields.md。

***

## 三、分步表单（⭐ 低频）

`$form->step($label, $description, $active)` 返回 FieldsContent，后续字段直接用 `$form->` 调用（不能用 `$step1->text()`）。想手动收尾可调 `$form->stepEnd()`。

```php
$form->step('第一步：基础信息', '填写基本资料', true);
$form->text('name', '名称')->required();
$form->select('type', '类型')->required();

$form->step('第二步：详细', '上传封面与说明');
$form->textarea('desc', '描述');
$form->image('cover', '封面');

$form->step('第三步：提交', '二次确认后提交');
$form->checkbox('agree', '我已阅读协议')->required();
```

> 大表单优先用 Tab 分页，仅向导式"下一步"流程才用 Step。

***

## 四、查看/详情模式（$isEdit == 2，⭐⭐⭐ 高频）

HasView 调用前已自动 `$form->readonly()`，并把输入元素自动转换为只读展示，一般不需要手动处理。

```php
if ($isEdit == 2) {
    $form->match('type')->options([1 => 'A', 2 => 'B']);
}

// 编辑/查看时显示时间，新增时隐藏
if ($isEdit) {
    $form->show('create_time', '添加时间');
    $form->show('update_time', '修改时间');
}
```

***

## 五、items 只读列表（日志/流水）

```php
$logList = MemberLog::where(['member_id' => $data['id']])
    ->with(['admin'])->order('id desc')->select();

$form->tab('操作日志(' . count($logList) . ')');

$form->items('log_list', ' ')->dataWithId($logList)->with(
    $form->show('desc', '描述'),
    $form->show('change', '变动'),
    $form->show('admin_id', '管理员')->to('{admin.username}({admin.name})'),
    $form->show('create_time', '操作时间')->getWrapper()->addStyle('width:150px')
)->canNotAddOrDelete()->size(0, 12)->showLabel(false);
```

***

## 六、items 可编辑子表（⭐⭐⭐ 高频）

> `->with()` 内部自动调用 `itemsEnd()`，无需手动调用。

```php
$specList = $isEdit ? SpecModel::where(['main_id' => $data['id']])->select() : [];

// with 自动结束，不需要 itemsEnd
$form->items('spec_list', '规格')->dataWithId($specList)->with(
    $form->text('name', '规格名')->required()->getWrapper()->addStyle('width:200px;'),
    $form->tags('value', '可选值')->required()
)->help('多个值用回车分割');

// 闭包写法（with 自动结束）
$form->items('rights_list', '权益')->with(function () use ($form) {
    $form->show('name', '权益名称');
    $form->show('desc', '权益描述');
    $form->switchBtn('is_have', '是否开启');
    $form->match('input_type', '填写类型')
         ->options(['rate'=>'百分比','discount'=>'折扣','text'=>'文本','number'=>'数字','none'=>'无']);
})->canNotAddOrDelete()->dataWithId($allRights, 'id')->size(12, 12);
```

***

## 七、When 联动（⭐⭐ 中频）

写在**触发字段**上（select/radio/checkbox/transfer），不是写在 `$form` 上。

```php
// 单值
$form->select('type', '类型')->options([1=>'文章', 2=>'链接', 3=>'视频'])
     ->when(1)       ->with($form->textarea('content', '文章内容'))
     ->when(2)       ->with($form->text('url', '链接地址'))
     ->when(3)       ->with($form->text('duration', '时长'));

// 多值数组（任一命中即展开）
$form->select('type', '类型')->options($opts)
     ->when([2, 3])->with($form->text('url', '链接地址'));

// 闭包写法（复杂嵌套）
$form->radio('reward_type', '奖励类型')->options([0=>'积分',1=>'余额'])->default(1)->required()
     ->when(0, function () use ($form) {
         $form->select('integral_type', '积分类型')->options([0=>'固定值',1=>'比例'])->required()
              ->when(0)->with($form->number('integral_value', '积分值'))
              ->when(1)->with($form->number('integral_ratio', '积分比例'));
     })
     ->when(1, function () use ($form) {
         $form->select('balance_type', '余额类型')->options([0=>'固定值',1=>'比例'])->default(0)
              ->when(0)->with($form->number('balance_value', '余额值'))
              ->when(1)->with($form->number('balance_ratio','余额比例'));
     });

// Checkbox 多选：加号拼接 = 必须全部选中
$form->checkbox('notify', '通知方式')
     ->options(['sms'=>'短信','email'=>'邮件','wx'=>'微信'])
     ->when('sms+wx')->with($form->text('phone', '手机号'))
     ->when('email')->with($form->text('email_addr', '邮箱地址'));
```

***

## 八、URL 驱动上下文

```php
// 新增时从 URL 取默认值，编辑时从数据取
$store_id = $isEdit ? $data['store_id'] : input('store_id/d', 1);

// 写入 hidden 字段，提交时回传
$form->hidden('store_id')->value($store_id);

// URL 带 member_id 时锁定搜索上下文
$search->defaultDisplayerColSize(3);
if (input('member_id/d')) {
    $search->hidden('member_id')->value(input('member_id/d'));
} else {
    $search->select('member_id', '会员')
           ->dataUrl(url('/admin/member/member/selectPage'));
}

// URL 驱动默认选中
$search->select('pay_status', '支付状态')
       ->options($payStatusOptions)
       ->default(input('pay_status'));
```

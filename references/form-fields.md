# 表单字段速查

所有字段通过 `$form->{类型}('字段名', '标签', $cloSize)` 创建，第三个参数是列宽（Bootstrap 1~12 栅格，等价于 `->cloSize(4)`）。返回字段对象支持链式调用。

***

## 通用方法（Field 基类）

```php
// 值
->value($val)           // 设置当前值
->default($val)         // 设置默认值（value 为空时使用）
->lockValue($bool)      // 锁定值，阻止 fill() 覆盖

// 状态
->required($bool)       // 必填，✅ 支持条件：->required(!$isEdit)
->readonly($bool)       // 只读，✅ 支持条件：->readonly($count > 0)
->disabled($bool)       // 禁用，✅ 支持条件

// 标签 & 尺寸
->label($text)
->showLabel($bool)      // 隐藏 label，配合 size(0, 12)
->size($labelW, $fieldW)  // label 宽度 + 控件宽度（默认 [2, 8]）
->fullSize($labelMin)     // 占满一行
->cloSize($val)           // 外层列宽（等价构造第 3 参数）

// 提示 / 错误
->help($text)           // 帮助文本（可含 HTML）
->error($text)

// 样式
->class($cls)
->style($css)
->attr($name, $val)

// 值转换
->to($callbackOrFormat) // 闭包 或 '{field}#{rel.field}'

// 其他
->name($val)
->jsOptions($options)
->addScript($script)    // 内联 JS

// 条件样式
->mapClass($values, $class, 'field', $logic)
->mapClassGroup([[$val, $class], ...])

// 包装器
->getWrapper()->addStyle('width:200px;')
->getWrapper()->addClass('my-css-class')
```

***

## text — 文本输入

```php
$form->text('name', '名称', 6)
    ->placeholder('请输入名称')
    ->maxlength(100)
    ->befor('<span class="input-group-addon">前缀 HTML</span>')
    ->after('<span class="input-group-addon">后缀 HTML</span>')
    ->beforSymbol('<i class="mdi mdi-account-key"></i>')   // Material 图标前缀
    ->afterSymbol('.00')
    ->required();
```

## number / password / textarea / hidden

```php
$form->number('sort', '排序', 4)->min(0)->max(9999)->step(1)->default(0)->required();

$form->password('pwd', '密码')
     ->required(!$isEdit);   // 添加必填，编辑留空不修改

$form->textarea('desc', '描述', 12)->rows(5)->maxlength(500)->placeholder('请输入');

$form->hidden('id');
$form->hidden('store_id')->value($store_id);
```

## select — 下拉选择

```php
$form->select('status', '状态', 4)
    // 1) 静态
    ->options([1 => '启用', 0 => '禁用'])
    ->texts(['启用', '禁用'])                       // value & label 相同，不能与 options 方法同时使用
    ->beforOptions(['' => '全部'])                  // 前置兜底选项
    ->afterOptions([999 => '其他'])                 // 后置选项

    // 2) 从数据集，仅数据集数据较少时使用，大量数据时请使用远程下拉
    ->optionsData(SomeModel::order('sort')->select(), 'name', 'id')

    // 3) 远程下拉（配合 selectPage 属性）
    ->dataUrl(url('/admin/somedict/selectPage'), 'name', 'id', 250, true)

    // 分组 OPTIONS
    // ->options([
    //     '' => ['label' => '根分组', 'options' => ['' => '请选择']],
    //     'module' => ['label' => '[某模块]', 'options' => ['/a/b' => '/a/b']],
    // ])

    // 级联
    ->withNext($form->select('city_id', '城市')
                    ->dataUrl(url('api/areacity/city'), 'ext_name')
                    ->withNext($form->select('area_id', '区县')
                                    ->dataUrl(url('api/areacity/area'), 'ext_name')))
    ->withParams(['province_id'])     // AJAX 附带其他字段值
    ->select2(true)
    ->disabledOptions([1, 3])
    ->readonlyOptions([2])
    ->placeholder('请选择')
    ->required()

    // When 联动
    ->when(1)->with($form->text('other', '其他说明')->required())
    ->when([2, 3])->with($form->textarea('reason', '原因'));
```

## multipleSelect / radio / checkbox / transfer

```php
// 多选
$form->multipleSelect('tags', '标签')
     ->optionsData(SomeTagModel::select(), 'name', 'id')
     ->dataUrl(url('/admin/sometagdict/selectPage'))
     ->default([1, 2]);

// 单选
$form->radio('gender', '性别', 6)
     ->options(['m' => '男', 'f' => '女', 0 => '未知'])
     ->inline(true)          // 行内（默认）
     ->blockStyle()          // 块级显示
     ->yesOrNo()             // 快捷 [1=>是,0=>否]
     ->default(0)
     ->readonlyOptions(['m'])
     ->when('m')->with($form->text('m_name', '男士名'));

// 复选框
$form->checkbox('attr', '属性', 12)
     ->blockStyle()
     ->checkallBtn('全选')
     ->options(['is_recommend' => '推荐', 'is_hot' => '热门'])
     ->default([]);

// 穿梭框
$form->transfer('permissions', '权限')
     ->options($options)
     ->default([1, 3]);
```

## switchBtn — 开关

```php
$form->switchBtn('enable', '启用', 4)
     ->pair(1, 0)           // [on, off] 值，默认 [1,0]
     ->default(1)
     ->disabled($isEdit && $data['id'] == 1)
     ->help('禁用后无法登录');
```

## match — 只读匹配展示

```php
$form->match('level', '等级', 4)
     ->optionsData($levels, 'name', 'level')//仅数据集数据较少时使用，大量数据使用模型关联并改用show('xxxx.yy')显示
     ->afterOptions([0 => '普通用户']);
```

## image / file

```php
// 单图
$form->image('avatar', '头像', 4)
     ->thumbSize(200, 200)
     ->smallSize() / ->mediumSize() / ->bigSize() / ->largeSize()   // 50/120/240/480
     ->imageResize(200, 200)           // 上传后自动裁剪
     ->default('/assets/images/no-avatar.jpg');

// 多图（支持视频）
$form->multipleImage('images', '多图/视频', 12)
     ->video()                          // 允许视频
     ->mediumSize()
     ->limit(9)
     ->default([]);

// 单文件
$form->file('file', '附件')
     ->extTypes('doc,docx,xls,xlsx,pdf')
     ->office() / ->video() / ->audio() / ->pkg()   // 快捷类型
     ->limit(1)
     ->jsOptions(['fileSingleSizeLimit' => 50 * 1024 * 1024]);

// 多文件
$form->multipleFile('files', '附件', 12)
     ->limit(10)->office()->default([]);
```

## 日期 / 时间 / 范围

```php
$form->datetime('created_at', '创建时间', 6)->format('YYYY-MM-DD HH:mm:ss')->timespan('Y-m-d H:i:s');
$form->date('publish_date', '发布日期', 4)->format('YYYY-MM-DD')->timespan('Y-m-d');
$form->time('start_time', '开始时间', 4)->format('HH:mm:ss');
$form->month('month', '月份', 4);
$form->year('year', '年份', 4);

// 范围字段：separator 设置分隔符
$form->datetimeRange('date_range', '时间范围', 12)->separator('—');
$form->dateRange('date_range', '日期范围', 12)->separator('~');
$form->timeRange('time_range', '时间范围', 12)->separator('-');
```

## tags / color / icon / rangeSlider / rate

```php
$form->tags('keyword', '关键字', 12)->placeholder('输入后回车确认');
$form->color('theme_color', '主题色')->hex();
$form->icon('icon_class', '图标')->required()->default('mdi mdi-access-point');
$form->rangeSlider('price_range', '价格')->min(0)->max(1000)->step(10)->default([100, 500]);
$form->rate('score', '评分')->max(5)->default(3);
```

## map — 地图选点

```php
$form->map('location', '位置')->amap()->zoom(15)->height('400px')->width('100%');
```

## show / raw / html / button / divider

```php
$form->show('create_time', '添加时间', 4)->default('-');     // 只读展示
$form->show('address_text', '收货地址', 8)->value($address); // 动态 value
$form->raw('relation_text', '相关图', 12)->value($html);     // 自定义 HTML
$form->html('', '', 12);                                      // 空行分隔
$form->divider('基本信息');                                     // 带标题分割线
$form->button('custom_btn', '点击执行');

// raw + layerOpen 弹窗链接
$form->raw('return', '售后')
     ->to('<a href="#" onclick="layerOpen(this);return false;" '
          . 'data-layer-size="100%,100%" title="{goods_sn}退货" '
          . 'data-url="/admin/shop/orderreturn/add?id={id}">[退款]</a>');
```

## 富文本编辑器

```php
$form->ueditor('content', '详情')->required()   // UEditor（推荐）
     ->imageResize(500, 500)
     ->imageText('水印', 100, 100, 12, '#1ab394', 'top', 'right')
     ->imageWater('./icon.png', 'bottom-left');

$form->editor('content', '内容');     // 通用
$form->wangEditor('content', '内容'); // 轻量
$form->tinymce('content', '内容');    // TinyMCE
$form->ckeditor('content', '内容');   // CKEditor
$form->aceEditor('code', '代码');     // 代码
$form->mdEditor('markdown', 'MD');    // Markdown
$form->mdReader('readme', '预览');    // MD 只读
```

## selectTree — 树形选择

```php
$form->selectTree('tree_fk_id', '树形外键')
     ->treeModel(\app\common\model\SomeTreeModel::class);
```

***

## items — 可重复子表单（一对多）

> 📊 **频率：⭐⭐⭐ 高频**。`->with()` 内部会自动调用 `itemsEnd()`，无需手动调用。

```php
// 可变参数写法（with 自动结束，不需要 itemsEnd）
$form->items('attr_list', '产品属性')->with(
    $form->text('key', '属性名')->required()->getWrapper()->addStyle('width:200px;'),
    $form->text('sort', '排序')->default(1)->required()->getWrapper()->addStyle('width:80px;'),
    $form->text('value', '属性值')->required()->getWrapper()->addStyle('min-width:70%;')
)->canAdd(true)->canDelete(true)->size(12, 12)->help('提示文字');

// dataWithId 预填已有数据（with 自动结束）
$form->items('spec_list', '规格', 12)->dataWithId($specList)->with(
    $form->text('name', '规格名')->required(),
    $form->tags('value', '可选值')->required(),
);

// 闭包写法（with 自动结束）
$form->items('rights_list', '权益')->with(function () use ($form) {
    $form->show('name', '权益名称');
    $form->switchBtn('is_have', '是否开启');
})->canNotAddOrDelete()->dataWithId($allRights, 'id')->size(12, 12);

// 只读列表（日志/流水，with 自动结束）
$form->items('log_list', ' ')->dataWithId($logList)->with(
    $form->show('desc', '描述'),
    $form->show('change', '变动'),
    $form->show('admin_id', '管理员')->to('{admin.username}({admin.name})'),
    $form->show('create_time', '操作时间')->getWrapper()->addStyle('width:150px')
)->canNotAddOrDelete()->size(0, 12)->showLabel(false);

// ⚠️ 不用 with() 的开放写法：必须手动调 itemsEnd()
$form->items('price_list', '库存价格', 12)->dataWithId($priceList);
foreach ($specList as $spec) {
    $form->select('spec_id' . $spec['id'], $spec['name'])
         ->optionsData($valueList, 'value')->required();
}
$form->text('sale_price', '销售价格')->required();
$form->number('stock', '库存')->required();
$form->itemsEnd();   // ✅ 开放写法必须手动结束
```

### items 方法

| 方法 | 说明 |
|---|---|
| `dataWithId($data, $idField)` | 预填已有数据，`$idField` 默认 'id' |
| `canAdd(bool)` / `canDelete(bool)` | 控制新增/删除按钮 |
| `canNotAddOrDelete()` | 等价 `canAdd(false)->canDelete(false)` |
| `canRecover($bool)` | 删除后是否可撤销（默认 true） |
| `readonly(bool)` | 整表只读 |

### items 保存时 JSON 编码

```php
protected function save($id = 0)
{
    $data = request()->only(['name', 'attr_list'], 'post');
    foreach (['attr_list'] as $k) {
        if (isset($data[$k]) && is_array($data[$k])) {
            $data[$k] = json_encode($data[$k], JSON_UNESCAPED_UNICODE);
        }
    }
    return $this->doSave($data, $id);
}
```

***

## fields — 内联字段组

> `->with()` 内部会自动调用 `fieldsEnd()`，无需手动调用。

```php
// 可变参数写法（with 自动结束，不需要 fieldsEnd）
$form->fields('group', '字段组', 12)->with(
    $form->text('sub1', '子字段1', 6),
    $form->select('sub2', '子字段2', 6)->options([...])
);

// 闭包写法（with 自动结束）
$form->fields('', '', 7)->size(0, 12)->showLabel(false)->with(
    function () use ($form) {
        $form->text('name', '名称')->required();
    }
);

// 省市区一行（with 自动结束）
$form->fields('省/市/区')->with(
    $form->select('province', ' ', 4)->size(0, 12)->showLabel(false)
         ->dataUrl(url('/api/areacity/province'), 'ext_name')
         ->withNext(
             $form->select('city', ' ', 4)->size(0, 12)->showLabel(false)
                  ->dataUrl(url('/api/areacity/city'), 'ext_name')
                  ->withNext(
                      $form->select('area', ' ', 4)->size(0, 12)->showLabel(false)
                           ->dataUrl(url('/api/areacity/area'), 'ext_name')
                  )
         )
);

// ⚠️ 不用 with() 的开放写法：必须手动调 fieldsEnd()
$form->fields('', '', 7)->size(0, 12)->showLabel(false);
$form->text('name', '名称')->required();
$form->select('type_fk_id', '类型')->required();
$form->fieldsEnd();   // ✅ 开放写法必须手动结束
```

***

## 选择类字段 Options 方法速查

`select / multipleSelect / radio / checkbox / transfer` 共有：

| 方法 | 说明 |
|---|---|
| `options($array)` | `[value => label]`；支持二维分组 |
| `texts($arr)` | value & label 相同 |
| `optionsData($list, $textField, $idField)` | 从数据集构建，仅数据集数据较少时使用，大量数据时请使用远程下拉（select） |
| `beforOptions / afterOptions / mergeOptions / forget` | 前置/后置/合并/移除 |
| `disabledOptions($val)` | 禁用值 |
| `readonlyOptions($val)` | 只读值 |

## When 联动 & withNext 级联

> 📊 When **⭐⭐ 中频**，withNext **⭐⭐⭐ 高频**。

```php
// When：在触发字段上调用（select / radio / checkbox / transfer）
$form->select('type', '类型')->options([1=>'文章', 2=>'链接', 3=>'视频'])
     ->when(1)       ->with($form->textarea('content', '文章内容'))
     ->when([2, 3])  ->with($form->text('url', '链接地址'));

// Checkbox 多选：加号拼接 = 必须全部选中
$form->checkbox('notify', '通知方式')
     ->options(['sms'=>'短信','email'=>'邮件','wx'=>'微信'])
     ->when('sms+wx')->with($form->text('phone', '手机号'));

// withNext：省→市→区链式 AJAX 级联
$form->select('province', '省')->dataUrl(url('api/areacity/province'), 'ext_name')
     ->withNext($form->select('city', '市')->dataUrl(url('api/areacity/city'), 'ext_name')
                   ->withNext($form->select('area', '区')->dataUrl(url('api/areacity/area'), 'ext_name')));
```

> ⚠️ 省市区 API 是 `/api/areacity/{level}`（areacity 不是 region）。显示字段用 `ext_name`（全名）或 `name`（短名）。

***

## 通用技巧

| 技巧 | 写法 |
|---|---|
| 字段占整行 | `->size(0, 12)->showLabel(false)` |
| 列宽简写 | 构造第 3 参数：`text($name, $label, 4)` |
| 条件只读 | `->readonly($count > 0)` |
| 条件必填 | `->required(!$isEdit)` |
| 条件禁用 | `->disabled($id == 1)` |
| 动态设值 | `->value($computed)` |
| 默认值空时显示 | `->default('-')` |

***

## URL 驱动的默认值 & Tab 激活（⭐⭐⭐ 高频）

从 URL 参数读取上下文是常见的状态管理方式。**新增**时用 `input()` 取默认值并写入 `hidden`；**编辑**时用已存数据。

```php
protected function buildForm($isEdit, &$data = [])
{
    $form = $this->form;

    // 1) URL 取当前要激活的 tab
    $tab = input('tab', 0);

    // 2) URL 取上下文默认值
    $store_id  = $isEdit ? $data['store_id']  : input('store_id/d', '1');
    $goodsType = $isEdit ? $data['goods_type'] : input('type', 0);

    // 3) hidden 写入上下文
    $form->hidden('store_id')->value($store_id);
    $form->hidden('goods_type')->default($goodsType);

    // 4) 用 $goodsType 驱动字段渲染分支
    $salePriceName = [0 => '售价', 1 => '售价', 2 => '秒杀价', 3 => '团购价', 4 => '起始价'];
    $form->text('sale_price', $salePriceName[$goodsType] ?? '售价', 4)->required();

    // 5) tab 第二参数 $active
    $form->tab('规格型号/属性', $tab == 3)->with(
        $form->items('spec_list', '规格型号')->dataWithId($specList)->with(
            $form->text('name', '名称')->required(),
            $form->tags('value', '可选值')->required()
        )
    );
    $form->tab('活动时间设置', $tab == 4);
}
```

### 典型场景对照

| 场景 | URL 参数 | 表单写法 |
|---|---|---|
| 多商家切换 | `store_id` | `hidden('store_id')->value(input('store_id/d', 1))` |
| 商品类型分支 | `type` | `hidden('goods_type')->default(input('type', 0))` |
| 定位选项卡 | `tab` | `tab('标题', $tab == N)` |
| 树形作用域 | `store_id` | `dataModel->reInit(['treeScope' => [['store_id','=',$sid]]])` |

***

## datetimeRange 嵌入 items（活动时间段设置）

```php
$form->items('promote_time_list', '时间设置')->dataWithId($timeList)->with(
    $form->datetimeRange('time_range', '起止时间')->separator('—')->required()
         ->getWrapper()->addStyle('width:74%;'),
    $form->number('num', '数量')->default(1)->required()->getWrapper()->addStyle('width:30%;')
)->help('可设置多个时间段，时间段之间不能冲突');
```

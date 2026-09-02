# Builder API 参考

***

## Action Trait 对照

| Trait | 提供的 Action |
|---|---|
| `HasBase` | `$dataModel`、`builder()`、`checkToken()`、`doSave()`、`getPk()`、所有空 Hook |
| `HasIndex` | `index()` + `selectPage()` |
| `HasAdd` | `add()` — GET 显示表单，POST 调用 `save(0)` |
| `HasEdit` | `edit($id)` — GET 取数，POST 调用 `save($id)` |
| `HasView` | `view($id)` — 只读详情，自动 `$form->readonly()` |
| `HasDelete` | `delete()` — 单条/批量删除 |
| `HasEnable` | `enable($id)` — 启用/禁用 |
| `HasAutopost` | `autopost()` — 行内 AJAX 字段编辑 |
| `HasExport` | `export()` — CSV/XLS/XLSX 导出 |
| `HasLeftTree` | 左树 + 右列表布局 |
| `HasIAED` | = HasBase + HasIndex + HasAdd + HasEdit + HasDelete |
| `HasBuilder` | = HasBase + HasIndex + HasAdd + HasEdit + HasView + HasAutopost + HasEnable + HasDelete |

***

## Layer 响应

```php
// 关弹窗 + 刷新表格（最常用）
$this->builder()->layer()->closeRefresh(1, '保存成功');

// 关弹窗 + 跳转
$this->builder()->layer()->closeGo(1, url('index'), '完成');

// 仅关弹窗
$this->builder()->layer()->close(0, '用户不存在！');
$this->builder()->layer()->close(1, '操作成功');

// 成功/失败 JSON
$this->success('修改成功');
$this->error('错误信息');

// 带跳转 + 附加脚本（保存后跳回指定 Tab）
$script = '<script>parent.$(".search-refresh").trigger("click");</script>';
$this->success('保存成功', url('edit', ['id' => $id, 'tab' => 4]),
    ['script' => $script], 1);
```

***

## JS / CSS 注入

```php
$builder = $this->builder();

// 内联（会被合并 minify）
$builder->addJs('alert(1)');
$builder->addCss('body{margin:0}');

// 内联（不会被 minify，可含 <script>/<style> 标签）
$builder->addScript('<script>parent.$(".search-refresh").trigger("click");</script>');
$builder->addStyleSheet('.row-name{white-space:normal;}');

// 外部文件
$builder->addStyle('/static/your.css');
$builder->addScript('/static/your.js');
```

> 日常补样式优先 `addStyleSheet`，弹窗脚本优先 `addScript`。

### creating() — 全局注入

每次 `$this->builder()` 调用时触发，所有 action 共享。

```php
/**
 * @param \tpext\builder\code\Builder $builder
 * @param string $type index/add/edit/view 或空
 */
protected function creating($builder, $type = '')
{
    if ($type === 'index') {
        $builder->addStyleSheet('.some-list-class{color:#f00;}');
    }
}
```

***

## 全局尺寸

```php
$form->defaultDisplayerColSize(6);     // 每个字段外层列宽 6（一行 2 个）
$form->defaultDisplayerSize(12, 12);   // label 12 / field 12 = 各占一行
$table->defaultDisplayerColSize(4);
$search->defaultDisplayerColSize(3);   // 搜索常用，4 个一行
```

单个字段：

- `->size($labelW, $fieldW)` — label 宽度 + 控件宽度
- `->cloSize($N)` — 外层列宽
- `->fullSize($labelMin)` — 占满一行
- `->showLabel(false) + size(0, 12)` — 无 label，field 独占 12 列

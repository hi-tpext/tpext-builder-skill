# tpext-builder-skill

供 `tpext-builder` / `tpext-tinyvue` / `tpext-vexipui` 使用的 AI skill。

## 介绍

基于 **tpext-builder**（ThinkORM 后台 UI 生成器，UI 主题 LightYearAdmin / Bootstrap 3）的 AI 辅助开发 skill。安装后自动复制到 Trae 的 `.trae/skills/` 目录，AI 助手开发 `app/admin/controller/` 下控制器时自动加载，提供准确的 trait 组合、字段/列 API、Hook 签名等参考。

## 安装

```bash
composer require ichynul/tpext-builder-skill
```

安装后运行一次以下命令触发 skill 复制（首次 require 后 autoload 需重新生成）：

```bash
composer dump-autoload
composer install
```

或者直接用 Trae 打开项目（自动生成 `.trae/` 目录），然后运行 `composer install`。

## 自动复制机制

双机制并存，跨框架通用：

### 1. Composer Plugin（所有 Composer 项目通用）

本包 `type` 为 `composer-plugin`，通过 `SkillPlugin` 监听 `post-autoload-dump` 事件：

1. `composer install` / `composer update` / `composer dump-autoload` 后触发
2. autoload 已生成，所有类可加载
3. 调用 `Install::install()` 复制 skill 到检测到的编辑器目录

Composer 2.0+ 需在根 `composer.json` 允许 plugin：

```json
{
    "config": {
        "allow-plugins": {
            "ichynul/tpext-builder-skill": true
        }
    }
}
```

首次 `composer require` 时 Composer 会提示是否允许，选择 yes 即可。

### 2. webman Install 钩子（webman 项目额外生效）

webman 项目的 `composer.json` 已配置 `post-package-install` → `support\Plugin::install`：

1. 包安装/更新时，webman 查找 PSR-4 命名空间下的 `Install` 类
2. 调用 `tpext\builder\skill\Install::install()`

两者并存时可能重复执行，覆盖复制无害（仅 7 个文件）。

## 编辑器支持

| 编辑器      | 目录                 | 状态            |
| -------- | ------------------ | ------------- |
| Trae     | `.trae/skills/`    | ✅ 自动复制        |
| Cursor   | `.cursor/rules/`   | 预留（.mdc 格式不同） |
| Windsurf | `.windsurf/rules/` | 预留（.mdc 格式不同） |

其他编辑器可在 `src/Install.php` 的 `detectEditors()` 中扩展。

## 手动复制

如自动机制未触发，可手动复制：

```bash
# Windows PowerShell
mkdir -p .trae/skills/tpext-builder-skill
cp vendor/ichynul/tpext-builder-skill/SKILL.md .trae/skills/tpext-builder-skill/
cp -r vendor/ichynul/tpext-builder-skill/references .trae/skills/tpext-builder-skill/
```

## 目录结构

```
tpext-builder-skill/
├── SKILL.md              # Skill 入口（红线/速查/骨架）
├── references/           # 配套详细文档
│   ├── AGENTS-DEMO.md    # 框架总览 + trait 组合 + 属性名速记
│   ├── builder.md        # Action Trait / Layer / JS-CSS / Hook / 尺寸
│   ├── controller-tpl.md # 控制器模板 + initialize 默认值表 + Hook 覆写
│   ├── form-fields.md    # 全字段类型 + items/fields 嵌套 + When/withNext
│   ├── form-patterns.md  # Tab/Step/分栏/联动/查看模式
│   └── table-columns.md  # 列类型 + mapClass + Actionbar/Toolbar/Search
├── src/
│   ├── Install.php       # webman 安装钩子（自动复制 skill）
│   ├── helper.php        # ExtLoader classmap 注册
│   └── common/
│       └── Resource.php  # tpext 资源注册
├── composer.json
└── README.md
```

## 卸载

```bash
composer remove ichynul/tpext-builder-skill
```

卸载时会自动调用 `Install::uninstall()`，删除 `.trae/skills/tpext-builder-skill/` 目录。

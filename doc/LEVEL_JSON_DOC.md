# `level.json` 配置说明

本文档说明自定义关卡 `level.json` 的主要字段、嵌套结构、运行时约束和一个最小示例。

## 1. 顶层结构

```json
{
  "levelMode": "uninstall",
  "email": {},
  "startMenu": {},
  "expertTip": {},
  "software": {},
  "uninstallUI": {},
  "installUI": {}
}
```

## 2. 顶层字段

| 字段 | 类型 | 必填 | 说明 |
|---|---|---:|---|
| `levelMode` | `string` | 是 | 关卡模式，`uninstall` 或 `install` |
| `email` | `object` | 是 | 任务邮件配置 |
| `startMenu` | `object` | 是 | 客户机开始菜单可用工具 |
| `expertTip` | `object` | 是 | 专家提示内容与价格 |
| `software` | `object` | 是 | 软件本体、安装目录、可执行文件、卸载器等 |
| `uninstallUI` | `object` | 否 | 卸载模式窗口基础配置 |
| `installUI` | `object` | 否 | 安装模式窗口基础配置 |

说明：
- `levelMode = "uninstall"` 时，主要使用 `uninstallUI`
- `levelMode = "install"` 时，主要使用 `installUI`
- 如果 `software.customUninstallScene` / `software.customInstallScene` 存在，则优先加载自定义场景

## 3. `email`

```json
"email": {
  "senderName": "狗蛋",
  "senderUserId": "USR-LEVEL-2",
  "preview": "帮我卸载快速阅读器",
  "content": "正文……",
  "target": "卸载“快速阅读器”软件",
  "reward": 10
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `senderName` | `string` | 发件人显示名 |
| `senderUserId` | `string` | 发件人 ID |
| `preview` | `string` | 任务列表预览文本 |
| `content` | `string` | 邮件正文 |
| `target` | `string` | 任务目标描述 |
| `reward` | `int` | 任务奖励，运行时会限制到 `0..100` |

## 4. `startMenu` (请禁用，暂未提供任务管理器和注册表的支持)

```json
"startMenu": {
  "taskManager": false,
  "regedit": false
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `taskManager` | `bool` | 是否允许任务管理器 |
| `regedit` | `bool` | 是否允许注册表编辑器 |

## 5. `expertTip`

```json
"expertTip": {
  "content": "进度条不动了？帮它动起来。",
  "cost": 6
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `content` | `string` | 专家提示文本 |
| `cost` | `int` | 提示价格 |

### `cost` 规则

- 允许范围：`0..1000`
- 小于 `0` 或大于 `1000`：运行时自动重置为 `0`
- 编辑器内也会限制输入范围为 `0..1000`

## 6. `software`

```json
"software": {
  "displayName": "快速阅读器",
  "shortcutName": "快速阅读器.lnk",
  "executableName": "FastReader.exe",
  "uninstallerName": "uninstall.exe",
  "installDirectory": "C:\\Program Files\\FastReader",
  "taskProblemId": "uninstall_fastreader",
  "customUninstallScene": "res://resources/FastReaderUninstall.tscn",
  "customInstallScene": null,
  "iconBase64": null,
  "files": [],
  "customFileStructure": {}
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `displayName` | `string` | 软件显示名 |
| `shortcutName` | `string` | 桌面快捷方式名 |
| `executableName` | `string` | 主程序名 |
| `uninstallerName` | `string` | 卸载程序名 |
| `installDirectory` | `string` | 安装目录 |
| `taskProblemId` | `string` | 对应问题 ID |
| `iconBase64` | `string?` | 软件图标 base64，可选 |
| `customUninstallScene` | `string?` | 自定义卸载场景路径 |
| `customInstallScene` | `string?` | 自定义安装场景路径 |
| `files` | `array` | 传统递归文件树定义 |
| `customFileStructure` | `object` | 高级平铺文件结构定义，推荐 |

### 6.1 `files`

```json
"files": [
  {
    "name": "custom.exe",
    "type": "file",
    "size": 1024,
    "hidden": false,
    "readonly": false,
    "protected": false,
    "children": []
  }
]
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `name` | `string` | 文件或文件夹名 |
| `type` | `string` | `file` 或 `folder` |
| `hidden` | `bool` | 是否隐藏 |
| `readonly` | `bool` | 是否只读 |
| `protected` | `bool` | 是否受保护；为 `true` 时不能被删除 |
| `size` | `long` | 文件大小；文件夹通常填 `0` |

### 6.2 `customFileStructure`

推荐用于高级关卡。键是相对安装目录的路径，值是文件/目录定义。

```json
"customFileStructure": {
  "FastReader.exe": {
    "type": "executable",
    "size": 26048000,
    "protected": true
  },
  "uninstall.exe": {
    "type": "executable",
    "size": 512000,
    "actualPath": "custom_uninstall:uninstall_fast_reader_level",
    "protected": true
  },
  "temp/good": {
    "type": "folder"
  }
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `type` | `string` | `file` / `folder` / `executable` |
| `size` | `long` | 文件大小 |
| `hidden` | `bool` | 是否隐藏 |
| `readonly` | `bool` | 是否只读 |
| `protected` | `bool` | 是否受保护；为 `true` 时不能被删除 |
| `actualPath` | `string?` | 双击时实际打开的路径或逻辑路由 |

`protected` 说明：
- 仅控制“是否允许删除”，不会自动隐藏，也不会自动变成只读。
- 对 `files` 和 `customFileStructure` 两种写法都生效。
- 可用于文件和文件夹；如果文件夹标记为 `protected`，整个路径会被视为不可删除。

### 6.3 `actualPath` 常见用法

| 值前缀 | 含义 |
|---|---|
| `custom_uninstall:{pcId}` | 打开自定义卸载逻辑 |
| `custom_install:{pcId}` | 打开自定义安装逻辑 |

## 7. `uninstallUI` / `installUI`

最小写法：

```json
"uninstallUI": {
  "windowTitle": "快速阅读器 卸载程序",
  "windowSize": [720, 520],
  "resizable":false
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `windowTitle` | `string` | 窗口标题 |
| `windowSize` | `int[2]` | `[宽, 高]` |
|`resizable` | `bool` | 是否禁用窗口拖动大小,建议禁用|
|`enableDoubleClickMaximize`|bool|是否禁用双击标题栏全屏展示窗口,建议禁用|

补充说明：
- `windowSize` 应与自定义软件面板场景中根 `Control` 节点的 `Custom Minimum Size` 保持一致
- 如果两者不一致，运行时容易出现窗口内容裁切、留白或布局错位
- 对自定义面板来说，不要依赖 `resizable`、`enableDoubleClickMaximize`、最大化按钮、最小化按钮这类宿主窗口属性
- 自定义面板的实际交互和外观应以场景本身为准

## 8. 最小卸载关卡示例

```json
{
  "levelMode": "uninstall",
  "email": {
    "senderName": "Customer",
    "senderUserId": "USR-001",
    "preview": "请帮我卸载软件",
    "content": "这软件出问题了，帮我卸载一下。",
    "target": "卸载目标软件",
    "reward": 10
  },
  "startMenu": {
    "taskManager": false,
    "regedit": false
  },
  "expertTip": {
    "content": "先看看卸载程序卡在哪里。",
    "cost": 5
  },
  "software": {
    "displayName": "Custom Software",
    "shortcutName": "Custom Software.lnk",
    "executableName": "Custom.exe",
    "uninstallerName": "uninstall.exe",
    "installDirectory": "C:\\Program Files\\CustomSoftware",
    "taskProblemId": "uninstall_customsoftware",
    "customUninstallScene": "res://resources/CustomUninstall.tscn",
    "customFileStructure": {
      "Custom.exe": { "type": "executable", "size": 1024000, "protected": true },
      "uninstall.exe": { "type": "executable", "size": 256000, "actualPath": "custom_uninstall:my_level_id", "protected": true }
    }
  },
  "uninstallUI": {
    "windowTitle": "Custom Software 卸载程序",
    "windowSize": [720, 520],
    "resizable":false,
    "enableDoubleClickMaximize":false
  }
}
```

## 9. 当前运行时限制

- `email.reward` 会在运行时限制到 `0..100`
- `expertTip.cost` 允许 `0..1000`，越界会重置为 `0`
- `software.executableName` 不能为空

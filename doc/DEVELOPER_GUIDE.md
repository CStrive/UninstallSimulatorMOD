# 自定义卸载窗口开发指南

## 概述

本指南详细说明如何创建 PCK 格式的自定义关卡，包括关键的信号机制实现。

如果有问题，请加群 QQ 576218953 反馈。

## 小提示

在 Steam 中选择 调试模式 打开，方便 PCK 类型关卡调试哦！

## 核心要求

### 1. 使用内嵌 GDScript

PCK 关卡**必须使用内嵌 GDScript**，不能使用外部脚本文件。

**原因**：
- PCK 解压路径 (`user://custom_levels/`) 与 `res://` 路径不匹配
- 外部脚本文件的路径解析会失败
- 内嵌脚本完全避免了路径问题

**实现方式**：

```tscn
[sub_resource type="GDScript" id="GDScript_main"]
script/source = "extends Control

signal level_completed(success: bool)

func _ready() -> void:
    # 初始化逻辑
"

[node name="SuperBoosterUninstall" type="Control"]
script = SubResource("GDScript_main")
mouse_filter = 2
```



### 2. ⚠️ 关键：实现信号机制

**这是最重要的部分！** 如果不实现信号机制，关卡无法完成。

#### 2.1 定义信号

在脚本开头定义 `level_completed` 信号：

```gdscript
extends Control

signal level_completed(success: bool)  # 必须定义

var _step1: Control
var _step2: Control
# ... 其他变量
```

#### 2.2 发射信号

在关卡完成时**必须发射信号**：

```gdscript
func _on_finish() -> void:
	# 检查过关条件
	var success: bool = check_win_conditions()

	if success:
		print('[YourLevel] ✓ PASSED')
	else:
		print('[YourLevel] ✗ FAILED')

	#核心：使用此发射信号通知游戏系统任务成功
	level_completed.emit(success)

	# 然后关闭窗口
	#
```

#### 2.3 工作原理

```
GDScript                    C# 游戏引擎
   |                             |
   | level_completed.emit(true)  |
   |--------------------------->|
   |                             |
   |                        监听信号
   |                             |
   |                   调用 CompleteUninstallAsync()
   |                             |
   |                        标记任务完成
   |                             |
   |                        返回桌面
```

**如果不发射信号**：
- 游戏引擎不知道关卡已完成
- 任务不会被标记为完成
- 不会返回桌面
- 玩家卡在关卡中


## 卸载模板-模板GDScript

```gdscript
#此模板大部分都不用改(除非你更改了模板场景的结构)。自己扩充需要的逻辑即可
extends Control
#窗口背景
@onready var window_bg = $WindowBG
#窗口内容容器
@onready var content_container = $WindowBG/WindowBGHBOX/ContentContainer
#窗口标题栏
@onready var title_bar = $WindowBG/WindowBGHBOX/WindowTitleBar
#窗口标题栏的关闭按钮
@onready var close_btn = $WindowBG/WindowBGHBOX/WindowTitleBar/HBoxContainer/HBoxContainer/CloseBtn

#任务完成信号，必须定义
signal level_completed(success: bool)
#主窗口设置，必须定义
var host_window: Node

func _ready():
	#连接按钮事件
	close_btn.pressed.connect(_on_cancel_pressed)
	#主窗口适配设置
	host_window = get_node_or_null("HostWindow")
	if host_window == null:
		return
	#必须
	host_window.set_chromeless(true)
	window_bg.gui_input.connect(_on_window_surface_gui_input)
	content_container.gui_input.connect(_on_window_surface_gui_input)
	#设置窗口通过标题栏拖动
	title_bar.gui_input.connect(_on_title_bar_gui_input)

#窗口点击置于顶层并获得焦点
func _on_window_surface_gui_input(event: InputEvent):
	if event is InputEventMouseButton and event.button_index == MOUSE_BUTTON_LEFT and event.pressed:
		_activate_host_window()
#窗口拖动
func _on_title_bar_gui_input(event: InputEvent):
	if event is InputEventMouseButton and event.button_index == MOUSE_BUTTON_LEFT and event.pressed:
		_activate_host_window()
	if host_window == null:
		return
	#拖动窗口
	host_window.handle_title_bar_input(event, title_bar.size.y)

func _on_cancel_pressed():
	#如果卸载成功
	if 如果任务成功:
		#重要：触发任务成功
		level_completed.emit(true)
	#关闭窗口
	$HostWindow.close_window()

func _activate_host_window():
	if host_window != null and host_window.has_method("activate_window"):
		host_window.activate_window()
```

## 常见错误

### ❌ 错误 1：忘记定义信号

```gdscript
extends Control
# 缺少：signal level_completed(success: bool)

func _on_finish() -> void:
	level_completed.emit(true)  # 错误：信号未定义
```

**后果**：运行时错误，关卡无法完成

### ❌ 错误 2：忘记发射信号

```gdscript
func _on_finish() -> void:
	var success: bool = check_conditions()
	print('PASSED' if success else 'FAILED')
	$HostWindow.close_window()  # 只关闭窗口，没有发射信号
```

**后果**：窗口关闭，但任务未完成，无法返回桌面

### ❌ 错误 3：在关闭窗口后发射信号

```gdscript
func _on_finish() -> void:
	$HostWindow.close_window()
	level_completed.emit(true)  # 错误：窗口已关闭，节点可能已销毁
```

**后果**：信号可能无法发射

### ❌ 错误 4：使用外部脚本文件

```tscn
[ext_resource type="Script" path="res://my_script.gd" id="1"]

[node name="Root" type="Control"]
script = ExtResource("1")  # 错误：PCK 中无法解析 res:// 路径
```

**后果**：脚本无法加载，关卡无法运行

### ❌ 错误 5：根节点未设置 mouse_filter

```tscn
[node name="Root" type="Control"]
# 缺少：mouse_filter = 2
```

**后果**：按钮无法点击

## 调试技巧

### 1. 检查信号是否定义

在 Godot 编辑器中打开 `.tscn` 文件，选择根节点，查看"节点"面板的"信号"标签，应该能看到 `level_completed(success: bool)` 信号。

### 2. 检查信号是否发射

在对应的发射成功信号方法中添加调试输出：

```gdscript
func _on_finish() -> void:
	var success: bool = check_conditions()
	print('[DEBUG] Emitting level_completed signal: success=', success)
	level_completed.emit(success)
	print('[DEBUG] Signal emitted')
	_close_window()
```

### 3. 检查游戏引擎是否收到信号

查看游戏日志，应该能看到：

```
[VirtualDesktop] Connected to level_completed signal
[DEBUG] Emitting level_completed signal: success=true
[DEBUG] Signal emitted
[VirtualDesktop] level_completed signal received: success=true
```

如果看不到 `[VirtualDesktop] level_completed signal received`，说明信号未被监听或未发射。

## 打包清单

确保你的 PCK 包含以下文件：

```
your_level.uslevel (ZIP 格式)
├── manifest.json          # 关卡元数据
├── level.json             # 关卡定义
└── resources/
	└── your_scene.tscn    # 场景文件（内嵌 GDScript）
```

**不要包含**：
- `.gd` 外部脚本文件
- `.cs` C# 脚本文件
- `.import` 文件

## 自定义安装目录结构（高级功能）

对于高级关卡，你可以使用 `customFileStructure` 字段自定义软件的安装目录结构，而不是使用传统的 `files` 数组。

### 使用场景

- 需要复杂的多层目录结构
- 需要指定特定文件的 `actualPath`（用于卸载程序等）
- 需要更灵活的文件类型控制

### 配置示例

Level配置参考level_json_doc.md

在 `level.json` 的 `software` 部分添加 `customFileStructure`：

```json
{
  "software": {
	"displayName": "Super Booster Pro",
	"installDirectory": "C:\\Program Files\\SuperBooster",
	"uninstallerName": "uninstall.exe",
	"customFileStructure": {
	  "bin/app.exe": {
		"type": "executable",
		"size": 2048000
	  },
	  "bin/helper.dll": {
		"type": "file",
		"size": 512000
	  },
	  "data/config.ini": {
		"type": "file",
		"size": 1024
	  },
	  "data/cache/": {
		"type": "folder"
	  },
	  "uninstall.exe": {
		"type": "executable",
		"size": 256000,
		"actualPath": "custom_uninstall:custom_your_level_id"
	  }
	}
  }
}
```

### 字段说明

- **键（相对路径）**：相对于 `installDirectory` 的路径，使用 `/` 分隔符
  - 文件夹路径可以以 `/` 结尾（可选）
  - 支持多层嵌套，如 `bin/plugins/plugin.dll`

- **type**：文件类型
  - `"file"` - 普通文件
  - `"executable"` - 可执行文件（.exe）
  - `"folder"` - 文件夹

- **size**：文件大小（字节），文件夹可省略

- **hidden**：是否隐藏（可选，默认 false）

- **actualPath**：实际可执行路径（可选）
  - 用于卸载程序：`"custom_uninstall:custom_your_level_id"`
  - 用于其他自定义行为

### 与传统 `files` 的区别

**传统方式（`files` 数组）**：
```json
{
  "files": [
	{
	  "name": "bin",
	  "type": "folder",
	  "children": [
		{
		  "name": "app.exe",
		  "type": "file",
		  "size": 2048000
		}
	  ]
	}
  ]
}
```

**新方式（`customFileStructure`）**：
```json
{
  "customFileStructure": {
	"bin/app.exe": {
	  "type": "executable",
	  "size": 2048000
	}
  }
}
```

### 注意事项

1. 使用`customFileStructure`
2. 父目录会自动创建，无需手动定义每一层
3. 卸载程序的 `actualPath` 会自动设置，最好手动指定

## 测试流程

1. 在 Godot 编辑器中测试场景
2. 确认所有按钮可以点击
3. 确认信号正确发射
4. 将resources文件夹(包括内容) + level.json + manifest.json打包为zip,然后将后缀改成 `uslevel` 文件
5. 在游戏中导入并测试
6. 检查是否能正确返回桌面

---

*最后更新: 2026-03-29*
*基于 FastReader 关卡*

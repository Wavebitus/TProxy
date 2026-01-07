# 需求文档

## 简介

TProxy2 是一款 Windows 全局透明代理软件，定位为 Proxifier 的简化版本。该系统将本机网络流量根据路由规则转发给代理服务器或直连，提供可视化配置界面、内核插件架构、连接监控与统一日志功能。

技术栈：Python，UI 支持两种实现方式：
1. 类似 htop 的终端用户界面（TUI）
2. 基于 PyQt 的图形用户界面（GUI）

架构要求：采用 MVVM 架构，除表现层（View）外，其他层（Model、ViewModel）不依赖 PyQt。GUI 和 TUI 使用独立的 ViewModel 实现，共享相同的 Model 层。

## 术语表

- **TProxy2**：本系统的主程序，负责配置管理、内核控制、连接监控与日志展示
- **内核（Kernel）**：负责实际转发/透明代理能力的后端实现（如 singbox）
- **内核适配器（Kernel_Adapter）**：将内核实现抽象为可插拔模块的接口层
- **路由规则（Routing_Rule）**：依据进程名、目标 hosts、目标 ports 决定"直连/封锁/指定代理"的规则
- **默认路由规则（Default_Rule）**：当无其他规则命中时使用的兜底规则，固定在规则列表最下方
- **代理服务器（Proxy_Server）**：接收并转发网络流量的中间服务器
- **代理回环（Proxy_Loop）**：代理流量被再次代理导致循环/死循环/异常延迟
- **内核状态（Kernel_Status）**：内核运行状态枚举，包括已停止、启动中、运行中、停止中
- **连接状态（Connection_Status）**：网络连接的当前状态，包括连接中、连接成功、连接出错、连接关闭
- **业务规则文件（Rules_File）**：存储代理服务器、路由规则、DNS 设置的 rules.json 文件
- **偏好设置文件（Preference_File）**：存储 UI 显示偏好的 preference.json 文件
- **View**：MVVM 架构中的视图层，负责 UI 渲染，PyQt GUI 或 TUI
- **ViewModel**：MVVM 架构中的视图模型层，负责业务逻辑与状态管理，不依赖 PyQt
- **Model**：MVVM 架构中的模型层，负责数据存储与持久化，不依赖 PyQt
- **TUI**：终端用户界面（Terminal User Interface），类似 htop 的全屏终端交互界面

## 需求

### 需求 1：内核选择与显示

**用户故事：** 作为用户，我希望能够在界面上选择和查看可用的代理内核，以便控制使用哪个内核进行透明代理。

#### 验收标准

1. WHEN 用户查看主界面工具栏 THEN TProxy2 SHALL 显示内核选择下拉列表，列出所有可用的内核插件
2. WHEN 用户从下拉列表选择一个内核 THEN TProxy2 SHALL 将该内核设置为当前选中内核
3. THE TProxy2 SHALL 在任意时刻仅允许一个内核处于工作状态
4. WHEN 用户查看内核选择下拉框右侧 THEN TProxy2 SHALL 显示启动、停止、重启三个操作按钮
5. WHEN 内核状态发生变化 THEN TProxy2 SHALL 在 500 毫秒内更新操作按钮的可用性状态

### 需求 2：内核状态机与按钮控制

**用户故事：** 作为用户，我希望内核操作按钮的可用性能够根据内核状态自动变化，以便避免无效操作。

#### 验收标准

1. WHILE Kernel_Status 处于已停止状态 THEN TProxy2 SHALL 启用启动按钮，禁用停止按钮和重启按钮
2. WHILE Kernel_Status 处于启动中状态 THEN TProxy2 SHALL 禁用启动按钮、停止按钮和重启按钮
3. WHILE Kernel_Status 处于运行中状态 THEN TProxy2 SHALL 禁用启动按钮，启用停止按钮和重启按钮
4. WHILE Kernel_Status 处于停止中状态 THEN TProxy2 SHALL 禁用启动按钮、停止按钮和重启按钮
5. WHEN Kernel_Status 变化时 THEN ViewModel SHALL 通知 View 更新按钮状态

### 需求 3：内核安全切换

**用户故事：** 作为用户，我希望在切换或停止内核时收到适当的提示，以便了解操作可能带来的影响并做出决定。

#### 验收标准

1. WHEN 用户对运行中的内核执行停止或切换操作 THEN TProxy2 SHALL 弹出确认对话框，提示"正在联网的应用的网络连接会中断"
2. WHEN 用户在确认对话框点击确定 THEN TProxy2 SHALL 执行停止或切换操作
3. WHEN 用户在确认对话框点击取消 THEN TProxy2 SHALL 取消操作，保持当前内核状态不变
4. WHEN 用户对已停止的内核执行切换操作 THEN TProxy2 SHALL 直接执行切换，不显示确认对话框
5. WHILE Kernel_Status 处于启动中或停止中状态 THEN TProxy2 SHALL 禁止用户执行切换内核操作
6. WHEN 切换内核时 THEN TProxy2 SHALL 先请求当前内核安全退出，等待退出完成后再启动新内核
7. IF 旧内核退出失败 THEN TProxy2 SHALL 显示用户可理解的错误提示，并阻止启动新内核
8. WHEN 内核停止后 THEN TProxy2 SHALL 清空连接监控表格中的所有连接记录

### 需求 4：内核插件接口

**用户故事：** 作为开发者，我希望内核实现被抽象为可插拔模块，以便后续扩展支持其他内核。

#### 验收标准

1. THE Kernel_Adapter 接口 SHALL 提供 start 方法用于启动内核进程
2. THE Kernel_Adapter 接口 SHALL 提供 stop 方法用于优雅停止内核进程
3. THE Kernel_Adapter 接口 SHALL 提供 restart 方法用于重启内核进程
4. THE Kernel_Adapter 接口 SHALL 提供 apply_config 方法用于根据业务规则更新内核配置
5. THE Kernel_Adapter 接口 SHALL 提供 get_status 方法返回 Kernel_Status（已停止、启动中、运行中、停止中）
6. THE Kernel_Adapter 接口 SHALL 提供 get_connections_snapshot 方法返回当前连接快照
7. THE Kernel_Adapter 接口 SHALL 提供 subscribe_connections 方法用于订阅连接事件流
8. THE Kernel_Adapter 接口 SHALL 提供 subscribe_status 方法用于订阅内核状态变化
9. WHEN singbox 内核被调用 THEN TProxy2 SHALL 通过 Kernel_Adapter 接口调用，不直接依赖 singbox 实现细节
10. THE Kernel_Adapter 接口 SHALL 定义为 Python 抽象基类，不依赖 PyQt

### 需求 5：内核连接状态输出

**用户故事：** 作为用户，我希望能够实时查看当前的网络连接状态，以便监控代理运行情况。

#### 验收标准

1. WHILE Kernel_Status 处于运行中状态 THEN Kernel_Adapter SHALL 持续提供实时的进程网络连接状态信息
2. THE 连接状态信息 SHALL 包含进程名、目标地址、目标端口、Connection_Status、连接时长、路由结果、发送流量、接收流量、上传速度、下载速度字段
3. WHEN 新连接建立时 THEN Kernel_Adapter SHALL 通过事件流推送新增连接事件
4. WHEN 连接状态变化时 THEN Kernel_Adapter SHALL 通过事件流推送更新事件
5. WHEN 连接关闭时 THEN Kernel_Adapter SHALL 通过事件流推送关闭事件

### 需求 6：Singbox 内核实现

**用户故事：** 作为用户，我希望能够使用 singbox 作为代理内核，以便实现透明代理功能。

#### 验收标准

1. THE singbox 插件 SHALL 将可执行文件放置在 ./plugins/singbox/singbox.exe 路径
2. THE singbox 插件 SHALL 将配置文件放置在 ./plugins/singbox/config.json 路径
3. WHEN 启动 singbox 内核时 THEN TProxy2 SHALL 根据 Routing_Rule 列表更新 config.json 文件
4. IF config.json 文件不存在 THEN TProxy2 SHALL 报错并判定启动失败，保持 Kernel_Status 为已停止
5. IF singbox.exe 文件不存在 THEN TProxy2 SHALL 报错并判定启动失败，保持 Kernel_Status 为已停止
6. WHEN singbox 进程启动后 THEN TProxy2 SHALL 将 Kernel_Status 设置为启动中
7. WHEN singbox 控制台输出包含字符串 sing-box started THEN TProxy2 SHALL 将 Kernel_Status 设置为运行中
8. WHEN 请求停止 singbox 进程时 THEN TProxy2 SHALL 将 Kernel_Status 设置为停止中
9. WHEN singbox 进程退出后 THEN TProxy2 SHALL 将 Kernel_Status 设置为已停止

### 需求 7：设置持久化

**用户故事：** 作为用户，我希望我的配置能够自动保存和加载，以便下次启动时无需重新配置。

#### 验收标准

1. WHEN 用户修改业务规则后 THEN TProxy2 SHALL 自动将业务规则保存到应用程序同目录的 rules.json 文件
2. WHEN 用户修改显示偏好后 THEN TProxy2 SHALL 自动将显示偏好保存到应用程序同目录的 preference.json 文件
3. WHEN TProxy2 启动时 THEN TProxy2 SHALL 从 rules.json 文件加载业务规则数据
4. WHEN TProxy2 启动时 THEN TProxy2 SHALL 从 preference.json 文件加载 GUI 偏好设置
5. IF rules.json 文件不存在 THEN TProxy2 SHALL 使用默认业务规则配置
6. IF preference.json 文件不存在 THEN TProxy2 SHALL 使用默认偏好设置
7. THE rules.json 文件 SHALL 包含 version 字段用于版本管控
8. THE preference.json 文件 SHALL 包含 version 字段用于版本管控
9. THE Model 层 SHALL 负责 JSON 文件的读写操作，不依赖 PyQt

### 需求 8：多语言支持

**用户故事：** 作为用户，我希望能够切换界面语言，以便使用我熟悉的语言操作软件。

#### 验收标准

1. THE TProxy2 SHALL 支持切换界面显示语言
2. THE 多语言配置文件 SHALL 使用 JSON 格式，文件名为 {语言简写}.language，存放在 ./languages/ 目录
3. THE TProxy2 SHALL 内置 en-US.language 英文语言文件
4. THE TProxy2 SHALL 内置 zh-CN.language 简体中文语言文件
5. WHEN 用户切换语言后 THEN TProxy2 SHALL 更新工具栏按钮、弹窗标题、表格列头等界面文本
6. THE 语言切换逻辑 SHALL 在 ViewModel 层实现，不依赖 PyQt

### 需求 9：主题支持

**用户故事：** 作为用户，我希望能够切换界面主题颜色，以便获得舒适的视觉体验。

#### 验收标准

1. THE TProxy2 SHALL 支持切换界面主题颜色
2. THE 主题配置文件 SHALL 使用 JSON 格式，文件名为 {主题名}.theme，存放在 ./themes/ 目录
3. THE TProxy2 SHALL 内置 light.theme 白色主题作为默认主题
4. THE TProxy2 SHALL 内置 dark.theme 暗色主题
5. WHEN 用户切换主题后 THEN TProxy2 SHALL 更新强调色、表格选中色、按钮颜色等主要控件颜色
6. THE 主题数据 SHALL 在 Model 层加载，主题切换逻辑 SHALL 在 ViewModel 层实现

### 需求 10：代理服务器管理

**用户故事：** 作为用户，我希望能够管理代理服务器列表，以便配置不同的代理连接。

#### 验收标准

1. WHEN 用户点击工具栏代理服务器管理按钮 THEN TProxy2 SHALL 弹出代理服务器管理界面
2. THE 代理服务器管理界面 SHALL 显示包含名称、地址、端口、协议类型列的表格
3. THE 代理服务器管理界面 SHALL 在右侧提供添加、修改、删除、检查操作按钮
4. WHEN 用户添加或修改 Proxy_Server 时 THEN TProxy2 SHALL 提供名称、地址、端口、协议字段的编辑界面
5. THE 协议字段 SHALL 支持 http 和 socks5 两种选项
6. WHEN 用户保存 Proxy_Server 时 THEN TProxy2 SHALL 校验名称唯一性，重复名称无法保存并提示原因
7. WHEN 用户删除被 Routing_Rule 引用的 Proxy_Server 时 THEN TProxy2 SHALL 弹窗列出引用该代理的规则列表
8. WHEN 存在引用时 THEN TProxy2 SHALL 允许用户为这些规则批量调整路由行为为直连、封锁或其他代理服务器

### 需求 11：代理服务器检查器

**用户故事：** 作为用户，我希望能够测试代理服务器的连接状态，以便确认代理配置正确。

#### 验收标准

1. WHEN 用户点击检查按钮 THEN TProxy2 SHALL 打开代理服务器检查器界面
2. THE 检查器界面 SHALL 显示被检查 Proxy_Server 的地址与协议
3. THE 检查器 SHALL 提供连接代理服务器测试选项，可开关
4. THE 检查器 SHALL 提供通过代理访问指定网址测试选项，可开关，支持自定义网址，默认值为 www.google.com:80
5. THE 检查器 SHALL 提供测试连接延迟选项，可开关
6. WHEN 测试成功时 THEN 检查器日志窗口 SHALL 以绿色显示成功结果
7. WHEN 测试失败时 THEN 检查器日志窗口 SHALL 以红色显示失败结果
8. THE 检查器测试逻辑 SHALL 在 ViewModel 层实现，不依赖 PyQt

### 需求 12：路由规则管理

**用户故事：** 作为用户，我希望能够配置路由规则，以便控制不同应用和目标的网络流量走向。

#### 验收标准

1. WHEN 用户点击工具栏路由规则配置按钮 THEN TProxy2 SHALL 弹出路由规则配置界面
2. THE 路由规则表格 SHALL 显示是否启用、规则名称、应用程序名称、目标 hosts、目标 ports、路由行为列
3. WHEN 应用程序名称、目标 hosts 或目标 ports 为空时 THEN 表格单元格 SHALL 以灰色显示"任何"
4. THE 路由行为列 SHALL 提供直连、指定代理服务器、封锁三个选项的下拉单选框
5. WHEN 用户双击规则行 THEN TProxy2 SHALL 弹出编辑路由规则属性对话框
6. THE 配置界面下边缘 SHALL 提供添加、复制、编辑、移除按钮
7. WHEN 用户点击复制按钮 THEN TProxy2 SHALL 生成新规则，名称为"复制 自 {原路由规则名}"，其他属性与原规则一致

### 需求 13：路由规则编辑

**用户故事：** 作为用户，我希望能够编辑路由规则的详细属性，以便精确控制流量匹配条件。

#### 验收标准

1. THE 规则编辑对话框 SHALL 提供是否启用复选框
2. THE 规则编辑对话框 SHALL 提供规则名称单行输入框，新建默认值为"新规则"，不允许为空
3. THE 规则编辑对话框 SHALL 提供应用程序名称多行输入框，占位符为"任何"，多值以分号分隔
4. THE 应用程序名称输入框下方 SHALL 显示示例文本：example.exe; chrom*.exe;
5. THE 规则编辑对话框 SHALL 提供目标 hosts 多行输入框，占位符为"任何"，多值以分号分隔
6. THE 目标 hosts 输入框下方 SHALL 显示示例文本：127.0.0.1; *.baidu.com; 10.10.1.1-10.10.255.255;
7. THE 规则编辑对话框 SHALL 提供目标 ports 单行输入框，占位符为"任何"，多值以分号分隔
8. THE 目标 ports 输入框下方 SHALL 显示示例文本：80;443;1000-2000
9. THE 规则编辑对话框 SHALL 提供路由行为下拉框，选项包含直连、指定代理服务器、封锁，不允许为空
10. WHEN 用户点击确定按钮 THEN TProxy2 SHALL 校验必填字段，校验不通过时提示具体字段
11. WHEN 用户点击取消按钮 THEN TProxy2 SHALL 关闭对话框，不保存修改

### 需求 14：路由规则顺序与默认规则

**用户故事：** 作为用户，我希望能够调整路由规则的优先级顺序，并确保始终有一条兜底规则。

#### 验收标准

1. THE Routing_Rule 列表 SHALL 按从上到下的顺序处理
2. THE 配置界面 SHALL 提供上下箭头按钮用于调整规则顺序
3. WHEN 用户调整规则顺序后 THEN TProxy2 SHALL 保存新顺序并按此顺序传递给内核
4. THE 规则列表最下方 SHALL 固定显示一条 Default_Rule
5. THE Default_Rule SHALL 不可更改顺序，始终位于最下方
6. THE Default_Rule SHALL 除路由行为外其他字段不可编辑
7. THE Default_Rule SHALL 固定启用状态，名称为"默认"，应用程序名称、hosts、ports 均为空
8. WHEN 用户双击 Default_Rule THEN TProxy2 SHALL 弹窗提示"无法编辑默认路由规则"
9. THE Default_Rule SHALL 不可删除

### 需求 15：路由规则确认与取消

**用户故事：** 作为用户，我希望能够确认或取消路由规则的修改，以便控制配置变更的生效时机。

#### 验收标准

1. THE 路由规则配置界面最下方 SHALL 提供确认和取消按钮
2. WHEN 用户点击确认按钮 THEN TProxy2 SHALL 应用并保存规则修改到 rules.json 文件，然后关闭界面
3. WHEN 用户点击取消按钮 THEN TProxy2 SHALL 不应用修改，直接关闭界面
4. WHEN 用户取消后重新打开配置界面 THEN TProxy2 SHALL 显示上次确认保存的规则配置

### 需求 16：DNS 设置

**用户故事：** 作为用户，我希望能够配置 DNS 解析方式，以便控制域名解析的行为。

#### 验收标准

1. WHEN 用户点击工具栏 DNS 设置按钮 THEN TProxy2 SHALL 弹出 DNS 配置界面
2. THE DNS 配置界面 SHALL 提供默认 DNS 设置选项
3. THE DNS 配置界面 SHALL 提供通过代理解析 DNS 选项
4. WHEN 用户保存 DNS 设置后 THEN TProxy2 SHALL 将设置写入 rules.json 文件

### 需求 17：连接监控表格

**用户故事：** 作为用户，我希望能够实时查看当前的网络连接情况，以便监控代理运行状态。

#### 验收标准

1. THE 主窗口上半部分 SHALL 显示当前联网进程连接情况表格
2. THE 连接表格 SHALL 显示进程名字、目标地址和端口、连接时长和状态、使用的路由规则、发送流量、收到流量、上传速度、下载速度列
3. THE 连接表格列头 SHALL 支持调整宽度
4. WHEN 新连接建立时 THEN TProxy2 SHALL 将新连接显示在表格最后一行
5. WHILE Connection_Status 处于连接中状态 THEN 该行 SHALL 以蓝色显示
6. WHILE Connection_Status 处于连接出错状态 THEN 该行 SHALL 以红色显示
7. WHILE Connection_Status 处于连接成功状态 THEN 该行 SHALL 以黑灰色显示
8. WHEN Connection_Status 变为连接关闭时 THEN 该行 SHALL 以灰色显示，2 秒后从表格消失
9. WHEN Kernel_Status 变为已停止后 THEN TProxy2 SHALL 清空连接监控表格

### 需求 18：连接表格过滤

**用户故事：** 作为用户，我希望能够过滤连接列表，以便快速找到关注的连接。

#### 验收标准

1. THE 连接表格上边缘 SHALL 提供搜索过滤输入框
2. WHEN 用户输入过滤关键字 THEN TProxy2 SHALL 仅显示任意字段匹配该关键字的连接行
3. THE 连接表格上边缘 SHALL 提供显示直连复选框
4. THE 显示直连复选框 SHALL 默认为关闭状态
5. WHEN 显示直连复选框关闭时 THEN TProxy2 SHALL 隐藏路由行为为直连的连接
6. WHEN 用户修改显示直连设置后 THEN TProxy2 SHALL 将该设置持久化到 preference.json 文件

### 需求 19：日志采集与持久化

**用户故事：** 作为用户，我希望程序日志能够被记录到文件，以便后续排查问题。

#### 验收标准

1. WHEN TProxy2 启动后 THEN TProxy2 SHALL 自动创建日志文件，文件名格式为 {年-月-日-小时-分钟-秒}.log
2. THE 日志文件 SHALL 保存在应用程序同目录
3. WHILE TProxy2 运行期间 THEN TProxy2 SHALL 实时将程序日志和内核日志追加写入日志文件
4. THE 日志采集逻辑 SHALL 在 Model 层实现，不依赖 PyQt

### 需求 20：日志 UI 显示

**用户故事：** 作为用户，我希望能够在界面上查看程序日志，以便实时了解运行状态。

#### 验收标准

1. THE 主窗口下半部分 SHALL 显示程序输出日志与内核日志
2. THE 日志 SHALL 以只读文本组件显示，每条日志换行
3. THE 日志文本 SHALL 支持选中复制
4. THE 普通日志文字 SHALL 以黑灰色显示
5. WHEN 日志行包含 error、warning、warn 或 failed 关键字时 THEN 该行 SHALL 以红色显示
6. THE 关键字匹配 SHALL 大小写不敏感，按子串匹配
7. THE 日志高亮逻辑 SHALL 在 ViewModel 层实现，不依赖 PyQt

### 需求 21：日志窗口功能

**用户故事：** 作为用户，我希望能够过滤、清除和控制日志显示行为，以便更好地查看日志。

#### 验收标准

1. THE 日志窗口上边缘 SHALL 提供搜索过滤输入框
2. WHEN 用户输入过滤关键字 THEN TProxy2 SHALL 仅显示匹配该关键字的日志行
3. WHEN 用户清空过滤关键字 THEN TProxy2 SHALL 恢复显示全部日志
4. THE 日志窗口上边缘 SHALL 提供清除日志按钮
5. WHEN 用户点击清除日志按钮 THEN TProxy2 SHALL 清除 UI 显示的日志，不清除磁盘上的日志文件
6. THE 日志窗口上边缘 SHALL 提供自动刷新开关，默认开启
7. THE 日志窗口上边缘 SHALL 提供自动滚动开关，默认开启
8. WHEN 用户修改自动刷新或自动滚动设置后 THEN TProxy2 SHALL 将设置持久化到 preference.json 文件

### 需求 22：代理回环监测

**用户故事：** 作为用户，我希望系统能够避免代理回环，以便防止网络异常。

#### 验收标准

1. THE TProxy2 SHALL 监测并规避 Proxy_Loop
2. THE Proxy_Loop 规避 SHALL 覆盖 TProxy2 主程序自身的网络访问
3. THE Proxy_Loop 规避 SHALL 覆盖内核程序（如 singbox）的网络访问
4. THE Proxy_Loop 规避 SHALL 覆盖上游代理服务器应用的网络访问，确保其使用直连

### 需求 23：错误处理与可靠性

**用户故事：** 作为用户，我希望系统在出错时能够给出明确提示，以便了解问题并采取措施。

#### 验收标准

1. WHEN 内核启动失败时 THEN TProxy2 SHALL 显示用户可理解的错误提示
2. WHEN 内核停止失败时 THEN TProxy2 SHALL 显示用户可理解的错误提示
3. WHEN 内核切换失败时 THEN TProxy2 SHALL 显示用户可理解的错误提示
4. WHEN 配置文件写入失败时 THEN TProxy2 SHALL 显示错误提示
5. THE TProxy2 SHALL 不允许静默失败，所有错误情况都应有明确的用户反馈
6. THE 错误处理逻辑 SHALL 在 ViewModel 层实现，View 层仅负责显示错误信息

### 需求 24：MVVM 架构约束

**用户故事：** 作为开发者，我希望系统采用 MVVM 架构，以便实现 UI 与业务逻辑的分离，支持多种 UI 实现。

#### 验收标准

1. THE TProxy2 SHALL 采用 MVVM 架构，分为 Model、ViewModel、View 三层
2. THE Model 层 SHALL 负责数据存储、持久化、内核适配器接口定义
3. THE ViewModel 层 SHALL 负责业务逻辑、状态管理、数据转换
4. THE View 层 SHALL 负责 UI 渲染和用户交互
5. THE Model 层 SHALL 不依赖 PyQt，使用纯 Python 实现
6. THE ViewModel 层 SHALL 不依赖 PyQt，使用纯 Python 实现
7. THE View 层 SHALL 支持两种实现：PyQt GUI 和终端 TUI
8. THE GUI ViewModel SHALL 通过 Observable 模式通知 View 更新
9. THE TUI ViewModel SHALL 采用命令式交互模式，通过轮询或回调获取数据
10. THE View SHALL 通过命令模式或方法调用与 ViewModel 交互
11. THE GUI 和 TUI SHALL 使用独立的 ViewModel 实现，共享相同的 Model 层

### 需求 25：控制台 TUI 界面

**用户故事：** 作为用户，我希望能够通过类似 htop 的终端用户界面操作 TProxy2，以便在无图形界面环境下获得与 GUI 相似的交互体验。

#### 验收标准

1. THE TProxy2 SHALL 提供类似 htop 风格的终端用户界面（TUI）作为 View 层的一种实现
2. THE TUI SHALL 采用全屏终端界面，布局与 PyQt GUI 相同
3. THE TUI 顶部 SHALL 显示快捷键提示栏，格式为"F1:启动 F2:停止 F3:重启 F4:代理服务器 F5:路由规则 F6:DNS Q:退出"
4. THE TUI 主体上半部分 SHALL 显示当前连接列表表格，实时更新
5. THE TUI 主体下半部分 SHALL 显示日志输出区域，实时滚动
6. THE TUI SHALL 支持通过快捷键执行内核启动、停止、重启操作
7. THE TUI SHALL 支持通过快捷键打开代理服务器管理、路由规则配置、DNS 设置的子界面
8. THE TUI 连接列表 SHALL 支持上下方向键滚动浏览
9. THE TUI 连接列表 SHALL 支持按列排序
10. THE TUI 日志区域 SHALL 支持自动滚动到最新日志
11. THE TUI SHALL 使用 Python curses 或 rich 库实现终端渲染
12. THE TUI SHALL 使用独立的 CLI ViewModel 层，不复用 GUI ViewModel
13. THE TUI ViewModel SHALL 采用命令式交互模式，不使用 Observable 模式
14. THE TUI SHALL 复用与 PyQt GUI 相同的 Model 层
15. WHEN 内核状态变化时 THEN TUI SHALL 实时更新顶部状态显示
16. WHEN 新连接建立或状态变化时 THEN TUI SHALL 实时更新连接列表
17. THE TUI 连接列表 SHALL 使用颜色区分连接状态：蓝色表示连接中、红色表示出错、灰色表示已关闭
18. THE TUI 日志区域 SHALL 使用红色高亮显示包含 error、warning、warn、failed 关键字的日志行

# 实现计划：TProxy2 Windows 全局透明代理软件

## 概述

本实现计划将 TProxy2 设计文档转换为可执行的编码任务。采用测试驱动开发（TDD），从核心基础设施开始，逐层构建 Model、ViewModel 和 View 层。

## 任务列表

- [x] 1. 项目初始化与核心基础设施
  - [x] 1.1 创建项目结构和 Python 虚拟环境
    - 创建目录结构：src/core/, src/model/, src/viewmodel/, src/view/, tests/
    - 初始化 pyproject.toml，配置 pytest、hypothesis、pytest-cov 依赖
    - 创建 conftest.py 配置测试 fixtures
    - _Requirements: 24.5, 24.6_、、
  - [x] 1.2 实现值对象模块 (src/core/value_objects.py)
    - 实现 IpAddress、Port、PortRange、HostPattern、ProxyProtocol 值对象
    - 实现 FilePath、ExistingFilePath、DirectoryPath 值对象
    - 实现 ServerName、RuleName、ProcessPattern、RouteActionType 值对象
    - 所有值对象在 __post_init__ 中验证业务规则
    - _Requirements: 10.4, 10.6, 13.2, 13.7, 13.8_

  - [x] 1.3 编写值对象属性测试 (tests/core/test_value_objects.py)
    - **Property 19: IpAddress 有效性验证**
    - **Property 20: Port 范围验证 (1-65535)**
    - **Property 21: PortRange 格式和范围验证**
    - **Property 22: ServerName/RuleName 非空验证**
    - **Validates: Requirements 10.4, 10.6, 13.2, 13.7, 13.8**

  - [x] 1.4 实现抽象接口模块 (src/core/interfaces.py)
    - 定义 IConfigReader、IConfigWriter Protocol
    - 定义 ILogSink Protocol
    - 定义 IKernelAdapter 抽象基类
    - _Requirements: 4.1-4.10, 24.5, 24.6_

  - [x] 1.5 实现依赖注入容器 (src/core/container.py)
    - 实现 Container 类，支持 register/resolve
    - 支持单例和工厂模式
    - _Requirements: 24.5, 24.6_

- [x] 2. 检查点 - 核心基础设施完成
  - 确保所有值对象测试通过 ✓ (48 tests)
  - 确保接口定义完整 ✓
  - 如有问题请询问用户

- [x] 3. Model 层数据类型与持久化
  - [x] 3.1 实现数据类型模块 (src/model/data_types.py)
    - 实现 ProxyServer、RouteAction、RoutingRule 数据类（使用值对象）
    - 实现 DnsSettings、RulesConfig、PreferenceConfig 数据类
    - 实现 to_dict()/from_dict() 序列化方法
    - _Requirements: 7.7, 7.8_

  - [x] 3.2 编写数据类型属性测试 (tests/model/test_data_types.py)
    - **Property 23: 数据序列化往返一致性**
    - **Validates: Requirements 7.1, 7.3**

  - [x] 3.3 实现配置持久化 (src/model/config_store.py)
    - 实现 ConfigStore 类，实现 IConfigReader、IConfigWriter 接口
    - 实现 load_rules()、save_rules() 方法
    - 实现 load_preference()、save_preference() 方法
    - 实现默认配置生成
    - _Requirements: 7.1-7.9_

  - [x] 3.4 编写配置持久化属性测试 (tests/model/test_config_store.py)
    - **Property 8: 配置持久化往返一致性**
    - **Property 9: 偏好设置持久化往返一致性**
    - **Validates: Requirements 7.1, 7.2, 7.3, 7.4**

- [x] 4. 检查点 - 数据层完成
  - 确保配置文件读写测试通过 ✓ (14 tests)
  - 确保序列化往返一致性 ✓ (18 tests)
  - 如有问题请询问用户

- [x] 5. Model 层核心业务逻辑
  - [x] 5.1 实现配置验证器 (src/model/config_validator.py)
    - 实现 ConfigValidator 类
    - 实现 validate_proxy_server_name() 名称唯一性验证
    - 实现 validate_routing_rule() 规则验证
    - 实现 get_rules_referencing_proxy() 引用检测
    - _Requirements: 10.6, 10.7, 13.10_

  - [x] 5.2 编写配置验证器属性测试 (tests/model/test_config_validator.py)
    - **Property 7: 代理服务器名称唯一性**
    - **Property 18: 代理服务器删除引用检测**
    - **Validates: Requirements 10.6, 10.7**

  - [x] 5.3 实现规则引擎 (src/model/rule_engine.py)
    - 实现 RuleEngine 类
    - 实现 add_rule()、move_rule()、copy_rule() 方法
    - 实现 ensure_default_rule() 默认规则约束
    - _Requirements: 12.7, 14.1-14.9_

  - [x] 5.4 编写规则引擎属性测试 (tests/model/test_rule_engine.py)
    - **Property 10: 默认规则不变量**
    - **Property 11: 默认规则不可移动**
    - **Property 12: 规则复制属性一致**
    - **Validates: Requirements 12.7, 14.4, 14.5, 14.9**

  - [x] 5.5 实现代理检查器 (src/model/proxy_checker.py)
    - 实现 ProxyChecker 类
    - 实现连接测试、延迟测试、目标访问测试
    - _Requirements: 11.3-11.7_

- [x] 6. 检查点 - 业务逻辑完成
  - 确保所有业务逻辑测试通过 ✓ (19 + 26 tests)
  - 确保规则引擎约束正确 ✓
  - 如有问题请询问用户

- [x] 7. Model 层内核管理
  - [x] 7.1 实现内核适配器抽象 (src/model/kernel_adapter.py)
    - 定义 KernelStatus 枚举 (在 interfaces.py)
    - 定义 ConnectionStatus 枚举 (在 interfaces.py)
    - 定义 ConnectionInfo 数据类 (在 interfaces.py)
    - 实现 KernelAdapter 抽象基类
    - _Requirements: 4.1-4.10_

  - [x] 7.2 实现 Singbox 适配器 (plugins/singbox/adapter.py)
    - 实现 SingboxAdapter 类
    - 实现 start()、stop()、restart() 进程管理
    - 实现 apply_config() 配置转换
    - 实现状态检测（STARTED_MARKER）
    - 实现连接事件订阅
    - 可执行文件路径: ./plugins/singbox/singbox.exe (Requirement 6.1)
    - 配置文件路径: ./plugins/singbox/config.json (Requirement 6.2)
    - _Requirements: 6.1-6.9_

  - [x] 7.3 实现内核管理器 (src/model/kernel_manager.py)
    - 实现 KernelManager 类
    - 实现 start()、stop()、switch_kernel() 方法
    - 实现单实例约束
    - 实现 requires_confirmation() 确认逻辑
    - _Requirements: 1.3, 3.1-3.8_

  - [x] 7.4 编写内核管理属性测试 (tests/model/test_kernel_manager.py)
    - **Property 1: 内核单实例不变量**
    - **Property 2: 内核状态机按钮可用性**
    - **Property 3: 运行中内核操作需确认**
    - **Property 4: 已停止内核切换无需确认**
    - **Property 5: 过渡状态禁止切换**
    - **Validates: Requirements 1.3, 2.1-2.4, 3.1, 3.3, 3.4, 3.5**

- [x] 8. 检查点 - 内核管理完成
  - 确保内核状态机测试通过 ✓ (26 tests)
  - 确保单实例约束正确 ✓
  - 如有问题请询问用户

- [x] 9. Model 层辅助组件
  - [x] 9.1 实现日志采集器 (src/model/log_collector.py)
    - 实现 LogEntry 数据类 (在 interfaces.py)
    - 实现 LogCollector 类
    - 实现日志文件创建（时间戳命名）
    - 实现订阅/取消订阅机制
    - _Requirements: 19.1-19.4_

  - [x] 9.2 实现国际化管理器 (src/model/i18n_manager.py)
    - 实现 I18nManager 类
    - 实现 load_language()、get_string() 方法
    - 语言文件已存在: zh-CN.language、en-US.language
    - _Requirements: 8.1-8.6_

  - [x] 9.3 实现主题管理器 (src/model/theme_manager.py)
    - 实现 ThemeColors 数据类
    - 实现 ThemeManager 类
    - 主题文件已存在: light.theme、dark.theme
    - _Requirements: 9.1-9.6_

  - [x] 9.4 实现 TUI 配色管理器 (src/model/tui_color_manager.py)
    - 实现 TuiColorManager 类
    - 支持 default、monochrome 配色方案
    - _Requirements: 25.17, 25.18_

- [x] 10. 检查点 - Model 层完成
  - 确保所有 Model 层测试通过 ✓ (151 tests total)
  - 确保 Model 层不依赖 PyQt ✓
  - 如有问题请询问用户

- [x] 11. ViewModel 层基础设施
  - [x] 11.1 实现 Observable 基类 (src/viewmodel/observable.py)
    - 实现 Observable[T] 泛型类
    - 实现 subscribe/unsubscribe/notify 机制
    - _Requirements: 2.5, 24.8_

  - [x] 11.2 编写 Observable 属性测试
    - **Property 17: Observable 通知一致性**
    - **Validates: Requirements 2.5**

- [x] 12. GUI ViewModel 实现
  - [x] 12.1 实现 GUI 内核 ViewModel (src/viewmodel/gui/kernel_vm.py)
    - 实现 GuiKernelViewModel 类
    - 实现 Observable 状态属性
    - 实现按钮可用性逻辑
    - 实现确认对话框逻辑
    - _Requirements: 1.1-1.5, 2.1-2.5, 3.1-3.4_

  - [x] 12.2 实现 GUI 代理服务器 ViewModel (src/viewmodel/gui/proxy_server_vm.py)
    - 实现 GuiProxyServerViewModel 类
    - 实现 CRUD 操作
    - 实现引用检测和批量更新
    - _Requirements: 10.1-10.8_

  - [x] 12.3 实现 GUI 路由规则 ViewModel (src/viewmodel/gui/routing_rule_vm.py)
    - 实现 GuiRoutingRuleViewModel 类
    - 实现规则编辑、复制、移动
    - 实现确认/取消逻辑
    - _Requirements: 12.1-12.7, 13.1-13.11, 14.1-14.9, 15.1-15.4_

  - [x] 12.4 编写 GUI 路由规则属性测试
    - **Property 16: 路由规则确认/取消语义**
    - **Validates: Requirements 15.3, 15.4**

  - [x] 12.5 实现 GUI 连接 ViewModel (src/viewmodel/gui/connection_vm.py)
    - 实现 GuiConnectionViewModel 类
    - 实现过滤逻辑
    - 实现关闭连接延迟消失
    - _Requirements: 17.1-17.9, 18.1-18.6_

  - [x] 12.6 编写 GUI 连接属性测试
    - **Property 6: 内核停止清空连接**
    - **Property 13: 连接过滤正确性**
    - **Property 14: 关闭连接延迟消失**
    - **Validates: Requirements 3.8, 17.8, 17.9, 18.2, 18.5**

  - [x] 12.7 实现 GUI 日志 ViewModel (src/viewmodel/gui/log_vm.py)
    - 实现 GuiLogViewModel 类
    - 实现过滤、错误检测、自动刷新/滚动
    - _Requirements: 20.1-20.7, 21.1-21.8_

  - [x] 12.8 编写 GUI 日志属性测试
    - **Property 15: 日志错误关键字检测**
    - **Validates: Requirements 20.5, 20.6**

  - [x] 12.9 实现 GUI 设置 ViewModel (src/viewmodel/gui/settings_vm.py)
    - 实现 GuiSettingsViewModel 类
    - 实现语言切换、主题切换
    - _Requirements: 8.5, 9.5_

- [x] 13. 检查点 - GUI ViewModel 完成
  - 确保所有 GUI ViewModel 测试通过 ✓ (138 tests)
  - 确保 ViewModel 不依赖 PyQt ✓
  - 如有问题请询问用户

- [x] 14. TUI ViewModel 实现
  - [x] 14.1 实现 TUI 内核 ViewModel (src/viewmodel/tui/kernel_vm.py)
    - 实现 TuiKernelViewModel 类
    - 实现 get_status() 返回字典
    - 实现状态回调机制
    - _Requirements: 1.1-1.5, 2.1-2.5, 25.6, 25.15_

  - [x] 14.2 实现 TUI 代理服务器 ViewModel (src/viewmodel/tui/proxy_server_vm.py)
    - 实现 TuiProxyServerViewModel 类
    - 实现 get_servers() 返回渲染数据
    - _Requirements: 10.1-10.8, 25.7_

  - [x] 14.3 实现 TUI 路由规则 ViewModel (src/viewmodel/tui/routing_rule_vm.py)
    - 实现 TuiRoutingRuleViewModel 类
    - 实现 get_rules() 返回渲染数据
    - _Requirements: 12.1-12.7, 25.7_

  - [x] 14.4 实现 TUI 连接 ViewModel (src/viewmodel/tui/connection_vm.py)
    - 实现 TuiConnectionViewModel 类
    - 实现 get_connections() 定时刷新
    - 实现颜色映射
    - _Requirements: 17.1-17.9, 25.4, 25.16, 25.17_

  - [x] 14.5 实现 TUI 日志 ViewModel (src/viewmodel/tui/log_vm.py)
    - 实现 TuiLogViewModel 类
    - 实现 get_logs() 返回渲染数据
    - 实现错误行颜色标记
    - _Requirements: 20.1-20.7, 25.5, 25.10, 25.18_

  - [x] 14.6 实现 TUI 主 ViewModel (src/viewmodel/tui/main_vm.py)
    - 实现 TuiMainViewModel 类
    - 实现 get_hotkeys() 快捷键列表
    - 实现 handle_key() 快捷键处理
    - _Requirements: 25.3, 25.6, 25.7_

- [x] 15. Headless ViewModel 实现
  - [x] 15.1 实现 Headless 内核 ViewModel (src/viewmodel/headless/kernel_vm.py)
    - 实现 HeadlessKernelViewModel 类
    - 实现同步 start/stop 方法
    - 实现 wait_for_status() 等待方法
    - _Requirements: 24.1-24.11_

  - [x] 15.2 实现 Headless 代理服务器 ViewModel (src/viewmodel/headless/proxy_server_vm.py)
    - 实现 HeadlessProxyServerViewModel 类
    - 实现同步 CRUD 方法
    - _Requirements: 10.1-10.8_

  - [x] 15.3 实现 Headless 路由规则 ViewModel (src/viewmodel/headless/routing_rule_vm.py)
    - 实现 HeadlessRoutingRuleViewModel 类
    - 实现同步规则管理方法
    - _Requirements: 12.1-12.7, 14.1-14.9_

  - [x] 15.4 实现 Headless 连接 ViewModel (src/viewmodel/headless/connection_vm.py)
    - 实现 HeadlessConnectionViewModel 类
    - 实现 get_connections() 和 get_connection_count()
    - _Requirements: 17.1-17.9_

  - [x] 15.5 实现 Headless 日志 ViewModel (src/viewmodel/headless/log_vm.py)
    - 实现 HeadlessLogViewModel 类
    - 实现 wait_for_log() 等待方法
    - _Requirements: 20.1-20.7_

  - [x] 15.6 实现 Headless 主 ViewModel (src/viewmodel/headless/main_vm.py)
    - 实现 HeadlessMainViewModel 类
    - 实现 create_for_testing() 工厂方法
    - _Requirements: 24.1-24.11_

- [x] 16. 检查点 - ViewModel 层完成
  - 确保所有 ViewModel 测试通过 ✓ (289 tests)
  - 确保 ViewModel 不依赖 PyQt ✓
  - 如有问题请询问用户

- [x] 17. TUI View 实现
  - [x] 17.1 实现 TUI 组件 (src/view/tui/widgets.py)
    - 实现表格组件（支持滚动、排序）
    - 实现日志区组件（支持自动滚动）
    - 实现快捷键栏组件
    - _Requirements: 25.8, 25.9, 25.10_

  - [x] 17.2 实现 TUI 子界面 (src/view/tui/screens.py)
    - 实现代理服务器管理界面
    - 实现路由规则配置界面
    - 实现 DNS 设置界面
    - _Requirements: 25.7_

  - [x] 17.3 实现 TUI 主程序 (src/view/tui/tui_app.py)
    - 实现 TuiApp 类（使用 rich 库）
    - 实现主循环和事件处理
    - 实现界面布局（快捷键栏、连接表格、日志区）
    - _Requirements: 25.1-25.18_

- [x] 18. 检查点 - TUI 完成
  - 确保 TUI 界面可正常运行
  - 确保快捷键响应正确
  - 如有问题请询问用户

- [x] 19. GUI View 实现
  - [x] 19.1 实现 GUI 主窗口 (src/view/gui/main_window.py)
    - 实现 MainWindow 类
    - 实现工具栏（内核选择、操作按钮）
    - 实现连接表格和日志区布局
    - _Requirements: 1.1, 1.4, 17.1-17.3, 20.1-20.3_

  - [x] 19.2 实现代理服务器对话框 (src/view/gui/proxy_server_dialog.py)
    - 实现代理服务器管理对话框
    - 实现添加/编辑/删除界面
    - 实现检查器界面
    - _Requirements: 10.1-10.8, 11.1-11.8_

  - [x] 19.3 实现路由规则对话框 (src/view/gui/routing_rule_dialog.py)
    - 实现路由规则配置对话框
    - 实现规则编辑对话框
    - 实现确认/取消按钮
    - _Requirements: 12.1-12.7, 13.1-13.11, 15.1-15.4_

  - [x] 19.4 实现 DNS 设置对话框 (src/view/gui/dns_dialog.py)
    - 实现 DNS 配置对话框
    - _Requirements: 16.1-16.4_

  - [x] 19.5 实现 GUI 组件 (src/view/gui/widgets/)
    - 实现连接表格组件（颜色、过滤）
    - 实现日志显示组件（颜色、过滤）
    - _Requirements: 17.4-17.8, 20.4-20.6_

- [x] 20. 检查点 - GUI 完成
  - 确保 GUI 界面可正常运行
  - 确保所有对话框功能正确
  - 如有问题请询问用户

- [x] 21. 集成与入口点
  - [x] 21.1 实现应用入口点 (main.py)
    - 实现命令行参数解析（--gui/--tui）
    - 实现依赖注入容器配置
    - 实现应用启动逻辑
    - _Requirements: 24.7, 25.11_

  - [x] 21.2 实现代理回环规避
    - 在路由规则中自动添加 TProxy2、singbox 进程的直连规则
    - _Requirements: 22.1-22.4_

- [x] 22. 集成测试
  - [x] 22.1 编写内核生命周期集成测试 (tests/integration/test_kernel_lifecycle.py)
    - 测试完整的启动/停止/切换流程
    - 使用 Headless ViewModel
    - _Requirements: 3.6, 3.7, 6.6-6.9_

  - [x] 22.2 编写 Headless 工作流集成测试 (tests/integration/test_headless_workflow.py)
    - 测试配置修改后内核行为
    - 测试多 ViewModel 协作
    - _Requirements: 24.1-24.11_

- [x] 23. 最终检查点 - 项目完成
  - 确保所有测试通过 ✓ (309 tests)
  - 确保 GUI 和 TUI 均可正常运行 ✓
  - 确保配置持久化正确 ✓
  - 如有问题请询问用户

## 备注

- 所有任务（包括测试任务）都必须完成
- 每个检查点确保当前阶段功能完整后再继续
- 属性测试使用 hypothesis 库，每个测试至少运行 100 次
- 所有代码注释使用简体中文
- 测试驱动开发：先写测试，再写实现

## 当前进度

- **已完成**: 
- **测试数量**: 
- **项目状态**: 
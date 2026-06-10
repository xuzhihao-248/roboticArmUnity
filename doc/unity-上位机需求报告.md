# Dummy 机械臂 Unity3D 上位机 — 需求报告

> 版本：v1.0 | 日期：2026-06-10 | 作者：xuzhi

---

## 一、项目背景

基于 Dummy 6 轴机械臂项目，开发一套 Unity3D 上位机软件，用于：
- 图形化控制机械臂运动（正/逆运动学）
- 实时同步物理机械臂位姿到 Unity 3D 场景
- 后续集成雷达扫描的 3D 场景环境

**与原版 DummyStudio 的关系：** 从零重新开发，参考原版通信逻辑（反编译 Assembly-CSharp.dll），UI 和场景独立设计。

---

## 二、需求分析

### 2.1 功能需求

#### P0 — 核心功能（第一阶段）

| 编号 | 功能 | 描述 |
|------|------|------|
| F-01 | 设备连接 | 通过 USB 串口连接 Controller(STM32F4)，支持连接/断开/状态显示 |
| F-02 | 关节控制 | 6 个独立滑块控制 J1~J6 关节角度，实时发送指令 |
| F-03 | 3D 模型加载 | 加载机械臂 URDF 模型，各关节可独立旋转 |
| F-04 | 正运动学显示 | 根据关节角度实时计算并显示末端位姿（XYZ + 姿态） |
| F-05 | 姿态同步 | 物理机械臂 → Unity：读取实际关节角度，更新 3D 模型姿态 |

#### P1 — 增强功能（第二阶段）

| 编号 | 功能 | 描述 |
|------|------|------|
| F-06 | 逆运动学控制 | 在 Unity 中拖拽末端执行器 → 计算关节角度 → 发送给机械臂 |
| F-07 | 笛卡尔运动 | 输入目标位姿（XYZABC），通过 move_l 指令运动 |
| F-08 | 数据监控面板 | 关节角度/电流/温度的实时数值显示（纯数值，不含曲线） |
| F-09 | 预设姿态 | 保存/加载/一键回到预设关节角度（如休息位、工作位） |

#### P2 — 扩展功能（第三阶段）

| 编号 | 功能 | 描述 |
|------|------|------|
| F-10 | 场景环境 | 导入雷达扫描的 3D 场景模型（Mesh 格式 .obj/.fbx），作为机械臂的工作环境背景 |
| F-11 | 多传输层 | 支持 WiFi/TCP 等传输方式，可在界面切换 |
| F-12 | 数据曲线 | 关节角度/电流等历史曲线图表 |
| F-13 | SolidWorks 模型导入 | 直接加载 SW 导出的 URDF（含颜色/材质） |

### 2.2 非功能需求

| 编号 | 类别 | 要求 |
|------|------|------|
| NF-01 | 性能 | 关节控制指令延迟 < 50ms，3D 模型刷新 ≥ 30fps |
| NF-02 | 可扩展 | 通信层抽象接口，新增传输方式不修改业务逻辑 |
| NF-03 | 可维护 | C# 代码分层架构（通信层 / 业务层 / 表现层） |
| NF-04 | 易用性 | 界面简洁，首次使用 5 分钟内能完成设备连接和基本控制 |

---

## 三、技术选型

| 层级 | 选型 | 理由 |
|------|------|------|
| 引擎 | Unity 2022.3 LTS | 稳定成熟，插件生态好，长期支持 |
| 渲染管线 | URP (Universal Render Pipeline) | 性能好，适合桌面应用，支持材质/PBR |
| 编程语言 | C# | Unity 原生语言 |
| IDE | VSCode + Unity 扩展 | 用户已有 VSCode，轻量 |
| 通信（一期） | USB CDC 串口（ASCII 协议） | 固件已支持，开发最快，文本协议易调试 |
| 通信（二期） | 抽象接口 + Fibre/其他 | 预留传输层接口，后续可插拔 |
| URDF 加载 | Unity URDF-Importer（官方包） | 支持 URDF 关节/连杆/碰撞体导入 |
| UI 框架 | Unity UI Toolkit 或 UGUI | 2022 LTS 两者都支持，UI Toolkit 更现代 |
| 3D 模型 | SolidWorks → URDF → Unity | 复用已有的 URDF 管线（含颜色/材质后处理） |

---

## 四、系统架构

```
┌─────────────────────────────────────────────────┐
│                   Unity 上位机                    │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │ 表现层    │  │ 业务层    │  │ 通信层         │  │
│  │          │  │          │  │               │  │
│  │ 3D场景   │←→│ 运动学    │←→│ ITransport    │  │
│  │ UI面板   │  │ 状态管理  │  │  ├ Serial     │  │
│  │ 交互控制 │  │ 指令编排  │  │  ├ TCP (预留) │  │
│  │          │  │          │  │  └ UDP (预留) │  │
│  └──────────┘  └──────────┘  └───────┬───────┘  │
│                                       │          │
└───────────────────────────────────────┼──────────┘
                                        │ USB/Serial
                                        ▼
                              ┌─────────────────┐
                              │ Controller       │
                              │ STM32F405        │
                              │ ASCII / Fibre    │
                              └────────┬────────┘
                                       │ CAN 1Mbps
                          ┌────┬────┬──┴──┬────┬────┐
                          J1   J2   J3    J4   J5   J6
                         MotorDriver-42 × 6
```

### 4.1 通信层设计

**ASCII 协议命令（一期使用）：**

| 命令 | 方向 | 功能 | 示例 |
|------|------|------|------|
| `>j1,j2,j3,j4,j5,j6` | PC→MCU | 关节运动 | `>90,0,90,0,45,0` |
| `>j1,j2,j3,j4,j5,j6,speed` | PC→MCU | 带速度的关节运动 | `>90,0,90,0,45,0,30` |
| `@x,y,z,a,b,c` | PC→MCU | 笛卡尔运动 | `@100,0,200,0,0,0` |
| `#GETJPOS` | PC→MCU | 查询关节角度 | 返回 `J:90.0,0.0,90.0,...` |
| `#GETLPOS` | PC→MCU | 查询笛卡尔位姿 | 返回 `L:100.0,0.0,200.0,...` |
| `!START` | PC→MCU | 使能所有关节 | |
| `!STOP` | PC→MCU | 急停 | |
| `!DISABLE` | PC→MCU | 禁用所有关节 | |

**抽象接口设计（预留扩展）：**

```csharp
public interface ITransport
{
    bool IsConnected { get; }
    void Connect(string address);
    void Disconnect();
    void Send(string command);
    event Action<string> OnDataReceived;
}

public class SerialTransport : ITransport { /* 一期实现 */ }
public class TcpTransport : ITransport { /* 二期预留 */ }
public class UdpTransport : ITransport { /* 二期预留 */ }
```

### 4.2 运动学方案

**双端计算策略：**
- **Unity 本地（C#）**：实现 FK/IK 算法，用于实时预览、拖拽交互、运动规划验证
- **固件端（STM32F4）**：通过 `#GETJPOS`/`#GETLPOS` 查询实际位姿，通过 `>`/`@` 命令控制实际运动
- 本地 IK 结果发送前，先在 Unity 中验证可达性（关节限位检查），再下发给固件

### 4.3 URDF 模型加载方案

1. SolidWorks → SW URDF Exporter → ROS1 URDF
2. 按已有管线转换为 ROS2 格式
3. Unity URDF-Importer 插件加载（支持 joint/link/collision）
4. 在 Unity 中手动赋予材质/颜色（PBR 材质）

---

## 五、分阶段开发计划

### 阶段一：环境 + 基础控制（约 1 周）

| 任务 | 内容 | 产出 |
|------|------|------|
| T-01 | 安装 Unity 2022.3 LTS + VSCode 扩展 | 可运行的 Unity 空项目 |
| T-02 | 安装 URDF-Importer，导入机械臂 URDF | Unity 中显示机械臂 3D 模型 |
| T-03 | 实现串口通信层（SerialTransport） | 能通过串口发送/接收 ASCII 命令 |
| T-04 | 实现关节控制 UI（6 个滑块） | 拖动滑块 → 机械臂运动 |
| T-05 | 实现姿态同步（定时查询关节角度） | 物理机械臂运动 → Unity 模型同步 |

**阶段一验收标准：** 拖动 Unity 滑块，机械臂跟随运动；手动移动机械臂，Unity 模型同步更新。

### 阶段二：运动学 + 监控（约 1 周）

| 任务 | 内容 | 产出 |
|------|------|------|
| T-06 | C# 实现正运动学（FK）| 输入关节角度 → 显示末端位姿 |
| T-07 | C# 实现逆运动学（IK）| 拖拽末端 → 计算关节角度 |
| T-08 | 末端执行器拖拽交互 | 鼠标拖拽 3D 末端 → IK 求解 → 发送指令 |
| T-09 | 数据监控面板 | 实时显示关节角度/温度等数值 |
| T-10 | 预设姿态管理 | 保存/加载/一键回到预设位 |

**阶段二验收标准：** 拖拽 Unity 中的末端执行器，机械臂跟随运动到目标位姿。

### 阶段三：场景 + 扩展（约 1 周）

| 任务 | 内容 | 产出 |
|------|------|------|
| T-11 | 雷达扫描 3D 场景导入 | 机械臂在真实场景环境中显示 |
| T-12 | 多传输层支持（TCP/WiFi） | ITransport 接口新增实现 |
| T-13 | 数据曲线图表 | 关节角度/电流历史曲线 |
| T-14 | SolidWorks 模型材质优化 | 颜色/材质/光照效果 |

**阶段三验收标准：** 机械臂在扫描的真实场景中可视化，支持 WiFi 连接。

---

## 六、环境搭建指南（T-01 详细步骤）

### 6.1 安装 Unity

1. 下载 [Unity Hub](https://unity.com/download)
2. 安装 Unity Editor **2022.3 LTS**（最新 2022.3.x）
3. 安装模块：**Windows Build Support**、**Universal RP 模板**

### 6.2 配置 VSCode

1. 安装 VSCode 扩展：
   - `Visual Studio Code for Unity`（官方扩展）
   - `C#`（ms-dotnettools.csharp）
2. 安装 [.NET 6.0 SDK](https://dotnet.microsoft.com/download/dotnet/6.0)（Unity 2022 需要）
3. Unity 中设置 External Editor：`Edit → Preferences → External Tools → Visual Studio Code`

### 6.3 创建项目

1. Unity Hub → New Project → **3D (URP)** 模板
2. 项目名：`DummyStudio`
3. 安装 URDF-Importer：`Window → Package Manager → + → Add package from git URL`
   - URL: `https://github.com/Unity-Technologies/URDF-Importer.git`

### 6.4 验证

1. VSCode 打开 Unity 项目文件夹，确认 C# 代码补全可用
2. Unity 中创建空 C# Script，双击在 VSCode 中打开，确认跳转正常

---

## 七、风险与对策

| 风险 | 影响 | 对策 |
|------|------|------|
| URDF-Importer 不支持自定义材质 | 模型显示无颜色 | 先用默认材质，后续手动赋值或改用 FBX |
| ASCII 协议查询频率受限 | 姿态同步延迟 | 控制查询频率 10~20Hz，够用即可 |
| Fibre 协议 C# 实现复杂 | 二期开发量大 | 参考 Python 实现逐步移植，或用 TCP 桥接 |
| 串口被其他程序占用 | 无法连接 | 实现连接状态检测和重连机制 |
| Unity 版本与插件不兼容 | 功能受限 | 锁定 2022.3 LTS，避免追新 |

---

## 八、已确认事项

- [x] **运动学方案：两者结合** — Unity 本地 C# 实现 FK/IK 用于预览和规划，固件端 IK 用于实际控制验证
- [x] **雷达模型格式：Mesh 模型**（.obj/.fbx），Unity 可直接导入
- [x] **通信策略：先简单后扩展** — 一期用 ASCII 串口，留好 ITransport 接口方便后续切换

### 待后续确认

- [ ] T-02：URDF-Importer 是否需要额外配置才能正确加载你的 URDF？（需要实际测试）

---

## 附录：原版 DummyStudio 参考信息

### 通信架构

- USB 双端口：CDC（ASCII 文本）+ Native（Fibre 二进制）
- 一期使用 CDC 端口，串口波特率 115200
- VID:PID = `0x1209:0x0D31`

### Fibre Endpoint 对象树（二期参考）

```
root
├── serial_number
├── get_temperature()
└── robot
    ├── homing()          → 回零
    ├── resting()         → 回休息位
    ├── move_j()          → 关节空间运动
    ├── move_l()          → 笛卡尔空间运动
    ├── set_enable()      → 使能/禁用
    ├── set_joint_speed() → 设置关节速度
    ├── joint_1 ~ joint_6
    │   ├── SetAngle()
    │   ├── SetEnable()
    │   └── SetCurrentLimit()
    └── hand
        ├── set_angle()
        └── set_enable()
```

### DH 参数

```
L_BASE=0.109m, D_BASE=0.035m, L_ARM=0.146m,
L_FOREARM=0.115m, D_ELBOW=0.052m, L_WRIST=0.072m
```

### 关节配置

| 关节 | CAN ID | 方向 | 减速比 | 角度范围 |
|------|--------|------|--------|----------|
| J1 | 1 | 正 | 50 | -170° ~ 170° |
| J2 | 2 | 反 | 30 | -73° ~ 90° |
| J3 | 3 | 正 | 30 | 35° ~ 180° |
| J4 | 4 | 反 | 24 | -180° ~ 180° |
| J5 | 5 | 正 | 30 | -120° ~ 120° |
| J6 | 6 | 正 | 50 | -720° ~ 720° |
| Hand | 7 | - | - | 0 ~ 30 |

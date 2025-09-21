# Mindustry 架构设计文档

## 概述

Mindustry采用模块化架构设计，基于Java和Arc框架构建，支持多平台部署。整体架构遵循分层设计原则，将游戏功能划分为多个相对独立的模块，便于维护、扩展和跨平台移植。

## 整体架构概览

### 分层架构
```
┌─────────────────────────────────────┐
│           应用层 (Application)       │  <- 平台特定启动器
├─────────────────────────────────────┤
│            核心层 (Core)            │  <- 游戏核心逻辑
├─────────────────────────────────────┤
│           框架层 (Arc Framework)     │  <- 图形、输入、音频等
├─────────────────────────────────────┤
│         平台抽象层 (Platform)       │  <- 跨平台抽象接口
└─────────────────────────────────────┘
```

### 模块化设计
```
core/                 # 核心游戏逻辑（跨平台）
├── entities/         # 实体组件系统
├── world/           # 世界和建筑系统
├── game/            # 游戏规则和状态
├── ui/              # 用户界面
├── net/             # 网络系统
├── logic/           # 逻辑编程系统
├── ai/              # 人工智能
├── content/         # 游戏内容定义
└── mod/             # Mod支持系统

desktop/              # 桌面平台专用代码
server/               # 无界面服务器
android/              # Android平台专用代码
ios/                  # iOS平台专用代码
tools/                # 开发工具
tests/                # 测试框架
```

## 核心架构组件

### 1. 应用程序入口层

#### 平台启动器
- **桌面版** (`desktop/src/`): 基于LWJGL的桌面启动器
- **服务器版** (`server/src/`): 无GUI的服务器启动器
- **Android版** (`android/src/`): Android Activity包装器
- **iOS版** (`ios/src/`): iOS UIViewController包装器

#### 关键文件
- `ClientLauncher.java`: 客户端启动入口
- `Vars.java`: 全局变量和配置中心

### 2. 核心控制层

#### Control模块 (`core/Control.java`)
```java
public class Control implements ApplicationListener, Loadable {
    public Saves saves;           // 存档管理
    public SoundControl sound;    // 音效控制
    public InputHandler input;    // 输入处理
    public AttackIndicators indicators; // 攻击指示器
}
```

**职责**：
- 处理所有用户输入
- 管理存档和设置
- 控制游戏状态转换
- 不处理逻辑关键状态（避免在无界面服务器中创建）

#### GameState模块 (`core/GameState.java`)
```java
public class GameState {
    public int wave;              // 当前波次
    public double tick;           // 逻辑时钟
    public boolean gameOver;      // 游戏结束标志
    public Map map;              // 当前地图
    public Rules rules;          // 游戏规则
    public Teams teams;          // 队伍数据
    private State state;         // 当前状态
}
```

**核心状态**：
- `menu`: 主菜单状态
- `playing`: 游戏进行状态
- `paused`: 暂停状态
- `settings`: 设置状态

### 3. 世界管理层

#### World模块 (`core/World.java`)
```java
public class World {
    public Tiles tiles;          // 地图瓦片数据
    public int tileChanges;      // 瓦片变化计数器
    private boolean generating;   // 生成状态标志
}
```

**核心功能**：
- 地图瓦片管理
- 碰撞检测
- 路径查找支持
- 地图生成和加载

#### Tiles系统
- **Tile**: 单个瓦片，包含地形、建筑、液体等信息
- **Building**: 瓦片上的建筑实体
- **Floor**: 地面类型（如金属地面、岩石等）

### 4. 实体组件系统 (ECS)

#### 组件设计 (`entities/comp/`)
```java
// 基础组件示例
@Component
abstract class Posc {
    public float x, y;           // 位置
}

@Component
abstract class Velc {
    public float velx, vely;     // 速度
}

@Component
abstract class Hitboxc {
    public float hitSize;        // 碰撞箱大小
}
```

#### 实体生成 (`gen/`)
```java
// 代码生成的实体类型
public class Unit extends UnitEntity {}
public class Building extends BuildingEntity {}
public class Bullet extends BulletEntity {}
```

**设计优势**：
- 组件可重用性
- 运行时代码生成优化
- 内存和性能优化

### 5. 内容管理系统

#### ContentLoader (`core/ContentLoader.java`)
```java
public class ContentLoader {
    private OrderedMap<ContentType, TypeLoader<?>> loaders;

    public void load() {
        // 按依赖顺序加载内容
        loadContent(ContentType.item);
        loadContent(ContentType.liquid);
        loadContent(ContentType.block);
        loadContent(ContentType.unit);
        // ...
    }
}
```

#### 内容类型系统
```java
public enum ContentType {
    item, liquid, block, unit, bullet,
    effect, sector, planet, weather,
    status, logic_block, objective
}
```

**内容定义示例**：
- `Items.java`: 定义所有物品
- `Blocks.java`: 定义所有建筑
- `UnitTypes.java`: 定义所有单位类型

## 跨平台架构设计

### 1. 平台抽象层

#### Platform接口
```java
public abstract class Platform {
    public abstract void showFileChooser(...)
    public abstract void openURI(String uri)
    public abstract void shareFile(Fi file)
    public abstract void updateRPC()
    // 平台特定功能抽象
}
```

#### 平台实现
- **桌面平台**: 完整功能实现
- **移动平台**: 适配触摸输入和文件系统
- **服务器平台**: 最小化实现，无UI功能

### 2. 输入系统抽象

#### InputHandler体系
```java
abstract class InputHandler {
    public abstract void buildPlacementUI(Table table)
    public abstract void drawOverlay()
    public abstract void update()
}
```

**平台特定实现**：
- `DesktopInput`: 键盘鼠标操作
- `MobileInput`: 触摸手势操作

### 3. 渲染系统抽象

基于Arc框架的渲染抽象：
- **Core.graphics**: 图形上下文抽象
- **Draw**: 2D绘制API
- **Core.camera**: 摄像机系统

## 网络架构设计

### 1. 客户端-服务器架构

```
┌─────────────┐    网络包    ┌─────────────┐
│   客户端     │ ←────────→  │   服务器     │
│(NetClient)   │             │(NetServer)   │
└─────────────┘             └─────────────┘
```

#### 核心网络组件
- `NetClient.java`: 客户端网络处理
- `NetServer.java`: 服务器网络处理
- `Packets.java`: 网络包定义
- `Administration.java`: 服务器管理

### 2. 网络包系统

#### 代码生成的包类型
```java
@Remote(called = Loc.server)
public static void sendMessage(Player player, String message) {
    // 自动生成对应的网络包
}
```

**包类型**：
- 位置同步包
- 建筑操作包
- 聊天消息包
- 游戏状态包

### 3. 同步策略

- **客户端预测**: 减少延迟感知
- **服务器权威**: 确保游戏状态一致性
- **增量同步**: 只传输变化的数据

## 性能优化架构

### 1. 内存管理

#### 对象池化 (`arc.util.pooling`)
```java
Pools.obtain(Bullet.class);  // 从池中获取
Pools.free(bullet);          // 返回到池中
```

#### 专用集合类型
- `IntSeq`: 整数序列，避免装箱
- `IntMap`: 整数映射，高性能
- `ObjectMap`: 对象映射，优化的HashMap

### 2. 渲染优化

#### 批量渲染
- **SpriteBatch**: 2D精灵批量渲染
- **ShapeRenderer**: 几何图形批量渲染
- **FrameBuffer**: 离屏渲染缓冲

#### LOD系统
- 远距离简化渲染
- 动态调整细节级别
- 视锥剔除优化

### 3. 多线程架构

#### 异步系统 (`async/`)
```java
public class AsyncCore {
    public AsyncProcess logic;     // 逻辑处理线程
    public PhysicsProcess physics; // 物理计算线程
}
```

**线程分工**：
- **主线程**: 渲染和UI
- **逻辑线程**: 游戏逻辑计算
- **物理线程**: 碰撞和运动计算
- **网络线程**: 网络I/O处理

## 可扩展性设计

### 1. Mod系统架构

#### 类加载器隔离
```java
public class ModClassLoader extends URLClassLoader {
    // 为每个Mod创建独立的类加载器
    // 支持热加载和卸载
}
```

#### 内容解析器
```java
public class ContentParser {
    // 解析JSON格式的Mod内容定义
    // 支持自定义建筑、单位、物品等
}
```

### 2. 脚本系统

#### JavaScript集成 (Rhino引擎)
- 支持JavaScript脚本编写Mod
- 沙盒环境确保安全性
- 与Java代码无缝集成

#### 逻辑编程系统
- 可视化编程界面
- 支持复杂的自动化逻辑
- 导入/导出逻辑蓝图

### 3. 地图编辑器架构

#### 编辑器组件 (`editor/`)
- **MapEditor**: 核心编辑器逻辑
- **EditorTool**: 编辑工具抽象
- **MapRenderer**: 地图渲染器
- **操作栈**: 支持撤销/重做

## 构建和部署架构

### 1. Gradle构建系统

```groovy
// 多模块项目结构
project(':core')     // 核心游戏逻辑
project(':desktop')  // 桌面版
project(':android')  // Android版
project(':server')   // 服务器版
project(':tools')    // 开发工具
```

### 2. 代码生成流程

#### 注解处理器
- `@Remote`: 生成网络包类
- `@Component`: 生成实体组件
- `@TypeIO`: 生成序列化代码

#### 资源处理
- **精灵打包**: 纹理图集生成
- **音频转换**: 跨平台音频格式
- **字体生成**: 多语言字体支持

### 3. 平台特定优化

#### Android优化
- ProGuard代码混淆
- APK大小优化
- 内存使用优化

#### 桌面优化
- JVM参数调优
- 原生库集成
- 启动速度优化

## 质量保证架构

### 1. 测试框架

#### 单元测试 (`tests/`)
```java
@Test
public void testBlockPlacement() {
    // 建筑放置逻辑测试
}
```

#### 集成测试
- 网络功能测试
- 存档兼容性测试
- 跨平台功能测试

### 2. 错误处理

#### 崩溃报告系统
```java
public class CrashHandler {
    public static void handle(Throwable e) {
        // 收集崩溃信息
        // 生成崩溃报告
        // 优雅降级处理
    }
}
```

### 3. 性能监控

#### 指标收集
- FPS监控
- 内存使用统计
- 网络延迟测量
- 加载时间分析

## 未来架构演进

### 1. 微服务化
- 游戏逻辑服务
- 匹配服务
- 统计服务
- 内容分发服务

### 2. 云原生支持
- 容器化部署
- 自动扩缩容
- 服务发现
- 配置中心

### 3. 现代化技术栈
- 响应式编程模型
- 更好的并发控制
- 函数式编程元素
- 类型安全增强

---

*本文档详细阐述了Mindustry的整体架构设计，包括模块化分层、跨平台支持、性能优化和可扩展性设计，为理解这个复杂游戏系统的技术架构提供了全面的指导。*
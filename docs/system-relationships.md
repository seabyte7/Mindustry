# Mindustry 系统关系图文档

## 概述

本文档详细描述了Mindustry各个系统之间的相互关系、数据流向和交互模式。通过理解这些关系，可以更好地把握游戏的整体架构和各系统如何协作构成完整的游戏体验。

## 1. 整体系统架构关系

### 1.1 核心系统层次图

```
┌─────────────────────────────────────────────────────────┐
│                    应用层 (Application)                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │  桌面启动    │  │  移动端启动  │  │  服务器启动  │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
└─────────────────────────────────────────────────────────┘
                           │
┌─────────────────────────────────────────────────────────┐
│                    控制层 (Control)                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   UI控制     │  │  输入处理    │  │  音频控制    │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
└─────────────────────────────────────────────────────────┘
                           │
┌─────────────────────────────────────────────────────────┐
│                    核心层 (Core)                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │  游戏状态    │  │   世界管理   │  │  逻辑处理    │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
└─────────────────────────────────────────────────────────┘
                           │
┌─────────────────────────────────────────────────────────┐
│                   内容层 (Content)                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   物品系统   │  │   建筑系统   │  │   单位系统   │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
└─────────────────────────────────────────────────────────┘
                           │
┌─────────────────────────────────────────────────────────┐
│                   基础层 (Foundation)                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │  ECS框架     │  │   网络通信   │  │   资源管理   │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### 1.2 系统交互关系网络

```
游戏状态管理 ←→ 世界管理 ←→ 实体系统
     ↕           ↕           ↕
  事件系统 ←→ 内容加载器 ←→ 渲染系统
     ↕           ↕           ↕
  输入处理 ←→ UI系统 ←→ 音频系统
     ↕           ↕           ↕
  网络同步 ←→ Mod系统 ←→ 逻辑编程
```

## 2. 核心系统间关系

### 2.1 游戏状态与世界管理

#### 状态-世界同步关系
```java
// 游戏状态影响世界更新
public class GameState {
    public void set(State newState) {
        State oldState = this.state;
        this.state = newState;

        // 状态变化触发世界响应
        Events.fire(new StateChangeEvent(oldState, newState));

        switch(newState) {
            case playing:
                world.beginPlay();      // 世界开始运行
                logic.beginPlay();      // 逻辑系统激活
                break;

            case paused:
                world.pause();          // 世界暂停
                break;

            case menu:
                world.reset();          // 重置世界状态
                break;
        }
    }
}

// 世界事件影响游戏状态
public class World {
    public void update() {
        if(state.is(State.playing)) {
            // 更新游戏世界
            updateTiles();
            updateEntities();

            // 检查游戏结束条件
            if(checkGameOver()) {
                state.gameOver = true;
                Events.fire(new GameOverEvent());
            }
        }
    }
}
```

#### 相互依赖模式
```
GameState ←────→ World
    │               │
    └─── Rules ─────┘
    │               │
    └─── Teams ─────┘
    │               │
    └─── Wave ──────┘
```

### 2.2 实体组件系统关系

#### ECS架构中的系统协作
```java
// 实体更新系统协调
public class EntitySystems {
    public void update() {
        // 1. 物理系统 - 更新位置和碰撞
        PhysicsSystem.update();

        // 2. AI系统 - 处理单位行为
        AISystem.update();

        // 3. 武器系统 - 处理射击和攻击
        WeaponSystem.update();

        // 4. 状态系统 - 更新生命值和状态效果
        StatusSystem.update();

        // 5. 渲染系统 - 绘制实体
        RenderSystem.update();

        // 6. 音频系统 - 播放音效
        AudioSystem.update();
    }
}

// 组件间依赖关系
Posc ←─ Velc ←─ Hitboxc ←─ Healthc ←─ Teamc
 │        │        │         │        │
 └────── PhysicsSystem ──────┘        │
          │        │                  │
          └─── CollisionSystem ───────┘
                   │
              DamageSystem
```

### 2.3 内容系统与游戏逻辑

#### 内容驱动的游戏机制
```java
// 内容定义影响游戏行为
public class BlockUpdate {
    public void update() {
        // 建筑类型决定更新逻辑
        switch(block.getClass()) {
            case Drill:
                updateDrilling();       // 开采逻辑
                break;
            case Crafter:
                updateCrafting();       // 制造逻辑
                break;
            case Turret:
                updateShooting();       // 射击逻辑
                break;
            case PowerNode:
                updatePower();          // 电力逻辑
                break;
        }

        // 通用更新
        updateEfficiency();
        updatePowerConsumption();
        updateItemTransfer();
    }
}

// 内容与系统的映射关系
ContentType → System
    item    → ItemSystem
    liquid  → LiquidSystem
    block   → BuildingSystem
    unit    → UnitSystem
    bullet  → BulletSystem
```

## 3. 数据流向分析

### 3.1 用户输入数据流

```
用户输入 → InputHandler → 事件系统 → 相关系统 → 游戏状态更新
    │                                               │
    └─────────── 网络同步 ────────────────────────────┘
```

#### 详细输入处理流程
```java
// 输入事件流
public class InputFlow {
    // 1. 原始输入捕获
    Core.input.isTouched() → InputHandler.update()

    // 2. 输入转换和验证
    → validateInput() → translateInput()

    // 3. 生成游戏事件
    → Events.fire(new BuildRequestEvent())

    // 4. 系统响应
    → BuildingSystem.handleBuild()

    // 5. 状态更新
    → world.setBlock() → tile.setBlock()

    // 6. 网络同步 (多人游戏)
    → netServer.sendToAll(new BlockUpdatePacket())

    // 7. 视觉反馈
    → renderer.invalidate() → ui.updateInfo()
}
```

### 3.2 游戏状态数据流

```
核心状态 → 派生状态 → UI显示
    │         │         │
    └─── 存档系统 ←───────┘
    │         │
    └─── 网络同步
```

#### 状态传播机制
```java
public class StateFlow {
    // 主状态更新
    public void updateGameState() {
        // 1. 核心状态计算
        state.tick++;
        state.wave = waveController.getCurrentWave();

        // 2. 派生状态更新
        updateTeamStates();
        updatePlayerStates();
        updateWorldStats();

        // 3. UI状态同步
        ui.hudFragment.updateWaveInfo();
        ui.hudFragment.updateResourceInfo();

        // 4. 网络状态同步
        if(net.server()) {
            netServer.syncState();
        }
    }
}
```

### 3.3 资源管理数据流

```
资源生产 → 资源存储 → 资源消费
    │         │         │
    └─── 传输系统 ←───────┘
    │         │
    └─── 统计系统
```

#### 资源流动链路
```java
public class ResourceFlow {
    // 生产链路
    Drill.mine() → ItemStack.create()
                → Conveyor.transport()
                → Storage.store()
                → Crafter.consume()
                → newItem.produce()

    // 液体链路
    Pump.extract() → LiquidStack.create()
                  → Conduit.transport()
                  → Tank.store()
                  → Consumer.use()

    // 电力链路
    Generator.produce() → PowerModule.supply()
                       → PowerNode.distribute()
                       → Consumer.consume()
}
```

## 4. 事件系统架构

### 4.1 事件传播模型

```
事件源 → EventBus → 监听器 → 处理器 → 副作用
  │                              │
  └────────── 事件链 ──────────────┘
```

#### 核心事件类型及其影响
```java
public class EventRelationships {
    // 建筑事件链
    TileChangeEvent → {
        world.tileChanges++,
        renderer.invalidateChunk(),
        pathfinder.updatePath(),
        ai.recalculateRoutes()
    }

    // 单位事件链
    UnitDestroyEvent → {
        team.unitCount--,
        spawner.updateQuota(),
        ai.rebalanceTargets(),
        fx.deathEffect()
    }

    // 玩家事件链
    PlayerJoinEvent → {
        netServer.assignTeam(),
        ui.showWelcome(),
        stats.playerCount++,
        admin.checkPermissions()
    }

    // 波次事件链
    WaveEvent → {
        spawner.spawnUnits(),
        ui.updateWaveInfo(),
        audio.playAlarm(),
        stats.waveStartTime = time
    }
}
```

### 4.2 跨系统通信

#### 系统间松耦合通信
```java
// 发布-订阅模式示例
public class SystemCommunication {
    // 生产系统通知
    @EventHandler
    public void onItemProduce(ItemProduceEvent e) {
        // 统计系统记录
        stats.itemsProduced.add(e.item, e.amount);

        // UI系统更新
        ui.notifyProduction(e.item, e.amount);

        // 成就系统检查
        achievements.checkProduction(e.item, e.amount);
    }

    // 建筑系统通知
    @EventHandler
    public void onBuildingDestroy(BuildingDestroyEvent e) {
        // 经济系统更新
        economy.refundResources(e.building);

        // 电力系统重计算
        powerSystem.recalculateGrid();

        // AI系统重新规划
        ai.replanning();
    }
}
```

## 5. 渲染系统关系

### 5.1 渲染管线

```
游戏数据 → 渲染队列 → 批处理 → GPU渲染 → 屏幕显示
    │                                      │
    └─────── 相机系统 ──────────────────────┘
    │                                      │
    └─────── UI层级 ───────────────────────┘
```

#### 渲染层级关系
```java
public class RenderLayers {
    // 渲染顺序（从底到顶）
    static final float
        floor = -1f,           // 地面层
        block = 0f,            // 建筑层
        overlay = 20f,         // 覆盖层
        blockOver = 25f,       // 建筑顶层
        turret = 30f,          // 炮塔层
        unit = 50f,            // 单位层
        flyingUnit = 60f,      // 飞行单位层
        bullet = 70f,          // 子弹层
        effect = 80f,          // 特效层
        shields = 90f,         // 护盾层
        buildBeam = 95f,       // 建造光束层
        ui = 100f;             // UI层

    // 层级依赖关系
    floor ← world.tiles
    block ← world.buildings
    unit ← Groups.unit
    bullet ← Groups.bullet
    effect ← Groups.effect
}
```

### 5.2 渲染优化关系

```
视锥剔除 → LOD系统 → 批处理 → 缓存复用
    │         │         │         │
    └─── 相机系统 ←───────┴─────────┘
```

## 6. 网络系统关系

### 6.1 客户端-服务器同步

```
客户端状态 ←→ 网络包 ←→ 服务器状态
     │                      │
     └── 预测系统 ──┐    ┌── 权威验证
     │             │    │         │
     └── 回滚系统 ──┴────┴── 冲突解决
```

#### 同步数据层次
```java
public class NetworkSync {
    // 高频同步 (每帧)
    PlayerInput, UnitMovement, ProjectileUpdate

    // 中频同步 (每秒数次)
    BuildingHealth, ResourceCounts, TeamStats

    // 低频同步 (每秒一次或更少)
    MapData, PlayerInfo, GameRules

    // 事件同步 (按需)
    BuildRequest, UnitCommand, ChatMessage
}
```

### 6.2 多人游戏状态管理

```
本地状态 → 网络层 → 远程状态
    │                    │
    └── 冲突检测 ←────────┘
    │                    │
    └── 状态协调 ←────────┘
```

## 7. 扩展系统集成

### 7.1 Mod系统与核心游戏

```
核心游戏 ←→ Mod API ←→ Mod内容
    │                    │
    └── 事件系统 ←────────┘
    │                    │
    └── 内容注册器 ←──────┘
```

#### Mod集成点
```java
public class ModIntegration {
    // 内容扩展点
    ContentLoader.load() → Mod.loadContent()
                        → Content.register()
                        → Game.updateRegistry()

    // 行为扩展点
    GameLogic.update() → Mod.updateLogic()
                      → CustomBehavior.execute()

    // 事件扩展点
    Events.fire() → Mod.handleEvent()
                 → CustomResponse.process()

    // 渲染扩展点
    Renderer.draw() → Mod.customRender()
                   → ExtraEffects.render()
}
```

### 7.2 逻辑编程系统集成

```
游戏世界 ←→ 逻辑接口 ←→ 用户脚本
    │                      │
    └── 权限检查 ←──────────┘
    │                      │
    └── 沙盒执行 ←──────────┘
```

## 8. 性能优化关系

### 8.1 系统负载均衡

```
主线程 ←→ 工作线程 ←→ 后台线程
   │         │         │
   UI更新   游戏逻辑   网络I/O
   渲染     物理计算   文件操作
   音频     AI计算     资源加载
```

#### 线程任务分配
```java
public class ThreadAllocation {
    // 主线程 (60 FPS)
    - UI渲染和交互
    - 音频播放
    - 输入处理
    - 关键游戏逻辑

    // 逻辑线程 (60 TPS)
    - 世界更新
    - 实体系统
    - 物理计算
    - AI决策

    // 网络线程 (按需)
    - 数据包处理
    - 连接管理
    - 状态同步

    // 后台线程 (低优先级)
    - 资源加载
    - 存档操作
    - 统计计算
    - 垃圾回收
}
```

### 8.2 缓存系统关系

```
热数据缓存 → 内存池 → 对象复用
     │          │         │
     └── 预计算 ←─┴────────┘
     │          │
     └── 懒加载 ←┘
```

## 9. 系统依赖图

### 9.1 启动依赖顺序

```
1. Core.init()
2. Platform.init()
3. ContentLoader.load()
4. World.init()
5. NetProvider.init()
6. UI.init()
7. Game.ready()
```

### 9.2 运行时依赖关系

```
强依赖 (必需):
GameState → World → Entities
Events → All Systems
Content → Game Logic

弱依赖 (可选):
Network → Multiplayer Features
Mods → Extended Content
Logic → Automation Features
```

### 9.3 模块耦合度分析

```
高耦合:
- GameState ↔ World
- Entities ↔ Components
- Rendering ↔ Camera

中耦合:
- UI ↔ Game Logic
- Network ↔ State Sync
- Audio ↔ Events

低耦合:
- Mods ↔ Core Game
- Logic ↔ World State
- Tools ↔ Game Systems
```

## 10. 系统演化路径

### 10.1 系统扩展模式

```
核心系统 → 接口定义 → 扩展实现
    │                    │
    └── 向后兼容 ←────────┘
    │                    │
    └── 版本管理 ←────────┘
```

### 10.2 未来架构演进

```
当前架构 → 微服务化 → 云原生
    │                    │
    └── 容器化 ←──────────┘
    │                    │
    └── 服务网格 ←────────┘
```

---

*本文档深入分析了Mindustry各系统间的复杂关系网络，揭示了这个看似简单的游戏背后精密的技术架构设计，为理解大型游戏系统的协作模式提供了宝贵的参考。*
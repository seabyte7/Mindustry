# Mindustry 核心系统文档

## 系统概述

Mindustry的核心系统构成了游戏的技术基础，包括游戏状态管理、世界与地图系统、实体组件系统、资源管理等。这些系统相互协作，为复杂的游戏机制提供可靠的技术支撑。

## 1. 游戏状态管理系统

### 1.1 核心状态机 (GameState)

#### 状态定义
```java
public enum State {
    menu,      // 主菜单状态
    playing,   // 游戏进行状态
    paused,    // 暂停状态
    settings   // 设置状态
}
```

#### 状态数据结构
```java
public class GameState {
    // 时间和波次管理
    public int wave = 1;           // 当前波次
    public float wavetime;         // 波次倒计时
    public double tick;            // 游戏逻辑时钟
    public long updateId;          // 更新计数器

    // 游戏状态标志
    public boolean gameOver = false;
    public boolean won = false;

    // 核心游戏数据
    public Map map;               // 当前地图
    public Rules rules;           // 游戏规则
    public GameStats stats;       // 游戏统计
    public Teams teams;           // 队伍数据
    public Attributes envAttrs;   // 环境属性

    private State state = State.menu; // 当前状态
}
```

#### 状态转换机制
```java
public void set(State astate) {
    if(state == astate) return;

    // 发送状态变化事件
    Events.fire(new StateChangeEvent(state, astate));
    state = astate;
}
```

### 1.2 游戏规则系统 (Rules)

#### 核心规则配置
```java
public class Rules {
    // 游戏模式控制
    public boolean infiniteResources;    // 沙盒模式
    public boolean waves;               // 波次模式
    public boolean pvp;                 // PvP模式
    public boolean editor;              // 编辑器模式
    public boolean attackMode;          // 攻击模式

    // 游戏机制开关
    public boolean waveTimer = true;    // 自动波次
    public boolean ghostBlocks = true;  // 幽灵方块
    public boolean unitAmmo = false;    // 单位弹药
    public boolean fire = true;         // 火焰机制
    public boolean coreCapture;         // 核心夺取

    // 数值调整系统
    public float unitHealthMultiplier = 1f;   // 单位血量倍数
    public float unitDamageMultiplier = 1f;   // 单位伤害倍数
    public float blockHealthMultiplier = 1f;  // 建筑血量倍数
    public float buildSpeedMultiplier = 1f;   // 建造速度倍数

    // 环境和关卡数据
    public Sector sector;               // 当前星区
    public Seq<SpawnGroup> spawns;      // 敌人生成配置
    public ObjectMap<String, String> tags; // 关卡标签
}
```

### 1.3 事件系统 (Events)

#### 事件类型定义
```java
// 核心游戏事件
public class EventType {
    public static class PlayEvent {}
    public static class StateChangeEvent {
        public State from, to;
    }
    public static class WorldLoadEvent {}
    public static class TileChangeEvent {
        public Tile tile;
    }
    public static class BuildingDamageEvent {
        public Building build;
        public float damage;
    }
}
```

#### 事件处理机制
```java
// 注册事件监听器
Events.on(PlayEvent.class, event -> {
    player.team(netServer.assignTeam(player));
    player.add();
    state.set(State.playing);
});

Events.on(WorldLoadEvent.class, event -> {
    // 重置玩家位置到核心
    Building core = player.bestCore();
    if(core != null) {
        player.set(core);
        camera.position.set(core);
    }
});
```

## 2. 世界与地图系统

### 2.1 世界管理 (World)

#### 世界数据结构
```java
public class World {
    public Tiles tiles;                    // 地图瓦片数组
    public int tileChanges = -1;          // 瓦片变化计数

    private boolean generating;            // 生成状态标志
    private boolean invalidMap;            // 无效地图标志
    private ObjectMap<Map, Runnable> customMapLoaders; // 自定义地图加载器
}
```

#### 核心功能方法
```java
// 碰撞检测
public boolean solid(int x, int y);       // 是否固体
public boolean passable(int x, int y);    // 是否可通过
public boolean wallSolid(int x, int y);   // 墙体碰撞

// 瓦片访问
public Tile tile(int x, int y);          // 获取瓦片
public Tile ltile(int x, int y);         // 获取链接瓦片
public Building build(int x, int y);     // 获取建筑

// 范围操作
public void getQuadBounds(Rect bounds);  // 获取象限边界
public Seq<Building> rect(int x, int y, int width, int height); // 范围内建筑
```

### 2.2 瓦片系统 (Tiles)

#### 瓦片数据结构
```java
public class Tile {
    // 位置信息
    public short x, y;                    // 瓦片坐标

    // 基础数据 (压缩存储)
    protected short data;                 // 压缩的瓦片数据

    // 核心组件
    private Block block;                  // 建筑类型
    private Floor floor;                  // 地面类型
    private Block overlay;                // 覆盖层

    // 状态数据
    private byte rotation;                // 旋转角度
    private byte team;                    // 所属队伍
}
```

#### 瓦片操作接口
```java
// 建筑操作
public void setBlock(Block block, Team team, int rotation);
public void setFloor(Floor floor);
public void setOverlay(Block overlay);

// 状态查询
public boolean solid();                   // 是否阻挡
public boolean passable();               // 是否可通过
public boolean interactable(Team team);  // 是否可交互

// 链接系统
public Tile linked();                    // 获取链接瓦片
public boolean isLinked();               // 是否为链接瓦片
```

### 2.3 地图管理系统

#### 地图数据结构
```java
public class Map {
    public String name;                   // 地图名称
    public String description;            // 地图描述
    public String author;                 // 作者信息

    public int width, height;             // 地图尺寸
    public StringMap tags;                // 地图标签
    public Rules rules;                   // 默认规则

    public Fi file;                       // 地图文件
    public Texture texture;               // 预览图
}
```

#### 地图加载流程
```java
public void loadMap(Map map) {
    // 1. 验证地图数据
    if(!map.valid()) throw new MapException();

    // 2. 重置世界状态
    resetState();

    // 3. 加载地图数据
    SaveIO.load(map.file);

    // 4. 应用地图规则
    state.rules = map.rules();

    // 5. 触发世界加载事件
    Events.fire(new WorldLoadEvent());
}
```

## 3. 实体组件系统 (ECS)

### 3.1 组件设计模式

#### 基础组件定义
```java
// 位置组件
@Component
public abstract class Posc {
    public float x, y;                    // 世界坐标

    public void set(float x, float y) {
        this.x = x;
        this.y = y;
    }
}

// 速度组件
@Component
public abstract class Velc {
    public float velx, vely;              // 速度向量

    public void move(float delta) {
        x += velx * delta;
        y += vely * delta;
    }
}

// 碰撞组件
@Component
public abstract class Hitboxc {
    public float hitSize = 4f;            // 碰撞半径

    public boolean within(Hitboxc other, float dst) {
        return Mathf.within(x, y, other.x(), other.y(), dst + hitSize + other.hitSize());
    }
}
```

#### 高级组件示例
```java
// 生命值组件
@Component
public abstract class Healthc {
    public float health, maxHealth = 100f;

    public void damage(float amount) {
        health -= amount;
        if(health <= 0) {
            kill();
        }
    }
}

// 队伍组件
@Component
public abstract class Teamc {
    public Team team = Team.derelict;

    public boolean isEnemyOf(Team other) {
        return team.isEnemyOf(other);
    }
}
```

### 3.2 实体生成系统

#### 代码生成机制
通过注解处理器自动生成完整的实体类：

```java
// 组件组合定义
@EntityDef(value = {Posc.class, Velc.class, Hitboxc.class, Healthc.class})
@Component
abstract class UnitComp implements Posc, Velc, Hitboxc, Healthc {
    // 单位特有的属性和方法
}

// 自动生成的实体类
public class Unit extends UnitEntity {
    // 包含所有组件的功能
    // 优化的内存布局
    // 类型安全的访问方法
}
```

#### 实体管理器
```java
public class EntityGroup<T extends Entity> {
    private Seq<T> entities = new Seq<>();

    public void add(T entity) {
        entities.add(entity);
        entity.group = this;
    }

    public void remove(T entity) {
        entities.remove(entity);
        entity.group = null;
    }

    public void update() {
        for(T entity : entities) {
            entity.update();
        }
    }
}
```

### 3.3 实体类型系统

#### 主要实体类型
```java
// 单位实体
public class Unit extends UnitEntity {
    public UnitType type;                 // 单位类型
    public UnitController controller;     // AI控制器
    public float ammo;                    // 弹药量
    public StatusEntry[] statuses;        // 状态效果
}

// 建筑实体
public class Building extends BuildingEntity {
    public Block block;                   // 建筑类型
    public Tile tile;                     // 所在瓦片
    public int rotation;                  // 旋转方向
    public float efficiency;              // 工作效率
}

// 子弹实体
public class Bullet extends BulletEntity {
    public BulletType type;               // 子弹类型
    public Weapon weapon;                 // 发射武器
    public float damage;                  // 伤害值
    public float lifetime;                // 生存时间
}
```

## 4. 资源管理系统

### 4.1 物品系统 (Items)

#### 物品数据结构
```java
public class Item extends UnlockableContent {
    // 基础属性
    public Color color = Color.black;     // 物品颜色
    public float cost = 1f;               // 建造成本权重
    public int hardness = 0;              // 开采难度

    // 特殊属性
    public float explosiveness = 0f;      // 爆炸性
    public float flammability = 0f;       // 易燃性
    public float radioactivity = 0f;      // 放射性
    public float charge = 0f;             // 导电性

    // 功能标志
    public boolean buildable = true;      // 可用于建造
    public boolean lowPriority = false;   // 低优先级
}
```

#### 物品堆栈系统
```java
public class ItemStack {
    public Item item;                     // 物品类型
    public int amount;                    // 数量

    public ItemStack(Item item, int amount) {
        this.item = item;
        this.amount = amount;
    }

    public boolean equals(ItemStack other) {
        return item == other.item && amount == other.amount;
    }
}
```

### 4.2 液体系统 (Liquids)

#### 液体属性定义
```java
public class Liquid extends UnlockableContent {
    // 视觉属性
    public Color color = Color.white;     // 液体颜色
    public TextureRegion icon;            // 图标纹理

    // 物理属性
    public float temperature = 0.5f;      // 温度 (0-1)
    public float viscosity = 0.5f;        // 粘度
    public float heatCapacity = 0.5f;     // 热容量

    // 特殊效果
    public Effect effect = Fx.none;       // 环境效果
    public StatusEffect status;           // 状态效果
}
```

#### 液体模块管理
```java
public class LiquidModule {
    private float[] liquids;              // 液体存储数组

    public void add(Liquid liquid, float amount) {
        liquids[liquid.id] += amount;
    }

    public float get(Liquid liquid) {
        return liquids[liquid.id];
    }

    public void clear() {
        Arrays.fill(liquids, 0);
    }
}
```

### 4.3 队伍系统 (Teams)

#### 队伍数据结构
```java
public class Team {
    public static final Team
        crux = new Team(1, "crux", Color.red),
        sharded = new Team(2, "sharded", Color.blue),
        derelict = new Team(0, "derelict", Color.gray);

    public final int id;                  // 队伍ID
    public final String name;             // 队伍名称
    public final Color color;             // 队伍颜色

    public boolean isEnemyOf(Team other) {
        return this != other && this != derelict && other != derelict;
    }
}
```

#### 队伍数据管理
```java
public class TeamData {
    public Team team;                     // 所属队伍
    public CoreBuild core;                // 核心建筑
    public Seq<Building> buildings;       // 所有建筑
    public Seq<Unit> units;               // 所有单位

    // 统计数据
    public int unitCount;                 // 单位数量
    public int buildingCount;             // 建筑数量
    public boolean hasCore;               // 是否有核心

    // 资源数据
    public ItemModule items;              // 物品存储
    public int unitCap;                   // 单位上限
}
```

## 5. 输入输出系统

### 5.1 存档系统 (SaveIO)

#### 存档数据结构
```java
public class SaveVersion {
    public static final int
        legacyWave = 4,
        waves = 17,
        teams = 18,
        content = 27,
        block = 132;
}
```

#### 存档流程
```java
public static void save(Fi file) {
    // 1. 创建输出流
    DataOutputStream stream = new DataOutputStream(file.write());

    // 2. 写入版本信息
    stream.writeInt(SaveVersion.block);

    // 3. 保存游戏状态
    SaveIO.writeStringMap(stream, state.map.tags);
    stream.writeInt(state.wave);

    // 4. 保存世界数据
    SaveIO.writeContentHeader(stream);
    SaveIO.writeTileData(stream, world.tiles);

    // 5. 保存实体数据
    SaveIO.writeEntities(stream);

    stream.close();
}
```

### 5.2 网络序列化

#### 类型序列化系统
```java
@TypeIOHandler
public class TypeIO {
    public static void writeBuilding(DataOutput stream, Building building) {
        stream.writeInt(building.id);
        stream.writeShort(building.x);
        stream.writeShort(building.y);
        stream.writeByte(building.rotation);
        stream.writeByte(building.team.id);
    }

    public static Building readBuilding(DataInput stream) {
        int id = stream.readInt();
        Building building = EntityGroup.all.getByID(id);
        // 读取并应用状态...
        return building;
    }
}
```

## 6. 异步处理系统

### 6.1 多线程架构

#### 异步核心
```java
public class AsyncCore {
    public AsyncProcess logic;            // 逻辑处理线程
    public PhysicsProcess physics;        // 物理计算线程

    public void begin() {
        logic.begin();
        physics.begin();
    }

    public void end() {
        logic.end();
        physics.end();
    }
}
```

#### 线程同步机制
```java
public abstract class AsyncProcess {
    protected volatile boolean running = true;
    protected final Object lock = new Object();

    public void begin() {
        synchronized(lock) {
            // 开始异步处理
        }
    }

    public void end() {
        synchronized(lock) {
            // 等待处理完成
        }
    }
}
```

### 6.2 性能优化策略

#### 对象池化
```java
public class Pools {
    private static ObjectMap<Class<?>, Pool> pools = new ObjectMap<>();

    public static <T> T obtain(Class<T> type) {
        Pool<T> pool = pools.get(type);
        if(pool == null) {
            pool = new Pool<T>();
            pools.put(type, pool);
        }
        return pool.obtain();
    }

    public static void free(Object object) {
        pools.get(object.getClass()).free(object);
    }
}
```

#### 批量处理
```java
public class BatchProcessor {
    private Seq<Runnable> tasks = new Seq<>();

    public void addTask(Runnable task) {
        tasks.add(task);
    }

    public void process() {
        for(Runnable task : tasks) {
            task.run();
        }
        tasks.clear();
    }
}
```

## 7. 调试和监控系统

### 7.1 性能监控

#### FPS和性能统计
```java
public class PerformanceCounter {
    private long frameStart;
    private float frameTime;
    private int fps;

    public void startFrame() {
        frameStart = System.nanoTime();
    }

    public void endFrame() {
        frameTime = (System.nanoTime() - frameStart) / 1_000_000f;
        fps = (int)(1000f / frameTime);
    }
}
```

### 7.2 错误处理

#### 崩溃处理器
```java
public class CrashHandler {
    public static void handle(Throwable e) {
        Log.err("Game crashed:", e);

        // 生成崩溃报告
        String report = generateCrashReport(e);

        // 保存到文件
        Fi crashFile = Core.files.local("crashes/crash-" + System.currentTimeMillis() + ".txt");
        crashFile.writeString(report);

        // 优雅退出
        Core.app.exit();
    }
}
```

---

*本文档详细介绍了Mindustry的核心系统架构，包括状态管理、世界系统、实体组件系统、资源管理等关键技术组件，为深入理解游戏的技术实现提供了全面的技术参考。*
# Mindustry 游戏玩法机制文档

## 概述

Mindustry的核心玩法融合了塔防、即时战略和工厂自动化三大机制。玩家需要建立复杂的生产链来制造防御材料，设计自动化系统维持资源供应，并运用策略性布局来抵御敌人的波次攻击。

## 1. 塔防机制

### 1.1 波次系统 (Wave System)

#### 波次生成机制
```java
public class WaveSpawner {
    // 波次配置
    public class SpawnGroup {
        public UnitType type;              // 敌人类型
        public int begin, end;             // 出现波次范围
        public float spacing;              // 生成间隔
        public int amount;                 // 数量
        public float unitScaling;          // 数量缩放
        public Point2 spawn;               // 生成点
    }

    // 计算当前波次的敌人配置
    public Seq<SpawnGroup> getSpawns(int wave) {
        return spawns.select(group ->
            wave >= group.begin &&
            (group.end < 0 || wave <= group.end)
        );
    }
}
```

#### 波次强度算法
```java
// 敌人数量随波次增长
int spawnAmount = Math.round(group.amount + (wave - group.begin) * group.unitScaling);

// 敌人血量和伤害随波次增强
float healthMultiplier = 1f + (wave - 1) * 0.1f;
float damageMultiplier = 1f + (wave - 1) * 0.05f;
```

#### 波次时间控制
```java
public class GameState {
    public float wavetime;                // 波次倒计时 (秒)

    // 自动波次：每2-5分钟一波
    public float getWaveSpacing() {
        return Math.max(2 * 60f, 5 * 60f - wave * 10f);
    }

    // 手动触发：玩家主动召唤下一波
    public void launchWave() {
        if(rules.waveSending) {
            wave++;
            wavetime = getWaveSpacing();
            spawnEnemies();
        }
    }
}
```

### 1.2 敌人AI系统

#### AI类型分类
```java
public abstract class AIController {
    // 基础地面AI
    public static class GroundAI extends AIController {
        // 寻路到最近的敌对建筑
        // 优先攻击防御设施
        // 避开危险区域
    }

    // 飞行单位AI
    public static class FlyingAI extends AIController {
        // 直线飞行到目标
        // 可跨越地形障碍
        // 优先攻击高价值目标
    }

    // 特殊任务AI
    public static class MissileAI extends AIController {
        // 锁定目标直接冲击
        // 接触后自爆
        // 高速移动，难以拦截
    }
}
```

#### 目标选择策略
```java
public Building findTarget() {
    // 1. 优先级排序
    // 核心建筑 > 发电站 > 生产设施 > 防御塔 > 其他建筑

    // 2. 距离权重
    // 距离越近，优先级越高

    // 3. 威胁评估
    // 避开高火力密度区域

    return Buildings.closestEnemyCore(team);
}
```

### 1.3 防御建筑系统

#### 炮塔分类
```java
// 基础攻击炮塔
public class Turret extends Block {
    public float range = 80f;            // 攻击范围
    public float reload = 10f;           // 装填时间
    public float inaccuracy = 0f;        // 散布范围
    public BulletType ammoTypes;         // 弹药类型
}

// 激光炮塔
public class LaserTurret extends Turret {
    public float laserLength = 160f;     // 激光长度
    public float powerUse = 10f;         // 能耗
    // 持续伤害，无弹药消耗
}

// 导弹炮塔
public class MissileTurret extends Turret {
    public float homingPower = 0.1f;     // 追踪能力
    public float homingRange = 40f;      // 追踪范围
    // 高伤害，范围爆炸
}
```

#### 弹药机制
```java
public class BulletType {
    public float damage = 1f;            // 基础伤害
    public float speed = 1f;             // 飞行速度
    public float lifetime = 60f;         // 存在时间
    public float pierce = 0f;            // 穿透力
    public float splash = 0f;            // 溅射伤害
    public float knockback = 0f;         // 击退力

    // 特殊效果
    public StatusEffect status;          // 状态效果
    public boolean incendiary;           // 燃烧效果
    public boolean explosive;            // 爆炸效果
}
```

## 2. 资源采集与生产链

### 2.1 资源开采系统

#### 钻头机制
```java
public class Drill extends Block {
    public float drillTime = 300f;       // 开采时间
    public int tier = 1;                 // 开采等级
    public float hardnessDrillMultiplier = 50f; // 硬度惩罚

    // 开采速度计算
    public float getDrillTime(Item item) {
        return drillTime + item.hardness * hardnessDrillMultiplier;
    }

    // 可开采判断
    public boolean canMine(Tile tile) {
        return tile != null &&
               tile.floor().itemDrop != null &&
               tier >= tile.floor().itemDrop.hardness;
    }
}
```

#### 矿物分布系统
```java
// 地面类型决定可开采资源
public class Floor extends Block {
    public Item itemDrop;                // 掉落物品
    public float playerUnmineable;       // 玩家开采难度
}

// 矿物丰富度计算
public int getOreCount(int x, int y, Item item) {
    int count = 0;
    for(int dx = -2; dx <= 2; dx++) {
        for(int dy = -2; dy <= 2; dy++) {
            Tile other = world.tile(x + dx, y + dy);
            if(other != null && other.floor().itemDrop == item) {
                count++;
            }
        }
    }
    return count;
}
```

### 2.2 生产制造系统

#### 制造机分类
```java
// 基础制造机
public class GenericCrafter extends Block {
    public ItemStack[] requirements;     // 输入需求
    public ItemStack outputItem;         // 输出物品
    public LiquidStack outputLiquid;     // 输出液体
    public float craftTime = 80f;        // 制造时间
    public float powerUse = 0f;          // 能耗
}

// 多配方制造机
public class MultiCrafter extends GenericCrafter {
    public Seq<Recipe> recipes;          // 配方列表

    public static class Recipe {
        public ItemStack[] input;
        public LiquidStack[] inputLiquids;
        public ItemStack output;
        public LiquidStack outputLiquid;
        public float craftTime;
    }
}
```

#### 配方系统
```java
// 基础配方示例
graphite = new GenericCrafter("graphite-press") {{
    requirements(Category.crafting, with(Items.copper, 75, Items.lead, 30));

    // 配方：煤炭 → 石墨
    craftTime = 90f;
    outputItem = new ItemStack(Items.graphite, 1);
    size = 2;

    // 能耗和效率
    hasPower = true;
    powerUse = 1.5f;
    itemCapacity = 10;
}};

// 复杂配方：硅 = 煤炭 + 石英砂
silicon = new GenericCrafter("silicon-smelter") {{
    requirements(Category.crafting, with(Items.copper, 30, Items.lead, 25));

    craftTime = 40f;
    outputItem = new ItemStack(Items.silicon, 1);

    // 多种输入材料
    consumeItems(with(Items.coal, 1, Items.sand, 2));
    consumePower(1f);
}};
```

### 2.3 传输系统

#### 传送带机制
```java
public class Conveyor extends Block {
    public float speed = 0f;             // 传输速度
    public int capacity = 4;             // 载货量

    // 物品传输逻辑
    public void updateTransport(Tile tile) {
        for(int i = 0; i < 4; i++) {
            Item item = tile.entity.items[i];
            if(item != null) {
                // 计算传输进度
                float progress = tile.entity.time / speed;
                if(progress >= 1f) {
                    // 传输到下一个瓦片
                    transferItem(tile, item);
                }
            }
        }
    }
}
```

#### 分拣器系统
```java
public class Sorter extends Block {
    public Item sortItem;                // 分拣物品类型

    public boolean accepts(Item item) {
        return sortItem == null || item == sortItem;
    }

    // 反向分拣器：排除指定物品
    public class InvertedSorter extends Sorter {
        public boolean accepts(Item item) {
            return sortItem == null || item != sortItem;
        }
    }
}
```

#### 路由器和分配器
```java
public class Router extends Block {
    // 平均分配到所有输出方向
    public void distribute(Item item) {
        Building[] outputs = getOutputs();
        int index = lastDistribution % outputs.length;
        outputs[index].handleItem(item);
        lastDistribution++;
    }
}

public class Junction extends Block {
    // 立体交叉，避免物品流冲突
    public void updateTransport() {
        // 垂直方向优先通过
        // 水平方向等待空隙
    }
}
```

## 3. 建造与自动化系统

### 3.1 建造机制

#### 建造范围系统
```java
public class BuildingRange {
    public static final float buildingRange = 220f; // 默认建造范围

    public boolean validPlace(int x, int y, Block block, Team team) {
        // 1. 检查是否在建造范围内
        if(!withinBuildRange(x, y, team)) return false;

        // 2. 检查地形兼容性
        if(!block.canPlaceOn(world.tile(x, y))) return false;

        // 3. 检查资源需求
        if(!hasRequirements(block.requirements, team)) return false;

        return true;
    }

    public boolean withinBuildRange(int x, int y, Team team) {
        return team.data().buildings.contains(b ->
            Mathf.within(b.x, b.y, x * tilesize, y * tilesize, buildingRange)
        );
    }
}
```

#### 建造成本系统
```java
public class Block {
    public ItemStack[] requirements;     // 建造需求
    public Category category;            // 建筑分类
    public float buildCost = 20f;        // 建造时间基数

    // 计算实际建造时间
    public float getBuildTime() {
        float cost = 0f;
        for(ItemStack req : requirements) {
            cost += req.amount * req.item.cost;
        }
        return cost * buildCost;
    }
}
```

### 3.2 自动化建造

#### 建造机器人
```java
public class BuilderAI extends AIController {
    public Seq<BuildPlan> queue;         // 建造队列

    public void update() {
        // 1. 寻找最近的建造任务
        BuildPlan plan = findClosestPlan();
        if(plan == null) return;

        // 2. 移动到建造位置
        if(!unit.within(plan, buildingRange)) {
            moveTo(plan.x, plan.y);
            return;
        }

        // 3. 执行建造
        if(hasRequiredItems(plan)) {
            build(plan);
        } else {
            // 4. 收集所需材料
            collectItems(plan.requirements);
        }
    }
}
```

#### 蓝图系统
```java
public class Schematic {
    public Seq<Stile> tiles;             // 建筑布局
    public ItemStack[] requirements;     // 总需求
    public int width, height;            // 尺寸

    public static class Stile {
        public Block block;              // 建筑类型
        public int x, y;                 // 相对坐标
        public int rotation;             // 旋转方向
        public Object config;            // 配置数据
    }

    // 部署蓝图
    public void place(int x, int y, Team team) {
        for(Stile stile : tiles) {
            int px = x + stile.x;
            int py = y + stile.y;

            // 添加到建造队列
            BuildPlan plan = new BuildPlan(px, py, stile.rotation, stile.block);
            team.data().plans.add(plan);
        }
    }
}
```

### 3.3 单位控制系统

#### 单位指令系统
```java
public enum UnitCommand {
    move,      // 移动指令
    attack,    // 攻击指令
    rally,     // 集结指令
    idle       // 待机指令
}

public class CommandAI extends AIController {
    public UnitCommand command = UnitCommand.idle;
    public Position target;              // 目标位置

    public void update() {
        switch(command) {
            case move:
                if(target != null) {
                    moveTo(target);
                }
                break;
            case attack:
                if(target != null) {
                    attackTarget(target);
                }
                break;
            case rally:
                rallyTo(target);
                break;
            case idle:
                // 自动寻找附近敌人
                autoTarget();
                break;
        }
    }
}
```

#### 编队系统
```java
public class Formation {
    public Seq<Unit> units;              // 编队单位
    public Position center;              // 编队中心
    public float spacing = 16f;          // 间距

    public void move(float x, float y) {
        center.set(x, y);

        // 计算每个单位的相对位置
        for(int i = 0; i < units.size; i++) {
            Unit unit = units.get(i);
            Vec2 offset = getFormationOffset(i);
            unit.moveTo(x + offset.x, y + offset.y);
        }
    }

    private Vec2 getFormationOffset(int index) {
        // 计算网格排列位置
        int cols = (int)Math.sqrt(units.size);
        int col = index % cols;
        int row = index / cols;

        return new Vec2(
            (col - cols/2f) * spacing,
            (row - cols/2f) * spacing
        );
    }
}
```

## 4. 战斗与伤害系统

### 4.1 伤害计算机制

#### 基础伤害公式
```java
public class Damage {
    public static float applyArmor(float damage, float armor) {
        // 护甲减伤公式：damage * (100 / (100 + armor))
        return damage * 100f / (100f + armor);
    }

    public static void damage(Healthc target, float damage, boolean pierce) {
        if(target.dead()) return;

        // 应用护甲减免
        if(!pierce && target instanceof Armored) {
            damage = applyArmor(damage, ((Armored) target).armor());
        }

        // 应用伤害
        target.health = Math.max(target.health - damage, 0);

        if(target.health <= 0) {
            target.kill();
        }
    }
}
```

#### 伤害类型系统
```java
public enum DamageType {
    normal,    // 普通伤害
    energy,    // 能量伤害 (忽略部分护甲)
    explosive, // 爆炸伤害 (范围伤害)
    piercing,  // 穿甲伤害 (忽略护甲)
    fire,      // 火焰伤害 (持续伤害)
    arc        // 电弧伤害 (链式传播)
}
```

### 4.2 状态效果系统

#### 状态效果定义
```java
public class StatusEffect {
    public float damage = 0f;            // 持续伤害
    public float speedMultiplier = 1f;   // 速度影响
    public float damageMultiplier = 1f;  // 伤害影响
    public float reloadMultiplier = 1f;  // 装填影响
    public Color color = Color.white;    // 效果颜色

    // 效果更新
    public void update(Unit unit, float time) {
        if(damage > 0) {
            unit.damagePierce(damage * time);
        }
    }
}

// 具体状态效果示例
public static StatusEffect
    burning = new StatusEffect("burning") {{
        damage = 0.06f;                  // 每秒6%最大血量
        color = Color.orange;
        effect = Fx.burning;
    }},

    freezing = new StatusEffect("freezing") {{
        speedMultiplier = 0.6f;          // 减速40%
        color = Color.cyan;
        effect = Fx.freezing;
    }},

    shocked = new StatusEffect("shocked") {{
        damage = 0.1f;
        effect = Fx.chainLightning;
        // 电击可传播到附近敌人
    }};
```

#### 状态效果叠加
```java
public class StatusEntry {
    public StatusEffect effect;          // 状态类型
    public float time;                   // 剩余时间

    public void add(float duration) {
        // 刷新持续时间
        time = Math.max(time, duration);
    }

    public void update(float delta) {
        time -= delta;
        if(time > 0) {
            effect.update(unit, delta);
        }
    }
}
```

### 4.3 范围效果系统

#### 爆炸机制
```java
public class Explosion {
    public static void damage(float x, float y, float radius, float damage, Team team) {
        Units.nearbyEnemies(team, x, y, radius, unit -> {
            // 计算距离衰减
            float dist = Mathf.dst(unit.x, unit.y, x, y);
            float falloff = 1f - dist / radius;

            // 应用伤害
            float finalDamage = damage * falloff;
            unit.damage(finalDamage);

            // 击退效果
            float knockback = falloff * 10f;
            unit.impulse(x, y, knockback);
        });

        // 对建筑造成伤害
        Buildings.rect(x - radius, y - radius, radius * 2, radius * 2, building -> {
            if(building.team != team) {
                float dist = Mathf.dst(building.x, building.y, x, y);
                if(dist < radius) {
                    float falloff = 1f - dist / radius;
                    building.damage(damage * falloff);
                }
            }
        });
    }
}
```

#### 链式效果
```java
public class ChainLightning {
    public void trigger(Unit source, float damage, int bounces) {
        Seq<Unit> targets = new Seq<>();
        Unit current = source;

        for(int i = 0; i < bounces; i++) {
            // 寻找最近的敌人
            Unit next = Units.closestEnemy(current.team, current.x, current.y, 80f,
                u -> !targets.contains(u));

            if(next == null) break;

            // 创建闪电效果
            Lightning.create(current.x, current.y, next.x, next.y);

            // 造成伤害
            next.damage(damage);

            targets.add(next);
            current = next;
            damage *= 0.8f; // 每次传播伤害递减
        }
    }
}
```

## 5. 环境与天气系统

### 5.1 环境属性

#### 星球环境
```java
public class Env {
    // 环境标志位
    public static final int
        terrestrial = 1,     // 陆地环境
        underwater = 2,      // 水下环境
        spores = 4,          // 孢子环境
        scorching = 8,       // 灼热环境
        groundOil = 16,      // 地面石油
        groundWater = 32,    // 地下水
        oxygen = 64;         // 含氧环境

    public static boolean has(int env, int flag) {
        return (env & flag) != 0;
    }
}

// 环境影响机制
public class EnvironmentEffects {
    public void update() {
        int env = state.rules.env;

        // 灼热环境：单位过热
        if(Env.has(env, Env.scorching)) {
            Groups.unit.each(unit -> {
                if(!unit.type.immuneScorching) {
                    unit.apply(StatusEffects.burning, 60f);
                }
            });
        }

        // 孢子环境：腐蚀效果
        if(Env.has(env, Env.spores)) {
            Groups.unit.each(unit -> {
                if(!unit.type.immuneSpores) {
                    unit.apply(StatusEffects.corroded, 60f);
                }
            });
        }
    }
}
```

### 5.2 天气系统

#### 天气类型定义
```java
public class Weather {
    public float intensity = 1f;         // 强度
    public float duration = 600f;        // 持续时间
    public Vec2 windVector;              // 风向

    // 天气效果更新
    public abstract void update(WeatherState state);
}

// 具体天气类型
public class RainWeather extends Weather {
    public void update(WeatherState state) {
        // 降雨效果：
        // 1. 熄灭火焰
        Groups.fire.each(fire -> fire.remove());

        // 2. 增加水的生成
        if(Mathf.chance(0.1f)) {
            Puddles.deposit(state.x, state.y, Liquids.water, 10f);
        }

        // 3. 减少能见度
        Renderer.fogIntensity = Math.min(1f, Renderer.fogIntensity + 0.01f);
    }
}

public class SandstormWeather extends Weather {
    public void update(WeatherState state) {
        // 沙尘暴效果：
        // 1. 降低单位移动速度
        Groups.unit.each(unit -> {
            if(!unit.isFlying()) {
                unit.apply(StatusEffects.slow, 60f);
            }
        });

        // 2. 损坏暴露的建筑
        Groups.build.each(build -> {
            if(!build.block.roofed) {
                build.damage(0.1f);
            }
        });
    }
}
```

## 6. 胜负条件系统

### 6.1 任务目标系统

#### 目标类型定义
```java
public interface Objective {
    boolean complete();                  // 是否完成
    String display();                    // 显示文本
}

// 具体目标类型
public class SurviveObjective implements Objective {
    public int waves;                    // 需要存活的波次数

    public boolean complete() {
        return state.wave >= waves;
    }
}

public class DestroyObjective implements Objective {
    public Block target;                 // 目标建筑类型
    public int amount;                   // 需要摧毁的数量
    private int destroyed = 0;

    public boolean complete() {
        return destroyed >= amount;
    }
}

public class ProduceObjective implements Objective {
    public Item item;                    // 目标物品
    public int amount;                   // 需要生产的数量

    public boolean complete() {
        return state.teams.get(player.team()).items.get(item) >= amount;
    }
}
```

### 6.2 胜负判定

#### 失败条件
```java
public class GameOverConditions {
    public void checkGameOver() {
        // 核心被摧毁
        if(!state.teams.get(player.team()).hasCore()) {
            state.gameOver = true;
            state.won = false;
            Events.fire(new GameOverEvent(player.team()));
        }

        // 生命值耗尽 (部分关卡)
        if(state.rules.limitedLives && state.stats.lives <= 0) {
            state.gameOver = true;
            state.won = false;
        }

        // 时间限制 (部分关卡)
        if(state.rules.limitedTime && state.tick > state.rules.timeLimit) {
            state.gameOver = true;
            state.won = false;
        }
    }
}
```

#### 胜利条件
```java
public class VictoryConditions {
    public void checkVictory() {
        // 攻击模式：摧毁所有敌方核心
        if(state.rules.attackMode) {
            boolean enemiesRemain = false;
            for(Team team : Team.all) {
                if(team != player.team() && state.teams.get(team).hasCore()) {
                    enemiesRemain = true;
                    break;
                }
            }

            if(!enemiesRemain) {
                state.gameOver = true;
                state.won = true;
                Events.fire(new GameOverEvent(player.team()));
            }
        }

        // 任务模式：完成所有目标
        if(state.map.objectives != null) {
            boolean allComplete = true;
            for(Objective obj : state.map.objectives) {
                if(!obj.complete()) {
                    allComplete = false;
                    break;
                }
            }

            if(allComplete) {
                state.gameOver = true;
                state.won = true;
            }
        }
    }
}
```

---

*本文档详细描述了Mindustry的核心游戏机制，包括塔防系统、资源管理、建造自动化、战斗系统等，为深入理解游戏的玩法设计提供了全面的参考。*
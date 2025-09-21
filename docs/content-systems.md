# Mindustry 内容系统文档

## 概述

Mindustry的内容系统是游戏的核心组成部分，包含了物品、液体、建筑、单位、科技树等各种游戏内容。该系统采用数据驱动的设计，支持高度的模块化和可扩展性，为Mod制作提供了强大的基础。

## 1. 内容类型系统

### 1.1 内容类型枚举

```java
public enum ContentType {
    item,          // 物品
    liquid,        // 液体
    block,         // 建筑/方块
    unit,          // 单位
    bullet,        // 子弹
    effect,        // 特效
    sector,        // 星区
    planet,        // 星球
    weather,       // 天气
    status,        // 状态效果
    logic_block,   // 逻辑块
    objective      // 任务目标
}
```

### 1.2 内容基类架构

```java
// 内容基类
public abstract class Content {
    public int id = -1;                  // 内容ID
    public String name;                  // 内容名称
    public ContentType contentType;      // 内容类型

    public abstract void load();         // 加载方法
    public String localizedName() {      // 本地化名称
        return Core.bundle.get(getContentType() + "." + name + ".name", name);
    }
}

// 可解锁内容
public abstract class UnlockableContent extends MappableContent {
    public ItemStack[] researchRequirements = {}; // 研究需求
    public TechNode techNode;            // 科技树节点
    public boolean alwaysUnlocked = false; // 始终解锁

    public boolean unlocked() {          // 是否已解锁
        return alwaysUnlocked || techNode != null && techNode.content.unlocked();
    }
}
```

## 2. 物品系统 (Items)

### 2.1 物品基础属性

```java
public class Item extends UnlockableContent {
    // 视觉属性
    public Color color = Color.black;    // 物品颜色
    public String emoji = "";            // 表情符号

    // 基础属性
    public float cost = 1f;              // 建造成本权重
    public int hardness = 0;             // 开采硬度等级
    public boolean buildable = true;     // 可用于建造
    public boolean lowPriority = false;  // 低传输优先级

    // 特殊属性
    public float explosiveness = 0f;     // 爆炸性 (0-1)
    public float flammability = 0f;      // 易燃性 (0-1)
    public float radioactivity = 0f;     // 放射性 (0-1)
    public float charge = 0f;            // 导电性 (0-1)
    public float healthScaling = 0f;     // 血量加成

    // 特殊标志
    public boolean isLiquid = false;     // 是否为液态
    public boolean hidden = false;       // 是否隐藏
}
```

### 2.2 Serpulo星球物品

#### 基础资源
```java
public static Item
// 基础金属
copper = new Item("copper", Color.valueOf("d99d73")) {{
    hardness = 1;
    cost = 0.5f;
    alwaysUnlocked = true;           // 游戏开始即可用
}},

lead = new Item("lead", Color.valueOf("8c7fa9")) {{
    hardness = 1;
    cost = 0.7f;
}},

// 基础材料
sand = new Item("sand", Color.valueOf("f7cba4")) {{
    lowPriority = true;              // 低传输优先级
    buildable = false;               // 不用于建造
    alwaysUnlocked = true;
}},

coal = new Item("coal", Color.valueOf("272727")) {{
    explosiveness = 0.2f;            // 具有爆炸性
    flammability = 1f;               // 高度易燃
    hardness = 2;
    buildable = false;
}};
```

#### 合金材料
```java
// 加工材料
graphite = new Item("graphite", Color.valueOf("b2c6d2")) {{
    cost = 1f;
}},

silicon = new Item("silicon", Color.valueOf("53565c")) {{
    cost = 0.8f;
}},

metaglass = new Item("metaglass", Color.valueOf("ebeef5")) {{
    cost = 1.5f;
}},

// 高级合金
plastanium = new Item("plastanium", Color.valueOf("cbd97f")) {{
    flammability = 0.1f;
    explosiveness = 0.2f;
    cost = 1.3f;
    healthScaling = 0.1f;            // 提供10%血量加成
}},

surgeAlloy = new Item("surge-alloy", Color.valueOf("f3e979")) {{
    cost = 1.2f;
    charge = 0.75f;                  // 高导电性
    healthScaling = 0.25f;
}},

phaseFabric = new Item("phase-fabric", Color.valueOf("f4ba6e")) {{
    cost = 1.3f;
    radioactivity = 0.6f;            // 具有放射性
    healthScaling = 0.25f;
}};
```

### 2.3 Erekir星球物品

#### 新时代材料
```java
// Erekir基础材料
beryllium = new Item("beryllium", Color.valueOf("b8c6e8")) {{
    hardness = 1;
    cost = 0.5f;
}},

tungsten = new Item("tungsten", Color.valueOf("767a84")) {{
    hardness = 4;
    cost = 1.1f;
}},

// 高科技材料
oxide = new Item("oxide", Color.valueOf("ff5845")) {{
    flammability = 0.3f;
    explosiveness = 0.2f;
    cost = 1.2f;
}},

carbide = new Item("carbide", Color.valueOf("c8c8d2")) {{
    cost = 1.4f;
    healthScaling = 0.15f;
}},

fissileMatter = new Item("fissile-matter", Color.valueOf("ffaa5f")) {{
    explosiveness = 0.7f;            // 高爆炸性
    radioactivity = 1f;              // 强放射性
    cost = 1.5f;
    healthScaling = 0.4f;
}};
```

### 2.4 物品分组系统

```java
// 物品分组
public static final Seq<Item>
    serpuloItems = new Seq<>(),      // Serpulo星球物品
    erekirItems = new Seq<>(),       // Erekir星球物品
    erekirOnlyItems = new Seq<>();   // Erekir独有物品

// 自动分组逻辑
static {
    for(Item item : content.items()) {
        if(item.unlocked()) {
            serpuloItems.add(item);
        }
        // 根据科技树判断归属星球
    }
}
```

## 3. 液体系统 (Liquids)

### 3.1 液体基础属性

```java
public class Liquid extends UnlockableContent {
    // 视觉属性
    public Color color = Color.white;    // 液体颜色
    public TextureRegion fullIcon;       // 完整图标
    public String emoji = "";            // 表情符号

    // 物理属性
    public float temperature = 0.5f;     // 温度 (0-1)
    public float viscosity = 0.5f;       // 粘度
    public float heatCapacity = 0.5f;    // 热容量
    public float coolant = 1f;          // 冷却效果

    // 特殊效果
    public Effect effect = Fx.none;      // 环境特效
    public StatusEffect status = StatusEffects.none; // 状态效果
    public float statusDuration = 60f;   // 状态持续时间

    // 功能标志
    public boolean gas = false;          // 是否为气体
    public boolean hidden = false;       // 是否隐藏
}
```

### 3.2 基础液体类型

```java
public static Liquid
// 基础液体
water = new Liquid("water", Color.valueOf("596ab8")) {{
    temperature = 0.25f;             // 低温
    effect = Fx.wet;
    coolant = 1.2f;                  // 优秀的冷却剂
}},

// 工业液体
slag = new Liquid("slag", Color.valueOf("ffa166")) {{
    temperature = 1f;                // 高温
    viscosity = 0.8f;               // 高粘度
    effect = Fx.burning;
    status = StatusEffects.melting;  // 熔化状态
    statusDuration = 240f;
}},

oil = new Liquid("oil", Color.valueOf("313131")) {{
    viscosity = 0.7f;
    flammability = 0.7f;            // 易燃
    explosiveness = 0.2f;           // 轻微爆炸性
}},

// 高科技液体
cryofluid = new Liquid("cryofluid", Color.valueOf("87ceeb")) {{
    temperature = 0f;                // 极低温
    coolant = 2f;                   // 超强冷却
    status = StatusEffects.freezing; // 冰冻效果
}};
```

### 3.3 液体传输系统

```java
public class LiquidModule {
    private float[] liquids;             // 液体存储数组
    private float total;                 // 总液体量

    // 添加液体
    public void add(Liquid liquid, float amount) {
        if(amount <= 0) return;

        liquids[liquid.id] += amount;
        total += amount;

        // 检查容量限制
        if(total > capacity) {
            handleOverflow();
        }
    }

    // 移除液体
    public float remove(Liquid liquid, float amount) {
        float removed = Math.min(liquids[liquid.id], amount);
        liquids[liquid.id] -= removed;
        total -= removed;
        return removed;
    }

    // 液体混合机制
    public void mix(Liquid liquid, float amount) {
        if(liquids[liquid.id] == 0 && total > 0) {
            // 不同液体混合：清空容器或产生反应
            if(liquid.reactsWith(getCurrentLiquid())) {
                triggerReaction(liquid, getCurrentLiquid());
            } else {
                clear(); // 清空不兼容液体
            }
        }
        add(liquid, amount);
    }
}
```

## 4. 建筑系统 (Blocks)

### 4.1 建筑基类架构

```java
public class Block extends UnlockableContent {
    // 基础属性
    public int size = 1;                 // 建筑尺寸 (NxN)
    public float health = 40f;           // 生命值
    public int armor = 0;                // 护甲值
    public boolean solid = false;        // 是否阻挡移动
    public boolean destructible = true;  // 是否可被摧毁

    // 建造属性
    public Category category;            // 建筑分类
    public ItemStack[] requirements = {}; // 建造需求
    public float buildCostMultiplier = 1f; // 建造成本倍数

    // 功能属性
    public boolean hasItems = false;     // 是否存储物品
    public boolean hasLiquids = false;   // 是否存储液体
    public boolean hasPower = false;     // 是否使用电力
    public int itemCapacity = 10;        // 物品容量
    public float liquidCapacity = 10f;   // 液体容量

    // 视觉属性
    public TextureRegion[] regions;      // 纹理区域
    public Color mapColor = Color.black; // 地图颜色

    // 环境适应性
    public int envEnabled = Env.terrestrial; // 适用环境
    public int envDisabled = 0;          // 禁用环境
}
```

### 4.2 建筑分类系统

```java
public enum Category {
    // 基础设施
    distribution(Color.valueOf("4a4b53")),  // 传输
    liquid(Color.valueOf("52a5cc")),        // 液体
    power(Color.valueOf("f9a825")),         // 电力
    production(Color.valueOf("8da1e3")),    // 生产

    // 军事设施
    defense(Color.valueOf("d4816b")),       // 防御
    turret(Color.valueOf("b73239")),        // 炮塔
    units(Color.valueOf("89e8b6")),         // 单位

    // 特殊设施
    effect(Color.valueOf("ea8878")),        // 特效
    crafting(Color.valueOf("8176c1")),      // 合成
    logic(Color.valueOf("a457ce"));         // 逻辑

    public final Color color;               // 分类颜色
}
```

### 4.3 生产建筑

#### 基础钻头
```java
public class Drill extends Block {
    public float drillTime = 300f;       // 开采时间 (ticks)
    public int tier = 1;                 // 开采等级
    public float hardnessDrillMultiplier = 50f; // 硬度惩罚

    // 开采效率计算
    public float getDrillTime(Item item) {
        return drillTime + item.hardness * hardnessDrillMultiplier;
    }

    // 开采范围内矿物统计
    public int countOre(Tile tile) {
        ItemCounter ores = new ItemCounter();
        for(int dx = -size/2; dx < size/2; dx++) {
            for(int dy = -size/2; dy < size/2; dy++) {
                Tile other = world.tile(tile.x + dx, tile.y + dy);
                if(other != null && other.floor().itemDrop != null) {
                    ores.add(other.floor().itemDrop, 1);
                }
            }
        }
        return ores.total();
    }
}

// 具体钻头实现
mechanicalDrill = new Drill("mechanical-drill") {{
    requirements(Category.production, with(Items.copper, 12));
    drillTime = 600f;
    size = 2;
    tier = 1; // 只能开采硬度1的矿物
}};

laserDrill = new Drill("laser-drill") {{
    requirements(Category.production, with(Items.copper, 35, Items.graphite, 30, Items.silicon, 30, Items.titanium, 20));
    drillTime = 280f;
    size = 3;
    tier = 2; // 可以开采硬度2的矿物
    hasPower = true;
    powerUse = 1.5f;
}};
```

#### 制造机
```java
public class GenericCrafter extends Block {
    public float craftTime = 80f;        // 制造时间
    public ItemStack outputItem;         // 输出物品
    public LiquidStack outputLiquid;     // 输出液体
    public Effect craftEffect = Fx.none; // 制造特效

    // 制造进度更新
    public void updateCrafting(Building build) {
        if(hasRequiredItems(build) && hasRequiredPower(build)) {
            build.progress += 1f / craftTime;

            if(build.progress >= 1f) {
                // 完成制造
                consume(build);
                produce(build);
                build.progress = 0f;

                // 播放特效
                if(craftEffect != Fx.none) {
                    craftEffect.at(build.x, build.y);
                }
            }
        }
    }
}

// 具体制造机示例
graphitePress = new GenericCrafter("graphite-press") {{
    requirements(Category.crafting, with(Items.copper, 75, Items.lead, 30));

    craftTime = 90f;
    outputItem = new ItemStack(Items.graphite, 1);
    size = 2;

    hasPower = true;
    powerUse = 1.5f;
    hasItems = true;
    itemCapacity = 10;

    consumeItems(with(Items.coal, 2));
    craftEffect = Fx.pulverizeMedium;
}};
```

### 4.4 防御建筑

#### 墙体系统
```java
public class Wall extends Block {
    public float chanceDeflect = -1f;    // 偏转几率
    public boolean deflectSound = true;   // 偏转音效

    // 伤害吸收
    public void damage(Building build, float damage) {
        // 计算偏转
        if(chanceDeflect > 0 && Mathf.chance(chanceDeflect)) {
            if(deflectSound) {
                Sounds.spark.at(build.tile, Mathf.random(0.9f, 1.1f));
            }
            return; // 完全偏转
        }

        // 正常伤害
        super.damage(build, damage);
    }
}

// 具体墙体类型
copperWall = new Wall("copper-wall") {{
    requirements(Category.defense, with(Items.copper, 6));
    health = 80 * 4; // 4倍基础血量
}};

titaniumWall = new Wall("titanium-wall") {{
    requirements(Category.defense, with(Items.titanium, 6));
    health = 110 * 4;
    armor = 2; // 护甲值
}};

phaseWall = new Wall("phase-wall") {{
    requirements(Category.defense, with(Items.phaseFabric, 6));
    health = 150 * 4;
    armor = 10;
    chanceDeflect = 0.1f; // 10%偏转几率
}};
```

#### 炮塔系统
```java
public class Turret extends Block {
    public float range = 80f;            // 攻击范围
    public float reload = 10f;           // 装填时间
    public float inaccuracy = 0f;        // 散布角度
    public float velocityInaccuracy = 0f; // 速度偏差
    public int shots = 1;                // 每次射击数
    public float spread = 4f;            // 多发散布

    // 弹药系统
    public ObjectMap<Item, BulletType> ammoTypes = new ObjectMap<>();

    // 目标选择
    public Unit findTarget(Building build) {
        return Units.closestTarget(build.team, build.x, build.y, range,
            u -> u.checkTarget(targetAir, targetGround),
            b -> b.team != build.team && targetGround
        );
    }

    // 射击逻辑
    public void updateShooting(Building build) {
        Unit target = findTarget(build);
        if(target == null) return;

        // 预测目标位置
        Vec2 intercept = Predict.intercept(build, target, getBulletSpeed());

        // 旋转炮塔
        float targetRot = build.angleTo(intercept.x, intercept.y);
        build.rotation = Angles.moveToward(build.rotation, targetRot, rotateSpeed);

        // 检查射击条件
        if(Angles.within(build.rotation, targetRot, shootCone) &&
           build.efficiency() > 0.5f && build.timer(timerShoot, reload)) {

            shoot(build, target);
        }
    }
}
```

### 4.5 单位建筑

#### 单位工厂
```java
public class UnitFactory extends Block {
    public UnitType produceUnit;         // 生产的单位类型
    public float produceTime = 1000f;    // 生产时间
    public float speedupRate = 0f;       // 加速倍率

    // 生产逻辑
    public void updateProduction(Building build) {
        if(hasRequiredItems(build) && hasRequiredPower(build)) {
            build.progress += 1f / produceTime * build.edelta();

            // 加速效果
            if(speedupRate > 0) {
                build.progress += speedupRate * build.efficiency();
            }

            if(build.progress >= 1f) {
                Unit unit = produceUnit.create(build.team);
                unit.set(build.x, build.y);
                unit.add();

                consume(build);
                build.progress = 0f;

                Events.fire(new UnitCreateEvent(unit, build));
            }
        }
    }
}

// 具体工厂示例
groundFactory = new UnitFactory("ground-factory") {{
    requirements(Category.units, with(Items.copper, 50, Items.lead, 120, Items.silicon, 80));
    plans = Seq.with(
        new UnitPlan(UnitTypes.dagger, 60f * 15f, with(Items.silicon, 10, Items.lead, 10)),
        new UnitPlan(UnitTypes.mace, 60f * 25f, with(Items.silicon, 15, Items.lead, 20, Items.graphite, 10))
    );
    size = 3;
    produceTime = 900f;
    maxBlockSize = 2; // 最大可建造单位尺寸
}};
```

## 5. 单位系统 (Units)

### 5.1 单位类型架构

```java
public class UnitType extends UnlockableContent {
    // 基础属性
    public float health = 100f;          // 生命值
    public float armor = 0f;             // 护甲值
    public float speed = 1.1f;           // 移动速度
    public float rotateSpeed = 5f;       // 旋转速度
    public float accel = 0.5f;           // 加速度
    public float drag = 0.3f;            // 阻力

    // 物理属性
    public float hitSize = 6f;           // 碰撞大小
    public boolean flying = false;       // 是否飞行
    public float groundLayer = Layer.groundUnit; // 渲染层级

    // 武器系统
    public Seq<Weapon> weapons = new Seq<>();

    // AI控制
    public Prov<? extends UnitController> controller = () -> new AIController();
    public Prov<? extends UnitController> defaultController = controller;

    // 能力系统
    public Seq<Ability> abilities = new Seq<>();

    // 环境适应
    public int envEnabled = Env.terrestrial;
    public int envDisabled = 0;
    public boolean canDrown = true;      // 是否会溺水
}
```

### 5.2 Serpulo单位体系

#### 地面单位
```java
// T1 基础单位
dagger = new UnitType("dagger") {{
    health = 130f;
    speed = 0.75f;
    weapons.add(new Weapon("dagger-weapon") {{
        reload = 15f;
        x = 4f;
        y = 2f;
        top = false;
        ejectEffect = Fx.shellEjectSmall;
        bullet = Bullets.standardCopper;
    }});
}};

// T2 中级单位
mace = new UnitType("mace") {{
    health = 350f;
    speed = 0.65f;
    armor = 3f;
    weapons.add(new Weapon("mace-weapon") {{
        reload = 35f;
        x = 5f;
        y = 0f;
        bullet = Bullets.flakGlass;
    }});
}};

// T3 高级单位
fortress = new UnitType("fortress") {{
    health = 750f;
    speed = 0.56f;
    armor = 8f;
    weapons.add(new Weapon("fortress-weapon") {{
        reload = 60f;
        x = 9f;
        y = 1f;
        bullet = Bullets.standardThorium;
    }});
}};
```

#### 空中单位
```java
// T1 侦察机
flare = new UnitType("flare") {{
    flying = true;
    health = 70f;
    speed = 2.8f;
    weapons.add(new Weapon() {{
        reload = 17f;
        x = 2f;
        y = 0f;
        bullet = Bullets.standardCopper;
    }});
}};

// T2 战斗机
horizon = new UnitType("horizon") {{
    flying = true;
    health = 170f;
    speed = 2.2f;
    armor = 1f;
    weapons.add(new Weapon() {{
        reload = 20f;
        x = 4f;
        y = 0f;
        bullet = Bullets.standardSilicon;
    }});
}};
```

#### 海军单位
```java
// T1 巡逻艇
risso = new UnitType("risso") {{
    health = 280f;
    speed = 1.1f;
    armor = 2f;
    envEnabled = Env.underwater;
    weapons.add(new Weapon("risso-weapon") {{
        reload = 30f;
        x = 3f;
        y = 0f;
        bullet = Bullets.waterShot;
    }});
}};
```

### 5.3 Erekir单位体系

#### 新世代单位
```java
// 攻击型单位
stell = new UnitType("stell") {{
    health = 150f;
    speed = 1.8f;
    envEnabled = Env.terrestrial;
    envDisabled = Env.underwater;

    weapons.add(new Weapon("stell-weapon") {{
        reload = 24f;
        x = 0f;
        y = 0f;
        bullet = new BasicBulletType(2.5f, 12f) {{
            lifetime = 35f;
            trailColor = Color.white;
            trailLength = 3;
        }};
    }});
}};

// 支援型单位
locus = new UnitType("locus") {{
    health = 600f;
    speed = 1.4f;
    armor = 4f;

    // 治疗能力
    abilities.add(new RepairFieldAbility(15f, 60f * 2, 30f));

    weapons.add(new Weapon("locus-weapon") {{
        reload = 45f;
        x = 5f;
        y = 0f;
        bullet = new HealBulletType(5.5f, 13) {{
            healPercent = 3f;
            collidesTeam = true;
        }};
    }});
}};
```

### 5.4 AI控制系统

#### AI类型分类
```java
// 地面AI
public class GroundAI extends AIController {
    public void update() {
        // 1. 寻找目标
        Teamc target = target(unit.x, unit.y, unit.range(), unit.team.enemies().first(), false);

        // 2. 移动控制
        if(target != null) {
            // 攻击移动
            moveTo(target, unit.type.range * 0.8f);
            unit.lookAt(target);
        } else {
            // 巡逻移动
            patrolMove();
        }

        // 3. 战斗控制
        if(unit.canShoot()) {
            unit.aimLook(target);
        }
    }
}

// 飞行AI
public class FlyingAI extends AIController {
    public void update() {
        // 飞行单位可以直接飞向目标
        Teamc target = target(unit.x, unit.y, unit.range(), unit.team.enemies().first(), true);

        if(target != null) {
            // 保持攻击距离
            circle(target, unit.type.range * 0.8f);
            unit.lookAt(target);
        }
    }
}

// 建造AI
public class BuilderAI extends AIController {
    public Seq<BuildPlan> plans = new Seq<>();

    public void update() {
        // 1. 寻找建造任务
        BuildPlan plan = findPlan();

        if(plan != null) {
            // 2. 移动到建造位置
            if(!unit.within(plan, unit.type.buildRange)) {
                moveTo(plan.x * tilesize, plan.y * tilesize);
                return;
            }

            // 3. 执行建造
            if(unit.canBuild()) {
                unit.addBuild(plan);
            }
        }
    }
}
```

## 6. 科技树系统

### 6.1 科技树架构

```java
public class TechTree {
    public static Seq<TechNode> all = new Seq<>();     // 所有节点
    public static Seq<TechNode> roots = new Seq<>();   // 根节点

    public static class TechNode {
        public UnlockableContent content;    // 解锁内容
        public ItemStack[] requirements;     // 研究需求
        public Seq<Objective> objectives;    // 额外目标
        public TechNode parent;             // 父节点
        public Seq<TechNode> children;      // 子节点
        public Planet planet;               // 所属星球

        // 研究状态
        public boolean researched() {
            return content.unlocked();
        }

        public boolean canResearch() {
            return parent == null || parent.researched();
        }
    }
}
```

### 6.2 Serpulo科技线

```java
public class SerpuloTechTree {
    public static void load() {
        // 基础材料科技线
        nodeRoot("serpulo", Blocks.coreShard, () -> {
            // 铜科技线
            node(Items.copper, () -> {
                node(Blocks.mechanicalDrill, () -> {
                    node(Blocks.mechanicalPump, () -> {
                        node(Liquids.water);
                    });
                });

                node(Blocks.conveyor, () -> {
                    node(Blocks.junction, () -> {
                        node(Blocks.router);
                    });
                });

                node(Blocks.copperWall, () -> {
                    node(Blocks.copperWallLarge);
                });
            });

            // 铅科技线
            node(Items.lead, () -> {
                node(Blocks.duo, () -> {
                    node(Blocks.scatter, () -> {
                        node(Blocks.scorch);
                    });
                });

                node(Blocks.powerNode, () -> {
                    node(Blocks.combustionGenerator, () -> {
                        node(Blocks.steamGenerator);
                    });
                });
            });

            // 高级材料
            node(Items.graphite, Seq.with(new Produce(Items.graphite)), () -> {
                node(Blocks.graphitePress, () -> {
                    node(Blocks.kiln, () -> {
                        node(Items.metaglass, Seq.with(new Produce(Items.metaglass)));
                    });
                });
            });
        });
    }
}
```

### 6.3 Erekir科技线

```java
public class ErekirTechTree {
    public static void load() {
        nodeRoot("erekir", Blocks.coreBastion, true, () -> {
            // 铍科技线
            node(Items.beryllium, () -> {
                node(Blocks.turboDrill, () -> {
                    node(Blocks.beamDrill);
                });

                node(Blocks.duct, () -> {
                    node(Blocks.ductRouter, () -> {
                        node(Blocks.ductBridge);
                    });
                });

                node(Blocks.breach, () -> {
                    node(Blocks.berylliumWall, () -> {
                        node(Blocks.berylliumWallLarge);
                    });
                });
            });

            // 钨科技线
            node(Items.tungsten, () -> {
                node(Blocks.impactDrill, () -> {
                    node(Blocks.largePlasmaBore);
                });

                node(Blocks.diffuse, () -> {
                    node(Blocks.sublimate, () -> {
                        node(Blocks.titan);
                    });
                });

                node(Blocks.tungstenWall, () -> {
                    node(Blocks.tungstenWallLarge, () -> {
                        node(Blocks.blastDoor);
                    });
                });
            });
        });
    }
}
```

### 6.4 研究机制

```java
public class Research {
    // 研究进度数据
    public static class ResearchProgress {
        public TechNode node;
        public ItemStack[] progress;     // 当前进度
        public boolean completed;        // 是否完成

        public float getProgress() {
            if(completed) return 1f;

            float total = 0f, current = 0f;
            for(int i = 0; i < node.requirements.length; i++) {
                ItemStack req = node.requirements[i];
                ItemStack prog = progress[i];

                total += req.amount;
                current += Math.min(prog.amount, req.amount);
            }

            return current / total;
        }
    }

    // 投入研究资源
    public static void research(TechNode node, ItemStack[] items) {
        if(!node.canResearch()) return;

        ResearchProgress progress = getProgress(node);

        // 投入物品
        for(ItemStack stack : items) {
            for(int i = 0; i < node.requirements.length; i++) {
                ItemStack req = node.requirements[i];
                if(req.item == stack.item) {
                    ItemStack prog = progress.progress[i];
                    int added = Math.min(stack.amount, req.amount - prog.amount);
                    prog.amount += added;
                    stack.amount -= added;
                }
            }
        }

        // 检查完成条件
        if(progress.getProgress() >= 1f && checkObjectives(node)) {
            completeResearch(node);
        }
    }

    // 完成研究
    public static void completeResearch(TechNode node) {
        node.content.unlock();

        // 触发解锁事件
        Events.fire(new ResearchEvent(node.content));

        // 播放解锁效果
        Fx.research.at(Core.camera.position.x, Core.camera.position.y);
    }
}
```

---

*本文档全面描述了Mindustry的内容系统，包括物品、液体、建筑、单位和科技树等核心内容类型，展示了游戏丰富的内容生态和精心设计的进度系统。*
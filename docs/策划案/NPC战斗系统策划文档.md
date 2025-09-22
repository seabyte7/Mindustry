# Mindustry NPC战斗系统策划文档

> **文档版本**：v1.0
> **创建时间**：2025-09-22
> **文档类型**：系统策划文档
> **适用范围**：Mindustry游戏NPC敌对单位系统

---

## 1. 项目概述

### 1.1 游戏定位
Mindustry是一款融合塔防与RTS元素的策略游戏，玩家需要建造防御设施、管理资源，同时指挥单位对抗不断来袭的敌军波次。

### 1.2 NPC战斗系统的核心价值
- **挑战性递增**：通过波次系统为玩家提供持续的挑战
- **策略深度**：多样化的敌军类型要求不同的应对策略
- **长期可玩性**：动态难度调整确保游戏持续有趣
- **战术丰富性**：复杂的AI行为和技能系统提供丰富的战斗体验

### 1.3 系统目标
1. 创造有挑战性但公平的敌军AI
2. 提供多样化的敌军类型和战术
3. 保持良好的性能表现
4. 支持模组扩展和自定义

---

## 2. 系统架构设计

### 2.1 整体架构

```mermaid
graph TB
    A[游戏核心 Game Core] --> B[波次管理器 WaveSpawner]
    A --> C[AI控制器 AIController]
    A --> D[单位系统 UnitSystem]
    A --> E[路径寻找 Pathfinder]

    B --> F[生成组 SpawnGroup]
    B --> G[难度调整 Difficulty]

    C --> H[目标选择 Target Selection]
    C --> I[移动控制 Movement]
    C --> J[战斗控制 Combat]

    D --> K[生命系统 Health]
    D --> L[武器系统 Weapons]
    D --> M[能力系统 Abilities]

    E --> N[流场寻路 Flowfield]
    E --> O[A*寻路 AStar]
```

### 2.2 核心模块说明

| 模块名称 | 核心类 | 主要功能 |
|---------|-------|---------|
| 波次管理 | `WaveSpawner.java` | 敌军生成、波次控制、难度调整 |
| AI控制 | `AIController.java` | 行为决策、目标选择、路径规划 |
| 单位系统 | `UnitComp.java` | 生命值、移动、战斗、技能管理 |
| 路径寻找 | `Pathfinder.java` | 地形分析、路径计算、导航 |
| 伤害系统 | `Damage.java` | 伤害计算、护甲减免、效果应用 |

---

## 3. AI行为系统设计

### 3.1 AI架构继承体系

```
UnitController (接口)
├── AIController (基类)
    ├── FlyingAI (飞行单位AI)
    ├── GroundAI (地面单位AI)
    │   └── SuicideAI (自杀式AI)
    ├── DefenderAI (防守AI)
    ├── MissileAI (导弹AI)
    ├── CommandAI (指挥AI)
    ├── BuilderAI (建造AI)
    ├── MinerAI (挖矿AI)
    ├── RepairAI (修复AI)
    └── CargoAI (货运AI)
```

### 3.2 AI行为类型详细设计

#### 3.2.1 FlyingAI（飞行单位AI）

**设计理念**：灵活机动，多目标攻击，环绕战术

**核心参数**：
- 搜索范围：无地形限制
- 环绕距离：120像素
- 攻击距离：射程的80%

**目标优先级**：
```java
static BlockFlag[] randomTargets = {
    core,       // 核心设施 (最高优先级)
    storage,    // 储存设施
    generator,  // 发电设施
    launchPad,  // 发射台
    factory,    // 工厂
    repair,     // 修复设施
    battery,    // 电池
    reactor,    // 反应堆
    drill       // 钻头
};
```

**行为逻辑**：
1. **随机目标模式**：30%概率切换目标类型
2. **环绕攻击**：`circleAttack(120f)` - 保持安全距离攻击
3. **直接突进**：对核心目标采用直接攻击模式

#### 3.2.2 GroundAI（地面单位AI）

**设计理念**：直接突进，专注核心，集群作战

**核心机制**：
- 主要目标：敌方核心
- 移动方式：路径寻找 `pathfind(Pathfinder.fieldCore)`
- 攻击判定：距离核心 `unit.range() / 1.3f + core.block.size * tilesize / 2f` 内

**决策流程**：
```
开始 → 检查核心距离 →
├─ 在攻击范围内 → 攻击核心
└─ 超出攻击范围 → 向核心移动
```

#### 3.2.3 DefenderAI（防守AI）

**设计理念**：区域防守，目标优选，威胁评估

**目标选择算法**：
```java
priority = -u.maxHealth + Mathf.dst2(u.x, u.y, tx, ty) / 6400f
```

**算法解析**：
- 优先攻击高血量单位（威胁更大）
- 距离近的单位优先级更高
- 使用归一化距离平衡血量和距离因素

#### 3.2.4 SuicideAI（自杀式AI）

**设计理念**：冲锋自爆，破坏建筑，路径直达

**核心机制**：
- 射线检测：`World.raycast` 检测路径障碍
- 阻挡攻击：遇到建筑物立即开火
- 目标过滤：避免攻击传送带和管道

**行为决策**：
```java
if(射程内有目标) {
    直接攻击();
} else if(路径被阻挡) {
    攻击阻挡建筑();
} else {
    继续前进();
}
```

#### 3.2.5 MissileAI（导弹AI）

**设计理念**：制导追踪，延迟激活，精确打击

**制导机制**：
- 延迟激活：`unit.type.homingDelay` 后开始制导
- 加速公式：`Mathf.pow(Math.min(time / unit.type.missileAccelTime, 1f), 2f) * unit.speed()`
- 重定位频率：每4秒重新选择目标

### 3.3 目标选择系统

#### 3.3.1 目标优先级算法

**主要函数**：`closestTarget()`, `bestTarget()`, `closestEnemy()`

**优先级计算**：
```java
float dst2 = e.dst2(x, y) - (e.hitSize * e.hitSize);
if(dst2 < range*range &&
   (result == null || dst2 < cdist || e.type.targetPriority > cpriority) &&
   e.type.targetPriority >= cpriority) {
    result = e;
    cdist = dst2;
    cpriority = e.type.targetPriority;
}
```

#### 3.3.2 目标排序策略

| 排序类型 | 计算公式 | 适用场景 |
|---------|---------|---------|
| closest | `Unit::dst2` | 快速清理附近威胁 |
| farthest | `(u, x, y) -> -u.dst2(x, y)` | 远程火力支援 |
| strongest | `-u.maxHealth + Mathf.dst2(u.x, u.y, x, y) / 6400f` | 防御型AI |
| weakest | `u.maxHealth + Mathf.dst2(u.x, u.y, x, y) / 6400f` | 快速清场 |

---

## 4. 单位系统设计

### 4.1 ECS组件架构

Mindustry采用Entity-Component-System架构，每个单位由多个组件组合而成：

```
Unit (实体)
├── UnitComp (单位组件)
├── HealthComp (生命组件)
├── WeaponsComp (武器组件)
├── ShieldComp (护盾组件)
├── VelComp (速度组件)
├── PhysicsComp (物理组件)
└── AbilityComp (能力组件)
```

### 4.2 移动类型分类

#### 4.2.1 移动类型特性表

| 移动类型 | 地形适应 | 速度特性 | 代表单位 | 战术定位 |
|---------|---------|---------|---------|---------|
| **Mech (机甲)** | 陆地 | 中等速度 | Dagger, Scepter | 主力步兵 |
| **Legs (腿部)** | 跨越障碍 | 适应性强 | Atrax, Corvus | 特种渗透 |
| **Flying (飞行)** | 无视地形 | 高速度 | Flare, Eclipse | 空中支援 |
| **Naval (海军)** | 水面专用 | 中低速度 | Risso, Sei | 海战控制 |
| **Tank (坦克)** | 陆地重型 | 低速高甲 | Stell, Conquer | 重装突击 |
| **Hover (悬浮)** | 低空悬浮 | 中等速度 | Elude | 多地形适应 |
| **Crawl (爬行)** | 特殊移动 | 多变速度 | 特殊单位 | 特化功能 |

#### 4.2.2 速度计算公式

```java
最终速度 = 基础速度 × 扫射惩罚 × 推进器倍数 × 地面倍数 × 状态效果倍数

其中：
扫射惩罚 = isGrounded() || !isPlayer() ? 1f :
          Mathf.lerp(1f, type.strafePenalty, Angles.angleDist(vel().angle(), rotation) / 180f)

推进器倍数 = Mathf.lerp(1f, type.canBoost ? type.boostMultiplier : 1f, elevation)
```

### 4.3 生命系统设计

#### 4.3.1 生命值管理

**核心属性**：
- `health`: 当前生命值
- `maxHealth`: 最大生命值
- `healthMultiplier`: 生命倍数（受状态效果影响）
- `armor`: 护甲值
- `shield`: 护盾值

**生命百分比计算**：
```java
float healthf() {
    return health / maxHealth;
}
```

#### 4.3.2 护盾系统机制

**护盾伤害吸收**：
```java
protected void rawDamage(float amount) {
    boolean hadShields = shield > 0.0001f;

    // 护盾优先吸收伤害
    float shieldDamage = Math.min(Math.max(shield, 0), amount);
    shield -= shieldDamage;
    amount -= shieldDamage;

    // 剩余伤害作用于生命值
    if(amount > 0 && type.killable) {
        health -= amount;
        if(health <= 0 && !dead) {
            kill();
        }
    }
}
```

---

## 5. 战斗机制设计

### 5.1 伤害计算系统

#### 5.1.1 护甲减伤公式

**核心公式**：
```java
最终伤害 = max(原始伤害 - 护甲值, 原始伤害 × 最小护甲伤害比例)

具体实现：
public static float applyArmor(float damage, float armor) {
    return Math.max(damage - armor, minArmorDamage * damage);
}

其中：minArmorDamage = 0.1f  // 保证至少10%伤害穿透
```

#### 5.1.2 综合伤害处理流程

```java
public void damage(float amount) {
    // 1. 应用护甲减伤
    float armorValue = armorOverride >= 0f ? armorOverride : armor;
    float reducedDamage = Damage.applyArmor(amount, armorValue);

    // 2. 应用健康倍数和游戏规则倍数
    float finalDamage = reducedDamage / healthMultiplier / Vars.state.rules.unitHealth(team);

    // 3. 调用实际伤害处理
    rawDamage(finalDamage);
}
```

#### 5.1.3 范围伤害衰减

**爆炸伤害衰减公式**：
```java
private static float calculateDamage(float dist, float radius, float damage) {
    float falloff = 0.4f;  // 衰减系数
    float scaled = Mathf.lerp(1f - dist / radius, 1f, falloff);
    return damage * scaled;
}
```

**衰减特性**：
- 中心点：100%伤害
- 边缘点：60%伤害
- 衰减方式：线性衰减

### 5.2 武器系统设计

#### 5.2.1 武器核心属性

| 属性名称 | 数据类型 | 说明 | 典型数值 |
|---------|---------|------|---------|
| reload | float | 重装时间（帧） | 13-120 |
| inaccuracy | float | 不准确度（度） | 0-45 |
| range | float | 射程 | 60-300 |
| damage | float | 伤害值 | 9-80 |
| rotate | boolean | 是否可旋转 | true/false |
| mirror | boolean | 是否镜像（双管） | true/false |

#### 5.2.2 DPS计算公式

```java
DPS = (子弹伤害 ÷ 重装时间) × 每次射击子弹数 × 60

具体实现：
public float dps() {
    return (bullet.estimateDPS() / reload) * shoot.shots * 60f;
}

射击频率：
public float shotsPerSec() {
    return shoot.shots * 60f / reload;
}
```

#### 5.2.3 弹药容量自动计算

```java
// 基于武器射速自动计算弹药容量
if (ammoCapacity < 0) {
    float shotsPerSecond = weapons.sumf(w -> w.useAmmo ? 60f / w.reload : 0f);
    float targetSeconds = 35;  // 目标持续射击时间
    ammoCapacity = Math.max(1, (int)(shotsPerSecond * targetSeconds));
}
```

### 5.3 穿透机制

#### 5.3.1 穿透参数

- **pierce**: 是否可穿透单位
- **pierceBuilding**: 是否可穿透建筑
- **pierceCap**: 最大穿透数量
- **pierceDamageFactor**: 每次穿透的伤害衰减

#### 5.3.2 穿透伤害处理

```java
public void handlePierce(Bullet b, float initialHealth, float x, float y) {
    float sub = Mathf.zero(pierceDamageFactor) ? 0f :
                Math.max(initialHealth * pierceDamageFactor, 0);
    b.damage -= Float.isNaN(sub) ? b.damage : Math.min(b.damage, sub);

    if(removeAfterPierce && b.damage <= 0) {
        b.hit = true;
        b.remove();
    }
}
```

---

## 6. NPC种类设计

### 6.1 单位分类体系

#### 6.1.1 按战术角色分类

| 战术角色 | 特征 | 代表单位 | 设计目的 |
|---------|------|---------|---------|
| **基础步兵** | 大量、低成本、快速生产 | Dagger, Crawler | 提供基础压力 |
| **重装单位** | 高血量、高护甲、低速度 | Fortress, Reign | 正面突破 |
| **空中单位** | 高机动、多目标、低血量 | Flare, Zenith | 骚扰和侦察 |
| **特种单位** | 特殊能力、战术价值 | Atrax, Toxopid | 破局和变数 |
| **支援单位** | 提供增益、治疗、护盾 | Nova, Vela | 团队协作 |
| **BOSS单位** | 超高属性、多技能 | Scepter, Corvus | 终极挑战 |

#### 6.1.2 典型NPC单位详细分析

##### **Dagger（短剑）- 基础步兵**

**设计定位**：新手友好的基础敌军，教学型单位

**属性配置**：
```
生命值：150 HP
移动速度：0.5f
攻击范围：60 单位
护甲：0
武器：基础弹道武器
  - 伤害：9
  - 射速：13帧
  - 子弹速度：2.5f
```

**战术价值**：
- 大量生产提供基础压力
- 教导玩家基础防御机制
- 后期作为数量型威胁

##### **Mace（狼牙棒）- 火焰步兵**

**设计定位**：近战高伤，建筑克星

**属性配置**：
```
生命值：550 HP
移动速度：0.5f
护甲：4
武器：火焰喷射器
  - 伤害：37
  - 射程：短距离
  - 特效：燃烧状态（4分钟）
  - 穿透：true
```

**特殊机制**：
- 免疫燃烧状态
- 对建筑造成持续伤害
- 近战高威胁

##### **Zenith（天顶）- 导弹轰炸机**

**设计定位**：空中火力支援，多目标攻击

**属性配置**：
```
生命值：700 HP
移动速度：1.7f
护甲：5
武器：制导导弹系统
  - 追踪范围：60
  - 爆炸伤害：25
  - 爆炸范围：15
  - 双发齐射：shots = 2
```

**目标优先级**：发射台 > 储存设施 > 电池 > 其他建筑

##### **Scepter（权杖）- 重型支援BOSS**

**设计定位**：中期BOSS，支援型领袖

**属性配置**：
```
生命值：9000 HP
移动速度：0.36f
护甲：10
特殊能力：
  - 护盾再生力场：范围250，再生25HP/s
  - 单目标锁定：singleTarget = true
```

**武器配置**：
- 主武器：3连发高爆弹 + 闪电伤害
- 副武器：2门辅助机枪

**战术意义**：
- 提供团队护盾支援
- 高生存能力
- 需要集中火力消灭

##### **Toxopid（毒蛛）- 终极BOSS**

**设计定位**：终极挑战，多阶段BOSS

**属性配置**：
```
生命值：超高（具体数值待平衡测试）
移动类型：腿部单位
特殊能力：
  - 多条腿移动系统
  - 毒液攻击
  - 范围伤害技能
  - 召唤小型单位
```

**生成条件**：第210波开始，每35波生成，1000基础护盾

### 6.2 单位克制关系设计

#### 6.2.1 克制关系表

| 攻击方 | 被克制目标 | 克制原因 | 对抗方法 |
|-------|-----------|---------|---------|
| **高爆武器** | 轻甲群体单位 | 范围伤害 | 分散部署 |
| **穿透武器** | 直线排列单位 | 穿透特性 | 避免直线移动 |
| **火焰武器** | 建筑物 | 持续燃烧 | 修复设施 |
| **防空武器** | 飞行单位 | 针对性设计 | 地面掩护 |
| **激光武器** | 高血量单位 | 持续伤害 | 快速接近 |

#### 6.2.2 数值平衡原则

1. **岩石剪刀布原则**：每种单位都有明确的克制和被克制关系
2. **成本效益平衡**：强力单位需要更高的生产成本（时间/资源）
3. **多元化应对**：避免单一最优解，鼓励多样化防御
4. **渐进式引入**：新单位类型逐步引入，避免学习曲线过陡

---

## 7. 波次系统设计

### 7.1 波次生成算法

#### 7.1.1 SpawnGroup核心计算

**单位数量计算**：
```java
int spawned = Math.min(
    unitAmount + (int)(((wave - begin) / spacing) / unitScaling),
    max
);

参数说明：
- unitAmount: 初始数量
- wave: 当前波次
- begin: 开始波次
- spacing: 波次间隔
- unitScaling: 缩放系数
- max: 最大数量
```

**护盾计算**：
```java
float shields = Math.max(shields + shieldScaling * (wave - begin), 0);
```

**生成条件判定**：
```java
if(wave < begin || wave > end || (wave - begin) % spacing != 0) {
    return 0; // 不生成
}
```

#### 7.1.2 难度系数应用

**战役模式难度调整**：
```java
if(state.isCampaign()) {
    spawned = Math.max(1, group.effect == StatusEffects.boss ?
              (int)(spawned * enemySpawnMultiplier) :
              Mathf.round(spawned * enemySpawnMultiplier));
}
```

### 7.2 动态波次配置

#### 7.2.1 基础波次进程

**第1-10波：教学阶段**
```java
// Dagger单位进程
SpawnGroup dagger_basic = new SpawnGroup(UnitTypes.dagger){{
    begin = 1;           // 第1波开始
    spacing = 1;         // 每波都有
    unitAmount = 1;      // 初始1个
    unitScaling = 2f;    // 每波增加2个
    max = 30;           // 最大30个
}};
```

**第4-13波：引入自爆单位**
```java
SpawnGroup crawler_wave = new SpawnGroup(UnitTypes.crawler){{
    begin = 4;
    end = 13;
    spacing = 2;         // 每2波生成
    unitAmount = 2;      // 每次2个
    unitScaling = 1.5f;
}};
```

**第12波+：空中威胁**
```java
SpawnGroup flare_air = new SpawnGroup(UnitTypes.flare){{
    begin = 12;
    spacing = 3;
    unitAmount = 1;
    unitScaling = 1f;
    spawn = -1;          // 从地图边缘生成
}};
```

#### 7.2.2 BOSS波次机制

**中期BOSS（Scepter）**：
```java
SpawnGroup scepter_boss = new SpawnGroup(UnitTypes.scepter){{
    begin = 41;
    spacing = 30;        // 每30波一次
    unitAmount = 1;
    shields = 30;        // 基础护盾
    shieldScaling = 30;  // 每波增加30护盾
    effect = StatusEffects.boss;
}};
```

**终极BOSS（Toxopid）**：
```java
SpawnGroup toxopid_final = new SpawnGroup(UnitTypes.toxopid){{
    begin = 210;
    spacing = 35;
    unitAmount = 1;
    shields = 1000;      // 高基础护盾
    shieldScaling = 35;
}};
```

### 7.3 动态难度调整

#### 7.3.1 物种分类系统

```java
UnitType[][] species = {
    {dagger, mace, fortress, scepter, reign},      // 地面攻击
    {nova, pulsar, quasar, vela, corvus},          // 地面支援
    {crawler, atrax, spiroct, arkyid, toxopid},    // 蜘蛛系
    {risso, minke, bryde, sei, omura},             // 海军单位
    {risso, oxynoe, cyerce, aegires, navanax},     // 海军支援
    {flare, horizon, zenith, antumbra, eclipse}    // 空军单位
};
```

#### 7.3.2 难度递增曲线

**幂次曲线调整**：
```java
// 应用幂次曲线使起始区域更容易
return generate(Mathf.pow(difficulty, 1.12f), new Rand(), false);
```

**护盾递增系统**：
```java
float shieldStart = 30;
float shieldsPerWave = 20 + difficulty * 30f;
float shieldAmount = Math.max((wave - shieldStart) * shieldsPerWave, 0);
```

**BOSS生成频率**：
```java
int bossWave = (int)(rand.random(50, 70) * Mathf.lerp(1f, 0.5f, difficulty));
int bossSpacing = (int)(rand.random(25, 40) * Mathf.lerp(1f, 0.5f, difficulty));
```

### 7.4 生成位置策略

#### 7.4.1 地面单位生成

**固定生成点模式**：
- 扫描地图中overlay为spawn的tile
- 避开玩家核心周围的安全区域
- 支持多点同时生成

**攻击模式生成**：
- 从敌方核心出发计算位置
- 使用3格半径避障检测
- 最多30步寻找合适位置

#### 7.4.2 飞行单位生成

**边缘生成算法**：
```java
// 从地图边缘按角度生成
float flyRange = range + 80f;
float angle = Angles.angle(spawnX, spawnY, core.x, core.y);
Tmp.v1.trns(angle, flyRange);
```

### 7.5 波次控制机制

#### 7.5.1 时间管理

**基础波次间隔**：
```java
public float waveSpacing = 2 * Time.toMinutes; // 2分钟基础间隔
```

**动态时间调整**：
```java
state.wavetime = state.rules.waveSpacing *
                state.getPlanet().campaignRules.difficulty.waveTimeMultiplier;
```

#### 7.5.2 波次暂停条件

```java
public boolean isWaitingWave() {
    return (state.rules.waitEnemies ||
           (state.wave >= state.rules.winWave && state.rules.winWave > 0)) &&
           state.enemies > 0;
}
```

---

## 8. 路径寻找和导航系统

### 8.1 三层路径寻找架构

#### 8.1.1 系统层次结构

```
路径寻找系统
├── 流场寻路 (Flowfield) - 大规模单位移动
├── 分层A* (Hierarchical A*) - 复杂地形导航
└── 传统A* (Traditional A*) - 精确路径计算
```

#### 8.1.2 适用场景

| 寻路类型 | 适用场景 | 性能特点 | 精度 |
|---------|---------|---------|------|
| **流场寻路** | 大量单位向同一目标移动 | 高性能，多线程 | 中等 |
| **分层A*** | 复杂地形，长距离路径 | 中等性能 | 高 |
| **传统A*** | 精确路径，短距离移动 | 低性能 | 最高 |

### 8.2 流场寻路系统

#### 8.2.1 地形成本计算

**地面单位成本函数**：
```java
(team, tile) ->
    (PathTile.allDeep(tile) ||
     ((PathTile.team(tile) == team && !PathTile.teamPassable(tile)) ||
       PathTile.team(tile) == 0) && PathTile.solid(tile)) ? impassable :
    1 +                                    // 基础成本
    PathTile.health(tile) * 5 +           // 建筑血量影响
    (PathTile.nearSolid(tile) ? 2 : 0) +  // 靠近固体地形
    (PathTile.nearLiquid(tile) ? 6 : 0) + // 靠近液体地形
    (PathTile.deep(tile) ? 6000 : 0) +    // 深水区域
    (PathTile.damages(tile) ? 30 : 0)     // 伤害地形
```

**海军单位成本函数**：
```java
(team, tile) ->
    (!PathTile.liquid(tile) ? 6000 : 1) + // 必须在液体上
    PathTile.health(tile) * 5 +
    (PathTile.nearGround(tile) || PathTile.nearSolid(tile) ? 14 : 0) +
    (PathTile.deep(tile) ? 0 : 1) +       // 深水更适合
    (PathTile.damages(tile) ? 35 : 0)
```

#### 8.2.2 流场更新算法

```java
private void updateFrontier(Flowfield path, long nsToRun) {
    while(path.frontier.size > 0) {
        int tile = path.frontier.removeLast();
        int cost = path.weights[tile];

        // 检查4个方向的邻居
        for(Point2 point : Geometry.d4) {
            int newPos = tile + point.x + point.y * wwidth;
            int otherCost = path.cost.getCost(path.team.id, tiles[newPos]);

            if((path.weights[newPos] > cost + otherCost ||
                path.searches[newPos] < path.search) &&
               otherCost != impassable) {

                path.frontier.addFirst(newPos);
                path.weights[newPos] = cost + otherCost;
                path.searches[newPos] = (short)path.search;
            }
        }
    }
}
```

### 8.3 分层A*寻路系统

#### 8.3.1 集群分割机制

**集群大小**：12x12像素块
**门户系统**：集群间的连接点自动检测

**集群更新算法**：
```java
Cluster updateCluster(int team, int pathCost, int cx, int cy) {
    // 扫描4个方向寻找门户
    for(int direction = 0; direction < 4; direction++) {
        for(int i = 0; i < clusterSize; i++) {
            int x = baseX + addX * i, y = baseY + addY * i;

            if(solid(team, cost, x, y) ||
               solid(team, cost, nextBaseX + addX * i, nextBaseY + addY * i)) {
                // 遇到墙壁，创建门户
                if(!prevSolid && previous >= lastPortal) {
                    portals.add(Point2.pack(previous, lastPortal));
                }
                prevSolid = true;
            } else {
                // 空地区域，标记门户开始
                if(prevSolid) {
                    lastPortal = i;
                }
                prevSolid = false;
            }
        }
    }
}
```

### 8.4 性能优化策略

#### 8.4.1 多线程优化

**异步计算**：
- 路径计算在独立线程进行
- 主线程不阻塞
- 使用线程池管理

**时间分片**：
```java
// 限制每帧计算时间
static final int maxUpdate = 8; // 8毫秒上限
```

#### 8.4.2 缓存机制

**路径缓存**：
- 相同起点终点的路径结果缓存
- LRU淘汰策略
- 地形变化时失效

**增量更新**：
- 只更新变化的地形区域
- 局部路径重计算
- 全局路径保持

---

## 9. 技能和能力系统

### 9.1 能力系统架构

#### 9.1.1 基础架构

```java
abstract class Ability {
    public float data;  // 网络同步数据

    public void update(Unit unit) {}     // 每帧更新
    public void draw(Unit unit) {}       // 绘制效果
    public void death(Unit unit) {}      // 死亡时触发
    public void init(UnitType type) {}   // 初始化
}
```

#### 9.1.2 集成机制

在UnitComp中：
```java
public class UnitComp {
    public Ability[] abilities = {};

    @Override
    public void update() {
        // 更新所有能力
        for(Ability ability : abilities) {
            ability.update(self());
        }
    }
}
```

### 9.2 护盾技能系统

#### 9.2.1 力场护盾 (ForceFieldAbility)

**设计理念**：360度全方位防护，可被持续攻击破坏

**核心参数**：
```java
float radius = 60f;          // 护盾半径
float regen = 0.1f;         // 每帧再生速度 (6点/秒)
float max = 200f;           // 最大护盾值
float cooldown = 60f * 5;   // 破盾冷却时间 (5秒)
int sides = 6;              // 多边形边数
```

**碰撞检测**：
```java
if(Intersector.isInRegularPolygon(sides, rx, ry, radius, rotation, bullet.x, bullet.y)) {
    // 子弹被护盾拦截
    shieldConsumer.get(bullet);
    return true;
}
```

#### 9.2.2 弧形护盾 (ShieldArcAbility)

**设计理念**：方向性防护，适合近战单位

**核心机制**：
- 扇形区域护盾
- 可配置角度和偏移
- 支持射击时激活模式

**角度计算**：
```java
float angleTo = Angles.angle(unit.x, unit.y, bullet.x, bullet.y);
if(Angles.within(angleTo, unit.rotation + angleOffset, angle)) {
    // 在护盾保护范围内
}
```

### 9.3 治疗技能系统

#### 9.3.1 修复场 (RepairFieldAbility)

**设计理念**：团队支援，范围治疗

**工作机制**：
```java
Units.nearby(unit.team, unit.x, unit.y, range, other -> {
    if(other.damaged()) {
        healEffect.at(other, parentizeEffects);
        wasHealed = true;
    }
    other.heal(amount);
});
```

**参数配置**：
```java
float amount = 24f;         // 每次治疗量
float reload = 100f;        // 冷却时间（帧）
float range = 50f;          // 治疗范围
```

#### 9.3.2 液体再生 (LiquidRegenAbility)

**设计理念**：环境交互，资源消耗型治疗

**机制**：
- 从地面液体中吸取能量
- 消耗液体恢复生命
- 治疗量与消耗量成正比

### 9.4 召唤技能系统

#### 9.4.1 单位生成 (UnitSpawnAbility)

**设计理念**：战术召唤，增加战场复杂度

**生成逻辑**：
```java
if(timer >= spawnTime && Units.canCreate(unit.team, this.unit)) {
    Unit u = this.unit.create(unit.team);
    u.set(x, y);
    u.rotation = unit.rotation;
    Events.fire(new UnitCreateEvent(u, null, unit));
}
```

**限制机制**：
- 检查单位数量上限
- 考虑队伍单位容量
- 避免无限生成

#### 9.4.2 死亡召唤 (SpawnDeathAbility)

**设计理念**：死亡报复，延续威胁

**实现细节**：
```java
int spawned = amount + Mathf.random(randAmount);  // 随机数量
for(int i = 0; i < spawned; i++) {
    Tmp.v1.rnd(spread);  // 随机位置
    Unit u = unit.create(unit.team);
    u.set(unit.x + Tmp.v1.x, unit.y + Tmp.v1.y);
}
```

### 9.5 场域控制技能

#### 9.5.1 状态场 (StatusFieldAbility)

**设计理念**：区域控制，状态效果施加

**作用机制**：
- 范围内友军获得增益
- 可选择射击时激活
- 支持多种状态效果

#### 9.5.2 抑制场 (SuppressionFieldAbility)

**设计理念**：武器抑制，战术控制

**抑制机制**：
```java
Units.nearbyEnemies(unit.team, unit.x, unit.y, range, other -> {
    if(other.hasWeapons()) {
        for(WeaponMount mount : other.mounts) {
            mount.reload = Math.max(mount.reload, reload);
        }
    }
});
```

### 9.6 技能平衡设计

#### 9.6.1 冷却时间管理

**标准冷却公式**：
```java
timer += Time.delta;
if(timer >= reload) {
    // 技能激活
    timer = 0f;
}
```

**平衡原则**：
- 强力技能更长冷却
- 被动技能持续生效
- 主动技能瞬间爆发

#### 9.6.2 资源消耗设计

**能量消耗**：部分技能消耗单位能量
**生命消耗**：自损型技能机制
**弹药消耗**：射击类技能消耗弹药

---

## 10. 数值平衡设计

### 10.1 核心数值表

#### 10.1.1 基础单位数值表

| 单位名称 | 生命值 | 护甲 | 速度 | 射程 | DPS | 生产时间 | 定位 |
|---------|-------|------|------|------|-----|---------|------|
| **Dagger** | 150 | 0 | 0.5 | 60 | 41.5 | 快速 | 基础步兵 |
| **Mace** | 550 | 4 | 0.5 | 35 | 67.2 | 中等 | 火焰兵 |
| **Fortress** | 900 | 8 | 0.43 | 65 | 58.8 | 较慢 | 重装步兵 |
| **Scepter** | 9000 | 10 | 0.36 | 100 | 85.5 | 很慢 | 支援BOSS |
| **Reign** | 24000 | 18 | 0.4 | 120 | 156.0 | 超慢 | 攻击BOSS |

#### 10.1.2 平衡性指标

**生存能力指数** = 生命值 × (1 + 护甲/10)
**机动性指数** = 速度 × (1 + 射程/100)
**威胁等级** = DPS × 生存能力指数 / 生产时间

### 10.2 波次难度曲线

#### 10.2.1 难度递增公式

**基础波次强度**：
```
波次强度 = 基础强度 × (1 + 波次数 × 0.05)^1.2
```

**护盾递增**：
```
护盾值 = max(0, (波次 - 30) × (20 + 难度 × 30))
```

**BOSS出现频率**：
```
BOSS间隔 = 基础间隔 × (0.5 + 0.5 × (1 - 难度))
```

#### 10.2.2 资源平衡

**单位成本效益**：
- 低级单位：高性价比，数量优势
- 高级单位：低性价比，质量优势
- BOSS单位：特殊价值，关键时刻

### 10.3 武器平衡设计

#### 10.3.1 武器类型平衡

| 武器类型 | 伤害特点 | 射程特点 | 射速特点 | 适用目标 |
|---------|---------|---------|---------|---------|
| **子弹武器** | 中等伤害 | 中等射程 | 高射速 | 通用目标 |
| **激光武器** | 持续伤害 | 长射程 | 连续射击 | 高血量目标 |
| **导弹武器** | 高爆伤害 | 超长射程 | 低射速 | 重要目标 |
| **火焰武器** | 范围伤害 | 短射程 | 中射速 | 群体/建筑 |

#### 10.3.2 克制关系平衡

**装甲克制**：
```
实际伤害 = max(原伤害 - 装甲, 原伤害 × 0.1)
```

**范围伤害衰减**：
```
边缘伤害 = 中心伤害 × 0.6
```

### 10.4 长期平衡策略

#### 10.4.1 动态调整机制

**胜率监控**：
- 统计各波次的平均胜率
- 识别过难或过易的波次
- 自动调整生成参数

**多样化鼓励**：
- 避免单一最优策略
- 鼓励多元化防御组合
- 定期引入新的挑战元素

#### 10.4.2 扩展性考虑

**模组支持**：
- 开放的数值配置接口
- 可替换的AI行为模块
- 自定义波次配置支持

**未来扩展**：
- 新单位类型的设计空间
- 新技能系统的集成点
- 新游戏模式的适配性

---

## 11. 实现建议和技术要点

### 11.1 性能优化建议

#### 11.1.1 计算优化

**批量处理**：
- 相同类型的AI批量更新
- 路径计算批量处理
- 碰撞检测空间分割

**时间分片**：
```java
// 控制每帧AI更新数量
int maxAIUpdatesPerFrame = 50;
int aiUpdateIndex = 0;

void updateAI() {
    for(int i = 0; i < maxAIUpdatesPerFrame && aiUpdateIndex < allUnits.size; i++) {
        allUnits.get(aiUpdateIndex).updateAI();
        aiUpdateIndex = (aiUpdateIndex + 1) % allUnits.size;
    }
}
```

#### 11.1.2 内存优化

**对象池化**：
- 子弹对象重用
- 临时向量对象池
- AI状态对象缓存

**数据结构优化**：
- 使用原始类型数组
- 避免装箱操作
- 紧凑的数据布局

### 11.2 AI调试和可视化

#### 11.2.1 调试系统

**AI状态可视化**：
- 当前目标显示
- 路径线绘制
- 决策状态指示器

**性能监控**：
- AI更新耗时统计
- 路径计算性能监控
- 内存使用情况跟踪

#### 11.2.2 平衡测试工具

**自动化测试**：
- 模拟大量波次
- 收集胜率数据
- 生成平衡报告

**手动调试接口**：
- 实时参数调整
- 单步AI执行
- 波次快进功能

### 11.3 扩展性设计

#### 11.3.1 模块化架构

**接口抽象**：
```java
interface AIBehavior {
    void init(Unit unit);
    void update(Unit unit);
    void onTargetChanged(Teamc newTarget);
}
```

**插件系统**：
- 动态加载AI模块
- 运行时行为替换
- 配置文件驱动

#### 11.3.2 数据驱动设计

**配置文件化**：
- 单位属性配置文件
- 波次生成配置文件
- AI行为参数配置

**热更新支持**：
- 运行时重载配置
- 无需重启的调整
- A/B测试支持

---

## 12. 总结和展望

### 12.1 系统特色

1. **多层次AI架构**：从基础反应到复杂策略的完整AI体系
2. **动态平衡机制**：自适应难度调整确保持续挑战
3. **丰富的战术深度**：多样化的单位类型和技能系统
4. **优秀的性能表现**：多线程、缓存、时间分片等优化策略
5. **高度可扩展性**：模块化设计支持灵活的内容扩展

### 12.2 设计亮点

**1. 渐进式学习曲线**
- 从简单单位到复杂BOSS的平滑过渡
- 新机制的循序渐进引入
- 错误容忍度的合理设计

**2. 多维度平衡**
- 数量vs质量的权衡
- 速度vs防御的取舍
- 成本vs效果的平衡

**3. 战术多样性**
- 每种单位都有明确定位
- 克制关系鼓励多元化策略
- 技能组合创造丰富可能性

### 12.3 未来改进方向

#### 12.3.1 短期优化

1. **AI行为精细化**：更自然的移动模式，更智能的目标选择
2. **技能系统扩展**：更多创新技能类型，更复杂的技能组合
3. **视觉效果增强**：更丰富的战斗特效，更清晰的状态指示

#### 12.3.2 长期发展

1. **机器学习集成**：使用ML技术优化AI决策
2. **程序化内容生成**：自动生成新的单位类型和波次配置
3. **社区内容支持**：完善的模组API和创作工具

### 12.4 成功指标

**玩家体验指标**：
- 平均游戏时长增长
- 玩家留存率提升
- 难度曲线满意度

**技术性能指标**：
- AI计算开销控制在5%以内
- 路径寻找响应时间<16ms
- 内存使用稳定无泄漏

**内容丰富度指标**：
- 单位类型多样性指数
- 战术策略有效性分布
- 模组扩展活跃度

---

**文档结束**

> 本文档基于对Mindustry游戏源代码的深度分析，提取了完整的NPC战斗系统设计思路和实现细节。文档涵盖了从AI架构到数值平衡的所有关键设计要素，为理解和扩展该系统提供了全面的参考。
>
> 所有数值和公式均来自实际代码分析，具有高度的准确性和可操作性。
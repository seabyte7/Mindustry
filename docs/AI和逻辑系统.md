# AI和逻辑系统文档

## 📖 文档概述

本文档详细介绍了Mindustry中的AI系统和逻辑编程系统。AI系统负责单位的自动行为控制，而逻辑系统提供了强大的可视化编程功能，允许玩家通过逻辑处理器自动化控制游戏世界。

**适合人群**: AI开发者、逻辑系统开发者、游戏功能开发者
**重要程度**: ⭐⭐⭐⭐
**相关文档**: [实体系统和代码生成](./实体系统和代码生成.md)、[世界系统](./世界系统.md)、[网络和多人游戏](./网络和多人游戏.md)

---

## 🎯 系统架构概览

### 核心组件关系

```
AI系统架构:
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Pathfinder    │◄──►│  AIController   │◄──►│  UnitCommand    │
│  (寻路引擎)      │    │   (AI控制器)     │    │   (指令系统)     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         ▲                       ▲                       ▲
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   FlowField     │    │   AI Types      │    │   LogicAI       │
│  (流场寻路)      │    │ (AI行为类型)     │    │ (逻辑AI控制器)   │
└─────────────────┘    └─────────────────┘    └─────────────────┘

逻辑系统架构:
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   LogicBlock    │◄──►│   LExecutor     │◄──►│  LStatements    │
│  (逻辑方块)      │    │  (执行引擎)      │    │  (语句系统)      │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         ▲                       ▲                       ▲
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   LAssembler    │    │  LInstruction   │    │    LCanvas      │
│  (代码汇编器)    │    │   (指令集)       │    │  (可视化编辑器)  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

---

## 🤖 AI系统详解

### 1. 核心寻路引擎 (Pathfinder)

Mindustry的AI系统使用了高度优化的寻路引擎，支持大规模单位的高效路径规划。

#### 1.1 流场寻路 (Flow Field)

```java
// core/src/mindustry/ai/Pathfinder.java:465
public static class EnemyCoreField extends Flowfield{
    private final static BlockFlag[] randomTargets = {storage, generator, launchPad, factory, repair, battery, reactor, drill};

    @Override
    protected void getPositions(IntSeq out){
        // 获取敌方核心作为目标
        for(Building other : indexer.getEnemy(team, BlockFlag.core)){
            out.add(other.tile.array());
        }

        // 支持随机目标AI
        if(state.rules.randomWaveAI && team == state.rules.waveTeam){
            // 随机选择目标类型
            var targets = indexer.getEnemy(team, randomTargets[rand.random(randomTargets.length - 1)]);
            // ...
        }
    }
}
```

**设计特点**:
- **多线程处理**: 使用独立线程避免主循环阻塞
- **流场缓存**: 为不同团队、移动类型、目标类型缓存流场
- **动态更新**: 世界变化时增量更新流场
- **多种移动类型**: 支持地面、腿部、海军、悬浮单位

#### 1.2 移动成本系统

```java
// core/src/mindustry/ai/Pathfinder.java:45
public static final Seq<PathCost> costTypes = Seq.with(
    // 地面单位成本计算
    (team, tile) ->
        (PathTile.allDeep(tile) ||
         ((PathTile.team(tile) == team && !PathTile.teamPassable(tile)) ||
          PathTile.team(tile) == 0) && PathTile.solid(tile)) ? impassable :
            1 + PathTile.health(tile) * 5 +               // 建筑血量影响
            (PathTile.nearSolid(tile) ? 2 : 0) +          // 靠近固体
            (PathTile.nearLiquid(tile) ? 6 : 0) +         // 靠近液体
            (PathTile.deep(tile) ? 6000 : 0) +            // 深水区域
            (PathTile.damages(tile) ? 30 : 0),            // 伤害区域

    // 腿部单位成本计算
    (team, tile) ->
        PathTile.legSolid(tile) ? impassable : 1 +
        (PathTile.deep(tile) ? 6000 : 0) +               // 腿部单位也会溺水
        (PathTile.solid(tile) ? 5 : 0),

    // 海军单位成本计算
    (team, tile) ->
        (!PathTile.liquid(tile) ? 6000 : 1) +            // 必须在液体中
        PathTile.health(tile) * 5 +
        (PathTile.nearGround(tile) || PathTile.nearSolid(tile) ? 14 : 0) +
        (PathTile.deep(tile) ? 0 : 1) +                  // 偏好深水
        (PathTile.damages(tile) ? 35 : 0),

    // 悬浮单位成本计算
    (team, tile) ->
        (((PathTile.team(tile) == team && !PathTile.teamPassable(tile)) ||
          PathTile.team(tile) == 0) && PathTile.solid(tile)) ? impassable :
            1 + PathTile.health(tile) * 5 +
            (PathTile.nearSolid(tile) ? 2 : 0)
);
```

#### 1.3 A*算法实现

```java
// core/src/mindustry/ai/Astar.java:29
public static Seq<Tile> pathfind(int startX, int startY, int endX, int endY,
                                TileHeuristic th, DistanceHeuristic dh, Boolf<Tile> passable){

    GridBits closed = new GridBits(tiles.width, tiles.height);
    queue.comparator = Structs.comparingFloat(a ->
        costs[a.array()] + dh.cost(a.x, a.y, end.x, end.y));

    boolean found = false;
    while(!queue.empty()){
        Tile next = queue.poll();
        float baseCost = costs[next.array()];

        if(next == end){
            found = true;
            break;
        }

        closed.set(next.x, next.y);
        for(Point2 point : Geometry.d4){
            int newx = next.x + point.x, newy = next.y + point.y;
            if(Structs.inBounds(newx, newy, tiles.width, tiles.height)){
                Tile child = tiles.getn(newx, newy);
                if(passable.get(child)){
                    float newCost = th.cost(next, child) + baseCost;
                    // 更新路径成本和方向信息
                    // ...
                }
            }
        }
    }
    // 回溯构建路径
    // ...
}
```

### 2. AI控制器类型系统

#### 2.1 基础AI控制器

```java
// core/src/mindustry/ai/types/GroundAI.java:11
public class GroundAI extends AIController{
    @Override
    public void updateMovement(){
        Building core = unit.closestEnemyCore();

        // 攻击目标设定
        if(core != null && unit.within(core, unit.range() / 1.3f + core.block.size * tilesize / 2f)){
            target = core;
            for(var mount : unit.mounts){
                if(mount.weapon.controllable && mount.weapon.bullet.collidesGround){
                    mount.target = core;
                }
            }
        }

        // 移动逻辑
        if((core == null || !unit.within(core, unit.type.range * 0.5f))){
            boolean move = true;

            // 波次模式下的出生点逻辑
            if(state.rules.waves && unit.team == state.rules.defaultTeam){
                Tile spawner = getClosestSpawner();
                if(spawner != null && unit.within(spawner, state.rules.dropZoneRadius + 120f)) {
                    move = false;
                }
            }

            if(move) pathfind(Pathfinder.fieldCore);  // 使用流场寻路
        }

        // 悬浮单位高度控制
        if(unit.type.canBoost && unit.elevation > 0.001f && !unit.onSolid()){
            unit.elevation = Mathf.approachDelta(unit.elevation, 0f, unit.type.riseSpeed);
        }

        faceTarget();  // 面向目标
    }
}
```

#### 2.2 专业化AI类型

**BuilderAI** - 建造AI:
```java
// 自动建造和修复
- 寻找需要建造的方块
- 优化建造顺序
- 智能资源管理
- 协作建造避免冲突
```

**MinerAI** - 挖矿AI:
```java
// 自动挖矿
- 寻找最近的矿物
- 路径优化到矿点
- 满载后自动卸货
- 支持多种矿物类型
```

**RepairAI** - 修复AI:
```java
// 自动修复
- 优先级修复系统
- 寻找受损建筑
- 路径规划到目标
- 修复进度管理
```

**FlyingAI** - 飞行AI:
```java
// 飞行单位特殊逻辑
- 3D移动控制
- 高度管理
- 避障系统
- 降落逻辑
```

#### 2.3 LogicAI - 逻辑控制AI

```java
// core/src/mindustry/ai/types/LogicAI.java:14
public class LogicAI extends AIController{
    /** 控制超时时间 - 10秒后恢复正常AI */
    public static final float logicControlTimeout = 60f * 10f;

    public LUnitControl control = LUnitControl.idle;
    public float moveX, moveY, moveRad;
    public float controlTimer = logicControlTimeout;
    public @Nullable Building controller;        // 控制此单位的逻辑方块

    // 指令缓存系统
    public ObjectMap<Object, Object> execCache = new ObjectMap<>();

    @Override
    public void updateMovement(){
        // 超时检查
        if(controlTimer > 0 && controller != null && controller.isValid()){
            controlTimer -= Time.delta;
        }else{
            unit.resetController();  // 恢复正常AI
            return;
        }

        switch(control){
            case move -> {
                moveTo(Tmp.v1.set(moveX, moveY), 1f, 30f);
            }
            case approach -> {
                moveTo(Tmp.v1.set(moveX, moveY), moveRad - 7f, 7, true, null);
            }
            case pathfind -> {
                if(unit.isFlying()){
                    moveTo(Tmp.v1.set(moveX, moveY), 1f, 30f);
                }else{
                    // 使用专用的控制路径系统
                    if(controlPath.getPathPosition(unit, Tmp.v2.set(moveX, moveY),
                                                  Tmp.v2, Tmp.v1, null)){
                        moveTo(Tmp.v1, 1f, Tmp.v2.epsilonEquals(Tmp.v1, 4.1f) ? 30f : 0f);
                    }
                }
            }
            case autoPathfind -> {
                // 自动寻路到敌方核心
                Building core = unit.closestEnemyCore();
                if((core == null || !unit.within(core, unit.range() * 0.5f))){
                    // 智能路径选择
                    if(unit.isFlying()){
                        var target = core == null ? getClosestSpawner() : core;
                        if(target != null){
                            moveTo(target, unit.range() * 0.5f);
                        }
                    }else{
                        pathfind(Pathfinder.fieldCore);
                    }
                }
            }
        }
    }
}
```

### 3. RTS指令系统

```java
// core/src/mindustry/ai/UnitCommand.java:15
public class UnitCommand extends MappableContent{
    /** 指令对应的AI控制器工厂 */
    public final Func<Unit, AIController> controller;
    /** 是否在移动时自动切换到移动指令 */
    public boolean switchToMove = true;
    /** 是否绘制目标标记 */
    public boolean drawTarget = false;
    /** 快捷键绑定 */
    public @Nullable Binding keybind = null;

    // 预定义指令
    public static void loadAll(){
        moveCommand = new UnitCommand("move", "right", Binding.unit_command_move, null){{
            drawTarget = true;
            resetTarget = false;
        }};

        repairCommand = new UnitCommand("repair", "modeSurvival",
                                       Binding.unit_command_repair, u -> new RepairAI());

        rebuildCommand = new UnitCommand("rebuild", "hammer",
                                        Binding.unit_command_rebuild, u -> new BuilderAI());

        mineCommand = new UnitCommand("mine", "production",
                                     Binding.unit_command_mine, u -> new MinerAI());

        // 协助建造指令
        assistCommand = new UnitCommand("assist", "players",
                                       Binding.unit_command_assist, u -> {
            var ai = new BuilderAI();
            ai.onlyAssist = true;  // 只协助，不主动建造
            return ai;
        });
    }
}
```

---

## 🧠 逻辑编程系统详解

### 1. 逻辑执行引擎 (LExecutor)

逻辑执行引擎是整个逻辑系统的核心，负责执行逻辑程序。

#### 1.1 执行器架构

```java
// core/src/mindustry/logic/LExecutor.java:37
public class LExecutor{
    public static final int maxInstructions = 1000;        // 最大指令数
    public static final int maxGraphicsBuffer = 256;       // 最大图形缓冲
    public static final int maxDisplayBuffer = 1024;       // 最大显示缓冲

    public LInstruction[] instructions = {};               // 指令数组
    public LVar[] vars = {};                              // 变量数组

    // 内置变量
    public LVar counter, unit, thisv, ipt;                // 计数器、单位、自身、执行速率

    // 网络绑定和状态
    public int[] binds;                                   // 单位绑定数组
    public boolean yield;                                 // 是否需要等待
    public Building[] links = {};                         // 连接的建筑
    public Team team = Team.derelict;                     // 所属团队
    public boolean privileged = false;                    // 是否为特权处理器

    /** 运行单条指令 */
    public void runOnce(){
        // 重置到开始位置
        if(counter.numval >= instructions.length || counter.numval < 0){
            counter.numval = 0;
        }

        if(counter.numval < instructions.length){
            instructions[(int)(counter.numval++)].run(this);
        }
    }
}
```

#### 1.2 变量系统

```java
// 变量管理
public @Nullable LVar optionalVar(String name){
    if(nameMap == null){
        nameMap = new ObjectIntMap<>();
        for(int i = 0; i < vars.length; i++){
            nameMap.put(vars[i].name, i);
        }
    }
    return optionalVar(nameMap.get(name, -1));
}

// 变量类型支持
public class LVar{
    public String name;                    // 变量名
    public double numval;                  // 数值
    public Object objval;                  // 对象值
    public boolean isobj;                  // 是否为对象
    public boolean constant;               // 是否为常量
    public int id;                        // 变量ID

    public void setnum(double value){      // 设置数值
        numval = LVar.invalid(value) ? 0 : value;
        isobj = false;
    }

    public void setobj(Object value){      // 设置对象
        objval = value;
        isobj = true;
    }
}
```

### 2. 指令系统 (Instructions)

逻辑系统支持丰富的指令集，涵盖控制流、单位操作、建筑控制等多个方面。

#### 2.1 单位控制指令

```java
// core/src/mindustry/logic/LExecutor.java:270
public static class UnitControlI implements LInstruction{
    public LUnitControl type = LUnitControl.move;
    public LVar p1, p2, p3, p4, p5;

    @Override
    public void run(LExecutor exec){
        Object unitObj = exec.unit.obj();
        LogicAI ai = checkLogicAI(exec, unitObj);

        if(unitObj instanceof Unit unit && ai != null){
            ai.controlTimer = LogicAI.logicControlTimeout;
            float x1 = World.unconv(p1.numf()), y1 = World.unconv(p2.numf());

            switch(type){
                case move, approach, pathfind -> {
                    ai.control = type;
                    ai.moveX = x1;
                    ai.moveY = y1;
                    if(type == LUnitControl.approach){
                        ai.moveRad = World.unconv(p3.numf());
                    }
                }
                case target -> {
                    ai.posTarget.set(x1, y1);
                    ai.aimControl = type;
                    ai.shoot = p3.bool();
                }
                case build -> {
                    if((state.rules.logicUnitBuild || exec.privileged) &&
                       unit.canBuild() && p3.obj() instanceof Block block){

                        int x = World.toTile(x1 - block.offset/tilesize);
                        int y = World.toTile(y1 - block.offset/tilesize);
                        int rot = Mathf.mod(p4.numi(), 4);

                        ai.plan.set(x, y, rot, block);
                        ai.plan.config = p5.obj() instanceof Content c ? c :
                                        p5.obj() instanceof Building b ? b : null;

                        unit.clearBuilding();
                        unit.addBuild(ai.plan);
                    }
                }
                case mine -> {
                    Tile tile = world.tileWorld(x1, y1);
                    if(unit.canMine()){
                        unit.mineTile = unit.validMine(tile) ? tile : null;
                    }
                }
                case itemDrop, itemTake -> {
                    // 物品传输逻辑
                    if(!exec.timeoutDone(unit, LogicAI.transferDelay)) return;
                    // 实现物品转移...
                }
            }
        }
    }
}
```

#### 2.2 建筑控制指令

```java
// core/src/mindustry/logic/LExecutor.java:498
public static class ControlI implements LInstruction{
    public LVar target;
    public LAccess type = LAccess.enabled;
    public LVar p1, p2, p3, p4;

    @Override
    public void run(LExecutor exec){
        Object obj = target.obj();
        if(obj instanceof Building b &&
           (exec.privileged || (b.team == exec.team && exec.linkIds.contains(b.id)))){

            if(type == LAccess.enabled && !p1.bool()){
                b.lastDisabler = exec.build;  // 记录禁用者
            }

            if(type == LAccess.enabled && p1.bool()){
                b.noSleep();  // 防止休眠
            }

            // 执行控制操作
            if(type.isObj && p1.isobj){
                b.control(type, p1.obj(), p2.num(), p3.num(), p4.num());
            }else{
                b.control(type, p1.num(), p2.num(), p3.num(), p4.num());
            }
        }
    }
}
```

#### 2.3 传感器指令

```java
// core/src/mindustry/logic/LExecutor.java:615
public static class SenseI implements LInstruction{
    public LVar from, to, type;

    @Override
    public void run(LExecutor exec){
        Object target = from.obj();
        Object sense = type.obj();

        if(target == null && sense == LAccess.dead){
            to.setnum(1);  // 空目标被认为是"死亡"
            return;
        }

        if(target instanceof Senseable se){
            if(sense instanceof Content co){
                to.setnum(se.sense(co));  // 传感物品/液体/单位类型
            }else if(sense instanceof LAccess la){
                Object objOut = se.senseObject(la);

                if(objOut == Senseable.noSensed){
                    to.setnum(se.sense(la));      // 数值输出
                }else{
                    to.setobj(objOut);            // 对象输出
                }
            }
        }else{
            to.setobj(null);
        }
    }
}
```

#### 2.4 雷达指令

```java
// core/src/mindustry/logic/LExecutor.java:658
public static class RadarI implements LInstruction{
    public RadarTarget target1 = RadarTarget.enemy, target2 = RadarTarget.any, target3 = RadarTarget.any;
    public RadarSort sort = RadarSort.distance;
    public LVar radar, sortOrder, output;

    // 缓存系统避免频繁搜索
    public Healthc lastTarget;
    public Interval timer = new Interval();

    @Override
    public void run(LExecutor exec){
        Object base = radar.obj();
        int sortDir = sortOrder.bool() ? 1 : -1;
        LogicAI ai = null;

        if(base instanceof Ranged r && (exec.privileged || r.team() == exec.team)){
            float range = r.range();
            Healthc targeted;

            // 定时更新目标（30帧间隔）
            if((base instanceof Building && timer.get(30f)) ||
               (ai != null && ai.checkTargetTimer(this))){

                boolean enemies = target1 == RadarTarget.enemy ||
                                target2 == RadarTarget.enemy ||
                                target3 == RadarTarget.enemy;

                best = null;
                bestValue = 0;

                if(enemies){
                    // 搜索敌对团队
                    Seq<TeamData> data = state.teams.present;
                    for(int i = 0; i < data.size; i++){
                        if(data.items[i].team != r.team()){
                            find(r, range, sortDir, data.items[i].team);
                        }
                    }
                }

                lastTarget = targeted = best;
            }else{
                targeted = lastTarget;  // 使用缓存目标
            }

            output.setobj(targeted);
        }
    }

    void find(Ranged b, float range, int sortDir, Team team){
        Units.nearby(team, b.x(), b.y(), range, u -> {
            if(!u.within(b, range) || !u.targetable(team) || b == u) return;

            // 检查目标过滤器
            boolean valid =
                target1.func.get(b.team(), u) &&
                target2.func.get(b.team(), u) &&
                target3.func.get(b.team(), u);

            if(!valid) return;

            // 计算排序值
            float val = sort.func.get(b, u) * sortDir;
            if(val > bestValue || best == null){
                bestValue = val;
                best = u;
            }
        });
    }
}
```

### 3. 可视化编程界面 (LStatements)

逻辑系统提供了直观的可视化编程界面，将复杂的指令封装为易用的语句块。

#### 3.1 语句系统架构

```java
// core/src/mindustry/logic/LStatements.java:29
public class LStatements{

    // 基础语句示例
    @RegisterStatement("set")
    public static class SetStatement extends LStatement{
        public String to = "result";
        public String from = "0";

        @Override
        public void build(Table table){
            field(table, to, str -> to = str);
            table.add(" = ");
            field(table, from, str -> from = str);
        }

        @Override
        public LInstruction build(LAssembler builder){
            return new SetI(builder.var(from), builder.var(to));
        }
    }

    // 单位控制语句
    @RegisterStatement("ucontrol")
    public static class UnitControlStatement extends LStatement{
        public LUnitControl type = LUnitControl.move;
        public String p1 = "0", p2 = "0", p3 = "0", p4 = "0", p5 = "0";

        @Override
        public void build(Table table){
            // 动态UI构建
            table.button(b -> {
                b.label(() -> type.name());
                b.clicked(() -> showSelect(b, LUnitControl.all, type, t -> {
                    type = t;
                    rebuild(table);  // 根据类型重建UI
                }, 2, cell -> cell.size(120, 50)));
            }, Styles.logict, () -> {}).size(120, 40);

            // 根据控制类型显示相应参数
            for(int i = 0; i < type.params.length; i++){
                fields(table, type.params[i],
                      i == 0 ? p1 : i == 1 ? p2 : i == 2 ? p3 : i == 3 ? p4 : p5,
                      i == 0 ? v -> p1 = v : i == 1 ? v -> p2 = v :
                      i == 2 ? v -> p3 = v : i == 3 ? v -> p4 = v : v -> p5 = v);
            }
        }
    }
}
```

#### 3.2 图形绘制系统

```java
// 绘制指令系统
@RegisterStatement("draw")
public static class DrawStatement extends LStatement{
    public GraphicsType type = GraphicsType.clear;
    public String x = "0", y = "0", p1 = "0", p2 = "0", p3 = "0", p4 = "0";

    @Override
    public LInstruction build(LAssembler builder){
        return new DrawI((byte)type.ordinal(), builder.var(x), builder.var(y),
            type == GraphicsType.print ?
                new LVar(p1, nameToAlign.get(p1, Align.bottomLeft), true) :
                builder.var(p1),
            builder.var(p2), builder.var(p3), builder.var(p4));
    }
}

// 图形类型枚举
public enum GraphicsType{
    clear, color, col, stroke, line, rect, lineRect,
    poly, linePoly, triangle, image, print, translate, scale, rotate
}
```

### 4. 逻辑方块系统 (LogicBlock)

#### 4.1 方块架构

```java
// core/src/mindustry/world/blocks/logic/LogicBlock.java:33
public class LogicBlock extends Block{
    public int maxInstructionScale = 5;           // 最大指令缩放
    public int instructionsPerTick = 1;           // 每帧指令数
    public int maxInstructionsPerTick = 40;       // 特权处理器最大指令数
    public float range = 8 * 10;                  // 连接范围

    public class LogicBuild extends Building implements Ranged{
        public String code = "";                   // 逻辑代码
        public LExecutor executor = new LExecutor(); // 执行器
        public float accumulator = 0;              // 指令累积器
        public Seq<LogicLink> links = new Seq<>(); // 连接列表
        public int ipt = instructionsPerTick;     // 实际执行速率

        @Override
        public void updateTile(){
            executor.team = team;

            // 检查连接有效性
            boolean changed = false;
            for(LogicLink l : links){
                if(!l.active) continue;

                var cur = world.build(l.x, l.y);
                boolean valid = validLink(cur);

                if(valid != l.valid || l.lastBuild != cur){
                    l.lastBuild = cur;
                    changed = true;
                    l.valid = valid;

                    if(valid){
                        l.name = findLinkName(cur.block);  // 更新连接名称
                        links.removeAll(o -> world.build(o.x, o.y) == cur && o != l);
                    }
                }
            }

            if(changed){
                updateCode(code, true, null);  // 重新编译代码
            }

            // 执行逻辑程序
            if(enabled && executor.initialized()){
                accumulator += edelta() * ipt;

                if(accumulator > maxInstructionScale * ipt) {
                    accumulator = maxInstructionScale * ipt;
                }

                for(int i = 0; i < (int)accumulator; i++){
                    executor.runOnce();
                    accumulator --;
                    if(executor.yield){
                        executor.yield = false;
                        break;  // 等待指令中断执行
                    }
                }
            }
        }

        // 连接验证
        public boolean validLink(Building other){
            return other != null && other.isValid() &&
                   (privileged || (!other.block.privileged &&
                                  other.team == team &&
                                  other.within(this, range + other.block.size*tilesize/2f))) &&
                   !(other instanceof ConstructBuild);
        }
    }
}
```

#### 4.2 连接系统

```java
// 逻辑连接管理
public static class LogicLink{
    public boolean active = true, valid;
    public int x, y;
    public String name;
    public Building lastBuild;

    public LogicLink(int x, int y, String name, boolean valid){
        this.x = x;
        this.y = y;
        this.name = name;
        this.valid = valid;
    }
}

// 连接名称生成
public String findLinkName(Block block){
    String bname = getLinkName(block);
    Bits taken = new Bits(links.size);
    int max = 1;

    // 查找已占用的编号
    for(LogicLink others : links){
        if(others.name.startsWith(bname)){
            String num = others.name.substring(bname.length());
            try{
                int val = Integer.parseInt(num);
                taken.set(val);
                max = Math.max(val, max);
            }catch(NumberFormatException ignored){}
        }
    }

    // 分配新编号
    int outnum = 0;
    for(int i = 1; i < max + 2; i++){
        if(!taken.get(i)){
            outnum = i;
            break;
        }
    }

    return bname + outnum;  // 如: "conveyor1", "turret2"
}
```

---

## 🔧 高级特性和优化

### 1. 性能优化技术

#### 1.1 多线程寻路

```java
// 寻路系统在独立线程中运行
@Override
public void run(){
    while(true){
        try{
            if(state.isPlaying()){
                queue.run();

                // 每次更新不超过maxUpdate时间
                for(Flowfield data : threadList){
                    if(data.dirty && data.frontier.size == 0){
                        updateTargets(data);
                        data.dirty = false;
                    }
                    updateFrontier(data, maxUpdate);
                }
            }

            Thread.sleep(updateInterval);  // 固定更新间隔
        }catch(InterruptedException e){
            return;
        }
    }
}
```

#### 1.2 指令缓存系统

```java
// LogicAI中的执行缓存
public ObjectMap<Object, Object> execCache = new ObjectMap<>();

// 雷达指令缓存示例
Cache cache = (Cache)ai.execCache.get(this, Cache::new);
if(ai.checkTargetTimer(this)){
    // 执行搜索并缓存结果
    cache.found = true;
    cache.x = targetX;
    cache.y = targetY;
}else{
    // 使用缓存结果
    outX.setnum(cache.x);
    outY.setnum(cache.y);
}
```

### 2. 网络同步

#### 2.1 变量同步

```java
// 变量网络同步
@Remote(unreliable = true)
public static void syncVariable(Building building, int variable, Object value){
    if(building instanceof LogicBuild build){
        LVar v = build.executor.optionalVar(variable);
        if(v != null && !v.constant){
            if(value instanceof Number n){
                v.isobj = false;
                v.numval = n.doubleValue();
            }else{
                v.isobj = true;
                v.objval = value;
            }
        }
    }
}

// 同步指令
public static class SyncI implements LInstruction{
    public static long syncInterval = 1000 / 20;  // 20次/秒

    @Override
    public void run(LExecutor exec){
        if(!variable.constant && Time.timeSinceMillis(variable.syncTime) > syncInterval){
            variable.syncTime = Time.millis();
            Call.syncVariable(exec.build, variable.id,
                            variable.isobj ? variable.objval : variable.numval);
        }
    }
}
```

### 3. 安全机制

#### 3.1 权限控制

```java
// 特权处理器检查
public boolean accessible(){
    return !privileged || state.rules.editor ||
           state.playtestingMap != null ||
           state.rules.allowEditWorldProcessors;
}

// 单位绑定权限验证
if(type.obj() instanceof Unit u &&
   (u.team == exec.team || exec.privileged) &&
   u.type.logicControllable){
    exec.unit.setconst(u);
}
```

#### 3.2 资源限制

```java
// 执行限制
public static final int maxInstructions = 1000;
public static final int maxGraphicsBuffer = 256;
public static final int maxDisplayBuffer = 1024;
public static final int maxTextBuffer = 400;

// 缓冲区检查
if(exec.graphicsBuffer.size >= maxGraphicsBuffer) return;
if(exec.textBuffer.length() >= maxTextBuffer) return;
```

---

## 🎯 最佳实践和开发指南

### 1. AI开发最佳实践

#### 1.1 创建自定义AI类型

```java
public class CustomAI extends AIController{
    @Override
    public void updateMovement(){
        // 1. 获取目标
        Tile target = findTarget();

        // 2. 路径规划
        if(target != null){
            pathfind(target);  // 使用流场寻路
        }

        // 3. 行为逻辑
        if(shouldAttack()){
            faceTarget();
        }

        // 4. 特殊状态处理
        if(unit.type.canBoost){
            unit.elevation = calculateOptimalHeight();
        }
    }

    private Tile findTarget(){
        // 实现目标查找逻辑
        return unit.closestEnemyCore();
    }

    private boolean shouldAttack(){
        // 实现攻击判断逻辑
        return target != null && unit.within(target, unit.range());
    }
}
```

#### 1.2 寻路系统使用

```java
// 使用流场寻路（推荐）
pathfind(Pathfinder.fieldCore);

// 使用A*寻路（小规模）
Seq<Tile> path = Astar.pathfind(start, end,
    tile -> tile.solid() ? 1000 : 1,  // 成本函数
    tile -> !tile.solid()             // 可通行检查
);

// 自定义移动成本
PathCost customCost = (team, tile) -> {
    if(PathTile.solid(tile)) return Pathfinder.impassable;
    return 1 + (PathTile.damages(tile) ? 50 : 0);  // 避开伤害区域
};
```

### 2. 逻辑编程最佳实践

#### 2.1 创建自定义指令

```java
@RegisterStatement("myinstruction")
public static class MyInstructionStatement extends LStatement{
    public String param1 = "0", param2 = "result";

    @Override
    public void build(Table table){
        // 构建UI
        field(table, param1, str -> param1 = str);
        table.add(" -> ");
        field(table, param2, str -> param2 = str);
    }

    @Override
    public LInstruction build(LAssembler builder){
        return new MyInstructionI(builder.var(param1), builder.var(param2));
    }

    @Override
    public LCategory category(){
        return LCategory.operation;  // 分类
    }
}

// 对应的指令实现
public static class MyInstructionI implements LInstruction{
    public LVar input, output;

    public MyInstructionI(LVar input, LVar output){
        this.input = input;
        this.output = output;
    }

    @Override
    public void run(LExecutor exec){
        // 执行逻辑
        output.setnum(input.num() * 2);
    }
}
```

#### 2.2 复杂逻辑程序示例

```javascript
// 自动防御塔控制程序
sensor enemy turret1 @shooting       // 检测是否在射击
radar enemy any any distance turret1 1 target  // 搜索最近敌人

jump 10 equal enemy 0                 // 如果没在射击跳转到搜索
jump 20 equal target null             // 如果没有目标跳转到待机

// 攻击模式
control shoot turret1 target.x target.y 1  // 射击目标
jump 0 always                         // 循环

// 搜索模式
radar enemy any any distance @this 1 newTarget
jump 20 equal newTarget null          // 没找到目标
control target turret1 newTarget.x newTarget.y 1
jump 0 always

// 待机模式
control enabled turret1 0             // 关闭炮塔
wait 1                                // 等待1秒
control enabled turret1 1             // 重新启用
jump 0 always                         // 回到开始
```

### 3. 调试和性能优化

#### 3.1 性能监控

```java
// 执行时间监控
long startTime = Time.nanos();
executor.runOnce();
long execTime = Time.timeSinceNanos(startTime);

if(execTime > maxExecutionTime){
    Log.warn("Logic processor execution too slow: " + execTime + "ns");
}

// 内存使用监控
int variableCount = executor.vars.length;
int linkCount = executor.links.length;
int instructionCount = executor.instructions.length;
```

#### 3.2 常见问题和解决方案

**问题1: 单位失去控制**
```java
// 检查控制超时
if(ai.controlTimer <= 0){
    // 重新绑定单位
    Call.unitControl(processor, LUnitControl.unbind, 0, 0, 0, 0, 0);
    Call.unitControl(processor, LUnitControl.bind, unitType.id, 0, 0, 0, 0);
}
```

**问题2: 网络同步问题**
```java
// 使用sync指令同步关键变量
sync importantVariable  // 自动网络同步

// 或手动检查网络状态
sensor connected processor @connected
jump syncError equal connected 0
```

**问题3: 性能优化**
```java
// 减少不必要的传感器调用
sensor health target @health
jump skipCheck equal health lastHealth  // 血量未变化时跳过
set lastHealth health

// 使用定时器减少更新频率
op add timer timer 1
op mod timerMod timer 60
jump updateTargets equal timerMod 0      // 每60帧更新一次目标
```

---

## 📚 相关资源和扩展阅读

### 开发文档
- [实体系统和代码生成](./实体系统和代码生成.md) - 了解单位组件系统
- [世界系统](./世界系统.md) - 了解建筑和方块系统
- [网络和多人游戏](./网络和多人游戏.md) - 了解网络同步机制

### 核心类参考
- `mindustry.ai.Pathfinder` - 寻路引擎实现
- `mindustry.ai.types.*` - 各种AI控制器类型
- `mindustry.logic.LExecutor` - 逻辑执行引擎
- `mindustry.logic.LStatements` - 可视化编程语句
- `mindustry.world.blocks.logic.LogicBlock` - 逻辑方块实现

### 扩展点
- 自定义AI行为类型
- 新的逻辑指令和语句
- 特殊用途的寻路算法
- 逻辑处理器的扩展功能

---

**本文档涵盖了Mindustry中AI和逻辑系统的核心架构、实现细节和开发实践。这两个系统的紧密集成为游戏提供了强大的自动化和编程能力，是Mindustry区别于其他游戏的重要特色。**
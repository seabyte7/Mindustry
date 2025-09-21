# MOD开发完整指南

Mindustry拥有功能强大的MOD系统，支持Java类MOD和JavaScript脚本MOD两种开发方式，提供了完整的内容定义、依赖管理、资源打包和API扩展能力，为开发者创建自定义游戏内容提供了全面的解决方案。

## 核心架构概述

### 1. MOD系统架构层次

```
┌─────────────────────────────────────┐
│            Mods                     │
│         (MOD管理器)                  │
├─────────────────────────────────────┤
│   Java类MOD   │   JavaScript脚本MOD  │
│   (Mod/Plugin) │   (Scripts)         │
├─────────────────┬─────────────────────┤
│  ContentParser  │   资源打包系统        │
│  (内容解析器)    │   (Sprite Packing)   │
├─────────────────┼─────────────────────┤
│  依赖管理系统    │   类加载器            │
│  (Dependencies) │   (ClassLoaders)     │
└─────────────────┴─────────────────────┘
```

### 2. MOD类型比较

| MOD类型 | 适用场景 | 开发语言 | 性能 | 复杂度 |
|---------|----------|----------|------|--------|
| **Java类MOD** | 复杂功能，新建筑/单位类型，深度定制 | Java | 高 | 高 |
| **JavaScript脚本MOD** | 简单逻辑，事件处理，数据修改 | JavaScript | 中等 | 低 |
| **纯内容MOD** | 新建筑/单位/物品定义，无自定义逻辑 | JSON/HJSON | 高 | 最低 |
| **插件(Plugin)** | 服务器端功能，管理工具，隐藏功能 | Java | 高 | 中等 |

## MOD项目结构

### 1. 标准MOD目录结构

```
my-mod/
├── mod.json/mod.hjson          # MOD元数据配置
├── icon.png                    # MOD图标 (可选)
├── preview.png                 # Steam预览图 (可选)
├── content/                    # 游戏内容定义
│   ├── blocks/                 # 建筑定义
│   ├── items/                  # 物品定义
│   ├── liquids/                # 液体定义
│   ├── units/                  # 单位定义
│   ├── sectors/                # 区域定义
│   └── planets/                # 星球定义
├── bundles/                    # 国际化文本
│   ├── bundle.properties       # 默认语言
│   ├── bundle_zh.properties    # 中文
│   └── bundle_en.properties    # 英文
├── sprites/                    # 精灵图像
│   ├── blocks/                 # 建筑贴图
│   ├── items/                  # 物品贴图
│   ├── units/                  # 单位贴图
│   └── ui/                     # UI贴图
├── sprites-override/           # 覆盖原版贴图
├── scripts/                    # JavaScript脚本 (可选)
│   ├── main.js                 # 主脚本文件
│   └── lib/                    # 库文件
├── sounds/                     # 音效文件 (可选)
└── src/                        # Java源代码 (Java MOD)
    └── example/
        └── ExampleMod.java
```

### 2. MOD元数据配置 (mod.json)

```json
{
  "name": "example-mod",
  "displayName": "示例MOD",
  "author": "作者名称",
  "version": "1.0.0",
  "minGameVersion": "140",
  "description": "这是一个示例MOD，展示基本功能。",
  "subtitle": "示例MOD副标题",
  "dependencies": ["other-mod"],
  "softDependencies": ["optional-mod"],
  "hidden": false,
  "java": true,
  "main": "example.ExampleMod",
  "repo": "username/example-mod",
  "texturescale": 1.0,
  "pregenerated": false,
  "contentOrder": ["my-block", "my-item"]
}
```

### 3. HJSON格式配置

```hjson
{
  # MOD基本信息
  name: example-mod
  displayName: 示例MOD
  author: 作者名称
  version: 1.0.0
  minGameVersion: 140

  # 描述信息
  description: '''
    这是一个示例MOD，展示以下功能：
    - 新建筑类型
    - 自定义物品
    - JavaScript脚本扩展
  '''

  # 依赖关系
  dependencies: [
    other-mod
  ]

  # MOD设置
  java: true
  main: example.ExampleMod
}
```

## Java类MOD开发

### 1. 基础MOD类结构

```java
package example;

import arc.*;
import arc.util.*;
import mindustry.*;
import mindustry.content.*;
import mindustry.game.EventType.*;
import mindustry.gen.*;
import mindustry.mod.*;
import mindustry.ui.dialogs.*;

public class ExampleMod extends Mod{

    public ExampleMod(){
        Log.info("ExampleMod 构造函数被调用");

        // 注册事件监听器
        Events.on(ClientLoadEvent.class, e -> {
            Log.info("客户端加载完成");
        });
    }

    @Override
    public void init(){
        Log.info("ExampleMod 初始化");

        // 添加自定义UI
        Vars.ui.menufrag.addButton("示例按钮", Icon.settings, () -> {
            new BaseDialog("示例对话框"){{
                cont.add("这是一个自定义对话框").row();
                cont.button("确定", this::hide).size(100f, 50f);
            }}.show();
        });
    }

    @Override
    public void loadContent(){
        Log.info("加载MOD内容");

        // 这里会自动加载content/文件夹中的内容
        // 也可以手动创建内容
        createCustomContent();
    }

    @Override
    public void registerServerCommands(CommandHandler handler){
        // 注册服务器命令
        handler.register("hello", "<player>", "向玩家问好", args -> {
            Player player = Groups.player.find(p -> p.name.equalsIgnoreCase(args[0]));
            if(player != null){
                player.sendMessage("[green]你好，" + player.name + "!");
            }
        });
    }

    @Override
    public void registerClientCommands(CommandHandler handler){
        // 注册客户端命令
        handler.register("info", "显示MOD信息", args -> {
            Log.info("ExampleMod v1.0.0 - 示例MOD");
        });
    }

    private void createCustomContent(){
        // 手动创建自定义内容的示例会在后面详细说明
    }
}
```

### 2. 自定义建筑类型

```java
package example.world.blocks;

import arc.graphics.g2d.*;
import arc.math.*;
import arc.util.*;
import mindustry.content.*;
import mindustry.entities.*;
import mindustry.entities.units.*;
import mindustry.gen.*;
import mindustry.graphics.*;
import mindustry.type.*;
import mindustry.world.*;
import mindustry.world.blocks.production.*;
import mindustry.world.consumers.*;
import mindustry.world.draw.*;
import mindustry.world.meta.*;

public class ExampleDrill extends Drill{

    public ExampleDrill(String name){
        super(name);

        // 基础属性
        requirements(Category.production, with(Items.copper, 50, Items.graphite, 25));
        size = 2;
        tier = 2;
        drillTime = 200f;
        liquidCapacity = 30f;

        // 能耗设置
        consumePower(1.2f);
        consumeLiquid(Liquids.water, 0.1f).boost();

        // 外观设置
        drawer = new DrawMulti(
            new DrawDefault(),
            new DrawRegion("-rotator"){{
                spinSprite = true;
                rotateSpeed = 2f;
            }},
            new DrawLiquidTile(Liquids.water, 2f)
        );
    }

    public class ExampleDrillBuild extends DrillBuild{

        @Override
        public void updateTile(){
            super.updateTile();

            // 自定义更新逻辑
            if(efficiency > 0.5f && Mathf.chance(0.03)){
                // 产生特效
                Fx.steam.at(x + Mathf.range(size * 4), y + Mathf.range(size * 4));
            }
        }

        @Override
        public void draw(){
            super.draw();

            // 自定义绘制
            if(isWorking()){
                Draw.color(Pal.accent);
                Draw.alpha(efficiency * 0.8f);
                Draw.rect("example-mod-drill-glow", x, y);
                Draw.reset();
            }
        }
    }
}
```

### 3. 自定义单位类型

```java
package example.entities.units;

import arc.graphics.g2d.*;
import arc.math.*;
import arc.util.*;
import mindustry.ai.types.*;
import mindustry.content.*;
import mindustry.entities.*;
import mindustry.entities.abilities.*;
import mindustry.entities.bullet.*;
import mindustry.gen.*;
import mindustry.graphics.*;
import mindustry.type.*;
import mindustry.type.weapons.*;

public class ExampleUnitType extends UnitType{

    public ExampleUnitType(String name){
        super(name);

        // 基础属性
        speed = 1.2f;
        accel = 0.04f;
        drag = 0.018f;
        flying = true;
        health = 250;
        engineSize = 3f;
        hitSize = 12f;
        armor = 2f;

        // AI设置
        constructor = UnitEntity::create;
        controller = u -> new FlyingAI();

        // 武器配置
        weapons.add(new Weapon("example-mod-laser"){{
            reload = 60f;
            x = 4f;
            y = 2f;
            top = false;
            ejectEffect = Fx.casing1;

            bullet = new LaserBoltBulletType(5.2f, 10){{
                lifetime = 35f;
                backColor = Pal.techBlue;
                frontColor = Color.white;
                width = 2f;
                height = 8f;
            }};
        }});

        // 特殊能力
        abilities.add(new ShieldArcAbility(){{
            region = "example-mod-shield";
            radius = 25f;
            angle = 160f;
            width = 3f;
            y = -3f;
        }});
    }
}
```

### 4. 自定义效果和能力

```java
package example.entities.abilities;

import arc.graphics.*;
import arc.graphics.g2d.*;
import arc.math.*;
import arc.util.*;
import mindustry.content.*;
import mindustry.entities.*;
import mindustry.entities.abilities.*;
import mindustry.gen.*;
import mindustry.graphics.*;

public class RegenerationAbility extends Ability{
    public float healAmount = 2f;
    public float range = 60f;
    public float reload = 100f;

    protected float timer;

    @Override
    public void update(Unit unit){
        timer += Time.delta;

        if(timer >= reload){
            // 治疗周围友军
            Units.nearby(unit.team, unit.x, unit.y, range, other -> {
                if(other != unit && other.damaged()){
                    other.heal(healAmount);
                    Fx.heal.at(other.x, other.y);
                }
            });

            timer = 0f;
        }
    }

    @Override
    public void draw(Unit unit){
        if(timer > reload * 0.8f){
            // 绘制治疗光环预览
            Draw.color(Pal.heal, 0.4f);
            Lines.stroke(2f);
            Lines.circle(unit.x, unit.y, range);
            Draw.reset();
        }
    }

    @Override
    public String localized(){
        return "regeneration-ability";
    }
}
```

## JavaScript脚本MOD开发

### 1. 基础脚本结构 (main.js)

```javascript
// 导入必要的Java类
const {
    Events, Vars, Log, Timer, Time,
    Groups, Call, Fx, Pal
} = require('mindustry-imports');

// MOD信息会自动注入
print(`${modName} JavaScript脚本已加载`);

// 事件监听
Events.on(ClientLoadEvent, () => {
    Log.info("客户端加载完成 - JavaScript");
});

Events.on(PlayerJoin, event => {
    const player = event.player;
    Call.sendMessage(`[green]欢迎 ${player.name} 加入游戏!`);
});

Events.on(BlockBuildEndEvent, event => {
    if (event.team === Vars.player.team()) {
        Log.info(`建造了: ${event.tile.block().name}`);
    }
});

// 自定义函数
function healAllPlayers() {
    Groups.player.each(player => {
        if (player.unit()) {
            player.unit().heal(100);
            Fx.heal.at(player.unit().x, player.unit().y);
        }
    });
}

// 定时器任务
Timer.schedule(() => {
    if (Vars.state.isGame()) {
        healAllPlayers();
        Call.sendMessage("[green]所有玩家已被治疗!");
    }
}, 0, 300); // 每5分钟执行一次

// 控制台命令
if (Vars.headless) {
    // 服务器命令
    Vars.netServer.admins.addActionFilter(action => {
        if (action.type === AdminAction.kick) {
            Log.info(`管理员踢出了玩家: ${action.player.name}`);
        }
        return true;
    });
}

// 导出函数供其他脚本使用
exports.healAllPlayers = healAllPlayers;
```

### 2. 模块化脚本开发

```javascript
// lib/utils.js - 工具函数库
const Utils = {
    // 格式化时间
    formatTime(seconds) {
        const minutes = Math.floor(seconds / 60);
        const secs = Math.floor(seconds % 60);
        return `${minutes}:${secs.toString().padStart(2, '0')}`;
    },

    // 获取随机玩家
    getRandomPlayer() {
        const players = Groups.player.copy();
        return players.size > 0 ? players.get(Mathf.random(players.size - 1)) : null;
    },

    // 广播消息
    broadcast(message, color = 'white') {
        Call.sendMessage(`[${color}]${message}`);
    },

    // 在指定位置创建效果
    effect(fx, x, y, rotation = 0, color = Color.white) {
        Call.effect(fx, x, y, rotation, color);
    }
};

module.exports = Utils;
```

```javascript
// lib/commands.js - 命令系统
const Utils = require('utils');

const Commands = {
    // 注册服务器命令
    registerServerCommands(handler) {
        handler.register("heal", "<player>", "治疗指定玩家", args => {
            const player = Groups.player.find(p =>
                p.name.toLowerCase().includes(args[0].toLowerCase())
            );

            if (player && player.unit()) {
                player.unit().heal(player.unit().maxHealth);
                Utils.effect(Fx.heal, player.unit().x, player.unit().y);
                Utils.broadcast(`${player.name} 已被治疗`, 'green');
            } else {
                Log.info("找不到玩家: " + args[0]);
            }
        });

        handler.register("teleport", "<player1> <player2>", "传送玩家", args => {
            const from = Groups.player.find(p =>
                p.name.toLowerCase().includes(args[0].toLowerCase())
            );
            const to = Groups.player.find(p =>
                p.name.toLowerCase().includes(args[1].toLowerCase())
            );

            if (from && to && from.unit() && to.unit()) {
                from.unit().set(to.unit().x, to.unit().y);
                Utils.effect(Fx.spawn, from.unit().x, from.unit().y);
                Utils.broadcast(`${from.name} 传送到了 ${to.name}`, 'yellow');
            }
        });
    },

    // 注册客户端命令
    registerClientCommands(handler) {
        handler.register("time", "显示游戏时间", args => {
            const gameTime = Utils.formatTime(Vars.state.tick / 60);
            Utils.broadcast(`游戏时间: ${gameTime}`, 'cyan');
        });
    }
};

module.exports = Commands;
```

```javascript
// main.js - 主脚本文件
const Utils = require('lib/utils');
const Commands = require('lib/commands');

print("高级JavaScript MOD已加载");

// 游戏状态管理
const GameState = {
    lastWave: 0,
    playerStats: new Map()
};

// 事件处理
Events.on(WaveEvent, () => {
    const currentWave = Vars.state.wave;
    if (currentWave > GameState.lastWave) {
        Utils.broadcast(`第 ${currentWave} 波敌人来袭!`, 'red');
        GameState.lastWave = currentWave;

        // 奖励生存玩家
        Groups.player.each(player => {
            if (player.unit() && !player.unit().dead) {
                player.unit().heal(50);
                Utils.effect(Fx.heal, player.unit().x, player.unit().y);
            }
        });
    }
});

Events.on(PlayerJoin, event => {
    const player = event.player;
    GameState.playerStats.set(player.uuid(), {
        joinTime: Time.millis(),
        blocksBuilt: 0,
        deaths: 0
    });

    Utils.broadcast(`欢迎 ${player.name} 加入游戏!`, 'green');
});

Events.on(PlayerLeave, event => {
    const player = event.player;
    const stats = GameState.playerStats.get(player.uuid());
    if (stats) {
        const playTime = (Time.millis() - stats.joinTime) / 1000;
        Log.info(`${player.name} 游戏了 ${Utils.formatTime(playTime)}`);
        GameState.playerStats.delete(player.uuid());
    }
});

// 注册命令
if (typeof registerServerCommands !== 'undefined') {
    Commands.registerServerCommands(registerServerCommands);
}
if (typeof registerClientCommands !== 'undefined') {
    Commands.registerClientCommands(registerClientCommands);
}
```

### 3. 高级脚本功能

```javascript
// 自定义UI系统
const CustomUI = {
    createInfoPanel() {
        if (Vars.mobile) return; // 移动端跳过

        const infoTable = new Table();
        infoTable.top().left();
        infoTable.setFillParent(true);

        infoTable.update(() => {
            infoTable.clear();
            infoTable.table(t => {
                t.defaults().left();
                t.add(`[yellow]在线玩家: [white]${Groups.player.size()}`).row();
                t.add(`[yellow]当前波次: [white]${Vars.state.wave}`).row();
                t.add(`[yellow]游戏时间: [white]${Utils.formatTime(Vars.state.tick / 60)}`).row();

                if (Vars.state.rules.sector) {
                    const sector = Vars.state.rules.sector;
                    t.add(`[yellow]区域: [white]${sector.name()}`).row();
                }
            }).top().left().margin(10);
        });

        Vars.ui.hudGroup.addChild(infoTable);
    }
};

// 自动化系统
const Automation = {
    autoRepair: {
        enabled: false,
        range: 100,

        update() {
            if (!this.enabled || !Vars.player.unit()) return;

            const player = Vars.player.unit();
            Vars.indexer.eachBlock(Vars.player.team(), player.x, player.y, this.range,
                block => block.damaged(),
                build => {
                    if (build.damaged()) {
                        build.heal(build.maxHealth * 0.1);
                        if (Mathf.chance(0.1)) {
                            Utils.effect(Fx.healBlockFull, build.x, build.y);
                        }
                    }
                }
            );
        }
    }
};

// 数据统计系统
const Statistics = {
    data: {
        blocksBuilt: 0,
        blocksDestroyed: 0,
        enemiesKilled: 0,
        itemsProduced: new Map()
    },

    save() {
        const data = JSON.stringify(this.data);
        Vars.core.settings.put('mod-statistics', data);
    },

    load() {
        const data = Vars.core.settings.getString('mod-statistics', '{}');
        try {
            this.data = JSON.parse(data);
            if (!this.data.itemsProduced) {
                this.data.itemsProduced = new Map();
            }
        } catch (e) {
            Log.warn("统计数据加载失败: " + e);
        }
    },

    display() {
        Utils.broadcast(`建造: ${this.data.blocksBuilt}, ` +
                       `摧毁: ${this.data.blocksDestroyed}, ` +
                       `击杀: ${this.data.enemiesKilled}`, 'cyan');
    }
};

// 初始化
Events.on(ClientLoadEvent, () => {
    CustomUI.createInfoPanel();
    Statistics.load();
});

// 定期更新
Timer.schedule(() => {
    Automation.autoRepair.update();
    Statistics.save();
}, 0, 60); // 每秒更新
```

## 内容定义系统

### 1. 方块定义 (content/blocks/example-drill.json)

```json
{
  "type": "Drill",
  "name": "高级钻头",
  "description": "一个高效的钻头，可以快速开采矿物。",
  "size": 2,
  "tier": 3,
  "drillTime": 120,
  "liquidCapacity": 40,
  "health": 180,
  "requirements": [
    "copper/80",
    "graphite/35",
    "silicon/20"
  ],
  "category": "production",
  "research": "pneumatic-drill",
  "researchCost": [
    "copper/200",
    "graphite/100"
  ],
  "consumes": {
    "power": 1.5,
    "liquid": {
      "liquid": "water",
      "amount": 0.15,
      "booster": true,
      "optional": true
    }
  },
  "drawer": {
    "type": "DrawMulti",
    "drawers": [
      {
        "type": "DrawDefault"
      },
      {
        "type": "DrawRegion",
        "suffix": "-rotator",
        "spinSprite": true,
        "rotateSpeed": 3
      },
      {
        "type": "DrawLiquidTile",
        "drawLiquid": "water",
        "padding": 2
      }
    ]
  }
}
```

### 2. 单位定义 (content/units/example-fighter.json)

```json
{
  "type": "flying",
  "name": "示例战机",
  "description": "一个装备激光武器的快速战机。",
  "speed": 1.8,
  "accel": 0.08,
  "drag": 0.016,
  "health": 320,
  "armor": 3,
  "hitSize": 14,
  "engineSize": 4,
  "engineOffset": 8,
  "flying": true,
  "lowAltitude": true,
  "research": "flare",
  "researchCost": [
    "copper/300",
    "lead/200",
    "graphite/150"
  ],
  "weapons": [
    {
      "name": "example-laser-weapon",
      "x": 0,
      "y": 0,
      "reload": 40,
      "mirror": false,
      "shootSound": "laser",
      "bullet": {
        "type": "LaserBoltBulletType",
        "damage": 18,
        "speed": 4,
        "lifetime": 45,
        "backColor": "665C9F",
        "frontColor": "FFFFFF",
        "width": 3,
        "height": 12,
        "collidesGround": true,
        "collidesAir": true
      }
    }
  ],
  "abilities": [
    {
      "type": "ShieldArcAbility",
      "radius": 28,
      "angle": 120,
      "width": 4,
      "cooldown": 300,
      "max": 400,
      "regen": 2
    }
  ]
}
```

### 3. 物品定义 (content/items/advanced-alloy.json)

```json
{
  "name": "高级合金",
  "description": "一种由多种金属合成的坚固合金。",
  "color": "A0A0A0",
  "cost": 2.5,
  "flammability": 0,
  "explosiveness": 0,
  "radioactivity": 0,
  "charge": 0,
  "hardness": 4,
  "research": "plastanium",
  "researchCost": [
    "copper/500",
    "titanium/300",
    "plastanium/200"
  ]
}
```

### 4. 液体定义 (content/liquids/cooling-fluid.json)

```json
{
  "name": "冷却液",
  "description": "一种高效的冷却液体，可以显著降低建筑温度。",
  "color": "87CEEB",
  "barColor": "6495ED",
  "effect": "freezing",
  "temperature": 0.2,
  "heatCapacity": 0.8,
  "viscosity": 0.6,
  "flammability": 0,
  "explosiveness": 0,
  "research": "cryofluid",
  "researchCost": [
    "water/100",
    "cryofluid/50"
  ]
}
```

## 依赖管理和版本控制

### 1. 依赖声明

```json
{
  "name": "my-expansion-mod",
  "dependencies": [
    "base-mod",
    "utility-mod"
  ],
  "softDependencies": [
    "optional-content-mod"
  ],
  "minGameVersion": "140.1"
}
```

### 2. 依赖解析流程

```java
// Mods.java中的依赖解析逻辑
public OrderedMap<String, ModState> resolveDependencies(Seq<ModMeta> metas){
    var context = new ModResolutionContext();

    // 构建依赖关系图
    for(var meta : metas){
        Seq<ModDependency> dependencies = new Seq<>();
        for(var dependency : meta.dependencies){
            dependencies.add(new ModDependency(dependency, true)); // 必需依赖
        }
        for(var dependency : meta.softDependencies){
            dependencies.add(new ModDependency(dependency, false)); // 可选依赖
        }
        context.dependencies.put(meta.internalName, dependencies);
    }

    // 拓扑排序解析加载顺序
    for(var key : context.dependencies.keys()){
        if(!context.ordered.contains(key)){
            resolve(key, context);
            context.visited.clear();
        }
    }

    return context.ordered;
}
```

### 3. 版本兼容性检查

```java
public boolean isSupported(){
    // 游戏版本检查
    if(!Version.isAtLeast(meta.minGameVersion)) return false;

    // 平台兼容性检查
    if(meta.getMinMajor() < 136 && !headless) return false; // v7兼容性

    // 黑名单检查
    if(isBlacklisted()) return false;

    return true;
}
```

## 资源打包和优化

### 1. 精灵图打包系统

```java
// Mods.java中的精灵打包逻辑
private void packSprites(Seq<Fi> sprites, LoadedMod mod, boolean prefix, Seq<Future<Runnable>> tasks){
    boolean bleed = Core.settings.getBool("linear", true) && !mod.meta.pregenerated;
    float textureScale = mod.meta.texturescale;

    for(Fi file : sprites){
        String baseName = file.nameWithoutExtension();
        String regionName = baseName.contains(".") ?
                           baseName.substring(0, baseName.indexOf(".")) : baseName;

        // 并行读取和处理图像
        tasks.add(mainExecutor.submit(() -> {
            try{
                Pixmap pix = new Pixmap(file.readBytes());
                // 线性过滤时进行出血处理
                if(bleed){
                    Pixmaps.bleed(pix, 2);
                }

                return () -> {
                    // 生成最终的纹理名称
                    String fullName = ((prefix && !baseName.contains(mod.name + "-")) ?
                                      mod.name + "-" : "") + baseName;

                    packer.add(getPage(file), fullName, new PixmapRegion(pix));
                    if(textureScale != 1.0f){
                        textureResize.put(fullName, textureScale);
                    }
                    pix.dispose();
                };
            }catch(Exception e){
                throw new Exception("Failed to load image " + file + " for mod " + mod.name, e);
            }
        }));
    }
}
```

### 2. 纹理页面分类

```java
private PageType getPage(Fi file){
    String path = file.path();
    return
        path.contains("sprites/blocks/environment") || path.contains("sprites-override/blocks/environment") ?
            PageType.environment :
        path.contains("sprites/editor") || path.contains("sprites-override/editor") ?
            PageType.editor :
        path.contains("sprites/rubble") || path.contains("sprites-override/rubble") ?
            PageType.rubble :
        path.contains("sprites/ui") || path.contains("sprites-override/ui") ?
            PageType.ui :
        PageType.main;
}
```

### 3. 图标自动生成

```java
// 为MOD内容自动生成图标
for(Seq<Content> arr : content.getContentMap()){
    arr.each(c -> {
        if(c instanceof UnlockableContent u && c.minfo.mod != null){
            u.load();
            u.loadIcon();
            if(u.generateIcons && !c.minfo.mod.meta.pregenerated){
                u.createIcons(packer); // 自动生成图标
            }
        }
    });
}
```

## 国际化支持

### 1. 语言文件结构

```properties
# bundles/bundle.properties (默认英文)
block.example-mod-advanced-drill.name = Advanced Drill
block.example-mod-advanced-drill.description = A highly efficient drill for mining operations.
item.example-mod-advanced-alloy.name = Advanced Alloy
item.example-mod-advanced-alloy.description = A durable alloy made from multiple metals.
unit.example-mod-fighter.name = Example Fighter
ui.example-button = Example Button
message.welcome = Welcome to the server!
```

```properties
# bundles/bundle_zh.properties (中文)
block.example-mod-advanced-drill.name = 高级钻头
block.example-mod-advanced-drill.description = 一个用于采矿作业的高效钻头。
item.example-mod-advanced-alloy.name = 高级合金
item.example-mod-advanced-alloy.description = 一种由多种金属制成的耐用合金。
unit.example-mod-fighter.name = 示例战机
ui.example-button = 示例按钮
message.welcome = 欢迎来到服务器！
```

### 2. 脚本中使用国际化

```javascript
// JavaScript中使用本地化文本
const localizedText = Core.bundle.get("message.welcome", "Welcome!");
Utils.broadcast(localizedText, 'green');

// 带参数的本地化
const playerCount = Groups.player.size();
const message = Core.bundle.format("message.player-count", playerCount);
```

### 3. Java代码中使用国际化

```java
// Java代码中获取本地化文本
String localizedName = Core.bundle.get("block.example-mod-advanced-drill.name");
String description = Core.bundle.get("block.example-mod-advanced-drill.description");

// 使用统计信息格式化文本
String statusMessage = Core.bundle.format("ui.status.players", Groups.player.size());
```

## 高级开发技巧

### 1. 自定义渲染

```java
public class CustomRenderer{

    public static void drawHealthBar(Building build){
        if(build.damaged()){
            float healthf = build.healthf();

            Draw.color(Color.red, Color.green, healthf);
            Fill.rect(build.x, build.y + build.block.size * 4,
                     build.block.size * 6, 2);

            Draw.color(Color.white);
            Lines.stroke(1f);
            Lines.rect(build.x, build.y + build.block.size * 4,
                      build.block.size * 6, 2);

            Draw.reset();
        }
    }

    public static void drawConnectionLines(Building from, Building to){
        Draw.color(Pal.accent, 0.6f);
        Lines.stroke(2f);
        Lines.line(from.x, from.y, to.x, to.y);

        // 绘制数据传输粒子
        float progress = (Time.time / 60f) % 1f;
        float x = Mathf.lerp(from.x, to.x, progress);
        float y = Mathf.lerp(from.y, to.y, progress);

        Draw.color(Color.white);
        Fill.circle(x, y, 2f);
        Draw.reset();
    }
}
```

### 2. 数据持久化

```java
public class ModData{
    private static final String SAVE_KEY = "example-mod-data";

    public static void saveData(String key, Object data){
        try{
            String jsonData = JsonIO.write(data);
            Core.settings.put(SAVE_KEY + "-" + key, jsonData);
        }catch(Exception e){
            Log.err("保存数据失败: " + key, e);
        }
    }

    public static <T> T loadData(String key, Class<T> type, T defaultValue){
        try{
            String jsonData = Core.settings.getString(SAVE_KEY + "-" + key, null);
            if(jsonData != null){
                return JsonIO.read(type, jsonData);
            }
        }catch(Exception e){
            Log.err("加载数据失败: " + key, e);
        }
        return defaultValue;
    }
}
```

### 3. 网络同步

```java
// 定义网络包
public class CustomPackets{

    public static void registerPackets(){
        Net.registerPacket("custom-data", CustomDataPacket::new);
    }

    public static class CustomDataPacket extends Packet{
        public String data;
        public int playerId;

        @Override
        public void write(DataOutput buffer) throws IOException{
            TypeIO.writeString(buffer, data);
            buffer.writeInt(playerId);
        }

        @Override
        public void read(DataInput buffer) throws IOException{
            data = TypeIO.readString(buffer);
            playerId = buffer.readInt();
        }

        @Override
        public void handled(){
            // 处理收到的包
            Player player = Groups.player.getByID(playerId);
            if(player != null){
                Log.info("收到来自 " + player.name + " 的数据: " + data);
            }
        }
    }
}
```

## 调试和测试

### 1. 调试工具

```java
public class DebugTools{

    public static void enableDebugMode(){
        // 启用调试信息显示
        Events.on(ClientLoadEvent.class, e -> {
            Timer.schedule(() -> {
                if(Vars.player != null && Vars.player.unit() != null){
                    drawDebugInfo();
                }
            }, 0f, 1f/60f);
        });
    }

    private static void drawDebugInfo(){
        Unit unit = Vars.player.unit();

        // 在屏幕上显示调试信息
        Draw.color(Color.yellow);
        Fonts.outline.draw("位置: " + (int)unit.x + ", " + (int)unit.y,
                          10, Vars.core.graphics.getHeight() - 10);
        Fonts.outline.draw("生命值: " + (int)unit.health + "/" + (int)unit.maxHealth,
                          10, Vars.core.graphics.getHeight() - 30);
        Draw.reset();
    }
}
```

### 2. 单元测试

```java
// 测试自定义建筑
public class DrillTest{

    @Test
    public void testDrillEfficiency(){
        // 创建测试环境
        World.load(new Tiles(10, 10));
        Tile tile = new CachedTile();
        tile.setFloor(Blocks.stone.asFloor());

        // 放置建筑
        ExampleDrill drill = new ExampleDrill("test-drill");
        Building build = drill.newBuilding();
        build.tile = tile;
        build.team = Team.sharded;

        // 测试效率计算
        build.updateTile();
        Assert.assertTrue("钻头应该正常工作", build.efficiency > 0);
    }
}
```

### 3. 性能分析

```java
public class PerformanceProfiler{
    private static ObjectMap<String, Long> timers = new ObjectMap<>();

    public static void startTimer(String name){
        timers.put(name, Time.nanos());
    }

    public static void endTimer(String name){
        Long start = timers.get(name);
        if(start != null){
            long elapsed = Time.nanos() - start;
            Log.info("计时器 [" + name + "]: " + (elapsed / 1000000f) + "ms");
        }
    }

    public static void profileMethod(String methodName, Runnable method){
        startTimer(methodName);
        try{
            method.run();
        }finally{
            endTimer(methodName);
        }
    }
}
```

## 发布和分发

### 1. MOD打包

```bash
# 创建MOD包
cd my-mod
zip -r my-mod-v1.0.zip . -x "*.git*" "*.idea*" "build/*" "*.gradle*"
```

### 2. Steam Workshop发布

```java
// 自动生成Steam Workshop预览图
public void generatePreview(){
    Fi previewFile = mod.root.child("preview.png");
    if(!previewFile.exists()){
        // 自动生成预览图
        Pixmap preview = new Pixmap(512, 512);
        // ... 绘制预览内容
        previewFile.writePng(preview);
        preview.dispose();
    }
}
```

### 3. 版本更新

```json
{
  "name": "example-mod",
  "version": "1.1.0",
  "changelog": [
    "Added new advanced drill",
    "Fixed efficiency calculation bug",
    "Improved performance"
  ]
}
```

## 最佳实践

### 1. 代码规范

```java
// 使用明确的命名
public class AdvancedDrill extends Drill {
    // 常量使用大写
    private static final float BASE_EFFICIENCY = 1.0f;

    // 使用有意义的变量名
    private float lastUpdateTime;
    private boolean isOperational;

    // 添加注释说明复杂逻辑
    @Override
    public void updateTile(){
        // 计算基于液体加成的效率
        float liquidBoost = calculateLiquidBoost();
        efficiency = BASE_EFFICIENCY * liquidBoost;

        super.updateTile();
    }
}
```

### 2. 性能优化

```java
// 缓存频繁访问的对象
private static final ObjectMap<Team, Seq<Building>> teamBuildings = new ObjectMap<>();

// 使用对象池避免垃圾回收
private static final Pool<EffectData> effectPool = Pools.get(EffectData.class, EffectData::new);

// 限制更新频率
private float updateTimer = 0f;
private static final float UPDATE_INTERVAL = 60f; // 1秒

@Override
public void updateTile(){
    updateTimer += Time.delta;
    if(updateTimer >= UPDATE_INTERVAL){
        doExpensiveUpdate();
        updateTimer = 0f;
    }
}
```

### 3. 兼容性保证

```java
// 检查API可用性
private static final boolean HAS_NEW_API =
    Reflect.hasMethod(Building.class, "newMethod");

public void useNewFeature(){
    if(HAS_NEW_API){
        // 使用新API
        ((Building)this).newMethod();
    }else{
        // 使用兼容方法
        fallbackMethod();
    }
}
```

## 故障排除

### 1. 常见错误

```java
// 错误：空指针异常
// 原因：未检查对象是否为null
// 解决：添加空值检查
if(player != null && player.unit() != null){
    player.unit().heal(amount);
}

// 错误：ClassCastException
// 原因：类型转换错误
// 解决：使用instanceof检查
if(building instanceof PowerBuilding power){
    power.power.status = 1f;
}

// 错误：依赖未找到
// 原因：MOD加载顺序问题
// 解决：正确声明依赖关系
```

### 2. 调试方法

```java
// 启用详细日志
public static void enableVerboseLogging(){
    Log.level = LogLevel.debug;

    Events.on(EventType.class, event -> {
        Log.debug("事件触发: " + event.getClass().getSimpleName());
    });
}

// 内存使用监控
public static void monitorMemory(){
    Timer.schedule(() -> {
        Runtime runtime = Runtime.getRuntime();
        long used = runtime.totalMemory() - runtime.freeMemory();
        Log.info("内存使用: " + (used / 1024 / 1024) + "MB");
    }, 0f, 10f);
}
```

## 总结

Mindustry的MOD系统为开发者提供了强大而灵活的扩展能力：

**核心优势：**
1. **多语言支持** - Java和JavaScript两种开发方式
2. **完整的API** - 涵盖游戏的所有核心系统
3. **内容定义** - 通过JSON/HJSON轻松定义游戏内容
4. **依赖管理** - 智能的依赖解析和加载顺序
5. **资源打包** - 自动的精灵图打包和优化
6. **国际化** - 完整的多语言支持

**开发建议：**
- **从简单开始** - 先尝试纯内容MOD，再进阶到脚本和Java
- **学习现有代码** - 研究游戏源码和其他优秀MOD
- **重视兼容性** - 确保MOD与不同版本的游戏兼容
- **性能优先** - 避免在主循环中进行昂贵操作
- **文档完善** - 为MOD编写清晰的使用文档

这个MOD系统为Mindustry提供了无限的扩展可能性，开发者可以创造从简单的质量改进到完整的游戏重制的各种内容。
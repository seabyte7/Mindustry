# Mindustry 扩展系统文档

## 概述

Mindustry的扩展系统是游戏高度可定制化的基础，包括Mod系统、逻辑编程系统、地图编辑器和自定义内容支持。这些系统为玩家和开发者提供了强大的创作工具，大大扩展了游戏的可玩性和生命力。

## 1. Mod系统架构

### 1.1 Mod加载器

#### 核心Mod管理器
```java
public class Mods implements Loadable {
    private Seq<LoadedMod> mods = new Seq<>();
    private Seq<ModMeta> disabled = new Seq<>();
    private ObjectMap<String, ModMeta> metadata = new ObjectMap<>();

    // 类加载器管理
    private ModClassLoader mainLoader;
    private ObjectMap<LoadedMod, ModClassLoader> loaders = new ObjectMap<>();

    public void load() {
        // 1. 扫描Mod目录
        scanMods();

        // 2. 解析依赖关系
        resolveDependencies();

        // 3. 按依赖顺序加载
        loadOrderedMods();
    }

    private void scanMods() {
        Fi modDirectory = Core.files.local("mods");

        for(Fi file : modDirectory.list()) {
            if(file.extension().equals("jar") || file.isDirectory()) {
                try {
                    ModMeta meta = loadModMeta(file);
                    metadata.put(meta.name, meta);
                } catch(Exception e) {
                    Log.err("Failed to load mod: {0}", file.name(), e);
                }
            }
        }
    }
}
```

#### Mod元数据系统
```java
public class ModMeta {
    public String name;                  // Mod名称
    public String author;                // 作者
    public String description;           // 描述
    public String version;               // 版本
    public String minGameVersion;        // 最低游戏版本

    // 依赖关系
    public Seq<String> dependencies = new Seq<>();    // 必需依赖
    public Seq<String> softDependencies = new Seq<>(); // 可选依赖

    // Java Mod特有
    public String main;                  // 主类名
    public boolean hidden = false;       // 是否隐藏

    // 脚本Mod特有
    public String script;                // 脚本文件名

    public static ModMeta parse(String content) {
        Json json = new Json();
        return json.fromJson(ModMeta.class, content);
    }
}

// mod.hjson 示例文件
/*
{
    name: "example-mod"
    author: "Developer"
    description: "An example mod"
    version: "1.0"
    minGameVersion: "126"

    dependencies: [
        "another-mod"
    ]

    # Java Mod
    main: "example.ExampleMod"

    # 或者脚本Mod
    script: "main.js"
}
*/
```

### 1.2 Java Mod系统

#### Mod主类架构
```java
public abstract class Mod {
    public ModMeta meta;                 // Mod元数据
    public Fi root;                      // Mod根目录
    public Fi file;                      // Mod文件

    // 生命周期方法
    public void init() {}                // 初始化
    public void loadContent() {}         // 加载内容
    public void postInit() {}            // 后初始化

    // 注册自定义内容
    protected void registerContent() {
        // 示例：注册自定义物品
        Items.customItem = new Item("custom-item") {{
            color = Color.purple;
            cost = 2f;
        }};

        // 示例：注册自定义建筑
        Blocks.customBlock = new Block("custom-block") {{
            requirements(Category.production, with(Items.copper, 50));
            health = 200f;
            size = 2;
        }};
    }
}

// 具体Mod实现示例
public class ExampleMod extends Mod {
    @Override
    public void init() {
        Log.info("Example Mod initialized!");

        // 注册事件监听器
        Events.on(ClientLoadEvent.class, e -> {
            Log.info("Client loaded with mod!");
        });
    }

    @Override
    public void loadContent() {
        // 注册自定义内容
        registerContent();
    }
}
```

#### 类加载器隔离
```java
public class ModClassLoader extends URLClassLoader {
    private LoadedMod mod;
    private ClassLoader parent;

    public ModClassLoader(LoadedMod mod, URL[] urls, ClassLoader parent) {
        super(urls, parent);
        this.mod = mod;
        this.parent = parent;
    }

    @Override
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        // 1. 检查是否已加载
        Class<?> clazz = findLoadedClass(name);
        if(clazz != null) return clazz;

        // 2. 尝试从当前Mod加载
        try {
            clazz = findClass(name);
            if(resolve) resolveClass(clazz);
            return clazz;
        } catch(ClassNotFoundException e) {
            // 继续尝试其他加载器
        }

        // 3. 从依赖Mod加载
        for(LoadedMod dependency : mod.dependencies) {
            try {
                clazz = dependency.loader.loadClass(name);
                if(resolve) resolveClass(clazz);
                return clazz;
            } catch(ClassNotFoundException e) {
                // 继续下一个依赖
            }
        }

        // 4. 从父加载器加载
        return super.loadClass(name, resolve);
    }
}
```

### 1.3 脚本Mod系统

#### JavaScript引擎集成
```java
public class Scripts {
    private ScriptEngine engine;         // Rhino引擎
    private Scriptable scope;            // 脚本作用域
    private ObjectMap<String, Object> bindings = new ObjectMap<>();

    public void init() {
        // 初始化JavaScript引擎
        engine = new ScriptEngineManager().getEngineByName("rhino");

        // 创建全局作用域
        scope = engine.createBindings();

        // 绑定游戏API
        bindGameAPI();
    }

    private void bindGameAPI() {
        // 核心游戏对象
        scope.put("Vars", Vars.class);
        scope.put("Core", Core.class);
        scope.put("Events", Events.class);
        scope.put("Log", Log.class);

        // 内容类型
        scope.put("Items", Items.class);
        scope.put("Blocks", Blocks.class);
        scope.put("UnitTypes", UnitTypes.class);
        scope.put("Liquids", Liquids.class);

        // 工具函数
        scope.put("print", (Consumer<Object>) obj -> Log.info(String.valueOf(obj)));
        scope.put("require", (Function<String, Object>) this::require);
    }

    public Object runScript(String script) {
        try {
            return engine.eval(script, scope);
        } catch(ScriptException e) {
            Log.err("Script execution failed", e);
            return null;
        }
    }
}

// 脚本Mod示例 (main.js)
/*
// 注册自定义物品
const customMetal = extend(Item, "custom-metal", {
    color: Color.gold,
    cost: 1.5
});

// 注册自定义建筑
const customDrill = extend(Drill, "custom-drill", {
    requirements: ItemStack.with(Items.copper, 100, customMetal, 50),
    drillTime: 200,
    tier: 3,
    size: 3
});

// 注册事件监听器
Events.on(ClientLoadEvent, () => {
    print("Custom mod loaded!");
});

// 注册自定义逻辑
customDrill.update = function(tile) {
    // 自定义更新逻辑
    if(tile.entity.efficiency() > 0.5) {
        // 特殊效果
        Effects.spark.at(tile.drawx(), tile.drawy());
    }
};
*/
```

### 1.4 内容解析器

#### JSON内容定义
```java
public class ContentParser {
    private Json json = new Json();
    private ObjectMap<Class<?>, Parser<?>> parsers = new ObjectMap<>();

    public void loadContent(Fi contentFile, LoadedMod mod) {
        JsonValue root = new JsonReader().parse(contentFile);

        for(JsonValue entry : root) {
            String type = entry.name;
            ContentType contentType = ContentType.valueOf(type);

            for(JsonValue item : entry) {
                parseContent(contentType, item, mod);
            }
        }
    }

    private void parseContent(ContentType type, JsonValue data, LoadedMod mod) {
        switch(type) {
            case item:
                parseItem(data, mod);
                break;
            case block:
                parseBlock(data, mod);
                break;
            case unit:
                parseUnit(data, mod);
                break;
            // ... 其他类型
        }
    }

    private void parseItem(JsonValue data, LoadedMod mod) {
        String name = data.getString("name");
        Item item = new Item(mod.name + "-" + name);

        // 解析属性
        item.color = Color.valueOf(data.getString("color", "ffffff"));
        item.cost = data.getFloat("cost", 1f);
        item.hardness = data.getInt("hardness", 0);
        item.explosiveness = data.getFloat("explosiveness", 0f);
        item.flammability = data.getFloat("flammability", 0f);

        // 添加到内容注册表
        content.items().add(item);
    }
}

// content.json 示例
/*
{
    "items": {
        "rare-metal": {
            "color": "gold",
            "cost": 2.5,
            "hardness": 3,
            "healthScaling": 0.3
        }
    },

    "blocks": {
        "advanced-drill": {
            "type": "Drill",
            "size": 3,
            "requirements": [
                { "item": "copper", "amount": 150 },
                { "item": "rare-metal", "amount": 75 }
            ],
            "drillTime": 150,
            "tier": 4,
            "powerUse": 2.5
        }
    }
}
*/
```

## 2. 逻辑编程系统

### 2.1 逻辑处理器

#### 逻辑指令系统
```java
public abstract class LInstruction {
    // 指令执行
    public abstract void run(LExecutor exec);

    // 指令类型
    public static class PrintInstruction extends LInstruction {
        public LAccess target;           // 目标变量
        public String message;           // 消息内容

        public void run(LExecutor exec) {
            Object value = exec.var(target);
            exec.textBuffer.append(message).append(value).append("\n");
        }
    }

    public static class SetInstruction extends LInstruction {
        public String result;            // 结果变量
        public String value;             // 值

        public void run(LExecutor exec) {
            exec.setVar(result, exec.getVar(value));
        }
    }

    public static class OperationInstruction extends LInstruction {
        public String result;            // 结果变量
        public String a, b;              // 操作数
        public LOperation op;            // 操作类型

        public void run(LExecutor exec) {
            double va = exec.num(a);
            double vb = exec.num(b);
            double result = op.function.get(va, vb);
            exec.setVar(this.result, result);
        }
    }

    public static class JumpInstruction extends LInstruction {
        public int address;              // 跳转地址
        public LCondition condition;     // 跳转条件
        public String a, b;              // 比较值

        public void run(LExecutor exec) {
            if(condition.function.get(exec.num(a), exec.num(b))) {
                exec.counter = address;
            }
        }
    }
}
```

#### 逻辑执行器
```java
public class LExecutor {
    public LInstruction[] instructions;  // 指令数组
    public int counter = 0;             // 程序计数器
    public ObjectMap<String, Object> vars = new ObjectMap<>(); // 变量表
    public StringBuilder textBuffer = new StringBuilder(); // 文本缓冲

    // 内置变量
    public Building building;           // 关联建筑
    public int linkCount;              // 链接数量
    public Building[] links;           // 链接的建筑

    public void run(int maxInstructions) {
        int executed = 0;

        while(counter < instructions.length && executed < maxInstructions) {
            LInstruction instruction = instructions[counter];

            try {
                instruction.run(this);
                counter++;
                executed++;
            } catch(Exception e) {
                Log.err("Logic execution error at {0}: {1}", counter, e.getMessage());
                break;
            }
        }

        // 重置计数器
        if(counter >= instructions.length) {
            counter = 0;
        }
    }

    public Object getVar(String name) {
        // 内置变量
        switch(name) {
            case "@this": return building;
            case "@thisx": return building.tileX();
            case "@thisy": return building.tileY();
            case "@time": return Time.time;
            case "@tick": return state.tick;
            case "@links": return linkCount;
            default: return vars.get(name, 0);
        }
    }

    public void setVar(String name, Object value) {
        vars.put(name, value);
    }

    public double num(String name) {
        Object obj = getVar(name);
        if(obj instanceof Number) {
            return ((Number) obj).doubleValue();
        }
        return 0;
    }
}
```

### 2.2 逻辑建筑

#### 逻辑处理器建筑
```java
public class LogicBlock extends Block {
    public int maxInstructionLength = 1000;    // 最大指令数
    public int instructionsPerTick = 2;        // 每tick执行指令数

    public class LogicBuild extends Building {
        public LExecutor executor = new LExecutor();
        public String code = "";             // 源代码
        public boolean privileged = false;   // 特权模式

        @Override
        public void updateTile() {
            // 执行逻辑指令
            if(executor.instructions != null) {
                executor.run(instructionsPerTick);
            }

            // 处理文本输出
            if(executor.textBuffer.length() > 0) {
                String text = executor.textBuffer.toString();
                executor.textBuffer.setLength(0);

                // 显示文本
                Call.logicDisplay(this, text);
            }
        }

        public void updateCode(String newCode) {
            this.code = newCode;

            // 编译代码为指令
            try {
                LAssembler assembler = new LAssembler();
                executor.instructions = assembler.assemble(newCode);
                executor.counter = 0;
            } catch(Exception e) {
                Log.err("Logic compilation failed: {0}", e.getMessage());
            }
        }

        // 获取链接的建筑
        public Building link(int index) {
            if(index < 0 || index >= executor.links.length) {
                return null;
            }
            return executor.links[index];
        }
    }
}
```

#### 逻辑显示器
```java
public class LogicDisplay extends Block {
    public int displayBuffer = 256;         // 显示缓冲区大小

    public class LogicDisplayBuild extends Building {
        public LCanvas canvas = new LCanvas(); // 画布
        public FrameBuffer buffer;           // 帧缓冲

        // 绘制指令
        public void draw(LGraphics graphics) {
            canvas.draw(graphics);
        }

        // 逻辑绘制API
        public void clear(Color color) {
            canvas.clear(color);
        }

        public void color(Color color) {
            canvas.setColor(color);
        }

        public void rect(float x, float y, float width, float height) {
            canvas.rect(x, y, width, height);
        }

        public void line(float x1, float y1, float x2, float y2) {
            canvas.line(x1, y1, x2, y2);
        }

        public void text(String text, float x, float y) {
            canvas.text(text, x, y);
        }
    }
}

// 逻辑代码示例
/*
print "Starting automated mining system"
set mineX 100
set mineY 150

loop:
    sensor isDrilling drill1 @enabled
    jump endloop equal isDrilling true

    control enabled drill1 true
    wait 2

    sensor items drill1 @totalItems
    jump transfer greaterThan items 50

    jump loop always

transfer:
    control enabled drill1 false
    control shootp sorter1 mineX mineY 1
    set items 0
    jump loop always

endloop:
    print "Mining complete"
    end
*/
```

### 2.3 世界处理器

#### 全局逻辑系统
```java
public class WorldProcessor extends Block {
    public class WorldProcessorBuild extends Building {
        public LExecutor executor = new LExecutor();
        public boolean globalAccess = true;  // 全局访问权限

        @Override
        public void updateTile() {
            // 世界处理器可以访问全局状态
            executor.setVar("@waveNumber", state.wave);
            executor.setVar("@enemyCount", state.enemies);
            executor.setVar("@coreItems", player.core().items.total());

            executor.run(instructionsPerTick);
        }

        // 全局控制功能
        public void controlWave() {
            if(state.rules.waveTimer) {
                state.wavetime = 0; // 触发下一波
            }
        }

        public void setGameRule(String rule, Object value) {
            // 动态修改游戏规则
            switch(rule) {
                case "unitCap":
                    state.rules.unitCap = ((Number) value).intValue();
                    break;
                case "buildSpeed":
                    state.rules.buildSpeedMultiplier = ((Number) value).floatValue();
                    break;
                // ... 其他规则
            }
        }
    }
}
```

## 3. 地图编辑器系统

### 3.1 编辑器核心

#### 地图编辑器主类
```java
public class MapEditor implements ApplicationListener {
    public EditorTiles tiles;            // 编辑器瓦片系统
    public Seq<EditorTool> tools;        // 编辑工具
    public EditorTool currentTool;       // 当前工具

    // 编辑操作栈
    public OperationStack stack = new OperationStack();

    // 编辑器状态
    public boolean grid = true;          // 显示网格
    public boolean names = false;        // 显示名称
    public int brushSize = 1;            // 笔刷大小

    public void render() {
        // 渲染地图
        renderer.render(tiles);

        // 渲染工具预览
        if(currentTool != null) {
            currentTool.drawPreview();
        }

        // 渲染UI
        ui.render();
    }

    public void handleInput() {
        if(Core.input.justTouched()) {
            Vec2 pos = toWorldCoords(Core.input.mouseX(), Core.input.mouseY());

            if(currentTool != null) {
                DrawOperation op = currentTool.createOperation(pos);
                if(op != null) {
                    stack.add(op);
                    op.operate();
                }
            }
        }
    }
}
```

#### 编辑工具系统
```java
public abstract class EditorTool {
    public String name;                  // 工具名称
    public KeyCode hotkey;               // 快捷键

    // 工具操作
    public abstract DrawOperation createOperation(Vec2 position);
    public abstract void drawPreview();

    // 画笔工具
    public static class BrushTool extends EditorTool {
        public Block block;              // 要绘制的建筑
        public int size = 1;             // 笔刷大小

        public DrawOperation createOperation(Vec2 pos) {
            return new BlockOperation((int) pos.x, (int) pos.y, block, size);
        }

        public void drawPreview() {
            Vec2 mouse = Core.input.mouseWorld();
            int x = (int) mouse.x;
            int y = (int) mouse.y;

            // 绘制笔刷预览
            Draw.color(Color.white, 0.5f);
            for(int dx = -size/2; dx <= size/2; dx++) {
                for(int dy = -size/2; dy <= size/2; dy++) {
                    Draw.rect(block.region, (x + dx) * tilesize, (y + dy) * tilesize, tilesize, tilesize);
                }
            }
        }
    }

    // 填充工具
    public static class FillTool extends EditorTool {
        public Block block;

        public DrawOperation createOperation(Vec2 pos) {
            EditorTile target = editor.tiles.get((int) pos.x, (int) pos.y);
            return new FloodFillOperation((int) pos.x, (int) pos.y, target.block(), block);
        }
    }

    // 选择工具
    public static class SelectTool extends EditorTool {
        public Rect selection = new Rect();
        private boolean selecting = false;
        private Vec2 start = new Vec2();

        public DrawOperation createOperation(Vec2 pos) {
            if(!selecting) {
                start.set(pos);
                selecting = true;
                return null;
            } else {
                selection.set(start.x, start.y, pos.x - start.x, pos.y - start.y);
                selecting = false;
                return new SelectOperation(selection);
            }
        }
    }
}
```

### 3.2 操作系统

#### 操作栈（撤销/重做）
```java
public class OperationStack {
    private Seq<DrawOperation> operations = new Seq<>();
    private int index = -1;
    private final int maxSize = 100;

    public void add(DrawOperation operation) {
        // 移除当前位置之后的操作
        while(operations.size > index + 1) {
            operations.removeIndex(operations.size - 1);
        }

        // 添加新操作
        operations.add(operation);
        index++;

        // 限制栈大小
        if(operations.size > maxSize) {
            operations.removeIndex(0);
            index--;
        }
    }

    public boolean canUndo() {
        return index >= 0;
    }

    public boolean canRedo() {
        return index < operations.size - 1;
    }

    public void undo() {
        if(canUndo()) {
            operations.get(index).undo();
            index--;
        }
    }

    public void redo() {
        if(canRedo()) {
            index++;
            operations.get(index).operate();
        }
    }
}
```

#### 绘制操作
```java
public abstract class DrawOperation {
    public abstract void operate();     // 执行操作
    public abstract void undo();       // 撤销操作

    // 方块操作
    public static class BlockOperation extends DrawOperation {
        public int x, y;
        public Block newBlock, oldBlock;
        public int size;

        public BlockOperation(int x, int y, Block block, int size) {
            this.x = x;
            this.y = y;
            this.newBlock = block;
            this.size = size;

            // 记录原有状态
            this.oldBlock = editor.tiles.get(x, y).block();
        }

        public void operate() {
            for(int dx = -size/2; dx <= size/2; dx++) {
                for(int dy = -size/2; dy <= size/2; dy++) {
                    editor.tiles.set(x + dx, y + dy, newBlock);
                }
            }
        }

        public void undo() {
            for(int dx = -size/2; dx <= size/2; dx++) {
                for(int dy = -size/2; dy <= size/2; dy++) {
                    editor.tiles.set(x + dx, y + dy, oldBlock);
                }
            }
        }
    }

    // 洪水填充操作
    public static class FloodFillOperation extends DrawOperation {
        public int x, y;
        public Block targetBlock, newBlock;
        public Seq<Point2> affectedTiles = new Seq<>();

        public void operate() {
            floodFill(x, y, targetBlock, newBlock);
        }

        private void floodFill(int x, int y, Block target, Block replacement) {
            if(x < 0 || y < 0 || x >= editor.width() || y >= editor.height()) return;
            if(editor.tiles.get(x, y).block() != target) return;
            if(target == replacement) return;

            editor.tiles.set(x, y, replacement);
            affectedTiles.add(new Point2(x, y));

            // 递归填充相邻格子
            floodFill(x + 1, y, target, replacement);
            floodFill(x - 1, y, target, replacement);
            floodFill(x, y + 1, target, replacement);
            floodFill(x, y - 1, target, replacement);
        }

        public void undo() {
            for(Point2 point : affectedTiles) {
                editor.tiles.set(point.x, point.y, targetBlock);
            }
        }
    }
}
```

### 3.3 地图生成系统

#### 程序化生成
```java
public class MapGenerator {
    public static class GenerateInput {
        public int width = 256, height = 256;
        public int seed = 0;
        public Seq<GenerateFilter> filters = new Seq<>();
    }

    // 噪声地形生成
    public static class NoiseFilter extends GenerateFilter {
        public float scl = 40f;          // 噪声缩放
        public float threshold = 0.5f;   // 阈值
        public Block block = Blocks.stone; // 生成的建筑

        public void apply(Tiles tiles, GenerateInput input) {
            SimplexNoise noise = new SimplexNoise(input.seed);

            for(int x = 0; x < input.width; x++) {
                for(int y = 0; y < input.height; y++) {
                    float value = noise.octaveNoise2D(4, 0.6f, 1f/scl, x, y);

                    if(value > threshold) {
                        tiles.set(x, y, new Tile(x, y, Blocks.air.id, block.id));
                    }
                }
            }
        }
    }

    // 矿物分布生成
    public static class OreFilter extends GenerateFilter {
        public Block ore = Blocks.oreCopper;
        public float scl = 2f;
        public float threshold = 0.7f;

        public void apply(Tiles tiles, GenerateInput input) {
            SimplexNoise noise = new SimplexNoise(input.seed + 1);

            for(int x = 0; x < input.width; x++) {
                for(int y = 0; y < input.height; y++) {
                    Tile tile = tiles.get(x, y);
                    if(tile.floor() != Blocks.air) continue;

                    float value = noise.octaveNoise2D(3, 0.5f, 1f/scl, x, y);
                    if(value > threshold) {
                        tile.setFloor((Floor) ore);
                    }
                }
            }
        }
    }

    // 河流生成
    public static class RiverFilter extends GenerateFilter {
        public void apply(Tiles tiles, GenerateInput input) {
            // 生成河流路径
            Seq<Vec2> riverPath = generateRiverPath(input);

            // 沿路径挖掘河道
            for(Vec2 point : riverPath) {
                int radius = 3;
                for(int dx = -radius; dx <= radius; dx++) {
                    for(int dy = -radius; dy <= radius; dy++) {
                        if(dx*dx + dy*dy <= radius*radius) {
                            int x = (int) point.x + dx;
                            int y = (int) point.y + dy;

                            if(tiles.in(x, y)) {
                                tiles.set(x, y, Blocks.water);
                            }
                        }
                    }
                }
            }
        }
    }
}
```

## 4. 自定义内容支持

### 4.1 材质系统

#### 动态材质加载
```java
public class ModTextures {
    private ObjectMap<String, TextureRegion> modTextures = new ObjectMap<>();

    public void loadModTextures(LoadedMod mod) {
        Fi textureDir = mod.root.child("sprites");
        if(!textureDir.exists()) return;

        // 扫描材质文件
        textureDir.walk(file -> {
            if(file.extension().equals("png")) {
                String name = getTextureName(file, textureDir);
                Texture texture = new Texture(file);
                TextureRegion region = new TextureRegion(texture);

                modTextures.put(mod.name + "-" + name, region);
                Core.atlas.addRegion(mod.name + "-" + name, region);
            }
        });
    }

    private String getTextureName(Fi file, Fi root) {
        String path = file.path();
        String rootPath = root.path();

        // 移除根路径和扩展名
        String name = path.substring(rootPath.length() + 1);
        name = name.substring(0, name.lastIndexOf('.'));

        return name.replace('/', '-');
    }
}
```

### 4.2 音频系统

#### 自定义音效支持
```java
public class ModSounds {
    private ObjectMap<String, Sound> modSounds = new ObjectMap<>();

    public void loadModSounds(LoadedMod mod) {
        Fi soundDir = mod.root.child("sounds");
        if(!soundDir.exists()) return;

        soundDir.walk(file -> {
            if(file.extension().equals("ogg") || file.extension().equals("wav")) {
                String name = file.nameWithoutExtension();
                Sound sound = Core.audio.newSound(file);

                modSounds.put(mod.name + "-" + name, sound);

                // 注册到全局音效表
                Reflect.set(Sounds.class, mod.name + Strings.capitalize(name), sound);
            }
        });
    }
}
```

### 4.3 本地化支持

#### 多语言文本系统
```java
public class ModBundles {
    public void loadModBundles(LoadedMod mod) {
        Fi bundleDir = mod.root.child("bundles");
        if(!bundleDir.exists()) return;

        for(Fi file : bundleDir.list()) {
            if(file.extension().equals("properties")) {
                String locale = file.nameWithoutExtension();
                loadBundle(mod, locale, file);
            }
        }
    }

    private void loadBundle(LoadedMod mod, String locale, Fi file) {
        I18NBundle bundle = I18NBundle.createBundle(file, new Locale(locale));

        // 添加到全局文本表
        for(String key : bundle.getKeys()) {
            String value = bundle.get(key);
            String modKey = mod.name + "." + key;

            Core.bundle.getProperties().put(modKey, value);
        }
    }
}

// bundle_en.properties 示例
/*
block.example-mod-custom-drill.name = Advanced Drill
block.example-mod-custom-drill.description = A powerful drill that can mine rare materials.

item.example-mod-rare-metal.name = Rare Metal
item.example-mod-rare-metal.description = A precious metal used in advanced construction.

ui.mod.example-title = Example Mod Settings
ui.mod.example-description = Configure the example mod options.
*/
```

---

*本文档全面介绍了Mindustry的扩展系统，包括强大的Mod支持、逻辑编程、地图编辑和自定义内容系统，展示了游戏卓越的可扩展性和创作自由度。*
# Mindustry UI系统

## 概述

Mindustry的UI系统基于Arc框架的Scene2D构建，采用Fragment-Dialog架构模式。系统支持多分辨率适配、主题样式定制、响应式布局和多平台输入处理，为游戏提供了统一而灵活的用户界面解决方案。

## 整体架构

### UI架构层次

```
UI系统架构
├── Scene2D (Arc框架UI基础)
│   ├── Element (UI元素基类)
│   ├── Group (容器组件)
│   └── Stage (UI舞台)
├── Fragment系统 (游戏内UI组件)
│   ├── HudFragment (游戏主界面)
│   ├── MenuFragment (主菜单)
│   └── ChatFragment (聊天界面)
├── Dialog系统 (弹窗对话框)
│   ├── BaseDialog (对话框基类)
│   ├── SettingsMenuDialog (设置对话框)
│   └── ResearchDialog (科技树对话框)
├── 样式系统 (Styles)
│   ├── 按钮样式
│   ├── 文本样式
│   └── 面板样式
└── 输入处理 (InputHandler)
    ├── 键盘输入
    ├── 鼠标输入
    └── 触摸输入
```

## Fragment系统

Fragment是游戏内UI的基本组件，负责特定功能区域的界面管理。

### Fragment基类设计

```java
// Fragment没有统一基类，但都遵循相似的模式
public class SampleFragment {
    private Table container;           // 主容器
    private boolean shown = true;      // 是否显示

    public void build(Group parent) {
        // 构建UI结构
        container = new Table();
        parent.addChild(container);
        setupLayout();
    }

    public void act(float delta) {
        // 更新逻辑
        if(shown) {
            updateContent();
        }
    }

    public void toggle() {
        shown = !shown;
        container.setVisible(shown);
    }
}
```

### 核心Fragment详解

#### 1. HudFragment - 游戏主界面

HudFragment是游戏中最重要的UI组件，管理所有游戏内的界面元素：

```java
public class HudFragment {
    public PlacementFragment blockfrag = new PlacementFragment(); // 建造面板
    public CoreItemsDisplay coreItems = new CoreItemsDisplay();   // 核心物品显示
    public boolean shown = true;                                  // 是否显示HUD

    private ImageButton flip;                // 翻转按钮
    private String hudText = "";             // HUD文本
    private Table lastUnlockTable;           // 最新解锁表格

    public void build(Group parent) {
        // 构建HUD布局
        Table table = new Table();
        table.setFillParent(true);

        // 顶部信息栏
        table.top().add(buildTopBar()).growX().height(60f);
        table.row();

        // 中间游戏区域 (空白，用于游戏视窗)
        table.add().grow();
        table.row();

        // 底部控制栏
        table.bottom().add(buildBottomBar()).growX().height(80f);

        parent.addChild(table);
    }

    private Table buildTopBar() {
        Table top = new Table();

        // 左侧：波次信息、核心血量
        top.left().add(buildWaveInfo()).expandX().left();

        // 中间：资源显示
        top.add(coreItems).expandX();

        // 右侧：FPS、设置按钮
        top.right().add(buildUtilityButtons()).expandX().right();

        return top;
    }

    private Table buildBottomBar() {
        Table bottom = new Table();

        // 左侧：建造面板
        bottom.left().add(blockfrag).expandY().left();

        // 右侧：小地图、单位信息
        bottom.right().add(buildRightPanel()).expandY().right();

        return bottom;
    }

    private Table buildWaveInfo() {
        Table wave = new Table();

        // 波次计数器
        wave.add(new Label(() -> {
            if(!state.rules.waves) return "";
            return Core.bundle.format("waves.wave", state.wave);
        })).pad(8f);

        // 波次进度条
        wave.row();
        wave.add(new Bar(() -> {
            return state.rules.waves ? state.wavetime / state.rules.waveSpacing : 0f;
        })).width(100f).height(20f);

        return wave;
    }
}
```

**HudFragment的核心功能：**
- **资源显示**：实时显示核心中的物品数量
- **波次信息**：显示当前波次和倒计时
- **建造界面**：集成PlacementFragment进行建筑放置
- **系统信息**：FPS显示、网络状态等

#### 2. PlacementFragment - 建造面板

```java
public class PlacementFragment {
    private Table blockTable;                // 方块选择表格
    private Block selectedBlock;              // 当前选中方块
    private int selectedCategory = 0;         // 选中的分类

    public void build(Group parent) {
        Table main = new Table();

        // 分类选择器
        Table categories = new Table();
        for(Category category : Category.all) {
            ImageButton button = new ImageButton(category.icon, Styles.cleari);
            button.clicked(() -> selectCategory(category));
            categories.add(button).size(50f);
        }

        main.add(categories).growX().height(50f);
        main.row();

        // 方块选择区域
        ScrollPane pane = new ScrollPane(blockTable = new Table());
        main.add(pane).grow();

        parent.addChild(main);
        rebuildBlocks();
    }

    private void selectCategory(Category category) {
        selectedCategory = category.ordinal();
        rebuildBlocks();
    }

    private void rebuildBlocks() {
        blockTable.clear();

        // 获取当前分类的所有方块
        Seq<Block> blocks = Vars.content.blocks()
            .select(b -> b.category == Category.all[selectedCategory] && b.unlocked());

        // 按网格布局显示方块
        int cols = 6;
        int i = 0;
        for(Block block : blocks) {
            ImageButton button = new ImageButton(new TextureRegionDrawable(block.uiIcon), Styles.selecti);
            button.clicked(() -> selectBlock(block));
            button.update(() -> button.setChecked(selectedBlock == block));

            blockTable.add(button).size(48f);
            if(++i % cols == 0) blockTable.row();
        }
    }

    private void selectBlock(Block block) {
        selectedBlock = block;
        // 通知输入处理器更新选中方块
        control.input.block = block;
    }
}
```

#### 3. ChatFragment - 聊天系统

```java
public class ChatFragment {
    private Table chatTable;                 // 聊天消息表格
    private TextField chatField;             // 聊天输入框
    private ScrollPane chatPane;             // 聊天滚动面板
    private Seq<ChatMessage> messages = new Seq<>(); // 消息列表

    public void build(Group parent) {
        Table main = new Table();
        main.top().left();

        // 聊天消息显示区域
        chatTable = new Table();
        chatTable.top().left().defaults().left().growX().pad(4f);

        chatPane = new ScrollPane(chatTable, Styles.noBarPane);
        chatPane.setFadeScrollBars(false);
        main.add(chatPane).width(400f).height(200f);
        main.row();

        // 聊天输入框
        chatField = new TextField("", Styles.defaultField);
        chatField.setMessageText("@chat.type");
        chatField.typed(character -> {
            if(character == '\n' || character == '\r') {
                sendMessage();
            }
        });

        main.add(chatField).growX().height(40f);
        parent.addChild(main);
    }

    public void addMessage(String message, String sender) {
        messages.add(new ChatMessage(message, sender, Time.millis()));

        // 限制消息数量
        if(messages.size > 100) {
            messages.removeIndex(0);
        }

        rebuildChat();
    }

    private void rebuildChat() {
        chatTable.clear();

        for(ChatMessage msg : messages) {
            Label label = new Label(msg.format());
            label.setWrap(true);
            chatTable.add(label).growX().left();
            chatTable.row();
        }

        // 滚动到底部
        Core.app.post(() -> chatPane.setScrollPercentY(1f));
    }

    private void sendMessage() {
        String text = chatField.getText().trim();
        if(!text.isEmpty()) {
            // 发送聊天消息
            Call.sendChatMessage(player, text);
            chatField.setText("");
        }
    }

    public static class ChatMessage {
        public String message, sender;
        public long timestamp;

        public String format() {
            return "[gray][[" + formatTime(timestamp) + "] " + sender + ":[white] " + message;
        }
    }
}
```

## Dialog系统

Dialog是模态对话框，用于设置、信息显示和用户交互。

### BaseDialog基类

```java
public class BaseDialog extends Dialog {
    protected boolean wasPaused;              // 是否之前暂停过
    protected boolean shouldPause;            // 是否应该暂停游戏

    public BaseDialog(String title, DialogStyle style) {
        super(title, style);
        setFillParent(true);
        this.title.setAlignment(Align.center);

        // 添加标题分隔线
        titleTable.row();
        titleTable.image(Tex.whiteui, Pal.accent).growX().height(3f).pad(4f);

        // 隐藏时恢复游戏状态
        hidden(() -> {
            if(shouldPause && state.isGame() && !net.active() && !wasPaused) {
                state.set(State.playing);
            }
            Sounds.back.play();
        });

        // 显示时暂停游戏
        shown(() -> {
            if(shouldPause && state.isGame() && !net.active()) {
                wasPaused = state.is(State.paused);
                state.set(State.paused);
            }
        });
    }

    // 便捷构造函数
    public BaseDialog(String title) {
        this(title, Core.scene.getStyle(DialogStyle.class));
    }

    // 创建按钮覆盖层
    protected void makeButtonOverlay() {
        clearChildren();
        add(titleTable).growX().row();
        stack(cont, buttons).grow();
        buttons.bottom();
    }

    // 添加关闭按钮
    public void addCloseButton(float width) {
        buttons.defaults().size(width, 64f);
        buttons.button("@back", Icon.left, this::hide).size(width, 64f);
        addCloseListener();
    }
}
```

### 重要Dialog实现

#### 1. SettingsMenuDialog - 设置对话框

```java
public class SettingsMenuDialog extends SettingsDialog {
    public SettingsMenuDialog() {
        super();
        shouldPause = true;

        // 游戏设置分类
        addCategory("@setting.game", Icon.settings, settings -> {
            settings.checkPref("vsync", true);
            settings.sliderPref("fpscap", 120, 15, 300, 15, i -> i + "fps");
            settings.checkPref("fullscreen", false);
            settings.sliderPref("sensitivity", 100, 10, 300, 10, i -> i + "%");
        });

        // 图形设置分类
        addCategory("@setting.graphics", Icon.image, settings -> {
            settings.checkPref("effects", true);
            settings.checkPref("bloom", true);
            settings.sliderPref("bloomlevel", 100, 0, 200, 25, i -> i + "%");
            settings.checkPref("linear", true);
        });

        // 音频设置分类
        addCategory("@setting.sound", Icon.volume, settings -> {
            settings.sliderPref("mastervolume", 100, 0, 100, 1, i -> i + "%");
            settings.sliderPref("sfxvolume", 100, 0, 100, 1, i -> i + "%");
            settings.sliderPref("musicvolume", 100, 0, 100, 1, i -> i + "%");
        });

        // 控制设置分类
        addCategory("@setting.controls", Icon.controller, settings -> {
            settings.table(controls -> {
                controls.button("@settings.controls", () -> {
                    ui.controls.show();
                }).size(200f, 50f);
            });
        });
    }

    private void addCategory(String name, Drawable icon, Cons<SettingsTable> builder) {
        content.table(button -> {
            button.button(name, icon, Styles.flatTogglet, () -> {
                rebuildSettings();
                builder.get(main);
            }).size(200f, 50f).checked(b -> current == builder);
        });

        if(all.isEmpty()) {
            current = builder;
        }
        all.add(builder);
    }
}
```

#### 2. ResearchDialog - 科技树对话框

```java
public class ResearchDialog extends BaseDialog {
    private Table nodeTable;                 // 节点表格
    private TechNode selected;               // 选中的节点

    public ResearchDialog() {
        super("@research");
        shouldPause = true;

        cont.table(table -> {
            // 左侧：科技树
            table.left().add(buildTechTree()).growY().width(600f);

            // 右侧：节点详情
            table.add(buildNodeInfo()).grow();
        });

        addCloseButton();
    }

    private Element buildTechTree() {
        nodeTable = new Table();
        ScrollPane pane = new ScrollPane(nodeTable);

        rebuildTree();
        return pane;
    }

    private void rebuildTree() {
        nodeTable.clear();

        // 绘制科技树节点
        for(TechNode root : TechTree.roots) {
            drawNode(root, nodeTable, 0, 0);
        }
    }

    private void drawNode(TechNode node, Table table, int x, int y) {
        // 创建节点按钮
        ImageButton button = new ImageButton(node.content.uiIcon, Styles.nodei);
        button.clicked(() -> selectNode(node));
        button.update(() -> {
            button.setDisabled(!node.content.unlocked());
            button.setChecked(selected == node);
        });

        table.add(button).size(64f);

        // 绘制连接线到子节点
        if(!node.children.isEmpty()) {
            table.row();
            Table children = new Table();
            for(TechNode child : node.children) {
                drawNode(child, children, x + 1, y);
            }
            table.add(children);
        }
    }

    private Element buildNodeInfo() {
        Table info = new Table();

        info.table(title -> {
            title.add(new Label(() -> selected != null ? selected.content.localizedName : "")).style(Styles.techLabel);
        }).growX().pad(8f);

        info.row();
        info.image().growX().height(4f).color(Pal.accent);
        info.row();

        // 节点描述
        info.table(desc -> {
            desc.add(new Label(() -> {
                if(selected == null) return "@none.selected";
                return selected.content.description;
            })).wrap().growX();
        }).grow().pad(8f);

        // 研究按钮
        info.row();
        info.button("@research", () -> {
            if(selected != null && !selected.content.unlocked()) {
                research(selected);
            }
        }).disabled(b -> selected == null || selected.content.unlocked()).size(200f, 50f);

        return info;
    }

    private void research(TechNode node) {
        // 检查研究条件
        if(!canResearch(node)) return;

        // 消耗研究材料
        for(ItemStack stack : node.requirements) {
            player.core().items.remove(stack.item, stack.amount);
        }

        // 解锁内容
        node.content.unlock();
        Events.fire(new ResearchEvent(node.content));

        rebuildTree();
    }
}
```

## 样式系统

Styles类定义了所有UI组件的外观样式：

```java
@StyleDefaults
public class Styles {
    // 基础样式
    public static Drawable black, black9, grayPanel, none, flatDown, flatOver;

    // 按钮样式
    public static TextButtonStyle
        defaultt,        // 默认文本按钮 - 灰色圆角
        flatt,          // 平面按钮 - 方形不透明
        grayt,          // 灰色按钮 - 方形灰色
        flatTogglet,    // 平面切换按钮
        togglet,        // 切换按钮
        cleart;         // 透明按钮

    // 图片按钮样式
    public static ImageButtonStyle
        defaulti,       // 默认图片按钮
        emptyi,         // 空背景按钮
        selecti,        // 选择按钮（有边框）
        cleari;         // 透明图片按钮

    // 其他组件样式
    public static ScrollPaneStyle defaultPane, smallPane, noBarPane;
    public static LabelStyle defaultLabel, outlineLabel, techLabel;
    public static TextFieldStyle defaultField, nodeField;
    public static DialogStyle defaultDialog, fullDialog;

    public static void load() {
        // 加载基础图形资源
        black = Tex.whiteui.tint(0f, 0f, 0f, 1f);
        grayPanel = Tex.button.tint(0.4f, 0.4f, 0.4f, 1f);

        // 创建按钮样式
        defaultt = new TextButtonStyle() {{
            font = Fonts.def;
            fontColor = Color.white;
            up = Tex.button;
            down = Tex.buttonDown;
            over = Tex.buttonOver;
        }};

        flatt = new TextButtonStyle() {{
            font = Fonts.def;
            fontColor = Color.white;
            up = Tex.flatButtonUp;
            down = Tex.flatButtonDown;
        }};

        // 创建图片按钮样式
        defaulti = new ImageButtonStyle() {{
            up = Tex.button;
            down = Tex.buttonDown;
            over = Tex.buttonOver;
            imageUpColor = Color.white;
        }};

        emptyi = new ImageButtonStyle() {{
            imageUpColor = Color.white;
            imageOverColor = Color.lightGray;
            imageDownColor = Color.gray;
        }};

        // 创建对话框样式
        defaultDialog = new DialogStyle() {{
            background = Tex.windowEmpty;
            titleFont = Fonts.def;
            titleFontColor = Pal.accent;
        }};
    }
}
```

## 输入处理系统

InputHandler负责处理所有用户输入事件：

```java
public abstract class InputHandler implements InputProcessor, GestureListener {
    // 输入锁定条件
    public Seq<Boolp> inputLocks = Seq.with(
        () -> renderer.isCutscene(),    // 过场动画中
        () -> logicCutscene,           // 逻辑过场中
        () -> ui.hasDialog()           // 有对话框时
    );

    // 控制状态
    public boolean isBuilding = true;
    public Block block;                // 当前选中方块
    public int rotation;               // 建造朝向
    public boolean droppingItem;       // 是否在丢弃物品

    // 手势检测
    public GestureDetector detector;
    public PlaceLine line = new PlaceLine();

    @Override
    public boolean keyDown(int keycode) {
        if(inputLocks.contains(b -> b.get())) return false;

        // 处理快捷键
        if(Core.input.keyTap(Binding.menu)) {
            ui.paused.toggle();
            return true;
        }

        if(Core.input.keyTap(Binding.inventory)) {
            ui.inventory.toggle();
            return true;
        }

        // 建造快捷键
        for(int i = 0; i < controlGroupBindings.length; i++) {
            if(Core.input.keyTap(controlGroupBindings[i])) {
                selectBlock(i);
                return true;
            }
        }

        return false;
    }

    @Override
    public boolean touchDown(int screenX, int screenY, int pointer, int button) {
        if(inputLocks.contains(b -> b.get())) return false;

        Vec2 world = Core.input.mouseWorld(screenX, screenY);

        if(button == KeyCode.mouseLeft) {
            return handleLeftClick(world.x, world.y);
        } else if(button == KeyCode.mouseRight) {
            return handleRightClick(world.x, world.y);
        }

        return false;
    }

    protected boolean handleLeftClick(float x, float y) {
        if(block != null) {
            // 建造方块
            return tryPlaceBlock(x, y);
        } else {
            // 选择单位或建筑
            return trySelect(x, y);
        }
    }

    protected boolean handleRightClick(float x, float y) {
        if(block != null) {
            // 取消建造
            block = null;
            return true;
        } else {
            // 单位移动或攻击指令
            return tryCommand(x, y);
        }
    }

    // 方块选择
    private void selectBlock(int index) {
        Category category = Category.all[index];
        Seq<Block> blocks = content.blocks().select(b ->
            b.category == category && b.unlocked() && b.synthetic());

        if(!blocks.isEmpty()) {
            block = blocks.first();
            ui.hudfrag.blockfrag.selectBlock(block);
        }
    }

    // 建造尝试
    private boolean tryPlaceBlock(float x, float y) {
        int tileX = World.toTile(x);
        int tileY = World.toTile(y);

        Tile tile = world.tile(tileX, tileY);
        if(tile != null && block.canPlaceOn(tile, player.team())) {
            // 发送建造请求
            Call.buildBlock(player, tileX, tileY, block, rotation);
            return true;
        }

        return false;
    }
}
```

### 平台特定输入处理

#### 1. 桌面平台输入

```java
public class DesktopInput extends InputHandler {
    @Override
    public boolean scrolled(float amountX, float amountY) {
        if(inputLocks.contains(b -> b.get())) return false;

        // 缩放控制
        if(amountY > 0) {
            renderer.scaleCamera(0.9f);
        } else {
            renderer.scaleCamera(1.1f);
        }
        return true;
    }

    @Override
    public boolean mouseMoved(int screenX, int screenY) {
        // 更新鼠标世界坐标
        mouseWorldPos.set(Core.input.mouseWorld(screenX, screenY));

        // 更新建造预览
        if(block != null) {
            updateBuildPreview();
        }

        return false;
    }

    private void updateBuildPreview() {
        int tileX = World.toTile(mouseWorldPos.x);
        int tileY = World.toTile(mouseWorldPos.y);

        Tile tile = world.tile(tileX, tileY);
        if(tile != null) {
            // 检查是否可以建造
            boolean canPlace = block.canPlaceOn(tile, player.team());
            ui.hudfrag.blockfrag.setPlaceValid(canPlace);
        }
    }
}
```

#### 2. 移动平台输入

```java
public class MobileInput extends InputHandler {
    private boolean zooming = false;
    private float lastZoomDistance = 0f;

    @Override
    public boolean zoom(float initialDistance, float distance) {
        if(inputLocks.contains(b -> b.get())) return false;

        if(!zooming) {
            zooming = true;
            lastZoomDistance = initialDistance;
        }

        float ratio = distance / lastZoomDistance;
        renderer.scaleCamera(ratio);
        lastZoomDistance = distance;

        return true;
    }

    @Override
    public boolean pan(float x, float y, float deltaX, float deltaY) {
        if(inputLocks.contains(b -> b.get())) return false;

        // 摄像机平移
        renderer.panCamera(-deltaX, -deltaY);
        return true;
    }

    @Override
    public boolean tap(float x, float y, int count, int button) {
        if(inputLocks.contains(b -> b.get())) return false;

        Vec2 world = Core.input.mouseWorld(x, y);

        if(count == 1) {
            return handleSingleTap(world.x, world.y);
        } else if(count == 2) {
            return handleDoubleTap(world.x, world.y);
        }

        return false;
    }

    private boolean handleDoubleTap(float x, float y) {
        // 双击缩放到目标位置
        renderer.panCameraTo(x, y);
        renderer.scaleCamera(1.5f);
        return true;
    }
}
```

## 响应式布局系统

Mindustry的UI支持多分辨率适配：

```java
public class ResponsiveLayout {
    public static Table createResponsiveLayout() {
        Table main = new Table();
        main.setFillParent(true);

        // 根据屏幕大小调整布局
        if(mobile) {
            createMobileLayout(main);
        } else {
            createDesktopLayout(main);
        }

        return main;
    }

    private static void createMobileLayout(Table main) {
        // 移动端布局：垂直堆叠
        main.defaults().growX().pad(4f);

        main.add(createTopBar()).height(80f);
        main.row();
        main.add(createGameView()).grow();
        main.row();
        main.add(createBottomControls()).height(120f);
    }

    private static void createDesktopLayout(Table main) {
        // 桌面布局：侧边栏 + 主区域
        main.add(createSidebar()).growY().width(250f);
        main.add(createMainArea()).grow();
    }

    // 动态字体大小
    public static float getScaledFontSize() {
        float baseSize = 18f;
        float scale = Math.min(Core.graphics.getWidth() / 1920f,
                              Core.graphics.getHeight() / 1080f);
        return baseSize * Mathf.clamp(scale, 0.8f, 1.5f);
    }

    // 动态按钮大小
    public static float getButtonSize() {
        if(mobile) {
            return 64f * (Core.graphics.getDensity() / 160f);
        } else {
            return 48f;
        }
    }
}
```

## 国际化支持

UI文本支持多语言：

```java
public class LocalizationExample {
    public void buildLocalizedUI(Table table) {
        // 使用Bundle获取本地化文本
        table.add(new Label("@wave.incoming")).pad(8f);
        table.row();

        // 带参数的本地化
        table.add(new Label(() -> {
            return Core.bundle.format("player.died", player.name);
        })).pad(8f);

        // 按钮本地化
        table.button("@settings.game", () -> {
            ui.settings.show();
        }).size(200f, 50f);

        // 动态更新的本地化标签
        table.add(new Label(() -> {
            if(state.rules.waves) {
                return Core.bundle.format("waves.wave", state.wave);
            } else {
                return Core.bundle.get("mode.sandbox");
            }
        })).update(label -> {
            // 根据游戏状态更新文本
            label.setColor(state.wave % 5 == 0 ? Color.red : Color.white);
        });
    }
}
```

## 性能优化

### UI性能优化技巧

```java
public class UIOptimization {
    // 1. 对象池化UI元素
    private static Pool<Label> labelPool = Pools.get(Label.class, Label::new);

    public Label getPooledLabel(String text) {
        Label label = labelPool.obtain();
        label.setText(text);
        return label;
    }

    public void returnLabel(Label label) {
        label.setText("");
        labelPool.free(label);
    }

    // 2. 延迟UI更新
    private Interval updateInterval = new Interval();

    public void updateUI() {
        if(updateInterval.get(0, 60f)) { // 每秒更新一次
            updateExpensiveElements();
        }
    }

    // 3. 虚拟化长列表
    public ScrollPane createVirtualizedList(Seq<Item> items) {
        Table content = new Table();
        ScrollPane pane = new ScrollPane(content);

        pane.setOverscroll(false, false);
        pane.setScrollingDisabled(true, false);

        // 只渲染可见项目
        pane.update(() -> {
            float scrollY = pane.getScrollY();
            float height = pane.getHeight();

            int startIndex = Math.max(0, (int)(scrollY / 50f) - 2);
            int endIndex = Math.min(items.size, (int)((scrollY + height) / 50f) + 2);

            content.clear();
            for(int i = startIndex; i < endIndex; i++) {
                content.add(createItemElement(items.get(i))).height(50f);
                content.row();
            }
        });

        return pane;
    }

    // 4. 批量UI更新
    private Seq<Runnable> pendingUpdates = new Seq<>();

    public void scheduleUIUpdate(Runnable update) {
        pendingUpdates.add(update);
    }

    public void flushUIUpdates() {
        for(Runnable update : pendingUpdates) {
            update.run();
        }
        pendingUpdates.clear();
    }
}
```

## 主题定制

支持UI主题的动态切换：

```java
public class ThemeSystem {
    public static class Theme {
        public Color primary, secondary, accent, background;
        public Color textPrimary, textSecondary;
        public String name;

        public Theme(String name, Color primary, Color accent) {
            this.name = name;
            this.primary = primary;
            this.accent = accent;
            this.background = primary.cpy().mul(0.2f);
            this.textPrimary = Color.white;
            this.textSecondary = Color.lightGray;
        }
    }

    public static Theme defaultTheme = new Theme("default", Pal.accent, Pal.accentBack);
    public static Theme darkTheme = new Theme("dark", Color.valueOf("2b2b2b"), Color.orange);
    public static Theme currentTheme = defaultTheme;

    public static void applyTheme(Theme theme) {
        currentTheme = theme;

        // 更新所有样式
        Styles.defaultt.fontColor = theme.textPrimary;
        Styles.defaultDialog.background = createBackground(theme.background);

        // 触发UI重建
        Events.fire(new ThemeChangeEvent(theme));
    }

    private static Drawable createBackground(Color color) {
        return Tex.whiteui.tint(color);
    }
}
```

这个UI系统为Mindustry提供了完整的用户界面解决方案，支持多平台、多分辨率、国际化和主题定制，为玩家提供了一致而高质量的用户体验。
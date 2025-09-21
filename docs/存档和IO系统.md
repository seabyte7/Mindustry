# 存档和IO系统

Mindustry拥有功能全面的存档和IO系统，支持游戏存档保存/加载、地图导入导出、对象序列化、版本兼容性管理和预览生成，为游戏的数据持久化提供了稳定可靠的解决方案。

## 核心架构设计

### 1. 存档系统架构层次

```
┌─────────────────────────────────────┐
│            Saves                    │
│         (存档管理器)                 │
├─────────────────────────────────────┤
│   SaveIO    │   MapIO               │
│  (存档IO)    │  (地图IO)             │
├─────────────────┬─────────────────────┤
│    TypeIO       │   JsonIO            │
│  (类型序列化)    │  (JSON序列化)        │
├─────────────────┼─────────────────────┤
│  SaveVersion    │   SaveMeta          │
│  (版本管理)      │   (存档元数据)       │
└─────────────────┴─────────────────────┘
```

### 2. 关键类说明

| 类名 | 文件位置 | 职责 |
|------|----------|------|
| `SaveIO` | `mindustry/io/SaveIO.java` | 核心存档读写功能，版本管理 |
| `Saves` | `mindustry/game/Saves.java` | 存档管理器，自动保存，预览系统 |
| `TypeIO` | `mindustry/io/TypeIO.java` | 对象序列化系统，处理所有游戏类型 |
| `SaveMeta` | `mindustry/io/SaveMeta.java` | 存档元数据结构 |
| `MapIO` | `mindustry/io/MapIO.java` | 地图文件读写和预览生成 |
| `JsonIO` | `mindustry/io/JsonIO.java` | JSON序列化工具，配置和规则序列化 |
| `SaveVersion` | `mindustry/io/SaveVersion.java` | 存档版本接口 |
| `SaveFileReader` | `mindustry/io/SaveFileReader.java` | 存档文件读取器 |

## 核心存档系统 (SaveIO)

### 1. 存档格式和头部结构

```java
public class SaveIO{
    /** 存档格式头部标识 */
    public static final byte[] header = {'M', 'S', 'A', 'V'};

    // 版本管理系统
    public static final IntMap<SaveVersion> versions = new IntMap<>();
    public static final Seq<SaveVersion> versionArray = Seq.with(
        new Save1(), new Save2(), new Save3(), new Save4(),
        new Save5(), new Save6(), new Save7(), new Save8()
    );

    // 获取当前写入版本
    public static SaveVersion getSaveWriter(){
        return versionArray.peek(); // 最新版本
    }
}
```

### 2. 存档写入流程

```java
// 保存存档到文件
public static void save(Fi file){
    boolean exists = file.exists();
    if(exists) file.moveTo(backupFileFor(file)); // 创建备份

    try{
        write(file);
    }catch(Throwable e){
        if(exists) backupFileFor(file).moveTo(file); // 恢复备份
        throw new RuntimeException(e);
    }
}

// 核心写入方法
public static void write(OutputStream os, StringMap tags){
    try(DataOutputStream stream = new DataOutputStream(os)){
        Events.fire(new SaveWriteEvent()); // 触发保存事件
        SaveVersion ver = getVersion();

        // 写入文件头
        stream.write(header);           // "MSAV"标识
        stream.writeInt(ver.version);   // 版本号

        // 写入存档数据
        if(tags == null){
            ver.write(stream);
        }else{
            ver.write(stream, tags);
        }
    }catch(Throwable e){
        throw new RuntimeException(e);
    }
}

// 获取存档元数据
public static SaveMeta getMeta(Fi file){
    try{
        return getMeta(getStream(file));
    }catch(Throwable e){
        Log.err(e);
        return getMeta(getBackupStream(file)); // 尝试备份文件
    }
}

public static SaveMeta getMeta(DataInputStream stream){
    try{
        readHeader(stream);                    // 验证文件头
        int version = stream.readInt();        // 读取版本
        SaveVersion ver = versions.get(version);

        if(ver == null) throw new IOException("Unknown save version: " + version);

        SaveMeta meta = ver.getMeta(stream);   // 读取元数据
        stream.close();
        return meta;
    }catch(IOException e){
        throw new RuntimeException(e);
    }
}
```

### 3. 存档加载流程

```java
// 加载存档
public static void load(Fi file, WorldContext context) throws SaveException{
    try{
        load(new InflaterInputStream(file.read(bufferSize)), context);
    }catch(SaveException e){
        Log.err(e);
        Fi backup = file.sibling(file.name() + "-backup." + file.extension());
        if(backup.exists()){
            load(new InflaterInputStream(backup.read(bufferSize)), context);
        }else{
            throw new SaveException(e.getCause());
        }
    }
}

// 核心加载方法
public static void load(InputStream is, WorldContext context) throws SaveException{
    try(CounterInputStream counter = new CounterInputStream(is);
        DataInputStream stream = new DataInputStream(counter)){

        logic.reset();                        // 重置游戏逻辑
        readHeader(stream);                   // 验证文件头
        int version = stream.readInt();       // 读取版本
        SaveVersion ver = versions.get(version);

        if(ver == null) throw new IOException("Unknown save version: " + version);

        ver.read(stream, counter, context);   // 读取存档数据
        Events.fire(new SaveLoadEvent(context.isMap()));
    }catch(Throwable e){
        throw new SaveException(e);
    }finally{
        world.setGenerating(false);
        content.setTemporaryMapper(null);
    }
}

// 验证存档有效性
public static boolean isSaveValid(Fi file){
    return isSaveFileValid(file) || isSaveFileValid(backupFileFor(file));
}

private static boolean isSaveFileValid(Fi file){
    try(DataInputStream stream = new DataInputStream(new InflaterInputStream(file.read(bufferSize)))){
        getMeta(stream);
        return true;
    }catch(Throwable e){
        return false;
    }
}
```

## 存档管理系统 (Saves)

### 1. 存档管理器架构

```java
public class Saves{
    Seq<SaveSlot> saves = new Seq<>();           // 存档槽列表
    @Nullable SaveSlot current;                  // 当前存档
    private @Nullable SaveSlot lastSectorSave;   // 最后的区域存档
    private boolean saving;                      // 是否正在保存
    private float time;                          // 自动保存计时器

    long totalPlaytime;                          // 总游戏时间
    private long lastTimestamp;                  // 上次时间戳
}
```

### 2. 存档槽管理

```java
public class SaveSlot{
    public final Fi file;                        // 存档文件
    boolean requestedPreview;                    // 是否请求预览
    public SaveMeta meta;                        // 存档元数据

    public SaveSlot(Fi file){
        this.file = file;
    }

    // 加载存档
    public void load() throws SaveException{
        try{
            SaveIO.load(file);
            meta = SaveIO.getMeta(file);
            current = this;
            totalPlaytime = meta.timePlayed;
            savePreview();                       // 保存预览图
        }catch(Throwable e){
            throw new SaveException(e);
        }
    }

    // 保存存档
    public void save(){
        long prev = totalPlaytime;

        SaveIO.save(file);
        meta = SaveIO.getMeta(file);
        if(state.isGame()){
            current = this;
        }

        totalPlaytime = prev;
        savePreview();                           // 更新预览图
    }

    // 生成预览图
    private void savePreview(){
        if(Core.assets.isLoaded(loadPreviewFile().path())){
            Core.assets.unload(loadPreviewFile().path());
        }
        mainExecutor.submit(() -> {
            try{
                previewFile().writePng(renderer.minimap.getPixmap());
                requestedPreview = false;
            }catch(Throwable t){
                Log.err(t);
            }
        });
    }

    // 获取预览纹理
    public Texture previewTexture(){
        if(!previewFile().exists()){
            return null;
        }else if(Core.assets.isLoaded(loadPreviewFile().path())){
            return Core.assets.get(loadPreviewFile().path());
        }else if(!requestedPreview){
            Core.assets.load(new AssetDescriptor<>(loadPreviewFile(), Texture.class));
            requestedPreview = true;
        }
        return null;
    }
}
```

### 3. 自动保存系统

```java
// 更新自动保存
public void update(){
    // 更新游戏时间
    if(current != null && state.isGame() && !(state.isPaused() && Core.scene.hasDialog())){
        if(lastTimestamp != 0){
            totalPlaytime += Time.timeSinceMillis(lastTimestamp);
        }
        lastTimestamp = Time.millis();
    }

    // 自动保存逻辑
    if(state.isGame() && !state.gameOver && current != null && current.isAutosave()){
        time += Time.delta;
        if(time > Core.settings.getInt("saveinterval") * 60 && !Vars.disableSave){
            saving = true;

            try{
                current.save();
            }catch(Throwable t){
                Log.err(t);
            }

            Time.runTask(3f, () -> saving = false);
            time = 0;
        }
    }else{
        time = 0;
    }
}

// 区域保存
public void saveSector(Sector sector){
    if(sector.save == null){
        sector.save = new SaveSlot(getSectorFile(sector));
        sector.save.setName(sector.save.file.nameWithoutExtension());
        saves.add(sector.save);
    }
    sector.save.setAutosave(true);
    sector.save.save();
    lastSectorSave = sector.save;
    Core.settings.put("last-sector-save", sector.save.getName());
}

// 获取区域文件路径
public Fi getSectorFile(Sector sector){
    return saveDirectory.child("sector-" + sector.planet.name + "-" + sector.id + "." + saveExtension);
}
```

### 4. 存档导入导出

```java
// 导入存档
public SaveSlot importSave(Fi file) throws IOException{
    SaveSlot slot = new SaveSlot(getNextSlotFile());
    slot.importFile(file);
    slot.setName(file.nameWithoutExtension());

    saves.add(slot);
    slot.meta = SaveIO.getMeta(slot.file);
    current = slot;
    return slot;
}

// 导出存档
public void exportFile(Fi to) throws IOException{
    try{
        file.copyTo(to);
    }catch(Exception e){
        throw new IOException(e);
    }
}

// 删除存档
public void delete(){
    if(SaveIO.backupFileFor(file).exists()){
        SaveIO.backupFileFor(file).delete();
    }
    file.delete();
    saves.remove(this, true);
    if(this == current){
        current = null;
    }

    if(Core.assets.isLoaded(loadPreviewFile().path())){
        Core.assets.unload(loadPreviewFile().path());
    }
}
```

## 类型序列化系统 (TypeIO)

### 1. 通用对象序列化

```java
/** 线程安全的对象序列化系统 */
@TypeIOHandler
public class TypeIO{

    // 写入对象
    public static void writeObject(Writes write, Object object){
        if(object == null){
            write.b((byte)0);
        }else if(object instanceof Integer i){
            write.b((byte)1);
            write.i(i);
        }else if(object instanceof Long l){
            write.b((byte)2);
            write.l(l);
        }else if(object instanceof Float f){
            write.b((byte)3);
            write.f(f);
        }else if(object instanceof String s){
            write.b((byte)4);
            writeString(write, s);
        }else if(object instanceof Content map){
            write.b((byte)5);
            write.b((byte)map.getContentType().ordinal());
            write.s(map.id);
        }else if(object instanceof IntSeq arr){
            write.b((byte)6);
            write.s((short)arr.size);
            for(int i = 0; i < arr.size; i++){
                write.i(arr.items[i]);
            }
        }
        // ... 更多类型处理
    }

    // 读取对象
    @Nullable
    public static Object readObject(Reads read){
        byte type = read.b();
        return switch(type){
            case 0 -> null;
            case 1 -> read.i();
            case 2 -> read.l();
            case 3 -> read.f();
            case 4 -> readString(read);
            case 5 -> content.getByID(ContentType.all[read.b()], read.s());
            case 6 -> {
                short length = read.s();
                IntSeq arr = new IntSeq(length);
                for(int i = 0; i < length; i ++) arr.add(read.i());
                yield arr;
            }
            // ... 更多类型处理
            default -> throw new IllegalArgumentException("Unknown object type: " + type);
        };
    }
}
```

### 2. 特定类型序列化

```java
// 单位序列化
public static void writeUnit(Writes write, Unit unit){
    write.b(unit == null ? 0 : unit instanceof BlockUnitc ? 1 : 2);

    if(unit instanceof BlockUnitc){
        write.i(((BlockUnitc)unit).tile().pos());
    }else if(unit == null){
        write.i(0);
    }else{
        write.i(unit.id);
    }
}

public static Unit readUnit(Reads read){
    byte type = read.b();
    int id = read.i();
    if(type == 0) return null;
    if(type == 2){ // 标准单位
        return Groups.unit.getByID(id);
    }else if(type == 1){ // 方块单位
        Building tile = world.build(id);
        return tile instanceof ControlBlock cont ? cont.unit() : null;
    }
    return null;
}

// 建筑计划序列化
public static void writePlan(Writes write, BuildPlan plan){
    write.b(plan.breaking ? (byte)1 : 0);
    write.i(Point2.pack(plan.x, plan.y));
    if(!plan.breaking){
        write.s(plan.block.id);
        write.b((byte)plan.rotation);
        write.b(1); // 总是有配置
        writeObject(write, plan.config);
    }
}

public static BuildPlan readPlan(Reads read){
    byte type = read.b();
    int position = read.i();

    if(world.tile(position) == null){
        return null;
    }

    if(type == 1){ // 拆除
        return new BuildPlan(Point2.x(position), Point2.y(position));
    }else{ // 建造
        short block = read.s();
        byte rotation = read.b();
        boolean hasConfig = read.b() == 1;
        Object config = readObject(read);
        BuildPlan current = new BuildPlan(Point2.x(position), Point2.y(position),
                                         rotation, content.block(block));
        if(hasConfig){
            current.config = config;
        }
        return current;
    }
}

// 单位控制器序列化
public static void writeController(Writes write, UnitController control){
    if(control instanceof Player p){
        write.b(0);
        write.i(p.id);
    }else if(control instanceof LogicAI logic && logic.controller != null){
        write.b(3);
        write.i(logic.controller.pos());
    }else if(control instanceof CommandAI ai){
        write.b(8);
        write.bool(ai.attackTarget != null);
        write.bool(ai.targetPos != null);

        if(ai.targetPos != null){
            write.f(ai.targetPos.x);
            write.f(ai.targetPos.y);
        }
        if(ai.attackTarget != null){
            write.b(ai.attackTarget instanceof Building ? 1 : 0);
            if(ai.attackTarget instanceof Building b){
                write.i(b.pos());
            }else{
                write.i(((Unit)ai.attackTarget).id);
            }
        }
        write.b(ai.command == null ? -1 : ai.command.id);

        // 命令队列
        write.b(ai.commandQueue.size);
        for(var pos : ai.commandQueue){
            if(pos instanceof Building b){
                write.b(0);
                write.i(b.pos());
            }else if(pos instanceof Unit u){
                write.b(1);
                write.i(u.id);
            }else if(pos instanceof Vec2 v){
                write.b(2);
                write.f(v.x);
                write.f(v.y);
            }else{
                write.b(3);
            }
        }

        writeStance(write, ai.stance);
    }else{
        write.b(2);
    }
}
```

### 3. 状态效果序列化

```java
// 状态效果序列化
public static void writeStatus(Writes write, StatusEntry entry){
    write.s(entry.effect.id);
    write.f(entry.time);

    // 写入动态字段
    if(entry.effect.dynamic){
        // 使用位标记哪些字段被实际使用
        write.b(
        (entry.damageMultiplier != 1f ?     (1 << 0) : 0) |
        (entry.healthMultiplier != 1f ?     (1 << 1) : 0) |
        (entry.speedMultiplier != 1f ?      (1 << 2) : 0) |
        (entry.reloadMultiplier != 1f ?     (1 << 3) : 0) |
        (entry.buildSpeedMultiplier != 1f ? (1 << 4) : 0) |
        (entry.dragMultiplier != 1f ?       (1 << 5) : 0) |
        (entry.armorOverride >= 0f ?        (1 << 6) : 0)
        );

        if(entry.damageMultiplier != 1f) write.f(entry.damageMultiplier);
        if(entry.healthMultiplier != 1f) write.f(entry.healthMultiplier);
        if(entry.speedMultiplier != 1f) write.f(entry.speedMultiplier);
        if(entry.reloadMultiplier != 1f) write.f(entry.reloadMultiplier);
        if(entry.buildSpeedMultiplier != 1f) write.f(entry.buildSpeedMultiplier);
        if(entry.dragMultiplier != 1f) write.f(entry.dragMultiplier);
        if(entry.armorOverride >= 0f) write.f(entry.armorOverride);
    }
}

public static StatusEntry readStatus(Reads read){
    short id = read.s();
    float time = read.f();

    StatusEntry result = new StatusEntry().set(content.getByID(ContentType.status, id), time);

    if(result.effect.dynamic){
        // 读取标记位确定哪些字段被设置
        int flags = read.ub();

        if((flags & (1 << 0)) != 0) result.damageMultiplier = read.f();
        if((flags & (1 << 1)) != 0) result.healthMultiplier = read.f();
        if((flags & (1 << 2)) != 0) result.speedMultiplier = read.f();
        if((flags & (1 << 3)) != 0) result.reloadMultiplier = read.f();
        if((flags & (1 << 4)) != 0) result.buildSpeedMultiplier = read.f();
        if((flags & (1 << 5)) != 0) result.dragMultiplier = read.f();
        if((flags & (1 << 6)) != 0) result.armorOverride = read.f();
    }

    return result;
}
```

## 地图IO系统 (MapIO)

### 1. 地图文件处理

```java
public class MapIO{
    private static final int[] pngHeader = {0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A};

    // 检测是否为图像文件
    public static boolean isImage(Fi file){
        try(InputStream stream = file.read(32)){
            for(int i1 : pngHeader){
                if(stream.read() != i1){
                    return false;
                }
            }
            return true;
        }catch(IOException e){
            return false;
        }
    }

    // 创建地图对象
    public static Map createMap(Fi file, boolean custom) throws IOException{
        try(InputStream is = new InflaterInputStream(file.read(bufferSize));
            CounterInputStream counter = new CounterInputStream(is);
            DataInputStream stream = new DataInputStream(counter)){

            SaveIO.readHeader(stream);
            int version = stream.readInt();
            SaveVersion ver = SaveIO.getSaveWriter(version);
            StringMap tags = new StringMap();
            ver.region("meta", stream, counter, in -> tags.putAll(ver.readStringMap(in)));
            return new Map(file, tags.getInt("width"), tags.getInt("height"),
                          tags, custom, version, Version.build);
        }
    }

    // 写入地图文件
    public static void writeMap(Fi file, Map map) throws IOException{
        try{
            SaveIO.write(file, map.tags);
        }catch(Exception e){
            throw new IOException(e);
        }
    }
}
```

### 2. 地图预览生成

```java
// 生成地图预览图
public static Pixmap generatePreview(Map map) throws IOException{
    map.spawns = 0;
    map.teams.clear();

    try(InputStream is = new InflaterInputStream(map.file.read(bufferSize));
        CounterInputStream counter = new CounterInputStream(is);
        DataInputStream stream = new DataInputStream(counter)){

        SaveIO.readHeader(stream);
        int version = stream.readInt();
        SaveVersion ver = SaveIO.getSaveWriter(version);
        ver.region("meta", stream, counter, ver::readStringMap);

        Pixmap floors = new Pixmap(map.width, map.height);
        Pixmap walls = new Pixmap(map.width, map.height);
        int black = 255;
        int shade = Color.rgba8888(0f, 0f, 0f, 0.5f);

        CachedTile tile = new CachedTile(){
            @Override
            public void setBlock(Block type){
                super.setBlock(type);

                int c = colorFor(block(), Blocks.air, Blocks.air, team());
                if(c != black){
                    walls.setRaw(x, floors.height - 1 - y, c);
                    floors.set(x, floors.height - 1 - y + 1, shade);
                }
            }
        };

        ver.region("content", stream, counter, ver::readContentHeader);
        ver.region("preview_map", stream, counter, in -> ver.readMap(in, new WorldContext(){
            @Override public void resize(int width, int height){}
            @Override public boolean isGenerating(){return false;}
            @Override public void begin(){world.setGenerating(true);}
            @Override public void end(){world.setGenerating(false);}

            @Override
            public void onReadBuilding(){
                // 读取队伍颜色
                if(tile.build != null){
                    int c = tile.build.team.color.rgba8888();
                    int size = tile.block().size;
                    int offsetx = -(size - 1) / 2;
                    int offsety = -(size - 1) / 2;
                    for(int dx = 0; dx < size; dx++){
                        for(int dy = 0; dy < size; dy++){
                            int drawx = tile.x + dx + offsetx, drawy = tile.y + dy + offsety;
                            walls.set(drawx, floors.height - 1 - drawy, c);
                        }
                    }

                    if(tile.build.block instanceof CoreBlock){
                        map.teams.add(tile.build.team.id);
                    }
                }
            }

            @Override
            public Tile create(int x, int y, int floorID, int overlayID, int wallID){
                if(overlayID != 0){
                    floors.set(x, floors.height - 1 - y,
                              colorFor(Blocks.air, Blocks.air, content.block(overlayID), Team.derelict));
                }else{
                    floors.set(x, floors.height - 1 - y,
                              colorFor(Blocks.air, content.block(floorID), Blocks.air, Team.derelict));
                }
                if(content.block(overlayID) == Blocks.spawn){
                    map.spawns ++;
                }
                return tile;
            }
        }));

        floors.draw(walls, true);
        walls.dispose();
        return floors;
    }finally{
        content.setTemporaryMapper(null);
    }
}
```

### 3. 图像转地图功能

```java
// 从图像读取地图
public static void readImage(Pixmap pixmap, Tiles tiles){
    for(int x = 0; x < pixmap.width && x < tiles.width; x++){
        for(int y = 0; y < pixmap.height && y < tiles.height; y++){
            int color = pixmap.get(x, pixmap.height - 1 - y);
            Block floor = Blocks.stone;
            Block wall = Blocks.air;

            // 颜色到方块的映射
            if(Color.equals(color, Color.black)){
                wall = Blocks.metalWall;
            }else if(Color.equals(color, Color.blue)){
                floor = Blocks.water;
            }else if(Color.equals(color, Color.green)){
                floor = Blocks.grass;
            }else if(Color.equals(color, Color.brown)){
                floor = Blocks.dirt;
            }
            // ... 更多颜色映射

            tiles.set(x, y, new CachedTile(){
                {
                    this.floor = floor.asFloor();
                    this.block = wall;
                    this.x = (short)x;
                    this.y = (short)y;
                }
            });
        }
    }
}

// 将地图写入图像
public static Pixmap writeImage(Tiles tiles){
    Pixmap out = new Pixmap(tiles.width, tiles.height);

    for(int x = 0; x < tiles.width; x++){
        for(int y = 0; y < tiles.height; y++){
            Tile tile = tiles.get(x, y);
            Color color = getColorFor(tile.floor(), tile.block(), tile.overlay());
            out.set(x, tiles.height - 1 - y, color);
        }
    }

    return out;
}
```

## JSON序列化系统 (JsonIO)

### 1. JSON序列化配置

```java
public class JsonIO{
    private static final CustomJson jsonBase = new CustomJson();

    public static final Json json = new Json(){
        { apply(this); }

        @Override
        public void writeValue(Object value, Class knownType, Class elementType){
            if(value instanceof MappableContent c){
                try{
                    getWriter().value(c.name);
                }catch(IOException e){
                    throw new RuntimeException(e);
                }
            }else{
                super.writeValue(value, knownType, elementType);
            }
        }

        @Override
        protected String convertToString(Object object){
            if(object instanceof MappableContent c) return c.name;
            return super.convertToString(object);
        }
    };

    // 通用序列化方法
    public static String write(Object object){
        return json.toJson(object, object.getClass());
    }

    public static <T> T read(Class<T> type, String string){
        return json.fromJson(type, string.replace("io.anuke.", ""));
    }

    public static <T> T copy(T object){
        return read((Class<T>)object.getClass(), write(object));
    }
}
```

### 2. 自定义序列化器

```java
static void apply(Json json){
    json.setElementType(Rules.class, "spawns", SpawnGroup.class);
    json.setElementType(Rules.class, "loadout", ItemStack.class);

    // 颜色序列化器
    json.setSerializer(Color.class, new Serializer<>(){
        @Override
        public void write(Json json, Color object, Class knownType){
            json.writeValue(object.toString());
        }

        @Override
        public Color read(Json json, JsonValue jsonData, Class type){
            if(jsonData.isString()){
                return Color.valueOf(jsonData.asString());
            }else{
                return new Color(jsonData.asFloat());
            }
        }
    });

    // 坐标序列化器
    json.setSerializer(Point2.class, new Serializer<>(){
        @Override
        public void write(Json json, Point2 object, Class knownType){
            json.writeObjectStart();
            json.writeValue("x", object.x);
            json.writeValue("y", object.y);
            json.writeObjectEnd();
        }

        @Override
        public Point2 read(Json json, JsonValue jsonData, Class type){
            return new Point2(jsonData.getInt("x"), jsonData.getInt("y"));
        }
    });

    // 内容类型序列化器
    json.setSerializer(MappableContent.class, new Serializer<>(){
        @Override
        public void write(Json json, MappableContent object, Class knownType){
            json.writeValue(object.name);
        }

        @Override
        public MappableContent read(Json json, JsonValue jsonData, Class type){
            String name = jsonData.asString();
            MappableContent content = content.getByName(ContentType.all, name);
            if(content == null){
                Log.warn("Unknown content: '@'", name);
            }
            return content;
        }
    });
}
```

### 3. 游戏规则序列化

```java
// 写入游戏规则
public static void writeRules(Writes write, Rules rules){
    String string = JsonIO.write(rules);
    byte[] bytes = string.getBytes(charset);
    write.i(bytes.length);
    write.b(bytes);
}

// 读取游戏规则
public static Rules readRules(Reads read){
    int length = read.i();
    String string = new String(read.b(new byte[length]), charset);
    return JsonIO.read(Rules.class, string);
}

// 写入地图目标
public static void writeObjectives(Writes write, MapObjectives executor){
    String string = JsonIO.write(executor);
    byte[] bytes = string.getBytes(charset);
    write.i(bytes.length);
    write.b(bytes);
}

// 读取地图目标
public static MapObjectives readObjectives(Reads read){
    int length = read.i();
    String string = new String(read.b(new byte[length]), charset);
    return JsonIO.read(MapObjectives.class, string);
}
```

## 版本兼容性系统

### 1. 版本管理架构

```java
// 版本接口
public abstract class SaveVersion{
    public final int version;

    protected SaveVersion(int version){
        this.version = version;
    }

    public abstract void write(DataOutput stream) throws IOException;
    public abstract void read(DataInputStream stream, CounterInputStream counter, WorldContext context) throws IOException;
    public abstract SaveMeta getMeta(DataInputStream stream) throws IOException;
}

// 最新版本实现 (Save8)
public class Save8 extends SaveVersion{
    public Save8(){
        super(8);
    }

    @Override
    public void write(DataOutput stream) throws IOException{
        writeStringMap(stream, getStringMap());

        region("content", stream, this::writeContentHeader);
        region("map", stream, this::writeMap);
        region("entities", stream, this::writeEntities);
    }

    @Override
    public void read(DataInputStream stream, CounterInputStream counter, WorldContext context) throws IOException{
        StringMap map = readStringMap(stream);

        region("content", stream, counter, this::readContentHeader);
        region("map", stream, counter, in -> readMap(in, context));
        region("entities", stream, counter, this::readEntities);
    }

    @Override
    public SaveMeta getMeta(DataInputStream stream) throws IOException{
        StringMap map = readStringMap(stream);
        return new SaveMeta(version, map.getLong("saved"), map.getLong("playtime"),
                           map.getInt("build"), map.get("mapname"),
                           map.getInt("wave"), readRules(map.get("rules")), map);
    }
}
```

### 2. 版本转换和兼容

```java
// 版本注册系统
static{
    for(SaveVersion version : versionArray){
        versions.put(version.version, version);
    }
}

// 获取保存版本
public static SaveVersion getSaveWriter(){
    return versionArray.peek(); // 总是使用最新版本保存
}

public static SaveVersion getSaveWriter(int version){
    return versions.get(version); // 获取特定版本
}

// 版本兼容性检查
public static SaveMeta getMeta(DataInputStream stream){
    try{
        readHeader(stream);
        int version = stream.readInt();
        SaveVersion ver = versions.get(version);

        if(ver == null) {
            throw new IOException("Unknown save version: " + version +
                                ". Are you trying to load a save from a newer version?");
        }

        return ver.getMeta(stream);
    }catch(IOException e){
        throw new RuntimeException(e);
    }
}
```

### 3. 向后兼容处理

```java
// 旧版本支持 (LegacyIO)
public class LegacyIO{
    public static void readLegacySave(DataInputStream stream, WorldContext context) throws IOException{
        // 读取旧格式存档
        int version = stream.readInt();

        if(version == 1){
            readSave1(stream, context);
        }else if(version == 2){
            readSave2(stream, context);
        }
        // ... 更多版本处理
    }

    private static void readSave1(DataInputStream stream, WorldContext context) throws IOException{
        // Save1格式的特殊处理
        int width = stream.readInt();
        int height = stream.readInt();

        context.resize(width, height);
        context.begin();

        // 读取旧格式的瓦片数据
        for(int i = 0; i < width * height; i++){
            byte floor = stream.readByte();
            byte wall = stream.readByte();
            byte rotation = stream.readByte();
            byte team = stream.readByte();

            // 转换为新格式
            context.create(i % width, i / width, floor, 0, wall);
        }

        context.end();
    }
}
```

## 性能优化策略

### 1. 压缩和缓冲

```java
// 使用压缩流
public static DataInputStream getStream(Fi file){
    return new DataInputStream(new InflaterInputStream(file.read(bufferSize)));
}

public static void write(Fi file, StringMap tags){
    write(new FastDeflaterOutputStream(file.write(false, bufferSize)), tags);
}

// 缓冲区大小优化
private static final int bufferSize = 8192; // 8KB缓冲区

// 批量网络传输
public static int getMaxPlans(Queue<BuildPlan> plans){
    // 限制计划数量防止缓冲区溢出
    int used = Math.min(plans.size, 20);
    int totalLength = 0;

    // 通过检查配置长度防止缓冲区溢出
    for(int i = 0; i < used; i++){
        BuildPlan plan = plans.get(i);
        if(plan.config instanceof byte[] b){
            totalLength += b.length;
        }
        if(plan.config instanceof String b){
            totalLength += b.length();
        }
        if(totalLength > 500){
            used = i + 1;
            break;
        }
    }

    return used;
}
```

### 2. 异步处理

```java
// 异步加载存档
public void load(){
    saves.clear();

    // 并行读取存档
    Seq<Future<SaveSlot>> futures = new Seq<>();

    for(Fi file : saveDirectory.list()){
        if(!file.name().contains("backup") && SaveIO.isSaveValid(file)){
            futures.add(mainExecutor.submit(() -> {
                SaveSlot slot = new SaveSlot(file);
                slot.meta = SaveIO.getMeta(file);
                return slot;
            }));
        }
    }

    for(var future : futures){
        try{
            saves.add(future.get());
        }catch(Exception e){
            Log.err(e);
        }
    }
}

// 异步预览生成
private void savePreview(){
    if(Core.assets.isLoaded(loadPreviewFile().path())){
        Core.assets.unload(loadPreviewFile().path());
    }
    mainExecutor.submit(() -> {
        try{
            previewFile().writePng(renderer.minimap.getPixmap());
            requestedPreview = false;
        }catch(Throwable t){
            Log.err(t);
        }
    });
}
```

### 3. 内存管理

```java
// 对象复用
public static class BuildingBox implements Boxed<Building>{
    public int pos;

    public BuildingBox(int pos){
        this.pos = pos;
    }

    @Override
    public Building unbox(){
        return world.build(pos);
    }
}

public static class UnitBox implements Boxed<Unit>{
    public int id;

    public UnitBox(int id){
        this.id = id;
    }

    @Override
    public Unit unbox(){
        return Groups.unit.getByID(id);
    }
}

// 延迟解析
@Nullable
public static Object readObjectBoxed(Reads read, boolean box){
    // ... 类型判断
    case 12 -> !box ? world.build(read.i()) : new BuildingBox(read.i());
    case 17 -> !box ? Groups.unit.getByID(read.i()) : new UnitBox(read.i());
}
```

### 4. 数据优化

```java
// 位压缩存储
public static void writeStatus(Writes write, StatusEntry entry){
    // 使用位标记减少存储空间
    write.b(
    (entry.damageMultiplier != 1f ?     (1 << 0) : 0) |
    (entry.healthMultiplier != 1f ?     (1 << 1) : 0) |
    (entry.speedMultiplier != 1f ?      (1 << 2) : 0) |
    (entry.reloadMultiplier != 1f ?     (1 << 3) : 0) |
    (entry.buildSpeedMultiplier != 1f ? (1 << 4) : 0) |
    (entry.dragMultiplier != 1f ?       (1 << 5) : 0) |
    (entry.armorOverride >= 0f ?        (1 << 6) : 0)
    );

    // 只写入非默认值
    if(entry.damageMultiplier != 1f) write.f(entry.damageMultiplier);
    // ... 其他字段
}

// 字符串池化
private static final ObjectMap<String, String> stringPool = new ObjectMap<>();

public static String intern(String str){
    if(str == null) return null;
    String existing = stringPool.get(str);
    if(existing == null){
        stringPool.put(str, str);
        return str;
    }
    return existing;
}
```

## 错误处理和恢复

### 1. 备份系统

```java
// 自动备份
public static void save(Fi file){
    boolean exists = file.exists();
    if(exists) file.moveTo(backupFileFor(file)); // 创建备份

    try{
        write(file);
    }catch(Throwable e){
        if(exists) backupFileFor(file).moveTo(file); // 恢复备份
        throw new RuntimeException(e);
    }
}

// 备份文件路径
public static Fi backupFileFor(Fi file){
    return file.sibling(file.name() + "-backup." + file.extension());
}

// 从备份恢复
public static void load(Fi file, WorldContext context) throws SaveException{
    try{
        load(new InflaterInputStream(file.read(bufferSize)), context);
    }catch(SaveException e){
        Log.err(e);
        Fi backup = file.sibling(file.name() + "-backup." + file.extension());
        if(backup.exists()){
            load(new InflaterInputStream(backup.read(bufferSize)), context);
        }else{
            throw new SaveException(e.getCause());
        }
    }
}
```

### 2. 验证和修复

```java
// 存档验证
public static boolean isSaveValid(Fi file){
    return isSaveFileValid(file) || isSaveFileValid(backupFileFor(file));
}

private static boolean isSaveFileValid(Fi file){
    try(DataInputStream stream = new DataInputStream(new InflaterInputStream(file.read(bufferSize)))){
        getMeta(stream);
        return true;
    }catch(Throwable e){
        return false;
    }
}

// 谨慎加载（检查MOD依赖）
public void cautiousLoad(Runnable run){
    Seq<String> mods = Seq.with(getMods());
    mods.removeAll(Vars.mods.getModStrings());

    if(!mods.isEmpty()){
        ui.showConfirm("@warning",
                      Core.bundle.format("mod.missing", mods.toString("\n")),
                      run);
    }else{
        run.run();
    }
}
```

### 3. 异常处理

```java
// 存档异常类
public static class SaveException extends RuntimeException{
    public SaveException(Throwable throwable){
        super(throwable);
    }
}

// 加载错误处理
try{
    SaveIO.load(file);
    meta = SaveIO.getMeta(file);
    current = this;
    totalPlaytime = meta.timePlayed;
    savePreview();
}catch(Throwable e){
    throw new SaveException(e);
}

// 文件损坏处理
saves.removeAll(s -> {
    if(s.getSector() != null && (s.getSector().id == 108 || s.getSector().id == 216)
       && s.meta.build <= 130 && s.meta.build > 0){
        s.getSector().clearInfo();
        s.file.delete();
        return true;
    }
    return false;
});
```

## 总结

Mindustry的存档和IO系统体现了企业级软件的设计标准：

**核心优势：**
1. **版本兼容性** - 完整的版本管理系统支持向后兼容
2. **数据完整性** - 自动备份和验证确保数据安全
3. **性能优化** - 压缩、异步处理和内存优化
4. **类型安全** - 强类型序列化系统避免数据错误
5. **扩展性** - 模块化设计支持新类型和功能扩展
6. **错误恢复** - 完善的错误处理和恢复机制

**设计模式：**
- **策略模式** - 不同版本的读写策略
- **工厂模式** - 版本对象的创建
- **代理模式** - BuildingBox和UnitBox延迟解析
- **模板方法模式** - SaveVersion的通用读写流程
- **单例模式** - TypeIO的全局序列化管理

这个存档系统为Mindustry提供了可靠的数据持久化能力，支持复杂的游戏状态保存和恢复，同时保持了优秀的性能和用户体验。开发者可以基于这个架构轻松扩展新的数据类型或实现自定义的存档功能。
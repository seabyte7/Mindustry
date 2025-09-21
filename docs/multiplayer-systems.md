# Mindustry 多人游戏系统文档

## 概述

Mindustry的多人游戏系统是游戏的重要组成部分，支持多种游戏模式，包括合作防御、PvP对战、共享建造等。系统采用客户端-服务器架构，具备完整的网络同步、反作弊、服务器管理等功能。

## 1. 网络架构设计

### 1.1 客户端-服务器模型

```java
// 网络架构核心组件
┌─────────────┐    TCP/UDP    ┌─────────────┐
│   客户端     │ ←────────→   │   服务器     │
│(NetClient)   │   数据包      │(NetServer)   │
│             │   同步        │             │
└─────────────┘              └─────────────┘
       ↑                            ↑
   本地游戏状态              权威游戏状态
```

#### 网络提供商抽象
```java
public interface NetProvider {
    void host(int port);               // 启动服务器
    void connect(String ip, int port); // 连接服务器
    void disconnect();                 // 断开连接
    void send(Object packet);          // 发送数据包
    void receive(NetListener listener); // 接收数据包
}

// 具体实现
public class ArcNetProvider implements NetProvider {
    private Server server;             // Kryonet服务器
    private Client client;             // Kryonet客户端

    public void host(int port) {
        server = new Server(8192, 2048) {{
            start();
            bind(port);
        }};

        // 注册数据包类型
        registerPackets(server.getKryo());

        // 设置网络监听器
        server.addListener(netListener);
    }
}
```

### 1.2 网络数据包系统

#### 数据包基类
```java
public abstract class Packet implements Streamable {
    public boolean reliable = true;    // 是否需要可靠传输
    public boolean unimportant = false; // 是否可丢弃

    // 序列化接口
    public abstract void write(Writes write);
    public abstract void read(Reads read);

    // 处理接口
    public abstract void handled();
}
```

#### 核心数据包类型
```java
// 连接相关
public class ConnectPacket extends Packet {
    public String name;                // 玩家名称
    public String uuid;                // 唯一标识
    public int version;                // 游戏版本
    public String platform;            // 平台信息
}

// 游戏状态同步
public class WorldDataBegin extends Packet {
    public int stream;                 // 数据流ID
}

public class WorldDataPacket extends Packet {
    public byte[] data;                // 世界数据块
}

// 实体同步
public class EntitySnapshot extends Packet {
    public int id;                     // 实体ID
    public float x, y;                 // 位置
    public float rotation;             // 旋转
    public byte team;                  // 队伍
    public short health;               // 生命值
}

// 建筑操作
public class BuildRequestPacket extends Packet {
    public int x, y;                   // 建造位置
    public short block;                // 建筑类型
    public byte rotation;              // 旋转方向
    public byte team;                  // 建造队伍
}
```

#### 自动生成的调用包
```java
// 使用@Remote注解自动生成网络调用
@Remote(called = Loc.server)
public static void transferInventory(Player player, Building build) {
    // 在服务器端执行物品转移
    if(build.acceptStack(player.item(), player.stack().amount, player) > 0) {
        build.handleStack(player.item(), player.stack().amount, player);
        player.clearItem();
    }
}

// 自动生成的网络包
public class TransferInventoryCallPacket extends Packet {
    public Player player;
    public Building build;

    public void handled() {
        Call.transferInventory(player, build);
    }
}
```

### 1.3 同步策略

#### 状态同步模型
```java
public class StateSynchronization {
    // 权威服务器模型
    public void serverUpdate() {
        // 1. 服务器计算权威状态
        world.update();
        logic.update();

        // 2. 收集需要同步的变化
        Seq<EntitySnapshot> snapshots = new Seq<>();
        Groups.all.each(entity -> {
            if(entity.changed()) {
                snapshots.add(entity.snapshot());
            }
        });

        // 3. 发送状态更新到所有客户端
        netServer.sendWorldData(snapshots);
    }

    // 客户端预测
    public void clientUpdate() {
        // 1. 应用本地输入预测
        if(player.isShooting()) {
            predictShooting();
        }

        // 2. 接收服务器权威状态
        if(hasServerUpdate()) {
            reconcileState();
        }

        // 3. 平滑插值显示
        interpolateVisuals();
    }
}
```

#### 增量同步优化
```java
public class DeltaSync {
    private ObjectMap<Integer, EntityState> lastStates = new ObjectMap<>();

    public void sendDelta(Entity entity) {
        EntityState current = entity.getState();
        EntityState last = lastStates.get(entity.id);

        if(last == null || !current.equals(last)) {
            // 只发送变化的数据
            EntityDelta delta = current.diff(last);
            netServer.sendToAll(delta);

            lastStates.put(entity.id, current.copy());
        }
    }
}
```

## 2. 服务器管理系统

### 2.1 NetServer核心

```java
public class NetServer implements ApplicationListener {
    // 连接管理
    public Seq<NetConnection> connections = new Seq<>();
    public ObjectMap<String, Player> players = new ObjectMap<>();

    // 服务器状态
    public boolean hosting = false;
    public int port = 6567;
    public String mapname = "Unknown";

    // 管理系统
    public Administration admins;

    public void host(int port) {
        this.port = port;

        try {
            // 启动网络服务
            provider.host(port);
            hosting = true;

            // 初始化管理系统
            admins.load();

            Log.info("Server started on port {0}", port);

        } catch(Exception e) {
            Log.err("Failed to host server", e);
            hosting = false;
        }
    }

    public void handleConnect(NetConnection connection) {
        // 验证连接
        if(connections.size >= Vars.maxConnections) {
            connection.kick("Server is full");
            return;
        }

        // 添加到连接列表
        connections.add(connection);

        // 发送世界数据
        sendWorldData(connection);

        Log.info("Player connected: {0}", connection.address);
    }
}
```

### 2.2 玩家管理

#### 玩家连接流程
```java
public class PlayerConnection {
    public void handleConnect(ConnectPacket packet) {
        // 1. 验证玩家信息
        if(!validName(packet.name)) {
            kick("Invalid name");
            return;
        }

        if(!validVersion(packet.version)) {
            kick("Version mismatch");
            return;
        }

        // 2. 检查封禁状态
        if(admins.isBanned(packet.uuid)) {
            kick("You are banned");
            return;
        }

        // 3. 创建玩家实体
        Player player = Player.create();
        player.name = packet.name;
        player.uuid = packet.uuid;
        player.con = this;

        // 4. 分配队伍
        player.team(assignTeam(player));

        // 5. 添加到游戏世界
        player.add();

        // 6. 通知其他玩家
        Call.playerConnect(player);

        Log.info("Player {0} joined team {1}", player.name, player.team().name);
    }

    public Team assignTeam(Player player) {
        if(state.rules.pvp) {
            // PvP模式：平衡队伍人数
            return Teams.get(getSmallestTeam());
        } else {
            // 合作模式：加入主队伍
            return state.rules.defaultTeam;
        }
    }
}
```

#### 玩家状态同步
```java
public class PlayerSync {
    public void updatePlayer(Player player) {
        // 收集玩家状态
        PlayerSnapshot snapshot = new PlayerSnapshot();
        snapshot.x = player.x;
        snapshot.y = player.y;
        snapshot.rotation = player.rotation;
        snapshot.shooting = player.isShooting();
        snapshot.boosting = player.isBoosting();

        // 发送给其他玩家
        netServer.sendExcept(player, snapshot);
    }

    public void handlePlayerInput(InputPacket packet) {
        Player player = packet.player;

        // 验证输入合法性
        if(!validInput(packet)) {
            return;
        }

        // 应用玩家输入
        player.setInput(packet.input);

        // 更新玩家状态
        player.update();
    }
}
```

### 2.3 权限管理系统

#### 管理员系统
```java
public class Administration {
    public Seq<PlayerInfo> admins = new Seq<>();
    public Seq<PlayerInfo> banned = new Seq<>();
    public ObjectMap<String, String> config = new ObjectMap<>();

    // 权限等级
    public enum Perm {
        kick, ban, unban, admin,
        pause, items, spawn,
        load, save, gamemode
    }

    public boolean isAdmin(String uuid, String ip) {
        return admins.contains(p ->
            p.id.equals(uuid) || p.lastIP.equals(ip));
    }

    public boolean can(Player player, Perm perm) {
        if(!isAdmin(player.uuid(), player.con.address)) {
            return false;
        }

        PlayerInfo info = getInfo(player.uuid());
        return info != null && info.admin;
    }

    public void handleAdminAction(Player admin, AdminAction action) {
        if(!can(admin, action.permission)) {
            admin.sendMessage("No permission");
            return;
        }

        // 执行管理操作
        action.execute();

        // 记录日志
        Log.info("Admin {0} executed {1}", admin.name, action.name);
    }
}
```

#### 反作弊系统
```java
public class AntiCheat {
    private ObjectMap<String, CheatData> playerData = new ObjectMap<>();

    public static class CheatData {
        public float lastX, lastY;         // 上次位置
        public long lastUpdate;            // 上次更新时间
        public int violations;             // 违规次数
        public Seq<String> reasons = new Seq<>(); // 违规原因
    }

    public void checkPlayer(Player player) {
        CheatData data = playerData.get(player.uuid(), CheatData::new);

        // 检查移动速度
        float distance = Mathf.dst(player.x, player.y, data.lastX, data.lastY);
        float deltaTime = (System.currentTimeMillis() - data.lastUpdate) / 1000f;
        float maxSpeed = player.type.speed * 1.5f; // 允许150%速度容错

        if(distance / deltaTime > maxSpeed) {
            flagPlayer(player, "Speed hacking", data);
        }

        // 检查建造速度
        if(player.buildRequests().size > 10) {
            flagPlayer(player, "Build speed hacking", data);
        }

        // 检查资源获取
        if(player.team().items().total() > lastTotal + 1000) {
            flagPlayer(player, "Item duplication", data);
        }

        // 更新数据
        data.lastX = player.x;
        data.lastY = player.y;
        data.lastUpdate = System.currentTimeMillis();
    }

    public void flagPlayer(Player player, String reason, CheatData data) {
        data.violations++;
        data.reasons.add(reason);

        Log.warn("Player {0} flagged for {1} (violations: {2})",
                player.name, reason, data.violations);

        // 自动处理
        if(data.violations >= 5) {
            player.kick("Cheating detected: " + data.reasons.toString());
            admins.ban(player.uuid(), reason);
        }
    }
}
```

## 3. 游戏模式系统

### 3.1 PvP模式

#### 队伍对抗
```java
public class PvPMode {
    public void setupPvP() {
        state.rules.pvp = true;
        state.rules.canGameOver = true;
        state.rules.coreCapture = true;    // 核心夺取胜利

        // 分配起始资源
        for(Team team : state.teams.getActive()) {
            TeamData data = state.teams.get(team);

            // 给予起始物品
            data.items.add(Items.copper, 300);
            data.items.add(Items.lead, 300);
            data.items.add(Items.graphite, 150);
        }

        // 设置胜利条件
        Events.on(BlockDestroyEvent.class, e -> {
            if(e.tile.block() instanceof CoreBlock) {
                checkPvPVictory();
            }
        });
    }

    public void checkPvPVictory() {
        Team winner = null;
        int coreTeams = 0;

        for(Team team : Team.all) {
            if(state.teams.get(team).hasCore()) {
                winner = team;
                coreTeams++;
            }
        }

        if(coreTeams <= 1) {
            // 游戏结束
            state.gameOver = true;
            state.won = (winner == player.team());

            Call.gameOver(winner);
            Log.info("PvP game ended, winner: {0}", winner.name);
        }
    }
}
```

#### 资源竞争
```java
public class ResourceCompetition {
    public void updateResourceControl() {
        // 计算各队伍控制的矿点
        ObjectMap<Team, Int> controlPoints = new ObjectMap<>();

        world.tiles.each(tile -> {
            if(tile.floor().itemDrop != null) {
                Building nearest = getNearestBuilding(tile, 80f);
                if(nearest != null) {
                    controlPoints.get(nearest.team, Int::new).value++;
                }
            }
        });

        // 给予资源奖励
        for(ObjectMap.Entry<Team, Int> entry : controlPoints) {
            TeamData data = state.teams.get(entry.key);
            int bonus = entry.value.value * 2; // 每个控制点每秒2个资源

            data.items.add(Items.copper, bonus);
        }
    }
}
```

### 3.2 合作模式

#### 共享建造
```java
public class CoopMode {
    public void setupCoop() {
        state.rules.pvp = false;
        state.rules.waves = true;
        state.rules.waveTimer = true;

        // 共享队伍
        for(Player player : Groups.player) {
            player.team(Team.sharded);
        }

        // 共享资源
        state.rules.infiniteResources = false;
        enableResourceSharing();
    }

    public void enableResourceSharing() {
        Events.on(DepositEvent.class, e -> {
            // 物品存入时自动共享
            TeamData team = e.player.team().data();
            team.items.add(e.item, e.amount);

            // 通知其他玩家
            Call.announce("[accent]" + e.player.name +
                         "[] deposited [accent]" + e.amount +
                         "[] " + e.item.localizedName);
        });
    }
}
```

#### 建造协调
```java
public class BuildCoordination {
    public void coordinateBuild(Player player, BuildPlan plan) {
        // 检查重复建造
        for(Player other : Groups.player) {
            if(other != player && other.isBuilding()) {
                BuildPlan otherPlan = other.buildPlan();
                if(plan.samePos(otherPlan)) {
                    // 冲突处理：让延迟低的玩家优先
                    if(player.con.ping < other.con.ping) {
                        other.clearBuilding();
                        other.sendMessage("Build conflict, yielding to " + player.name);
                    } else {
                        player.sendMessage("Already being built by " + other.name);
                        return;
                    }
                }
            }
        }

        // 执行建造
        player.addBuild(plan);
    }
}
```

### 3.3 攻击模式

#### 攻击目标设定
```java
public class AttackMode {
    public void setupAttack() {
        state.rules.attackMode = true;
        state.rules.waves = false;
        state.rules.unitCapVariable = false;

        // 设置攻击目标
        designateTargets();

        // 给予攻击部队
        giveAttackUnits();
    }

    public void designateTargets() {
        // 标记敌方核心为攻击目标
        for(Building build : state.teams.get(state.rules.waveTeam).buildings) {
            if(build.block instanceof CoreBlock) {
                // 添加目标标记
                addObjectiveMarker(build.tileX(), build.tileY(), "Destroy this core");
            }
        }
    }

    public void giveAttackUnits() {
        TeamData team = player.team().data();

        // 给予初始部队
        for(int i = 0; i < 5; i++) {
            Unit unit = UnitTypes.dagger.create(team.team);
            Call.unitSpawn(unit);
        }

        // 启用单位工厂
        enableUnitProduction();
    }
}
```

## 4. 网络优化

### 4.1 带宽优化

#### 数据压缩
```java
public class NetworkCompression {
    public byte[] compressWorldData(Tile[][] tiles) {
        ByteArrayOutputStream output = new ByteArrayOutputStream();

        try(DeflaterOutputStream deflater = new DeflaterOutputStream(output)) {
            DataOutputStream data = new DataOutputStream(deflater);

            // 使用增量编码
            short lastBlock = 0;
            for(Tile[] row : tiles) {
                for(Tile tile : row) {
                    short currentBlock = tile.block().id;
                    data.writeShort(currentBlock - lastBlock); // 差值编码
                    lastBlock = currentBlock;

                    // 只在需要时写入额外数据
                    if(tile.entity != null) {
                        data.writeBoolean(true);
                        tile.entity.writeSync(data);
                    } else {
                        data.writeBoolean(false);
                    }
                }
            }
        } catch(IOException e) {
            Log.err("Failed to compress world data", e);
        }

        return output.toByteArray();
    }
}
```

#### 智能更新频率
```java
public class AdaptiveSync {
    private ObjectMap<Entity, Float> updateFrequencies = new ObjectMap<>();

    public void updateEntity(Entity entity) {
        // 根据重要性调整更新频率
        float importance = calculateImportance(entity);
        float frequency = Math.max(1f, importance * 60f); // 1-60 Hz

        updateFrequencies.put(entity, frequency);

        if(entity.timer.get(1, frequency)) {
            sendEntityUpdate(entity);
        }
    }

    public float calculateImportance(Entity entity) {
        float importance = 1f;

        // 玩家单位最重要
        if(entity instanceof Player) importance = 10f;

        // 正在战斗的单位重要
        if(entity instanceof Unit && ((Unit)entity).isShooting()) {
            importance *= 3f;
        }

        // 距离玩家近的更重要
        float distance = entity.dst(player);
        importance *= Math.max(0.1f, 1f - distance / 1000f);

        return importance;
    }
}
```

### 4.2 延迟补偿

#### 客户端预测
```java
public class ClientPrediction {
    private Seq<InputSnapshot> inputHistory = new Seq<>();

    public void predictMovement(Player player, Input input) {
        // 记录输入历史
        InputSnapshot snapshot = new InputSnapshot();
        snapshot.input = input.copy();
        snapshot.tick = state.tick;
        snapshot.x = player.x;
        snapshot.y = player.y;
        inputHistory.add(snapshot);

        // 预测移动
        float dx = input.mouseX - player.x;
        float dy = input.mouseY - player.y;
        float len = Mathf.len(dx, dy);

        if(len > 0.01f) {
            dx /= len;
            dy /= len;

            float speed = player.type.speed;
            player.x += dx * speed * Time.delta;
            player.y += dy * speed * Time.delta;
        }

        // 清理旧历史
        inputHistory.removeAll(s -> state.tick - s.tick > 60);
    }

    public void reconcileWithServer(PlayerSnapshot serverState) {
        // 找到对应的输入快照
        InputSnapshot snapshot = inputHistory.find(s -> s.tick == serverState.tick);
        if(snapshot == null) return;

        // 检查预测误差
        float error = Mathf.dst(snapshot.x, snapshot.y, serverState.x, serverState.y);

        if(error > 5f) { // 误差超过阈值
            // 回滚并重新应用输入
            player.set(serverState.x, serverState.y);

            for(InputSnapshot replay : inputHistory) {
                if(replay.tick > serverState.tick) {
                    predictMovement(player, replay.input);
                }
            }
        }
    }
}
```

#### 滞后补偿
```java
public class LagCompensation {
    public void handleShot(Player shooter, Vec2 target, float lag) {
        // 回溯目标位置到射击时刻
        long shotTime = System.currentTimeMillis() - (long)(lag * 1000);

        // 查找目标在射击时的位置
        Vec2 targetPos = getHistoricalPosition(target, shotTime);

        // 使用历史位置进行命中判定
        if(Mathf.within(targetPos.x, targetPos.y, target.x, target.y, shooter.weapon.range)) {
            // 命中处理
            handleHit(shooter, target);
        }
    }

    private ObjectMap<Entity, Seq<PositionSnapshot>> positionHistory = new ObjectMap<>();

    private Vec2 getHistoricalPosition(Entity entity, long time) {
        Seq<PositionSnapshot> history = positionHistory.get(entity);
        if(history == null) return new Vec2(entity.x, entity.y);

        // 线性插值历史位置
        for(int i = 0; i < history.size - 1; i++) {
            PositionSnapshot a = history.get(i);
            PositionSnapshot b = history.get(i + 1);

            if(time >= a.time && time <= b.time) {
                float t = (time - a.time) / (float)(b.time - a.time);
                return new Vec2(
                    Mathf.lerp(a.x, b.x, t),
                    Mathf.lerp(a.y, b.y, t)
                );
            }
        }

        return new Vec2(entity.x, entity.y);
    }
}
```

---

*本文档详细描述了Mindustry的多人游戏系统，涵盖网络架构、服务器管理、游戏模式和网络优化等方面，展示了一个成熟多人游戏的技术实现。*
# Mindustry 未完成策划案 TODO 清单

## 文档说明

本文档记录了从Mindustry源代码中发现的、尚未被反推为游戏策划案的功能模块。这些系统都有完整的代码实现，但缺乏对应的策划文档分析。

---

## 🎮 **游戏平台与服务系统**

### 成就系统策划案 ❌ 未完成
**源码位置**: `service/Achievement.java`, `service/GameService.java`, `service/SStat.java`

**需要分析的内容**:
- Steam成就集成机制
- 统计数据追踪系统 (击杀敌人数、建造方块数、波数存活等)
- 跨平台成就同步机制
- 成就解锁条件设计
- 玩家激励体系设计

---

## 🌍 **星球与世界生成系统**

### 程序化世界生成策划案 ❌ 未完成
**源码位置**: `maps/planet/`, `maps/generators/`

**需要分析的内容**:
- **Serpulo星球生成算法** (`SerpuloPlanetGenerator.java`)
  - 地形噪声生成规则
  - 资源分布算法
  - 生物群落分布
- **Erekir星球生成算法** (`ErekirPlanetGenerator.java`)
  - 不同于Serpulo的地形特征
  - 独特的环境生成规则
- **小行星生成系统** (`AsteroidGenerator.java`)
  - 小行星带的生成机制
  - 资源小行星的分布规律
- **程序化地形生成规则**
  - 地形高度图生成
  - 矿物分布算法
  - 敌人基地生成规则

---

## 🔥 **热量传导系统**

### 热量机制策划案 ❌ 未完成
**源码位置**: `world/blocks/heat/`

**需要分析的内容**:
- **热量传导网络** (`HeatConductor.java`)
  - 热量在建筑间的传导机制
  - 热量损失计算
- **热量生产与消费** (`HeatProducer.java`, `HeatConsumer.java`)
  - 热量产生建筑的设计
  - 热量消费建筑的效率机制
- **热量平衡计算**
  - 热量供需平衡设计
  - 过热保护机制
  - 热量存储系统

---

## 🚀 **战役与任务系统**

### 星际战役系统策划案 ❌ 未完成
**源码位置**: `world/blocks/campaign/`

**需要分析的内容**:
- **发射台机制** (`LaunchPad.java`)
  - 资源发射到太空的机制
  - 发射成本与效率
  - 发射任务系统
- **着陆台系统** (`LandingPad.java`)
  - 从太空接收资源的机制
  - 着陆台的建造与维护
- **加速器设施** (`Accelerator.java`)
  - 粒子加速器的作用机制
  - 高级研究解锁功能
  - 能量需求与产出

---

## 🎯 **高级消费者系统**

### 复杂资源消费机制策划案 ❌ 未完成
**源码位置**: `world/consumers/`

**需要分析的内容**:
- **动态消费系统**
  - 动态液体消费 (`ConsumeLiquidsDynamic.java`)
  - 动态物品消费 (`ConsumeItemDynamic.java`)
  - 动态电力消费 (`ConsumePowerDynamic.java`)
- **条件消费机制**
  - 条件电力消费 (`ConsumePowerCondition.java`)
  - 载荷过滤器 (`ConsumePayloadFilter.java`)
- **特殊消费类型**
  - 爆炸性物品消费 (`ConsumeItemExplode.java`)
  - 易燃物品消费 (`ConsumeItemFlammable.java`)
  - 放射性物品消费 (`ConsumeItemRadioactive.java`)
  - 冷却剂消费 (`ConsumeCoolant.java`)

---

## 🌦️ **天气系统**

### 环境天气策划案 ❌ 未完成
**源码位置**: `type/weather/`

**需要分析的内容**:
- **粒子天气效果** (`ParticleWeather.java`)
  - 粒子天气的视觉效果
  - 对游戏玩法的影响
- **极端天气事件**
  - 磁暴天气 (`MagneticStorm.java`) - 对电子设备的影响
  - 太阳耀斑 (`SolarFlare.java`) - 对太阳能的影响
- **常规天气**
  - 降雨系统 (`RainWeather.java`) - 对冷却和生产的影响
- **天气预报与应对机制**

---

## ⚡ **单位能力系统**

### 特殊单位能力策划案 ❌ 未完成
**源码位置**: `entities/abilities/`

**需要分析的内容**:
- **防护类能力**
  - 力场护盾 (`ForceFieldAbility.java`) - 能量护盾机制
  - 装甲板系统 (`ArmorPlateAbility.java`) - 物理装甲强化
- **攻击类能力**
  - 移动闪电 (`MoveLightningAbility.java`) - 移动时产生闪电
  - 单位生成 (`UnitSpawnAbility.java`) - 战斗中召唤单位
- **支援类能力**
  - 能量场效果 (`EnergyFieldAbility.java`) - 范围能量增益
  - 修复场 (`RepairFieldAbility.java`) - 范围修复效果
  - 状态场效果 (`StatusFieldAbility.java`) - 范围状态效果
- **特殊机制能力**
  - 液体爆炸 (`LiquidExplodeAbility.java`) - 死亡时液体爆炸
  - 压制场 (`SuppressionFieldAbility.java`) - 减缓敌人行动

---

## 🎨 **3D渲染系统**

### 立体视觉系统策划案 ❌ 未完成
**源码位置**: `graphics/g3d/`

**需要分析的内容**:
- **星球渲染系统**
  - 星球网格渲染 (`PlanetGrid.java`)
  - 星球表面材质系统 (`PlanetMesh.java`)
  - 星球大气效果 (`PlanetRenderer.java`)
- **3D构建系统**
  - 3D网格构建 (`MeshBuilder.java`)
  - 六边形网格 (`HexMesh.java`)
  - 天空网格 (`HexSkyMesh.java`)
- **太阳系渲染**
  - 太阳网格 (`SunMesh.java`)
  - 噪声网格 (`NoiseMesh.java`)

---

## 🔄 **异步处理系统**

### 多线程架构策划案 ❌ 未完成
**源码位置**: `async/`

**需要分析的内容**:
- **异步核心系统** (`AsyncCore.java`)
  - 主线程与异步线程的协调
  - 任务调度机制
- **物理进程** (`PhysicsProcess.java`)
  - 物理计算的异步处理
  - 碰撞检测优化
- **异步任务处理** (`AsyncProcess.java`)
  - 后台任务管理
  - 性能优化策略

---

## 🎲 **沙盒调试系统**

### 开发者工具策划案 ❌ 未完成
**源码位置**: `world/blocks/sandbox/`

**需要分析的内容**:
- **资源调试工具**
  - 物品源/虚空 (`ItemSource.java`, `ItemVoid.java`)
  - 液体源/虚空 (`LiquidSource.java`, `LiquidVoid.java`)
  - 电力虚空 (`PowerVoid.java`)
- **沙盒模式设计理念**
  - 创造模式的教育价值
  - 实验环境的搭建
  - 调试工具的用户体验

---

## 🛠️ **特效与视觉系统**

### 复合特效系统策划案 ❌ 未完成
**源码位置**: `entities/effect/`

**需要分析的内容**:
- **组合特效机制**
  - 多重特效 (`MultiEffect.java`) - 同时播放多个特效
  - 序列特效 (`SeqEffect.java`) - 按顺序播放特效
- **空间特效**
  - 径向特效 (`RadialEffect.java`) - 径向扩散效果
  - 波浪特效 (`WaveEffect.java`) - 波纹传播效果
- **音频视觉结合**
  - 声音特效 (`SoundEffect.java`) - 音效与视效的结合
- **特效包装系统**
  - 包装特效 (`WrapEffect.java`) - 特效的二次包装

---

## 🔧 **内容类型系统**

### 底层内容框架策划案 ❌ 未完成
**源码位置**: `ctype/`

**需要分析的内容**:
- **内容分类系统** (`ContentType.java`)
  - 游戏内容的分类机制
  - 内容类型的扩展性设计
- **解锁机制框架** (`UnlockableContent.java`)
  - 统一的解锁系统设计
  - 前置条件检查机制
- **内容映射系统** (`MappableContent.java`)
  - 内容ID映射机制
  - 跨版本兼容性设计

---

## 📋 **策划案优先级建议**

### 🔥 高优先级 (核心gameplay)
1. **热量传导系统** - 影响中后期玩法
2. **单位能力系统** - 丰富战斗策略
3. **天气系统** - 增加环境挑战
4. **复杂消费机制** - 深化生产链设计

### 🟡 中优先级 (增强体验)
1. **成就系统** - 提升用户粘性
2. **星际战役系统** - 连接不同星球玩法
3. **程序化世界生成** - 技术架构理解
4. **特效视觉系统** - 用户体验优化

### 🔵 低优先级 (技术细节)
1. **3D渲染系统** - 主要为技术实现
2. **异步处理系统** - 性能优化相关
3. **沙盒调试系统** - 开发工具相关
4. **底层内容框架** - 架构设计相关

---

## 📝 **TODO完成标记说明**

- ❌ 未完成 - 尚未开始策划案编写
- 🟡 进行中 - 正在编写策划案
- ✅ 已完成 - 策划案已完成并审核

---

*本TODO清单将根据策划案完成情况定期更新，确保所有重要的游戏系统都得到充分的策划分析。*
# Mindustry 开发文档

欢迎来到Mindustry开发文档！这里提供了全面的技术文档，帮助您快速理解项目架构、掌握开发技能并进入开发状态。

## 📚 文档目录

### 🎯 核心架构文档

| 文档 | 描述 | 适合人群 | 重要程度 |
|------|------|----------|----------|
| [项目概述](./项目概述.md) | 项目技术栈、模块结构、设计原则 | 所有开发者 | ⭐⭐⭐⭐⭐ |
| [架构设计](./架构设计.md) | 核心模块、ApplicationListener模式、事件系统 | 架构师、核心开发者 | ⭐⭐⭐⭐⭐ |
| [实体系统和代码生成](./实体系统和代码生成.md) | ECS架构、注解处理器、自动代码生成 | 核心开发者、MOD开发者 | ⭐⭐⭐⭐ |

### 🎮 游戏系统文档

| 文档 | 描述 | 适合人群 | 重要程度 |
|------|------|----------|----------|
| [内容系统](./内容系统.md) | 游戏内容定义、科技树、MOD集成 | 内容开发者、MOD开发者 | ⭐⭐⭐⭐ |
| [世界系统](./世界系统.md) | Block/Building/Tile架构、模块系统、渲染 | 游戏逻辑开发者 | ⭐⭐⭐⭐ |
| [网络和多人游戏](./网络和多人游戏.md) | 客户端-服务器架构、同步机制、反作弊 | 网络开发者、服务器开发者 | ⭐⭐⭐ |

## 🚀 快速开始指南

### 第一次接触项目？

**推荐阅读顺序：**

1. **[项目概述](./项目概述.md)** (20分钟)
   - 了解项目整体架构和技术栈
   - 理解模块划分和职责
   - 掌握基本开发命令

2. **[架构设计](./架构设计.md)** (45分钟)
   - 深入理解六大核心模块
   - 掌握ApplicationListener模式
   - 理解事件驱动架构

3. **[实体系统和代码生成](./实体系统和代码生成.md)** (60分钟)
   - 理解ECS架构的创新设计
   - 掌握注解处理器和代码生成
   - 学会使用@EntityDef和@Remote

### 针对不同角色的学习路径

#### 🔧 **游戏功能开发者**
```
项目概述 → 内容系统 → 世界系统 → 架构设计
```
**重点关注：**
- 如何定义新的游戏内容（方块、单位、物品）
- Block-Building-Tile三层架构
- 消耗系统和模块系统
- 渲染系统和绘制流程

#### 🌐 **网络/多人游戏开发者**
```
项目概述 → 架构设计 → 实体系统和代码生成 → 网络和多人游戏
```
**重点关注：**
- @Remote注解和自动代码生成
- 快照同步机制
- 客户端-服务器架构
- 反作弊和性能优化

#### 🏗️ **架构/核心开发者**
```
架构设计 → 实体系统和代码生成 → 项目概述 → 世界系统 → 网络和多人游戏
```
**重点关注：**
- ApplicationListener生命周期模式
- ECS架构和组件组合
- 注解处理器实现
- 性能优化技术

#### 🎨 **MOD开发者**
```
项目概述 → 内容系统 → 实体系统和代码生成 → 世界系统
```
**重点关注：**
- 内容定义和注册系统
- 科技树扩展
- JavaScript脚本支持
- 组件系统扩展

## 🔍 核心概念速查

### 关键设计模式

| 模式 | 用途 | 核心类 | 文档位置 |
|------|------|--------|----------|
| **ApplicationListener** | 模块生命周期管理 | `Logic`, `Control`, `Renderer` | [架构设计](./架构设计.md#applicationlistener生命周期模式) |
| **ECS架构** | 实体组件系统 | `@EntityDef`, `@Component` | [实体系统](./实体系统和代码生成.md#ecs架构设计) |
| **事件驱动** | 模块间通信 | `Events`, `EventType` | [架构设计](./架构设计.md#事件系统设计) |
| **代码生成** | 自动化开发 | `@Remote`, 注解处理器 | [实体系统](./实体系统和代码生成.md#注解处理器系统) |

### 核心类参考

| 类别 | 核心类 | 功能 | 文档位置 |
|------|--------|------|----------|
| **全局状态** | `Vars` | 全局变量和游戏状态 | [架构设计](./架构设计.md#游戏状态管理) |
| **内容管理** | `ContentLoader`, `Blocks`, `Items` | 游戏内容定义和加载 | [内容系统](./内容系统.md) |
| **世界管理** | `Block`, `Building`, `Tile` | 世界和建筑系统 | [世界系统](./世界系统.md) |
| **网络通信** | `NetServer`, `NetClient`, `Call` | 多人游戏网络 | [网络和多人游戏](./网络和多人游戏.md) |

### 常用注解

| 注解 | 用途 | 示例 | 文档位置 |
|------|------|------|----------|
| `@EntityDef` | 定义实体类型 | `@EntityDef({Unitc.class, Mechc.class})` | [实体系统](./实体系统和代码生成.md#entitydef注解) |
| `@Component` | 定义组件 | `@Component abstract class VelComp` | [实体系统](./实体系统和代码生成.md#组件定义) |
| `@Remote` | 网络RPC调用 | `@Remote(called = Loc.server)` | [网络和多人游戏](./网络和多人游戏.md#remote方法调用) |
| `@Import` | 导入其他组件字段 | `@Import float x, y;` | [实体系统](./实体系统和代码生成.md#import机制) |

## 💡 开发最佳实践

### 性能优化
- **零分配原则**：主循环中避免创建新对象
- **对象池使用**：`Pools.obtain()` 和 `Pools.free()`
- **静态变量重用**：使用 `Tmp` 类的临时变量
- **专用集合类型**：`IntSeq` 代替 `ArrayList<Integer>`

### 代码风格
- **命名规范**：camelCase，包括常量和枚举
- **导入方式**：使用通配符导入 `import some.package.*`
- **方法组织**：避免不必要的getter/setter，优先使用public字段
- **性能考虑**：避免装箱类型，使用原生类型集合

### 架构原则
- **模块化设计**：职责清晰，低耦合高内聚
- **事件驱动**：模块间通过事件通信，避免直接调用
- **平台兼容**：严格遵循Java 8兼容性
- **可扩展性**：支持MOD和插件扩展

## 🛠️ 开发环境设置

### 必需工具
```bash
# 安装JDK 17（其他版本不兼容）
# 下载：https://adoptium.net/archive.html?variant=openjdk17

# 克隆项目
git clone https://github.com/Anuken/Mindustry.git
cd Mindustry

# 运行桌面版
./gradlew desktop:run

# 构建项目
./gradlew desktop:dist

# 运行测试
./gradlew tests:test
```

### IDE配置
- **推荐IDE**: IntelliJ IDEA Community Edition
- **代码格式**: 导入 `.github/Mindustry-CodeStyle-IJ.xml`
- **编译设置**: Project SDK: Java 17, Module language level: 8

## 🔗 外部资源

### 官方资源
- **主仓库**: [GitHub - Anuken/Mindustry](https://github.com/Anuken/Mindustry)
- **Wiki**: [官方Wiki](https://mindustrygame.github.io/wiki)
- **Javadoc**: [API文档](https://mindustrygame.github.io/docs/)
- **Discord**: [开发社区](https://discord.gg/mindustry)

### 开发相关
- **贡献指南**: [CONTRIBUTING.md](../CONTRIBUTING.md)
- **代码风格**: [.github/Mindustry-CodeStyle-IJ.xml](../.github/Mindustry-CodeStyle-IJ.xml)
- **构建状态**: [GitHub Actions](https://github.com/Anuken/Mindustry/actions)

## 📝 更新日志

- **2025-09-20**: 创建核心架构文档（项目概述、架构设计、实体系统）
- **2025-09-20**: 添加游戏系统文档（内容系统、世界系统、网络系统）
- **2025-09-20**: 完善文档索引和学习路径指南

## 🤝 贡献指南

欢迎对文档进行改进！请遵循以下步骤：

1. **Fork项目** 并创建分支
2. **添加或修改文档** 内容
3. **确保格式正确** 和链接有效
4. **提交Pull Request** 并说明修改内容

### 文档维护原则
- **准确性优先**：确保技术信息正确
- **实用性导向**：关注开发者实际需求
- **及时更新**：随代码变更同步更新
- **易于理解**：使用清晰的示例和说明

---

**Happy Coding! 🚀**

如果您在开发过程中遇到问题，欢迎查阅相关文档或在社区中寻求帮助。这些文档将帮助您快速掌握Mindustry的核心技术，并能够高效地进行功能开发和扩展。
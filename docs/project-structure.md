# FloatingKnights 项目结构与使用规范

> 适用工程：Godot 4.7（.NET / C#）、2D 像素风、GL Compatibility 渲染后端。
> 本文档规定**仓库结构与资源使用约定**；引擎行为以 Godot 官方文档为准（关键结论已标注出处）。
> 最近核对：与提交 `9f98363` 对应的盘面一致（46 个目录）。

---

## 1. 三条设计原则

1. **`assets/` 只放引擎会导入的东西。** 文档、源文件、参考图、脚本一律不进 `assets/`。
2. **源文件与资产分离。** 可编辑源文件（`.aseprite`）放 `art-source/`，用一个空的 `.gdignore` 让 Godot 完全不扫描该目录；git 照常跟踪它。
3. **一物一形态。** 同一张图只允许**一种**形态被 Godot 导入。整图与切好的小图**不得同时**存在于 `assets/`。

第 3 条的理由：Godot 会导入 `res://` 下每一个受支持的文件。同一张图的两种形态并存，等于双份磁盘与显存占用、两套 `.import` 文件，而且改动只更新其中一处时必然产生静默漂移。

---

## 2. 目录总览

```
FloatingKnights/
├─ assets/                                      引擎消费的资源（会被 Godot 导入）
│  ├─ art/
│  │  ├─ characters/                            角色图集、头像、立绘
│  │  ├─ environment/                           地形图集、视差背景、装饰图案
│  │  │  └─ tiles/                              （用途待定，见 §10）
│  │  ├─ props/                                 道具、机关、可交互物
│  │  ├─ ui/                                    面板、按钮、边框、九宫格底图
│  │  ├─ icons/                                 技能/物品/状态小图标
│  │  └─ vfx/                                   特效贴图、序列帧
│  ├─ audio/{bgm,sfx,voice}/                    音乐 / 音效 / 配音
│  ├─ fonts/                                    字体资源
│  └─ shaders/                                  着色器源码
├─ art-source/                                  可编辑源文件区（Godot 完全看不见）
│  ├─ .gdignore                                 空文件，本目录的"隐身开关"
│  └─ {characters,environment,props,ui,icons,vfx}/
├─ scenes/{main,levels,characters,props,ui,components}/
├─ scripts/{core,gameplay,ui,data,utils}/       C# 代码
├─ resources/{characters,items,abilities,config,tilesets}/
├─ data/                                        策划可编辑的外部数据表
├─ tests/                                       测试
├─ addons/                                      Godot 插件
├─ tools/                                       编辑器脚本、构建辅助脚本
└─ docs/                                        设计文档与规范
```

### 根目录文件

| 文件 | 作用 | 谁维护 |
|---|---|---|
| `project.godot` | 工程配置（名称、渲染、物理、命名风格、输入映射、自动加载） | **Godot 编辑器**（你可能没手改它，编辑器也会写它） |
| `.gitignore` | 排除 `.godot/`、`/android/`、`bin/`、`obj/`、IDE 目录 | 手动 |
| `.gitattributes` | 换行统一为 LF；`.cs` 用 C# diff；`.png/.svg` 标记为二进制 | 手动 |
| `.editorconfig` | 跨编辑器的代码风格 | 手动 |
| `FloatingKnights.csproj` / `.slnx` | C# 工程与解决方案 | dotnet / 手动 |
| `icon.svg` + `icon.svg.import` | 工程图标及其导入设置 | 自动生成 `.import` |
| `README.md` / `LICENSE` | 项目说明 / MIT 许可 | 手动 |

> 注意：**Godot 编辑器保存工程时会重写 `project.godot`**（例如 `config/features` 会被补上 `"C#"`）。看到 `git status` 出现 ` M project.godot` 不要慌，先 `git diff project.godot` 确认是不是你在编辑器里改过设置，是的话照常提交。**不要**为了"保持干净"去还原它，否则下次打开编辑器它又会被写回来。

---

## 3. 各目录职责

### 3.1 `assets/art/<主题>/` — 2D 美术

`<主题>` 固定为六个：`characters`、`environment`、`props`、`ui`、`icons`、`vfx`。**主题只是分类，不是格式分类**——图片一律放这里，3D 模型/材质在本工程不适用。

| 目录 | 放什么 | 谁消费它 | 不放什么 |
|---|---|---|---|
| `characters/` | 角色/敌人图集、对话头像、立绘 | `Sprite2D`、`AnimatedSprite2D`(`SpriteFrames`) | 3D 相关、参考图 |
| `environment/` | 地形图集、视差背景层、云层、装饰图案 | `TileMapLayer`(`TileSet`)、`Parallax2D` | 单帧装饰的场景逻辑 |
| `props/` | 道具、宝箱、机关图集 | 道具场景 | — |
| `ui/` | 面板、按钮、边框、九宫格底图、对话框 | `NinePatchRect`、`TextureRect` | 图标（放 `icons/`） |
| `icons/` | 技能/物品/状态小图标 | HUD、背包格子 | — |
| `vfx/` | 打击、爆点、烟雾贴图或序列帧 | `GPUParticles2D`、`AnimatedSprite2D` | — |

**规范**：

- 一个主题下的所有资源使用**小写 snake_case** 命名。
- 每个 `*.png` 旁边必然有一个 `*.png.import`——**两者都要提交**。
- **不做物理切图**：图集切分用 `TileSet` / `SpriteFrames` / `AtlasTexture` 的资源描述表达，磁盘上始终只有一张图。
- 只有在下列情况才允许物理切分，且**只提交切分后的那批**，整图另存到 `art-source/`：同一张图的不同部分需要**不同导入设置**（如 UI 要无损、Repeat 模式不同）、需要独立的 mipmap、导入器不支持你想要的切法。

### 3.2 `assets/audio/`、`assets/fonts/`、`assets/shaders/`

| 目录 | 放什么 | 规范 |
|---|---|---|
| `audio/bgm/` | 背景音乐 | 长音频用 `.ogg`；注意循环点设置 |
| `audio/sfx/` | 短音效 | 可用 `.wav` |
| `audio/voice/` | 配音、旁白 | 按台词/编号命名，便于对话系统索引 |
| `fonts/` | 像素字体 `.ttf/.otf` | 导入时 **Antialiasing = None**、Hinting 关闭，否则小字号发虚；位图字体可用图像的 `Font Data (Monospace Image Font)` 导入类型 |
| `shaders/` | `.gdshader` / `.gdshaderinc` | 一个效果一个文件；可复用函数抽到 `.gdshaderinc` |

### 3.3 `art-source/` — 可编辑源文件区

| 目录 | 放什么 |
|---|---|
| `art-source/.gdignore` | **空文件（0 字节）**，必须存在 |
| `art-source/<主题>/` | `.aseprite` 等源文件，主题与 `assets/art/` 一一对应 |

**`.gdignore` 的作用**（官方文档《Project organization → Ignoring specific folders》）：

1. Godot **不导入**该目录下的任何文件（含所有子目录）；
2. 该目录从编辑器的 **FileSystem 面板隐藏**，减少干扰；
3. 其内容**不会被打进导出包**，能减小 PCK 体积；
4. 文件内容必须为空，且**不支持 `.gitignore` 那样的通配符模式**。

**规范**：

- 本目录**不进** Godot 的资源体系：不能用 `load()` / `preload()` 加载，也不能从编辑器里拖拽。
- 导出纪律：改完源文件后手动导出整图到 `assets/art/<对应主题>/`；**不要**把整图同时留在 `assets/` 里（违反 §1 第 3 条）。
- Windows 上用资源管理器手建点开头文件会被吃掉开头的点：改用 PowerShell `New-Item`、`type nul > .gdignore`，或按官方建议命名为 `.gdignore.`（确认后 Windows 自动去掉结尾的点）。

### 3.4 `scenes/` — 组装层

场景负责"把资源和脚本装配起来"，**不存放资源本体**。

| 目录 | 放什么 |
|---|---|
| `main/` | 入口、主菜单、加载页、设置页 |
| `levels/` | 关卡、房间（内含 `TileMapLayer`） |
| `characters/` | 玩家、敌人、NPC 场景 |
| `props/` | 道具与机关场景 |
| `ui/` | HUD、背包、对话框、弹窗 |
| `components/` | 可复用子场景：血条、伤害数字、交互提示 |

**规范**：`*.tscn` 用 **PascalCase**（`project.godot` 已设 `naming/scene_name_casing=1`）。能被两处以上使用的 UI/逻辑必须抽到 `components/`，禁止复制粘贴场景。

### 3.5 `scripts/` — C# 代码

| 目录 | 放什么 |
|---|---|
| `core/` | 全局系统：游戏状态、存档、事件总线、Autoload 单例 |
| `gameplay/` | 玩法逻辑：移动、战斗、AI、技能、掉落 |
| `ui/` | 界面控制与数据绑定 |
| `data/` | **自定义 `Resource` 类型定义**（数据模型类） |
| `utils/` | 工具类、扩展方法、常量、枚举 |

**规范**：`*.cs` 用 **PascalCase**（`naming/script_name_casing=1`）。`scripts/data/` 里只放"数据长什么样"的类型定义，不放具体数值——具体数值是资源实例，放 `resources/`。

### 3.6 `resources/` — 数据实例

`.tres` / `.res` 资源实例，类型来自 `scripts/data/`。

| 目录 | 放什么 |
|---|---|
| `characters/` | 角色属性数值 |
| `items/` | 道具、装备配置 |
| `abilities/` | 技能/能力配置 |
| `config/` | 全局配置：难度、平衡数值 |
| `tilesets/` | **`TileSet` 资源**（见 §7） |

**规范**：数值调整只改这里，不碰代码；`TileSet` 资源必须**外部保存**到 `tilesets/`，禁止内嵌在关卡场景里（否则无法复用）。

### 3.7 其余目录

| 目录 | 放什么 | 规范 |
|---|---|---|
| `data/` | `.csv` / `.json` 外部数据表 | 策划可编辑；导入后生成 `resources/` 实例 |
| `tests/` | 单元/集成测试 | 与 `scripts/` 结构对应 |
| `addons/` | Godot 插件（编辑器扩展） | Godot 约定目录名；第三方插件注明来源与版本 |
| `tools/` | `@tool` 脚本、导入/构建辅助脚本 | 不参与运行时逻辑 |
| `docs/` | 设计文档与规范（含本文档） | 讨论结论沉淀到这里，不要只留在聊天记录里 |

---

## 4. 命名规范

| 对象 | 规范 | 理由 |
|---|---|---|
| 资源文件（图片/音频/字体/着色器） | `snake_case`，全小写 | 导出后的 PCK 虚拟文件系统**大小写敏感**，全小写可避免跨平台路径不一致 |
| 目录 | 全小写 | 同上 |
| `.tscn` 场景 | `PascalCase` | 已由 `project.godot` 的命名风格配置约束 |
| `.cs` 脚本 | `PascalCase` | 同上 |
| `TileSet` 瓦片命名 / atlas 名 | 语义化（`ground_center`、`terrain`） | 标注 peering bits 时必须一眼看出该填哪块 |

**禁止**用 `01.png`、`新建文件夹`、含空格或中文的文件名——这些在跨平台与命令行脚本里都会出问题。

---

## 5. 像素风的必备设置

| 项 | 值 | 出处/说明 |
|---|---|---|
| 图片 `Compress > Mode` | **Lossless** | 官方文档明确把 Lossless 列为像素风推荐设置；2D 默认即是 |
| 图片 `Mipmaps > Generate` | **关闭** | 2D 默认关闭 |
| **Detect 3D** | 留意 | 一旦该纹理被 3D 场景用到，Godot 会自动改成 VRAM Compressed 并打开 mipmap，像素会糊。修正方式：改 `Detect 3D > Compress To`，或事后手动改回 Lossless |
| `Sprite2D` / `TileMapLayer` 等 CanvasItem 的 `texture_filter` | **Nearest** | 自 Godot 4.0 起，**filter 与 repeat 是 CanvasItem 属性**，不在导入面板里 |
| 项目设置 `rendering/textures/canvas_textures/default_texture_filter` | **Nearest** | 设为全局默认，免得逐个节点改 |

**Aseprite 导出注意**：导出 sprite sheet 时，`Trim`、`Spacing`、`Border Padding` 之类选项**一律归零/关闭**，否则图集尺寸不再是"格子数 × 格子尺寸"，Godot 的网格会整体错位。画布请用 16×16 网格并开启吸附，保证所有图块边界落在网格线上。

---

## 6. 图片切分规范

**默认做法：整图进仓库，切分交给资源描述。** Godot 不会生成切好的图片文件，"分割"的结果是资源文件里的格子描述。

| 需求 | 做法 | 产物 |
|---|---|---|
| 地形瓦片 | 整图 + `TileSet`（见 §7） | `.tres` |
| 逐帧动画 | 整图 + `SpriteFrames` | `.tres` |
| 单帧精灵取局部 | `AtlasTexture`(.tres) 或 `Sprite2D.region_rect` | `.tres` 或节点属性 |
| 九宫格 UI | 整图 + `NinePatchRect` 的 patch margins | 无需额外文件 |
| 图像作为纹理图集资源 | 把图片的导入类型设为 **`TextureAtlas`** | 内置导入类型，仅支持 2D |

**必须注意**：`.import` 文件是导入设置，**必须提交**；`res://.godot/` 是导入缓存，**禁止提交**（已在 `.gitignore` 中排除）。

---

## 7. TileSet 工作流

### 7.1 创建顺序（顺序错了切不出正确格子）

1. 把图集 png 放进 `assets/art/environment/`，让 Godot 导入；
2. 新建 `TileMapLayer` 节点（Godot 4.3 起使用 `TileMapLayer`，旧的 `TileMap` 已弃用）；
3. 选中该节点，在检视面板的 `Tile Set` 属性上**新建 TileSet 资源**；
4. **先在 TileSet 检视面板里设 `Tile Size`**（如 16×16，必须与源图网格一致）；
5. 打开编辑器底部的 **TileSet 面板**，把 png **拖进去**，弹窗问"是否自动创建瓦片"时选 **Yes**；
6. 给 TileMapLayer 的 `texture_filter` 设为 **Nearest**；
7. 检视面板里建 **Physics Layers → Add Element**，逐格画碰撞多边形（编辑器里按 `F` 可一键生成矩形碰撞）；导航/遮挡用同样方式建层（遮挡多边形在 **Rendering** 子段里）；
8. 建 **terrain set** → 在它内部建至少一个 **terrain**（ID 从 `0` 开始，`-1` 表示"无"）；
9. 逐格填 **Terrains** 与 **Terrain Peering Bits**（8 位，`-1` 表示空位）；
10. TileSet 资源 **Save As** → `resources/tilesets/<名字>.tres`。

补充要点：

- **完全透明的区域不会生成瓦片**——源图里留空的格子天然不会变成废瓦片。
- 不想要的瓦片用 **Eraser** 工具点掉，或右键 → `Delete`。
- 改动 atlas 的 `Margins` / `Separation` / `Texture Region Size` 可能导致瓦片丢失，用面板右上"三个点"菜单里的 **Create Tiles in Non-Transparent Texture Regions** 重新生成。
- atlas 属性：`Margins` 用于图集边缘留白，`Separation` 用于格与格之间的间隔，`Use Texture Padding` 默认开启（防止开启过滤时纹理渗色），像素风建议保持开启。

### 7.2 混合尺寸图块：不要全切成 16×16

一个 atlas 的 `Texture Region Size` 是**统一**的，所以不能在同一网格里混用不同格尺寸。混合尺寸有三种处理方式：

| 方案 | 做法 | 适用 |
|---|---|---|
| **A. 统一网格 + 多格瓦片（推荐）** | 只建一个 16×16 的 atlas，大图块声明为"占据多个格子" | 所有图块尺寸都是 16 的整数倍 |
| B. 每个尺寸一个 atlas | 16×16 一个、32×32 一个、16×32 一个……各自独立；多个 atlas 也可用 **Open Atlas Merging Tool** 合并 | 尺寸不规整，或想分开管理 |
| C. 场景瓦片（Scenes Collection） | 把图块做成场景放进 TileSet | 需要动画、粒子或交互的图块 |

方案 A 的尺寸换算：

| 原图块 | 网格占比（宽 × 高） |
|---|---|
| 16×16 | 1 × 1 |
| 32×32 | 2 × 2 |
| 16×32 | 1 × 2 |
| 16×48 | 1 × 3 |

**操作（点击级）**：

1. 执行 §7.1 第 5 步后，自动创建会把 32×32 的图案拆成 4 个 1×1、16×48 拆成 3 个——这是正常现象，不是出错。
2. **先删掉**覆盖区域上的那些 1×1 瓦片：用顶部 **Eraser** 工具点掉，或右键 → `Delete`。
   ⚠️ 必须先删：[godot#98170](https://github.com/godotengine/godot/issues/98170) 中 TileSet 维护者的原话是 "You cannot create big tiles on top of already created tiles. You need to remove the underlying tiles first."，这是为避免数据丢失的有意设计。
3. 切到 **Select** 模式，鼠标悬停在 atlas 上时编辑器会显示提示 **"Hold shift to create big tiles"**；按住 **Shift** 拖拽出目标矩形（左上 → 右下），松手即创建一个多格瓦片。
4. 在中间栏核对它占据的格数，应等于上表的"网格占比"。
5. 大瓦片**必须是基础格尺寸的整数倍**。维护者说明："big tiles need to be constant multiplies, e.g. base size 16x16, big tiles 16x32, 32x32, 64x64 etc."（[godot#68299](https://github.com/godotengine/godot/issues/68299)）。任意不规则尺寸（如 24×24、120×102）**不支持**，那类需求要用 tile patterns 表达。

对应 API（脚本化或核对状态时用）：

| API | 作用 |
|---|---|
| `TileSetAtlasSource.create_tile(atlas_coords, size)` | 在指定坐标创建指定占格数的瓦片 |
| `get_tile_size_in_atlas(atlas_coords)` | 查询某瓦片占据几格 |
| `get_tile_at_coords(atlas_coords)` | 返回覆盖该坐标的瓦片的左上角坐标——说明**大瓦片会占用其覆盖区域内的所有坐标**，这正是第 2 步必须先删的原因 |

**判断"该整体还是该拆开"的标准**（比尺寸更重要）：

- 该图案**永远作为一个整体出现**（藤蔓、大树、招牌）→ **多格瓦片**；
- 该图案是**若干可独立摆放的小格**（地砖、砖墙）→ **保留 1×1**，可单独旋转、翻转、替换。

### 7.3 分层是硬约束

- **同一个 TileMapLayer 里，每个格子只能有一个瓦片。** 所以装饰与地面/侧面重叠时**必须分属不同层**——这是机制要求，不是风格选择。
- **层间**绘制顺序：用节点的 `z_index` 或 `y_sort_enabled` 控制。
- **层内**微调：单个瓦片的 `TileData` 有 `z_index`（相对绘制顺序）、`y_sort_origin`（排序基准点）、`texture_origin`（相对所在格子的绘制偏移）。装饰要"精确压在接缝上"时用 `texture_origin` 调，不必回头改图。

### 7.4 地形系统的能力边界（重要）

**terrains / peering bits 只做一件事**：为**单个格子**按邻居形状自动挑选**变体**（边缘、角落、内部）。官方文档的表述是"terrains 只是赋给 atlas 瓦片的一组属性，被一个专用绘制模式用来'聪明地'选择瓦片"。

它**不负责**：

- 按"连续 N 格"这样的多格图案去**生成/放置**一个装饰；
- 判断"此处没有地面"这类条件；
- peering bits 只有 8 位、描述"邻居是否同类"，**表达不了"必须连续 3 格"**。

因此**条件装饰**（藤蔓、坑洞装饰等）只有三条路：

| 方案 | 做法 | 评价 |
|---|---|---|
| 1. 手动放 + 多格瓦片 | 装饰单独一层；整条藤蔓做成一个多格瓦片，一次点击放整条，位置天然对齐 | 建议起步，最可控 |
| 2. 场景瓦片 | 装饰做成场景（`Sprite2D` 或带动画的 `AnimatedSprite2D`），从 Scenes Collection 放 | 需要动画/交互时；注意官方性能警告：每个实例都会真实实例化节点 |
| 3. `@tool` 脚本自动放置 | 扫描地形层，匹配图案条件后向装饰层 `set_cell` | **唯一真正的"自动生成"**；关卡量大或程序化生成时值得写 |

方案 3 的要点：装饰必须与地面分层；脚本用 `get_cell_tile_data()` 读地面层（判断 terrain ID 或自定义数据），再写到装饰层。

地形绘制本身有三种模式（在 TileMap 编辑器的 **Terrains** 标签下）：**Connect**（与同一层周围瓦片连接）、**Path**（只与同一次笔画内涂的瓦片连接，更可控）、以及**指定具体瓦片**（处理地形系统未覆盖的冲突）。

---

## 8. 提交规范

采用 **Conventional Commits**：

```
<type>(<scope>): <description>
```

| type | 用途 |
|---|---|
| `feat` | 新增功能 |
| `fix` | 修 bug |
| `docs` | 文档 |
| `refactor` | 重构（不改行为） |
| `perf` | 性能优化 |
| `test` | 测试 |
| `build` | 构建/依赖 |
| `chore` | 工程维护：目录结构、配置文件、资源入库 |

本仓库的 scope 约定：

| scope | 覆盖范围 |
|---|---|
| `project` | `project.godot`、`.csproj`、`.slnx`、`.gitignore` 等工程配置 |
| `assets` | `assets/` 与目录结构 |
| `art` | 美术源文件与美术资产 |
| `scenes` / `scripts` / `resources` / `data` | 对应目录 |
| `docs` | 文档 |

历史示例（本仓库真实提交）：

```
chore(project): enable C# feature tag
chore(art): add FloatingKnightsTiles Aseprite source
chore(assets): add tilesets resource dir and sliced-tile layout
chore(assets): scaffold resource directory structure
```

**必须提交**：`*.import`、`*.tres`、`*.tscn`、`*.cs`、`art-source/` 下的源文件、`project.godot`、`.gitattributes` / `.gitignore` / `.editorconfig`、占位用的空 `.gitkeep`。

**禁止提交**：`res://.godot/`（导入缓存）、`bin/`、`obj/`、IDE 目录、`*.user`。

多行提交信息建议用文件传入，避免 PowerShell 引号转义问题：

```powershell
$msg = @'
chore(assets): <标题>

<正文说明为什么这样改>
'@
$msg | Out-File "$env:TEMP\msg.txt" -Encoding utf8 -NoNewline
git commit -F "$env:TEMP\msg.txt"
```

---

## 9. 反模式清单

| 反模式 | 后果 | 正确做法 |
|---|---|---|
| 整图与切好的小图同时在 `assets/` | 双份占用、两套 `.import`、改动漂移 | 只保留一种形态（§1 第 3 条） |
| 把 `.aseprite` 放进 `assets/` | Godot 没有该格式的导入器，等于放了个死文件 | 放 `art-source/` |
| 删除 `.gdignore` 或往里写内容 | Godot 会开始扫描该目录并可能导出源文件 | 保持空文件 |
| 忘记提交 `*.import` | 换机器/CI 上资源错乱、重新导入后设置丢失 | `.import` 必须入库 |
| 提交 `.godot/` | 仓库体积膨胀、无意义冲突 | 已由 `.gitignore` 排除 |
| 把 `TileSet` 内嵌在关卡场景里 | 无法在多个关卡间复用 | 外部保存到 `resources/tilesets/` |
| 装饰与地面放同一个 TileMapLayer | 机制上做不到（每格只能一个瓦片） | 装饰单独分层 |
| 期望 terrains 自动生成长条装饰 | 地形系统没有这个能力 | 手动 / 场景瓦片 / `@tool` 脚本（§7.4） |
| 资源文件用大写或驼峰 | PCK 大小写敏感，跨平台路径失败 | 一律小写 `snake_case` |
| 把混合尺寸图块全部切成 16×16 | 丢失"整体"语义、工作量大 | 统一网格 + 多格瓦片（§7.2） |
| 为保持工作区干净而还原编辑器写入的 `project.godot` | 下次打开编辑器又会被写回，反复骚扰 | 确认后照常提交（§2 根目录文件） |

---

## 10. 待决事项

1. **`assets/art/environment/tiles/` 的去留**：若最终采用"整张图集 + 多格瓦片"（§7.2 方案 A），则不需要该子目录，整图直接放 `assets/art/environment/`；若改为物理切分成多张单瓦片 png，则该目录用于存放单瓦片文件。
2. **`.ase` → `.png` 自动导出脚本**：已决定**不做**，改由人工在 Aseprite 中导出。`tools/` 因此暂时为空。
3. **Git LFS**：当前**未启用**（`.gitattributes` 只有 `*.png binary`，这只关掉文本 diff，不等于 LFS）。触发条件：单个文件接近 50 MB，或美术资源累计上百 MB。届时启用需执行 `git lfs migrate import`（**会重写历史**，需单独确认）。
4. **装饰自动放置脚本（`@tool`）**：待 TileSet 建成、装饰分层确定后再设计实现。

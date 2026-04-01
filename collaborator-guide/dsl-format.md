# DSL格式

## 引言

本文档描述了酸橙熊熊与其 XML DSL 的完整语言规范。该 DSL 采用声明式 XML 编写图像排版与渲染逻辑，通过内嵌 NCalc 表达式引擎与 SmartFormat 模板系统，实现动态数据绑定和运行时条件渲染。

---

## DSL 语法规范与格式说明

DSL 根节点必须为 `<Canvas>`，且 `<Canvas>` 仅允许作为根节点使用，不允许嵌套在其他节点内部。文档应完全通过 XML 属性驱动逻辑。

当前渲染运行时不会自动执行 Relax NG (`Layout.rng`) 校验，推荐在协作流程中采用以下策略：
- 在布局 XML 顶部保留 `<?xml-model href="Layout.rng"?>`，利用编辑器实时提示结构错误。
- 在提交前通过本地或 CI 的 XML/RNG 校验步骤提前阻断结构性问题，再进入运行时解析。

### 1. 属性计算类型

在 XML 标签中，属性值分为两类求值模式：

1. **模板格式 (TemplateText)**：
   - 使用 `SmartFormat` 处理字符串。
   - 支持插入基于大括号的数据绑定：`{varName}`
   - 支持内嵌基于 `@{...}` 语法的表达式运算：`namespace="@{[typeName] + '_img'}"`
2. **表达式格式 (ExprText)**：
   - 纯粹通过 `NCalc` 表达式引擎计算，属性值整体被当成计算式。
   - 支持常量、变量与索引等表达式，变量名需要使用 `[]` 包裹（如 `[idx]`）。
   - 例如：`width="10 + [idx]"`，`rule="[score] > 100"`。

### 2. 核心节点规范

下表详细列出了各节点的属性、用途、预期类型与默认值。`key` 为可选属性（`TemplateText` 类型），用于节点稳定标识与结构保留；在 `Layer` 等容器场景下可用于避免无键节点被内部扁平化处理。仅对 `Canvas`、`Layer`、`Positioned`、`Resized`、`Stack`、`Grid`、`Image`、`Text`、`Set` 生效；`If`、`ElseIf`、`Else`、`For`、`Include` 不支持 `key`。

#### `Canvas`（画布）
顶级容器，定义图像基础大小和背景色。
- **`width`** (ExprText, `int`, 必填): 画布宽度。
- **`height`** (ExprText, `int`, 必填): 画布高度。
- **`background`** (TemplateText, `string`, 选填): 背景颜色（支持 Hex 颜色码如 `#FFFFFF` 或 `#80FFFFFF`）。

#### `Layer`（图层）
用于将多个节点组合在一起，并统一应用不透明度。
- **`opacity`** (ExprText, `number`, 选填): 图层的不透明度，范围 `0.0` 到 `1.0`。默认值：`1.0`。

#### `Positioned`（绝对定位）
允许次级节点在给定的坐标系中进行绝对定位。
- **`x`** (ExprText, `int`, 必填): X 轴绝对坐标。
- **`y`** (ExprText, `int`, 必填): Y 轴绝对坐标。
- **`anchorX`** (ExprText, `Enum`, 选填): X 轴锚点。可选值：`Start` (默认), `Center`, `End`。
- **`anchorY`** (ExprText, `Enum`, 选填): Y 轴锚点。可选值：`Start` (默认), `Center`, `End`。
- **`width`** (ExprText, `int`, 选填): 容器约束宽度。
- **`height`** (ExprText, `int`, 选填): 容器约束高度。

#### `Resized`（缩放）
对**唯一次级节点**进行缩放或重采样，必须且只能包含 1 个次级节点。
- **`scale`** (ExprText, `number`, 选填): 缩放比例，例如 `0.5` 表示缩小一半。默认值：`1.0`。
- **`width`** (ExprText, `int`, 选填): 强制缩放到的目标宽度。
- **`height`** (ExprText, `int`, 选填): 强制缩放到的目标高度。
- **`resampler`** (ExprText, `Enum`, 选填): 重采样算法。可选值如：`Bicubic`, `Box`, `Lanczos2`, `Lanczos3` (默认), `NearestNeighbor` 等。

#### `Stack`（线性堆叠）
将次级节点按水平或垂直方向依次排列（类似 Flexbox）。
- **`direction`** (ExprText, `Enum`, 必填): 排列方向。推荐值：`Row` (水平), `Column` (垂直)。解析时大小写不敏感，`row`/`column` 也可用。
- **`spacing`** (ExprText, `number`, 选填): 主轴方向的节点间距。默认值：`0`。
- **`runSpacing`** (ExprText, `number`, 选填): 开启换行时，交叉轴方向的行/列间距。默认继承 `spacing` 的值。
- **`wrap`** (ExprText, `bool`, 选填): 元素超出容器时是否换行。默认值：`false`。
- **`justify`** (ExprText, `Enum`, 选填): 主轴对齐方式。可选值：`Start` (默认), `Center`, `End`, `SpaceBetween`, `SpaceAround`, `SpaceEvenly`。
- **`alignItems`** (ExprText, `Enum`, 选填): 交叉轴对齐方式。可选值：`Start` (默认), `Center`, `End`, `Stretch`。
- **`alignContent`** (ExprText, `Enum`, 选填): 多行/多列整体在交叉轴上的对齐方式。可选值：`Start` (默认), `Center`, `End`。仅在 `wrap=true` 且出现多行/多列时生效。
- **`width`** (ExprText, `int`, 选填): 容器固定宽度。
- **`height`** (ExprText, `int`, 选填): 容器固定高度。

#### `Grid`（网格布局）
将次级节点放置在规则的网格中。
- **`columns`** (ExprText, `int`, 选填): 网格列数。默认值：`1`。
- **`columnGap`** (ExprText, `int`, 选填): 列间距。默认值：`0`。
- **`rowGap`** (ExprText, `int`, 选填): 行间距。默认值：`0`。
- **`justifyContent`** (ExprText, `Enum`, 选填): 网格整体内容块在容器内的水平对齐。可选值：`Start` (默认), `Center`, `End`。
- **`alignContent`** (ExprText, `Enum`, 选填): 网格整体内容块在容器内的垂直对齐。可选值：`Start` (默认), `Center`, `End`。
- **`justifyItems`** (ExprText, `Enum`, 选填): 单元格内水平对齐。可选值：`Start` (默认), `Center`, `End`, `Stretch`。
- **`alignItems`** (ExprText, `Enum`, 选填): 单元格内垂直对齐。可选值：`Start` (默认), `Center`, `End`, `Stretch`。
- **`width`** (ExprText, `int`, 选填): 容器固定宽度。
- **`height`** (ExprText, `int`, 选填): 容器固定高度。

#### `Image`（图像）
渲染一张静态图片，其路径由全局资源配置 (AssetProvider) 解析。无次级节点。
- **`namespace`** (TemplateText, `string`, 必填): 资源命名空间，用于匹配 `Resources.xml` 中的映射规则。
- **`id`** (TemplateText, `string`, 必填): 图片标识符，通常与 namespace 结合决定最终文件路径。
- **`colorBlending`** (ExprText, `Enum`, 选填): 像素颜色混合模式（如 `Normal`）。
- **`alphaComposition`** (ExprText, `Enum`, 选填): Alpha 合成模式（如 `SrcOver`）。
- **`repeat`** (ExprText, `int`, 选填): 前景图动画循环次数，默认值：`0`。通常保持默认值，仅在需要覆盖动画循环策略时设置。

#### `Text`（文本）
绘制矢量文本。无次级节点。
- **`value`** (TemplateText, `string`, 选填): 文本内容。
- **`fontFamily`** (TemplateText, `string`, 选填): 字体命名空间，需在 `Resources.xml` 中配置。默认值：`"BoldFont"`。
- **`fontSize`** (ExprText, `number`, 选填): 字体大小。默认值：`20`。
- **`color`** (TemplateText, `string`, 选填): 字体颜色。默认值：`"#FFFFFF"`。
- **`align`** (ExprText, `Enum`, 选填): 文本块对齐方式。可选值：`Start` (默认), `Center`, `End`。
- **`alignV`** (ExprText, `Enum`, 选填): 垂直对齐。可选值：`Top` (默认), `Center`, `Bottom`。
- **`alignH`** (ExprText, `Enum`, 选填): 水平对齐。可选值：`Left` (默认), `Center`, `Right`。
- **`strokeColor`** (TemplateText, `string`, 选填): 描边颜色。
- **`strokeWidth`** (ExprText, `number`, 选填): 描边宽度。默认值：`0`。
- **`truncateWidth`** (ExprText, `number`, 选填): 文本截断限制宽度。
- **`truncateSuffix`** (TemplateText, `string`, 选填): 文本截断时追加的后缀字符串（如 `...`）。

#### `Set`（变量声明）
在当前解析上下文中声明或更新一个变量，对其后续同级节点及次级节点生效。无次级节点。
- **`var`** (String, `string`, 必填): 变量名。
- **`value`** (ExprText, `object`, 必填): 变量计算表达式的值。

#### `If`（条件控制）
根据条件决定是否渲染其内部的次级节点。支持与后续同级的 `<ElseIf>` / `<Else>` 组成链式分支。
- **`rule`** (ExprText, `bool`, 必填): 判定规则，必须计算为布尔值。若为 `false`，内部所有次级节点将被忽略。

#### `ElseIf`（条件分支）
作为 `<If>` 或前一个 `<ElseIf>` 的同级后续分支使用；仅当前面分支均未命中时才会求值并尝试渲染。
- **`rule`** (ExprText, `bool`, 必填): 判定规则，必须计算为布尔值。

#### `Else`（兜底分支）
作为 `<If>` / `<ElseIf>` 链中的最终兜底分支使用；仅当前面所有分支都未命中时渲染。
- 无属性。

#### `For`（循环）
遍历集合，为每个元素复制并渲染次级节点。
- **`items`** (ExprText, `arraylike | int`, 必填): 需要遍历的集合或数组。也可以是能转换为**非负整数**的数值，表示循环次数。若为小数，仅接受小数部分为 `0` 的值（如 `3.0`）；`2.5` 这类非整数小数会直接抛出异常。表达式中变量名需使用 `[]`，例如 `[pageRecords]` 或 `50`。若计算结果为 `null`、负数，或既不是集合也不能转换为整数，将直接抛出异常。
- **`var`** (String, `string`, 选填): 注入循环体作用域的当前元素变量名。默认值：`"item"`。
- **`indexVar`** (String, `string`, 选填): 注入循环体作用域的当前索引变量名。默认值：`"idx"`。在表达式中引用默认索引变量时写作 `[idx]`。

#### `Include`（引入模板）
外部引入另一个 XML 文件进行解析，有助于组件复用。无次级节点。
- **`src`** (TemplateText, `string`, 必填): 要引入的外部 XML 文件的相对或绝对路径。解析后的路径必须位于当前布局根目录内（禁止越界访问）；同时禁止循环引入（如 `A -> B -> A`）。

---

## 全局资源配置规范

系统会通过配置文件读取并管理全局资源配置，配置定义了 DSL 中图像和字体的映射规则，供 `<Image>` 和 `<Text>` 节点在解析时调用。

### 1. 图像路径映射 (Path)

`<Path>` 节点用于定义图像资源的查找路径与规则。

- **`namespace`** (必填): 对应 `<Image>` 节点中的 `namespace` 属性。
- **`rule`** (选填): 资源文件名解析规则。若提供，将使用 `SmartFormat` 依据传入的 `key`（对应 `<Image>` 的 `id`）进行格式化，例如 `{key}.png`。
- **内容文本** (必填): 对应资源的根目录路径。

**配置示例**：
```xml
<Resources>
    <Path namespace="avatar" rule="{key}.jpg">./Resources/Images/Avatars</Path>
</Resources>
```
**调用示例**：
`<Image namespace="avatar" id="458001" />` 
将在运行时解析并加载路径：`./Resources/Images/Avatars/458001.jpg`

### 2. 字体映射 (FontFamily)

`<FontFamily>` 节点用于定义文本渲染使用的字体及回退字体（Fallback）。

- **`namespace`** (必填): 对应 `<Text>` 节点中的 `fontFamily` 属性。
- **`<Font>`** (必填): 主字体文件的路径（例如 `.ttf` 或 `.otf`）。
- **`<Fallbacks>`** (选填): 包含一个或多个 `<Font>` 次级节点，用于在主字体缺失某些字符（如生僻字、emoji）时按顺序尝试回退加载。

**配置示例**：
```xml
<Resources>
    <FontFamily namespace="BoldFont">
        <Font>./Resources/Fonts/Main-Bold.ttf</Font>
        <Fallbacks>
            <Font>./Resources/Fonts/Fallback-Emoji.ttf</Font>
            <Font>./Resources/Fonts/Fallback-CJK.ttf</Font>
        </Fallbacks>
    </FontFamily>
</Resources>
```
**调用示例**：
`<Text fontFamily="BoldFont" value="测试文本 🐼" />`
将首先使用 `Main-Bold.ttf` 渲染，当遇到无法显示的熊猫 emoji 时，自动使用 `Fallback-Emoji.ttf`。

---

## 内置变量参考

内置变量在渲染启动时注入当前求值上下文。根据不同的绘图入口（Bests 画图 或 List 画图），作用域变量会有所不同。

### 1. Bests 画图作用域变量

| 变量名 | 数据类型 | 说明 | 示例 |
| :--- | :--- | :--- | :--- |
| `userInfo` | `user` | 当前查询用户的概要数据对象 | `[userInfo.Name]` |
| `everRecords` | `record[]` | 用户历史最好成绩列表 | `[everRecords[0].Title]` |
| `currentRecords` | `record[]` | 用户现行最好成绩列表 | `[currentRecords[0].DXRating]` |
| `everRating` | `int` | 历史版本总 Rating (底分) | `1000` |
| `currentRating` | `int` | 现行版本总 Rating (底分) | `2000` |
| `typeName` | `string` | B50类型名称 | `"标准B50"` |
| `proberName` | `string` | 成绩来源查分器名称 | `"Lxns"` 或 `"DivingFish"` |
| `animeMode` | `bool` | 是否开启动态图片模式 | `true` |
| `needSuggestion` | `bool` | 是否绘制定数推荐分析 | `false` |
| `mayMask` | `bool` | 用户是否可能开启了掩码 | `false` |
| `everMax`, `everMin` | `int` | 历史版本中 Rating 的最高与最低值 | `300` |
| `currentMax`, `currentMin`| `int` | 现行版本中 Rating 的最高与最低值 | `280` |

### 2. List 画图作用域变量

| 变量名 | 数据类型 | 说明 | 示例 |
| :--- | :--- | :--- | :--- |
| `userInfo` | `user` | 当前查询用户的概要数据对象 | `[userInfo.Name]` |
| `pageRecords` | `record[]` | 当前页的成绩列表 | `[pageRecords[0]]` |
| `pageNumber` | `int` | 当前页码 | `1` |
| `totalPages` | `int` | 总页数 | `5` |
| `statCounts` | `int[]` | 统计分布数据 (分级统计等) | `[statCounts[0]]` |
| `totalCount` | `int` | 歌曲/成绩总数 | `150` |
| `startIndex` | `int` | 当前页起始索引 | `0` |
| `level` | `string` | 查询的特定等级标识 | `"14+"` |
| `proberName` | `string` | 成绩来源查分器名称 | `"Lxns"` |
| `mayMask` | `bool` | 用户是否可能开启了掩码 | `false` |

### 3. 实体对象结构

#### `user` 对象
- `Name` (`string`): 玩家昵称。
- `Rating` (`int`): 玩家总 Rating。
- `TrophyColor`, `TrophyText`: 称号框颜色及文本。
- `CourseRank`, `ClassRank`: 段位及阶级。
- `IconId`, `PlateId`, `FrameId` (`int`): 头像、姓名框、边框资源 ID。
- `RatingLevel` (`int`): 基于 Rating 计算出的底分等级段 (1~11)。

#### `record` 对象
- `Id` (`int`): 歌曲/谱面唯一 ID。
- `Title` (`string`): 歌曲标题。
- `Difficulty` (`Enum`): 谱面难度。
- `ComboFlag`, `SyncFlag` (`Enum`): 全连 (FC/AP) 及同步状态标记。
- `Rank` (`Enum`): 结算评级 (如 SS, SSS+ 等)。
- `Type` (`Enum`): 谱面类型 (Standard 或 DX)。
- `Achievements` (`number`): 达成率百分比 (如 100.5000)。
- `DXScore`, `TotalDXScore` (`int`): 玩家 DX 分数及谱面满 DX 分。
- `DXStar` (`int`): 获得星级数量。
- `DXRating` (`int`): 单曲 Rating 分数。
- `LevelValue` (`number`): 谱面具体定数 (如 14.8)。

---

## 内置函数参考

### 1. 核心扩展函数

渲染引擎注册了以下全局内置函数：

#### 1. `ToString(object x)`
- **功能描述**：将任意对象安全地转换为字符串，底层调用 `Convert.ToString`。
- **参数**：`x` (任意对象)。
- **返回值**：`string` 类型文本。
- **示例**：`<Text value="@{ToString([userInfo.Rating])}" />`

#### 2. `Count(IList x)`
- **功能描述**：获取数组、列表或集合的元素数量。
- **参数**：`x` (实现 `IList` 接口的集合，如 `[everRecords]`，`[currentRecords]`)。
- **返回值**：`int` 元素数量。
- **示例**：`<If rule="Count([everRecords]) > 0">...</If>`

### 2. NCalc 表达式引擎内置函数

由于底层依托 `NCalc`，DSL 自动支持标准数学与逻辑函数：
- **逻辑运算**：`if(condition, true_val, false_val)`, `in(val, set...)`
- **数学运算**：`Abs()`, `Max()`, `Min()`, `Pow()`, `Round()`, `Ceiling()`, `Floor()` 等。
- **类型转换与解析**：支持隐式及基于 `IConvertible` 的内置强转支持。

---

## 渲染管线生命周期与执行顺序

整个 DSL 解析及图像渲染过程在管线中分为三大阶段，完全同步且确定：

### 阶段 1: 加载与解析阶段 (Parsing & Evaluation)
1. **XML 树加载**：使用 `XDocument.Load` 将 `.xml` 文件读入内存。
2. **递归解析与求值**：从 `<Canvas>` 根节点开始自顶向下处理。
3. **作用域继承与隔离**：
   - 默认继承全局作用域。
   - 当遇到 `<Set>` 节点时，立即计算其 `value` 表达式，并将新变量写入**当前作用域**，仅对其后续同级节点和次级节点生效。
   - 当遇到 `<For>` 节点时，计算 `items` 集合。循环遍历时按顺序为每个元素创建独立的作用域闭包，包含局部变量 `var` 和 `indexVar`，并按 `items` 顺序依次展开生成相应的 `Node` 树。
   - 当遇到 `<If>` 节点时，立即求值 `rule`，若为 `false` 则跳过该子树加载，实现剪枝优化。
4. **生成抽象渲染树 (ART)**：此阶段完全抹除条件、循环和外部引入，输出一棵纯静态、纯数据的强类型 `Node` 对象树。

### 阶段 2: 测量与布局阶段 (Measurement & Layout)
1. 计算节点本身的内在大小 (基于 `width`/`height` 属性)。
2. 处理 `Stack` 与 `Grid` 等容器节点的排版规则 (`justifyItems`, `alignItems`, `wrap` 等)，确定次级节点的实际物理坐标 (X, Y) 与最终尺寸。

### 阶段 3: 光栅化与绘制阶段 (Rendering)
1. 初始化主画布。
2. 从抽象渲染树顶端开始进行 DFS 深度优先遍历。
3. 对于每个节点应用坐标偏移 (`origin`) 和不透明度 (`inheritedOpacity`) 的矩阵累加。
4. 调用 `ImageSharp` 处理管线对 `Image` 或 `Text` 节点进行像素填充或矢量文字绘制。所有混合 (`colorBlending`) 及重采样 (`resampler`) 操作在这一步被实际应用至图像内存中。

---

## 常见错误码与调试指南

当 DSL 编写不当时，系统将抛出以下核心异常，需查阅对应原因进行修复：

| 异常提示片段 | 常见原因 | 修复指南 |
| :--- | :--- | :--- |
| `DSL[MissingAttribute]` | 缺少强制要求的属性。 | 检查该 XML 标签，补全缺失的属性 (例如 `Canvas` 缺少 `width` 等)。 |
| `DSL[UnknownTag]` | 使用了不支持的节点名。 | 确保节点名大小写敏感，且在核心节点列表中。 |
| `DSL[ExpressionEval]` | 表达式语法错误，或引用了未声明/拼写错误的变量。 | 检查 `@{...}` 语法或纯表达式字符串中的变量名是否存在于当前作用域中。 |
| `DSL[AttributeNull]` | 必填属性表达式成功执行但返回 `null`。 | 使用 `if()` 兜底处理可能为 null 的变量；或检查数据源中字段是否确实为空。 |
| `System.InvalidCastException` | 表达式类型转换失败 (如字符串转整形)。 | 在 XML 中确保传递给参数的值合法，或显式使用强制转换/默认值。 |
| `DSL[Enum]` | 枚举属性的表达式结果不在允许值集合内（如 `direction`、`alignItems`、`alignContent`、`resampler`）。 | 对照异常中的 `allowed` 修正拼写或取值；变量表达式需保证最终结果为合法枚举字符串。 |
| `DSL[ConditionalOrphan]` | `<ElseIf>` 或 `<Else>` 未紧跟在同级 `<If>` / `<ElseIf>` 后，出现了孤立分支。 | 调整节点顺序，确保分支链形如 `If -> ElseIf* -> Else?`，且它们处于同一父节点下。 |
| `DSL[ConditionalOrder]` | 条件链顺序非法（如 `Else` 后还有 `ElseIf`，或出现多个 `Else`）。 | 将条件链改为合法顺序：最多一个 `Else`，且必须位于链末尾。 |
| `DSL[ForItems]` | `For.items` 的结果为 `null`、`bool`、负数、非整数小数、超大整数（超过 `int.MaxValue`），或既不是集合也无法转换为整数。 | 让 `items` 返回集合或非负整数；若是数值循环，确保值为整数（如 `3` / `3.0`，而非 `2.5`）；避免使用布尔值；检查变量是否存在并使用 `[]` 语法（如 `[pageRecords]`）。 |
| `DSL[IncludeCircular]` | 模板通过 `<Include>` 形成了循环引用（如 `A -> B -> A`）。 | 打开异常中的 include 链路，移除回环引用，将共享片段改为单向依赖。 |
| `DSL[IncludeOutsideRoot]` | `<Include src>` 解析后的路径越出布局根目录，被安全策略拒绝。 | 将被引入文件移动到布局根目录内，或调整 `src` 为根目录内的合法相对路径。 |

**调试技巧**：
- **占位调试法**：在复杂布局内加入 `<Text value="@{ToString([myVar])}" color="#FF0000" />` 以直接在画布上输出当前变量的状态或类型。
- **颜色调试法**：为各个透明的容器 (如 `Stack` 或 `Layer`) 内挂载带明显背景色的 `Canvas` 次级节点，以可视化其实际尺寸及排版结果。

---

## 性能优化建议与最佳实践

1. **避免深层递归表达式**：
   - 尽量使用 `<Set var="safeRating" value="[userInfo.Rating] + 100" />` 预先缓存需要重复计算的长表达式。
   - `<Set>` 的运算仅在解析期执行一次，能大幅降低渲染树构建期间的 `NCalc` 性能损耗。
2. **合理使用重采样算法 (`resampler`)**：
   - `<Resized>` 节点默认使用高质量的 `Lanczos3`，它在大幅度缩小图片时性能开销较大。
   - 对于非关键装饰图片或图标，可改用 `Bicubic` 或 `NearestNeighbor` (若风格合适) 降低光栅化延迟：`resampler="Bicubic"`。
3. **提取组件与复用 (`Include`)**：
   - 使用 `<Include src="relative_path.xml" />` 切分庞大复杂的页面 XML 文件（例如将单独的成绩条提取为 `record_bar.xml`），提升团队代码的维护性和可读性。
4. **尽早执行 DOM 剪枝**：
   - 将条件判断放置于层级较高的节点中：优先使用包裹外部的 `<If rule="[score] > 100">...</If>`，而不是在其内部各节点分别判断或隐藏透明度，这能显著降低解析与内存占用。
5. **字符串拼装优化**：
   - 当仅需要拼接字符串时，优先使用模板语法 `value="ID: {userInfo.Name}"`（SmartFormat 驱动，性能极高）。
   - 尽量减少启动完整的表达式引擎模式如 `value="@{'ID: ' + [userInfo.Name]}"`。

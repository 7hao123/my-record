# 低代码可视化画布项目 - 面试亮点分析

> 本文档从大厂面试视角系统性总结项目的技术深度、架构设计能力和工程复杂度，可直接用于简历项目描述、面试项目讲解和架构设计问题回答。

---

## 一、项目定位与业务价值

### 1.1 一句话定位

> 基于配置驱动的低代码可视化画布系统，支持通过拖拽物料快速搭建数据报表页面，实现"所见即所得"的报表设计体验。

### 1.2 解决的核心问题

**传统报表开发痛点：**
- 每个数据报表页面都需要前端开发介入，开发周期长、迭代成本高
- 非技术用户无法独立完成报表搭建，依赖开发资源
- 样式调整、布局修改需要重新发版，灵活性差

**低代码方案价值：**
- 将页面从"代码描述"转变为"配置描述"，实现配置即页面
- 拖拽式交互降低使用门槛，业务人员可独立完成报表搭建
- 配置化设计支持快速迭代，无需发版即可修改页面

---

## 二、核心架构设计

### 2.1 整体分层架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Pages（页面层）                           │
│         EditLayout / ViewLayout / PreviewLayout             │
├─────────────────────────────────────────────────────────────┤
│                    Layout（布局层）                          │
│    Header（工具栏）│ LeftSidebar（资产）│ RightSidebar（配置）│
├─────────────────────────────────────────────────────────────┤
│                    Canvas（画布层）                          │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  MoveableBox + Selecto（交互控制层）                   │  │
│  │  ├─ BoxList（条件渲染）                               │  │
│  │  │   └─ Box → BoxContent → ComponentMap[type]        │  │
│  │  └─ 对齐线、参考线、旋转指示器                         │  │
│  └───────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│                    State（状态层）                           │
│        Redux Toolkit + redux-undo（支持撤销/重做）           │
├─────────────────────────────────────────────────────────────┤
│                    Data（数据层）                            │
│           React Query（服务端状态缓存与管理）                 │
├─────────────────────────────────────────────────────────────┤
│                    Service（服务层）                         │
│               Axios + API（后端数据接口）                    │
└─────────────────────────────────────────────────────────────┘
```

**架构设计亮点：**

1. **职责分离清晰**：画布层只负责交互控制，渲染层只负责组件展示，状态层集中管理数据流
2. **双状态管理策略**：Redux 管理 UI 状态（组件列表、选中状态），React Query 管理服务端状态（数据缓存）
3. **交互与渲染解耦**：MoveableBox 作为交互控制器，不关心具体渲染的组件类型

### 2.2 核心模块关系

```
用户操作 → MoveableBox（拖拽/选择）→ dispatch(Redux Action)
                                            ↓
                                   editorSlice（状态更新）
                                            ↓
                                   redux-undo（历史记录）
                                            ↓
                              Selectors（memoized 选择器）
                                            ↓
                              BoxList → Box → BoxContent
                                            ↓
                              ComponentMap[type]（组件渲染）
                                            ↓
                              useBoxData（React Query 数据获取）
```

---

## 三、物料体系设计（插件化架构）

### 3.1 设计思想

采用**类 VSCode 插件式**的组件注册机制，每个物料都是 self-contained 的独立单元：

```javascript
// 组件类型映射表 - 类似插件注册表
const componentMap = {
  // 图表类
  lineChart: LineChart,
  barChart: BarChart,
  pieChart: PieChart,
  // 表格类
  detailTable: DetailTable,
  groupTable: GroupTable,
  // 筛选器类
  dimensionFilter: DimensionFilter,
  indicatorFilter: IndicatorFilter,
  // ...30+ 种组件
};

// 分发器 - 根据类型渲染对应组件
const BoxContent = ({ box }) => {
  const Component = componentMap[box.type];
  return <Component box={box} />;
};
```

### 3.2 物料元数据定义

每个物料通过统一的 Schema 描述其特性：

```javascript
{
  type: 'barChart',
  label: '柱形图',
  width: 500,
  height: 400,

  // 配置 Schema
  chartConfig: barChartConfig,
  dateRangeConfig: privateDateRangeConfig,

  // 数据字段定义
  dimensions: [],
  indicators: [],

  // 行为约束（Checker 模式）
  switchableCheckers: [partialRight(lte, 1), partialRight(gte, 1)],
  requestableCheckers: [partialRight(gte, 0), partialRight(gte, 1)],
  addableCheckers: [partialRight(lte, 1), partialRight(lte, Infinity)],
}
```

**Checker 模式的设计价值：**
- 通过函数式约束定义组件的数据配置规则
- `switchableCheckers`：控制组件类型切换的条件
- `requestableCheckers`：控制何时发起数据请求
- `addableCheckers`：控制维度/指标的添加数量限制

### 3.3 物料扩展机制

**新增物料的步骤：**
1. 在 `constants/box.js` 中声明物料元数据
2. 在 `components/` 下创建组件目录
3. 实现核心文件：`Component.jsx`、`useXXXOption.js`、`XXXConfig.js`、`XXXConfigTabs.jsx`
4. 在 `BoxContent.jsx` 的 componentMap 中注册

**扩展性保证**：无需修改核心画布代码，实现真正的"开放封闭原则"

---

## 四、Schema 配置驱动设计

### 4.1 配置结构设计

每个图表组件的配置采用 ECharts option 的子集 + 扩展字段：

```javascript
// 柱状图配置 Schema
const barChartConfig = {
  color: defaultColors,
  variant: 'regular', // 支持 'stacked', 'stackedNormalized', 'mirrored'
  grid: { left: 20, top: 20, right: 20, bottom: 20, containLabel: true },
  legend: { show: true, position: 'top' },
  tooltip: { trigger: 'axis', axisPointer: { type: 'shadow' } },
  xAxis: { type: 'category', axisLine: { show: true }, ... },
  yAxis: { type: 'value', splitNumber: 5, ... },
  series: [],
};
```

### 4.2 配置到视图的转换

采用 **Hook + 纯函数** 的模式实现 Schema 到 ECharts Option 的转换：

```javascript
const useBarChartOption = (box, data) => {
  const { chartConfig, sortConfig } = box;
  const { currentData = [] } = data ?? {};

  // 1. 基于 Schema 克隆配置
  let option = structuredClone(chartConfig);

  // 2. 动态构建 categories
  let categories = currentData.map(row => first(row));
  option.xAxis.data = categories;

  // 3. 构建 series
  option.series = indicators.map((indicator, index) => ({
    type: 'bar',
    name: getIndicatorDisplayName(indicator),
    data: currentData.map(row => tail(row).at(index)),
  }));

  // 4. 处理变体（堆叠、归一化、镜像对比）
  if (chartConfig.variant === 'stacked') {
    option.series.forEach(item => item.stack = 'total');
  }

  return { option };
};
```

**设计价值：**
- 配置与渲染逻辑分离，配置可序列化存储
- 支持配置版本升级和向后兼容
- 纯函数设计便于单元测试

---

## 五、画布交互系统

### 5.1 核心交互能力

基于 `react-moveable` + `react-selecto` 构建完整的画布交互体系：

| 能力 | 实现方案 | 技术要点 |
|------|---------|---------|
| 框选 | Selecto | `selectableTargets`、`toggleContinueSelect` |
| 拖拽移动 | Moveable | `draggable`、`onRenderEnd` |
| 缩放变形 | Moveable | `resizable`、`keepRatio` |
| 旋转 | Moveable | `rotatable`、`rotationPosition` |
| 对齐辅助线 | Moveable | `snappable`、`elementGuidelines` |
| 组合操作 | combinationId | 共享 ID 实现组合 |

### 5.2 交互状态同步

```javascript
// 拖拽结束时同步状态到 Redux
onRenderEnd={(e) => {
  const nextProps = getNextProps(e); // 计算新位置

  // 检测是否拖入/拖出 Tabs 容器
  const isDraggedIn = !isInside(prevBox, container) && isInside(nextBox, container);
  const isDraggedOut = isInside(prevBox, container) && !isInside(nextBox, container);

  // 自动绑定/解绑 paneId
  if (isDraggedIn) {
    payload.properties = { ...nextProps, tabsId: container.id, paneId: activePane.id };
  }

  dispatch(updateBoxProperties(payload));
}}
```

### 5.3 几何计算工具

实现了完整的几何计算工具集：

```javascript
// 碰撞检测 - 考虑旋转的边界框相交判断
export function isOverlapping(box1, box2) {
  const box1Points = getBoundingPoints(box1); // 获取旋转后的四个顶点
  const [left1, top1, right1, bottom1] = getBoundingBox(box1Points);
  // ... AABB 相交检测
}

// 包含判断 - 面积交集 >= 50% 视为"在容器内"
export function isInside(box1, box2) {
  const overlapArea = overlapX * overlapY;
  const area1 = width1 * height1;
  return overlapArea >= area1 * 0.5;
}
```

---

## 六、状态管理设计

### 6.1 Redux Slice 组织

```javascript
// 核心编辑状态 + 撤销/重做支持
editor: undoable(editorReducer, {
  limit: 50, // 历史记录上限
  filter: excludeAction([...]) // 过滤不需要记录的 action
})

// 状态结构
{
  boxList: [...],           // 组件实例列表
  canvas: { width, height, backgroundColor },
  isDirty: false,           // 脏检查
  reportId: null,
}
```

### 6.2 Selector 优化

使用 `createSelector` 实现 memoized 选择器，避免不必要的重渲染：

```javascript
export const boxListSelector = createSelector(
  editorSelector,
  (editor) => uniqBy(editor.boxList, 'id')
);

export const selectedBoxSelector = createSelector(
  boxListSelector,
  (boxList) => boxList.find(box => box.selected)
);
```

### 6.3 React Query 数据缓存

```javascript
const { data, isLoading, isFetching } = useQuery({
  queryKey: [box.type, 'data', box.id, params],  // 精细化缓存键
  queryFn: () => getReportData(params),
  enabled: requestable && !hasSnapshot,           // 条件请求
  placeholderData: placeholderDataMap[box.type], // 占位数据
  retry: false,
  refetchOnWindowFocus: false,
});
```

**缓存策略设计：**
- 按 `(type, id, params)` 三元组精确缓存
- 支持快照数据覆盖网络请求
- 占位数据提升首次渲染体验

---

## 七、技术难点与解决方案

### 难点一：画布节点性能优化

**问题背景：**
画布中可能存在大量组件（50+），每次状态变更都可能触发全量重渲染。

**核心挑战：**
- 拖拽过程中需要实时更新位置，频繁 dispatch 导致性能问题
- 多个图表同时渲染，DOM 操作密集

**解决方案：**

1. **拖拽时仅更新 CSS Transform**，结束时才同步 Redux：
```javascript
onRender={(e) => {
  e.target.style.cssText += e.cssText; // 实时更新样式
}}
onRenderEnd={(e) => {
  dispatch(updateBoxProperties(payload)); // 结束时同步状态
}}
```

2. **Selector Memoization**：避免 boxList 变化导致无关组件重渲染

3. **条件渲染优化**：只渲染当前激活 Pane 内的组件
```javascript
const useBoxList = (boxList) => {
  const activePaneIds = /* 获取激活的 pane */;
  return boxList.filter(box => {
    if (box.paneId && !activePaneIds.includes(box.paneId)) return false;
    return true;
  });
};
```

---

### 难点二：Tabs 容器的嵌套交互

**问题背景：**
Tabs 容器内可以放置其他组件，需要处理拖入/拖出的交互逻辑。

**核心挑战：**
- 如何判断组件是否"在"容器内？
- 拖入/拖出时如何自动绑定/解绑关系？
- Tabs 容器移动时，内部组件如何跟随移动？

**解决方案：**

1. **面积交集判定**：采用 50% 阈值判断"包含"关系
```javascript
export function isInside(box1, box2) {
  const overlapArea = overlapX * overlapY;
  return overlapArea >= area1 * 0.5;
}
```

2. **创建时自动绑定**：
```javascript
const checkAndBindToTabs = (box, boxList) => {
  for (const containerBox of boxList) {
    if (containerBox.type !== 'tabs') continue;
    if (isInside(box, containerBox)) {
      box.tabsId = containerBox.id;
      box.paneId = containerBox.panes.find(p => p.active).id;
      break;
    }
  }
};
```

3. **联动移动**：Tabs 移动时计算偏移量，批量更新内部组件位置

---

### 难点三：组件类型动态切换

**问题背景：**
柱状图可切换为折线图、饼图等，需要保留数据配置，只替换渲染逻辑。

**核心挑战：**
- 不同图表类型的配置结构不同
- 如何做到"配置迁移"？

**解决方案：**

采用 **配置合并** 策略：
```javascript
switchBoxType: (state, action) => {
  const { from, to } = action.payload;
  let box = { type: to };

  // 保留通用配置，替换类型特定配置
  if (to === 'detailTable' || to === 'groupTable') {
    merge(box, omit(from, 'type', 'tableConfig'), { tableConfig: config[to] });
  } else {
    merge(box, omit(from, 'type', 'chartConfig'), { chartConfig: config[to] });
  }

  set(state.boxList, index, box);
}
```

---

### 难点四：数据请求参数的复杂构建

**问题背景：**
每个组件的数据请求需要综合考虑：维度、指标、日期范围、筛选条件、排序配置、钻取状态等。

**核心挑战：**
- 参数来源分散（组件自身、全局筛选器、关联筛选器）
- 需要处理复杂的条件组合逻辑

**解决方案：**

设计 `useRequestParams` Hook 统一处理：
```javascript
export const useRequestParams = (box) => {
  const { dimensions, indicators, sortConfig, paginationConfig } = box;
  const { dateRange } = useBoxDateRange(box);
  const dimensionFilters = useSelector(selectDimensionFilters);
  const { dimensionIds, drilldownCondition } = useDrilldown(box);

  // 合并筛选条件
  const filterConditions = dimensionFilters
    .filter(filter => !filter.config.excludedBoxIds.includes(box.id))
    .map(filter => ({ /* 构建条件 */ }));

  return {
    params: {
      dimensionIds,
      indicatorVos: [...],
      date: { startDate, endDate },
      dimensionConditionGroup: [...],
      ...sortConfig,
      ...paginationConfig,
    }
  };
};
```

---

### 难点五：撤销/重做的状态隔离

**问题背景：**
需要支持撤销/重做，但并非所有状态变更都需要记录。

**核心挑战：**
- 哪些操作需要记录？（组件增删改）
- 哪些操作不需要？（选中状态变更、临时 UI 状态）
- 如何避免历史记录膨胀？

**解决方案：**

1. **redux-undo 集成**：只对 editorSlice 启用
```javascript
editor: undoable(editorReducer, {
  limit: 50,
  filter: excludeAction(['selectBox', 'selectBoxes'])
})
```

2. **isDirty 脏检查**：区分"需要保存的变更"和"临时变更"

---

## 八、工程化实践

### 8.1 代码组织

```
src/
├── app/                    # Redux Store 配置
├── features/               # Redux Slices（按功能域拆分）
├── layout/                 # 布局组件（三栏式）
├── components/             # 物料组件（插件式）
│   └── bar-chart/          # 每个物料独立目录
│       ├── BarChart.jsx        # 渲染组件
│       ├── barChartConfig.js   # 配置 Schema
│       ├── BarChartConfigTabs.jsx # 配置面板
│       ├── useBarChartOption.js   # 数据转换 Hook
│       └── BarChartLoading.jsx    # 加载状态
├── hooks/                  # 业务逻辑 Hooks
├── utils/                  # 工具函数
├── constants/              # 常量定义
└── requests/               # API 请求
```

### 8.2 技术栈选型

| 层级 | 技术选型 | 选型理由 |
|------|---------|---------|
| UI 框架 | React 18 | 主流、生态丰富 |
| 状态管理 | Redux Toolkit + redux-undo | 复杂状态 + 撤销重做 |
| 服务端状态 | React Query | 缓存、自动重试 |
| 拖拽交互 | react-moveable + react-selecto | 功能完备、性能优秀 |
| 图表库 | ECharts | 功能强大、配置灵活 |
| 表格库 | @visactor/react-vtable | 虚拟滚动、百万级数据 |
| 样式 | Tailwind CSS + Antd | 快速开发 + 统一设计 |

### 8.3 类型系统

项目使用 JavaScript + JSDoc 类型注释，关键数据结构有明确的类型定义，便于后续迁移 TypeScript。

---

## 九、可扩展性设计

### 9.1 物料扩展

当前架构完全支持新增物料类型，无需修改核心代码：
- 在 `constants/box.js` 声明元数据
- 在 `components/` 创建组件
- 在 `componentMap` 注册

### 9.2 图表库替换

图表渲染逻辑封装在 `useXXXOption` Hook 中，如需替换图表库：
1. 修改 Hook 返回格式
2. 修改 Chart 包装组件

### 9.3 平台化演进方向

如需升级为企业级低代码平台，需要补充：
- 权限系统（组件级、页面级权限控制）
- 模板与页面复用（模板库、组件库）
- 版本管理（页面版本、发布回滚）
- 多人协作（协同编辑、锁机制）
- 出码能力（将配置导出为代码）

---

## 十、面试重点总结

### 10.1 可以深挖的方向

1. **架构设计**
   - 为什么采用双状态管理（Redux + React Query）？
   - 如何实现组件的插件化注册？
   - 撤销/重做是如何实现的？

2. **交互实现**
   - 拖拽系统是如何设计的？
   - 如何实现对齐辅助线？
   - Tabs 容器的嵌套交互如何处理？

3. **性能优化**
   - 画布节点多时如何保证性能？
   - 数据请求如何做缓存？
   - 选择器如何避免重渲染？

4. **Schema 设计**
   - 配置结构是如何设计的？
   - 如何实现配置到视图的转换？
   - 配置如何做版本兼容？

### 10.2 简历描述示例

> **低代码可视化画布平台** | 前端核心开发
> - 设计并实现基于配置驱动的低代码画布系统，支持 30+ 种物料的拖拽搭建
> - 采用插件化架构设计物料体系，实现组件热插拔，扩展新物料无需修改核心代码
> - 基于 react-moveable 实现完整的画布交互系统，包括拖拽、缩放、旋转、对齐辅助线、组合操作
> - 设计双状态管理方案（Redux + React Query），Redux 管理 UI 状态，React Query 管理数据缓存
> - 集成 redux-undo 实现撤销/重做功能，支持 50 步历史记录
> - 实现 Tabs 容器嵌套交互，支持组件拖入/拖出自动绑定

### 10.3 项目讲解框架

1. **背景与价值**（1分钟）
   - 解决什么问题？传统开发的痛点
   - 核心使用场景

2. **架构设计**（2分钟）
   - 整体分层
   - 核心模块关系
   - 技术选型

3. **技术亮点**（3分钟）
   - 挑一个难点深入讲
   - 问题 → 方案 → 实现 → 效果

4. **总结与展望**（1分钟）
   - 当前架构的优势
   - 后续演进方向

---

## 附录：关键文件索引

| 文件 | 作用 | 亮点 |
|------|------|------|
| `features/editorSlice.js` | Redux 核心状态 | 复杂 Reducer、Selector 设计 |
| `layout/main/MoveableBox.jsx` | 画布交互核心 | Moveable + Selecto 集成 |
| `layout/main/BoxContent.jsx` | 组件分发器 | 插件式注册 |
| `constants/box.js` | 物料元数据定义 | Checker 模式 |
| `hooks/useBoxData.js` | 数据获取 | React Query 封装 |
| `hooks/useRequestParams.js` | 请求参数构建 | 复杂条件组合 |
| `utils/box.js` | 几何计算工具 | 碰撞检测、包含判断 |

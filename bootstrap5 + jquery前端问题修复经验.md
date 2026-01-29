# 前端问题修复经验

## 0. 项目技术栈说明

### 核心框架与库
*   **基础库**: jQuery (主要用于 DOM 操作、事件处理及插件调用)
*   **UI 框架**: Bootstrap 5 (基于 `data-bs-*` 属性及 class 命名判断)
*   **模板引擎**: CodeIgniter 4 PHP Views (服务端渲染 HTML)

### 构建工具与开发环境
*   **构建模式**: 传统多页应用 (MPA)，无现代前端构建工具 (Webpack/Vite) 配置。
*   **资源管理**: 静态资源托管于 `public/theme/` 目录，通过 PHP 常量 (`TEMPLATE_PATH`) 引入。
*   **依赖管理**: 前端依赖主要以 Vendor 形式直接引入 (`vendor.min.js`)，无 `package.json` 包管理。

### 样式与UI组件
*   **UI 模板**: Color Admin Responsive Admin Template
*   **CSS 方案**: 原生 CSS + Bootstrap Utility Classes
*   **图标库**: FontAwesome
*   **常用插件**:
    *   **Select2**: 增强型下拉选择框
    *   **SweetAlert**: 弹窗提示组件
    *   **PerfectScrollbar**: 自定义滚动条插件 (由 `data-scrollbar="true"` 触发)
    *   **DataTables**: 增强型表格插件 (部分复杂表格使用)

### 测试工具
*   **现状**: 当前项目未配置前端自动化测试 (Unit/E2E) 工具。

### 代码质量与规范
*   **现状**: 未配置 ESLint/Prettier 等前端代码静态检查工具，遵循团队内部约定。

## 1. 表格滚动与固定列问题

### 适用技术栈
*   **Bootstrap 5** (表格基础样式)
*   **Native CSS** (核心解决方案)
*   **jQuery** (部分 DOM 操作)
*   **排除项**: 本方案旨在移除 **PerfectScrollbar** 和 **DataTables** 对滚动的干扰。

### 问题现象
1.  **鼠标滚轮无法滚动**：在表格区域内滚动鼠标滚轮时，页面或表格无响应。
2.  **固定列内容透视**：在左右或上下滑动时，固定列（Sticky）下方的内容会透过固定列显示出来，或者在固定列之间存在间隙导致看到下方内容。

### 原因分析
1.  **滚动冲突**：
    *   父容器（如 `.app-content-padding`）上添加了 `data-scrollbar="true"` 属性，触发了自定义滚动插件（如 PerfectScrollbar 或类似），该插件可能拦截了滚轮事件或与 DataTables 等插件冲突。
    *   同时使用了 DataTables 的 `scrollY` 和自定义 CSS 的 `overflow`，导致多重滚动容器。
2.  **透视与间隙**：
    *   **透明背景**：CSS `position: sticky` 元素默认背景透明，滚动时下方内容自然可见。
    *   **渲染间隙**：表格默认的 `border-collapse: collapse` 模型在某些浏览器渲染 sticky 元素时，会产生微小的像素级间隙，导致下方内容“漏”出来。
    *   **高度计算误差**：多行表头（sticky）时，如果第二行的 `top` 值不是严格等于第一行的高度，或者高度计算依赖内容撑开导致小数像素差异，会出现水平缝隙。
    *   **Z-index 层级混乱**：未正确设置表头、固定列、普通内容的 `z-index` 层级关系。

### 解决方案

#### 1. 修复鼠标滚轮滚动
*   **移除干扰属性**：删除父容器上的 `data-scrollbar="true"` 和 `overflow-hidden`，恢复原生滚动行为。
    ```html
    <!-- 修改前 -->
    <div class="app-content-padding flex-grow-1 overflow-hidden" data-scrollbar="true" ...>
    <!-- 修改后 -->
    <div class="app-content-padding flex-grow-1">
    ```
*   **使用原生 CSS 滚动**：在表格容器上设置固定高度和溢出属性。
    ```css
    .qc-result-table-scroll {
        max-height: calc(100vh - 280px); /* 根据实际情况调整 */
        overflow: auto;
        position: relative; /* 建立定位上下文 */
    }
    ```

#### 2. 修复固定列透视与间隙
*   **切换边框模型**：将表格的边框模型改为 `separate`，避免 collapse 带来的渲染间隙。
    ```css
    .qc-result-table-scroll .table {
        border-collapse: separate;
        border-spacing: 0;
        /* ... */
    }
    ```
*   **设置不透明背景**：为所有 sticky 元素（th, td）设置明确的背景色（如 `#fff` 或 `#f8f9fa`）。
    ```css
    .qc-result-table-scroll thead th {
        background-color: #f8f9fa !important;
    }
    ```
*   **精确控制高度与定位**：对于多行表头，强制指定高度，并据此设置第二行的 `top` 值。
    ```css
    /* 第一行表头 */
    .qc-result-table-scroll .table thead tr:nth-child(1) th {
        top: 0;
        z-index: 101;
    }
    /* 第二行表头 */
    .qc-result-table-scroll .table thead th {
        height: 45px; /* 固定高度 */
    }
    .qc-result-table-scroll .table thead tr:nth-child(2) th {
        top: 45px; /* 严格等于第一行高度 */
        z-index: 100;
    }
    ```
*   **Z-index 层级规范**：
    *   **交叉点（左上角固定表头）**：`z-index: 110` (最高)
    *   **固定表头**：`z-index: 100`
    *   **固定列**：`z-index: 10`
    *   **普通内容**：默认

#### 3. 悬停效果优化
*   如果使用了 `rgba` 半透明色作为 hover 背景，在固定列上会再次导致透视。应改为不透明的实色近似值。
    ```css
    /* 修改前 */
    background-color: rgba(13, 110, 253, 0.05);
    /* 修改后 */
    background-color: #F3F8FF;
    ```

## 2. 临床质检结果页固定列重叠/颜色不一致

### 适用技术栈
*   **Native CSS** (Sticky 定位模型)
*   **Bootstrap 5** (表格条纹样式 `.table-striped`)

### 问题现象
1. 横向滚动时，浅色行内容与第三列固定列发生重叠。
2. 鼠标悬停浅色行时，前三列背景色与同一行其它列不一致。

### 原因分析
1. **固定列层级/渲染模型不一致**：表格仍使用 `border-collapse: collapse`，配合 `position: sticky` 时容易出现像素缝隙与覆盖层级混乱，导致滚动时内容“穿透/叠加”。
2. **固定列与表头/正文共用选择器**：同时给 `th/td` 设置 sticky 与 `z-index`，未区分表头与正文，交叉区域层级不稳定。
3. **背景色由行变量驱动但未与表格条纹统一**：行背景与固定列背景计算不一致，滚动/hover 时出现色差。
4. **列宽/left 位置与实际列宽不一致**：固定列 `left` 偏移与列宽不匹配时，滚动后第三列容易出现遮挡或空隙。

### 正确写法（参考数据逻辑质检结果页）
> 核心原则：**表格使用 separate 模型** + **表头/正文固定列分开设置** + **固定列宽度与 left 偏移严格匹配** + **奇数行固定列显式背景**。

```css
/* 表格基础 */
.qc-result-table-scroll .table {
  border-collapse: separate;
  border-spacing: 0;
  width: max-content;
  min-width: 100%;
  margin-bottom: 0;
}

/* 表头固定 */
.qc-result-table-scroll .table thead th {
  position: sticky;
  top: 0;
  z-index: 100;
  background-color: #f8f9fa !important;
  border-bottom: 2px solid #dee2e6;
}

/* 固定列：仅 tbody */
.qc-result-table-scroll tbody td:nth-child(1),
.qc-result-table-scroll tbody td:nth-child(2),
.qc-result-table-scroll tbody td:nth-child(3) {
  position: sticky;
  z-index: 10;
  background-color: #fff;
  border-right: 1px solid #dee2e6;
}

/* 固定列交叉区：thead */
.qc-result-table-scroll thead th:nth-child(1),
.qc-result-table-scroll thead th:nth-child(2),
.qc-result-table-scroll thead th:nth-child(3) {
  position: sticky;
  z-index: 110;
  background-color: #f8f9fa !important;
  border-right: 1px solid #dee2e6;
}

/* 固定列宽度与 left 偏移严格匹配 */
.qc-result-table-scroll th:nth-child(1),
.qc-result-table-scroll td:nth-child(1) { left: 0;   width: 100px; }
.qc-result-table-scroll th:nth-child(2),
.qc-result-table-scroll td:nth-child(2) { left: 100px; width: 150px; }
.qc-result-table-scroll th:nth-child(3),
.qc-result-table-scroll td:nth-child(3) { left: 250px; width: 150px; }

/* 奇数行固定列背景修正 */
.table-striped > tbody > tr:nth-of-type(odd) > td:nth-child(1),
.table-striped > tbody > tr:nth-of-type(odd) > td:nth-child(2),
.table-striped > tbody > tr:nth-of-type(odd) > td:nth-child(3) {
  background-color: #f2f4f5;
}
```

### 备注
- 若表头或固定列需改宽度，**必须同步调整对应 `left` 值**，否则滚动时会出现遮挡或错位。
- 只对 `tbody` 固定列设置 `sticky`，避免表头与正文层级相互影响。

## 3. 表格空状态下的列宽塌缩与布局异常

### 适用技术栈
*   **CodeIgniter 4 View** (PHP 动态渲染 colspan)
*   **Native CSS** (选择器优先级控制)
*   **Bootstrap 5** (辅助类如 `.text-center`)

### 问题现象
1.  **无数据时表头宽度不一致**：当表格无数据时，固定列的表头宽度变窄，与有数据时不一致。
2.  **固定列间出现空白间隙**：在无数据状态下，第二列与第三列之间出现明显的空白区域。
3.  **“暂无数据”行显示异常**：提示行被挤压在第一列的位置，无法跨列居中显示。

### 原因分析
1.  **CSS 选择器优先级冲突**：
    *   通用样式（如 `.table thead th:nth-child(2)`）的优先级低于或等于特定场景样式，但在某些情况下（如 DOM 结构变化）被浏览器优先应用。
    *   如果在通用样式中设置了较小的 `min-width`（如 100px），而固定列布局要求较大的宽度（如 150px），就会导致宽度塌缩。
2.  **宽度与 Left 偏移量不匹配**：
    *   `position: sticky` 的列，其 `left` 值是相对于容器左侧的距离。如果前一列的实际宽度小于后一列的 `left` 减去前一列的 `left`，中间就会出现间隙。
    *   例如：第2列 left=100px，第3列 left=250px。如果第2列宽度塌缩为 100px，则 100+100=200px < 250px，产生 50px 间隙。
3.  **Sticky 样式继承**：
    *   无数据时，唯一的 `<td>` 元素（通常带有 `colspan`）位于第一行第一列的位置，默认会匹配到针对 `td:nth-child(1)` 的 sticky 样式。
    *   这导致该单元格被固定在左侧且宽度被限制，无法正常 colspan 铺满整行。

### 解决方案

#### 1. 强化 CSS 选择器优先级与一致性
确保通用列宽定义的优先级足够高（加上父类限定符），且数值与固定列布局严格一致。

```css
/* 错误写法：优先级低，且数值可能冲突 */
.table thead th:nth-child(2) {
    min-width: 100px; /* 导致塌缩 */
}

/* 正确写法：增加父类限定，且数值与 left 偏移量匹配 */
.qc-result-table-scroll .table thead th:nth-child(2) {
    min-width: 150px; /* 确保填满到下一列的起始位置 */
}
```

#### 2. 处理空状态下的 Sticky 继承
为无数据单元格添加专用类（如 `.no-data`），并强制重置其定位和宽度属性。

```html
<!-- HTML -->
<td colspan="10" class="text-center no-data">暂无数据</td>
```

```css
/* CSS */
/* 必须重置 position 和 width，否则会继承第一列的 sticky 属性 */
.qc-result-table-scroll tbody td.no-data {
    position: static !important; /* 取消固定定位 */
    width: auto !important;
    min-width: 100% !important;
    max-width: none !important;
    left: auto !important;
    border-right: none !important;
    background-color: transparent !important;
}
```

#### 3. 动态计算 Colspan
不要写死 `colspan="999"`，建议根据表头列数动态计算，避免潜在的布局警告。

```php
$colCount = !empty($result_title) ? count($result_title) : 1;
// ...
<td colspan="<?php echo $colCount; ?>" ...>
```

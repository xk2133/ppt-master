# 代码生成强制规则

> 源于实际故障，每次生成 HTML 代码前必须逐条检查。

---

## 规则 1：CDN 选择（中国大陆兼容）

```
❌ 禁止：cdnjs.cloudflare.com（GFW 干扰，加载失败导致页面空白）
✅ 必须：cdn.jsdelivr.net
```

```html
<script src="https://cdn.jsdelivr.net/npm/gsap@3.12.5/dist/gsap.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/echarts@5.5.0/dist/echarts.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/html2canvas@1.4.1/dist/html2canvas.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/jspdf@2.5.1/dist/jspdf.umd.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/pptxgenjs@3.12.0/dist/pptxgen.bundle.js"></script>
```

---

## 规则 2：动画安全（防止内容不可见）

### CSS

```css
/* ✅ 必须使用 opacity: 0 */
.anim-ready { opacity: 0; }
/* ❌ 禁止 visibility: hidden — GSAP 无法覆盖 */

/* 降级规则：GSAP 加载失败时直接显示 */
.no-gsap .anim-ready { opacity: 1; }
```

### JS

```js
// 启动时检查 GSAP 可用性
const hasGSAP = typeof gsap !== 'undefined';
if (!hasGSAP) document.body.classList.add('no-gsap');

// ✅ 正确模式
const tl = gsap.timeline({ paused: true });
tl.set(el, { opacity: 0 });
tl.to(el, { opacity: 1, duration: 0.6, ease: 'power2.out' });

// ❌ 禁止 gsap.from() —— 依赖 CSS 初始状态，GSAP 不可用时元素不可见
```

### 父元素关键坑

父元素有 `.anim-ready` 时，动画中**必须显式** `tl.set(parent, {opacity:1})`，否则子元素即使 opacity:1 也不可见。

### 规则 2.1：anim-ready 作用域限制（硬性约束）

> **`anim-ready` 只能用于叶子内容元素。禁止用于容器/布局元素。**

**判断标准：** 如果一个元素包含其他带 `anim-ready` 的子元素，则该元素自身**禁止**有 `anim-ready`。

**允许（叶子元素）：** `.scene-title`、`.card-title`、`.card-body`、`.card`、`.cover-title`、`.s7-quad`、`.s6-tl-item`、图表容器、图片等独立内容元素。

**禁止（容器/布局元素）：** `.scene`、`.content-area`、`.cover-center`、`.mece-container`、`.s2-layout`、`.s3-grid`、`.s4-layout`、`.s5-layout`、`.s6-layout`、`.s7-layout`、`.s8-layout`、`.mece-left`、`.mece-right`、`.s2-left`、`.s2-right`、`.s4-left`、`.s4-right`、`.s6-left`、`.s6-right`、`.s7-left`、`.s7-right`、`.s8-left`、`.s8-right` 等。

**原因：** 容器由 `display: none`/`display: flex`（通过 `.active` 类切换）控制可见性，`anim-ready` 只在需要独立入场动画的元素上使用。容器带 `anim-ready` 会被 `tl.set({opacity:0})` 设为透明，若时间线中缺少对应的 `tl.to({opacity:1})`，其所有子元素的动画将永远不可见——这是 Harness Engineering 演示文稿空白页故障的直接根因。

---

## 规则 3：PDF/PPTX 导出

### 共享辅助函数

```js
// ⚠️ 定义在外层作用域，PDF 和 PPTX handler 共享
// ❌ 禁止定义在单个 handler 闭包内

function resetSceneToFinal(scene) {
  scene.querySelectorAll('.anim-ready, [style*="opacity: 0"]').forEach(el => {
    el.style.opacity = '1';
    el.style.transform = 'none';
  });
}

function setCounterFinal(sceneId) {
  const vals = FINAL_VALUES[sceneId];
  if (!vals) return;
  Object.entries(vals).forEach(([id, val]) => {
    const el = document.getElementById(id);
    if (el) el.textContent = val;
  });
}
```

### PDF

- 页面尺寸：`[297, 167]mm`（16:9）
- 捕获 `scene` 元素（非 stage）
- 捕获前：`resetSceneToFinal()` + `setCounterFinal()` + 图表初始化 + ≥400ms 等待

### PPTX（混合模式）

- 页面尺寸：`13.333" x 7.5"`
- 步骤：`extractTextBlocks` → `hideTextForCapture` → `html2canvas` → `restoreTextAfterCapture` → `addImage` + `addText`
- 坐标转换：`x_inch = px * 13.333 / 1200`，`y_inch = px * 7.5 / 675`
- 字号转换：`fontSize_pt = px * 0.8`
- 颜色转换：`rgbToHex()` 将 `rgb(r,g,b)` → `"RRGGBB"`
- ECharts 图表文字在 canvas 上，不可提取为可编辑文字
- 完整实现代码见 `.claude/export-code.md`

---

## 规则 4：高度计算（防溢出）

```css
--avail-h: calc(var(--stage-max) * 9 / 16);
--scene-gap: 20px;
--avail-inner: calc(var(--avail-h) - var(--scene-gap));
--header-h: calc(var(--avail-inner) * 0.25);
--content-h: calc(var(--avail-inner) * 0.75);

.scene { gap: var(--scene-gap); }  /* 用 gap 而非 margin-bottom */
.scene-header { height: var(--header-h); }
.content-area { height: var(--content-h); flex-shrink: 0; }  /* flex-shrink:0 防压缩 */
.card-body { flex: 1; overflow: hidden; }
```

### 强制卡片高度校验

> ⚠️ 生成含 `.hier-card`（层级卡片）或 `.column`（Grid 卡片）的布局时，必须人工计算最小高度，禁止凭感觉设值。

**核心原则：用 `min-height` 替代 `height`，让卡片高度随内容自适应。**

**计算公式：**
```
card_min_height = card_padding_top
                + card_padding_bottom
                + card-head 实际高度（图标 + 标题行 + 副标题行，取 flex 行高最大值）
                + card-head margin-bottom
                + card-body 内容高度（逐行累加：标签/描述/列表项 × 行数 + 行间距）
                + 8px 安全余量

CSS:  .card-xx { min-height: <card_min_height>; }  /* 禁用固定 height */
```

**内边距基准（自适应调优起点）：**
```
.card-inner { padding: 16px 24px 14px; }   /* 默认紧凑 */
.card-head  { margin-bottom: 6px; }         /* 默认紧凑 */
```

**校验流程：**
1. 列出卡片内所有可见元素及其估算高度
2. 代入公式得到 `card_min_height`（向上取整到 5 的倍数）
3. 用 `min-height` 写入 CSS，验证内容需 ≥ 实际渲染高度
4. 链式验证：L1.bottom + 连线 + L2.min-height + 连线 + L3.min-height ≤ content-area.height
5. 若溢出：优先压缩 padding / margin（最大可至 12px/8px/4px），其次调整 `--fs-*` 变量降级

```
L1 bottom ≈ L1.top + L1.min-height
L2 top   = L1 bottom + 连线高度
L2 bottom ≈ L2.top + L2.min-height
L3 top   = 水平分支线 top + 分支线高度
L3 bottom ≈ L3.top + L3.min-height
L3 bottom ≤ content-area.height  ← 硬性约束
```

---

## 规则 5：ECharts 图表

```js
// 横向柱形图：inverse: true + 数据原始顺序（不要 reverse，确保最大值在顶部）
yAxis: { type: 'category', inverse: true, data: categories }
series: [{ type: 'bar', data: values }]

// resize 需要 debounce
let resizeTimer;
window.addEventListener('resize', () => {
  clearTimeout(resizeTimer);
  resizeTimer = setTimeout(() => {
    Object.values(charts).forEach(c => { try { c.resize(); } catch(e) {} });
  }, 200);
});
```

---

## 规则 6：设计要求速查

### 默认模板 — 泰康保险集团（Taikang）
```css
/* 舞台 — 白色背景，无边框 */
.stage {
  width: 1200px; height: 675px; position: relative;
  margin: 40px auto 20px;
  background: #FFFFFF;
  box-shadow: 0 4px 24px rgba(0,0,0,0.10);
  overflow: hidden;
}
body { background: #E0E0E0; }

/* 标题 + 下方渐变装饰线 */
.scene-title {
  position: absolute; left: 65.9px; top: 25.0px;
  font-size: var(--fs-title); font-weight: 700; color: #EE7700;
  font-family: 'Microsoft YaHei', sans-serif;
}
.title-line {
  position: absolute; left: 70.5px; top: 84.1px;
  width: 897.7px; height: 2px;
  background: linear-gradient(90deg, #EE7700, transparent);
}

/* 右下角装饰条（水平）+ 页码 */
.deco-bar { position: absolute; left: 1151.2px; top: 659.0px; width: 48.8px; height: 7.5px; background: #EE7700; }
.page-num { position: absolute; left: 1129.9px; top: 641.2px; font-size: 10px; color: #999; }

/* Logo（必须用 base64 内嵌，禁止引用外部文件路径）
   ⚠️ html2canvas 无法捕获本地文件图片（assets/*.png），PDF/PPTX 导出时 Logo 会丢失。
   生成演示代码时，必须将 assets/taikang-logo.png 转为 base64 data URI 写入 src 属性。
   
   🔴 Logo base64 截断风险（实测 Edit 工具处理 >20KB 超长字符串会静默截断）：
     正确做法 — 用 Node.js 脚本批量替换（见下方），禁止用 Edit 工具替换 base64 字符串。
     验证标准 — taikang-logo.png 的正确 base64 为 24,244 字符，解码后 18,183 字节（328×154px）。
     
     替换脚本模板：
     node -e "
     const fs=require('fs');
     const html=fs.readFileSync('{主题}.html','utf8');
     const b64=fs.readFileSync('assets/taikang-logo-b64.txt','utf8').trim();
     const r=html.replace(/(<img class=\\\"stage-logo\\\" src=\\\")data:image\/png;base64,[^\\\"]+(\\\" alt=\\\"Logo\\\">)/g,'\$1data:image/png;base64,'+b64+'\$2');
     fs.writeFileSync('{主题}.html',r);
     " */
.stage-logo { position: absolute; left: 1011.8px; top: 16.9px; width: 163.9px; height: 77.3px; object-fit: contain; }

/* 内容区 */
.content-area { position: absolute; left: 65.0px; top: 111.2px; width: 1068.8px; height: 492.5px; }
```

### 配色 — Taikang 6色系统
| 用途 | 值 | 说明 |
|------|-----|------|
| accent1 | `#EE7700` | 泰康红/橙 — 标题、主强调、列1顶部条 |
| accent2 | `#FABE00` | 金色 — 辅助强调 |
| accent3 | `#1D2B83` | 深蓝 — 对比色、列2顶部条 |
| accent4 | `#0091DB` | 浅蓝 — 信息色 |
| accent5 | `#3B9C96` | 青绿 |
| accent6 | `#AACE1D` | 黄绿 |
| 正文深 | `#1A1A1A` | 标题/正文 |
| 正文中 | `#555555` | 次要文字 |
| 正文浅 | `#999999` | 辅助文字 |
| 卡片底 | `#F5F5F5` | 卡片背景 |
| 分割线 | `#E8E8E8` | 边框/分割 |

### 字体系统 — Taikang Type Scale

> 所有演示文稿必须使用 CSS 变量统一字号，禁止硬编码 `font-size`。

```css
:root {
  --fs-title:         28px;   /* 主标题 */
  --fs-card-title:    18px;   /* 卡片标题 */
  --fs-card-sub:      13px;   /* 卡片副标题 */
  --fs-body:          14px;   /* 正文 / 描述 / 列表 */
  --fs-section-label: 14px;   /* 段落小标题 — 必须 ≥ --fs-body */
  --fs-tag:           12px;   /* 标签 */
  --fs-small:         10px;   /* 辅助文字 / 页码 */
}
```

| 变量 | 值 | 适用元素 |
|------|-----|----------|
| `--fs-title` | 28px | `.scene-title` |
| `--fs-card-title` | 18px | `.card-title` |
| `--fs-card-sub` | 13px | `.card-subtitle` |
| `--fs-body` | 14px | `.card-desc`、`.card-bullets li`、`.card-icon`、`.nav-info` |
| `--fs-section-label` | 14px | `.section-label`（**必须 ≥ --fs-body，禁止小于正文**） |
| `--fs-tag` | 12px | `.card-role-tag`、`.export-btn` |
| `--fs-small` | 10px | `.page-num` |

**布局自适应规则：**
- 内容量大（>6个要点/卡片）→ `--fs-body: 13px`，`--fs-card-title: 17px`，`--fs-section-label: 13px`（同步缩小，仍 ≥ --fs-body）
- 内容量小（≤2个要点/卡片）→ 使用默认值即可
- 始终通过 CSS 变量调整，严禁逐个元素覆写 `font-size`
- **硬性约束：`--fs-section-label` ≥ `--fs-body`，段落小标题字号禁止小于正文。** 违反此条即为层级错误。

### 卡片 CSS（Taikang）
```css
.column {
  background: #F5F5F5; border-radius: 4px;
  padding: 26px 24px 20px; position: relative; overflow: hidden;
}
.column::before { content: ''; position: absolute; top: 0; left: 0; right: 0; height: 3px; }
.column.col1::before { background: #EE7700; }
.column.col2::before { background: #1D2B83; }
```

### 备选：Dark + Grid（用户明确要求时启用）
```css
.stage { background: #000000; border: 2px solid #444444; border-radius: 16px; }
```
| 背景 | `#000000` | 卡片 bg | `#2d2d2db3` | 强调 | `#00ff88` |

### 布局 CSS（通用）
```css
.grid-2-layout { display: grid; grid-template-columns: 1fr 1fr; gap: 32px; }
.grid-3-layout { display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 20px; }
```
### PPTX EMU→px 换算
```
px = EMU * 1200 / 12192000
```

### 防溢出
```
多层保护：
1. 父容器强制 height 限制
2. line-clamp: 3 限制文字行数
3. 图表高度：Math.min(350, 内容区可用高度 * 0.7)
4. 字体自适应：font-size: clamp(12px, 2vw, 16px)

垂直空间不足时：Grid-3 → Grid-2 → Split → 单列
```

### 标签
```css
.card-tags { display: flex; gap: 8px; flex-wrap: wrap; margin-top: 12px; }
.tag { padding: 2px 10px; border-radius: 100px;
       background: rgba(238,119,0,0.10); color: #EE7700;
       font-size: 11px; border: 1px solid rgba(238,119,0,0.20); }
/* 暗色主题：background: rgba(0,255,136,0.12); color: #00ff88; */
```

### 翻页器
```css
.nav-controls {
  position: fixed; right: 50%; bottom: 20px;
  transform: translateX(calc(var(--stage-max) / 2 + 20px));
  z-index: 9999;
}
```

### 无障碍
- 动画仅 `transform` + `opacity`（60fps）
- 色彩对比度 AAA 级
- ARIA 标签 + 键盘可达
- 图表 resize debounce ≥200ms
- ECharts 容器 `max-height` + `overflow:hidden`
- 图表 legend 位置根据空间自动调整（top/bottom/right）

---

## 规则 7：生成后强制自检（硬性阻断）

> ⚠️ **每次生成 HTML 演示代码后，在告知用户"完成"之前，必须逐条执行以下检查。任何一条未通过，必须先修复再汇报。**

### 检查 A：动画可见性 — 终态可达性（最高优先级）

```
□ 1. 搜索代码中所有 class="… anim-ready …" 的元素（HTML 侧）
□ 2. 对每个带 anim-ready 的元素，确认它在 GSAP 时间线结束时 opacity 能否到达 1：
     - tl.set(el, {opacity:1}) → ✓ 直接可达
     - tl.set(el, {opacity:0}) 且后续 tl.to(el, {opacity:1}) → ✓ 间接可达
     - 仅有 tl.set(el, {opacity:0})，无后续 tl.to({opacity:1}) → ✗ 不可达，硬性阻断
□ 3. 任一元素不可达 → 补齐 tl.to() 将该元素动画到 opacity:1
```

### 检查 A2：容器污染检测（硬性阻断）

```
□ 3a. 扫描所有带 anim-ready 的元素
□ 3b. 对每个元素，检查其子元素是否也包含 anim-ready
□ 3c. 如果父容器和子元素同时存在 anim-ready → 报错：容器污染
      示例：.scene.anim-ready 包含 .cover-title.anim-ready → 违规
□ 3d. 修复方式：从父容器移除 anim-ready，容器由 display 控制可见性
□ 3e. 硬性阻断：发现即必须修复
```

### 检查 B：动画模式

```
□ 4. 全文搜索 "gsap.from(" → 必须为 0 处
□ 5. 全文搜索 "visibility: hidden" → 必须为 0 处（CSS 中）
```

### 检查 C：CDN

```
□ 6. 全文搜索 "cdnjs.cloudflare.com" → 必须为 0 处
□ 7. 全文搜索 "jsdelivr.net" → 必须 ≥ 1 处
```

### 检查 D：导出就绪

```
□ 8. resetSceneToFinal / setCounterFinal 定义在外层作用域（非闭包内）
□ 9. PDF handler 和 PPTX handler 均可访问上述函数
□ 9b. Logo 图片必须为 base64 data URI（禁止 assets/*.png 文件路径），否则 PDF/PPTX 导出丢失 Logo
□ 9c. 🔴 Logo base64 完整性验证（硬性阻断）：
     - 提取每个 .stage-logo src 的 base64 长度 → 必须全部 = 24,244 字符
     - 解码一个 base64 验证 PNG 头（89 50 4E 47）→ 必须通过
     - 验证脚本：
       node -e "const fs=require('fs');const h=fs.readFileSync('{主题}.html','utf8');
       const re=/<img class=\\\"stage-logo\\\" src=\\\"data:image\/png;base64,([^\\\"]+)\\\"/g;let m,i=0;
       while((m=re.exec(h))!==null){i++;if(m[1].length!==24244)console.log('FAIL:scene '+i+' len='+m[1].length);}
       const b=Buffer.from(h.match(re)[1],'base64');
       if(b[0]!==0x89||b[1]!==0x50||b[2]!==0x4E||b[3]!==0x47)console.log('FAIL:invalid PNG header');
       console.log('PASS:'+i+' logos, each '+b.length+'B');"
     - 若未通过 → 用 Node.js 脚本重新替换（见规则 6 Logo 替换脚本模板），禁止用 Edit 工具修 base64
```

### 检查 E：卡片高度（防溢出）

```
□ 10. 对每个卡片，列出：padding(top+bottom) + card-head 实际高度 + margin-bottom + card-body 逐行内容高度
□ 11. card_min_height ≤ CSS 中设定的 height？若不满足 → 增大 height 或减小 padding
□ 12. L3 卡片 bottom ≤ content-area.height（492.5px）？
□ 13. 层级布局额外校验：L1.bottom + 连线 + L2.height + 连线 + L3.bottom ≤ content-area.height
```

### 检查 F：字体层级（防止小标题小于正文）

```
□ 14. 搜索所有段落小标题/分区标签的 font-size（如 .section-label），确认 ≥ --fs-body
□ 15. 搜索所有硬编码的 font-size（非 CSS 变量），逐一确认其值在 Type Scale 中级别合理
□ 16. 若存在 小标题字号 < 正文字号 → 层级错误，必须修复
```

### 执行方式

检查通过 → 告知用户完成。
检查未通过 → 静默修复，重新检查，直到全部通过。不向用户逐条汇报检查过程，只说"已修复"或"已完成"。

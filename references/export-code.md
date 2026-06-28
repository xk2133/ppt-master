# PDF/PPTX 导出 — 完整实现代码

生成 HTML 演示文稿中的 PDF 和 PPTX 导出功能时，直接参考以下实现。

---

## 一、共享辅助函数（外层作用域）

```js
// ══════════════════════════════════════
// 必须定义在 PDF 和 PPTX handler 都能访问的外层作用域
// ❌ 禁止定义在单个 handler 闭包内
// ══════════════════════════════════════

// 每个场景的计数器最终值
const FINAL_VALUES = {
  s3:  { 'kpi-revenue': '371.20', 'kpi-gross': '267.65', 'kpi-op': '168.90',
         'kpi-net': '130.12', 'kpi-nonifrs': '130.84', 'kpi-eps': '9.61' },
  s9:  { 'kpi-cash': '137.75', 'kpi-debt': '0', 'kpi-dar': '29.4%',
         'kpi-ar': '7', 'kpi-inv': '123', 'kpi-headcount': '10,879' },
  // ... 根据实际场景扩展
};

// 清除 GSAP 残留，强制所有元素为最终可见状态
function resetSceneToFinal(scene) {
  if (!scene) return;
  scene.querySelectorAll('.anim-ready, [style*="opacity: 0"]').forEach(el => {
    el.style.opacity = '1';
    el.style.transform = 'none';
  });
}

// 设置计数器为最终值
function setCounterFinal(sceneId) {
  const vals = FINAL_VALUES[sceneId];
  if (!vals) return;
  Object.entries(vals).forEach(([id, val]) => {
    const el = document.getElementById(id);
    if (el) el.textContent = val;
  });
}
```

---

## 二、PDF 导出完整实现

```js
btnPdf.addEventListener('click', async () => {
  if (typeof html2canvas === 'undefined') {
    alert('PDF生成失败：html2canvas 库未能加载，请检查网络连接后刷新。');
    return;
  }
  if (typeof jspdf === 'undefined') {
    alert('PDF生成失败：jsPDF 库未能加载，请检查网络连接后刷新。');
    return;
  }

  btnPdf.textContent = '⏳ 生成中...';
  btnPdf.disabled = true;

  try {
    const { jsPDF } = jspdf;
    const pdf = new jsPDF({ orientation: 'landscape', unit: 'mm', format: [297, 167] });

    const origCurrent = current;

    // 销毁所有 GSAP 时间线
    if (hasGSAP) {
      Object.values(timelines).forEach(tl => { try { tl.kill(); } catch(e) {} });
    }

    const stageEl = document.getElementById('stage');
    const origOverflow = stageEl.style.overflow;
    stageEl.style.overflow = 'visible';

    for (let i = 1; i <= TOTAL; i++) {
      // 只显示当前 scene
      document.querySelectorAll('.scene').forEach(s => { s.style.display = 'none'; });
      const scene = document.getElementById('s' + i);
      if (!scene) continue;
      scene.style.display = 'flex';

      // 重置动画 + 计数器
      resetSceneToFinal(scene);
      setCounterFinal('s' + i);

      // 图表场景需要初始化图表
      if (i === 4) { initS4Charts(); }
      if (i === 6) { initS6Chart(); }

      // 等待图表渲染完成
      await new Promise(r => setTimeout(r, 400));

      // 捕获 scene（非 stage），16:9 比例
      // 默认 Taikang 白色模板，暗色主题改用 '#000000'
      const canvas = await html2canvas(scene, {
        backgroundColor: '#FFFFFF', scale: 2,
        useCORS: true, logging: false
      });

      if (i > 1) pdf.addPage();
      pdf.addImage(canvas.toDataURL('image/png'), 'PNG', 0, 0, 297, 167);
    }

    stageEl.style.overflow = origOverflow;
    switchTo(origCurrent);

    pdf.save('演示文稿.pdf');
  } catch(e) {
    console.error('PDF error:', e);
    alert('PDF生成失败：' + e.message + '\n请检查网络连接后重试。');
    try { switchTo(current || 1); } catch(e2) {}
  }
  btnPdf.textContent = '📥 PDF';
  btnPdf.disabled = false;
});
```

---

## 三、PPTX 导出 — 混合模式完整实现

### 策略
背景图 + 原生文字叠加。文字在 PowerPoint/WPS 中可直接编辑，图表保留为图片。

### 实现步骤
1. `extractTextBlocks(scene)` — 提取所有文字元素及其位置/样式
2. `hideTextForCapture(scene)` — 隐藏文字，保留背景（卡片/网格/图表可见）
3. `html2canvas(scene)` — 捕获背景图
4. `restoreTextAfterCapture(scene)` — 恢复文字
5. `slide.addImage()` + `slide.addText()` — 组合输出

### 完整代码

```js
// ═══════════════════════════════════════════════
// PPTX Export — Hybrid Mode
// ═══════════════════════════════════════════════

// ── 文字提取选择器 ──
// TEXT_SELECTORS：需要提取为可编辑文字的元素
// HIDE_SELECTORS：捕获前需要隐藏的文字元素
const TEXT_SELECTORS = [
  '.scene-header .title', '.scene-header .subtitle',
  '.card-title', '.point-list li',
  '.kpi-label', '.kpi-value', '.kpi-sub', '.kpi-conclusion',
  '.cover-title', '.cover-sub',
  '.cover-num-item .num', '.cover-num-item .label',
  '.mece-question .q-label', '.mece-question .q-text',
  '.branch-title', '.branch-items',
  '.tl-year', '.tl-text',
  '.card-tags .tag'
];

const HIDE_SELECTORS = [
  '.scene-header .title', '.scene-header .subtitle',
  '.card-title', '.card-body', '.card-tags',
  '.kpi-label', '.kpi-value', '.kpi-sub', '.kpi-conclusion',
  '.cover-title', '.cover-sub', '.cover-num-item .num', '.cover-num-item .label',
  '.mece-question', '.branch-title', '.branch-items',
  '.tl-year', '.tl-text'
].join(', ');

// ── 颜色转换：CSS rgb(r,g,b) → hex "RRGGBB" ──
function rgbToHex(rgb) {
  const m = rgb.match(/rgb\((\d+),\s*(\d+),\s*(\d+)\)/);
  if (!m) return 'FFFFFF';
  return [1,2,3].map(i => parseInt(m[i]).toString(16).padStart(2,'0').toUpperCase()).join('');
}

// ── 文字提取（含坐标映射） ──
function extractTextBlocks(scene) {
  const blocks = [];
  const sceneRect = scene.getBoundingClientRect();
  if (sceneRect.width === 0 || sceneRect.height === 0) return blocks;

  const sw = sceneRect.width;
  const sh = sceneRect.height;
  // PPTX slide: 13.333" x 7.5"
  const scaleX = 13.333 / sw;   // px → inch
  const scaleY = 7.5 / sh;

  TEXT_SELECTORS.forEach(sel => {
    scene.querySelectorAll(sel).forEach(el => {
      const text = el.textContent.trim();
      if (!text) return;
      const rect = el.getBoundingClientRect();
      if (rect.width === 0 || rect.height === 0) return;
      const style = getComputedStyle(el);
      if (style.display === 'none' || style.visibility === 'hidden') return;

      const weight = parseInt(style.fontWeight);
      blocks.push({
        text: text,
        x: (rect.left - sceneRect.left) * scaleX,
        y: (rect.top - sceneRect.top) * scaleY,
        w: Math.max(rect.width * scaleX, 0.5),
        h: Math.max(rect.height * scaleY, 0.18),
        fontSize: Math.max(7, parseFloat(style.fontSize) * 0.8),
        color: rgbToHex(style.color),
        bold: !isNaN(weight) && weight >= 600,
        align: style.textAlign === 'center' ? 'center'
             : style.textAlign === 'right' ? 'right' : 'left'
      });
    });
  });
  return blocks;
}

// ── 文字隐藏/恢复 ──
function hideTextForCapture(scene) {
  scene.querySelectorAll(HIDE_SELECTORS).forEach(el => {
    el.dataset.prevOp = el.style.opacity;
    el.style.opacity = '0';
  });
}

function restoreTextAfterCapture(scene) {
  scene.querySelectorAll('[data-prev-op]').forEach(el => {
    el.style.opacity = el.dataset.prevOp;
    delete el.dataset.prevOp;
  });
}

// ── PPTX 导出主循环 ──
btnPptx.addEventListener('click', async () => {
  if (typeof html2canvas === 'undefined') {
    alert('PPTX生成失败：html2canvas 库未能加载，请检查网络连接后刷新。');
    return;
  }
  if (typeof PptxGenJS === 'undefined') {
    alert('PPTX生成失败：PptxGenJS 库未能加载，请检查网络连接后刷新。');
    return;
  }

  btnPptx.textContent = '⏳ 生成中...';
  btnPptx.disabled = true;

  try {
    const pptx = new PptxGenJS();
    pptx.defineLayout({ name:'WIDE', width:'13.333', height:'7.5' });
    pptx.layout = 'WIDE';

    const origCurrent = current;
    const stageEl = document.getElementById('stage');

    // 销毁所有 GSAP 时间线
    if (hasGSAP) {
      Object.values(timelines).forEach(tl => { try { tl.kill(); } catch(e) {} });
    }

    const origOverflow = stageEl.style.overflow;
    stageEl.style.overflow = 'visible';

    for (let i = 1; i <= TOTAL; i++) {
      // 只显示当前 scene
      document.querySelectorAll('.scene').forEach(s => { s.style.display = 'none'; });
      const scene = document.getElementById('s' + i);
      if (!scene) continue;
      scene.style.display = 'flex';

      resetSceneToFinal(scene);
      setCounterFinal('s' + i);

      // 图表场景初始化
      if (i === 4) { initS4Charts(); }
      if (i === 6) { initS6Chart(); }

      await new Promise(r => setTimeout(r, 400));

      // Step 1: 提取文字块
      const textBlocks = extractTextBlocks(scene);

      // Step 2: 隐藏文字 → 捕获背景图
      hideTextForCapture(scene);
      await new Promise(r => setTimeout(r, 60));

      let imgData = null;
      try {
        const canvas = await html2canvas(scene, {
          backgroundColor: '#FFFFFF', scale: 2,
          useCORS: true, logging: false, allowTaint: true
        });
        imgData = canvas.toDataURL('image/png');
      } catch(e) { console.warn('PPTX slide ' + i + ' capture failed:', e); }

      // Step 3: 恢复文字
      restoreTextAfterCapture(scene);

      if (!imgData) continue;

      // Step 4: 构建 slide — 背景图 + 可编辑文字
      const slide = pptx.addSlide();
      slide.addImage({
        data: imgData, x: 0, y: 0, w: '100%', h: '100%',
        sizing: { type:'cover', w:'100%', h:'100%' }
      });

      // Step 5: 逐条添加原生文字框
      textBlocks.forEach(block => {
        try {
          slide.addText(block.text, {
            x: block.x, y: block.y, w: block.w, h: block.h,
            fontSize: block.fontSize,
            color: block.color,
            bold: block.bold,
            fontFace: 'Microsoft YaHei',
            align: block.align,
            valign: 'middle',
            margin: 0
          });
        } catch(e) { /* skip individual text block errors */ }
      });
    }

    stageEl.style.overflow = origOverflow;
    switchTo(origCurrent);

    await pptx.writeFile({ fileName: '演示文稿.pptx' });
  } catch(e) {
    console.error('PPTX error:', e);
    alert('PPTX生成失败：' + e.message + '\n请检查网络连接后重试。');
    try { switchTo(current || 1); } catch(e2) {}
  }
  btnPptx.textContent = '📥 PPTX';
  btnPptx.disabled = false;
});
```

---

## 四、适配新演示文稿时的修改清单

生成新的演示文稿时，需要根据实际内容调整：

| 项 | 说明 |
|------|------|
| `FINAL_VALUES` | 填入每个场景的计数器最终值 |
| `TEXT_SELECTORS` | 根据实际 HTML 的 class 结构调整选择器 |
| `HIDE_SELECTORS` | 与 TEXT_SELECTORS 保持对应 |
| 图表场景 | `if (i === N) { initXNCharts(); }` 逻辑根据实际场景号调整 |
| 文件名 | `ppt.writeFile({ fileName: '...' })` 和 `pdf.save('...')` |

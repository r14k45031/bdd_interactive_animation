# bdd_interactive_animation[bdd_interactive_animation.html](https://github.com/user-attachments/files/26393851/bdd_interactive_animation.html)

<style>
* { box-sizing: border-box; }
body { margin: 0; }
#bdd-app { font-family: var(--font-sans); color: var(--color-text-primary); padding: 1rem 0; }
.step-header { display: flex; align-items: center; gap: 12px; margin-bottom: 1rem; }
.step-dots { display: flex; gap: 6px; }
.dot { width: 8px; height: 8px; border-radius: 50%; background: var(--color-border-secondary); transition: background .3s; }
.dot.active { background: #534AB7; }
.dot.done { background: #9FE1CB; }
.step-label { font-size: 13px; color: var(--color-text-secondary); }
.step-title { font-size: 16px; font-weight: 500; margin: 0 0 4px; }
.step-desc { font-size: 13px; color: var(--color-text-secondary); margin: 0 0 1rem; line-height: 1.6; min-height: 40px; }
.btn-row { display: flex; gap: 8px; margin-top: 1rem; }
button { cursor: pointer; font-size: 13px; padding: 6px 16px; border-radius: var(--border-radius-md); border: 0.5px solid var(--color-border-secondary); background: transparent; color: var(--color-text-primary); transition: background .15s; }
button:hover { background: var(--color-background-secondary); }
button:disabled { opacity: 0.35; cursor: default; }
button.primary { background: #EEEDFE; color: #3C3489; border-color: #AFA9EC; }
button.primary:hover { background: #CECBF6; }
svg text { font-family: var(--font-sans); }
.svg-wrap { overflow: visible; }
</style>

<div id="bdd-app">
  <div class="step-header">
    <div class="step-dots" id="dots"></div>
    <span class="step-label" id="step-counter"></span>
  </div>
  <div class="step-title" id="title"></div>
  <div class="step-desc" id="desc"></div>
  <div class="svg-wrap">
    <svg id="canvas" width="100%" viewBox="0 0 680 320" style="overflow:visible"></svg>
  </div>
  <div class="btn-row">
    <button id="btn-prev" onclick="go(-1)">← 上一步</button>
    <button id="btn-next" class="primary" onclick="go(1)">下一步 →</button>
  </div>
</div>

<script>
const STEPS = [
  {
    title: "真值表：從函數開始",
    desc: "我們有一個 3 變數函數 f(a,b,c)。最直接的表示是列出所有 8 種輸入組合的結果。問題：n 個變數就需要 2ⁿ 列，指數爆炸。",
    draw: drawTruthTable
  },
  {
    title: "Binary Decision Tree：樹狀展開",
    desc: "把真值表畫成二元決策樹。每個節點問一個變數的值（0 或 1），走到葉節點得到函數結果。大小依然是指數級——葉節點有 2ⁿ 個。",
    draw: drawTree
  },
  {
    title: "觀察：相同的子樹！",
    desc: "仔細看——很多子樹是一模一樣的。例如右邊和左邊的某些 c 節點，輸出完全相同。可以把它們合併！這是 BDD 化簡的核心觀察。",
    draw: drawTreeHighlight
  },
  {
    title: "Rule 1：合併相同節點（Uniqueness）",
    desc: "相同 level + 相同左右子節點 → 合併成同一個節點。用 Hash(level, left, right) 保證唯一性。多個節點「指向」同一個子節點，形成 DAG（有向無環圖）。",
    draw: drawMerge1
  },
  {
    title: "Rule 2：刪除冗餘節點（Non-redundant）",
    desc: "若一個節點的 left child = right child（不管這個變數是 0 還是 1，結果都一樣），這個節點毫無意義，直接刪除，讓父節點跳過它。",
    draw: drawMerge2
  },
  {
    title: "最終 ROBDD：緊湊且唯一",
    desc: "化簡完成！原本 15 個節點的樹，變成只有 4 個節點的 ROBDD。這個結構是 canonical 的——同一個函數永遠對應同一個 ROBDD，可以直接比較 BDD(f) 和 BDD(g) 是否相等。",
    draw: drawROBDD
  }
];

let cur = 0;

function init() {
  const dotsEl = document.getElementById('dots');
  dotsEl.innerHTML = STEPS.map((_, i) => `<div class="dot" id="dot${i}"></div>`).join('');
  render();
}

function render() {
  document.getElementById('title').textContent = STEPS[cur].title;
  document.getElementById('desc').textContent = STEPS[cur].desc;
  document.getElementById('step-counter').textContent = `${cur + 1} / ${STEPS.length}`;
  document.getElementById('btn-prev').disabled = cur === 0;
  document.getElementById('btn-next').disabled = cur === STEPS.length - 1;
  document.getElementById('btn-next').textContent = cur === STEPS.length - 1 ? '完成' : '下一步 →';
  STEPS.forEach((_, i) => {
    const d = document.getElementById('dot' + i);
    d.className = 'dot' + (i === cur ? ' active' : i < cur ? ' done' : '');
  });
  const svg = document.getElementById('canvas');
  svg.innerHTML = '';
  STEPS[cur].draw(svg);
}

function go(dir) {
  cur = Math.max(0, Math.min(STEPS.length - 1, cur + dir));
  render();
}

function el(tag, attrs, parent) {
  const e = document.createElementNS('http://www.w3.org/2000/svg', tag);
  for (const [k, v] of Object.entries(attrs)) e.setAttribute(k, v);
  if (parent) parent.appendChild(e);
  return e;
}

function node(svg, x, y, label, fill, stroke, textColor) {
  el('circle', { cx: x, cy: y, r: 22, fill: fill, stroke: stroke, 'stroke-width': '1.5' }, svg);
  const t = el('text', { x, y: y + 1, 'text-anchor': 'middle', 'dominant-baseline': 'central', 'font-size': '14', 'font-weight': '500', fill: textColor || '#3C3489' }, svg);
  t.textContent = label;
}

function leaf(svg, x, y, val, dim) {
  const fill = dim ? '#D3D1C7' : (val === '1' ? '#9FE1CB' : '#F5C4B3');
  const stroke = dim ? '#B4B2A9' : (val === '1' ? '#0F6E56' : '#993C1D');
  const tc = dim ? '#5F5E5A' : (val === '1' ? '#085041' : '#712B13');
  el('rect', { x: x - 16, y: y - 16, width: 32, height: 32, rx: 6, fill, stroke, 'stroke-width': '1.5' }, svg);
  const t = el('text', { x, y: y + 1, 'text-anchor': 'middle', 'dominant-baseline': 'central', 'font-size': '15', 'font-weight': '600', fill: tc }, svg);
  t.textContent = val;
}

function arrow(svg, x1, y1, x2, y2, color, dashed) {
  const l = el('line', { x1, y1, x2, y2, stroke: color || '#888780', 'stroke-width': '1.5', 'marker-end': 'url(#arr)' }, svg);
  if (dashed) l.setAttribute('stroke-dasharray', '4 3');
}

function addArrowDef(svg) {
  const defs = el('defs', {}, svg);
  const mk = el('marker', { id: 'arr', viewBox: '0 0 10 10', refX: '8', refY: '5', markerWidth: '6', markerHeight: '6', orient: 'auto-start-reverse' }, defs);
  const p = el('path', { d: 'M2 1L8 5L2 9', fill: 'none', stroke: 'context-stroke', 'stroke-width': '1.5', 'stroke-linecap': 'round', 'stroke-linejoin': 'round' }, mk);
}

function edgeLabel(svg, x, y, txt, color) {
  const t = el('text', { x, y, 'text-anchor': 'middle', 'font-size': '11', fill: color || '#888780' }, svg);
  t.textContent = txt;
}

function drawTruthTable(svg) {
  svg.setAttribute('viewBox', '0 0 680 290');
  const rows = [
    ['a','b','c','f'], ['0','0','0','1'], ['0','0','1','0'],
    ['0','1','0','0'], ['0','1','1','1'], ['1','0','0','1'],
    ['1','0','1','0'], ['1','1','0','0'], ['1','1','1','1']
  ];
  const cx = 340, startX = cx - 130, colW = 70, rowH = 28, startY = 20;
  rows.forEach((row, ri) => {
    row.forEach((cell, ci) => {
      const x = startX + ci * colW, y = startY + ri * rowH;
      const isHeader = ri === 0;
      const isFcol = ci === 3;
      let bgFill = 'none';
      if (isHeader) bgFill = '#EEEDFE';
      else if (isFcol) bgFill = cell === '1' ? '#E1F5EE' : '#FAECE7';
      if (bgFill !== 'none') el('rect', { x, y, width: colW - 2, height: rowH - 2, rx: 3, fill: bgFill }, svg);
      const t = el('text', { x: x + colW / 2 - 1, y: y + rowH / 2, 'text-anchor': 'middle', 'dominant-baseline': 'central', 'font-size': '13', 'font-weight': isHeader ? '600' : '400', fill: isHeader ? '#3C3489' : isFcol ? (cell==='1'?'#085041':'#712B13') : '#444441' }, svg);
      t.textContent = cell;
    });
  });
  const lbl = el('text', { x: 340, y: 278, 'text-anchor': 'middle', 'font-size': '12', fill: '#993C1D' }, svg);
  lbl.textContent = '問題：n 個變數 → 2ⁿ = ' + Math.pow(2,3) + ' 列，輸入多了就爆炸';
}

function treeCoords() {
  const aX = 340, aY = 40;
  const bLX = 200, bRX = 480, bY = 110;
  const cPositions = [[110,180],[250,180],[390,180],[530,180]];
  const leafRows = [
    [[60,250],[140,250]],
    [[190,250],[290,250]],
    [[330,250],[430,250]],
    [[470,250],[575,250]]
  ];
  const leafVals = [['1','0'],['0','1'],['1','0'],['0','1']];
  return { aX, aY, bLX, bRX, bY, cPositions, leafRows, leafVals };
}

function drawTree(svg) {
  svg.setAttribute('viewBox', '0 0 680 300');
  addArrowDef(svg);
  const { aX, aY, bLX, bRX, bY, cPositions, leafRows, leafVals } = treeCoords();
  const pur = '#EEEDFE', purS = '#534AB7', purT = '#3C3489';
  const blu = '#E6F1FB', bluS = '#185FA5', bluT = '#0C447C';
  const gry = '#F1EFE8', gryS = '#5F5E5A';

  arrow(svg, aX, aY+22, bLX, bY-22, '#534AB7'); edgeLabel(svg, aX-62, aY+50, '0', '#534AB7');
  arrow(svg, aX, aY+22, bRX, bY-22, '#534AB7'); edgeLabel(svg, aX+60, aY+50, '1', '#534AB7');
  [bLX, bRX].forEach((x,i) => {
    const [c0X,] = cPositions[i*2], [c1X,] = cPositions[i*2+1];
    arrow(svg, x, bY+22, c0X, 180-22, '#185FA5'); edgeLabel(svg, x-32, bY+50, '0', '#185FA5');
    arrow(svg, x, bY+22, c1X, 180-22, '#185FA5'); edgeLabel(svg, x+30, bY+50, '1', '#185FA5');
  });
  cPositions.forEach(([cx, cy], i) => {
    const [l0,l1] = leafRows[i];
    arrow(svg, cx, cy+22, l0[0], 250-16, gryS); edgeLabel(svg, cx-22, cy+50, '0', gryS);
    arrow(svg, cx, cy+22, l1[0], 250-16, gryS); edgeLabel(svg, cx+20, cy+50, '1', gryS);
  });
  node(svg, aX, aY, 'a', pur, purS, purT);
  node(svg, bLX, bY, 'b', blu, bluS, bluT);
  node(svg, bRX, bY, 'b', blu, bluS, bluT);
  cPositions.forEach(([cx,cy]) => node(svg, cx, cy, 'c', gry, gryS, '#444441'));
  leafRows.forEach(([l0,l1], i) => {
    leaf(svg, l0[0], l0[1], leafVals[i][0]);
    leaf(svg, l1[0], l1[1], leafVals[i][1]);
  });
  const t = el('text', { x: 340, y: 292, 'text-anchor': 'middle', 'font-size': '12', fill: '#993C1D' }, svg);
  t.textContent = '15 個節點 — 和真值表一樣是指數大小';
}

function drawTreeHighlight(svg) {
  svg.setAttribute('viewBox', '0 0 680 310');
  addArrowDef(svg);
  const { aX, aY, bLX, bRX, bY, cPositions, leafRows, leafVals } = treeCoords();
  const pur = '#EEEDFE', purS = '#534AB7', purT = '#3C3489';
  const blu = '#E6F1FB', bluS = '#185FA5', bluT = '#0C447C';
  const gry = '#F1EFE8', gryS = '#5F5E5A';

  arrow(svg, aX, aY+22, bLX, bY-22, '#534AB7'); edgeLabel(svg, aX-62, aY+50, '0', '#534AB7');
  arrow(svg, aX, aY+22, bRX, bY-22, '#534AB7'); edgeLabel(svg, aX+60, aY+50, '1', '#534AB7');
  [bLX, bRX].forEach((x,i) => {
    const [c0X,] = cPositions[i*2], [c1X,] = cPositions[i*2+1];
    arrow(svg, x, bY+22, c0X, 180-22, '#185FA5'); edgeLabel(svg, x-32, bY+50, '0', '#185FA5');
    arrow(svg, x, bY+22, c1X, 180-22, '#185FA5'); edgeLabel(svg, x+30, bY+50, '1', '#185FA5');
  });
  cPositions.forEach(([cx, cy], i) => {
    const [l0,l1] = leafRows[i];
    arrow(svg, cx, cy+22, l0[0], 250-16, gryS);
    arrow(svg, cx, cy+22, l1[0], 250-16, gryS);
  });
  node(svg, aX, aY, 'a', pur, purS, purT);
  node(svg, bLX, bY, 'b', blu, bluS, bluT);
  node(svg, bRX, bY, 'b', blu, bluS, bluT);

  const hlColor = '#FAEEDA', hlStroke = '#BA7517';
  cPositions.forEach(([cx,cy], i) => {
    const isHL = i === 0 || i === 2;
    node(svg, cx, cy, 'c', isHL ? hlColor : gry, isHL ? hlStroke : gryS, isHL ? '#633806' : '#444441');
  });

  leafRows.forEach(([l0,l1], i) => {
    const isHL = (i===0||i===2);
    leaf(svg, l0[0], l0[1], leafVals[i][0], !isHL);
    leaf(svg, l1[0], l1[1], leafVals[i][1], !isHL);
  });

  el('rect', { x: 60, y: 157, width: 110, height: 115, rx: 8, fill: 'none', stroke: '#BA7517', 'stroke-width': '2', 'stroke-dasharray': '5 3' }, svg);
  el('rect', { x: 328, y: 157, width: 110, height: 115, rx: 8, fill: 'none', stroke: '#BA7517', 'stroke-width': '2', 'stroke-dasharray': '5 3' }, svg);

  const t = el('text', { x: 340, y: 298, 'text-anchor': 'middle', 'font-size': '12', fill: '#854F0B' }, svg);
  t.textContent = '橘色框的子樹完全相同 → 可以合併！';
  const t2 = el('text', { x: 115, y: 155, 'text-anchor': 'middle', 'font-size': '11', fill: '#854F0B' }, svg);
  t2.textContent = '相同';
  const t3 = el('text', { x: 383, y: 155, 'text-anchor': 'middle', 'font-size': '11', fill: '#854F0B' }, svg);
  t3.textContent = '相同';
}

function drawMerge1(svg) {
  svg.setAttribute('viewBox', '0 0 680 300');
  addArrowDef(svg);
  const pur = '#EEEDFE', purS = '#534AB7', purT = '#3C3489';
  const blu = '#E6F1FB', bluS = '#185FA5', bluT = '#0C447C';
  const amb = '#FAEEDA', ambS = '#BA7517', ambT = '#633806';
  const gry = '#F1EFE8', gryS = '#5F5E5A';

  const aX=340, aY=40, bLX=200, bRX=480, bY=120;
  const c1X=130, c2X=310, c3X=490, cY=200;

  arrow(svg, aX, aY+22, bLX, bY-22, purS);
  arrow(svg, aX, aY+22, bRX, bY-22, purS);
  edgeLabel(svg, aX-62, aY+52, '0', purS);
  edgeLabel(svg, aX+60, aY+52, '1', purS);

  arrow(svg, bLX, bY+22, c1X, cY-22, bluS);
  arrow(svg, bLX, bY+22, c2X, cY-22, bluS);
  edgeLabel(svg, bLX-35, bY+52, '0', bluS);
  edgeLabel(svg, bLX+30, bY+52, '1', bluS);

  arrow(svg, bRX, bY+22, c2X, cY-22, ambS);
  arrow(svg, bRX, bY+22, c3X, cY-22, ambS);
  edgeLabel(svg, bRX-32, bY+52, '0', ambS);
  edgeLabel(svg, bRX+28, bY+52, '1', ambS);

  leaf(svg, c1X-28, 270, '1'); leaf(svg, c1X+28, 270, '0');
  leaf(svg, c2X-28, 270, '0'); leaf(svg, c2X+28, 270, '1');
  leaf(svg, c3X-28, 270, '1'); leaf(svg, c3X+28, 270, '0');

  [[c1X-28,270],[c1X+28,270]].forEach(([x,y],i) => arrow(svg, c1X, cY+22, x, y-16, gryS));
  [[c2X-28,270],[c2X+28,270]].forEach(([x,y],i) => arrow(svg, c2X, cY+22, x, y-16, gryS));
  [[c3X-28,270],[c3X+28,270]].forEach(([x,y],i) => arrow(svg, c3X, cY+22, x, y-16, gryS));

  node(svg, aX, aY, 'a', pur, purS, purT);
  node(svg, bLX, bY, 'b', blu, bluS, bluT);
  node(svg, bRX, bY, 'b', amb, ambS, ambT);

  el('rect', { x: c2X-30, y: cY-30, width: 60, height: 60, rx: 30, fill: '#FAEEDA', stroke: '#BA7517', 'stroke-width': '2.5' }, svg);
  node(svg, c1X, cY, 'c', gry, gryS, '#444441');
  node(svg, c2X, cY, 'c', amb, ambS, ambT);
  node(svg, c3X, cY, 'c', gry, gryS, '#444441');

  const t = el('text', { x: 340, y: 292, 'text-anchor': 'middle', 'font-size': '12', fill: '#854F0B' }, svg);
  t.textContent = '中間的 c 節點被兩條邊共用 — 節點數從 7 減到 5';
}

function drawMerge2(svg) {
  svg.setAttribute('viewBox', '0 0 680 300');
  addArrowDef(svg);
  const pur = '#EEEDFE', purS = '#534AB7', purT = '#3C3489';
  const amb = '#FAEEDA', ambS = '#BA7517', ambT = '#633806';
  const gry = '#F1EFE8', gryS = '#5F5E5A';
  const red = '#FCEBEB', redS = '#A32D2D';

  const aX=340, aY=40, bLX=200, bRX=490, bY=120;
  const cX=330, cY=210;

  arrow(svg, aX, aY+22, bLX, bY-22, purS);
  arrow(svg, aX, aY+22, bRX, bY-22, purS);
  edgeLabel(svg, aX-62, aY+50, '0', purS);
  edgeLabel(svg, aX+60, aY+50, '1', purS);

  el('line', { x1: bLX, y1: bY+22, x2: cX, y2: cY-22, stroke: redS, 'stroke-width': '1.5', 'stroke-dasharray': '4 3', 'marker-end': 'url(#arr)' }, svg);
  el('line', { x1: bLX, y1: bY+22, x2: cX, y2: cY-22, stroke: redS, 'stroke-width': '1.5', 'stroke-dasharray': '4 3', 'marker-end': 'url(#arr)' }, svg);
  edgeLabel(svg, bLX-35, bY+50, '0', redS);
  edgeLabel(svg, bLX+30, bY+50, '1', redS);

  arrow(svg, bRX, bY+22, 420, 270-16, ambS);
  arrow(svg, bRX, bY+22, 530, 270-16, ambS);
  edgeLabel(svg, bRX-32, bY+50, '0', ambS);
  edgeLabel(svg, bRX+28, bY+50, '1', ambS);

  leaf(svg, 290, 270, '1'); leaf(svg, 370, 270, '1');
  leaf(svg, 420, 270, '1'); leaf(svg, 530, 270, '0');

  arrow(svg, cX-10, cY+22, 290, 270-16, gryS);
  arrow(svg, cX+10, cY+22, 370, 270-16, gryS);

  node(svg, aX, aY, 'a', pur, purS, purT);

  el('rect', { x: bLX-28, y: bY-28, width: 56, height: 56, rx: 28, fill: red, stroke: redS, 'stroke-width': '2', 'stroke-dasharray': '5 3' }, svg);
  node(svg, bLX, bY, 'b', red, redS, '#791F1F');
  const cross = el('text', { x: bLX+26, y: bY-24, 'font-size': '18', fill: redS, 'font-weight': '600' }, svg);
  cross.textContent = '✕';

  node(svg, bRX, bY, 'b', amb, ambS, ambT);
  node(svg, cX, cY, 'c', gry, gryS, '#444441');

  const t = el('text', { x: 340, y: 292, 'text-anchor': 'middle', 'font-size': '12', fill: '#A32D2D' }, svg);
  t.textContent = '左邊 b 節點：0/1 兩個孩子都指同一個 c → 刪除此節點！';
}

function drawROBDD(svg) {
  svg.setAttribute('viewBox', '0 0 680 290');
  addArrowDef(svg);
  const pur = '#EEEDFE', purS = '#534AB7', purT = '#3C3489';
  const amb = '#FAEEDA', ambS = '#BA7517', ambT = '#633806';
  const gry = '#F1EFE8', gryS = '#5F5E5A';

  const aX=340, aY=40, bX=490, bY=130, cX=340, cY=210;

  arrow(svg, aX, aY+22, cX, cY-22, purS);
  edgeLabel(svg, aX-28, aY+75, '0', purS);
  const l1 = el('path', { d: `M${aX} ${aY+22} Q${aX+90} ${aY+80} ${bX} ${bY-22}`, fill: 'none', stroke: purS, 'stroke-width': '1.5', 'marker-end': 'url(#arr)' }, svg);
  edgeLabel(svg, aX+80, aY+48, '1', purS);

  el('path', { d: `M${bX} ${bY+22} Q${bX-60} ${bY+80} ${cX+22} ${cY-10}`, fill: 'none', stroke: ambS, 'stroke-width': '1.5', 'marker-end': 'url(#arr)' }, svg);
  edgeLabel(svg, bX-42, bY+62, '0', ambS);

  leaf(svg, 530, 210, '0');
  arrow(svg, bX, bY+22, 530, 210-16, ambS);
  edgeLabel(svg, bX+28, bY+62, '1', ambS);

  leaf(svg, 280, 270, '1');
  leaf(svg, 400, 270, '0');
  arrow(svg, cX, cY+22, 280, 270-16, gryS);
  arrow(svg, cX, cY+22, 400, 270-16, gryS);
  edgeLabel(svg, cX-32, cY+55, '0', gryS);
  edgeLabel(svg, cX+28, cY+55, '1', gryS);

  node(svg, aX, aY, 'a', pur, purS, purT);
  node(svg, bX, bY, 'b', amb, ambS, ambT);
  node(svg, cX, cY, 'c', gry, gryS, '#444441');

  const badge = el('text', { x: 130, y: 60, 'font-size': '13', fill: '#0F6E56' }, svg);
  badge.textContent = '4 個節點';
  el('rect', { x: 70, y: 40, width: 110, height: 30, rx: 6, fill: '#E1F5EE', stroke: '#0F6E56', 'stroke-width': '1' }, svg);
  const badge2 = el('text', { x: 125, y: 60, 'text-anchor': 'middle', 'font-size': '13', fill: '#085041', 'font-weight': '500' }, svg);
  badge2.textContent = '只需 4 個節點';

  const t = el('text', { x: 340, y: 284, 'text-anchor': 'middle', 'font-size': '12', fill: '#0F6E56' }, svg);
  t.textContent = 'Canonical：f = g 若且唯若 BDD(f) 和 BDD(g) 是完全相同的結構';
}

init();
</script>

# KB Diagram Panel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the canvas animation panel in the KB tab with an interactive SVG diagram panel that shows a drawing for each selected formula with editable parameters and step-by-step solutions.

**Architecture:** New standalone `diagrams.js` exports `renderDiagram(container, type, params)` and `getDiagramSteps(type, params)`. The canvas + ~600 lines of animation code are deleted from `app.js`. `selectFormula()` is updated to call `showDiagram(item)` instead of `renderAnimControls()`. Each KB entry gains a `"diagram"` field that controls which SVG renderer fires.

**Tech Stack:** Vanilla JS, inline SVG, CSS custom properties for theming (no new dependencies)

---

## File Map

| File | Action | Responsibility |
|------|--------|---------------|
| `d:/Project/Matmozg/knowledge_base.json` | Modify | Add `"diagram"` field to every entry |
| `d:/Project/Matmozg/diagrams.js` | **Create** | All SVG renderers + param defs + step generators |
| `d:/Project/Matmozg/style.css` | Modify | Remove animation styles; add diagram panel styles + SVG class styles |
| `d:/Project/Matmozg/index.html` | Modify | Replace `.kb-right` contents; add `<script src="diagrams.js">` |
| `d:/Project/Matmozg/app.js` | Modify | Remove animation code; add `showDiagram()` + `onDiagramParam()`; patch `selectFormula()` and `deleteFormula()` |

---

## Task 1: Add "diagram" field to knowledge_base.json

**Files:**
- Modify: `d:/Project/Matmozg/knowledge_base.json`

- [ ] **Step 1: Add `"diagram"` field after `"content"` in every entry**

Open `d:/Project/Matmozg/knowledge_base.json` and add `"diagram": "<value>"` to each object. Use this mapping:

| Title (begins with) | diagram |
|---|---|
| Квадратное уравнение | `"parabola"` |
| Виет теоремасы | `"parabola"` |
| Квадраттың қысқартылған | `"none"` |
| Линейное уравнение | `"line"` |
| Пифагор теоремасы | `"triangle"` |
| Треугольник — площадь | `"triangle"` |
| Шеңбер — формулалар | `"circle"` |
| Квадрат и прямоугольник | `"rectangle"` |
| Трапеция | `"trapezoid"` |
| Объёмы тел | `"volume_cube"` |
| Степени и корни | `"none"` |
| Проценты — формулы | `"none"` |
| Дроби — действия | `"none"` |
| Пропорция | `"none"` |
| Тригонометрия — основные | `"trig_triangle"` |
| Тригонометрия — формулы | `"unit_circle"` |
| Арифметическая прогрессия | `"progression"` |
| Геометрическая прогрессия | `"progression"` |
| Логарифмы | `"none"` |
| Производная | `"none"` |
| Интеграл | `"none"` |
| Комбинаторика | `"none"` |
| Теория вероятностей | `"none"` |
| Статистика | `"none"` |
| Делимость | `"none"` |
| НОД и НОК | `"none"` |
| Неравенства | `"none"` |
| Уравнение прямой | `"line"` |
| Парабола y = | `"parabola"` |
| Координатная плоскость | `"line"` |
| Векторы | `"none"` |
| Пифагор үшліктері | `"triangle"` |
| Тригонометрик теңлемелер | `"unit_circle"` |
| Матрицалар | `"none"` |
| Комплекс сандар | `"none"` |

Example of a modified entry:
```json
{
  "title": "Пифагор теоремасы",
  "content": "...",
  "diagram": "triangle"
}
```

- [ ] **Step 2: Verify JSON is valid**

```bash
python -c "import json; data=json.load(open('d:/Project/Matmozg/knowledge_base.json')); print(len(data), 'entries,', sum(1 for e in data if 'diagram' in e), 'with diagram field')"
```
Expected: `35 entries, 35 with diagram field`

---

## Task 2: Create diagrams.js

**Files:**
- Create: `d:/Project/Matmozg/diagrams.js`

- [ ] **Step 1: Create the file with DIAGRAM_PARAMS, SVG helpers, and all renderers**

Create `d:/Project/Matmozg/diagrams.js` with the following complete content:

```js
/* ============================================================
   MatematikaAI — diagrams.js
   SVG diagram renderers for the Knowledge Base panel.
   Public API:
     renderDiagram(container, type, params) → sets container.innerHTML
     getDiagramSteps(type, params)          → returns [{label, value}]
     DIAGRAM_PARAMS                         → param definitions per type
============================================================ */
'use strict';

// ── Parameter definitions per diagram type ───────────────────
const DIAGRAM_PARAMS = {
  triangle:      [{ id:'a', label:'a (катет)',    value:3,   min:0.5, max:20,  step:0.5 },
                  { id:'b', label:'b (катет)',    value:4,   min:0.5, max:20,  step:0.5 }],
  circle:        [{ id:'r', label:'r (радиус)',  value:5,   min:0.5, max:20,  step:0.5 }],
  rectangle:     [{ id:'a', label:'a (ширина)',  value:6,   min:0.5, max:20,  step:0.5 },
                  { id:'b', label:'b (высота)',  value:4,   min:0.5, max:20,  step:0.5 }],
  trapezoid:     [{ id:'a', label:'a (верх)',    value:6,   min:0.5, max:20,  step:0.5 },
                  { id:'b', label:'b (низ)',     value:10,  min:0.5, max:20,  step:0.5 },
                  { id:'h', label:'h (высота)',  value:4,   min:0.5, max:15,  step:0.5 }],
  parabola:      [{ id:'a', label:'a',           value:1,   min:-5,  max:5,   step:0.5 },
                  { id:'b', label:'b',           value:-4,  min:-10, max:10,  step:1   },
                  { id:'c', label:'c',           value:3,   min:-20, max:20,  step:1   }],
  line:          [{ id:'k', label:'k (наклон)',  value:2,   min:-5,  max:5,   step:0.5 },
                  { id:'b', label:'b (сдвиг)',   value:1,   min:-10, max:10,  step:1   }],
  trig_triangle: [{ id:'angle', label:'α (°)',   value:30,  min:1,   max:89,  step:1   }],
  unit_circle:   [{ id:'angle', label:'α (°)',   value:45,  min:0,   max:360, step:5   }],
  speed:         [{ id:'v', label:'v (км/ч)',    value:60,  min:1,   max:300, step:10  },
                  { id:'t', label:'t (ч)',       value:2,   min:0.5, max:10,  step:0.5 }],
  volume_cube:   [{ id:'a', label:'a (ребро)',   value:4,   min:1,   max:15,  step:1   }],
  progression:   [{ id:'a1', label:'a₁',        value:2,   min:-20, max:20,  step:1   },
                  { id:'d',  label:'d (шаг)',    value:3,   min:-10, max:10,  step:1   },
                  { id:'n',  label:'n (членов)', value:5,   min:2,   max:8,   step:1   }],
  none:          [],
};

// ── SVG renderers ─────────────────────────────────────────────

function svgTriangle(p) {
  const W = 300, H = 260;
  const a = Math.max(0.5, p.a ?? 3);
  const b = Math.max(0.5, p.b ?? 4);
  const c = Math.sqrt(a * a + b * b);
  const sc = Math.min(190 / b, 160 / a, 22);
  const ox = 55, oy = 220;
  const ax = ox, ay = oy - a * sc;
  const bx = ox + b * sc, by = oy;
  const mx = (ax + bx) / 2, my = (ay + by) / 2;
  const ang = Math.atan2(by - ay, bx - ax);
  const lx = (mx - Math.sin(ang) * 16).toFixed(1);
  const ly = (my + Math.cos(ang) * 16).toFixed(1);
  const m = Math.min(13, sc * 0.9);
  return `<svg viewBox="0 0 ${W} ${H}" xmlns="http://www.w3.org/2000/svg" class="diag-svg">
  <polygon points="${ax},${ay} ${ox},${oy} ${bx},${by}" class="shape-fill"/>
  <polyline points="${ax},${ay} ${ox},${oy} ${bx},${by} ${ax},${ay}" class="shape-stroke"/>
  <path d="M${ox},${oy - m} L${ox + m},${oy - m} L${ox + m},${oy}" class="angle-mark"/>
  <text x="${ox - 24}" y="${((ay + oy) / 2).toFixed(1)}" class="lbl-accent" dominant-baseline="middle" text-anchor="middle">a=${a}</text>
  <text x="${((ox + bx) / 2).toFixed(1)}" y="${oy + 20}" class="lbl-accent" text-anchor="middle">b=${b}</text>
  <text x="${lx}" y="${ly}" class="lbl-green" text-anchor="middle">c</text>
  <text x="${W / 2}" y="20" class="lbl-formula" text-anchor="middle">c = √(a²+b²) = ${c.toFixed(3)}</text>
</svg>`;
}

function svgCircle(p) {
  const W = 300, H = 260;
  const r = Math.max(0.5, p.r ?? 5);
  const r_px = Math.min(100, r * 8);
  const cx = W / 2, cy = H / 2 + 12;
  const C = (2 * Math.PI * r).toFixed(2);
  const S = (Math.PI * r * r).toFixed(2);
  return `<svg viewBox="0 0 ${W} ${H}" xmlns="http://www.w3.org/2000/svg" class="diag-svg">
  <circle cx="${cx}" cy="${cy}" r="${r_px}" class="shape-fill"/>
  <circle cx="${cx}" cy="${cy}" r="${r_px}" class="shape-stroke-only"/>
  <line x1="${cx - r_px}" y1="${cy}" x2="${cx + r_px}" y2="${cy}" class="dashed-line"/>
  <line x1="${cx}" y1="${cy}" x2="${cx + r_px}" y2="${cy}" class="accent-line"/>
  <text x="${(cx + r_px / 2).toFixed(1)}" y="${cy - 10}" class="lbl-accent" text-anchor="middle">r=${r}</text>
  <text x="${cx}" y="${cy + 7}" class="lbl-dim" text-anchor="middle">d=${2 * r}</text>
  <text x="${W / 2}" y="20" class="lbl-formula" text-anchor="middle">C = 2πr = ${C}</text>
  <text x="${W / 2}" y="38" class="lbl-formula" text-anchor="middle">S = πr² = ${S}</text>
</svg>`;
}

function svgRectangle(p) {
  const W = 300, H = 260;
  const a = Math.max(0.5, p.a ?? 6);
  const b = Math.max(0.5, p.b ?? 4);
  const sc = Math.min(210 / a, 160 / b, 20);
  const pw = a * sc, ph = b * sc;
  const rx = (W - pw) / 2, ry = (H - ph) / 2 + 15;
  const d = Math.sqrt(a * a + b * b).toFixed(2);
  return `<svg viewBox="0 0 ${W} ${H}" xmlns="http://www.w3.org/2000/svg" class="diag-svg">
  <rect x="${rx.toFixed(1)}" y="${ry.toFixed(1)}" width="${pw.toFixed(1)}" height="${ph.toFixed(1)}" class="shape-fill"/>
  <rect x="${rx.toFixed(1)}" y="${ry.toFixed(1)}" width="${pw.toFixed(1)}" height="${ph.toFixed(1)}" class="shape-stroke-only"/>
  <line x1="${rx.toFixed(1)}" y1="${ry.toFixed(1)}" x2="${(rx + pw).toFixed(1)}" y2="${(ry + ph).toFixed(1)}" class="dashed-line"/>
  <text x="${(rx + pw / 2).toFixed(1)}" y="${(ry - 10).toFixed(1)}" class="lbl-accent" text-anchor="middle">a=${a}</text>
  <text x="${(rx - 14).toFixed(1)}" y="${(ry + ph / 2).toFixed(1)}" class="lbl-accent" text-anchor="middle" dominant-baseline="middle">b=${b}</text>
  <text x="${W / 2}" y="20" class="lbl-formula" text-anchor="middle">S=${(a * b).toFixed(2)}  P=${2 * (a + b)}  d=${d}</text>
</svg>`;
}

function svgTrapezoid(p) {
  const W = 300, H = 260;
  const a = Math.max(0.5, p.a ?? 6);
  const b = Math.max(0.5, p.b ?? 10);
  const h = Math.max(0.5, p.h ?? 4);
  const sc = Math.min(220 / Math.max(a, b), 160 / h, 20);
  const bpx = b * sc, apx = a * sc, hpx = h * sc;
  const bx = (W - bpx) / 2;
  const byC = H / 2 + hpx / 2 + 20;
  const ax = (W - apx) / 2;
  const ayC = byC - hpx;
  const S = ((a + b) * h / 2).toFixed(2);
  return `<svg viewBox="0 0 ${W} ${H}" xmlns="http://www.w3.org/2000/svg" class="diag-svg">
  <polygon points="${ax.toFixed(1)},${ayC.toFixed(1)} ${(ax+apx).toFixed(1)},${ayC.toFixed(1)} ${(bx+bpx).toFixed(1)},${byC.toFixed(1)} ${bx.toFixed(1)},${byC.toFixed(1)}" class="shape-fill"/>
  <polygon points="${ax.toFixed(1)},${ayC.toFixed(1)} ${(ax+apx).toFixed(1)},${ayC.toFixed(1)} ${(bx+bpx).toFixed(1)},${byC.toFixed(1)} ${bx.toFixed(1)},${byC.toFixed(1)}" class="shape-stroke-only"/>
  <line x1="${W/2}" y1="${ayC.toFixed(1)}" x2="${W/2}" y2="${byC.toFixed(1)}" class="dashed-line"/>
  <text x="${W / 2}" y="${(ayC - 10).toFixed(1)}" class="lbl-accent" text-anchor="middle">a=${a}</text>
  <text x="${W / 2}" y="${(byC + 18).toFixed(1)}" class="lbl-accent" text-anchor="middle">b=${b}</text>
  <text x="${(W / 2 + 14).toFixed(1)}" y="${((ayC + byC) / 2).toFixed(1)}" class="lbl-green" dominant-baseline="middle">h=${h}</text>
  <text x="${W / 2}" y="20" class="lbl-formula" text-anchor="middle">S = (a+b)×h/2 = ${S}</text>
</svg>`;
}

function svgParabola(p) {
  const W = 300, H = 260;
  const a = p.a ?? 1;
  const b = p.b ?? -4;
  const c = p.c ?? 3;
  const ox = W / 2, oy = H * 0.72;
  const xs = 32, ys = 26;
  const tx = x => ox + x * xs;
  const ty = y => oy - y * ys;
  const pts = [];
  for (let x = -4.5; x <= 4.5; x += 0.12) {
    const y = a * x * x + b * x + c;
    const px = tx(x), py = ty(y);
    if (py > 5 && py < H - 5) pts.push(`${px.toFixed(1)},${py.toFixed(1)}`);
  }
  const D = b * b - 4 * a * c;
  let rootsText = '', rootDots = '';
  if (a !== 0) {
    if (D > 0.0001) {
      const x1 = (-b - Math.sqrt(D)) / (2 * a);
      const x2 = (-b + Math.sqrt(D)) / (2 * a);
      rootsText = `D=${D.toFixed(1)}: x₁=${x1.toFixed(2)}, x₂=${x2.toFixed(2)}`;
      rootDots = `<circle cx="${tx(x1).toFixed(1)}" cy="${ty(0).toFixed(1)}" r="4" class="root-dot"/><circle cx="${tx(x2).toFixed(1)}" cy="${ty(0).toFixed(1)}" r="4" class="root-dot"/>`;
    } else if (Math.abs(D) < 0.0001) {
      const x1 = -b / (2 * a);
      rootsText = `D=0: x=${x1.toFixed(2)}`;
      rootDots = `<circle cx="${tx(x1).toFixed(1)}" cy="${ty(0).toFixed(1)}" r="4" class="root-dot"/>`;
    } else {
      rootsText = `D=${D.toFixed(1)}: корней нет`;
    }
  }
  const xv = a !== 0 ? -b / (2 * a) : 0;
  const yv = a !== 0 ? c - b * b / (4 * a) : c;
  const bSign = b >= 0 ? '+' : '';
  const cSign = c >= 0 ? '+' : '';
  return `<svg viewBox="0 0 ${W} ${H}" xmlns="http://www.w3.org/2000/svg" class="diag-svg">
  <line x1="${tx(-4.5).toFixed(1)}" y1="${oy.toFixed(1)}" x2="${tx(4.5).toFixed(1)}" y2="${oy.toFixed(1)}" class="axis-line"/>
  <line x1="${ox.toFixed(1)}" y1="8" x2="${ox.toFixed(1)}" y2="${(H-8).toFixed(1)}" class="axis-line"/>
  <text x="${tx(4.5).toFixed(1)}" y="${oy.toFixed(1)}" class="lbl-dim" dominant-baseline="middle" dx="4">x</text>
  <text x="${ox.toFixed(1)}" y="14" class="lbl-dim" text-anchor="middle">y</text>
  <polyline points="${pts.join(' ')}" class="curve-line"/>
  <circle cx="${tx(xv).toFixed(1)}" cy="${ty(yv).toFixed(1)}" r="4" class="vertex-dot"/>
  ${rootDots}
  <text x="${(tx(xv) + 8).toFixed(1)}" y="${(ty(yv) - 6).toFixed(1)}" class="lbl-accent" font-size="10" dominant-baseline="auto">V(${xv.toFixed(1)};${yv.toFixed(1)})</text>
  <text x="${W / 2}" y="18" class="lbl-formula" text-anchor="middle">y = ${a}x² ${bSign}${b}x ${cSign}${c}</text>
  <text x="${W / 2}" y="34" class="lbl-formula" text-anchor="middle">${rootsText}</text>
</svg>`;
}

function svgLineDiagram(p) {
  const W = 300, H = 260;
  const k = p.k ?? 2;
  const b = p.b ?? 1;
  const ox = W / 2, oy = H / 2;
  const xs = 28, ys = 28;
  const tx = x => ox + x * xs;
  const ty = y => oy - y * ys;
  const y1 = k * (-4) + b, y2 = k * 4 + b;
  const yIntY = ty(b);
  let xIntDot = '';
  if (k !== 0) {
    const xi = -b / k;
    if (Math.abs(xi) <= 4.5) {
      xIntDot = `<circle cx="${tx(xi).toFixed(1)}" cy="${ty(0).toFixed(1)}" r="4" class="root-dot"/>
  <text x="${tx(xi).toFixed(1)}" y="${(ty(0) + 16).toFixed(1)}" class="lbl-dim" text-anchor="middle" font-size="10">${xi.toFixed(2)}</text>`;
    }
  }
  const bSign = b >= 0 ? '+' : '';
  return `<svg viewBox="0 0 ${W} ${H}" xmlns="http://www.w3.org/2000/svg" class="diag-svg">
  <line x1="${tx(-4.5).toFixed(1)}" y1="${oy.toFixed(1)}" x2="${tx(4.5).toFixed(1)}" y2="${oy.toFixed(1)}" class="axis-line"/>
  <line x1="${ox.toFixed(1)}" y1="8" x2="${ox.toFixed(1)}" y2="${(H-8).toFixed(1)}" class="axis-line"/>
  <text x="${tx(4.5).toFixed(1)}" y="${oy.toFixed(1)}" class="lbl-dim" dominant-baseline="middle" dx="4">x</text>
  <text x="${ox.toFixed(1)}" y="14" class="lbl-dim" text-anchor="middle">y</text>
  <line x1="${tx(-4).toFixed(1)}" y1="${ty(y1).toFixed(1)}" x2="${tx(4).toFixed(1)}" y2="${ty(y2).toFixed(1)}" class="accent-line"/>
  <circle cx="${tx(0).toFixed(1)}" cy="${yIntY.toFixed(1)}" r="4" class="vertex-dot"/>
  <text x="${(tx(0) + 8).toFixed(1)}" y="${(yIntY - 6).toFixed(1)}" class="lbl-accent" font-size="11">b=${b}</text>
  ${xIntDot}
  <text x="${W / 2}" y="18" class="lbl-formula" text-anchor="middle">y = ${k}x ${bSign}${b}</text>
  <text x="${W / 2}" y="34" class="lbl-formula" text-anchor="middle">k = ${k}  (наклон)</text>
</svg>`;
}

function svgTrigTriangle(p) {
  const W = 300, H = 260;
  const deg = Math.min(89, Math.max(1, p.angle ?? 30));
  const rad = deg * Math.PI / 180;
  const hyp = 140;
  const opp = Math.sin(rad) * hyp;
  const adj = Math.cos(rad) * hyp;
  const bleft  = { x: 65,         y: 215 };
  const bright = { x: 65 + adj,   y: 215 };
  const top    = { x: 65 + adj,   y: 215 - opp };
  const m = 10;
  const sinV = Math.sin(rad).toFixed(3);
  const cosV = Math.cos(rad).toFixed(3);
  const tanV = Math.tan(rad).toFixed(3);
  const arcR = 26;
  const arcEx = bleft.x + arcR;
  const arcEy = bleft.y;
  const arcEx2 = bleft.x + arcR * Math.cos(rad);
  const arcEy2 = bleft.y - arcR * Math.sin(rad);
  const pt = o => `${o.x.toFixed(1)},${o.y.toFixed(1)}`;
  return `<svg viewBox="0 0 ${W} ${H}" xmlns="http://www.w3.org/2000/svg" class="diag-svg">
  <polygon points="${pt(bleft)} ${pt(bright)} ${pt(top)}" class="shape-fill"/>
  <polyline points="${pt(bleft)} ${pt(bright)} ${pt(top)} ${pt(bleft)}" class="shape-stroke"/>
  <path d="M${bright.x - m},${bright.y} L${bright.x - m},${bright.y - m} L${bright.x},${bright.y - m}" class="angle-mark"/>
  <path d="M${arcEx.toFixed(1)},${arcEy.toFixed(1)} A${arcR},${arcR} 0 0,0 ${arcEx2.toFixed(1)},${arcEy2.toFixed(1)}" class="arc-line"/>
  <text x="${bleft.x + 36}" y="${bleft.y - 14}" class="lbl-accent" font-size="13">${deg}°</text>
  <text x="${(bright.x + 12).toFixed(1)}" y="${((bright.y + top.y) / 2).toFixed(1)}" class="lbl-green" dominant-baseline="middle" font-size="11">sin=${sinV}</text>
  <text x="${((bleft.x + bright.x) / 2).toFixed(1)}" y="${(bleft.y + 18).toFixed(1)}" class="lbl-accent" text-anchor="middle" font-size="11">cos=${cosV}</text>
  <text x="${W / 2}" y="20" class="lbl-formula" text-anchor="middle">sin=${sinV}, cos=${cosV}, tg=${tanV}</text>
</svg>`;
}

function svgUnitCircle(p) {
  const W = 300, H = 260;
  const deg = p.angle ?? 45;
  const rad = deg * Math.PI / 180;
  const cx = W / 2, cy = H / 2 + 12;
  const R = 88;
  const px = cx + R * Math.cos(rad);
  const py = cy - R * Math.sin(rad);
  const sinV = Math.sin(rad).toFixed(3);
  const cosV = Math.cos(rad).toFixed(3);
  const cosSign = Math.cos(rad) >= 0 ? 1 : -1;
  const sinSign = Math.sin(rad) >= 0 ? -1 : 1;
  return `<svg viewBox="0 0 ${W} ${H}" xmlns="http://www.w3.org/2000/svg" class="diag-svg">
  <circle cx="${cx}" cy="${cy}" r="${R}" class="circle-outline"/>
  <line x1="${cx - R - 14}" y1="${cy}" x2="${cx + R + 14}" y2="${cy}" class="axis-line"/>
  <line x1="${cx}" y1="${cy + R + 14}" x2="${cx}" y2="${cy - R - 14}" class="axis-line"/>
  <text x="${cx + R + 18}" y="${cy}" class="lbl-dim" dominant-baseline="middle">x</text>
  <text x="${cx}" y="${cy - R - 18}" class="lbl-dim" text-anchor="middle">y</text>
  <line x1="${cx}" y1="${cy}" x2="${px.toFixed(1)}" y2="${py.toFixed(1)}" class="accent-line"/>
  <circle cx="${px.toFixed(1)}" cy="${py.toFixed(1)}" r="5" class="vertex-dot"/>
  <line x1="${px.toFixed(1)}" y1="${py.toFixed(1)}" x2="${px.toFixed(1)}" y2="${cy}" class="sin-line"/>
  <line x1="${cx}" y1="${cy}" x2="${px.toFixed(1)}" y2="${cy}" class="cos-line"/>
  <text x="${(px + cosSign * 8).toFixed(1)}" y="${((py + cy) / 2).toFixed(1)}" class="lbl-green" dominant-baseline="middle" ${cosSign > 0 ? 'text-anchor="start"' : 'text-anchor="end"'} font-size="11">sin=${sinV}</text>
  <text x="${((cx + px) / 2).toFixed(1)}" y="${(cy + sinSign * 14).toFixed(1)}" class="lbl-accent" text-anchor="middle" font-size="11">cos=${cosV}</text>
  <text x="${cx + 28}" y="${cy - 20}" class="lbl-dim" font-size="10">${deg}°</text>
  <text x="${W / 2}" y="18" class="lbl-formula" text-anchor="middle">sin ${deg}° = ${sinV},  cos ${deg}° = ${cosV}</text>
</svg>`;
}

function svgSpeed(p) {
  const W = 300, H = 260;
  const v = Math.max(1, p.v ?? 60);
  const t = Math.max(0.5, p.t ?? 2);
  const s = v * t;
  const cy = H / 2 + 20;
  const x1 = 40, x2 = W - 50;
  return `<svg viewBox="0 0 ${W} ${H}" xmlns="http://www.w3.org/2000/svg" class="diag-svg">
  <defs><marker id="darr" markerWidth="10" markerHeight="7" refX="9" refY="3.5" orient="auto">
    <polygon points="0 0, 10 3.5, 0 7" class="arrow-head"/>
  </marker></defs>
  <line x1="${x1}" y1="${cy}" x2="${x2}" y2="${cy}" class="accent-line" marker-end="url(#darr)"/>
  <circle cx="${x1}" cy="${cy}" r="5" class="vertex-dot"/>
  <text x="${x1}" y="${cy + 20}" class="lbl-dim" text-anchor="middle">0</text>
  <text x="${x2}" y="${cy + 20}" class="lbl-accent" text-anchor="middle">s = ${s} км</text>
  <text x="${(x1 + x2) / 2}" y="${cy - 16}" class="lbl-green" text-anchor="middle">v = ${v} км/ч</text>
  <text x="${(x1 + x2) / 2}" y="${cy + 36}" class="lbl-accent" text-anchor="middle">t = ${t} ч</text>
  <text x="${W / 2}" y="22" class="lbl-formula" text-anchor="middle">s = v × t = ${v} × ${t} = ${s}</text>
</svg>`;
}

function svgVolumeCube(p) {
  const W = 300, H = 260;
  const a = Math.max(1, p.a ?? 4);
  const e = Math.min(88, a * 18);
  const dx = e * Math.cos(Math.PI / 6);
  const dy = e * Math.sin(Math.PI / 6);
  const cx2 = W / 2, cy2 = H / 2 + 26;
  const TF = { x: cx2,        y: cy2 - e + 2 * dy };
  const TR = { x: cx2 + dx,   y: cy2 - e + dy     };
  const TL = { x: cx2 - dx,   y: cy2 - e + dy     };
  const T  = { x: cx2,        y: cy2 - e           };
  const BF = { x: cx2,        y: cy2 + 2 * dy      };
  const BR = { x: cx2 + dx,   y: cy2 + dy          };
  const BL = { x: cx2 - dx,   y: cy2 + dy          };
  const pt = o => `${o.x.toFixed(1)},${o.y.toFixed(1)}`;
  const V = (a * a * a).toFixed(0);
  const S = (6 * a * a).toFixed(0);
  return `<svg viewBox="0 0 ${W} ${H}" xmlns="http://www.w3.org/2000/svg" class="diag-svg">
  <polygon points="${pt(TF)} ${pt(TR)} ${pt(T)} ${pt(TL)}" class="face-top"/>
  <polygon points="${pt(TF)} ${pt(TR)} ${pt(BR)} ${pt(BF)}" class="face-right"/>
  <polygon points="${pt(TF)} ${pt(TL)} ${pt(BL)} ${pt(BF)}" class="face-left"/>
  <polyline points="${pt(TF)} ${pt(TR)} ${pt(T)} ${pt(TL)} ${pt(TF)}" class="shape-stroke-only"/>
  <line x1="${TR.x.toFixed(1)}" y1="${TR.y.toFixed(1)}" x2="${BR.x.toFixed(1)}" y2="${BR.y.toFixed(1)}" class="shape-stroke-line"/>
  <line x1="${TF.x.toFixed(1)}" y1="${TF.y.toFixed(1)}" x2="${BF.x.toFixed(1)}" y2="${BF.y.toFixed(1)}" class="shape-stroke-line"/>
  <line x1="${TL.x.toFixed(1)}" y1="${TL.y.toFixed(1)}" x2="${BL.x.toFixed(1)}" y2="${BL.y.toFixed(1)}" class="shape-stroke-line"/>
  <line x1="${BF.x.toFixed(1)}" y1="${BF.y.toFixed(1)}" x2="${BR.x.toFixed(1)}" y2="${BR.y.toFixed(1)}" class="shape-stroke-line"/>
  <line x1="${BF.x.toFixed(1)}" y1="${BF.y.toFixed(1)}" x2="${BL.x.toFixed(1)}" y2="${BL.y.toFixed(1)}" class="shape-stroke-line"/>
  <text x="${(cx2 + dx / 2 + 8).toFixed(1)}" y="${((TF.y + TR.y) / 2 + 6).toFixed(1)}" class="lbl-accent" font-size="12">a=${a}</text>
  <text x="${W / 2}" y="20" class="lbl-formula" text-anchor="middle">V = a³ = ${V}</text>
  <text x="${W / 2}" y="36" class="lbl-formula" text-anchor="middle">S = 6a² = ${S}</text>
</svg>`;
}

function svgProgression(p) {
  const W = 300, H = 260;
  const a1 = p.a1 ?? 2;
  const d  = p.d  ?? 3;
  const n  = Math.min(8, Math.max(2, Math.round(p.n ?? 5)));
  const cy = H / 2 + 10;
  const x1 = 35, x2 = W - 35;
  const sp = (x2 - x1) / (n - 1);
  const members = Array.from({ length: n }, (_, i) => a1 + d * i);
  const last = members[n - 1];
  const Sn = (n * (a1 + last) / 2).toFixed(1);
  let dots = '', labels = '';
  members.forEach((val, i) => {
    const x = (x1 + i * sp).toFixed(1);
    dots   += `<circle cx="${x}" cy="${cy}" r="5" class="vertex-dot"/>`;
    labels += `<text x="${x}" y="${cy - 15}" class="lbl-accent" text-anchor="middle" font-size="11">${val}</text>`;
    if (i === 0) labels += `<text x="${x}" y="${cy + 20}" class="lbl-dim" text-anchor="middle" font-size="10">a₁</text>`;
    if (i === n - 1) labels += `<text x="${x}" y="${cy + 20}" class="lbl-dim" text-anchor="middle" font-size="10">a${n}</text>`;
  });
  const bY = cy + 32;
  const stepX2 = (x1 + sp).toFixed(1);
  return `<svg viewBox="0 0 ${W} ${H}" xmlns="http://www.w3.org/2000/svg" class="diag-svg">
  <line x1="${x1}" y1="${cy}" x2="${x2}" y2="${cy}" class="dashed-line"/>
  ${dots}
  ${labels}
  <path d="M${x1},${bY} L${x1},${bY + 8} L${stepX2},${bY + 8} L${stepX2},${bY}" class="angle-mark"/>
  <text x="${((x1 + parseFloat(stepX2)) / 2).toFixed(1)}" y="${bY + 22}" class="lbl-green" text-anchor="middle" font-size="11">d=${d}</text>
  <text x="${W / 2}" y="20" class="lbl-formula" text-anchor="middle">a₁=${a1}, d=${d}, n=${n}</text>
  <text x="${W / 2}" y="36" class="lbl-formula" text-anchor="middle">S${n} = ${Sn}</text>
</svg>`;
}

// ── Step generators ───────────────────────────────────────────

function stepsTriangle(p) {
  const a = p.a ?? 3, b = p.b ?? 4;
  const c = Math.sqrt(a * a + b * b);
  return [
    { label: 'Формула',      value: 'c = √(a² + b²)' },
    { label: 'Подстановка',  value: `c = √(${a}² + ${b}²)` },
    { label: 'Вычисление',   value: `c = √(${a*a} + ${b*b}) = √${a*a+b*b}` },
    { label: 'Ответ',        value: `c ≈ ${c.toFixed(4)}` },
  ];
}

function stepsCircle(p) {
  const r = p.r ?? 5;
  return [
    { label: 'Формула C',    value: 'C = 2 × π × r' },
    { label: 'Подстановка',  value: `C = 2 × 3.14159 × ${r}` },
    { label: 'Ответ C',      value: `C = ${(2*Math.PI*r).toFixed(4)}` },
    { label: 'Формула S',    value: 'S = π × r²' },
    { label: 'Ответ S',      value: `S = ${(Math.PI*r*r).toFixed(4)}` },
  ];
}

function stepsRectangle(p) {
  const a = p.a ?? 6, b = p.b ?? 4;
  return [
    { label: 'Площадь',      value: `S = a × b = ${a} × ${b} = ${a*b}` },
    { label: 'Периметр',     value: `P = 2(a+b) = 2(${a}+${b}) = ${2*(a+b)}` },
    { label: 'Диагональ',    value: `d = √(a²+b²) = √${a*a+b*b} ≈ ${Math.sqrt(a*a+b*b).toFixed(2)}` },
  ];
}

function stepsTrapezoid(p) {
  const a = p.a ?? 6, b = p.b ?? 10, h = p.h ?? 4;
  return [
    { label: 'Формула',      value: 'S = (a + b) × h / 2' },
    { label: 'Подстановка',  value: `S = (${a} + ${b}) × ${h} / 2` },
    { label: 'Ответ',        value: `S = ${(a+b)*h/2}` },
  ];
}

function stepsParabola(p) {
  const a = p.a ?? 1, b = p.b ?? -4, c = p.c ?? 3;
  const xv = a !== 0 ? (-b / (2 * a)).toFixed(2) : '—';
  const yv = a !== 0 ? (c - b*b/(4*a)).toFixed(2) : c;
  const D  = b*b - 4*a*c;
  const rows = [
    { label: 'Уравнение',  value: `y = ${a}x² ${b>=0?'+':''}${b}x ${c>=0?'+':''}${c}` },
    { label: 'Вершина xᵥ', value: `xᵥ = −b/(2a) = −(${b})/(2×${a}) = ${xv}` },
    { label: 'Вершина yᵥ', value: `yᵥ = ${yv}` },
    { label: 'D',          value: `D = b²−4ac = ${b}²−4×${a}×${c} = ${D.toFixed(2)}` },
  ];
  if (D > 0) {
    const x1 = ((-b - Math.sqrt(D)) / (2*a)).toFixed(2);
    const x2 = ((-b + Math.sqrt(D)) / (2*a)).toFixed(2);
    rows.push({ label: 'Корни', value: `x₁=${x1},  x₂=${x2}` });
  } else if (Math.abs(D) < 0.001) {
    rows.push({ label: 'Корень', value: `x = ${xv}` });
  } else {
    rows.push({ label: 'Корни', value: 'Нет (D < 0)' });
  }
  return rows;
}

function stepsLine(p) {
  const k = p.k ?? 2, b = p.b ?? 1;
  return [
    { label: 'Уравнение',   value: `y = ${k}x ${b>=0?'+':''}${b}` },
    { label: 'При x=0',     value: `y = ${b}  (точка (0; ${b}))` },
    { label: 'При x=1',     value: `y = ${k}×1${b>=0?'+':''}${b} = ${k+b}` },
    { label: 'При y=0',     value: k !== 0 ? `x = −b/k = ${(-b/k).toFixed(2)}` : 'нет (k=0)' },
  ];
}

function stepsTrigTriangle(p) {
  const deg = p.angle ?? 30;
  const rad = deg * Math.PI / 180;
  return [
    { label: 'Угол α',   value: `${deg}°` },
    { label: 'sin α',    value: `sin ${deg}° = ${Math.sin(rad).toFixed(4)}` },
    { label: 'cos α',    value: `cos ${deg}° = ${Math.cos(rad).toFixed(4)}` },
    { label: 'tg α',     value: `tg ${deg}° = ${Math.tan(rad).toFixed(4)}` },
    { label: 'Тождество', value: `sin²+cos² = ${(Math.sin(rad)**2+Math.cos(rad)**2).toFixed(4)}` },
  ];
}

function stepsUnitCircle(p) {
  const deg = p.angle ?? 45;
  const rad = deg * Math.PI / 180;
  return [
    { label: 'Угол α',   value: `${deg}° = ${rad.toFixed(4)} рад` },
    { label: 'sin',      value: Math.sin(rad).toFixed(4) },
    { label: 'cos',      value: Math.cos(rad).toFixed(4) },
    { label: 'tg',       value: Math.abs(Math.cos(rad)) > 0.0001 ? Math.tan(rad).toFixed(4) : '∞' },
  ];
}

function stepsSpeed(p) {
  const v = p.v ?? 60, t = p.t ?? 2;
  const s = v * t;
  return [
    { label: 'Дано',    value: `v=${v} км/ч, t=${t} ч` },
    { label: 'Путь',    value: `s = v×t = ${v}×${t} = ${s} км` },
    { label: 'Скорость',value: `v = s/t = ${s}/${t} = ${v} км/ч` },
    { label: 'Время',   value: `t = s/v = ${s}/${v} = ${t} ч` },
  ];
}

function stepsVolumeCube(p) {
  const a = p.a ?? 4;
  return [
    { label: 'Ребро',    value: `a = ${a}` },
    { label: 'Объём',    value: `V = a³ = ${a}³ = ${a**3}` },
    { label: 'Площадь',  value: `S = 6a² = 6×${a}² = ${6*a*a}` },
    { label: 'Диагональ',value: `d = a√3 = ${a}×1.732 ≈ ${(a*Math.sqrt(3)).toFixed(2)}` },
  ];
}

function stepsProgression(p) {
  const a1 = p.a1 ?? 2, d = p.d ?? 3;
  const n = Math.min(8, Math.max(2, Math.round(p.n ?? 5)));
  const an = a1 + d * (n - 1);
  const Sn = n * (a1 + an) / 2;
  return [
    { label: 'Формула aₙ', value: `aₙ = a₁ + (n−1)d` },
    { label: `a${n}`,       value: `${a1} + (${n}−1)×${d} = ${an}` },
    { label: 'Формула Sₙ',  value: `Sₙ = n(a₁+aₙ)/2` },
    { label: `S${n}`,       value: `${n}×(${a1}+${an})/2 = ${Sn}` },
  ];
}

// ── Public API ────────────────────────────────────────────────

function renderDiagram(container, type, params) {
  const renderers = {
    triangle:      () => svgTriangle(params),
    circle:        () => svgCircle(params),
    rectangle:     () => svgRectangle(params),
    trapezoid:     () => svgTrapezoid(params),
    parabola:      () => svgParabola(params),
    line:          () => svgLineDiagram(params),
    trig_triangle: () => svgTrigTriangle(params),
    unit_circle:   () => svgUnitCircle(params),
    speed:         () => svgSpeed(params),
    volume_cube:   () => svgVolumeCube(params),
    progression:   () => svgProgression(params),
  };
  container.innerHTML = renderers[type] ? renderers[type]() : '';
}

function getDiagramSteps(type, params) {
  const stepFns = {
    triangle:      () => stepsTriangle(params),
    circle:        () => stepsCircle(params),
    rectangle:     () => stepsRectangle(params),
    trapezoid:     () => stepsTrapezoid(params),
    parabola:      () => stepsParabola(params),
    line:          () => stepsLine(params),
    trig_triangle: () => stepsTrigTriangle(params),
    unit_circle:   () => stepsUnitCircle(params),
    speed:         () => stepsSpeed(params),
    volume_cube:   () => stepsVolumeCube(params),
    progression:   () => stepsProgression(params),
  };
  return stepFns[type] ? stepFns[type]() : [];
}
```

- [ ] **Step 2: Verify the file was created**

```bash
python -c "
import re
txt = open('d:/Project/Matmozg/diagrams.js').read()
fns = ['renderDiagram','getDiagramSteps','DIAGRAM_PARAMS','svgTriangle','svgCircle','svgRectangle','svgTrapezoid','svgParabola','svgLineDiagram','svgTrigTriangle','svgUnitCircle','svgSpeed','svgVolumeCube','svgProgression']
missing = [f for f in fns if f not in txt]
print('Missing:', missing if missing else 'none — OK')
"
```
Expected: `Missing: none — OK`

---

## Task 3: Update style.css

**Files:**
- Modify: `d:/Project/Matmozg/style.css`

- [ ] **Step 1: Remove animation panel styles**

Read `style.css` and delete these blocks entirely (they contain `anim-card`, `anim-canvas`, `anim-top-row`, `anim-controls-bar`, `anim-controls-grid`, `anim-param`, `reset-btn`). The section starts at the comment `/* ─── ANIMATION CANVAS ───` and ends before the `/* ─── SETTINGS ───` section.

- [ ] **Step 2: Add diagram panel styles**

Add the following block immediately after the `/* ─── KNOWLEDGE BASE ───` section (after the `.kb-detail { ... }` rule):

```css
/* ─────────────────────────────────────────────
   DIAGRAM PANEL
───────────────────────────────────────────── */
.diagram-card {
  height: calc(100vh - 130px);
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.diagram-svg-area {
  flex: 1;
  background: var(--surface2);
  border: 1px solid var(--border);
  border-radius: var(--radius-sm);
  min-height: 200px;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
}

.diagram-placeholder {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
  color: var(--text3);
  font-size: 13px;
  pointer-events: none;
}

.diagram-placeholder svg {
  width: 40px;
  height: 40px;
  color: var(--text3);
  opacity: 0.5;
}

.diagram-params {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  align-items: flex-end;
}

.diagram-steps {
  background: var(--surface2);
  border: 1px solid var(--border);
  border-radius: var(--radius-sm);
  padding: 10px 12px;
  font-family: var(--font-mono);
  font-size: 12px;
  color: var(--text);
  max-height: 120px;
  overflow-y: auto;
  line-height: 1.7;
}

.step-row {
  display: flex;
  gap: 10px;
}

.step-label {
  color: var(--text3);
  min-width: 110px;
  flex-shrink: 0;
}

.step-value {
  color: var(--green);
}

/* SVG diagram classes (inline SVG inherits document CSS) */
.diag-svg { width: 100%; height: 100%; }
.diag-svg .shape-fill        { fill: var(--accent-soft); }
.diag-svg .shape-stroke      { fill: none; stroke: var(--text2); stroke-width: 1.8; stroke-linejoin: round; }
.diag-svg .shape-stroke-only { fill: none; stroke: var(--text2); stroke-width: 1.8; }
.diag-svg .shape-stroke-line { fill: none; stroke: var(--text2); stroke-width: 1.8; }
.diag-svg .accent-line       { stroke: var(--accent); stroke-width: 2; fill: none; }
.diag-svg .sin-line          { stroke: var(--green); stroke-width: 1.8; stroke-dasharray: 4,3; fill: none; }
.diag-svg .cos-line          { stroke: var(--accent); stroke-width: 1.8; stroke-dasharray: 4,3; fill: none; }
.diag-svg .dashed-line       { stroke: var(--text3); stroke-width: 1; stroke-dasharray: 5,4; fill: none; }
.diag-svg .angle-mark        { fill: none; stroke: var(--text2); stroke-width: 1.2; }
.diag-svg .arc-line          { fill: none; stroke: var(--text2); stroke-width: 1.5; }
.diag-svg .axis-line         { stroke: var(--text3); stroke-width: 1.2; fill: none; }
.diag-svg .circle-outline    { fill: var(--accent-soft); stroke: var(--text2); stroke-width: 1.8; }
.diag-svg .curve-line        { fill: none; stroke: var(--accent); stroke-width: 2.2; stroke-linecap: round; stroke-linejoin: round; }
.diag-svg .vertex-dot        { fill: var(--accent); }
.diag-svg .root-dot          { fill: var(--green); }
.diag-svg .face-top          { fill: var(--surface3); stroke: var(--text2); stroke-width: 1.3; }
.diag-svg .face-right        { fill: var(--surface2); stroke: var(--text2); stroke-width: 1.3; }
.diag-svg .face-left         { fill: var(--bg);       stroke: var(--text2); stroke-width: 1.3; }
.diag-svg .arrow-head        { fill: var(--accent); }
.diag-svg text               { font-family: var(--font-mono); font-size: 13px; fill: var(--text2); dominant-baseline: middle; }
.diag-svg .lbl-accent        { fill: var(--accent); font-weight: 600; font-size: 12px; }
.diag-svg .lbl-green         { fill: var(--green);  font-weight: 600; font-size: 12px; }
.diag-svg .lbl-formula       { fill: var(--text);   font-size: 12px; }
.diag-svg .lbl-dim           { fill: var(--text3);  font-size: 11px; }
```

---

## Task 4: Update index.html

**Files:**
- Modify: `d:/Project/Matmozg/index.html`

- [ ] **Step 1: Replace `.kb-right` contents**

Find this block in `index.html` (the entire `.kb-right` div, lines ~216–231):

```html
        <!-- Right: animation canvas -->
        <div class="kb-right">
          <div class="card anim-card">
            <div class="anim-top-row">
              <label class="field-label">Animatsiya</label>
              <div class="anim-controls-bar" id="animControlsBar" style="display:none">
                <div class="anim-controls-grid" id="animControls"></div>
                <button class="btn-ghost-sm reset-btn" onclick="resetAnimParams()">
                  <svg viewBox="0 0 14 14" fill="none"><path d="M1 7a6 6 0 1011-3.5M11 1v3H8" stroke="currentColor" stroke-width="1.3" stroke-linecap="round" stroke-linejoin="round"/></svg>
                  Reset
                </button>
              </div>
            </div>
            <canvas id="animCanvas" class="anim-canvas"></canvas>
          </div>
        </div>
```

Replace it with:

```html
        <!-- Right: diagram panel -->
        <div class="kb-right">
          <div class="card diagram-card">
            <div class="diagram-svg-area" id="diagramSvgArea">
              <div class="diagram-placeholder" id="diagramPlaceholder">
                <svg viewBox="0 0 40 40" fill="none" xmlns="http://www.w3.org/2000/svg">
                  <rect x="4" y="4" width="32" height="32" rx="4" stroke="currentColor" stroke-width="1.5"/>
                  <path d="M12 20h16M20 12v16" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
                </svg>
                <span>Formulani tanlang</span>
              </div>
            </div>
            <div class="diagram-params" id="diagramParams"></div>
            <div class="diagram-steps" id="diagramSteps"></div>
          </div>
        </div>
```

- [ ] **Step 2: Add `<script src="diagrams.js">` before `<script src="app.js">`**

Find at the bottom of `<body>`:
```html
<script src="app.js"></script>
```

Replace with:
```html
<script src="diagrams.js"></script>
<script src="app.js"></script>
```

---

## Task 5: Update app.js

**Files:**
- Modify: `d:/Project/Matmozg/app.js`

- [ ] **Step 1: Remove state variables and ANIM_CONTROLS**

Find and delete these lines (around lines 26–58):
```js
let animTitle   = null;
let animT       = 0;

// User-controllable animation parameters
let animParams        = {};
let animParamDefaults = {};

// ── Animation parameter definitions ─────────────────────────
const ANIM_CONTROLS = {
  Pifagor: [ ... ],
  Kvadrat: [ ... ],
  Aylana:  [ ... ],
  burchak: [ ... ],
  Tezlik:  [ ... ],
  wave:    [ ... ],
};
```

- [ ] **Step 2: Remove `requestAnimationFrame(resizeCanvas)` from tab switcher**

Find in the tab switcher block (around line 71):
```js
    if (tab === 'kb') requestAnimationFrame(resizeCanvas);
```
Delete that line entirely.

- [ ] **Step 3: Remove `startAnimLoop()` call from DOMContentLoaded**

Find (around line 102):
```js
  startAnimLoop();
```
Delete that line.

- [ ] **Step 4: Update `selectFormula()` to call `showDiagram` instead of animation functions**

Find the current `selectFormula` function:
```js
function selectFormula(i) {
  activeKbIdx = i;
  const f = kb[i];
  $('kbDetail').textContent = f ? `${f.title}\n\n${f.content}` : '';
  animTitle = f ? f.title : null;
  animT     = 0;
  renderKBList();
  renderAnimControls(animTitle);
}
```

Replace with:
```js
function selectFormula(i) {
  activeKbIdx = i;
  const f = kb[i];
  $('kbDetail').textContent = f ? `${f.title}\n\n${f.content}` : '';
  renderKBList();
  showDiagram(f || null);
}
```

- [ ] **Step 5: Update `deleteFormula()` to call `showDiagram` instead of animation functions**

Find in `deleteFormula` (around line 484):
```js
    else { animTitle = null; $('kbDetail').textContent = ''; renderAnimControls(null); }
```

Replace with:
```js
    else { $('kbDetail').textContent = ''; showDiagram(null); }
```

- [ ] **Step 6: Remove the entire animation controls section**

Delete the block from the comment `// ════ ANIMATION CONTROLS` through `resetAnimParams()` (approximately lines 488–540):
```js
// ════════════════════════════════════════════════════════════
//  ANIMATION CONTROLS (interactive parameters)
// ════════════════════════════════════════════════════════════
function renderAnimControls(title) { ... }
function onParamChange() { ... }
function resetAnimParams() { ... }
```

- [ ] **Step 7: Remove the entire canvas animation engine**

Delete from `// ════ CANVAS ANIMATION ENGINE` through the end of all animation draw functions and canvas helpers. This covers approximately lines 542 to the end of the file. Specifically delete:
- `const canvas = ...`, `const ctx = ...`
- `const ANIM_SPEED = ...`
- `function resizeCanvas() { ... }`
- `window.addEventListener('resize', resizeCanvas);`
- `function startAnimLoop() { ... }` (including the nested `tick()`)
- All canvas helpers: `function rgba(...)`, `hexA(...)`, `setLine(...)`, `line(...)`, `circle(...)`, `arrow(...)`, `axes(...)`, `gridLines(...)`, `label(...)`, `infoBox(...)`, `round2(...)`
- All draw functions: `drawPythagoras(...)`, `drawParabola(...)`, `drawCircleArea(...)`, `drawRectangle(...)`, `drawSpeed(...)`, `drawWave(...)`

- [ ] **Step 8: Add `showDiagram()` and `onDiagramParam()` functions**

Add the following block after the `addFormula()` function (after line ~476):

```js
// ════════════════════════════════════════════════════════════
//  DIAGRAM PANEL
// ════════════════════════════════════════════════════════════
function showDiagram(item) {
  const type        = item && item.diagram ? item.diagram : 'none';
  const svgArea     = $('diagramSvgArea');
  const paramsDiv   = $('diagramParams');
  const stepsDiv    = $('diagramSteps');
  const placeholder = $('diagramPlaceholder');

  if (type === 'none' || !type) {
    svgArea.innerHTML = '';
    svgArea.appendChild(placeholder);
    placeholder.style.display = 'flex';
    paramsDiv.style.display   = 'none';
    stepsDiv.style.display    = 'none';
    return;
  }

  placeholder.style.display = 'none';

  // Build default params
  const paramDefs = DIAGRAM_PARAMS[type] || [];
  const params    = {};
  paramDefs.forEach(def => { params[def.id] = def.value; });

  // Render SVG
  renderDiagram(svgArea, type, params);

  // Build param inputs
  if (paramDefs.length > 0) {
    paramsDiv.style.display = 'flex';
    paramsDiv.innerHTML = paramDefs.map(def =>
      `<div class="anim-param">
        <span class="anim-param-label">${esc(def.label)}</span>
        <input type="number" data-param-id="${def.id}" value="${def.value}"
               min="${def.min}" max="${def.max}" step="${def.step}"
               oninput="onDiagramParam('${type}')">
      </div>`
    ).join('');
  } else {
    paramsDiv.style.display = 'none';
  }

  // Render steps
  _updateDiagramSteps(type, params, stepsDiv);
}

function onDiagramParam(type) {
  const paramsDiv = $('diagramParams');
  const params    = {};
  paramsDiv.querySelectorAll('input[type=number]').forEach(inp => {
    params[inp.dataset.paramId] = parseFloat(inp.value) || 0;
  });
  renderDiagram($('diagramSvgArea'), type, params);
  _updateDiagramSteps(type, params, $('diagramSteps'));
}

function _updateDiagramSteps(type, params, stepsDiv) {
  const steps = getDiagramSteps(type, params);
  if (steps.length > 0) {
    stepsDiv.style.display = 'block';
    stepsDiv.innerHTML = steps.map(s =>
      `<div class="step-row"><span class="step-label">${esc(s.label)}:</span><span class="step-value">${s.value}</span></div>`
    ).join('');
  } else {
    stepsDiv.style.display = 'none';
  }
}
```

- [ ] **Step 9: Verify no canvas or animation references remain**

```bash
python -c "
import re
txt = open('d:/Project/Matmozg/app.js').read()
dead = ['animCanvas','animTitle','animT','animParams','animParamDefaults','ANIM_CONTROLS','startAnimLoop','resizeCanvas','renderAnimControls','resetAnimParams','drawPythagoras','drawParabola','drawCircleArea','drawWave','ANIM_SPEED']
found = [x for x in dead if x in txt]
print('Still present:', found if found else 'none — OK')
"
```
Expected: `Still present: none — OK`

---

## Task 6: Manual verification

- [ ] **Step 1: Start the server**

```bash
cd d:/Project/Matmozg && python server.py
```

Open `http://localhost:5000` in a browser (or whatever port the server uses — check the terminal output).

- [ ] **Step 2: Verify KB tab default state**

Click the **Baza** tab.

Expected:
- Right panel shows a card with a `+` icon placeholder and text "Formulani tanlang"
- No canvas, no animation controls visible

- [ ] **Step 3: Verify triangle diagram**

Click **Пифагор теоремасы** in the formula list.

Expected:
- Right panel shows a right triangle SVG with a=3, b=4, c=5.000 labelled
- Two parameter inputs below: `a (катет)` and `b (катет)`
- Steps block shows formula, substitution, calculation, answer

- [ ] **Step 4: Verify live parameter update**

Change `a` to `5` and `b` to `12`.

Expected:
- SVG redraws immediately showing the new triangle
- Steps block updates to show c = √(25+144) = √169 = 13.0000

- [ ] **Step 5: Verify parabola diagram**

Click **Парабола y = ax² + bx + c**.

Expected:
- Coordinate axes and parabola curve drawn
- Vertex point marked
- Root dots shown (if D ≥ 0)
- Parameters a, b, c editable

- [ ] **Step 6: Verify unit circle diagram**

Click **Тригонометрия — формулы сложения**.

Expected:
- Unit circle drawn with angle projection lines
- sin and cos projections visible with values
- Angle parameter editable, circle updates

- [ ] **Step 7: Verify "none" diagram type**

Click **Логарифмы — правила**.

Expected:
- Placeholder with `+` icon shown (no SVG)
- No param inputs, no steps

- [ ] **Step 8: Verify both dark and light themes**

Toggle the theme using the button in the top bar.

Expected:
- SVG diagrams adapt to the active theme colours (light-blue axes on dark, darker strokes on light)
- No hardcoded dark colours remain visible in light mode

# KB Diagram Panel — Design Spec

**Date:** 2026-04-14
**Project:** MatematikaAI

---

## Summary

Replace the canvas animation panel on the right side of the Knowledge Base (Baza) tab with an interactive SVG diagram panel. When a user clicks a formula from the list on the left, the right panel renders an SVG drawing specific to that formula, with editable numeric parameters that update the drawing in real time, and a step-by-step solution block below.

---

## What Is Removed

- `<canvas id="animCanvas">` element from `index.html`
- `.anim-card`, `.anim-canvas`, `.anim-top-row`, `.anim-controls-bar`, `.anim-controls-grid`, `.anim-param`, `.reset-btn` CSS rules from `style.css`
- All canvas animation JS from `app.js`:
  - `ANIM_CONTROLS` constant
  - `startAnimLoop()`, `tick()`, `resizeCanvas()`
  - `drawPythagoras()`, `drawParabola()`, `drawCircleArea()`, `drawRectangle()`, `drawSpeed()`, `drawWave()`
  - Canvas helper functions: `rgba()`, `hexA()`, `setLine()`, `line()`, `circle()`, `arrow()`, `axes()`, `gridLines()`, `label()`, `infoBox()`, `round2()`
  - `animTitle`, `animT`, `animParams`, `animParamDefaults` state variables
  - `resetAnimParams()` function
  - The `requestAnimationFrame` to `resizeCanvas` in the tab switcher

---

## What Is Added

### `diagrams.js` (new file)

Standalone JS module with all SVG diagram rendering functions. Loaded via `<script src="diagrams.js">` in `index.html`. Exports one public function:

```js
renderDiagram(container, diagramType, params)
```

- `container` — DOM element to render into
- `diagramType` — string key (see Diagram Types table)
- `params` — object with numeric parameter values

Each diagram function returns an SVG string. `renderDiagram` sets `container.innerHTML`.

### `index.html` changes

Replace the `.kb-right` div contents:

```html
<div class="kb-right">
  <div class="card diagram-card">
    <div class="diagram-svg-area" id="diagramSvgArea">
      <!-- placeholder shown when no formula selected -->
      <div class="diagram-placeholder" id="diagramPlaceholder">
        <svg><!-- formula icon --></svg>
        <span>Formulani tanlang</span>
      </div>
    </div>
    <div class="diagram-params" id="diagramParams" style="display:none"></div>
    <div class="diagram-steps" id="diagramSteps" style="display:none"></div>
  </div>
</div>
```

### `style.css` changes

Add styles for `.diagram-card`, `.diagram-svg-area`, `.diagram-placeholder`, `.diagram-params`, `.diagram-steps`. Remove old animation styles.

### `knowledge_base.json` changes

Add `"diagram"` field to every entry. Value is one of the diagram type keys below, or `"none"` for table/text-only entries.

### `app.js` changes

- Remove all animation code (see above)
- Add `showDiagram(item)` function — called when a KB item is clicked
- `showDiagram` reads `item.diagram`, builds param inputs, calls `renderDiagram`, renders steps
- Param inputs have `oninput` handler that calls `renderDiagram` with updated values

---

## Diagram Types

| Key | Used for | Parameters | What is drawn |
|-----|----------|------------|---------------|
| `triangle` | Пифагор, Герон | a, b | Right triangle with sides a, b, c=√(a²+b²); right-angle mark; labels |
| `circle` | Шеңбер | r | Circle with radius line, diameter line, arc label C=2πr |
| `rectangle` | Квадрат/прямоугольник | a, b | Rectangle with dimension labels, diagonal line |
| `trapezoid` | Трапеция | a, b, h | Trapezoid with top/bottom base labels, height arrow |
| `parabola` | Парабола, Квадратное уравнение | a, b, c | Coordinate axes, parabola curve, vertex point, roots if D≥0 |
| `line` | Уравнение прямой | k, b | Coordinate axes, straight line y=kx+b, slope annotation |
| `trig_triangle` | Тригонометрия (основные функции) | angle | Right triangle with hypotenuse=1, legs=sin/cos, angle arc and label |
| `unit_circle` | Тригонометрические значения таблица | angle | Unit circle, radius at given angle, sin/cos projections on axes |
| `speed` | Tezlik | v, t | Horizontal arrow for distance s=v×t, speed label above, time below |
| `volume_cube` | Объёмы | a | Isometric cube wireframe with edge label a, V=a³ annotation |
| `progression` | Арифм. и геом. прогрессии | a1, d, n | Number line with n dots, spacing=d, first/last member labelled |
| `none` | Tables, divisibility rules, etc. | — | No SVG; only text content from `content` field is shown |

---

## SVG Theming

All SVG diagrams use CSS variables for colours:

- Stroke/lines: `var(--text2)`
- Axis lines: `var(--text3)`
- Accent lines (hypotenuse, radius, key element): `var(--accent)` (`#4a9eff`)
- Right-angle box: `var(--text2)`
- Labels: `var(--text)` for values, `var(--text2)` for variable names
- Fill for shapes: `var(--accent-soft)`
- Background: transparent (inherits from `.diagram-svg-area` which uses `var(--surface2)`)

This ensures diagrams look correct in both dark and light themes.

---

## Step-by-Step Block

Below the diagram, a `.diagram-steps` div shows a computed example using the current parameter values. Format:

```
Формула:   a² + b² = c²
Подстановка: 3² + 4² = c²
Вычисление:  9 + 16 = 25
Ответ:     c = √25 = 5
```

Each diagram type has a `steps(params)` function in `diagrams.js` that returns an array of `{label, value}` lines.

---

## `knowledge_base.json` diagram field assignments

| Title | diagram |
|-------|---------|
| Квадратное уравнение | parabola |
| Виет теоремасы | parabola |
| Квадраттың қысқартылған көбейтіндісі | none |
| Линейное уравнение | line |
| Пифагор теоремасы | triangle |
| Треугольник — площадь и периметр | triangle |
| Шеңбер — формулалар | circle |
| Квадрат и прямоугольник | rectangle |
| Трапеция | trapezoid |
| Объёмы тел | volume_cube |
| Степени и корни | none |
| Проценты — формулы | none |
| Дроби — действия | none |
| Пропорция | none |
| Тригонометрия — основные функции | trig_triangle |
| Тригонометрия — формулы сложения | unit_circle |
| Арифметическая прогрессия | progression |
| Геометрическая прогрессия | progression |
| Логарифмы — правила | none |
| Производная — основные правила | none |
| Интеграл — основные формулы | none |
| Комбинаторика | none |
| Теория вероятностей | none |
| Статистика | none |
| Делимость — признаки | none |
| НОД и НОК | none |
| Неравенства — правила | none |
| Уравнение прямой | line |
| Парабола y = ax²+bx+c | parabola |
| Координатная плоскость | line |
| Векторы | none |
| Пифагор үшліктері | triangle |
| Тригонометрик теңлемелер | unit_circle |
| Матрицалар | none |
| Комплекс сандар | none |

---

## File Map

| File | Action |
|------|--------|
| `diagrams.js` | Create — all SVG renderers + step functions |
| `index.html` | Modify — replace `.kb-right` contents, add `<script src="diagrams.js">` |
| `style.css` | Modify — remove animation styles, add diagram panel styles |
| `app.js` | Modify — remove animation code, add `showDiagram()` |
| `knowledge_base.json` | Modify — add `"diagram"` field to every entry |

---

## Success Criteria

1. Selecting any formula from the list shows the correct SVG diagram on the right
2. Changing a parameter number updates the SVG and steps in real time
3. Formulas with `"diagram": "none"` show a clean placeholder
4. All diagrams render correctly in both dark and light themes
5. No canvas or animation code remains in the project

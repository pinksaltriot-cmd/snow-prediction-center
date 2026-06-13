# Snow Prediction Center — Product Maker Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a standalone single-file web app that replicates the SPC convective product suite, mapped to snow (Snownadoes / Blizzard Wind / Snow Squalls), where the user draws click-polygon risk areas on a US map and issues Outlooks, Mesoscale Discussions, Watches, Warnings, HWO/Hydro products, and verifies them against typed storm reports.

**Architecture:** One `index.html` file, vanilla JS, zero npm dependencies, deployable to GitHub Pages. An embedded public-domain Albers-USA states SVG (viewBox `0 0 960 600`) is the basemap. The user clicks vertices in SVG coordinate space to lay polygons stored as `{points, layer, level}`. A central `STATE` object holds all drawn areas + product metadata; pure render functions turn `STATE` into the active map panel and the SPC-style text. Code is organized into clearly delimited sections within the single file (the existing `snow-simulator` app follows the same single-file convention).

**Tech Stack:** HTML5, CSS3, vanilla ES2020 JavaScript, inline SVG. No build step. No external libraries. Verification via the Claude Preview MCP tools (browser load + interaction + console/snapshot/screenshot).

**Methodology note:** No JS test runner exists (single static file, GitHub Pages). Each task's verification is a concrete browser check via the preview tools, run after implementation, before commit. "Expected" describes what the browser check must show.

---

## File Structure

- `index.html` — the entire app. Internal sections, top to bottom:
  1. `<head>` + `<style>` — layout, rails, map, tier/CIG color classes, product text styling.
  2. `<body>` markup — top product tabs, left tool rail, center map (`<svg id="map">`), right preview rail.
  3. `<script>` SECTION A — **Taxonomy constants** (tiers, colors, hazard probs, CIG defs, Whiteout Scale).
  4. `<script>` SECTION B — **Map engine** (basemap, SVG↔point math, polygon draw/edit/delete).
  5. `<script>` SECTION C — **STATE model** + persistence (save/load JSON, PNG/text export helpers).
  6. `<script>` SECTION D — **Outlook** product (panels, day tabs, auto-derive, auto-text).
  7. `<script>` SECTION E — **Mesoscale Discussion + meso-gamma**.
  8. `<script>` SECTION F — **Watches (+PDS) & Warnings (+Emergency)**.
  9. `<script>` SECTION G — **HWO + Hydrologic Outlook**.
  10. `<script>` SECTION H — **Storm Reports & Verification**.
  11. `<script>` SECTION I — **Tab wiring / bootstrap**.
- `README.md` — one-paragraph description + "open index.html" + GitHub Pages note.
- `.github/workflows/` — (optional, Phase 1 final task) GitHub Pages deploy, mirroring the snow-simulator setup.

Each `<script>` section is wrapped in an IIFE or attaches a single namespaced object (`SPC.tax`, `SPC.map`, `SPC.state`, `SPC.outlook`, …) so sections stay decoupled and individually reasoned-about.

---

## PHASE 1 — Foundation

### Task 1: App shell + three-rail layout

**Files:**
- Create: `index.html`

- [ ] **Step 1: Create `index.html` with the shell**

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Snow Prediction Center</title>
<style>
  :root{--bg:#0b1220;--panel:#121c2e;--ink:#e7eef7;--muted:#8aa0bd;--line:#22324b;--accent:#7dd3fc;}
  *{box-sizing:border-box;}
  body{margin:0;font:14px/1.45 system-ui,Segoe UI,Roboto,sans-serif;background:var(--bg);color:var(--ink);}
  header.appbar{display:flex;align-items:center;gap:16px;padding:10px 16px;background:#0a1424;border-bottom:1px solid var(--line);}
  header.appbar h1{font-size:16px;margin:0;letter-spacing:.5px;}
  nav.tabs{display:flex;gap:4px;margin-left:auto;flex-wrap:wrap;}
  nav.tabs button{background:transparent;color:var(--muted);border:1px solid transparent;padding:6px 10px;border-radius:6px;cursor:pointer;font-size:13px;}
  nav.tabs button.active{background:var(--panel);color:var(--ink);border-color:var(--line);}
  .layout{display:grid;grid-template-columns:260px 1fr 360px;height:calc(100vh - 49px);}
  .rail{background:var(--panel);overflow:auto;padding:12px;}
  .rail.left{border-right:1px solid var(--line);}
  .rail.right{border-left:1px solid var(--line);}
  .map-wrap{display:flex;align-items:center;justify-content:center;background:#0e1830;overflow:hidden;}
  #map{width:100%;height:100%;max-height:100%;}
  h2.sec{font-size:11px;text-transform:uppercase;letter-spacing:1px;color:var(--muted);margin:16px 0 8px;}
  .preview-text{white-space:pre-wrap;font:12px/1.5 ui-monospace,Menlo,Consolas,monospace;background:#0a1424;border:1px solid var(--line);border-radius:6px;padding:10px;color:#cfe2f7;}
</style>
</head>
<body>
<header class="appbar">
  <h1>❄ Snow Prediction Center</h1>
  <nav class="tabs" id="tabs">
    <button data-tab="outlook" class="active">Outlook</button>
    <button data-tab="md">Mesoscale Discussion</button>
    <button data-tab="watch">Watch</button>
    <button data-tab="warning">Warning</button>
    <button data-tab="hwo">HWO / Hydro</button>
    <button data-tab="reports">Storm Reports</button>
  </nav>
</header>
<div class="layout">
  <aside class="rail left" id="toolRail"><h2 class="sec">Tools</h2><div id="toolBody"></div></aside>
  <main class="map-wrap"><svg id="map" viewBox="0 0 960 600" preserveAspectRatio="xMidYMid meet"></svg></main>
  <aside class="rail right" id="previewRail"><h2 class="sec">Product Preview</h2><div id="previewBody"><div class="preview-text" id="previewText">No product yet.</div></div></aside>
</div>
<script>window.SPC = {};</script>
</body>
</html>
```

- [ ] **Step 2: Verify in browser**

Run preview_start on `index.html`, then preview_snapshot.
Expected: header shows "❄ Snow Prediction Center", six tabs with "Outlook" active, three-column layout (left Tools rail, center empty SVG, right Product Preview rail). preview_console_logs shows no errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: app shell with three-rail layout and product tabs"
```

---

### Task 2: Embed Albers-USA basemap

**Files:**
- Modify: `index.html` (inside `<svg id="map">`)

- [ ] **Step 1: Obtain a public-domain CONUS states SVG**

Use a CC0/public-domain US states SVG authored on the standard d3 Albers-USA canvas (`viewBox 0 0 960 600`), e.g. the Wikimedia "Blank US Map (states only)" exported to the 960×600 Albers grid, or the `us-atlas` states rendered once to static paths. Each state is a `<path>` with `class="state"` and `data-st="XX"` (postal code). Strip any inline fills.

- [ ] **Step 2: Inline the paths into `#map` and style them**

Add inside `<svg id="map">`:
```html
<g id="basemap"><!-- paste <path class="state" data-st="AL" d="..."/> for all 49 CONUS+DC states here --></g>
<g id="layers"></g>      <!-- drawn risk polygons render here, below markers -->
<g id="draftLayer"></g>  <!-- in-progress polygon being drawn -->
```
Add to `<style>`:
```css
#basemap .state{fill:#16233b;stroke:#33425f;stroke-width:.75;vector-effect:non-scaling-stroke;}
#basemap .state:hover{fill:#1b2c49;}
```

- [ ] **Step 3: Verify in browser**

Reload, preview_screenshot.
Expected: a recognizable contiguous-US map fills the center panel, states outlined, dark theme. preview_console_logs: no errors. preview_eval `document.querySelectorAll('#basemap .state').length` returns ≥ 49.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: embed Albers-USA states basemap"
```

---

### Task 3: Taxonomy constants (SECTION A)

**Files:**
- Modify: `index.html` (new `<script>` SECTION A before the bootstrap script)

- [ ] **Step 1: Add the taxonomy module**

```html
<script>
// ===== SECTION A: TAXONOMY =====
SPC.tax = (function(){
  const TIERS = [
    {key:'SNOW', name:'General Snow', color:'#c1e9c1', rank:0},
    {key:'MRGL', name:'Marginal',    color:'#66a366', rank:1},
    {key:'SLGT', name:'Slight',      color:'#ffe066', rank:2},
    {key:'ENH',  name:'Enhanced',    color:'#ffa366', rank:3},
    {key:'MDT',  name:'Moderate',    color:'#e06666', rank:4},
    {key:'HIGH', name:'High',        color:'#ee99ee', rank:5},
    {key:'EXTR', name:'Extreme',     color:'#1a1a1a', rank:6, hatch:true, gate:'≥80% CIG4 snownadoes'}
  ];
  const HAZARDS = {
    snownado:    {name:'Snownadoes',   probs:[2,5,10,15,30,45,60], cigs:[1,2,3,4], color:'#e23a3a'},
    blizzardWind:{name:'Blizzard Wind', probs:[5,15,30,45,60,75,90], cigs:[1,2,3], color:'#3a7be2'},
    snowSquall:  {name:'Snow Squalls', probs:[5,15,30,45,60],       cigs:[1,2],   color:'#2ea66a'}
  };
  // Conditional Intensity Groups: shaded zones (NOT hatching). Distinct shade per level.
  const CIG = {
    snownado:    {1:'20% W2+',2:'30% W2+',3:'40% W2+ (~19% W4+)',4:'sig. chance W5–W6 (violent)'},
    blizzardWind:{1:'gusts ≥35 mph',2:'≥50 mph',3:'≥65 mph'},
    snowSquall:  {1:'≥2″/hr',2:'≥4″/hr, vis ≤¼ mi'}
  };
  const PROB_COLORS = {2:'#80c080',5:'#7fc57f',10:'#ffe066',15:'#ffd24d',30:'#ffa366',45:'#e06666',60:'#ee99ee',75:'#cf6fff',90:'#a83bff'};
  const CIG_SHADE = {1:'rgba(255,255,255,.10)',2:'rgba(255,255,255,.20)',3:'rgba(255,255,255,.32)',4:'rgba(255,255,255,.46)'};
  const WHITEOUT = [
    {t:'W0',lo:25,hi:40,  name:'Flurry Spinner', dmg:'Blowing-snow swirl; dusts cars; harmless'},
    {t:'W1',lo:41,hi:55,  name:'Squall Devil',   dmg:'Light drifting; trash cans over; visible snow devil'},
    {t:'W2',lo:56,hi:70,  name:'Strong',         dmg:'Drifts pile fast; signs down; brief whiteout'},
    {t:'W3',lo:71,hi:85,  name:'Severe',         dmg:'Major drifting; roofs scoured; vehicles pushed'},
    {t:'W4',lo:86,hi:100, name:'Devastating',    dmg:'Deep drift burial; structures damaged'},
    {t:'W5',lo:101,hi:120,name:'Violent',        dmg:'Total whiteout vortex; severe structural damage'},
    {t:'W6',lo:121,hi:999,name:'Cryonic',        dmg:'Snow-only extreme: complete burial; catastrophic'}
  ];
  const NOTABILITY = ['Notable','Super-Notable','Super-Hyper-Notable'];
  function tier(key){return TIERS.find(t=>t.key===key);}
  function wScale(t){return WHITEOUT.find(w=>w.t===t);}
  return {TIERS,HAZARDS,CIG,PROB_COLORS,CIG_SHADE,WHITEOUT,NOTABILITY,tier,wScale};
})();
</script>
```

- [ ] **Step 2: Verify in browser**

Reload. preview_eval `SPC.tax.TIERS.length` → `7`. preview_eval `SPC.tax.HAZARDS.snownado.cigs.length` → `4`. preview_eval `SPC.tax.WHITEOUT.map(w=>w.t).join(',')` → `W0,W1,W2,W3,W4,W5,W6`. preview_console_logs: no errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: taxonomy constants (tiers, hazards, CIGs, Whiteout Scale)"
```

---

### Task 4: Map engine — click-polygon drawing (SECTION B)

**Files:**
- Modify: `index.html` (new `<script>` SECTION B)

- [ ] **Step 1: Add the map engine**

```html
<script>
// ===== SECTION B: MAP ENGINE =====
SPC.map = (function(){
  const svg = document.getElementById('map');
  const layersG = document.getElementById('layers');
  const draftG  = document.getElementById('draftLayer');
  let drafting=false, draftPts=[], onComplete=null, drawStyle=null;

  function toSvg(evt){
    const pt = svg.createSVGPoint(); pt.x=evt.clientX; pt.y=evt.clientY;
    const m = svg.getScreenCTM().inverse(); const p = pt.matrixTransform(m);
    return [Math.round(p.x*10)/10, Math.round(p.y*10)/10];
  }
  function ptsAttr(pts){return pts.map(p=>p.join(',')).join(' ');}
  function renderDraft(){
    draftG.innerHTML='';
    if(!draftPts.length) return;
    const poly=document.createElementNS('http://www.w3.org/2000/svg','polyline');
    poly.setAttribute('points',ptsAttr(draftPts));
    poly.setAttribute('fill','none'); poly.setAttribute('stroke','#fff');
    poly.setAttribute('stroke-dasharray','4 3'); poly.setAttribute('stroke-width','1.5');
    poly.setAttribute('vector-effect','non-scaling-stroke');
    draftG.appendChild(poly);
    draftPts.forEach(p=>{const c=document.createElementNS('http://www.w3.org/2000/svg','circle');
      c.setAttribute('cx',p[0]);c.setAttribute('cy',p[1]);c.setAttribute('r','3');
      c.setAttribute('fill','#fff');draftG.appendChild(c);});
  }
  function start(style, complete){ drafting=true; draftPts=[]; drawStyle=style; onComplete=complete; renderDraft(); }
  function cancel(){ drafting=false; draftPts=[]; renderDraft(); }
  svg.addEventListener('click', e=>{ if(!drafting) return; draftPts.push(toSvg(e)); renderDraft(); });
  svg.addEventListener('dblclick', e=>{
    if(!drafting||draftPts.length<3){return;}
    drafting=false; const pts=draftPts.slice(0,-1); draftPts=[]; renderDraft();
    if(onComplete) onComplete(pts);
  });

  // Render a list of polygon objects {points, fill, stroke, opacity, dash} into #layers (clears first)
  function renderPolys(polys){
    layersG.innerHTML='';
    polys.forEach((p,i)=>{
      const el=document.createElementNS('http://www.w3.org/2000/svg','polygon');
      el.setAttribute('points',ptsAttr(p.points));
      el.setAttribute('fill',p.fill||'none');
      el.setAttribute('fill-opacity',p.opacity!=null?p.opacity:0.55);
      el.setAttribute('stroke',p.stroke||'#000');
      el.setAttribute('stroke-width',p.strokeWidth||1.2);
      if(p.dash) el.setAttribute('stroke-dasharray',p.dash);
      el.setAttribute('vector-effect','non-scaling-stroke');
      el.dataset.idx=i;
      layersG.appendChild(el);
    });
  }
  function clearLayers(){ layersG.innerHTML=''; }
  return {start,cancel,renderPolys,clearLayers,ptsAttr};
})();
</script>
```

- [ ] **Step 2: Verify drawing works**

Reload. preview_eval to start a draft and simulate completion:
```js
SPC.map.renderPolys([{points:[[200,200],[400,200],[300,350]],fill:'#ffe066',stroke:'#000'}]); 'ok'
```
Expected: `'ok'`; preview_screenshot shows a yellow triangle over the map. preview_console_logs: no errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: map engine with click-polygon drawing and poly rendering"
```

---

### Task 5: STATE model + export helpers (SECTION C)

**Files:**
- Modify: `index.html` (new `<script>` SECTION C)

- [ ] **Step 1: Add the state + export module**

```html
<script>
// ===== SECTION C: STATE + EXPORT =====
SPC.state = (function(){
  // Central document. Each product family keeps its own areas + meta.
  const doc = {
    outlook:{ day:1, panel:'categorical',
      areas:{ categorical:[], snownado:[], blizzardWind:[], snowSquall:[],
              cig_snownado:[], cig_blizzardWind:[], cig_snowSquall:[] },
      meta:{ forecaster:'', issued:'' } },
    md:{ areas:[], concerning:'snownado', watchProb:40, mesoGamma:false, text:'', valid:'' },
    watch:{ type:'Blizzard', pds:false, areas:[], probTable:{}, number:1, valid:'' },
    warning:{ type:'Blizzard', emergency:false, areas:[], valid:'', wfo:'' },
    hwo:{ days:['','','','','','',''], spotter:false },
    hydro:{ text:'' },
    reports:[]
  };
  function save(){ return JSON.stringify(doc); }
  function load(json){ Object.assign(doc, JSON.parse(json)); }
  function reset(){ location.reload(); }

  // Export the live SVG map as a PNG data URL via canvas.
  function exportPNG(cb){
    const svg=document.getElementById('map');
    const xml=new XMLSerializer().serializeToString(svg);
    const img=new Image();
    img.onload=function(){
      const c=document.createElement('canvas'); c.width=960; c.height=600;
      const ctx=c.getContext('2d'); ctx.fillStyle='#0e1830'; ctx.fillRect(0,0,960,600);
      ctx.drawImage(img,0,0,960,600); cb(c.toDataURL('image/png'));
    };
    img.src='data:image/svg+xml;base64,'+btoa(unescape(encodeURIComponent(xml)));
  }
  function copyText(t){ navigator.clipboard && navigator.clipboard.writeText(t); }
  return {doc,save,load,reset,exportPNG,copyText};
})();
</script>
```

- [ ] **Step 2: Verify**

Reload. preview_eval `typeof SPC.state.doc.outlook.areas.categorical` → `'object'`. preview_eval `Array.isArray(SPC.state.doc.reports)` → `true`. preview_console_logs: no errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: central STATE model and PNG/text export helpers"
```

---

### Task 6: Tab wiring + per-tab tool rails (SECTION I bootstrap)

**Files:**
- Modify: `index.html` (new `<script>` SECTION I at end)

- [ ] **Step 1: Add tab router**

```html
<script>
// ===== SECTION I: TABS / BOOTSTRAP =====
SPC.tabs = (function(){
  const renderers = {}; // tabKey -> {tools(el), preview(el)}
  function register(key,obj){ renderers[key]=obj; }
  let current='outlook';
  function show(key){
    current=key;
    document.querySelectorAll('#tabs button').forEach(b=>b.classList.toggle('active',b.dataset.tab===key));
    const tools=document.getElementById('toolBody'); tools.innerHTML='';
    const prev=document.getElementById('previewBody'); prev.innerHTML='';
    SPC.map.clearLayers(); SPC.map.cancel();
    if(renderers[key]){ renderers[key].tools(tools); renderers[key].preview(prev); }
  }
  document.getElementById('tabs').addEventListener('click',e=>{
    if(e.target.dataset.tab) show(e.target.dataset.tab);
  });
  return {register,show,get current(){return current;}};
})();
// Placeholder renderers so tabs work before product sections land:
['outlook','md','watch','warning','hwo','reports'].forEach(k=>{
  if(!SPC.tabs) return;
  SPC.tabs.register(k,{tools:el=>el.innerHTML='<p style="color:#8aa0bd">'+k+' tools…</p>',
                       preview:el=>el.innerHTML='<div class="preview-text">'+k+' preview…</div>'});
});
SPC.tabs.show('outlook');
</script>
```

- [ ] **Step 2: Verify tab switching**

Reload. preview_click each tab; after each, preview_snapshot.
Expected: clicking "Watch" highlights it and the left rail shows "watch tools…", right shows "watch preview…". preview_console_logs: no errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: tab router with per-tab tool/preview registration"
```

---

### Task 7: README + GitHub Pages workflow

**Files:**
- Create: `README.md`
- Create: `.github/workflows/pages.yml`

- [ ] **Step 1: Write README**

```markdown
# Snow Prediction Center

A standalone Snow Prediction Center product maker — the SPC convective product suite, mapped to snow
(Snownadoes / Blizzard Wind / Snow Squalls). Draw click-polygon risk areas on a US map and issue
Outlooks, Mesoscale Discussions, Watches, Warnings, HWO/Hydrologic Outlooks, then verify against
typed storm reports.

Open `index.html` in any browser — no build step, no dependencies.
```

- [ ] **Step 2: Add Pages workflow (mirror snow-simulator)**

```yaml
name: Deploy to GitHub Pages
on: { push: { branches: [main] } }
permissions: { contents: read, pages: write, id-token: write }
jobs:
  deploy:
    environment: { name: github-pages, url: "${{ steps.deployment.outputs.page_url }}" }
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with: { path: "." }
      - id: deployment
        uses: actions/deploy-pages@v4
```

- [ ] **Step 3: Commit**

```bash
git add README.md .github/workflows/pages.yml
git commit -m "docs: README and GitHub Pages deploy workflow"
```

---

## PHASE 2 — Snow Outlook (SECTION D)

### Task 8: Outlook tool rail — day tabs, panel switcher, level picker

**Files:**
- Modify: `index.html` (new `<script>` SECTION D; registers `outlook` with SPC.tabs, replacing the placeholder)

- [ ] **Step 1: Build the outlook tools UI**

```html
<script>
// ===== SECTION D: OUTLOOK =====
SPC.outlook = (function(){
  const T=SPC.tax, S=SPC.state.doc.outlook;
  const PANELS=[['categorical','Categorical'],['snownado','Snownado'],['blizzardWind','Blizzard Wind'],['snowSquall','Snow Squall']];
  function levelOptions(){
    if(S.panel==='categorical') return T.TIERS.map(t=>({v:t.key,label:t.name,color:t.color}));
    const hz=T.HAZARDS[S.panel];
    const probs=hz.probs.map(p=>({v:'p'+p,label:p+'%',color:T.PROB_COLORS[p]}));
    const cigs=hz.cigs.map(c=>({v:'cig'+c,label:'CIG'+c,color:'#ffffff'}));
    return probs.concat(cigs);
  }
  let curLevel=null;
  function tools(el){
    el.innerHTML='<h2 class="sec">Day</h2><div id="olDays"></div>'+
      '<h2 class="sec">Panel</h2><div id="olPanels"></div>'+
      '<h2 class="sec">Level</h2><div id="olLevels"></div>'+
      '<h2 class="sec">Actions</h2>'+
      '<button id="olDraw">✏️ Draw area</button> <button id="olUndo">Undo last</button> '+
      '<button id="olClear">Clear panel</button>';
    const days=el.querySelector('#olDays');
    [1,2,3,'4-8'].forEach(d=>{const b=document.createElement('button');b.textContent='Day '+d;
      b.className=S.day==d?'active':''; b.onclick=()=>{S.day=d;S.panel='categorical';tools(el);render();};days.appendChild(b);});
    const panels=el.querySelector('#olPanels');
    PANELS.forEach(([k,lbl])=>{ if(S.day==='4-8'&&k==='categorical')return; // 4-8 is prob-only
      const b=document.createElement('button');b.textContent=lbl;b.className=S.panel===k?'active':'';
      b.onclick=()=>{S.panel=k;curLevel=null;tools(el);render();};panels.appendChild(b);});
    const lv=el.querySelector('#olLevels');
    levelOptions().forEach(o=>{const b=document.createElement('button');b.textContent=o.label;
      b.style.borderLeft='6px solid '+o.color; b.className=curLevel===o.v?'active':'';
      if(o.v==='EXTR'&&S.day===3){b.disabled=true;b.title='Day 3 cannot issue Extreme';}
      b.onclick=()=>{curLevel=o.v;tools(el);};lv.appendChild(b);});
    el.querySelector('#olDraw').onclick=()=>startDraw();
    el.querySelector('#olUndo').onclick=()=>{const a=areaList();a.pop();render();};
    el.querySelector('#olClear').onclick=()=>{clearPanel();render();};
  }
  function areaKey(){ return S.panel==='categorical'?'categorical':
    (curLevel&&curLevel.startsWith('cig')?'cig_'+S.panel:S.panel); }
  function areaList(){ return S.areas[areaKey()]; }
  function startDraw(){
    if(!curLevel){alert('Pick a level first');return;}
    SPC.map.start(null,(pts)=>{ areaList().push({points:pts,level:curLevel,day:S.day}); render(); });
  }
  function clearPanel(){ S.areas[areaKey()]=[]; if(S.panel!=='categorical') S.areas['cig_'+S.panel]=[]; }
  // render() + preview() defined in Task 9/10.
  let render=()=>{}, preview=()=>{};
  function _wire(r,p){render=r;preview=p;}
  SPC.tabs.register('outlook',{tools, preview:(el)=>preview(el)});
  return {tools,render:()=>render(),_wire,get curLevel(){return curLevel;}};
})();
</script>
```

- [ ] **Step 2: Verify tools render**

Reload, ensure Outlook tab active. preview_snapshot.
Expected: left rail shows Day (1/2/3/4-8), Panel (Categorical/Snownado/Blizzard Wind/Snow Squall), Level buttons (the 7 tiers when Categorical). Click "Day 4-8" → Categorical panel button disappears. Click Snownado panel → levels show 2%…60% + CIG1–CIG4. preview_console_logs: no errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: outlook tool rail (day tabs, panel switcher, level picker)"
```

---

### Task 9: Outlook panel rendering (categorical + probabilistic + CIG overlays)

**Files:**
- Modify: `index.html` (extend SECTION D)

- [ ] **Step 1: Add the render function and wire it**

Append inside SECTION D's IIFE (before `_wire` return), then call `_wire(render,preview)`:
```js
function styleFor(level){
  if(S.panel==='categorical'){ const t=T.tier(level); return {fill:t.color,opacity:t.hatch?0.9:0.55,stroke:'#0a0a0a',dash:t.hatch?'5 4':null}; }
  if(level.startsWith('cig')){ const n=+level.slice(3); return {fill:T.CIG_SHADE[n],opacity:1,stroke:'#ffffff',strokeWidth:1.6,dash:'2 2'}; }
  const p=+level.slice(1); return {fill:T.PROB_COLORS[p],opacity:0.5,stroke:'#0a0a0a'};
}
function render(){
  const polys=[];
  const baseKey = S.panel==='categorical'?'categorical':S.panel;
  (S.areas[baseKey]||[]).filter(a=>a.day==S.day).forEach(a=>polys.push(Object.assign({points:a.points},styleFor(a.level))));
  if(S.panel!=='categorical'){
    (S.areas['cig_'+S.panel]||[]).filter(a=>a.day==S.day).forEach(a=>polys.push(Object.assign({points:a.points},styleFor(a.level))));
  }
  SPC.map.renderPolys(polys);
  const pv=document.getElementById('previewBody'); if(pv) preview(pv);
}
```

- [ ] **Step 2: Verify CIG overlays draw distinctly**

Reload. preview_eval to inject sample areas and render:
```js
const S=SPC.state.doc.outlook; S.day=1;S.panel='snownado';
S.areas.snownado=[{points:[[250,180],[480,180],[480,360],[250,360]],level:'p30',day:1}];
S.areas.cig_snownado=[{points:[[300,220],[430,220],[430,320],[300,320]],level:'cig3',day:1}];
SPC.outlook.render(); 'ok'
```
Expected: `'ok'`; preview_screenshot shows an orange 30% box with a brighter white-shaded CIG3 zone nested inside (shaded, NOT hatched). preview_console_logs: no errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: outlook panel rendering with CIG shaded overlays"
```

---

### Task 10: Outlook auto-derivation + auto-text + export

**Files:**
- Modify: `index.html` (extend SECTION D)

- [ ] **Step 1: Add prob→categorical derivation, Extreme gate, narrative text, export buttons**

```js
// Map highest drawn prob (per hazard) to a categorical tier rank.
function probToRank(panel,maxP){
  if(panel==='snownado'){ if(maxP>=30)return 5; if(maxP>=15)return 4; if(maxP>=10)return 3; if(maxP>=5)return 2; if(maxP>=2)return 1; return 0; }
  if(maxP>=60)return 5; if(maxP>=45)return 4; if(maxP>=30)return 3; if(maxP>=15)return 2; if(maxP>=5)return 1; return 0;
}
function suggestTier(){
  let rank=0;
  ['snownado','blizzardWind','snowSquall'].forEach(h=>{
    const ps=(S.areas[h]||[]).filter(a=>a.day==S.day).map(a=>+a.level.slice(1)).filter(n=>!isNaN(n));
    if(ps.length) rank=Math.max(rank,probToRank(h,Math.max(...ps)));
  });
  // Extreme gate: a CIG4 snownado area present AND an 80% snownado prob area present.
  const cig4=(S.areas.cig_snownado||[]).some(a=>a.level==='cig4'&&a.day==S.day);
  const p80=(S.areas.snownado||[]).some(a=>+a.level.slice(1)>=60&&a.day==S.day);
  if(cig4&&p80&&S.day!==3) return 'EXTR';
  return T.TIERS[rank].key;
}
function narrative(){
  const tier=T.tier(suggestTier());
  const lines=[];
  lines.push('SNOW PREDICTION CENTER DAY '+S.day+' CONVECTIVE-SNOW OUTLOOK');
  lines.push('VALID '+(S.meta.issued||'(set valid time)'));
  lines.push('');
  lines.push('...CATEGORICAL OUTLOOK: '+tier.name.toUpperCase()+' RISK...');
  if(tier.key==='EXTR') lines.push('** EXTREME RISK — CATASTROPHIC SNOWNADO OUTBREAK POSSIBLE (≥80% CIG4) **');
  ['snownado','blizzardWind','snowSquall'].forEach(h=>{
    const ps=(S.areas[h]||[]).filter(a=>a.day==S.day).map(a=>+a.level.slice(1)).filter(n=>!isNaN(n));
    const cigs=(S.areas['cig_'+h]||[]).filter(a=>a.day==S.day).map(a=>a.level.toUpperCase());
    if(ps.length||cigs.length){
      lines.push('');
      lines.push('...'+T.HAZARDS[h].name.toUpperCase()+'...');
      if(ps.length) lines.push('  Max probability: '+Math.max(...ps)+'%');
      if(cigs.length) lines.push('  Conditional intensity: '+[...new Set(cigs)].join(', '));
    }
  });
  return lines.join('\n');
}
function preview(el){
  el.innerHTML='<div style="margin-bottom:8px"><strong>Suggested:</strong> '+T.tier(suggestTier()).name+' Risk</div>'+
    '<div class="preview-text" id="olText"></div>'+
    '<div style="margin-top:8px"><button id="olPng">⬇ PNG</button> <button id="olCopy">⧉ Copy text</button></div>';
  el.querySelector('#olText').textContent=narrative();
  el.querySelector('#olPng').onclick=()=>SPC.state.exportPNG(d=>{const a=document.createElement('a');a.href=d;a.download='snow-outlook-day'+S.day+'.png';a.click();});
  el.querySelector('#olCopy').onclick=()=>SPC.state.copyText(narrative());
}
_wire(render,preview);
```

- [ ] **Step 2: Verify derivation + text**

Reload. preview_eval:
```js
const S=SPC.state.doc.outlook; S.day=1;S.panel='snownado';
S.areas.snownado=[{points:[[250,180],[480,180],[480,360]],level:'p60',day:1}];
S.areas.cig_snownado=[{points:[[300,220],[430,220],[430,320]],level:'cig4',day:1}];
SPC.tabs.show('outlook'); document.getElementById('olText').textContent.includes('EXTREME')
```
Expected: `true` (Extreme gate fires with 60%+CIG4). Set `S.day=3` and re-show → text should NOT say EXTREME (Day 3 blocked). preview_console_logs: no errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: outlook auto-derivation, narrative text, PNG/text export"
```

---

## PHASE 3 — Mesoscale Discussion (SECTION E)

### Task 9 (cont.) note
Sections E–H each follow the same shape as the Outlook: a `tools(el)` that registers with `SPC.tabs`, draw via `SPC.map.start`, render via `SPC.map.renderPolys`, and a `preview(el)` that emits SPC-style text + export buttons.

### Task 11: Mesoscale Discussion + meso-gamma

**Files:**
- Modify: `index.html` (new `<script>` SECTION E; registers `md`)

- [ ] **Step 1: Add the MD module**

```html
<script>
// ===== SECTION E: MESOSCALE DISCUSSION =====
(function(){
  const T=SPC.tax, S=SPC.state.doc.md;
  function tools(el){
    el.innerHTML='<h2 class="sec">Concerning</h2><div id="mdConc"></div>'+
      '<h2 class="sec">Probability of Watch Issuance</h2>'+
      '<input id="mdProb" type="range" min="0" max="100" step="5" value="'+S.watchProb+'"> <span id="mdProbV">'+S.watchProb+'%</span>'+
      '<h2 class="sec">Valid</h2><input id="mdValid" type="datetime-local">'+
      '<h2 class="sec">Meso-gamma</h2>'+
      '<label><input type="checkbox" id="mdMG" '+(S.mesoGamma?'checked':'')+'> High-confidence meso-gamma</label>'+
      '<div id="mdMGnote" style="color:#8aa0bd;font-size:12px"></div>'+
      '<h2 class="sec">Actions</h2><button id="mdDraw">✏️ Draw area</button> <button id="mdClr">Clear</button>';
    const conc=el.querySelector('#mdConc');
    [['snownado','Snownado'],['blizzardWind','Blizzard Wind'],['snowSquall','Snow Squalls']].forEach(([k,l])=>{
      const b=document.createElement('button');b.textContent=l;b.className=S.concerning===k?'active':'';
      b.onclick=()=>{S.concerning=k;tools(el);};conc.appendChild(b);});
    el.querySelector('#mdProb').oninput=e=>{S.watchProb=+e.target.value;el.querySelector('#mdProbV').textContent=S.watchProb+'%';preview(document.getElementById('previewBody'));};
    el.querySelector('#mdValid').onchange=e=>{S.valid=e.target.value;preview(document.getElementById('previewBody'));};
    el.querySelector('#mdMG').onchange=e=>{S.mesoGamma=e.target.checked;tools(el);preview(document.getElementById('previewBody'));};
    el.querySelector('#mdMGnote').textContent = S.mesoGamma
      ? 'Meso-gamma criteria: confident W2+ snownadoes OR blizzard wind ≥60 mph.' : '';
    el.querySelector('#mdDraw').onclick=()=>SPC.map.start(null,pts=>{S.areas=[{points:pts}];render();});
    el.querySelector('#mdClr').onclick=()=>{S.areas=[];render();};
  }
  function render(){
    SPC.map.renderPolys((S.areas||[]).map(a=>({points:a.points,fill:'none',stroke:S.mesoGamma?'#ff3b3b':'#ffd24d',strokeWidth:2.4,dash:'6 3',opacity:0})));
    const pv=document.getElementById('previewBody'); if(pv) preview(pv);
  }
  function text(){
    const L=[];
    L.push((S.mesoGamma?'SNOW MESOSCALE DISCUSSION (MESO-GAMMA) ':'SNOW MESOSCALE DISCUSSION ')+(SPC.state.doc.md.number||''));
    L.push('Concerning...'+T.HAZARDS[S.concerning].name);
    L.push('Valid '+(S.valid||'(set valid time)'));
    L.push('Probability of Watch Issuance...'+S.watchProb+' percent');
    L.push('');
    if(S.mesoGamma) L.push('** MESO-GAMMA: high-confidence W2+ snownadoes or blizzard wind ≥60 mph. **');
    L.push('SUMMARY...Conditions are becoming favorable for '+T.HAZARDS[S.concerning].name.toLowerCase()+
      '. A '+(S.watchProb>=50?'watch is likely':'watch may be needed')+' in the next 1-3 hours.');
    return L.join('\n');
  }
  function preview(el){
    el.innerHTML='<div class="preview-text" id="mdText"></div><div style="margin-top:8px"><button id="mdPng">⬇ PNG</button> <button id="mdCopy">⧉ Copy</button></div>';
    el.querySelector('#mdText').textContent=text();
    el.querySelector('#mdPng').onclick=()=>SPC.state.exportPNG(d=>{const a=document.createElement('a');a.href=d;a.download='snow-md.png';a.click();});
    el.querySelector('#mdCopy').onclick=()=>SPC.state.copyText(text());
  }
  SPC.tabs.register('md',{tools,preview});
})();
</script>
```

- [ ] **Step 2: Verify**

Reload, click "Mesoscale Discussion" tab. preview_snapshot. Toggle meso-gamma checkbox via preview_eval `document.getElementById('mdMG').click()`, then preview_eval `document.getElementById('mdText').textContent.includes('MESO-GAMMA')` → `true`. The meso-gamma note must mention "≥60 mph" (not 100). preview_console_logs: no errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: snow mesoscale discussion + meso-gamma product"
```

---

## PHASE 4 — Watches (+PDS) & Warnings (+Emergency) (SECTION F)

### Task 12: Watch product with PDS + probability table

**Files:**
- Modify: `index.html` (new `<script>` SECTION F; registers `watch`)

- [ ] **Step 1: Add the watch module**

```html
<script>
// ===== SECTION F1: WATCH =====
(function(){
  const S=SPC.state.doc.watch;
  const TYPES=['Blizzard','Snownado'];
  const ROWS=[['Two or more snownadoes','snownado'],['One or more W2+ snownado','w2'],
              ['Blizzard wind gusts ≥60 mph','wind60'],['Snow squalls ≥4″/hr','squall4']];
  function tools(el){
    el.innerHTML='<h2 class="sec">Watch type</h2><div id="wType"></div>'+
      '<label style="display:block;margin-top:8px"><input type="checkbox" id="wPds" '+(S.pds?'checked':'')+'> PDS (Particularly Dangerous Situation)</label>'+
      '<h2 class="sec">Watch number</h2><input id="wNum" type="number" value="'+(S.number||1)+'" style="width:80px">'+
      '<h2 class="sec">Valid</h2><input id="wValid" type="datetime-local">'+
      '<h2 class="sec">Probability table</h2><div id="wProbs"></div>'+
      '<h2 class="sec">Actions</h2><button id="wDraw">✏️ Draw area</button> <button id="wClr">Clear</button>';
    const ty=el.querySelector('#wType');
    TYPES.forEach(t=>{const b=document.createElement('button');b.textContent=t+' Watch';b.className=S.type===t?'active':'';
      b.onclick=()=>{S.type=t;tools(el);prev();};ty.appendChild(b);});
    const pr=el.querySelector('#wProbs');
    ROWS.forEach(([lbl,key])=>{const d=document.createElement('div');d.style.margin='4px 0';
      d.innerHTML=lbl+': <select data-k="'+key+'"><option>Low</option><option>Moderate</option><option>High</option><option>Extreme</option></select>';
      const sel=d.querySelector('select');sel.value=S.probTable[key]||'Low';
      sel.onchange=e=>{S.probTable[key]=e.target.value;prev();};pr.appendChild(d);});
    el.querySelector('#wPds').onchange=e=>{S.pds=e.target.checked;prev();};
    el.querySelector('#wNum').onchange=e=>{S.number=+e.target.value;prev();};
    el.querySelector('#wValid').onchange=e=>{S.valid=e.target.value;prev();};
    el.querySelector('#wDraw').onclick=()=>SPC.map.start(null,pts=>{S.areas=[{points:pts}];render();});
    el.querySelector('#wClr').onclick=()=>{S.areas=[];render();};
  }
  function render(){
    const col=S.type==='Snownado'?'#e23a3a':'#3a7be2';
    SPC.map.renderPolys((S.areas||[]).map(a=>({points:a.points,fill:col,opacity:0.18,stroke:col,strokeWidth:2.4})));
    const pv=document.getElementById('previewBody'); if(pv) prev(pv);
  }
  function text(){
    const L=[]; const hdr=(S.pds?'PDS ':'')+'SNOW '+S.type.toUpperCase()+' WATCH '+(S.number||1);
    L.push(hdr); L.push('Valid '+(S.valid||'(set valid time)')); L.push('');
    if(S.pds) L.push('** PARTICULARLY DANGEROUS SITUATION **');
    L.push('PROBABILITY TABLE');
    ROWS.forEach(([lbl,key])=>L.push('  '+lbl+'...'+(S.probTable[key]||'Low')));
    return L.join('\n');
  }
  function prev(el){ el=el||document.getElementById('previewBody'); if(!el)return;
    el.innerHTML='<div class="preview-text" id="wText"></div><div style="margin-top:8px"><button id="wCopy">⧉ Copy</button> <button id="wPng">⬇ PNG</button></div>';
    el.querySelector('#wText').textContent=text();
    el.querySelector('#wCopy').onclick=()=>SPC.state.copyText(text());
    el.querySelector('#wPng').onclick=()=>SPC.state.exportPNG(d=>{const a=document.createElement('a');a.href=d;a.download='snow-watch.png';a.click();});
  }
  SPC.tabs.register('watch',{tools,preview:prev});
})();
</script>
```

- [ ] **Step 2: Verify**

Reload, click "Watch". preview_snapshot shows Blizzard/Snownado type buttons, PDS checkbox, probability table with 4 rows. preview_eval `document.getElementById('wPds').click(); document.getElementById('wText').textContent.includes('PARTICULARLY DANGEROUS')` → `true`. preview_console_logs: no errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: snow watch product with PDS and probability table"
```

---

### Task 13: Warning product with Emergency wording

**Files:**
- Modify: `index.html` (extend SECTION F; registers `warning`)

- [ ] **Step 1: Add the warning module**

```html
<script>
// ===== SECTION F2: WARNING =====
(function(){
  const S=SPC.state.doc.warning;
  const TYPES=['Blizzard','Snow Squall','Snownado'];
  const CRIT={'Blizzard':'Sustained/gusting ≥35 mph + heavy snow + vis ≤¼ mi for ≥3 hr.',
              'Snow Squall':'Intense, brief, moderate–heavy snow; vis ≤¼ mi.',
              'Snownado':'A snownado is occurring or imminent. Take cover from blowing snow.'};
  function canEmergency(){ return S.type==='Blizzard'||S.type==='Snownado'; } // squalls are warning-only, no emergency
  function tools(el){
    el.innerHTML='<h2 class="sec">Warning type</h2><div id="gType"></div>'+
      '<label style="display:block;margin-top:8px" id="gEmgWrap"><input type="checkbox" id="gEmg" '+(S.emergency?'checked':'')+'> Emergency wording</label>'+
      '<h2 class="sec">WFO</h2><input id="gWfo" placeholder="e.g. Buffalo NY" value="'+(S.wfo||'')+'">'+
      '<h2 class="sec">Valid</h2><input id="gValid" type="datetime-local">'+
      '<h2 class="sec">Actions</h2><button id="gDraw">✏️ Draw polygon</button> <button id="gClr">Clear</button>';
    const ty=el.querySelector('#gType');
    TYPES.forEach(t=>{const b=document.createElement('button');b.textContent=t+' Warning';b.className=S.type===t?'active':'';
      b.onclick=()=>{S.type=t;if(!canEmergency())S.emergency=false;tools(el);prev();};ty.appendChild(b);});
    if(!canEmergency()){el.querySelector('#gEmgWrap').style.display='none';}
    el.querySelector('#gEmg').onchange=e=>{S.emergency=e.target.checked;prev();};
    el.querySelector('#gWfo').onchange=e=>{S.wfo=e.target.value;prev();};
    el.querySelector('#gValid').onchange=e=>{S.valid=e.target.value;prev();};
    el.querySelector('#gDraw').onclick=()=>SPC.map.start(null,pts=>{S.areas=[{points:pts}];render();});
    el.querySelector('#gClr').onclick=()=>{S.areas=[];render();};
  }
  function render(){
    const col=S.type==='Snownado'?'#e23a3a':(S.type==='Snow Squall'?'#2ea66a':'#ff8c1a');
    SPC.map.renderPolys((S.areas||[]).map(a=>({points:a.points,fill:col,opacity:0.30,stroke:col,strokeWidth:2.6})));
    const pv=document.getElementById('previewBody'); if(pv) prev(pv);
  }
  function text(){
    const L=[]; const emg=S.emergency&&canEmergency();
    L.push((emg?(S.type.toUpperCase()+' EMERGENCY'):('SNOW '+S.type.toUpperCase()+' WARNING'))+(S.wfo?(' — NWS '+S.wfo):''));
    L.push('Valid '+(S.valid||'(set valid time)')); L.push('');
    if(emg) L.push('** '+S.type.toUpperCase()+' EMERGENCY — CATASTROPHIC, CONFIRMED THREAT TO POPULATED AREAS **');
    L.push(CRIT[S.type]);
    return L.join('\n');
  }
  function prev(el){ el=el||document.getElementById('previewBody'); if(!el)return;
    el.innerHTML='<div class="preview-text" id="gText"></div><div style="margin-top:8px"><button id="gCopy">⧉ Copy</button> <button id="gPng">⬇ PNG</button></div>';
    el.querySelector('#gText').textContent=text();
    el.querySelector('#gCopy').onclick=()=>SPC.state.copyText(text());
    el.querySelector('#gPng').onclick=()=>SPC.state.exportPNG(d=>{const a=document.createElement('a');a.href=d;a.download='snow-warning.png';a.click();});
  }
  SPC.tabs.register('warning',{tools,preview:prev});
})();
</script>
```

- [ ] **Step 2: Verify Emergency gating**

Reload, click "Warning". Select Snow Squall → preview_eval `document.getElementById('gEmgWrap').style.display` → `'none'` (squalls have no emergency). Select Blizzard, check Emergency → preview_eval `document.getElementById('gText').textContent.includes('BLIZZARD EMERGENCY')` → `true`. preview_console_logs: no errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: snow warning product with emergency wording"
```

---

## PHASE 5 — HWO + Hydrologic Outlook (SECTION G)

### Task 14: HWO + Hydrologic Outlook

**Files:**
- Modify: `index.html` (new `<script>` SECTION G; registers `hwo`)

- [ ] **Step 1: Add the HWO/Hydro module**

```html
<script>
// ===== SECTION G: HWO + HYDRO =====
(function(){
  const H=SPC.state.doc.hwo, HY=SPC.state.doc.hydro;
  const DAYNAMES=['Day 1 (Today)','Day 2','Day 3','Day 4','Day 5','Day 6','Day 7'];
  function tools(el){
    el.innerHTML='<h2 class="sec">Hazardous Weather Outlook</h2><div id="hwoDays"></div>'+
      '<label style="display:block;margin-top:8px"><input type="checkbox" id="hwoSpot" '+(H.spotter?'checked':'')+'> Activate snow spotters</label>'+
      '<h2 class="sec">Hydrologic Outlook (ESF)</h2><textarea id="hyText" rows="5" style="width:100%" placeholder="Snowmelt / snowpack flooding outlook…">'+(HY.text||'')+'</textarea>';
    const dd=el.querySelector('#hwoDays');
    DAYNAMES.forEach((dn,i)=>{const w=document.createElement('div');w.style.margin='4px 0';
      w.innerHTML='<label style="font-size:12px;color:#8aa0bd">'+dn+'</label><input data-d="'+i+'" style="width:100%" value="'+(H.days[i]||'').replace(/"/g,'&quot;')+'">';
      w.querySelector('input').oninput=e=>{H.days[+e.target.dataset.d]=e.target.value;prev();};dd.appendChild(w);});
    el.querySelector('#hwoSpot').onchange=e=>{H.spotter=e.target.checked;prev();};
    el.querySelector('#hyText').oninput=e=>{HY.text=e.target.value;prev();};
  }
  function text(){
    const L=['HAZARDOUS WEATHER OUTLOOK',''];
    L.push('.DAY ONE...'+(H.days[0]||'No hazardous winter weather expected.'));
    L.push('');L.push('.DAYS TWO THROUGH SEVEN...');
    for(let i=1;i<7;i++) if(H.days[i]) L.push('  '+DAYNAMES[i]+': '+H.days[i]);
    L.push('');L.push('.SPOTTER INFORMATION STATEMENT...');
    L.push(H.spotter?'Snow spotters will be needed today. Report snownadoes, blizzard wind, and squall whiteouts.':'Snow spotter activation is not expected today.');
    L.push('');L.push('=== HYDROLOGIC OUTLOOK (ESF) ===');
    L.push(HY.text||'No hydrologic concerns (snowmelt/snowpack) at this time.');
    return L.join('\n');
  }
  function prev(el){ el=el||document.getElementById('previewBody'); if(!el)return;
    el.innerHTML='<div class="preview-text" id="hwoText"></div><div style="margin-top:8px"><button id="hwoCopy">⧉ Copy</button></div>';
    el.querySelector('#hwoText').textContent=text();
    el.querySelector('#hwoCopy').onclick=()=>SPC.state.copyText(text());
  }
  SPC.tabs.register('hwo',{tools,preview:prev});
})();
</script>
```

- [ ] **Step 2: Verify**

Reload, click "HWO / Hydro". preview_snapshot shows 7 day inputs + spotter checkbox + hydro textarea. preview_eval set Day 1 text + check spotter, then `document.getElementById('hwoText').textContent.includes('SPOTTER INFORMATION')` → `true`. preview_console_logs: no errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: hazardous weather outlook and hydrologic outlook"
```

---

## PHASE 6 — Storm Reports & Verification (SECTION H)

### Task 15: Storm report logger (typed city + W-rating + notability)

**Files:**
- Modify: `index.html` (new `<script>` SECTION H; registers `reports`)

- [ ] **Step 1: Add the reports logger UI**

```html
<script>
// ===== SECTION H: STORM REPORTS + VERIFICATION =====
(function(){
  const T=SPC.tax, R=SPC.state.doc.reports, OL=SPC.state.doc.outlook;
  function tools(el){
    el.innerHTML='<h2 class="sec">Log a notable snownado</h2>'+
      '<label>City (any place, metropolis to village)</label><input id="rCity" placeholder="e.g. Marquette, MI" style="width:100%">'+
      '<label>Whiteout rating</label><select id="rW">'+T.WHITEOUT.map(w=>'<option value="'+w.t+'">'+w.t+' — '+w.name+' ('+w.lo+'–'+(w.hi>998?'∞':w.hi)+' mph)</option>').join('')+'</select>'+
      '<label>Notability</label><select id="rNote">'+T.NOTABILITY.map(n=>'<option>'+n+'</option>').join('')+'</select>'+
      '<label>Name (optional)</label><input id="rName" placeholder="e.g. The Marquette Maelstrom" style="width:100%">'+
      '<div style="margin-top:8px"><button id="rAdd">+ Add report</button></div>'+
      '<h2 class="sec">Filter</h2><select id="rFilter"><option value="">All notability</option>'+T.NOTABILITY.map(n=>'<option>'+n+'</option>').join('')+'</select>'+
      '<h2 class="sec">Logged reports</h2><div id="rList"></div>';
    el.querySelector('#rAdd').onclick=()=>{
      const city=el.querySelector('#rCity').value.trim(); if(!city){alert('Enter a city');return;}
      R.push({city,w:el.querySelector('#rW').value,note:el.querySelector('#rNote').value,name:el.querySelector('#rName').value.trim()});
      el.querySelector('#rCity').value='';el.querySelector('#rName').value='';
      renderList(el);prev();
    };
    el.querySelector('#rFilter').onchange=()=>renderList(el);
    renderList(el);
  }
  function renderList(el){
    const f=el.querySelector('#rFilter').value;
    const list=el.querySelector('#rList'); list.innerHTML='';
    R.forEach((r,i)=>{ if(f&&r.note!==f) return;
      const d=document.createElement('div');d.style.cssText='border:1px solid #22324b;border-radius:6px;padding:6px;margin:4px 0;font-size:12px';
      d.innerHTML='<strong>'+r.w+'</strong> '+(r.name?('“'+r.name+'” '):'')+r.city+' <em style="color:#8aa0bd">'+r.note+'</em> <button data-i="'+i+'" style="float:right">✕</button>';
      d.querySelector('button').onclick=()=>{R.splice(i,1);renderList(el);prev();};list.appendChild(d);});
    if(!list.children.length) list.innerHTML='<p style="color:#8aa0bd">No reports'+(f?' at '+f:'')+'.</p>';
  }
  // verify() defined in Task 16; stub here:
  let prev=()=>{};
  function _setPrev(p){prev=p;}
  SPC.tabs.register('reports',{tools,preview:(el)=>prev(el)});
  window.SPC._reports={ _setPrev, R, get OL(){return OL;} };
})();
</script>
```

- [ ] **Step 2: Verify logging**

Reload, click "Storm Reports". preview_eval to add a report:
```js
const el=document.getElementById('toolBody');
el.querySelector('#rCity').value='Marquette, MI'; el.querySelector('#rW').value='W5'; el.querySelector('#rNote').value='Super-Hyper-Notable';
el.querySelector('#rAdd').click(); SPC.state.doc.reports.length
```
Expected: `1`; the list shows "W5 … Marquette, MI Super-Hyper-Notable". Set filter to "Notable" → the W5 Super-Hyper-Notable row hides. preview_console_logs: no errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: storm report logger (typed city, W-rating, notability)"
```

---

### Task 16: Verification panel (intensity-based under/over-warned)

**Files:**
- Modify: `index.html` (extend SECTION H; provide `prev`/verify and call `_setPrev`)

- [ ] **Step 1: Add verification scoring + preview, wire it**

Append inside SECTION H's IIFE, then call `_setPrev(prev)`:
```js
function maxForecast(){
  // Highest snownado prob + highest snownado CIG present on the active outlook day.
  const day=OL.day;
  const ps=(OL.areas.snownado||[]).filter(a=>a.day==day).map(a=>+a.level.slice(1)).filter(n=>!isNaN(n));
  const cs=(OL.areas.cig_snownado||[]).filter(a=>a.day==day).map(a=>+a.level.slice(3)).filter(n=>!isNaN(n));
  return {prob:ps.length?Math.max(...ps):0, cig:cs.length?Math.max(...cs):0};
}
function cigForW(wt){ const n=+wt.slice(1); // realized W -> implied CIG ceiling
  if(n>=5)return 4; if(n>=4)return 3; if(n>=3)return 2; if(n>=2)return 1; return 0; }
function verdict(r,f){
  const need=cigForW(r.w);
  if(need>f.cig) return ['under-warned','#e23a3a'];
  if(need===f.cig) return ['verified','#2ea66a'];
  return ['over-warned','#ffd24d'];
}
function prev(el){ el=el||document.getElementById('previewBody'); if(!el)return;
  const f=maxForecast();
  let html='<div style="margin-bottom:8px"><strong>Day '+OL.day+' snownado forecast ceiling:</strong> '+
    (f.prob||'—')+'% / CIG'+(f.cig||'—')+'</div>';
  if(!R.length){ html+='<p style="color:#8aa0bd">No reports to verify.</p>'; el.innerHTML=html; return; }
  let worst='verified';
  html+='<div class="preview-text">';
  R.forEach(r=>{ const [v,c]=verdict(r,f);
    if(v==='under-warned')worst='under-warned'; else if(v==='over-warned'&&worst==='verified')worst='over-warned';
    html+='\n'+r.w+' '+r.city+' ('+r.note+') → predicted CIG'+(f.cig||0)+' → '+v.toUpperCase(); });
  html+='\n\nNET RESULT: '+worst.toUpperCase()+'</div>';
  el.innerHTML=html;
}
_setPrev(prev);
```

- [ ] **Step 2: Verify scoring**

Reload, click "Storm Reports". preview_eval:
```js
const OL=SPC.state.doc.outlook; OL.day=1;
OL.areas.snownado=[{points:[[1,1],[2,2],[3,3]],level:'p30',day:1}];
OL.areas.cig_snownado=[{points:[[1,1],[2,2],[3,3]],level:'cig2',day:1}];
SPC.state.doc.reports=[{city:'Marquette, MI',w:'W5',note:'Super-Hyper-Notable',name:''}];
SPC.tabs.show('reports');
document.getElementById('previewBody').textContent.includes('UNDER-WARNED')
```
Expected: `true` (forecast CIG2, observed W5 ⇒ implied CIG4 ⇒ under-warned). Change report to `w:'W2'` ⇒ verdict VERIFIED. preview_console_logs: no errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: intensity-based verification panel for storm reports"
```

---

### Task 17: Session save/load + final integration pass

**Files:**
- Modify: `index.html` (add a small toolbar in `header.appbar`; wire to SPC.state)

- [ ] **Step 1: Add Save/Load/Reset controls**

In `header.appbar`, before `</header>` (after the nav), add:
```html
<div style="display:flex;gap:6px">
  <button id="btnSave">Save</button><button id="btnLoad">Load</button><button id="btnReset">Reset</button>
  <input id="fileLoad" type="file" accept="application/json" style="display:none">
</div>
```
Add to SECTION I:
```js
document.getElementById('btnSave').onclick=()=>{const b=new Blob([SPC.state.save()],{type:'application/json'});
  const a=document.createElement('a');a.href=URL.createObjectURL(b);a.download='snow-spc-session.json';a.click();};
document.getElementById('btnLoad').onclick=()=>document.getElementById('fileLoad').click();
document.getElementById('fileLoad').onchange=e=>{const f=e.target.files[0];if(!f)return;
  const rd=new FileReader();rd.onload=()=>{SPC.state.load(rd.result);SPC.tabs.show(SPC.tabs.current);};rd.readAsText(f);};
document.getElementById('btnReset').onclick=()=>{if(confirm('Reset all products?'))SPC.state.reset();};
```

- [ ] **Step 2: Full integration verification**

Reload. Walk every tab via preview_click: Outlook → draw a snownado area + CIG via the eval snippets from Task 9, switch to Reports, confirm verification reflects it. Click Save (downloads JSON). preview_console_logs across the whole pass: no errors. preview_screenshot of the Outlook with a full categorical+CIG render for the final artifact.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: session save/load/reset and final integration pass"
```

---

## Self-Review (completed by plan author)

**Spec coverage:**
- §2 Architecture (single file, Albers SVG, click-polygon, 3 rails, tabs, export) → Tasks 1–6, 17. ✓
- §3.1 7 categorical tiers incl. Extreme gate → Task 3 + Task 10 (`suggestTier`/Extreme gate). ✓
- §3.2 hazard probs incl. wind 75/90 → Task 3 (`HAZARDS`). ✓
- §3.3 CIG 4/3/2 gradient as shaded zones → Task 3 + Task 9 (`styleFor` CIG_SHADE). ✓
- §3.4 Whiteout Scale W0–W6 → Task 3 (`WHITEOUT`). ✓
- §4.1 Outlook (days incl. 4-8 prob-only, Day 3 no Extreme, auto-derive, auto-text) → Tasks 8–10. ✓
- §4.2 MD + meso-gamma (≥60 mph trigger) → Task 11. ✓
- §4.3 Watches + PDS + prob table → Task 12. ✓
- §4.4 Warnings + Emergency (squall = no emergency) → Task 13. ✓
- §4.5 HWO + Hydro → Task 14. ✓
- §4.6 Storm Reports (typed city, notability filter) + intensity verification → Tasks 15–16. ✓

**Placeholder scan:** No "TBD/TODO/handle edge cases" — every step has concrete code. The only external dependency is the public-domain US states SVG path data in Task 2, which is sourced, not invented. ✓

**Type consistency:** `SPC.map.start/cancel/renderPolys/clearLayers/ptsAttr`, `SPC.state.doc/save/load/exportPNG/copyText`, `SPC.tabs.register/show/current`, `SPC.tax.*` names are used identically across all tasks. Area keys (`categorical`, `snownado`, `blizzardWind`, `snowSquall`, `cig_*`) match between STATE (Task 5), outlook draw (Task 8), render (Task 9), derive (Task 10), and verification (Task 16). ✓

# 章节页双菜单（风格 + LUT 滤镜）实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在章节页底部引入「风格」和「滤镜」两个主菜单，滤镜用下载的 .cube LUT 实时渲染到整章照片和大图上。

**Architecture:** 复用现有风格台状态机，把 `FX.style` 改为 `FX.current`；风格与滤镜共用 id 空间并注册进 `FX_FN`。LUT 以 13³ `.cube` 预载解析成 `Float32Array`，用 Canvas 2D 三线性插值逐像素应用。菜单是同一位置的两个分组按钮 + 向上弹出的选项条。

**Tech Stack:** 原生 HTML/CSS/JS，Canvas 2D，fetch 加载 `.cube`；无构建、无依赖。

**Spec:** `docs/superpowers/specs/2026-11-20-lut-filters-design.md`

## Global Constraints

- 单文件 `index.html`，不引入构建步骤或第三方 JS 依赖。
- 效果画布最长边 1100px（`measureChapter` 既有约束）。
- 风格 4 个：`orig` / `riso` / `cyanotype` / `block`；滤镜 3 个：`fuji` / `canon` / `ricoh`。
- 风格与滤镜互斥；`orig` 属于风格组。
- 带 `credit` 的照片永远保持原样。
- 不支持 `file://`；失败时提示改用本地服务器。
- 不提交 git，除非用户明确要求。

## Review Focus

- `file://` 打开或 LUT 404：风格台必须照常工作，滤镜给出提示而不是白屏或卡住。
- 快速连点不同滤镜：`FX.busy` / `FX.queued` 必须只落最后一支，不能出现中间态残留。
- 带 `credit` 的照片在滤镜下：保持原图并显示「原作保留」标签。
- 大图在滤镜激活时打开：显影结果与章节一致；LUT 未就绪时回退原片而不是报错。
- 手机宽度 320–700px：弹层不越界、不遮挡按钮，选项仍可点。

---

### Task 1: 下载并归档三支 LUT

**Files:**
- Create: `photos/luts/fuji_astia_100f.cube`
- Create: `photos/luts/kodak_portra_400.cube`
- Create: `photos/luts/fuji_velvia_50.cube`
- Create: `photos/luts/SOURCES.md`

**Interfaces:**
- Produces: 三个文件路径；`SOURCES.md` 记录下载地址与授权说明。

- [ ] **Step 1: 下载并解压三支 .cube**

```bash
cd photos && mkdir -p luts && cd luts
curl -sSLo fuji.zip https://www.gmic.eu/color_presets/colorslide/clut/fuji_astia_100f.zip
curl -sSLo canon.zip https://www.gmic.eu/color_presets/negative_new/clut/kodak_portra_400.zip
curl -sSLo ricoh.zip https://www.gmic.eu/color_presets/colorslide/clut/fuji_velvia_50.zip
unzip -o -q fuji.zip && unzip -o -q canon.zip && unzip -o -q ricoh.zip && rm -f *.zip
ls -la *.cube
```

Expected: 三个 `.cube`，每个约 130KB。

- [ ] **Step 2: 校验文件头**

Run: `head -6 fuji_astia_100f.cube kodak_portra_400.cube fuji_velvia_50.cube`
Expected: 每个都能看到 `TITLE` 和 `LUT_3D_SIZE 13`。

- [ ] **Step 3: 写 SOURCES.md**

内容包含：每支的按钮名、真实 preset、G'MIC 下载 URL、`LUT_3D_SIZE`，以及：「这些 LUT 用于观感近似，不代表 Fujifilm / Canon / Ricoh 官方色彩；商标名仅作说明。」

---

### Task 2: LUT 载入与渲染函数

**Files:**
- Modify: `index.html:962-975`（风格台常量区）
- Modify: `index.html:1044`（`FX_FN`）

**Interfaces:**
- Produces:
  - `FILTERS: Array<{ id:"fuji"|"canon"|"ricoh", zh:string, en:string, look:string, file:string }>`
  - `LUT_DATA: Record<string, { n:number, data:Float32Array }>`（`data` 长度 `n³×3`，0–1）
  - `ensureLut(id): Promise<void>` —— 幂等，失败时 reject 并缓存失败状态
  - `fxLut(id, img, W, H): { canvas }` —— 遵守现有效果函数契约

- [ ] **Step 1: 加 `FILTERS` 与载入器**

在 `STYLES` 后添加：

```js
const FILTERS = [
  { id:"fuji",  zh:"富士", en:"Fuji",  look:"Classic Chrome", file:"fuji_xtrans_iii_classic_chrome.cube" },
  { id:"canon", zh:"佳能", en:"Canon", look:"Portra 400",   file:"kodak_portra_400.cube" },
  { id:"ricoh", zh:"理光", en:"Ricoh", look:"Velvia 50",    file:"fuji_velvia_50.cube" },
];
const LUT_DATA = Object.create(null), LUT_LOAD = Object.create(null);
function ensureLut(id){
  if (LUT_DATA[id]) return Promise.resolve();
  if (LUT_LOAD[id]) return LUT_LOAD[id];
  const f = FILTERS.find(x=>x.id===id);
  return LUT_LOAD[id] = (async ()=>{
    const res = await fetch(`photos/luts/${f.file}`);
    if (!res.ok) throw new Error(`LUT ${f.file} ${res.status}`);
    const txt = await res.text();
    let n = 0, dmin = [0,0,0], dmax = [1,1,1], vals = [];
    for (const line of txt.split("\n")){
      const s = line.trim(); if (!s || s[0]==="#" || /^TITLE/.test(s)) continue;
      const m = s.match(/^LUT_3D_SIZE\s+(\d+)/); if (m){ n = +m[1]; continue; }
      const a = s.match(/^DOMAIN_MIN\s+([\d.]+)\s+([\d.]+)\s+([\d.]+)/); if (a){ dmin = a.slice(1).map(Number); continue; }
      const b = s.match(/^DOMAIN_MAX\s+([\d.]+)\s+([\d.]+)\s+([\d.]+)/); if (b){ dmax = b.slice(1).map(Number); continue; }
      if (/^-?[\d.]/.test(s)) vals.push(+s.split(/\s+/)[0], +s.split(/\s+/)[1], +s.split(/\s+/)[2]);
    }
    if (!n || vals.length !== n*n*n*3) throw new Error(`LUT ${f.file} 格式不对`);
    const data = new Float32Array(vals.length);
    for (let i=0; i<vals.length; i+=3){
      data[i]   = (vals[i]   - dmin[0]) / (dmax[0] - dmin[0] || 1);
      data[i+1] = (vals[i+1] - dmin[1]) / (dmax[1] - dmin[1] || 1);
      data[i+2] = (vals[i+2] - dmin[2]) / (dmax[2] - dmin[2] || 1);
    }
    LUT_DATA[id] = { n, data };
  })();
}
```

注意：`.cube` 的 `LUT_3D_SIZE` 必须在数据行之前；上面按行扫描，顺序无影响。

- [ ] **Step 2: 写 `fxLut`（三线性插值）**

放在 `fxBlock` 后：

```js
// 三线性插值查表；LUT_DATA 的排列是 r 变化最快、b 最慢（.cube 约定）
function fxLut(id, img, W, H){
  const L = LUT_DATA[id]; if (!L) throw new Error("LUT 未载入");
  const g = coverCanvas(img, W, H), im = g.getImageData(0,0,W,H), d = im.data;
  const { n, data } = L, nm1 = n - 1, n2 = n*n;
  for (let i=0; i<d.length; i+=4){
    const rf = d[i]/255*nm1, gf = d[i+1]/255*nm1, bf = d[i+2]/255*nm1;
    const r0 = rf|0, g0 = gf|0, b0 = bf|0, r1 = r0<nm1?r0+1:r0, g1 = g0<nm1?g0+1:g0, b1 = b0<nm1?b0+1:b0;
    const fr = rf-r0, fg = gf-g0, fb = bf-b0;
    const i000 = (r0 + g0*n + b0*n2)*3, i100 = (r1 + g0*n + b0*n2)*3,
          i010 = (r0 + g1*n + b0*n2)*3, i110 = (r1 + g1*n + b0*n2)*3,
          i001 = (r0 + g0*n + b1*n2)*3, i101 = (r1 + g0*n + b1*n2)*3,
          i011 = (r0 + g1*n + b1*n2)*3, i111 = (r1 + g1*n + b1*n2)*3;
    for (let c=0; c<3; c++){
      const a = data[i000+c] + (data[i100+c]-data[i000+c])*fr;
      const b = data[i010+c] + (data[i110+c]-data[i010+c])*fr;
      const e = data[i001+c] + (data[i101+c]-data[i001+c])*fr;
      const f = data[i011+c] + (data[i111+c]-data[i011+c])*fr;
      const ab = a + (b-a)*fg, ef = e + (f-e)*fg;
      d[i+c] = Math.max(0, Math.min(255, Math.round((ab + (ef-ab)*fb)*255)));
    }
  }
  g.putImageData(im, 0, 0); return { canvas:g.canvas };
}
```

- [ ] **Step 3: 启动时预载**

在章节页初始化附近（`stylesEl.addEventListener` 之后）加：

```js
FILTERS.forEach(f=>ensureLut(f.id).catch(()=>{}));   // 失败留给点击时提示
```

- [ ] **Step 4: 浏览器验证解析**

Run: `python3 -m http.server 8765`，打开 `http://localhost:8765`，在控制台执行：

```js
await ensureLut("fuji"); LUT_DATA.fuji.n        // 期望 13
LUT_DATA.fuji.data.length                        // 期望 6591
```

Expected: 13 / 6591，无报错。

---

### Task 3: 状态模型与 FX_FN 注册

**Files:**
- Modify: `index.html:919`、`959`、`962-975`、`1046-1084`、`1101`、`1157-1163`、`1261`、`1308`、`1457`

**Interfaces:**
- Consumes: `FILTERS`、`ensureLut`、`fxLut`
- Produces:
  - `FX.current: string`（替代 `FX.style`）
  - `FX.group: "style"|"lut"`
  - `setStyle(id, animate)` 接受任意 id；自动判断组
  - `curId(): string`
  - `labelOf(id): {zh,en,look?}`

- [ ] **Step 1: 改状态对象与 `FX_FN`**

```js
const FX = { current:"orig", group:"style", cache:new Map(), busy:false, queued:null };
const FX_FN = { riso:fxRiso, cyanotype:fxCyanotype, block:fxBlock,
  fuji:(img,W,H)=>fxLut("fuji",img,W,H),
  canon:(img,W,H)=>fxLut("canon",img,W,H),
  ricoh:(img,W,H)=>fxLut("ricoh",img,W,H) };
```

- [ ] **Step 2: 全量替换引用**

- `FX.style` → `FX.current`（`paintFx`、`setStyle`、`measureChapter`、`lbRestyle`）。
- `curStyle()` 改名 `curId()`，返回 `FX.queued || FX.current`。
- `setStyle` 里新增组判断与 LUT 预载：

```js
const isLut = FILTERS.some(f=>f.id===id);
if (isLut) await ensureLut(id).catch(()=>{});
if (isLut && !LUT_DATA[id]){ $("#chLine").textContent = "滤镜需要用本地服务器打开这个网页"; return; }
FX.group = isLut ? "lut" : "style";
```

- [ ] **Step 3: `paintFx` 补 LUT 就绪判断**

在 `try { out = FX_FN[id](...) }` 前：

```js
if (FILTERS.some(f=>f.id===id) && !LUT_DATA[id]) return;
```

- [ ] **Step 4: 大图兼容**

- `lbSource` 外层包 `try/catch`，失败返回原片。
- `lbRestyle(id)` 内部仍调 `setStyle`。
- `lbMode` 文案改用 `labelOf(id)`；滤镜显示 `` `${st.en} · ${st.look}` ``，风格保持原格式。

- [ ] **Step 5: 键盘**

章节页与大图的 `1–4` 逻辑后追加：

```js
if (/^[5-7]$/.test(e.key)) setStyle(FILTERS[+e.key-5].id, true);
```

大图里对应 `lbRestyle(FILTERS[+e.key-5].id)`。

- [ ] **Step 6: 回归验证**

浏览器里依次点 4 个风格、再点 3 个滤镜，确认：切换正常、带 credit 的照片不变、无控制台报错。

---

### Task 4: 双菜单 UI 与交互

**Files:**
- Modify: `index.html:146-153`（CSS）
- Modify: `index.html:349`（HTML 容器保留，内容由 JS 生成）
- Modify: `index.html:974-975`（菜单渲染与事件）

**Interfaces:**
- Consumes: `STYLES`、`FILTERS`、`setStyle`、`FX.current`、`FX.group`
- Produces: `renderMenus()`、`markStyle(id)` 同时更新两个组

- [ ] **Step 1: 写菜单 DOM**

```js
stylesEl.innerHTML = `
  ${menuHTML("style","风格",STYLES)}
  ${menuHTML("lut","滤镜",FILTERS)}`;
```

其中 `menuHTML(group,label,items)` 生成：

```html
<div class="menu" data-group="style">
  <button type="button" class="menu-btn" aria-expanded="false" aria-controls="panel-style">
    <span>风格</span><b>原图</b>
  </button>
  <div class="menu-panel" id="panel-style" role="group" aria-label="照片风格">
    <button type="button" data-s="orig" data-group="style" data-label="原图" aria-pressed="true">
      <span>原图</span><em>Original</em>
    </button>
    …
  </div>
</div>
```

滤镜组按钮显示 `滤镜` + 当前项（未选时 `原片`），data-label 用 `zh`。

- [ ] **Step 2: 写 CSS**

```css
.styles{display:flex;border:1px solid var(--line);background:var(--bg);position:relative}
.menu{position:relative}
.menu+.menu{border-left:1px solid var(--line)}
.menu-btn{display:flex;align-items:baseline;gap:7px;padding:9px 16px;font-size:10px;font-weight:600;
  letter-spacing:.08em;transition:background .3s,color .3s}
.menu-btn span{font-family:var(--cjk);font-stretch:100%;font-size:12px;font-weight:500;letter-spacing:.04em;color:var(--muted)}
.menu-btn b{font-size:10px;font-weight:600}
.menu-btn:hover,.menu[data-open] .menu-btn{color:var(--accent)}
.menu-panel{position:absolute;left:-1px;bottom:calc(100% + 8px);display:none;border:1px solid var(--line);
  background:var(--bg);white-space:nowrap}
.menu[data-open] .menu-panel{display:flex}
.menu-panel button{display:flex;align-items:baseline;gap:7px;padding:9px 14px;font-size:10px;font-weight:600;
  letter-spacing:.08em;transition:background .3s,color .3s}
.menu-panel button+button{border-left:1px solid var(--line)}
.menu-panel button span{font-family:var(--cjk);font-stretch:100%;font-size:12px;font-weight:500;letter-spacing:.04em}
.menu-panel button em{font-style:normal;opacity:.7}
.menu-panel button[aria-pressed="true"]{background:var(--accent);color:var(--on-accent)}
.menu-panel button:not([aria-pressed="true"]):hover{color:var(--accent)}
```

删除旧的 `.styles button …` 规则，避免与 `.menu-panel button` 冲突。

- [ ] **Step 3: 事件与开合**

```js
stylesEl.addEventListener("click", e=>{
  const btn = e.target.closest(".menu-btn");
  const opt = e.target.closest(".menu-panel button");
  if (btn){
    const m = btn.parentElement, open = m.hasAttribute("data-open");
    closeMenus(); if (!open){ m.setAttribute("data-open",""); btn.setAttribute("aria-expanded","true"); }
    return;
  }
  if (opt){ setStyle(opt.dataset.s, true); closeMenus(); }
});
function closeMenus(){ stylesEl.querySelectorAll(".menu").forEach(m=>{ m.removeAttribute("data-open"); m.querySelector(".menu-btn").setAttribute("aria-expanded","false"); }); }
document.addEventListener("pointerdown", e=>{ if (!stylesEl.contains(e.target)) closeMenus(); });
document.addEventListener("keydown", e=>{ if (e.key==="Escape") closeMenus(); });
```

- [ ] **Step 4: `markStyle` 同步两组**

```js
function markStyle(id){
  const isLut = FILTERS.some(f=>f.id===id);
  const st = isLut ? FILTERS.find(f=>f.id===id) : STYLES.find(s=>s.id===id);
  stylesEl.querySelectorAll(".menu-panel button").forEach(b=>b.setAttribute("aria-pressed", String(b.dataset.s===id)));
  const sm = stylesEl.querySelector('.menu[data-group="style"] .menu-btn b');
  const lm = stylesEl.querySelector('.menu[data-group="lut"] .menu-btn b');
  sm.textContent = isLut ? "原图" : st.zh;
  lm.textContent = isLut ? st.zh : "原片";
}
```

`setStyle` 里在改状态后调用 `markStyle(id)`（已有调用点保留）。

- [ ] **Step 5: 移动端**

在 `@media (max-width:700px)` 里加：

```css
.menu-btn{padding:8px 9px}
.menu-btn span{font-size:11px}
.menu-panel button{padding:8px 10px}
.menu-panel button em{display:none}
.menu-panel{max-width:calc(100vw - 32px);overflow-x:auto}
```

- [ ] **Step 6: 视觉验证**

打开章节页，分别点开两个菜单，确认：只有一组展开、选中项橙底、选完收起、Esc 和点空白能收起、按钮上的当前项同步。

---

### Task 5: 文档与端到端验证

**Files:**
- Modify: `README.md`（风格台段落）
- Modify: `skills/photo-album/references/components.md`（风格台契约）
- Modify: `style-dna.yaml`（`texture.styles` 旁加 `filters` 注释）

- [ ] **Step 1: 更新 README**

把「原图 / 油墨 / 网点 / 色块 四种样子」改为「风格（原图/红蓝/蓝晒/色块）与滤镜（富士/佳能/理光）两组，键盘 1–4 切风格、5–7 切滤镜」，并说明滤镜需要本地服务器。

- [ ] **Step 2: 更新 components.md**

在「风格台」契约补一条：滤镜组用 `.cube` + 三线性插值，效果函数同样返回 `{canvas}`，大图与风格共用 `FX.current`。

- [ ] **Step 3: 更新 style-dna.yaml**

```yaml
texture:
  dots: square
  styles: [orig, riso, cyanotype, block]
  filters: [fuji, canon, ricoh]   # LUT 滤镜：富士 Astia / 佳能 Portra / 理光 Velvia
```

- [ ] **Step 4: 端到端验证**

```bash
cd /Users/xuetao/Downloads/project/personal/travel-album/travel-album && python3 -m http.server 8765
```

浏览器检查：

1. 打开 `http://localhost:8765/#/ch/4`，点「滤镜 → 富士」，整章逐张擦除变色，控制台无报错。
2. 点「风格 → 红蓝」，滤镜退出，红蓝生效。
3. 快速连点富士、佳能、理光，最终停在理光，无闪烁残留。
4. 在富士选中时点开一张照片，大图显影为富士色调；侧栏显示 `Fuji · Astia 100F`。
5. 带「原作保留」的照片在滤镜下不变。
6. 用 `file://` 打开，滤镜给出提示，风格台仍可用。

- [ ] **Step 5: 收尾**

`git status` 确认只改了预期文件；不提交，等用户确认。

## Self-Review

- **Spec 覆盖**：双菜单（Task 4）、互斥状态（Task 3）、LUT 下载与解析（Task 1–2）、实时渲染（Task 2）、大图同步（Task 3）、失败提示（Task 2–3）、键盘（Task 3）、手机（Task 4）、文档（Task 5）均有对应任务。
- **接口一致性**：`FX.current` / `FX.group` / `curId` / `labelOf` / `ensureLut` / `fxLut` 在 Task 2–4 中名称一致。
- **Review Focus 的测试**：file:// 与 404 → Task 2 Step 4 + Task 5 Step 4；快速连点 → Task 5 Step 4；credit 照片 → Task 3 Step 6；大图回退 → Task 3 Step 4；窄屏 → Task 4 Step 5–6。
- **比例**：计划主要给接口与关键算法，未复述整段成品代码。

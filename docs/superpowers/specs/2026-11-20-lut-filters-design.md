# 章节页双菜单：风格 + LUT 滤镜 · 设计

日期：2026-11-20
状态：待确认

## 意图

章节页底部的风格台目前只有一组按钮（原图 / 红蓝 / 蓝晒 / 色块）。要在同一个位置引入两个主菜单：

- **风格**：原图、红蓝、蓝晒、色块（现有四种，保持不变）
- **滤镜**：富士、佳能、理光（点击后把整章照片实时套上对应 LUT）

用户要的是「换一种相机/胶片的观感」，不是精确的官方色彩科学。按钮用品牌名唤起直觉，界面上不冒充官方 LUT。

## 目标

- 红框位置出现两个主菜单，点击其一展开自己的选项。
- 点品牌后，整章照片用现有橙色擦除动画换成 LUT 调色结果。
- 大图打开时，显影与当前选中项一致。
- 风格与滤镜互斥：选风格即退出滤镜，选滤镜即退出风格。
- 键盘可切换：1–4 风格，5–7 滤镜。

## 非目标

- 不做滤镜强度滑杆。
- 不做单张照片单独套滤（仍是整章统一）。
- 不影响首页照片环、点阵字、声音合成逻辑。
- 不追求官方富士/佳能/理光色彩，不做相机原片对比。
- 不支持 `file://` 直开（与现有风格台一致，需要本地服务器）。

## 反目标

- 按钮让用户以为这是厂家官方 LUT。
- 为这个功能引入构建步骤或第三方依赖。
- 风格叠滤镜（红蓝/蓝晒/色块上再套 LUT）。

## 约束

- 单文件 `index.html`，无构建、无框架、无外部 JS 依赖。
- 现有风格台契约：`STYLES` · `FX_FN` · `coverCanvas` · `paintFx` · `setStyle` · `lbSource`，最长边限制 1100px，效果函数在主线程逐像素跑。
- 照片是手机拍的 sRGB JPEG；官方富士 3D-LUT 面向 F-Log2，不能直接用。
- 仓库为公开相册，第三方 LUT 需要可自由使用的来源并注明出处。

## 方案

### A. Canvas 2D + 三线性插值（已选）

- 新增 `photos/luts/`，放三支 G'MIC 的 `.cube`（13³，约 130KB/支），启动时 `fetch` 预载并解析。
- 新增 `fxLut(id)`，遵守现有效果函数契约：`coverCanvas(img,W,H)` → 逐像素查表 → `{ canvas }`。
- 复用 `paintFx` / `setStyle` / 擦除动画 / 大图显影，风格和滤镜共用同一个状态机。
- 单张 1100px 约 100–250ms；五张错峰擦除可以盖住。
- 代价：不是瞬时；换来零依赖、移动端稳定、改动集中。

### B. WebGL2 3D LUT（未选）

切换近乎瞬时、可加强度滑杆，但需要新增纹理上传、shader、离屏画布、上下文丢失处理，再把结果转回 canvas 喂给大图金字塔与点阵显影。对单文件项目侵入过大。

### C. 纯 CSS filter 近似（排除）

不产生 LUT 文件，也复现不了品牌观感，不满足需求。

## 设计

### 1. 状态模型

把现有的 `FX.style` 改名为 `FX.current`，取值是一个 id：

- 风格：`orig` · `riso` · `cyanotype` · `block`
- 滤镜：`fuji` · `canon` · `ricoh`

另加 `FX.group`（`"style"` / `"lut"`）只用于菜单高亮，不参与渲染。两组合并进 `FX_FN`：

```js
const FX_FN = {
  riso: fxRiso, cyanotype: fxCyanotype, block: fxBlock,
  fuji: (img,W,H) => fxLut("fuji", img, W, H),
  canon: ..., ricoh: ...,
};
```

这样 `paintFx`、`lbSource`、缓存 key（`src|id|WxH`）、`chapter[data-style]` 全部不用改逻辑，只把 `FX.style` 改成 `FX.current`。

`orig` 仍然只属于风格组；选「原图」即退出滤镜。

### 2. 菜单形态

底部仍是一条，居中，分成两个主菜单按钮：

```
[ 风格  红蓝 ] [ 滤镜  富士 ]
```

- 每个按钮显示组名 + 当前选中项；未选滤镜时显示「滤镜 原片」。
- 点击按钮，在上方弹出该组的选项条：风格 4 项、滤镜 3 项；选中项填强调色（沿用 `aria-pressed` 样式）。
- 同时只开一个弹层；再点同一按钮、点页面空白、按 Esc 收起；选完自动收起并响一声 `sndChime`。
- 视觉沿用现有语汇：1px `--line` 边框、`--bg` 底、无圆角、无阴影、11–12px 字号；弹层从按钮上方 6px 处淡入 + 轻微上移，`prefers-reduced-motion` 时不动。
- 手机上弹层与按钮同宽或按内容撑开，选项收紧（隐藏英文副标题），不超出屏幕。

### 3. LUT 管线

文件：

| 按钮 | 真实 LUT | 文件 | 来源 |
|---|---|---|---|
| 富士 | Fuji X-Trans III Classic Chrome | `fuji_xtrans_iii_classic_chrome.cube` | G'MIC |
| 佳能 | Kodak Portra 400（暖调人像） | `kodak_portra_400.cube` | G'MIC |
| 理光 | Fuji Velvia 50（高饱和正片） | `fuji_velvia_50.cube` | G'MIC |

- `loadLuts()` 在启动后并行 `fetch` 三支，解析 `LUT_3D_SIZE`、`DOMAIN_MIN/MAX` 与数据行，存成 `Float32Array`（`N³×3`）。
- `fxLut(id,img,W,H)`：`coverCanvas` → `getImageData` → 每像素归一化到 `[0,1]` → 三线性插值查表 → 写回 `ImageData`。
- 逐像素循环不分配对象，权重与索引预先算好；支持任意 `N`，不写死 13。
- 读取失败（`file://`、404、解析失败）时：滤镜不可用，`chLine` 提示「滤镜需要用本地服务器打开这个网页」；风格台不受影响；大图回退原片。
- `photos/luts/SOURCES.md` 记录每支 LUT 的下载地址、preset 名、G'MIC 的商标说明，以及「仅为观感近似，非厂家官方」的声明。

### 4. 大图与侧栏

- `lbSource` 直接用 `FX_FN[current]`，逻辑不变；外层加 try/catch，LUT 缺失时回退原片。
- 侧栏模式文案：风格显示 `Riso · 红蓝`，滤镜显示 `Fuji · Astia`；新增一个 id → `{zh,en}` 的查找表。
- 大图内 1–4 仍切风格，5–7 切滤镜。

### 5. 键盘与无障碍

- 章节页与大图共用：1–4 风格、5–7 滤镜。
- 菜单按钮 `aria-expanded`，选项按钮 `aria-pressed`，弹层用 `role="group"` + `aria-label`。
- Esc 先收弹层；弹层收起时焦点回到触发按钮。
- 现有 `reduce`（prefers-reduced-motion）路径不跑擦除动画，直接重绘。

### 6. 性能

- 画布最长边 1100（`measureChapter` 既有约束），13³ 查表约 8 次取值/像素。
- 预载三支 LUT 约 400KB，启动后空闲时加载，不阻塞首屏。
- 现有 `FX.cache` 最近 14 张的策略不变；切风格/滤镜时旧条目自然淘汰。

## 改动的文件

- `index.html`：CSS（菜单与弹层）、HTML（菜单结构）、JS（状态、LUT 加载与解析、`fxLut`、菜单交互、键盘、侧栏文案）。
- `photos/luts/*.cube`（3 支）与 `photos/luts/SOURCES.md`。
- `README.md`：风格台描述补一句滤镜。
- `skills/photo-album/references/components.md`：风格台契约补「滤镜组」。
- `style-dna.yaml`：`texture.styles` 旁边加 `filters` 注释（可选，保持配置可读）。

## 留给实现者的判断

- 三支 LUT 落在实际照片上的观感需要跑起来看；若某一支明显不合适，可在同一来源里换同名系列的其他 preset（如 `fuji_astia_100f` → `fuji_provia_100f`），不改接口。
- 弹层在 320–360px 窄屏的间距，按实际截图微调。
- `FX.cache` 上限是否要从 14 调到 10，等实测内存占用再定。

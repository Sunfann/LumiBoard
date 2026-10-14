# LumiBoard · 3D 成品模拟「交互复刻」提示词（v3）

> 用途：把 **LumiBoard 成品效果模拟（3D 成品卡片）** 的完整体验，复刻 / 重建到任意前端项目（或在本项目内从零实现）。
> 参考原型：`cassette`（OHM TAPE 卡带展示台，three@0.180）<https://zcy83821448.github.io/cassette/>
> 目标文件：`lumiboard.html`（单文件、vanilla JS、three.js、`file://` 离线直开、零构建）
> 版本：v3 · 2026-10-14（新增：四轴自由展开 / 逐层拾取聚焦置顶 / 双击双层复位 / 翻面层序 1→7 / 介绍卡片默认透明）

---

## 【角色与背景】

你是一名**资深前端 3D 交互工程师**。任务是在一个名为 **LumiBoard** 的**单文件 HTML 应用**（纯前端、vanilla JS + three.js、`file://` 离线直开、无构建工具、无外部依赖）中，**复刻并实现「3D 成品模拟」的完整交互体验**：把一张双面艺术卡牌的 **7 层叠层结构**（顶层丝印 / 顶层阻焊 / 顶层铜箔 / FR-4 基板 / 底层铜箔 / 底层阻焊 / 底层丝印）以摄影棚成品质感的 3D 卡片呈现，并支持多机位观察、四轴分层展开、逐层拾取、翻面、背光、档案浏览等交互。

**前置条件**：目标文件已存在且可运行；three.js 通过动态 `import()` 按需加载；2D 侧已能产出各层 canvas（`mlBuild()` / `physBuildCanvas()`）。若参考原型源码可用（`cassette_repo/src/`），实现时可逐条对照其手感参数。

---

## 【任务描述】

在目标文件 `lumiboard.html` 中**复刻 3D 成品模拟的全部交互**，并保证不破坏既有 2D 功能（卡面设计、多层导出、ZIP/EDA、标定、实时预览、撤销栈、Ctrl+K 命令面板）。子任务如下（每条都可判定是否完成）：

1. **3D 场景与模型**
   1.1 程序化构建 7 层卡牌模型（层序 `_seq = { silkB:0, maskB:1, copperB:2, core:3, copperT:4, maskT:5, silkT:6 }`），层材质能反映真实阻焊色、表面处理、板厚与长宽比。
   1.2 摄影棚五灯位 + PMREM 环境 + 地面反射；背景保持摄影棚灰阶，**不得纯黑坍塌**。
   1.3 three.js 加载失败或 WebGL 不可用时**自动降级为 2D 静态输出**（永久兜底，不白屏）。

2. **相机操控与机位系统**
   2.1 拖拽旋转跟手（阻尼 rate≈14），松手惯性滑行（rate≈7）自然衰减停止。
   2.2 滚轮 / 双指缩放平滑 clamp；UI 浮层上的滚轮不被劫持。
   2.3 **5 个机位**（iso 等轴 / front 正面 / side 侧面 / detail 细节 / macro 微距）循环切换，走最短角路径，时长约 0.9s inOut；`←` / `→` 切机位，`⇐` 回主机位。
   2.4 首次进入播放入场动画（卡片平躺 → 抬升悬浮 → 相机补入），**补入结束回调中执行机位复位**。

3. **全屏取景与机位复位（硬性，不得回归）**
   3.1 进入 3D 全屏后卡片必须**居中铺满**面板，不得缩到角落或偏轴。
   3.2 提供 `snapOrbitHome()`：清空 `fitCache` → 按当前视口重算机位表 → 吸附 `theta/phi/radius` 到当前机位拟合值 → `tween=null` → `apply()`。
   3.3 `resize()` 的 snap 条件为 `!dragging && (!tween || introDone)`；横竖屏切换 / DPR 变化时清 `fitCache` 后调用 `snapOrbitHome()`；在 iso/front/side/detail/macro 各机位、DPR 1/1.25/1.5/2 下取景均正确。

4. **四轴自由展开（硬性）**
   4.1 提供四个展开方向预设：`⤢ 对角(iso)` / `↔ 横展(x)` / `↕ 竖展(y)` / `⊙ 纵深(z)`。
   4.2 四方向**互不干扰**，可随时自由切换层叠展开方向；展开系数连续可调（合拢 ↔ 完全展开）。
   4.3 展开仅作用于几何位移，**不改变层序、不改变机位、不改变材质**。

5. **逐层拾取 · 聚焦 · 置顶（硬性）**
   5.1 在任一展开方向下，**单击任意一层**即选中该层。
   5.2 选中后该层**位于视觉中心**（相机缓动聚焦其包围盒）且**最上层展示**（提升 `renderOrder`、必要时 `polygonOffset`，`depthTest` 关闭防遮挡；未选中层淡化）。
   5.3 命中判定三重解耦（缺一不可）：① 拖拽/点击位移阈值判别（≈6px，超过即视为旋转相机，不选中）；② 射线拾取（画布坐标 → NDC → `raycaster.setFromCamera` → `intersectObjects(层 mesh)`，取最近命中，经 `mesh.userData.layerKey` 反查层 key）；③ 展开动画未收敛时（`sceneAnimating()` 门控）暂缓选中。
   5.4 选中层对应的**介绍卡片**同步显示该层档案。

6. **双击复位（两个目标态，硬性）**
   6.1 **双击某一层** → 恢复「本次选中之前」的显示状态（相机/聚焦/置顶/展开轴/翻面层序回放到选中前快照）。
   6.2 **双击其他任意位置**（拾取落空）→ 恢复「初始状态」（回写初始基线：轴/系数/聚焦/翻面/机位全回初始）。
   6.3 单双击必须显式解耦：单击延迟约 250ms 执行；窗口期内再次点击 → 清除定时器并按双击处理；**不**同时绑定原生 `dblclick`（避免双击被当成两次单击）。

7. **翻面层序 1→7（硬性）**
   7.1 翻面语义为「沿层序 1→7 逐层推进」，**不是**整卡一次性 180°。
   7.2 以 `_flipOrd = { silkT:0, maskT:1, copperT:2, core:3, copperB:4, maskB:5, silkB:6 }` 为权威翻面层序索引；推进过程中**每层都必须在位、不得缺失**。
   7.3 翻面不改变机位、不改变展开轴；正反面展开方向经翻转角补偿（`_cfl = cos(π*flip)`、`_sfl = sin(π*flip)`）保持一致，不回归。

8. **背光 / 扫光 / 标定**
   8.1 `空格` 点亮背光：非线性透光曲线（亮部透光、暗部遮光），透光路线点亮后边缘辉光可见。
   8.2 **金属扫光**：一束高光沿板面**自左向右线性扫射**而过（参数从 -7 → +7 线性推进，非往复抖动）；随工艺路线自动标注为「金属扫光 / 点亮背光」。
   8.3 `M` 键循环标定预设（std → 背光 → 沉金 → 喷锡 → 本地标定库），映射为 emissive 增益 + rim 灯色温 + 饱和/光晕微调，全程阻尼过渡。

9. **档案系统与介绍卡片**
   9.1 档案列表共 **9 条**（`no: '00'..'08'`）：00 整卡 / 01 顶层丝印 / 02 顶层阻焊 / 03 顶层铜箔 / 04 FR-4 基板 / 05 底层铜箔 / 06 底层阻焊 / 07 底层丝印 / 08 板边轮廓。
   9.2 `↑` / `↓` 切上一/下一档案，`I` 打开档案目录；**档案切换时固定当前视角**（不调用机位切换），只切换层级。
   9.3 **介绍卡片默认透明**：默认 `opacity:0` 且 `pointer-events:none`（不遮挡画布操作）；仅在**鼠标悬停、点击或选中**时淡入显示（`sim3d-dossier-show`）。

10. **按需渲染与性能**
    10.1 完全静止时**跳过重绘**（`sceneAnimating` + `_dirty` 门控）；有交互/动画时才重绘。
    10.2 切走「生产」Tab 或 `document.hidden` 时停止渲染循环，切回恢复并将 dt 夹 50ms。
    10.3 连续掉帧（3 秒滑动均值 > 22ms）自动降级链：关 AO → 关散焦 → 关平面反射 → `pixelRatio` 降 1 → 退 bloom；一次性提示「已进入流畅模式」，本会话锁定不自动回升。

11. **命令面板与状态记忆**
    11.1 `LUM_CMDK_ITEMS` 追加 3D 词条：`3D · 拆解视图 / 3D · 合拢视图 / 3D · 翻面 / 3D · 背光点亮 / 3D · 自动巡览 / 3D · 下一档案`，关键词含 `3d / 拆解 / 翻面 / 背光`。
    11.2 视图状态记忆于 **`lumi.ui.v2`**：`sim3d: {on, vantage, exploded, flipLayer, auto, backlit, record, explodeAxis}`，刷新后恢复（机位用 `finish()` / `snapOrbitHome()` 硬恢复，无闪跳）。

---

## 【输入说明】

- **必填**：目标文件 `lumiboard.html`（单文件，vanilla JS + three.js）。
- **必填（关键现有代码锚点，直接对接）**：
  - `mlBuild()` → `{v, p, L}`（`L` 含各层 canvas）；`physBuildCanvas(v, content)`。
  - 层序表 `_seq`（见 1.1）；翻面层序 `_flipOrd`（见 7.2）。
  - 四轴展开 `explodeAxis`；翻面 `setFlip` / 方向补偿 `_cfl` / `_sfl`；机位 `setVantage`。
  - 选中高亮 `applyFocus(key)` / 材质 `matSolid` / `matGhost`；按需渲染门控 `sceneAnimating()`。
  - 每层 mesh 存于 `App.layers`；档案数据由 `dossierData()` 提供。
- **可选**：2D → 3D 状态记忆键 `lumi.ui.v2`（刷新恢复用）。
- **可选**：参考原型源码 `cassette_repo/src/`（controls.js / anim.js / cassette.js / lights.js / env.js / post.js / floor.js / ao.js / textures.js / tags.js / audio.js / probe.js），用于逐条对表手感参数。
- **可选**：品牌色/主题变量、文案风格（中文界面即可）。

---

## 【输出要求】

- **交付物**：直接修改目标文件，给出**完整可用代码**；新增交互全部约束在「观察层」（相机 / 聚焦 / 置顶 / 展开轴 / 翻面层序 / 档案），不改动生产 `state` 与撤销栈。
- **语言**：代码注释用中文；关键处保留原有注释风格。
- **结构**：3D 模块用独立 `<script type="module">` 动态 `import()`，three 挂 `window.__THREE__` 桥供 IIFE 侧访问，**不破坏现有严格模式 IIFE**。
- **完整性**：7 层、四轴、拾取、双击复位、翻面层序、档案、介绍卡片显隐等**逐条实现且可验收**；给出一条可勾选的自测清单。

---

## 【约束与偏好】

- **技术栈**：单文件、零构建、`file://` 可离线运行；仅使用已有 three.js 能力，**不引入外部控件库**。
- **正确性**：
  - 单击/双击必须显式解耦（单击延迟约 250ms，双击清定时器；不绑原生 `dblclick`）。
  - 点击判定必须先做拖拽/点击位移阈值判别（≈5–8px）。
  - 置顶必须组合 `renderOrder`（选中层最高）+（必要时）`polygonOffset` + 选中层 `transparent:false`、未选中层 `depthWrite:false`；共面叠层不得 z-fighting 闪烁。
  - 复位/聚焦动画必须帧率无关（`MathUtils.damp`）且**可取消**（新操作接管当前补间，记录进度，避免抖动）。
  - 翻面层序推进中每层都在位、不缺失；正反面展开方向一致不回归。
  - 介绍卡片隐藏态必须 `pointer-events:none`，绝不遮挡画布拖拽与拾取。
- **状态解耦**：3D 一切视图状态**不进撤销栈**；`Ctrl+Z` 在生产 Tab 照旧作用于卡面参数。
- **无障碍与偏好**：全套控件为真实 `button` + `aria` 且键盘可达；`prefers-reduced-motion: reduce` 时聚焦/复位/入场动画退化为瞬时硬切，但**保留单双击意图判定**。
- **移动端**：生产 Tab ≤940px 单栏；3D 画布高 `min(56vw, 420px)`；画布 `touch-action: pan-y`（单指纵向滚页、横向旋转，双指立即锁捏合）；工具条纯图标 + `title`。
- **禁止**：不得把反面层静默回退成正面（如 `copperB = cuBA || cuA`）、不得用未镜像正面内容冒充反面、不得在档案切换时调用 `setVantage`、不得因新增交互而常驻重绘。

---

## 【示例】

**输入 → 输出的关键片段示意（逐层拾取 + 单双击解耦 + 置顶）：**

```js
// 1) 单双击显式解耦（不绑原生 dblclick）
let clickTimer = null;
canvas.addEventListener('pointerup', (e) => {
  if (dragDist > 6) return;                         // 拖拽/旋转，不判为点击
  if (sceneAnimating()) return;                     // 展开未收敛，暂缓拾取
  if (clickTimer) { clearTimeout(clickTimer); clickTimer = null; onDoubleClick(e); }
  else { clickTimer = setTimeout(() => { clickTimer = null; onClick(e); }, 250); }
});

// 2) 射线拾取（用画布 rect，不用 window 尺寸）
function pickLayer(px, py, rect) {
  const ndc = new THREE.Vector2((px - rect.left) / rect.width * 2 - 1,
                                -((py - rect.top) / rect.height * 2 - 1));
  raycaster.setFromCamera(ndc, camera);
  const hit = raycaster.intersectObjects(App.layers.map(l => l.mesh), false)[0];
  return hit ? hit.object.userData.layerKey : null; // 落空 = 空白
}

// 3) 选中 = 高亮 + 相机聚焦 + 置顶
function selectLayer(key) {
  pushPrevSnapshot();                                // 存「选中前」快照(__prev)
  applyFocus(key);                                   // 选中 matSolid / 其余 matGhost
  const box = new THREE.Box3().setFromObject(layerMesh(key));
  focusCameraOnBox(box);                             // 位置 damp + 朝向 slerp + fov 插值
  bringToFront(layerMesh(key));                      // renderOrder 最高，depthTest 关
  revealDossier(key);                                // 介绍卡片同步显示该层档案
}

// 4) 双击复位：命中层 → 回放 __prev；落空 → 回写 baseline
function onDoubleClick(e) {
  const key = pickLayer(e.clientX, e.clientY, canvas.getBoundingClientRect());
  key ? restorePrevView() : restoreInitialView();
}
```

**示例用法（一句话）**：把本文件「可直接复制的一整段提示词」整段粘贴给编码助手，并附上 `lumiboard.html`，即可复刻 3D 成品模拟的全部交互。

---

## 可直接复制的一整段提示词（总结版）

```
【角色】你是一名资深前端 3D 交互工程师，维护单文件 HTML 应用 LumiBoard（vanilla JS + three.js，file:// 离线直开，无构建）。

【任务】在 lumiboard.html 中复刻「3D 成品模拟」的完整交互，且不破坏既有 2D 功能（卡面设计、多层导出、ZIP/EDA、标定、实时预览、撤销栈、Ctrl+K）。实现以下 11 组子任务：
1) 程序化 7 层卡牌模型，层序 _seq={silkB:0,maskB:1,copperB:2,core:3,copperT:4,maskT:5,silkT:6}，材质反映真实阻焊色/表面处理/板厚/长宽比；摄影棚五灯位+PMREM+地面反射，背景摄影棚灰阶不纯黑；three.js/WebGL 不可用时自动降级为 2D 静态输出（永久兜底）。
2) 相机：拖拽跟手(rate≈14)松手惯性(rate≈7)；缩放平滑 clamp 且 UI 浮层不劫持滚轮；5 机位(iso/front/side/detail/macro)最短角路径 0.9s inOut，←→ 切机位、⇐ 回主机位；首次进入播入场动画，补入结束回调执行机位复位。
3) 全屏取景（硬性）：卡片居中铺满，不缩角；提供 snapOrbitHome()（清 fitCache→重算机位表→吸附 theta/phi/radius→tween=null→apply()）；resize 的 snap 条件 !dragging && (!tween||introDone)；横竖屏/DPR(1/1.25/1.5/2) 变化清 fitCache 并 snapOrbitHome()。
4) 四轴自由展开（硬性）：⤢对角(iso) / ↔横展(x) / ↕竖展(y) / ⊙纵深(z)，四方向互不干扰、可随时切换、系数连续可调；仅改几何位移，不改层序/机位/材质。
5) 逐层拾取（硬性）：四轴任一方向单击任一层即选中，该层位于视觉中心（相机缓动聚焦包围盒）且最上层展示（renderOrder 最高+必要时 polygonOffset+depthTest 关）；三重解耦：拖拽位移阈值≈6px 判点击、射线拾取(画布 rect→NDC→raycaster→intersectObjects(App.layers mesh)，mesh.userData.layerKey 反查)、sceneAnimating() 门控；选中层同步显示介绍卡片档案。
6) 双击复位（硬性）：双击某层→回放「选中前」快照(相机/聚焦/置顶/展开轴/翻面层序)；双击空白→回写初始基线；单双击显式解耦(单击延迟≈250ms，双击清定时器，不绑原生 dblclick)。
7) 翻面层序 1→7（硬性）：沿层序逐层推进而非整卡 180°，索引 _flipOrd={silkT:0,maskT:1,copperT:2,core:3,copperB:4,maskB:5,silkB:6}；推进中每层都在位不缺失；翻面不改机位/展开轴；正反面展开方向经 _cfl=cos(π*flip)/_sfl=sin(π*flip) 补偿保持一致。
8) 背光/扫光/标定：空格点亮背光(非线性透光，亮部透光亮部遮光)；金属扫光沿板面自左向右线性扫射(-7→+7，非往复)；M 键循环标定预设(std→背光→沉金→喷锡→本地库)，emissive+rim 色温+饱和/光晕微调，阻尼过渡。
9) 档案与介绍卡片：9 条档案 00 整卡/01 顶层丝印/02 顶层阻焊/03 顶层铜箔/04 FR-4 基板/05 底层铜箔/06 底层阻焊/07 底层丝印/08 板边轮廓；↑↓ 切档案、I 开目录；档案切换固定视角(不调 setVantage)；介绍卡片默认 opacity:0 且 pointer-events:none，仅悬停/点击/选中时淡入(sim3d-dossier-show)。
10) 按需渲染与性能：静止跳过重绘(sceneAnimating+_dirty)；切走生产 Tab/页面隐藏停渲染、切回恢复并夹 dt 50ms；掉帧自动降级链(关 AO→关散焦→关反射→pixelRatio 降 1→退 bloom)并一次性提示。
11) 命令面板与状态：LUM_CMDK_ITEMS 追加「3D·拆解视图/合拢视图/翻面/背光点亮/自动巡览/下一档案」；视图状态记忆于 lumi.ui.v2 的 sim3d 字段，刷新硬恢复机位。

【输入】目标文件 lumiboard.html；关键锚点：mlBuild()→{v,p,L}、physBuildCanvas、_seq、_flipOrd、explodeAxis、setFlip/_cfl/_sfl、setVantage、applyFocus/matSolid/matGhost、sceneAnimating()、App.layers、dossierData()；可选参考源码 cassette_repo/src/ 与状态键 lumi.ui.v2。

【输出】直接修改目标文件、给出完整可用代码；新增交互全部约束在观察层(相机/聚焦/置顶/展开轴/翻面层序/档案)，不改生产 state 与撤销栈；3D 用独立 module 脚本动态 import()，three 挂 window.__THREE__，不破坏严格模式 IIFE；关键处补中文注释；附可勾选自测清单。

【约束】单文件零构建 file:// 离线；不引入外部控件库；单击/双击显式解耦且先做拖拽阈值判别；置顶用 renderOrder+polygonOffset+depthWrite:false 联合消 z-fighting；聚焦/复位帧率无关(MathUtils.damp)且可取消；介绍卡片隐藏态 pointer-events:none 不遮挡；prefers-reduced-motion 时动画退化为瞬时但保留单双击判定；移动端画布 touch-action:pan-y、≤940px 单栏；禁止把反面层静默回退成正面、禁止档案切换调用 setVantage、禁止新增交互导致常驻重绘。
```

---

*附（复刻锚点速查）：状态键 `lumi.ui.v2`；档案 9 条（00–08）；机位 5 个（iso/front/side/detail/macro，`VANTAGE 01/05`）；四轴令牌 `iso|⤢|对角`、`x|↔|横展`、`y|↕|竖展`、`z|⊙|纵深`；翻面序 `_flipOrd`；介绍卡片类 `sim3d-dossier-show / -pin / -tight`；3D 命令词条 6 条。参考原型：cassette（OHM TAPE，three@0.180）。*

# WebGL2 Meteo Engine (`maplibre-gl` CustomLayer)

基于 **MapLibre GL JS (`CustomLayerInterface`)** 与原生 **WebGL2 (`#version 300 es`)** 的实时气象渲染引擎，实现百万级风场粒子平流、屏幕空间相机重投影拖尾、实例化 WMO 标准矢量风向杆（Procedural SDF）以及大气散射、等压线与 3D 体渲染。

---

## 技术栈与开源参考基线

- **GIS 底座**：`maplibre-gl` v4（共享原生 WebGL2 上下文）
- **数学与构建**：`TypeScript` + `Vite` + `gl-matrix` + `vite-plugin-glsl`
- **自动化验证**：`glslangValidator`（静态语法/精度检查）+ `Playwright`（SwiftShader 无头 WebGL2 渲染）+ `pixelmatch`（视觉回归比对）
- **行业参考基线**：
  - 粒子平流基础参考：`mapbox/webgl-wind`（升级至 WebGL2 并补齐相机重投影与地图无缝联动）
  - 地图图层接入参考：`sakitam-fdd/wind-layer`
  - 气象多要素对标：`weatherlayers/weatherlayers-gl`

---

## 渲染管线架构（4-Pass WebGL2 Pipeline）

| Pass | 模块 | 读写目标 | 核心能力 |
| :--- | :--- | :--- | :--- |
| **Pass 1** | 粒子状态平流 (`advection.frag.glsl`) | 读 `State_Tex_A` + 双时刻风场 UV；写 `State_FBO_B` | 时间线性插值、RK2 二阶龙格-库塔积分、墨卡托纬度余弦补偿、静风/越界哈希重生 |
| **Pass 2** | 尾迹重投影与落点 (`trail_fade` + `particle_draw`) | 读 `Trail_Tex_A` + `State_Tex_B`；写 `Trail_FBO_B` | 上一帧尾迹重投影防拖拽错位、0.96 衰减与低 Alpha 截断、`texelFetch` 落点着色（速度 LUT） |
| **Pass 3** | 气象矢量与大气 (`wind_barb` + `atmosphere`) | 读风场 UV / 气压 / 3D 云体纹理；写合成 FBO | RTC 防抖实例化 WMO 风向杆（SDF + `fwidth` 抗锯齿）、瑞利/米氏散射、`dFdx/dFdy` 实时等压线、蓝噪声 3D Raymarching |
| **Pass 4** | 底图融合 (`MeteoCustomLayer`) | 读 `Trail_Tex_B` + Pass 3；写主帧缓冲 (`null`) | 预乘 Alpha 混合接入 MapLibre 渲染队列，帧末零拷贝交换双缓冲指针 |

---

## 关键工程约束（Implementation Constraints）

1. **离屏 FBO 双缓冲（Ping-Pong）铁律**
   - 严禁同一纹理在单次 Draw Call 中同时作为采样输入与当前绑定 FBO 的颜色附件（规避 `GL_INVALID_OPERATION: Feedback loop`）。
   - 状态纹理优先检测 `EXT_color_buffer_float` 使用 `RGBA32F`；不支持浮点 FBO 写入的设备自动降级为 `RGBA8` 双字节定点编码（16-bit 精度）。
   - 所有粒子状态纹理强制采用 `gl.NEAREST`、`gl.CLAMP_TO_EDGE` 且禁用 Mipmap；Pass 1 执行期间关闭 `gl.BLEND`。
2. **气象坐标系与物理约定**
   - 输入风场 UV 纹理：`R` 通道为东向分量 U（m/s），`G` 通道为北向分量 V（m/s）；确保北半球低压涡旋呈**逆时针**旋转。
   - **WMO 风向杆标准**：羽毛端指向**风吹来的方向**（与粒子运动方向相差 180 度）；按 50kt 实心三角旗、10kt 长羽、5kt 短羽、小于 2kt 空心圆组合绘制；支持南北半球风羽朝向切换。
3. **数值稳定性与高精度防抖**
   - 所有入口 GLSL 强制以 `#version 300 es` 为第 1 行，显式声明 `precision highp float;`。
   - 静风区必须使用 `safeNormalize()` 保护，严禁 `normalize(vec2(0.0))` 产生 `NaN` 污染 Ping-Pong 尾迹缓冲。
   - 高缩放级别下风向杆采用相对中心偏移（RTC）避免 `float32` 抖动。

---

## 交付里程碑与自动化验收标准（CI Gates）

- [ ] **Gate 1 · 着色器静态编译**：`npm run validate:shaders` 零 Error、零 Warning。
- [ ] **Gate 2 · 确定性合成风场断言**：Playwright + SwiftShader 注入纯西风、北半球兰金涡旋、全零静风场，连续 5 帧无 `GL_ERROR`、无 `NaN` 扩散、涡旋满足北半球逆时针旋度。
- [ ] **Gate 3 · WMO 风向杆视觉回归**：覆盖 `0kt / 5kt / 15kt / 50kt / 65kt` 及南北半球测试矩阵，`pixelmatch` 像素差异小于 `0.5%`。

---

## 本地启动与验证

```bash
npm run validate:shaders
npm install && npx playwright install chromium
npm run dev
npm run verify:all
```

# 5-DOF 笛卡尔坐标打印系统

- 5-DOF 笛卡尔坐标打印系统是在当前 3-DOF 坐标系统的基础上，增加了物料底盘的 **倾斜** 和 **旋转** 自由度，可实现曲面打印
## 【最基础】方案一：基于曲面偏移与测地线距离的曲面打印算法
### TL;DR
- [Open5x: Accessible 5-axis 3D printing and conformal slicing](https://arxiv.org/pdf/2202.11426) 这篇论文是一篇很好的 5-DOF 3d 打印入门论文，提供了：
  - 低成本硬件改装方案：把一台常见的桌面级 Prusa i3 MK3s（3 轴）改装为 5 轴。
  - 可视化切片软件：在 Rhino + Grasshopper 中开发的 GUI 工具，能自动生成 conformal toolpath、G-code 并仿真打印。
- 5-DOF 打印演示视频：https://www.youtube.com/watch?v=x9rG15qrDIE

![ 2025-10-19 105211.png](https://s2.loli.net/2025/10/19/fXwmNaJzxBDAd4j.png)
> 5-DOF 曲面 3D 打印机示意图

![ 2025-10-19 105244.png](https://s2.loli.net/2025/10/19/dfFhAVzl6Ckpecg.png)
> 曲面切片路径规划 + G-code 生成

### 5-DOF 曲面打印的用途
#### 1. 无支撑打印（Support-less Printing）
- 通过动态调整喷嘴方向实现无支撑打印；示例：涡轮扇叶、风扇叶片；
- 优点：
  - 无需支撑材料；
  - 打印时间更短；
  - 表面更平滑。

![ 2025-10-19 110816.png](https://s2.loli.net/2025/10/19/qIdayfr6Nl2CuL8.png)

#### 2. 曲面沉积（Conformal Extrusion）
- 可在已有结构上再打印曲面层；
- 例如：
  - 在普通 PLA 基底上打印导电 PLA；
  - 在溶解性 PVA 上打印悬空结构；
  - 实现功能性打印，如 球面 LED 电路，解决了传统分层打印中导电路径在倾斜面上阻抗过高的问题。

![ 2025-10-19 110836.png](https://s2.loli.net/2025/10/19/epF92UthLXbkTw4.png)

#### 3. 结构强化（Structural Strengthening）
- 通过沿受力方向进行曲面沉积，提高各向同性强度；
- 桥梁模型实验表明：
  - 混合“平面层 + 曲面层”结构在多方向载荷下更稳定；
  - 显微图像显示层间粘结更紧密。
 
![ 2025-10-19 110851.png](https://s2.loli.net/2025/10/19/4T8wcryNuYbjMVq.png)

### 后续 TODO
1. 曲面切片软件研发，open5x 开源代码中只包含了切片结果，作者自身也没有解决自动化曲面切片算法的问题。
2. 目前论文中采用更换不同直径的打印床以减少喷嘴碰撞风险，后续曲面切片算法研发过程中，需要重点解决碰撞问题。
3. G-code 生成时，和 3-DOF 笛卡尔坐标打印系统不同，5-DOF 打印系统需要多轴联动时精确计算各轴运动距离与喷嘴速度匹配，需要计算运动的欧几里得距离： $d=\sqrt{\Delta x^2+\Delta y^2+\Delta z^2+\Delta u^2+\Delta v^2+\Delta e^2}$ 。

## 【进阶】方案二：基于场的优化

![ 2025-10-19 114108.png](https://s2.loli.net/2025/10/19/W2mprqIGDXLsz35.png)

![ 2025-10-19 114135.png](https://s2.loli.net/2025/10/19/jHASmsWbnDai5Ic.png)

![ 2025-10-19 114149.png](https://s2.loli.net/2025/10/19/5ISHKYhwvaQtODU.png)
### TL;DR
- paper: [Reinforced FDM: Multi-Axis Filament Alignment with Controlled  Anisotropic Strength](https://mewangcl.github.io/pubs/SIGAsia2020ReinforcedFDM.pdf)
- code: [ReinforcedFDM](https://github.com/GuoxinFang/ReinforcedFDM)
- 论文希望 **利用多轴打印机（multi-axis 3D printer）的能力，让喷头能在空间中改变方向，使打印丝线能沿最大主应力方向（principal stress direction）** 铺设，从而显著增强结构强度。
### 算法核心思想
- 算法核心思想是：基于场的优化框架（field-based optimization framework），主要包括：
#### 1. 场优化（Field Optimization）
- 计算一个标量场 $G(x)$ ，其梯度方向 $\Delta G(x)$ ，决定每一层曲面（curved layer）的法线方向。
- 每一层的切线方向则决定丝线铺设方向。
- 目标是让丝线方向与模型的最大主应力方向 $T_{max}$ 对齐。
#### 2. 制造约束（Manufacturability）
- 保证无碰撞（collision-free）；
- 可达性（accessibility）：喷头能触达每一打印点；
- 层厚控制（layer thickness control）；
- 过悬支撑（support generation）。
#### 3. 多轴打印路径生成（Toolpath Generation）
- 基于优化后的曲面，生成沿应力方向的丝线路径，实现力学增强。
### 算法流程
1. 输入：
  - 3D 模型（tetrahedral mesh）；
  - 外部载荷（loading conditions）。
2. 有限元分析（FEA）：
  - 计算每个四面体单元的应力张量，得到主应力方向。
3. 矢量场优化 $V(x)$：
  - 在高应力区域（critical regions）内，使 $V(x)$ 的方向与主应力方向一致；
  - 同时保持场的平滑性、避免方向冲突。
4. 标量场构造 $G(x)$：
  - 通过最小化 $||\Delta G(x)-V(x)||^2$ ，得到曲面切片层（isosurfaces）。
5. 制造可达性优化：
  - 搜索最佳打印姿态（orientation）；
  - 通过场松弛（field relaxation）调整 $V(x)$，避免喷头碰撞。
6. 支撑生成（Support Generation）：
  - 对悬空区域外推场，生成与主体层兼容的支撑层。
7. 路径规划（Toolpath Generation）：
  - 在关键区域沿应力方向生成路径；
  - 其他区域采用混合策略（directional-parallel + contour-parallel）；
  - 确保路径连续、填充率高。
### 技术细节
模块	|关键技术
---|---
应力计算|	FEA + 主应力分解（eigen decomposition）
场优化|	分段区域、矢量方向一致性优化（greedy 反转法）
层生成	|由 $G(x)$ 的等值面（iso-surface）生成曲面层
支撑生成	|外推 $V(x)$ 与 $G(x)$，得到兼容的支撑层
可达性分析|	基于 convex-shadow-volume 的快速可达度评估
路径规划|	混合式路径：应力导向 + 轮廓平行，确保连续性与强度

# 5-DOF 笛卡尔坐标打印系统
## 定义
- 5-DOF 笛卡尔坐标打印系统是在当前 3-DOF 坐标系统的基础上，增加了物料底盘的 **倾斜** 和 **旋转** 自由度，可实现曲面打印
- [Open5x: Accessible 5-axis 3D printing and conformal slicing](https://arxiv.org/pdf/2202.11426) 这篇论文是一篇很好的 5-DOF 3d 打印入门论文，提供了：
  - 低成本硬件改装方案：把一台常见的桌面级 Prusa i3 MK3s（3 轴）改装为 5 轴。
  - 可视化切片软件：在 Rhino + Grasshopper 中开发的 GUI 工具，能自动生成 conformal toolpath、G-code 并仿真打印。
- 5-DOF 打印演示视频：https://www.youtube.com/watch?v=x9rG15qrDIE

![ 2025-10-19 105211.png](https://s2.loli.net/2025/10/19/fXwmNaJzxBDAd4j.png)
> 5-DOF 曲面 3D 打印机示意图

![ 2025-10-19 105244.png](https://s2.loli.net/2025/10/19/dfFhAVzl6Ckpecg.png)
> 曲面切片路径规划 + G-code 生成

## 5-DOF 曲面打印的用途
### 1. 无支撑打印（Support-less Printing）
- 通过动态调整喷嘴方向实现无支撑打印；示例：涡轮扇叶、风扇叶片；
- 优点：
  - 无需支撑材料；
  - 打印时间更短；
  - 表面更平滑。

![ 2025-10-19 110816.png](https://s2.loli.net/2025/10/19/qIdayfr6Nl2CuL8.png)

### 2. 曲面沉积（Conformal Extrusion）
- 可在已有结构上再打印曲面层；
- 例如：
  - 在普通 PLA 基底上打印导电 PLA；
  - 在溶解性 PVA 上打印悬空结构；
  - 实现功能性打印，如 球面 LED 电路，解决了传统分层打印中导电路径在倾斜面上阻抗过高的问题。

![ 2025-10-19 110836.png](https://s2.loli.net/2025/10/19/epF92UthLXbkTw4.png)

### 3. 结构强化（Structural Strengthening）
- 通过沿受力方向进行曲面沉积，提高各向同性强度；
- 桥梁模型实验表明：
  - 混合“平面层 + 曲面层”结构在多方向载荷下更稳定；
  - 显微图像显示层间粘结更紧密。
 
![ 2025-10-19 110851.png](https://s2.loli.net/2025/10/19/4T8wcryNuYbjMVq.png)

## 后续 TODO
1. 曲面切片软件研发，open5x 开源代码中只包含了切片结果，作者自身也没有解决自动化曲面切片算法的问题。
2. 目前论文中采用更换不同直径的打印床以减少喷嘴碰撞风险，后续曲面切片算法研发过程中，需要重点解决碰撞问题。
3. G-code 生成时，和 3-DOF 笛卡尔坐标打印系统不同，5-DOF 打印系统需要多轴联动时精确计算各轴运动距离与喷嘴速度匹配，需要计算运动的欧几里得距离： $d=\sqrt{\Delta x^2+\Delta y^2+\Delta z^2+\Delta u^2+\Delta v^2+\Delta e^2}$ 。

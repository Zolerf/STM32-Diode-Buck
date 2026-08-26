# Buck 功率部分参数计算与选型

本文档记录非同步 Buck（二极管续流）功率级主要元器件的参数计算过程与选型结论。

---

## 1. 设计已知条件

| 参数 | 符号 | 取值 |
|------|------|------|
| 输入电压 | $V_i$ | 12V |
| 输出电压 | $V_o$ | 5V |
| 最大负载电阻 | $R$ | 5Ω |
| 最大输出电流 | $I_o$ | 1A |
| 开关频率 | $f$ | 100kHz（由 STM32 PWM 输出） |

---

## 2. 开关频率 $f$ 的选择

开关频率由单片机输出，其选择需要权衡多方面因素：

- **频率高的优势**：电感、电容等储能元件可以做小，有利于减小电源体积；同时输出纹波电压更小，电源质量更好。
- **频率高的劣势**：MOSFET 开关损耗增加，导致发热、效率降低；同时会带来更严重的电磁干扰（EMI）问题。
- **频率低的优势**：开关损耗降低，效率更高，发热更小，EMI 问题相对容易处理。
- **频率低的劣势**：所需电感、电容的体积和容量/感量增大，成本和 PCB 占用空间增加，纹波电压可能变大。

综合考虑，本设计取 **$f = 100\text{kHz}$**，在体积、效率与 EMI 之间取得平衡。

---

## 3. 占空比 $D$ 的计算

占空比计算的核心是电感的伏秒平衡原理。

设开关导通时间为 $t_1$，关断时间为 $t_2$，开关周期 $T = t_1 + t_2$。

根据电感电压公式：

$$U_L = L \frac{di}{dt}$$

导通期间电感两端电压为：

$$U_{on} = V_i - V_o$$

关断期间电感两端电压（以导通方向为参考）为：

$$U_{off} = V_o + V_d$$

由伏秒平衡原理（导通期间电感磁通增量等于关断期间磁通减量）：

$$U_{on} \times t_1 = U_{off} \times t_2$$

联立以上关系，可得：

$$\begin{cases}
t_1 = \dfrac{V_o + V_d}{V_i + V_d} \cdot T \\[8pt]
t_2 = \dfrac{V_i - V_o}{V_i + V_d} \cdot T \\[8pt]
D = \dfrac{t_1}{T} = \dfrac{V_o + V_d}{V_i + V_d}
\end{cases}$$

其中 $V_d$ 为续流二极管正向导通压降。

---

## 4. 电感 $L$ 的选型

电感有两个关键指标：**感量 $L$** 和 **饱和电流 $I_{sat}$**。实际流过电感的瞬时峰值电流必须低于 $I_{sat}$ 并留有余量，否则磁芯饱和会导致电感失效。

### 4.1 纹波电流 $\Delta I_L$ 与电感感量的关系

Buck 电路中电感电流呈现周期性三角波，其峰峰值纹波电流 $\Delta I_L$ 等于导通期间电流的增量，由伏秒定律决定：

$$\Delta I_L = \frac{U_{on}}{L} \times t_1 = \frac{V_i - V_o}{L} \times \frac{V_o + V_d}{V_i + V_d} \times \frac{1}{f}$$

整理得：

$$\Delta I_L = \frac{V_o + V_d}{Lf} \times \frac{V_i - V_o}{V_i + V_d}$$

由上式可知，$\Delta I_L$ 与 $L$ 和 $f$ 成反比，增大电感量或提高开关频率均可降低纹波电流。

### 4.2 电感感量计算

工程上，纹波电流 $\Delta I_L$ 通常取输出电流 $I_o$ 的 **20%~40%**。

$$L = \frac{V_o + V_d}{\Delta I_L \cdot f} \times \frac{V_i - V_o}{V_i + V_d}$$

代入 $V_i = 12\text{V}$，$V_o = 5\text{V}$，$f = 100\text{kHz}$，$\Delta I_L = 0.2I_o \sim 0.4I_o$，且忽略续流二极管压降 $V_d$：

$$L = \frac{5}{(0.2 \sim 0.4) \times 100 \times 10^3} \times \frac{12 - 5}{12}$$

计算得：

$$L \approx 72.9\mu\text{H} \sim 145.8\mu\text{H}$$

### 4.3 饱和电流计算

电感峰值电流：

$$I_{L\max} = I_o + \frac{\Delta I_L}{2}$$

因此，电感选型要求：

$$L \geq 100\mu\text{H} \quad \text{（取经验中值）}$$

$$I_{sat} \geq 1.5\text{A}$$

---

## 5. 续流二极管选型

续流二极管选型需考虑四个指标：

| 指标 | 设计要求 | 说明 |
|------|----------|------|
| **耐压** | $\geq 1.5 \sim 2$ 倍 $V_i$，即 $\geq 24\text{V}$ | 实际选 40V 或 60V 肖特基，以承受开关尖峰 |
| **电流** | 平均电流约 $0.58\text{A}$，峰值约 $1.2\text{A}$ | 额定电流选 $1.5\text{A} \sim 2\text{A}$，留足余量 |
| **速度** | 必须为快恢复或肖特基 | 避免反向恢复导致短路烧毁 |
| **压降** | 尽量小 | 减小导通损耗，提高效率 |

**选型结论**：选用 **SS34 肖特基二极管**（40V/3A，压降低，反向恢复时间极短，适合高频开关）。

---

## 6. 输入电容选型

输入电容的电压纹波由两部分组成：

- **$U_q$**：由电容充放电（电荷变化）引起的电压波动
- **$U_{esr}$**：由等效串联电阻（ESR）上的压降产生

$$\Delta V_i = U_q + U_{esr}$$

### 6.1 电荷引起的纹波分量 $U_q$

由能量守恒（忽略 MOS 管损耗），输入功率等于负载功率与二极管损耗功率之和：

$$V_i \times I_i = V_o \times I_o + V_d \times I_o \times \frac{V_i - V_o}{V_i + V_d}$$

整理得：

$$I_i = \frac{I_o \left( V_o + V_d \cdot \dfrac{V_i - V_o}{V_i + V_d} \right)}{V_i}$$

一个周期内，电容充电（开关关断期间）的电荷量为：

$$Q = I_i \times t_2 = \frac{I_o \left( V_o + V_d \cdot \dfrac{V_i - V_o}{V_i + V_d} \right)}{V_i} \times \frac{V_i - V_o}{V_i + V_d} \times \frac{1}{f}$$

因此：

$$U_q = \frac{Q}{C_i} = \frac{I_o \left( V_o + V_d \cdot \dfrac{V_i - V_o}{V_i + V_d} \right)}{V_i \cdot f \cdot C_i} \times \frac{V_i - V_o}{V_i + V_d}$$

### 6.2 ESR 引起的纹波分量 $U_{esr}$

- **充电阶段**：流过 ESR 的电流为输入电流 $I_i$
- **放电阶段**：流过 ESR 的电流为 $I_o + \dfrac{\Delta I_L}{2}$

取电流较大者，ESR 压降为：

$$U_{esr} = \left( I_o + \frac{\Delta I_L}{2} \right) \times ESR$$

其中：

$$\Delta I_L = \frac{V_o + V_d}{Lf} \times \frac{V_i - V_o}{V_i + V_d}$$

### 6.3 综合表达式

$$\begin{cases}
U_q = \dfrac{I_o \left( V_o + V_d \cdot \dfrac{V_i - V_o}{V_i + V_d} \right)}{V_i \cdot f \cdot C_i} \times \dfrac{V_i - V_o}{V_i + V_d} \\[10pt]
U_{esr} = \left( I_o + \dfrac{V_o + V_d}{2Lf} \times \dfrac{V_i - V_o}{V_i + V_d} \right) \times ESR \\[10pt]
\Delta V_i = U_q + U_{esr}
\end{cases}$$

工程上，$\Delta V_i$ 通常取输入电压的 **1%~2%**。

- 若使用 **陶瓷电容**（ESR 极小），$U_{esr}$ 可忽略，按 $U_q \leq \Delta V_i$ 选型
- 若使用 **电解电容**（ESR 较大），$U_{esr}$ 占纹波主要部分，按 $U_{esr} \leq \Delta V_i$ 选型

---

## 7. 输出电容选型

### 7.1 电荷引起的纹波分量 $U_{qo}$

输出端充放电电流为电感纹波电流 $\Delta I_L$，呈三角波。一个周期内电容充电电荷量为：

$$Q_o = \frac{\Delta I_L}{2} \times \frac{T}{2} \times \frac{1}{2} = \frac{\Delta I_L}{8f}$$

因此：

$$U_{qo} = \frac{Q_o}{C_o} = \frac{\Delta I_L}{8f C_o} = \frac{V_o + V_d}{8f^2 L C_o} \times \frac{V_i - V_o}{V_i + V_d}$$

### 7.2 ESR 引起的纹波分量 $U_{esro}$

输出端 ESR 上流过的电流变化量为 $\Delta I_L$，因此：

$$U_{esro} = \Delta I_L \times ESR = \frac{V_o + V_d}{Lf} \times \frac{V_i - V_o}{V_i + V_d} \times ESR$$

### 7.3 综合表达式

$$\begin{cases}
U_{qo} = \dfrac{V_o + V_d}{8f^2 L C_o} \times \dfrac{V_i - V_o}{V_i + V_d} \\[10pt]
U_{esro} = \dfrac{V_o + V_d}{Lf} \times \dfrac{V_i - V_o}{V_i + V_d} \times ESR \\[10pt]
\Delta V_o = U_{qo} + U_{esro}
\end{cases}$$

$\Delta V_o$ 通常取输出电压的 **0.5%~2%**。选型判据与输入电容相同。

---

## 8. 陶瓷电容与电解电容的说明

| 特性 | 陶瓷电容 | 电解电容 |
|------|----------|----------|
| ESR | 极低 | 较大 |
| ESL | 极低 | 较大 |
| 高频响应 | 好，能有效吸收尖峰 | 差 |
| 纹波/发热 | 小 | 较大 |
| 容量 | 较小 | 较大 |
| 寿命 | 长 | 有限 |
| 主要作用 | 高频滤波、吸收尖峰 | 储能、低频滤波、抑制阻尼振荡 |

在实际工程中，通常采用**电解电容 + 陶瓷电容**搭配使用。

---

## 9. 工程简化选型公式

### 输入电解电容：

$$C_i \geq \frac{I_o \times D \times (1 - D)}{f \times \Delta V_i}$$

### 输出电解电容：

$$C_o \geq \frac{\Delta I_L}{8 \times f \times \Delta V_o}$$

其中：

$$\Delta I_L = \frac{V_o + V_d}{L} \times \frac{1 - D}{f}$$

### ESR 约束：

$$ESR \leq \frac{\Delta V_{o(ESR)}}{\Delta I_L}$$

其中 $\Delta V_{o(ESR)}$ 为分配给 ESR 的纹波预算，通常取总纹波的 **30%~50%**。

### 高频陶瓷电容：

$$C_t \geq \frac{I_{\max} \times t_{up}}{\Delta V_{\max}}$$

其中：
- $I_{\max}$：电感尖峰电流
- $\Delta V_{\max}$：允许的尖峰电压幅值，约 0.5V~1V
- $t_{up}$：开关管上升时间，约 5ns~20ns

---

## 10. MOS 管选型要求

| 选型指标 | 要求 |
|----------|------|
| **耐压** | $\geq 1.5 \sim 2$ 倍 $V_i$，防止开关尖峰击穿 |
| **电流能力** | $\geq 1.5 \sim 2$ 倍 $I_{L\max}$ |
| **导通电阻 $R_{ds(on)}$** | 尽量小，降低导通损耗 |
| **栅极电荷 $Q_g$** | 尽量小，提高开关速度，降低开关损耗 |
| **栅极阈值电压 $V_{gs(th)}$** | 需与驱动电压匹配（本设计驱动电压 12V，选增强型 N-MOS） |

---

## 11. 最终元器件选型清单

### 功率开关管

| 型号 | 主要参数 |
|------|----------|
| **NCEP0135A** | $V_{DS}=100\text{V}$，$I_D$ 满足设计要求 |

### 续流二极管

| 型号 | 主要参数 |
|------|----------|
| **SS34** | 肖特基，40V/3A，压降低，反向恢复快 |

### 方案一：全陶瓷电容方案

| 位置 | 型号/规格 | 数量 |
|------|-----------|------|
| 输入电容 | 22μF / 50V（陶瓷，226/50V） | 2 颗（并联） |
| 输出电容 | 10μF / 50V（陶瓷，106/50V） | 2 颗（并联） |

### 方案二：电解 + 陶瓷搭配方案

| 位置 | 型号/规格 | 数量 |
|------|-----------|------|
| 输入电解电容 | 47μF / 50V | 1 颗 |
| 输入陶瓷电容 | 100nF / 50V | 1 颗 |
| 输出电解电容 | 47μF / 50V | 1 颗 |
| 输出陶瓷电容 | 100nF / 50V | 1 颗 |

> 两个方案可分别制作 PCB 进行对比测试，验证陶瓷电容与电解电容在实际纹波抑制和动态响应方面的差异。

---

## 附录：关键公式汇总

### 占空比：

$$D = \frac{V_o + V_d}{V_i + V_d}$$

### 电感感量：

$$L = \frac{V_o + V_d}{\Delta I_L \cdot f} \times \frac{V_i - V_o}{V_i + V_d}$$

### 输入电容纹波：

$$\begin{cases}
U_q = \dfrac{I_o \left( V_o + V_d \cdot \dfrac{V_i - V_o}{V_i + V_d} \right)}{V_i \cdot f \cdot C_i} \times \dfrac{V_i - V_o}{V_i + V_d} \\[10pt]
U_{esr} = \left( I_o + \dfrac{V_o + V_d}{2Lf} \times \dfrac{V_i - V_o}{V_i + V_d} \right) \times ESR \\[10pt]
\Delta V_i = U_q + U_{esr}
\end{cases}$$

### 输出电容纹波：

$$\begin{cases}
U_{qo} = \dfrac{V_o + V_d}{8f^2 L C_o} \times \dfrac{V_i - V_o}{V_i + V_d} \\[10pt]
U_{esro} = \dfrac{V_o + V_d}{Lf} \times \dfrac{V_i - V_o}{V_i + V_d} \times ESR \\[10pt]
\Delta V_o = U_{qo} + U_{esro}
\end{cases}$$

### 电解电容简化选型：

$$C_i \geq \frac{I_o \times D \times (1 - D)}{f \times \Delta V_i}$$

$$C_o \geq \frac{\Delta I_L}{8 \times f \times \Delta V_o}$$

$$ESR \leq \frac{\Delta V_{o(ESR)}}{\Delta I_L}$$

### 高频陶瓷电容选型：

$$C_t \geq \frac{I_{\max} \times t_{up}}{\Delta V_{\max}}$$

---

**文档版本**：v1.0  
**最后更新**：2026-08-26
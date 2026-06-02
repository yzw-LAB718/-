# 无 OBSS 干扰场景下的 SNR 计算

本文档补充说明在当前最小 Wi-Fi Mesh 场景中，**不考虑 OBSS 外部干扰**时，SNR 应如何由 ns-3 参数和理论公式计算。

> 显示说明：本文档已改用 GitHub 原生 LaTeX 数学公式，不再依赖 `latex.codecogs.com` 外链 SVG 图片，避免出现公式图片加载失败、只显示 `alt` 文本的问题。

当前只考虑：

```text
1 个 ONT + 1 个 AP + 1 个 STA
AP 与 ONT 之间有线连接
数据方向为下行 TCP
无 OBSS 干扰
无多 STA 竞争
```

由于当前业务方向是下行，因此需要计算的是 STA 端接收来自 ONT 或 AP 的信号时的 SNR：

```text
STA 接入 ONT：ONT → STA
STA 接入 AP：ONT → AP → STA，其中 AP → STA 是无线段
```

---

## 1. 计算链条

无 OBSS 干扰时，SNR 的计算链条为：

$$
d_{\mathrm{STA}-j}\rightarrow PL_j(d_{\mathrm{STA}-j})\rightarrow P_{r,j}(d_{\mathrm{STA}-j})\rightarrow N_{\mathrm{STA}}\rightarrow SNR_j(d_{\mathrm{STA}-j}).
$$

其中：

$$
j\in\{\mathrm{ONT},\mathrm{AP}\}.
$$

---

## 2. 距离计算

在当前最小场景中，典型坐标为：

| 节点 | 坐标 |
|---|---:|
| ONT | `(0, 0)` |
| AP | `(10, 0)` |
| STA | `(x_STA, y_STA)` |

因此：

$$
d_{\mathrm{STA-ONT}}=\sqrt{x_{\mathrm{STA}}^2+y_{\mathrm{STA}}^2},
$$

$$
d_{\mathrm{STA-AP}}=\sqrt{(x_{\mathrm{STA}}-10)^2+y_{\mathrm{STA}}^2}.
$$

实际计算路径损耗时，建议使用有效距离：

$$
d_{\mathrm{eff}}=\max(d,d_0),\qquad d_0=1\ \mathrm{m},
$$

避免 $d=0$ 或近场距离导致 $\log_{10}(d)$ 无定义或路径损耗不合理。

---

## 3. 路径损耗

当前代码使用：

```cpp
chHelper.AddPropagationLoss("ns3::LogDistancePropagationLossModel");
```

如果没有在代码中显式设置 `Exponent`、`ReferenceDistance` 和 `ReferenceLoss`，可按 ns-3 默认对数距离模型进行理论近似：

$$
PL_j(d_{\mathrm{STA}-j})=L_0+10n\log_{10}\!\left(\frac{d_{\mathrm{STA}-j}}{d_0}\right).
$$

其中默认可取：

| 参数 | 取值 | 含义 |
|---|---:|---|
| $L_0$ | `46.6777 dB` | 参考距离处路径损耗 |
| $n$ | `3` | 路径损耗指数 |
| $d_0$ | `1 m` | 参考距离 |

因此：

$$
PL_j(d_{\mathrm{STA}-j})=46.6777+30\log_{10}(d_{\mathrm{STA}-j}).
$$

这里距离单位为米，路径损耗单位为 dB。

---

## 4. 接收功率

一般接收功率为：

$$
P_{r,j}(d_{\mathrm{STA}-j})=P_{t,j}+G_{t,j}+G_{r,\mathrm{STA}}-PL_j(d_{\mathrm{STA}-j}).
$$

当前代码显式设置：

```cpp
sharedPhy.Set("TxPowerStart", DoubleValue(g_txPowerDbm));
sharedPhy.Set("TxPowerEnd", DoubleValue(g_txPowerDbm));
```

且：

$$
P_t=20\ \mathrm{dBm}.
$$

代码未显式设置 `TxGain` 和 `RxGain`，理论近似中可取：

$$
G_t=0\ \mathrm{dB},\qquad G_r=0\ \mathrm{dB}.
$$

所以：

$$
P_{r,j}(d_{\mathrm{STA}-j})=20-PL_j(d_{\mathrm{STA}-j}).
$$

代入路径损耗后：

$$
P_{r,j}(d_{\mathrm{STA}-j})=-26.6777-30\log_{10}(d_{\mathrm{STA}-j}).
$$

单位为 dBm。

---

## 5. 噪声功率

无 OBSS 干扰时，只需要考虑热噪声和接收机噪声系数。噪声功率为：

$$
N_{\mathrm{STA}}=-174+10\log_{10}(B)+NF_{\mathrm{STA}}.
$$

其中 `-174 dBm/Hz` 是室温下的热噪声功率谱密度，$B$ 需要使用 Hz。

当前代码中：

$$
B=160\ \mathrm{MHz}=160\times10^6\ \mathrm{Hz}.
$$

若使用 ns-3 默认接收机噪声系数：

$$
NF_{\mathrm{STA}}=7\ \mathrm{dB},
$$

则：

$$
N_{\mathrm{STA}}=-174+10\log_{10}(160\times10^6)+7\approx-84.96\ \mathrm{dBm}.
$$

如果暂时不考虑噪声系数，即 $NF=0$，则噪声功率约为：

$$
N_{\mathrm{STA}}\approx-91.96\ \mathrm{dBm}.
$$

但为了更贴近 ns-3 默认行为，本文档建议使用：

$$
N_{\mathrm{STA}}\approx-84.96\ \mathrm{dBm}.
$$

---

## 6. 无 OBSS 干扰下的 SNR

无外部干扰时，SNR 为：

$$
SNR_j(d_{\mathrm{STA}-j})=P_{r,j}(d_{\mathrm{STA}-j})-N_{\mathrm{STA}}.
$$

使用上面的默认参数，得到：

$$
SNR_j(d_{\mathrm{STA}-j})\approx58.28-30\log_{10}(d_{\mathrm{STA}-j}).
$$

单位为 dB。

因此：

$$
SNR_{\mathrm{ONT}}\approx58.28-30\log_{10}(d_{\mathrm{STA-ONT}}),
$$

$$
SNR_{\mathrm{AP}}\approx58.28-30\log_{10}(d_{\mathrm{STA-AP}}).
$$

进一步写成 STA 坐标形式：

$$
SNR_{\mathrm{ONT}}\approx58.28-30\log_{10}\!\left(\sqrt{x_{\mathrm{STA}}^2+y_{\mathrm{STA}}^2}\right),
$$

$$
SNR_{\mathrm{AP}}\approx58.28-30\log_{10}\!\left(\sqrt{(x_{\mathrm{STA}}-10)^2+y_{\mathrm{STA}}^2}\right).
$$

---

## 7. 数值例子

采用：

```text
P_t = 20 dBm
TxGain = 0 dB
RxGain = 0 dB
ReferenceLoss = 46.6777 dB
Exponent = 3
ReferenceDistance = 1 m
B = 160 MHz
RxNoiseFigure = 7 dB
```

### 7.1 STA 在 `(5, 0)`

此时：

$$
d_{\mathrm{STA-ONT}}=d_{\mathrm{STA-AP}}=5\ \mathrm{m}.
$$

路径损耗：

$$
PL(5)=46.6777+30\log_{10}(5)\approx67.65\ \mathrm{dB}.
$$

接收功率：

$$
P_r(5)=20-67.65=-47.65\ \mathrm{dBm}.
$$

SNR：

$$
SNR(5)=-47.65-(-84.96)=37.31\ \mathrm{dB}.
$$

因此 STA 在 `(5,0)` 时，ONT 和 AP 的 SNR 相同，约为 `37.31 dB`。

### 7.2 STA 在 `(8, 0)`

此时：

$$
d_{\mathrm{STA-ONT}}=8\ \mathrm{m},\qquad d_{\mathrm{STA-AP}}=2\ \mathrm{m}.
$$

ONT 链路：

$$
SNR_{\mathrm{ONT}}(8)\approx31.19\ \mathrm{dB}.
$$

AP 链路：

$$
SNR_{\mathrm{AP}}(2)\approx49.25\ \mathrm{dB}.
$$

因此 STA 在 `(8,0)` 时更靠近 AP，AP 链路 SNR 明显更高。

---

## 8. 与 MCS、PHY rate、吞吐的关系

SNR 不直接等于吞吐，而是通过 MCS 和 PHY rate 间接影响吞吐：

$$
SNR_j\rightarrow MCS_j\rightarrow R_j\rightarrow S_j^{\mathrm{agg}}.
$$

在当前代码中：

```cpp
wifi.SetRemoteStationManager("ns3::IdealWifiManager");
```

因此 MCS / PHY rate 由 `IdealWifiManager` 根据链路条件自动选择。理论上可写成：

$$
R_j(d_{\mathrm{STA}-j})=R_{MCS_j(SNR_j(d_{\mathrm{STA}-j}))}(160\ \mathrm{MHz},2,800\ \mathrm{ns}).
$$

也就是说，距离改变会先改变 SNR，再改变 MCS 和 PHY rate，最终影响多帧聚合吞吐。

---

## 9. 与 OBSS 场景的区别

本文档只讨论无 OBSS 干扰，因此使用：

$$
SNR_j=P_{r,j}-N_{\mathrm{STA}}.
$$

如果后续打开 OBSS，则需要改用 SINR。由于噪声功率和干扰功率都以 dBm 表示，计算时需要先转换到线性域相加，再转换回 dB 域：

$$
SINR_j=P_{r,j}-10\log_{10}\!\left(10^{N_{\mathrm{STA}}/10}+10^{I_j/10}\right).
$$

其中 $I_j$ 表示干扰功率。注意不能直接写成 `Pr - N - I`，因为噪声和干扰都是功率，必须先在线性域相加。

---

## 10. 建议在 ns-3 中显式设置的参数

为了让理论公式与 ns-3 配置更容易对齐，建议在后续代码中显式写出：

```cpp
sharedPhy.Set("RxNoiseFigure", DoubleValue(7.0));
sharedPhy.Set("TxGain", DoubleValue(0.0));
sharedPhy.Set("RxGain", DoubleValue(0.0));
```

并将路径损耗参数也显式写出：

```cpp
chHelper.AddPropagationLoss(
  "ns3::LogDistancePropagationLossModel",
  "Exponent", DoubleValue(3.0),
  "ReferenceDistance", DoubleValue(1.0),
  "ReferenceLoss", DoubleValue(46.6777)
);
```

这样理论公式：

$$
PL(d)=46.6777+30\log_{10}(d)
$$

就能和仿真参数保持一致。

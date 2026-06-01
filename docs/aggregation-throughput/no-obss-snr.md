# 无 OBSS 干扰场景下的 SNR 计算

本文档补充说明在当前最小 Wi-Fi Mesh 场景中，**不考虑 OBSS 外部干扰**时，SNR 应如何由 ns-3 参数和理论公式计算。为避免 GitHub 中 LaTeX 不渲染，关键公式均使用 SVG 公式图片。

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

<p align="center"><img alt="snr chain" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20d_%7B%5Cmathrm%7BSTA%7D-j%7D%5Crightarrow%20PL_j%28d_%7B%5Cmathrm%7BSTA%7D-j%7D%29%5Crightarrow%20P_%7Br%2Cj%7D%28d_%7B%5Cmathrm%7BSTA%7D-j%7D%29%5Crightarrow%20N_%7B%5Cmathrm%7BSTA%7D%7D%5Crightarrow%20SNR_j%28d_%7B%5Cmathrm%7BSTA%7D-j%7D%29" /></p>

其中：

<p align="center"><img alt="j set" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20j%5Cin%5C%7B%5Cmathrm%7BONT%7D%2C%5Cmathrm%7BAP%7D%5C%7D" /></p>

---

## 2. 距离计算

在当前最小场景中，典型坐标为：

| 节点 | 坐标 |
|---|---:|
| ONT | `(0, 0)` |
| AP | `(10, 0)` |
| STA | `(x_STA, y_STA)` |

因此：

<p align="center"><img alt="d sta ont" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20d_%7B%5Cmathrm%7BSTA-ONT%7D%7D%3D%5Csqrt%7Bx_%7B%5Cmathrm%7BSTA%7D%7D%5E2%2By_%7B%5Cmathrm%7BSTA%7D%7D%5E2%7D" /></p>

<p align="center"><img alt="d sta ap" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20d_%7B%5Cmathrm%7BSTA-AP%7D%7D%3D%5Csqrt%7B%28x_%7B%5Cmathrm%7BSTA%7D%7D-10%29%5E2%2By_%7B%5Cmathrm%7BSTA%7D%7D%5E2%7D" /></p>

---

## 3. 路径损耗

当前代码使用：

```cpp
chHelper.AddPropagationLoss("ns3::LogDistancePropagationLossModel");
```

如果没有在代码中显式设置 `Exponent`、`ReferenceDistance` 和 `ReferenceLoss`，可按 ns-3 默认对数距离模型进行理论近似：

<p align="center"><img alt="path loss default" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20PL_j%28d_%7B%5Cmathrm%7BSTA%7D-j%7D%29%3DL_0%2B10n%5Clog_%7B10%7D%5Cleft%28%5Cfrac%7Bd_%7B%5Cmathrm%7BSTA%7D-j%7D%7D%7Bd_0%7D%5Cright%29" /></p>

其中默认可取：

| 参数 | 取值 | 含义 |
|---|---:|---|
| <img alt="L0" src="https://latex.codecogs.com/svg.image?L_0" /> | `46.6777 dB` | 参考距离处路径损耗 |
| <img alt="n" src="https://latex.codecogs.com/svg.image?n" /> | `3` | 路径损耗指数 |
| <img alt="d0" src="https://latex.codecogs.com/svg.image?d_0" /> | `1 m` | 参考距离 |

因此：

<p align="center"><img alt="path loss simplified" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20PL_j%28d_%7B%5Cmathrm%7BSTA%7D-j%7D%29%3D46.6777%2B30%5Clog_%7B10%7D%28d_%7B%5Cmathrm%7BSTA%7D-j%7D%29" /></p>

这里距离单位为米，路径损耗单位为 dB。

---

## 4. 接收功率

一般接收功率为：

<p align="center"><img alt="received power general" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20P_%7Br%2Cj%7D%28d_%7B%5Cmathrm%7BSTA%7D-j%7D%29%3DP_%7Bt%2Cj%7D%2BG_%7Bt%2Cj%7D%2BG_%7Br%2C%5Cmathrm%7BSTA%7D%7D-PL_j%28d_%7B%5Cmathrm%7BSTA%7D-j%7D%29" /></p>

当前代码显式设置：

```cpp
sharedPhy.Set("TxPowerStart", DoubleValue(g_txPowerDbm));
sharedPhy.Set("TxPowerEnd", DoubleValue(g_txPowerDbm));
```

且：

<p align="center"><img alt="Pt value" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20P_t%3D20%5C%20%5Cmathrm%7BdBm%7D" /></p>

代码未显式设置 `TxGain` 和 `RxGain`，理论近似中可取：

<p align="center"><img alt="gain zero" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20G_t%3D0%5C%20%5Cmathrm%7BdB%7D%2C%5Cqquad%20G_r%3D0%5C%20%5Cmathrm%7BdB%7D" /></p>

所以：

<p align="center"><img alt="received power simplified" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20P_%7Br%2Cj%7D%28d_%7B%5Cmathrm%7BSTA%7D-j%7D%29%3D20-PL_j%28d_%7B%5Cmathrm%7BSTA%7D-j%7D%29" /></p>

代入路径损耗后：

<p align="center"><img alt="received power final" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20P_%7Br%2Cj%7D%28d_%7B%5Cmathrm%7BSTA%7D-j%7D%29%3D-26.6777-30%5Clog_%7B10%7D%28d_%7B%5Cmathrm%7BSTA%7D-j%7D%29" /></p>

单位为 dBm。

---

## 5. 噪声功率

无 OBSS 干扰时，只需要考虑热噪声和接收机噪声系数。噪声功率为：

<p align="center"><img alt="noise power" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20N_%7B%5Cmathrm%7BSTA%7D%7D%3D-174%2B10%5Clog_%7B10%7D%28B%29%2BNF_%7B%5Cmathrm%7BSTA%7D%7D" /></p>

其中 `-174 dBm/Hz` 是室温下的热噪声功率谱密度，<img alt="B" src="https://latex.codecogs.com/svg.image?B" /> 需要使用 Hz。

当前代码中：

<p align="center"><img alt="B value" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20B%3D160%5C%20%5Cmathrm%7BMHz%7D%3D160%5Ctimes10%5E6%5C%20%5Cmathrm%7BHz%7D" /></p>

若使用 ns-3 默认接收机噪声系数：

<p align="center"><img alt="NF value" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20NF_%7B%5Cmathrm%7BSTA%7D%7D%3D7%5C%20%5Cmathrm%7BdB%7D" /></p>

则：

<p align="center"><img alt="noise final" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20N_%7B%5Cmathrm%7BSTA%7D%7D%3D-174%2B10%5Clog_%7B10%7D%28160%5Ctimes10%5E6%29%2B7%5Capprox-84.96%5C%20%5Cmathrm%7BdBm%7D" /></p>

如果暂时不考虑噪声系数，即 <img alt="NF zero" src="https://latex.codecogs.com/svg.image?NF%3D0" />，则噪声功率约为：

<p align="center"><img alt="noise NF zero" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20N_%7B%5Cmathrm%7BSTA%7D%7D%5Capprox-91.96%5C%20%5Cmathrm%7BdBm%7D" /></p>

但为了更贴近 ns-3 默认行为，本文档建议使用：

<p align="center"><img alt="noise recommended" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20N_%7B%5Cmathrm%7BSTA%7D%7D%5Capprox-84.96%5C%20%5Cmathrm%7BdBm%7D" /></p>

---

## 6. 无 OBSS 干扰下的 SNR

无外部干扰时，SNR 为：

<p align="center"><img alt="snr definition" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20SNR_j%28d_%7B%5Cmathrm%7BSTA%7D-j%7D%29%3DP_%7Br%2Cj%7D%28d_%7B%5Cmathrm%7BSTA%7D-j%7D%29-N_%7B%5Cmathrm%7BSTA%7D%7D" /></p>

使用上面的默认参数，得到：

<p align="center"><img alt="snr final" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20SNR_j%28d_%7B%5Cmathrm%7BSTA%7D-j%7D%29%5Capprox58.28-30%5Clog_%7B10%7D%28d_%7B%5Cmathrm%7BSTA%7D-j%7D%29" /></p>

单位为 dB。

因此：

<p align="center"><img alt="snr ont" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20SNR_%7B%5Cmathrm%7BONT%7D%7D%5Capprox58.28-30%5Clog_%7B10%7D%28d_%7B%5Cmathrm%7BSTA-ONT%7D%7D%29" /></p>

<p align="center"><img alt="snr ap" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20SNR_%7B%5Cmathrm%7BAP%7D%7D%5Capprox58.28-30%5Clog_%7B10%7D%28d_%7B%5Cmathrm%7BSTA-AP%7D%7D%29" /></p>

进一步写成 STA 坐标形式：

<p align="center"><img alt="snr ont coord" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20SNR_%7B%5Cmathrm%7BONT%7D%7D%5Capprox58.28-30%5Clog_%7B10%7D%5Cleft%28%5Csqrt%7Bx_%7B%5Cmathrm%7BSTA%7D%7D%5E2%2By_%7B%5Cmathrm%7BSTA%7D%7D%5E2%7D%5Cright%29" /></p>

<p align="center"><img alt="snr ap coord" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20SNR_%7B%5Cmathrm%7BAP%7D%7D%5Capprox58.28-30%5Clog_%7B10%7D%5Cleft%28%5Csqrt%7B%28x_%7B%5Cmathrm%7BSTA%7D%7D-10%29%5E2%2By_%7B%5Cmathrm%7BSTA%7D%7D%5E2%7D%5Cright%29" /></p>

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

<p align="center"><img alt="d both 5" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20d_%7B%5Cmathrm%7BSTA-ONT%7D%7D%3Dd_%7B%5Cmathrm%7BSTA-AP%7D%7D%3D5%5C%20%5Cmathrm%7Bm%7D" /></p>

路径损耗：

<p align="center"><img alt="PL 5" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20PL%285%29%3D46.6777%2B30%5Clog_%7B10%7D%285%29%5Capprox67.65%5C%20%5Cmathrm%7BdB%7D" /></p>

接收功率：

<p align="center"><img alt="Pr 5" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20P_r%285%29%3D20-67.65%3D-47.65%5C%20%5Cmathrm%7BdBm%7D" /></p>

SNR：

<p align="center"><img alt="SNR 5" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20SNR%285%29%3D-47.65-%28-84.96%29%3D37.31%5C%20%5Cmathrm%7BdB%7D" /></p>

因此 STA 在 `(5,0)` 时，ONT 和 AP 的 SNR 相同，约为 `37.31 dB`。

### 7.2 STA 在 `(8, 0)`

此时：

<p align="center"><img alt="d 8 2" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20d_%7B%5Cmathrm%7BSTA-ONT%7D%7D%3D8%5C%20%5Cmathrm%7Bm%7D%2C%5Cqquad%20d_%7B%5Cmathrm%7BSTA-AP%7D%7D%3D2%5C%20%5Cmathrm%7Bm%7D" /></p>

ONT 链路：

<p align="center"><img alt="SNR ONT 8" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20SNR_%7B%5Cmathrm%7BONT%7D%7D%288%29%5Capprox31.19%5C%20%5Cmathrm%7BdB%7D" /></p>

AP 链路：

<p align="center"><img alt="SNR AP 2" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20SNR_%7B%5Cmathrm%7BAP%7D%7D%282%29%5Capprox49.25%5C%20%5Cmathrm%7BdB%7D" /></p>

因此 STA 在 `(8,0)` 时更靠近 AP，AP 链路 SNR 明显更高。

---

## 8. 与 MCS、PHY rate、吞吐的关系

SNR 不直接等于吞吐，而是通过 MCS 和 PHY rate 间接影响吞吐：

<p align="center"><img alt="snr mcs rate throughput" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20SNR_j%5Crightarrow%20MCS_j%5Crightarrow%20R_j%5Crightarrow%20S_j%5E%7B%5Cmathrm%7Bagg%7D%7D" /></p>

在当前代码中：

```cpp
wifi.SetRemoteStationManager("ns3::IdealWifiManager");
```

因此 MCS / PHY rate 由 `IdealWifiManager` 根据链路条件自动选择。理论上可写成：

<p align="center"><img alt="rate by snr" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20R_j%28d_%7B%5Cmathrm%7BSTA%7D-j%7D%29%3DR_%7BMCS_j%28SNR_j%28d_%7B%5Cmathrm%7BSTA%7D-j%7D%29%29%7D%5Cleft%28160%5C%20%5Cmathrm%7BMHz%7D%2C2%2C800%5C%20%5Cmathrm%7Bns%7D%5Cright%29" /></p>

也就是说，距离改变会先改变 SNR，再改变 MCS 和 PHY rate，最终影响多帧聚合吞吐。

---

## 9. 与 OBSS 场景的区别

本文档只讨论无 OBSS 干扰，因此使用：

<p align="center"><img alt="snr no obss" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20SNR_j%3DP_%7Br%2Cj%7D-N_%7B%5Cmathrm%7BSTA%7D%7D" /></p>

如果后续打开 OBSS，则需要改用 SINR：

<p align="center"><img alt="sinr" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20SINR_j%3DP_%7Br%2Cj%7D-10%5Clog_%7B10%7D%5Cleft%2810%5E%7BN_%7B%5Cmathrm%7BSTA%7D%2F10%7D%2B10%5E%7BI_j%2F10%7D%5Cright%29" /></p>

其中 <img alt="I_j" src="https://latex.codecogs.com/svg.image?I_j" /> 表示干扰功率。注意不能直接写成 `Pr - N - I`，因为噪声和干扰都是功率，必须先在线性域相加。

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

<p align="center"><img alt="PL explicit" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20PL%28d%29%3D46.6777%2B30%5Clog_%7B10%7D%28d%29" /></p>

就能和仿真参数保持一致。

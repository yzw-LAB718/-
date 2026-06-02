# MCS 选择与 PHY rate 计算说明

本文档说明在当前最小 Wi-Fi Mesh 场景中，已知 SNR 后如何选择 MCS，并进一步计算 PHY rate、有效数据长度和聚合后 PSDU 总长度。

当前讨论的场景为：

```text
1 个 ONT + 1 个 AP + 1 个 STA
数据方向：下行 TCP
STA 接入 ONT：ONT → STA
STA 接入 AP：ONT → AP → STA，其中 AP → STA 是无线段
无 OBSS 干扰
无多 STA 竞争
```

当前代码中的关键 Wi-Fi 配置可概括为：

| 参数 | 当前值 | 说明 |
|---|---:|---|
| Wi-Fi 标准 | `802.11be` | EHT / Wi-Fi 7 |
| 速率控制 | `IdealWifiManager` | 根据链路 SNR 自动选择 MCS |
| 信道带宽 | `160 MHz` | 进入 PHY rate 计算 |
| 空间流数 | `2` | 进入 PHY rate 计算 |
| Guard Interval | `800 ns` | 进入 PHY rate 计算 |
| 发射功率 | `20 dBm` | 进入 SNR 计算 |
| A-MPDU | 开启 | 聚合多个 MPDU |
| A-MSDU | 开启 | 每个 MPDU 内可聚合多个 MSDU |
| 应用包 / MSS | `1500 B` | 作为理论中的 MSDU 有效负载 |

---

## 1. 总体链条

无 OBSS 干扰时，链路速率和吞吐可以按下面链条理解：

<p align="center"><img alt="snr to throughput chain" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20d_%7B%5Cmathrm%7BSTA%7D-j%7D%5Crightarrow%20SNR_j%5Crightarrow%20MCS_j%5Crightarrow%20R_j%5Crightarrow%20S_j%5E%7B%5Cmathrm%7Bagg%7D%7D" /></p>

其中：

<p align="center"><img alt="j set" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20j%5Cin%5C%7B%5Cmathrm%7BONT%7D%2C%5Cmathrm%7BAP%7D%5C%7D" /></p>

前一个文档已经给出无 OBSS 场景下的 SNR 近似表达式：

<p align="center"><img alt="snr no obss" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20SNR_j%28d_%7B%5Cmathrm%7BSTA%7D-j%7D%29%5Capprox58.28-30%5Clog_%7B10%7D%28d_%7B%5Cmathrm%7BSTA%7D-j%7D%29" /></p>

这里的 SNR 单位为 dB，距离单位为 m。

---

## 2. 已知 SNR 后如何选择 MCS

当前代码使用：

```cpp
wifi.SetRemoteStationManager("ns3::IdealWifiManager");
```

因此，MCS 不是人工固定的，而是由 `IdealWifiManager` 根据链路 SNR 自动选择。

理论上可以写成：

<p align="center"><img alt="mcs selection" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20MCS_j%3D%5Cmax%5Cleft%5C%7Bm%3ASNR_j%5Cge%5CGamma_%7B%5Cmathrm%7Bth%7D%2Cm%7D%5Cright%5C%7D" /></p>

其中：

<p align="center"><img alt="gamma th" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20%5CGamma_%7B%5Cmathrm%7Bth%7D%2Cm%7D" /></p>

表示 MCS `m` 对应的最小 SNR 门限，单位为 dB。

`IdealWifiManager` 的默认思想是：对每个候选 MCS，计算它达到目标误码率所需的 SNR 门限，然后在当前 SNR 下选择满足门限的最高速率模式。默认目标 BER 通常可写为：

<p align="center"><img alt="ber threshold" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20BER_%7B%5Cmathrm%7Bth%7D%7D%3D10%5E%7B-6%7D" /></p>

因此，门限可抽象为：

<p align="center"><img alt="threshold def" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20%5CGamma_%7B%5Cmathrm%7Bth%7D%2Cm%7D%3D%5Cmin%5Cleft%5C%7BSNR%3ABER_m%28SNR%29%5Cle10%5E%7B-6%7D%5Cright%5C%7D" /></p>

需要注意：这个 `10^-6` 是速率选择时使用的目标 BER，不等于最终聚合帧的实际 PER。

---

## 3. 用 YANS 风格误码模型进行理论计算

为了获得可解析的理论模型，可以统一采用 YANS 风格误码模型。该模型的流程为：

<p align="center"><img alt="yans flow" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20SNR%5Crightarrow%20E_b%2FN_0%5Crightarrow%20P_b%5Crightarrow%20P_%7Bb%2C%5Cmathrm%7Bcoded%7D%7D%5Crightarrow%20PER%5Crightarrow%20MCS" /></p>

### 3.1 SNR 转线性值

如果 SNR 用 dB 表示，则先转为线性值：

<p align="center"><img alt="snr linear" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20%5Cgamma_j%3D10%5E%7BSNR_j%2F10%7D" /></p>

### 3.2 计算每个 MCS 的 PHY rate

对每个候选 MCS `m`，先计算它的 PHY rate：

<p align="center"><img alt="rate general" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20R_m%3D%5Cfrac%7BNSS%5Ccdot%20N_%7B%5Cmathrm%7BSD%7D%28B%29%5Ccdot%20%5Clog_2%28M_m%29%5Ccdot%20r_m%7D%7BT_%7B%5Cmathrm%7Bsym%7D%7D%28GI%29%7D" /></p>

其中：

| 符号 | 含义 |
|---|---|
| <img alt="M_m" src="https://latex.codecogs.com/svg.image?M_m" /> | 调制阶数，例如 BPSK 为 2，4096-QAM 为 4096 |
| <img alt="r_m" src="https://latex.codecogs.com/svg.image?r_m" /> | 编码率，例如 `1/2`、`3/4`、`5/6` |
| <img alt="N_SD" src="https://latex.codecogs.com/svg.image?N_%7B%5Cmathrm%7BSD%7D%28B%29" /> | 信道带宽对应的数据子载波数 |
| <img alt="T_sym" src="https://latex.codecogs.com/svg.image?T_%7B%5Cmathrm%7Bsym%7D%7D" /> | OFDM 符号时间 |

当前配置下可以近似取：

<p align="center"><img alt="rate current" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20R_m%3D%5Cfrac%7B2%5Ccdot1960%5Ccdot%5Clog_2%28M_m%29%5Ccdot r_m%7D%7B13.6%5Ctimes10%5E%7B-6%7D%7D" /></p>

其中：

```text
160 MHz 下 N_SD ≈ 1960
GI = 800 ns 时 T_sym = 12.8 μs + 0.8 μs = 13.6 μs
NSS = 2
```

### 3.3 由 SNR 计算 Eb/N0

对每个 MCS：

<p align="center"><img alt="ebn0" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20%5Cfrac%7BE_b%7D%7BN_0%7D%3D%5Cgamma_j%5Ccdot%5Cfrac%7BB%7D%7BR_m%7D" /></p>

其中：

```text
B = 160 × 10^6 Hz
R_m 使用 bps 单位
```

### 3.4 未编码 BER

对于 BPSK：

<p align="center"><img alt="ber bpsk" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20P_b%3D%5Cfrac%7B1%7D%7B2%7D%5Coperatorname%7Berfc%7D%5Cleft%28%5Csqrt%7BE_b%2FN_0%7D%5Cright%29" /></p>

对于 M-QAM，可用近似：

<p align="center"><img alt="ber qam" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20P_b%5Capprox%5Cfrac%7B1%7D%7B%5Clog_2M%7D%5Cleft%5B1-%5Cleft%281-%5Cleft%281-%5Cfrac%7B1%7D%7B%5Csqrt%7BM%7D%7D%5Cright%29%5Coperatorname%7Berfc%7D%5Cleft%28%5Csqrt%7B%5Cfrac%7B1.5%5Clog_2M%7D%7BM-1%7D%5Cfrac%7BE_b%7D%7BN_0%7D%7D%5Cright%29%5Cright%29%5E2%5Cright%5D" /></p>

### 3.5 编码后的 BER

YANS 风格模型会进一步考虑卷积编码带来的纠错增益。理论推导中可抽象写成：

<p align="center"><img alt="coded ber" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20P_%7Bb%2C%5Cmathrm%7Bcoded%7D%2Cm%7D%3DF_%7B%5Cmathrm%7BYANS%7D%7D%28P_%7Bb%2Cm%7D%2Cr_m%29" /></p>

其中：

```text
P_b,m 是调制后的未编码 BER
r_m 是当前 MCS 的编码率
F_YANS 表示 YANS 中的编码误码近似
```

---

## 4. BER 门限与 PER 门限的区别

`IdealWifiManager` 常用 BER 门限来选择 MCS：

<p align="center"><img alt="mcs ber" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20MCS_j%3D%5Cmax%5Cleft%5C%7Bm%3AP_%7Bb%2C%5Cmathrm%7Bcoded%7D%2Cm%7D%28SNR_j%29%5Cle10%5E%7B-6%7D%5Cright%5C%7D" /></p>

但如果研究 A-MPDU / A-MSDU 聚合，更应该额外关注 PER。若聚合后 PSDU 总长度为：

<p align="center"><img alt="Lpsdu" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7BPSDU%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D" /></p>

单位为 Byte，则近似 bit 数为：

<p align="center"><img alt="bits" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20N_%7B%5Cmathrm%7Bbits%7D%7D%3D8L_%7B%5Cmathrm%7BPSDU%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D" /></p>

误包率可近似为：

<p align="center"><img alt="per" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20PER_m%3D1-%5Cleft%281-P_%7Bb%2C%5Cmathrm%7Bcoded%7D%2Cm%7D%5Cright%29%5E%7B8L_%7B%5Cmathrm%7BPSDU%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%7D" /></p>

如果采用 PER 门限，则 MCS 选择可以写为：

<p align="center"><img alt="mcs per" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20MCS_j%3D%5Cmax%5Cleft%5C%7Bm%3APER_m%28SNR_j%2CL_%7B%5Cmathrm%7BPSDU%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%29%5Cle PER_%7B%5Cmathrm%7Bth%7D%7D%5Cright%5C%7D" /></p>

结论是：

```text
BER 门限：更贴近 IdealWifiManager 的速率选择口径。
PER 门限：更适合分析聚合帧长度对成功率的影响。
```

---

## 5. MCS 与 PHY rate 表

当前配置固定为：

```text
B = 160 MHz
NSS = 2
GI = 800 ns
```

因此 PHY rate 可由 MCS 唯一确定。典型速率如下：

| MCS | 调制 | 编码率 | PHY rate |
|---:|---|---:|---:|
| 0 | BPSK | `1/2` | `144.1 Mbps` |
| 1 | QPSK | `1/2` | `288.2 Mbps` |
| 2 | QPSK | `3/4` | `432.4 Mbps` |
| 3 | 16-QAM | `1/2` | `576.5 Mbps` |
| 4 | 16-QAM | `3/4` | `864.7 Mbps` |
| 5 | 64-QAM | `2/3` | `1152.9 Mbps` |
| 6 | 64-QAM | `3/4` | `1297.1 Mbps` |
| 7 | 64-QAM | `5/6` | `1441.2 Mbps` |
| 8 | 256-QAM | `3/4` | `1729.4 Mbps` |
| 9 | 256-QAM | `5/6` | `1921.6 Mbps` |
| 10 | 1024-QAM | `3/4` | `2161.8 Mbps` |
| 11 | 1024-QAM | `5/6` | `2402.0 Mbps` |
| 12 | 4096-QAM | `3/4` | `2594.1 Mbps` |
| 13 | 4096-QAM | `5/6` | `2882.4 Mbps` |

例如 MCS 13：

<p align="center"><img alt="mcs13 rate" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20R_%7B13%7D%3D%5Cfrac%7B2%5Ccdot1960%5Ccdot12%5Ccdot%285%2F6%29%7D%7B13.6%5Ctimes10%5E%7B-6%7D%7D%5Capprox2882.4%5C%20%5Cmathrm%7BMbps%7D" /></p>

---

## 6. 有效数据长度

理论中需要区分：

```text
有效数据长度：真正承载的 TCP / MSDU 负载
PSDU 总长度：空口上真正发送的聚合帧长度，包含 MAC 聚合开销
```

当前代码中应用包 / MSS 可取：

<p align="center"><img alt="L msdu" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7BMSDU%7D%7D%3D1500%5C%20%5Cmathrm%7BB%7D" /></p>

若一次 A-MPDU 中有 <img alt="N mpdu" src="https://latex.codecogs.com/svg.image?N_%7B%5Cmathrm%7BMPDU%7D%7D" /> 个 MPDU，每个 MPDU 内通过 A-MSDU 聚合 <img alt="N msdu" src="https://latex.codecogs.com/svg.image?N_%7B%5Cmathrm%7BMSDU%7D%7D" /> 个 MSDU，则有效数据长度为：

<p align="center"><img alt="payload agg" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7Bpayload%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%3DN_%7B%5Cmathrm%7BMPDU%7D%7DN_%7B%5Cmathrm%7BMSDU%7D%7DL_%7B%5Cmathrm%7BMSDU%7D%7D" /></p>

当前 A-MSDU 上限为 `11398 B`。若考虑每个 A-MSDU 子帧头约 `14 B`，并按 4 字节对齐，则一个 1500 B MSDU 对应子帧长度近似为：

<p align="center"><img alt="subframe" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7Bsub%7D%7D%3D14%2B1500%2B2%3D1516%5C%20%5Cmathrm%7BB%7D" /></p>

因此每个 A-MSDU 中最多约可放：

<p align="center"><img alt="n msdu 7" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20N_%7B%5Cmathrm%7BMSDU%7D%7D%5Capprox%5Cleft%5Clfloor%5Cfrac%7B11398%7D%7B1516%7D%5Cright%5Crfloor%3D7" /></p>

于是：

<p align="center"><img alt="payload simple" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7Bpayload%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%5Capprox10500N_%7B%5Cmathrm%7BMPDU%7D%7D%5C%20%5Cmathrm%7BB%7D" /></p>

---

## 7. 聚合后 PSDU 总长度

每个 A-MSDU 子帧长度近似为：

<p align="center"><img alt="subframe general" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7Bsub%7D%2Ci%7D%3D14%2BL_%7B%5Cmathrm%7BMSDU%7D%2Ci%7D%2Bp_i" /></p>

其中 <img alt="p_i" src="https://latex.codecogs.com/svg.image?p_i" /> 为 4 字节对齐 padding。

若每个 MPDU 中聚合 7 个 1500 B MSDU，则 A-MSDU 长度约为：

<p align="center"><img alt="amsdu length" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7BA%5Ctext%7B-%7DMSDU%7D%7D%5Capprox7%5Ctimes1516%3D10612%5C%20%5Cmathrm%7BB%7D" /></p>

每个 MPDU 还要加 MAC header 与 FCS，近似取：

<p align="center"><img alt="mac fcs" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7BMAC%2BFCS%7D%7D%5Capprox30%5C%20%5Cmathrm%7BB%7D" /></p>

则单个 MPDU 长度约为：

<p align="center"><img alt="mpdu length" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7BMPDU%7D%7D%5Capprox30%2B10612%3D10642%5C%20%5Cmathrm%7BB%7D" /></p>

A-MPDU 中每个 MPDU 前还有 delimiter，近似取：

<p align="center"><img alt="delimiter" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7Bdelimiter%7D%7D%3D4%5C%20%5Cmathrm%7BB%7D" /></p>

同时还要对齐到 4 字节边界。若：

<p align="center"><img alt="q" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20q_m%3D%5Cleft%284-%28L_%7B%5Cmathrm%7BMPDU%7D%7D%5Cbmod4%29%5Cright%29%5Cbmod4" /></p>

则单个 A-MPDU 子单元长度为：

<p align="center"><img alt="ampdu subframe" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7BA%5Ctext%7B-%7DMPDU%5C%20subframe%7D%7D%3D4%2BL_%7B%5Cmathrm%7BMPDU%7D%7D%2Bq_m" /></p>

在上面的近似中：

```text
L_MPDU = 10642 B
10642 mod 4 = 2
q_m = 2 B
```

因此：

<p align="center"><img alt="ampdu subframe numeric" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7BA%5Ctext%7B-%7DMPDU%5C%20subframe%7D%7D%5Capprox4%2B10642%2B2%3D10648%5C%20%5Cmathrm%7BB%7D" /></p>

如果一次 A-MPDU 聚合 <img alt="N mpdu 2" src="https://latex.codecogs.com/svg.image?N_%7B%5Cmathrm%7BMPDU%7D%7D" /> 个 MPDU，则：

<p align="center"><img alt="psdu final" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7BPSDU%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%5Capprox10648N_%7B%5Cmathrm%7BMPDU%7D%7D%5C%20%5Cmathrm%7BB%7D" /></p>

更一般地写：

<p align="center"><img alt="psdu general" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7BPSDU%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%3D%5Csum_%7Bm%3D1%7D%5E%7BN_%7B%5Cmathrm%7BMPDU%7D%7D%7D%5Cleft%284%2BL_%7B%5Cmathrm%7BMPDU%7D%2Cm%7D%2Bq_m%5Cright%29" /></p>

---

## 8. 聚合吞吐公式

聚合吞吐应使用：

```text
分子：有效数据长度
分母：包含聚合总长度发送时间和 MAC/PHY 开销的服务时间
```

数据部分发送时间为：

<p align="center"><img alt="data time" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20T_%7B%5Cmathrm%7Bdata%7D%2Cj%7D%3D%5Cfrac%7B8L_%7B%5Cmathrm%7BPSDU%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%7D%7BR_j%7D" /></p>

一次成功聚合传输时间为：

<p align="center"><img alt="Tsucc" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20T_%7B%5Cmathrm%7Bsucc%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%3DT_%7B%5Cmathrm%7BAIFS%7D%7D%2BE%5BT_%7B%5Cmathrm%7Bbo%7D%7D%5D%2BT_%7B%5Cmathrm%7BPHY%7D%2Cj%7D%2B%5Cfrac%7B8L_%7B%5Cmathrm%7BPSDU%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%7D%7BR_j%7D%2BT_%7B%5Cmathrm%7BSIFS%7D%7D%2BT_%7B%5Cmathrm%7BBA%7D%7D" /></p>

最终聚合吞吐为：

<p align="center"><img alt="throughput" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20S_j%5E%7B%5Cmathrm%7Bagg%7D%7D%3D%5Cfrac%7B8L_%7B%5Cmathrm%7Bpayload%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%7D%7BT_%7B%5Cmathrm%7Bsucc%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%7D" /></p>

---

## 9. 建议写法

在理论推导中，可以采用下面的统一表达：

<p align="center"><img alt="final model" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20SNR_j%5Crightarrow%20BER_m%5Crightarrow%20PER_m%5Crightarrow%20MCS_j%5Crightarrow%20R_j%5Crightarrow%20S_j%5E%7B%5Cmathrm%7Bagg%7D%7D" /></p>

其中：

<p align="center"><img alt="final mcs" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20MCS_j%3D%5Cmax%5Cleft%5C%7Bm%3APER_m%28SNR_j%2CL_%7B%5Cmathrm%7BPSDU%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%29%5Cle PER_%7B%5Cmathrm%7Bth%7D%7D%5Cright%5C%7D" /></p>

或者若要更贴近 `IdealWifiManager` 的 BER 门限口径，则写为：

<p align="center"><img alt="final mcs ber" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20MCS_j%3D%5Cmax%5Cleft%5C%7Bm%3AP_%7Bb%2C%5Cmathrm%7Bcoded%7D%2Cm%7D%28SNR_j%29%5Cle10%5E%7B-6%7D%5Cright%5C%7D" /></p>

需要强调：

```text
1. ns-3 最新版本的默认误码模型可能使用表格误码模型或高阶 MCS 回退机制；
2. 本文档中的 YANS 风格计算适合作为理论近似模型；
3. 若希望仿真与理论完全一致，应在 ns-3 代码中显式设置 ErrorRateModel，并通过 trace 输出实际 MCS、PHY rate 和 PSDU 长度。
```

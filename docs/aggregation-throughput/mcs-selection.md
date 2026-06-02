# MCS 选择与 PHY rate 计算说明

本文档说明在当前最小 Wi-Fi Mesh 场景中，已知 SNR 后如何选择 MCS，并进一步计算 PHY rate、有效数据长度和聚合后 PSDU 总长度。

为避免外部公式图片加载失败，本文档不再使用 CodeCogs 图片，公式统一使用 GitHub 原生 Markdown 数学公式。注意：GitHub 数学渲染不支持部分 LaTeX 宏，例如 `\operatorname`，因此本文档使用 `\mathrm{erfc}` 等更兼容的写法。表格中的数学符号也使用 `$...$` 写法，不再用代码块样式。

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
| Wi-Fi 标准 | 802.11be | EHT / Wi-Fi 7 |
| 速率控制 | IdealWifiManager | 根据链路 SNR 自动选择 MCS |
| 信道带宽 | $160\ \mathrm{MHz}$ | 进入 PHY rate 计算 |
| 空间流数 | $2$ | 进入 PHY rate 计算 |
| Guard Interval | $800\ \mathrm{ns}$ | 进入 PHY rate 计算 |
| 发射功率 | $20\ \mathrm{dBm}$ | 进入 SNR 计算 |
| A-MPDU | 开启 | 聚合多个 MPDU |
| A-MSDU | 开启 | 每个 MPDU 内可聚合多个 MSDU |
| 应用包 / MSS | $1500\ \mathrm{B}$ | 作为理论中的 MSDU 有效负载 |

---

## 1. 总体链条

无 OBSS 干扰时，链路速率和吞吐可以按下面链条理解：

$$
d_{\mathrm{STA}-j} \rightarrow SNR_j \rightarrow MCS_j \rightarrow R_j \rightarrow S_j^{\mathrm{agg}}
$$

其中：

$$
j \in \{\mathrm{ONT},\mathrm{AP}\}
$$

前一个文档已经给出无 OBSS 场景下的 SNR 近似表达式：

$$
SNR_j(d_{\mathrm{STA}-j}) \approx 58.28 - 30\log_{10}(d_{\mathrm{STA}-j})
$$

这里的 SNR 单位为 dB，距离单位为 m。

---

## 2. 已知 SNR 后如何选择 MCS

当前代码使用：

```cpp
wifi.SetRemoteStationManager("ns3::IdealWifiManager");
```

因此，MCS 不是人工固定的，而是由 IdealWifiManager 根据链路 SNR 自动选择。

理论上可以写成：

$$
MCS_j = \max \{m:SNR_j \ge \Gamma_{\mathrm{th},m}\}
$$

其中：

$$
\Gamma_{\mathrm{th},m}
$$

表示 MCS $m$ 对应的最小 SNR 门限，单位为 dB。

IdealWifiManager 的默认思想是：对每个候选 MCS，计算它达到目标误码率所需的 SNR 门限，然后在当前 SNR 下选择满足门限的最高速率模式。默认目标 BER 通常可写为：

$$
BER_{\mathrm{th}}=10^{-6}
$$

因此，门限可抽象为：

$$
\Gamma_{\mathrm{th},m}=\min\{SNR:BER_m(SNR)\le 10^{-6}\}
$$

需要注意：这个 $10^{-6}$ 是速率选择时使用的目标 BER，不等于最终聚合帧的实际 PER。

---

## 3. 用 YANS 风格误码模型进行理论计算

为了获得可解析的理论模型，可以统一采用 YANS 风格误码模型。该模型的流程为：

$$
SNR \rightarrow E_b/N_0 \rightarrow P_b \rightarrow P_{b,\mathrm{coded}} \rightarrow PER \rightarrow MCS
$$

### 3.1 SNR 转线性值

如果 SNR 用 dB 表示，则先转为线性值：

$$
\gamma_j = 10^{SNR_j/10}
$$

### 3.2 计算每个 MCS 的 PHY rate

对每个候选 MCS $m$，先计算它的 PHY rate：

$$
R_m = \frac{NSS \cdot N_{\mathrm{SD}}(B) \cdot \log_2(M_m) \cdot r_m}{T_{\mathrm{sym}}(GI)}
$$

其中：

| 符号 | 含义 |
|---|---|
| $M_m$ | 调制阶数，例如 BPSK 为 $2$，4096-QAM 为 $4096$ |
| $r_m$ | 编码率，例如 $1/2$、$3/4$、$5/6$ |
| $N_{\mathrm{SD}}(B)$ | 信道带宽对应的数据子载波数 |
| $T_{\mathrm{sym}}(GI)$ | OFDM 符号时间 |

当前配置下可以近似取：

$$
R_m = \frac{2 \cdot 1960 \cdot \log_2(M_m) \cdot r_m}{13.6 \times 10^{-6}}
$$

其中：

| 参数 | 取值 |
|---|---:|
| $N_{\mathrm{SD}}(160\ \mathrm{MHz})$ | $1960$ |
| $GI$ | $800\ \mathrm{ns}$ |
| $T_{\mathrm{sym}}$ | $12.8\ \mu s+0.8\ \mu s=13.6\ \mu s$ |
| $NSS$ | $2$ |

### 3.3 由 SNR 计算 $E_b/N_0$

对每个 MCS：

$$
\frac{E_b}{N_0}=\gamma_j \cdot \frac{B}{R_m}
$$

其中：

| 参数 | 取值 / 说明 |
|---|---|
| $B$ | $160\times10^6\ \mathrm{Hz}$ |
| $R_m$ | 使用 bps 单位 |

### 3.4 未编码 BER

对于 BPSK：

$$
P_b=\frac{1}{2}\mathrm{erfc}\left(\sqrt{E_b/N_0}\right)
$$

对于 M-QAM，可用近似：

$$
P_b \approx \frac{1}{\log_2 M}\left[1-\left(1-\left(1-\frac{1}{\sqrt{M}}\right)\mathrm{erfc}\left(\sqrt{\frac{1.5\log_2M}{M-1}\frac{E_b}{N_0}}\right)\right)^2\right]
$$

其中 $\mathrm{erfc}$ 是互补误差函数。这里使用 $\mathrm{erfc}$ 而不是 `\operatorname{erfc}`，是为了兼容 GitHub 的公式渲染。

### 3.5 编码后的 BER

YANS 风格模型会进一步考虑卷积编码带来的纠错增益。理论推导中可抽象写成：

$$
P_{b,\mathrm{coded},m}=F_{\mathrm{YANS}}(P_{b,m},r_m)
$$

其中：

| 符号 | 含义 |
|---|---|
| $P_{b,m}$ | 调制后的未编码 BER |
| $r_m$ | 当前 MCS 的编码率 |
| $F_{\mathrm{YANS}}$ | YANS 中的编码误码近似函数 |

---

## 4. BER 门限与 PER 门限的区别

IdealWifiManager 常用 BER 门限来选择 MCS：

$$
MCS_j=\max\{m:P_{b,\mathrm{coded},m}(SNR_j)\le 10^{-6}\}
$$

但如果研究 A-MPDU / A-MSDU 聚合，更应该额外关注 PER。若聚合后 PSDU 总长度为：

$$
L_{\mathrm{PSDU}}^{\mathrm{agg}}
$$

单位为 Byte，则近似 bit 数为：

$$
N_{\mathrm{bits}}=8L_{\mathrm{PSDU}}^{\mathrm{agg}}
$$

误包率可近似为：

$$
PER_m = 1-(1-P_{b,\mathrm{coded},m})^{8L_{\mathrm{PSDU}}^{\mathrm{agg}}}
$$

如果采用 PER 门限，则 MCS 选择可以写为：

$$
MCS_j=\max\{m:PER_m(SNR_j,L_{\mathrm{PSDU}}^{\mathrm{agg}})\le PER_{\mathrm{th}}\}
$$

结论是：

```text
BER 门限：更贴近 IdealWifiManager 的速率选择口径。
PER 门限：更适合分析聚合帧长度对成功率的影响。
```

---

## 5. MCS 与 PHY rate 表

当前配置固定为：

$$
B=160\ \mathrm{MHz},\qquad NSS=2,\qquad GI=800\ \mathrm{ns}
$$

因此 PHY rate 可由 MCS 唯一确定。典型速率如下：

| MCS | 调制 | 编码率 | PHY rate |
|---:|---|---:|---:|
| 0 | BPSK | $1/2$ | $144.1\ \mathrm{Mbps}$ |
| 1 | QPSK | $1/2$ | $288.2\ \mathrm{Mbps}$ |
| 2 | QPSK | $3/4$ | $432.4\ \mathrm{Mbps}$ |
| 3 | 16-QAM | $1/2$ | $576.5\ \mathrm{Mbps}$ |
| 4 | 16-QAM | $3/4$ | $864.7\ \mathrm{Mbps}$ |
| 5 | 64-QAM | $2/3$ | $1152.9\ \mathrm{Mbps}$ |
| 6 | 64-QAM | $3/4$ | $1297.1\ \mathrm{Mbps}$ |
| 7 | 64-QAM | $5/6$ | $1441.2\ \mathrm{Mbps}$ |
| 8 | 256-QAM | $3/4$ | $1729.4\ \mathrm{Mbps}$ |
| 9 | 256-QAM | $5/6$ | $1921.6\ \mathrm{Mbps}$ |
| 10 | 1024-QAM | $3/4$ | $2161.8\ \mathrm{Mbps}$ |
| 11 | 1024-QAM | $5/6$ | $2402.0\ \mathrm{Mbps}$ |
| 12 | 4096-QAM | $3/4$ | $2594.1\ \mathrm{Mbps}$ |
| 13 | 4096-QAM | $5/6$ | $2882.4\ \mathrm{Mbps}$ |

例如 MCS 13：

$$
R_{13}=\frac{2\cdot1960\cdot12\cdot(5/6)}{13.6\times10^{-6}}\approx2882.4\ \mathrm{Mbps}
$$

---

## 6. 有效数据长度

理论中需要区分：

```text
有效数据长度：真正承载的 TCP / MSDU 负载
PSDU 总长度：空口上真正发送的聚合帧长度，包含 MAC 聚合开销
```

当前代码中应用包 / MSS 可取：

$$
L_{\mathrm{MSDU}}=1500\ \mathrm{B}
$$

若一次 A-MPDU 中有 $N_{\mathrm{MPDU}}$ 个 MPDU，每个 MPDU 内通过 A-MSDU 聚合 $N_{\mathrm{MSDU}}$ 个 MSDU，则有效数据长度为：

$$
L_{\mathrm{payload}}^{\mathrm{agg}}=N_{\mathrm{MPDU}}N_{\mathrm{MSDU}}L_{\mathrm{MSDU}}
$$

当前 A-MSDU 上限为 $11398\ \mathrm{B}$。若考虑每个 A-MSDU 子帧头约 $14\ \mathrm{B}$，并按 4 字节对齐，则一个 $1500\ \mathrm{B}$ MSDU 对应子帧长度近似为：

$$
L_{\mathrm{sub}}=14+1500+2=1516\ \mathrm{B}
$$

因此每个 A-MSDU 中最多约可放：

$$
N_{\mathrm{MSDU}}\approx\left\lfloor\frac{11398}{1516}\right\rfloor=7
$$

于是：

$$
L_{\mathrm{payload}}^{\mathrm{agg}}\approx10500N_{\mathrm{MPDU}}\ \mathrm{B}
$$

---

## 7. 聚合后 PSDU 总长度

每个 A-MSDU 子帧长度近似为：

$$
L_{\mathrm{sub},i}=14+L_{\mathrm{MSDU},i}+p_i
$$

其中 $p_i$ 为 4 字节对齐 padding。

若每个 MPDU 中聚合 7 个 1500 B MSDU，则 A-MSDU 长度约为：

$$
L_{\mathrm{AMSDU}}\approx7\times1516=10612\ \mathrm{B}
$$

每个 MPDU 还要加 MAC header 与 FCS，近似取：

$$
L_{\mathrm{MAC+FCS}}\approx30\ \mathrm{B}
$$

则单个 MPDU 长度约为：

$$
L_{\mathrm{MPDU}}\approx30+10612=10642\ \mathrm{B}
$$

A-MPDU 中每个 MPDU 前还有 delimiter，近似取：

$$
L_{\mathrm{delimiter}}=4\ \mathrm{B}
$$

同时还要对齐到 4 字节边界。若：

$$
q_m=(4-(L_{\mathrm{MPDU}}\bmod 4))\bmod 4
$$

则单个 A-MPDU 子单元长度为：

$$
L_{\mathrm{AMPDU,sub}}=4+L_{\mathrm{MPDU}}+q_m
$$

在上面的近似中：

```text
L_MPDU = 10642 B
10642 mod 4 = 2
q_m = 2 B
```

因此：

$$
L_{\mathrm{AMPDU,sub}}\approx4+10642+2=10648\ \mathrm{B}
$$

如果一次 A-MPDU 聚合 $N_{\mathrm{MPDU}}$ 个 MPDU，则：

$$
L_{\mathrm{PSDU}}^{\mathrm{agg}}\approx10648N_{\mathrm{MPDU}}\ \mathrm{B}
$$

更一般地写：

$$
L_{\mathrm{PSDU}}^{\mathrm{agg}}=\sum_{m=1}^{N_{\mathrm{MPDU}}}(4+L_{\mathrm{MPDU},m}+q_m)
$$

---

## 8. 聚合吞吐公式

聚合吞吐应使用：

```text
分子：有效数据长度
分母：包含聚合总长度发送时间和 MAC/PHY 开销的服务时间
```

数据部分发送时间为：

$$
T_{\mathrm{data},j}=\frac{8L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_j}
$$

一次成功聚合传输时间为：

$$
T_{\mathrm{succ},j}^{\mathrm{agg}}=T_{\mathrm{AIFS}}+E[T_{\mathrm{bo}}]+T_{\mathrm{PHY},j}+\frac{8L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_j}+T_{\mathrm{SIFS}}+T_{\mathrm{BA}}
$$

最终聚合吞吐为：

$$
S_j^{\mathrm{agg}}=\frac{8L_{\mathrm{payload}}^{\mathrm{agg}}}{T_{\mathrm{succ},j}^{\mathrm{agg}}}
$$

---

## 9. 建议写法

在理论推导中，可以采用下面的统一表达：

$$
SNR_j \rightarrow BER_m \rightarrow PER_m \rightarrow MCS_j \rightarrow R_j \rightarrow S_j^{\mathrm{agg}}
$$

其中：

$$
MCS_j=\max\{m:PER_m(SNR_j,L_{\mathrm{PSDU}}^{\mathrm{agg}})\le PER_{\mathrm{th}}\}
$$

或者若要更贴近 IdealWifiManager 的 BER 门限口径，则写为：

$$
MCS_j=\max\{m:P_{b,\mathrm{coded},m}(SNR_j)\le10^{-6}\}
$$

需要强调：

```text
1. ns-3 最新版本的默认误码模型可能使用表格误码模型或高阶 MCS 回退机制；
2. 本文档中的 YANS 风格计算适合作为理论近似模型；
3. 若希望仿真与理论完全一致，应在 ns-3 代码中显式设置 ErrorRateModel，并通过 trace 输出实际 MCS、PHY rate 和 PSDU 长度。
```

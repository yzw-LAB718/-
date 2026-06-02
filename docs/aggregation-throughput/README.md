# 多帧聚合吞吐模型说明

本文档说明当前 Wi-Fi Mesh 最小场景下的多帧聚合吞吐模型。

> 显示说明：本文档使用 GitHub 原生 LaTeX 数学公式。为避免 GitHub Markdown 将公式中的单独等号行误判为标题，所有多行公式均采用 `aligned` 写法，不再让 `=` 单独占一行。

当前重点讨论的**最简单理论场景**为：

```text
1 个 ONT + 1 个 AP + 1 个 STA
AP 与 ONT 之间采用有线连接
无外部干扰，也不考虑 OBSS 干扰
无多 STA 竞争
```

当前代码的主业务方向是**下行 TCP**，即 ONT 向 STA 发送数据。因此：

```text
若 STA 接入 ONT：ONT → STA
若 STA 接入 AP：ONT → AP → STA
```

其中 AP 与 ONT 之间是有线回传，AP 与 STA、ONT 与 STA 之间是 Wi-Fi 无线链路。

---

## 1. 单帧模型与多帧聚合模型的区别

单帧模型描述的是：一次信道竞争成功后，只发送一个 DATA 帧，然后接收一个普通 ACK。

```text
AIFS / DIFS
+ 随机退避
+ PHY 前导码
+ 一个 DATA 帧
+ SIFS
+ ACK
```

多帧聚合模型描述的是：一次信道竞争成功后，发送一个 A-MPDU / A-MSDU 聚合帧，然后通过 Block ACK 统一确认。

```text
AIFS / DIFS
+ 随机退避
+ PHY 前导码
+ 聚合数据帧 A-MPDU / A-MSDU
+ SIFS
+ Block ACK
```

两者的 MAC 流程主干相同，区别在于：**单帧模型一次竞争只发送一个数据帧；聚合模型一次竞争可以发送多个数据单元，因此固定开销被更多数据分摊，吞吐通常明显更高。**

---

## 2. 多帧聚合传输时间

对候选接入点 $j\in\{\mathrm{ONT},\mathrm{AP}\}$，一次成功的聚合发送过程可写为：

```text
AIFS → Backoff → PHY Preamble → Aggregated Data → SIFS → Block ACK
```

不显式考虑传播时延时，一次成功聚合传输时间为：

$$
\begin{aligned}
T_{\mathrm{succ},j}^{\mathrm{agg}}
&=T_{\mathrm{AIFS}}+E[T_{\mathrm{bo}}]+T_{\mathrm{PHY},j}
+\frac{L_{\mathrm{PSDU},j}^{\mathrm{agg}}}{R_j(d_j)}
+T_{\mathrm{SIFS}}+T_{\mathrm{BA}}.
\end{aligned}
$$

如果后续考虑级联场景，建议使用第 8 章中的时延增强形式，把无线传播时延和有线回传时延显式加入路径时延模型。

其中：

| 符号 | 含义 |
|---|---|
| $T_{\mathrm{AIFS}}$ | 发送前的监听等待时间 |
| $E[T_{\mathrm{bo}}]$ | 平均随机退避时间 |
| $T_{\mathrm{PHY},j}$ | PHY 前导码和 PHY 头开销 |
| $L_{\mathrm{PSDU},j}^{\mathrm{agg}}$ | 聚合后在空口上传输的总 PSDU 比特数 |
| $R_j(d_j)$ | STA 与接入点 `j` 之间的 PHY rate |
| $T_{\mathrm{SIFS}}$ | 聚合数据帧与 Block ACK 之间的短帧间隔 |
| $T_{\mathrm{BA}}$ | Block ACK 发送时间 |

---

## 3. 聚合负载建模

设一次 A-MPDU 中包含 $N_{\mathrm{MPDU},j}$ 个 MPDU，每个 MPDU 内部通过 A-MSDU 聚合 $N_{\mathrm{MSDU},j}$ 个 MSDU。若每个 MSDU 的有效负载为 $L_{\mathrm{MSDU}}$，则一次聚合中真正承载的有效数据量为：

$$
\begin{aligned}
L_{\mathrm{payload},j}^{\mathrm{agg}}
&=N_{\mathrm{MPDU},j}N_{\mathrm{MSDU},j}L_{\mathrm{MSDU}}.
\end{aligned}
$$

空口上传输的 PSDU 总长度可以写成：

$$
\begin{aligned}
L_{\mathrm{PSDU},j}^{\mathrm{agg}}
&=L_{\mathrm{payload},j}^{\mathrm{agg}}+L_{\mathrm{oh},j}^{\mathrm{agg}}.
\end{aligned}
$$

其中 $L_{\mathrm{oh},j}^{\mathrm{agg}}$ 表示聚合相关开销，包括 MAC 头、FCS、A-MPDU delimiter、A-MSDU subframe header 和 padding 等。

如果只考虑 A-MPDU、不考虑 A-MSDU，可以令：

$$
N_{\mathrm{MSDU},j}=1.
$$

如果退化为单帧模型，可以令：

$$
N_{\mathrm{MPDU},j}=1,\qquad
N_{\mathrm{MSDU},j}=1,\qquad
T_{\mathrm{BA}}\rightarrow T_{\mathrm{ACK}}.
$$

---

## 4. 聚合吞吐表达式

聚合情况下，单链路吞吐定义为一次成功聚合发送中真正传输的有效数据量除以一次成功聚合发送所需时间：

$$
\begin{aligned}
S_j^{\mathrm{agg}}(d_j)
&=\frac{L_{\mathrm{payload},j}^{\mathrm{agg}}}{T_{\mathrm{succ},j}^{\mathrm{agg}}}.
\end{aligned}
$$

若 $L_{\mathrm{MSDU}}$ 以 Byte 为单位，则需要乘 8 转成 bit。于是可写成：

$$
\begin{aligned}
S_j^{\mathrm{agg}}(d_j)
&=\frac{8N_{\mathrm{MPDU},j}N_{\mathrm{MSDU},j}L_{\mathrm{MSDU}}}
{T_{\mathrm{AIFS}}+\frac{CW_{\min}}{2}\sigma+T_{\mathrm{PHY},j}
+\frac{8\left(N_{\mathrm{MPDU},j}N_{\mathrm{MSDU},j}L_{\mathrm{MSDU}}+L_{\mathrm{oh},j}^{\mathrm{agg}}\right)}{R_j(d_j)}
+T_{\mathrm{SIFS}}+T_{\mathrm{BA}}}.
\end{aligned}
$$

这个式子描述的是无线链路的聚合服务能力。实际测得的 TCP goodput 还可能受到有线回传、应用层供给、TCP ACK 反向流量、TCP 窗口和级联时延限制，因此后文会再加路径瓶颈、应用层供给和时延增强模型。

---

## 5. 多帧聚合与仿真的典型取值

### 5.1 最小理论场景建议取值

| 类别 | 参数 | 最小场景建议取值 | 说明 |
|---|---|---:|---|
| 节点 | 拓扑 | `1 ONT + 1 AP + 1 STA` | 不考虑 AP2 |
| 方向 | TCP 方向 | 下行 | ONT 向 STA 发送 |
| 接入选择 | 候选接入点 | `ONT`、`AP` | STA 二选一接入 |
| 回传 | AP-ONT | 有线 | AP 通过 ONT 连接外部网络 |
| 回传速率 | $C_{\mathrm{eth}}$ | `10 Gbps` | 通常不是瓶颈 |
| 回传时延 | $\tau_{\mathrm{eth}}$ | `500 ns` | 单跳影响很小，级联时建议保留 |
| 外部干扰 | OBSS | 关闭 | 最小理论场景 |
| 无线标准 | `WifiStandard` | `802.11be` | 代码配置 |
| 频段 | band | `5 GHz` | `BAND_5GHZ` |
| 信道号 | `ChannelNumber` | `50` | 代码配置 |
| 信道带宽 | $B$ | `160 MHz` | 进入 PHY rate |
| 空间流数 | $NSS$ | `2` | 进入 PHY rate |
| Guard Interval | $GI$ | `800 ns` | 进入 PHY rate |
| 速率控制 | `RemoteStationManager` | `IdealWifiManager` | 自动选择 MCS / PHY rate |
| 发射功率 | $P_t$ | `20 dBm` | 进入接收功率计算 |
| 路损模型 | Loss Model | `LogDistancePropagationLossModel` | 距离影响 SNR |
| 传播时延 | Delay Model | `ConstantSpeedPropagationDelayModel` | 单跳很小，级联时可累积 |
| RTS/CTS | `RtsCtsThreshold` | `999999` | 基本不触发 RTS/CTS |
| A-MPDU | `BE_MaxAmpduSize` | `15523200 B` | 聚合上限，不代表每次都发满 |
| A-MSDU | `BE_MaxAmsduSize` | `11398 B` | 聚合上限 |
| MSDU / MSS | $L_{\mathrm{MSDU}}$ | `1500 B` | 应用包 / TCP segment |
| 应用供给 | $\Lambda_{\mathrm{app}}$ | `20 Gbps` | 批量脚本实际：1 流 × 20 Gbps |
| 统计窗口 | $T_{\mathrm{test}}$ | `4 s` | 批量脚本实际值 |
| 预热时间 | $T_{\mathrm{prewarm}}$ | `1 s` | 批量脚本实际值 |
| seed | runs | `1,2,3` | 三轮取平均 |

### 5.2 C++ 默认值与批量脚本实际值的区别

需要特别区分：C++ 文件中的默认业务参数与批量脚本实际传入的参数不完全相同。

| 参数 | C++ 默认值 | 批量脚本实际值 | 说明 |
|---|---:|---:|---|
| TCP 流数 | `20` | `1` | Python 扫描脚本默认传入 `--tcp-streams 1` |
| 单流应用速率 | `1 Gbps` | `20 Gbps` | Python 扫描脚本默认传入 `--app-rate 20Gbps` |
| 总应用供给 | `20 Gbps` | `20 Gbps` | 两者总供给相同，但流数不同 |
| 包长 / MSS | `1500 B` | `1500 B` | 两者一致 |
| 预热时间 | 代码可配置 | `1 s` | 批量脚本实际值 |
| 统计窗口 | 代码可配置 | `4 s` | 批量脚本实际值 |
| seed runs | 代码可配置 | `1,2,3` | 批量脚本实际值 |

因此，如果讨论**理论最小场景**，可以把应用层看作饱和供给；如果讨论**批量脚本实际输出结果**，应按 `1` 条 TCP 流、`20 Gbps` 单流速率、`1 s` 预热和 `4 s` 统计窗口理解。

### 5.3 聚合负载的典型估算

由于应用包和 TCP segment 都为 `1500 B`，而 A-MSDU 最大尺寸约为 `11398 B`，若先忽略 A-MSDU 子帧头和 padding，则一个 A-MSDU 中最多可容纳约 7 个 1500 B MSDU：

$$
N_{\mathrm{MSDU}}\approx\left\lfloor\frac{11398}{1500}\right\rfloor=7.
$$

因此，每个 MPDU 内通过 A-MSDU 承载的有效负载可近似为：

$$
L_{\mathrm{payload,per\ MPDU}}^{\mathrm{agg}}\approx7\times1500=10500\ \mathrm{B}=84000\ \mathrm{bit}.
$$

推荐理论推导中取：

| 参数 | 推荐取值 | 说明 |
|---|---:|---|
| $L_{\mathrm{MSDU}}$ | `1500 B` | 与代码一致 |
| $N_{\mathrm{MSDU}}$ | `1 ~ 7`，常用近似 `7` | 由 A-MSDU 上限决定 |
| $N_{\mathrm{MPDU},j}$ | 保留为变量 | 由队列、MCS、PPDU 时长、Block ACK 和调度过程决定 |
| $T_{\mathrm{AIFS}}$ | `43 μs` | BE 业务常用近似 |
| $\sigma$ | `9 μs` | 5 GHz OFDM 类 Wi-Fi 常用时隙 |
| $CW_{\min}$ | `15` | BE 常用最小竞争窗口 |
| $E[T_{\mathrm{bo}}]$ | `67.5 μs` | 由 `15/2 × 9 μs` 得到 |
| $T_{\mathrm{SIFS}}$ | `16 μs` | 常用近似 |
| $T_{\mathrm{BA}}$ | `30 ~ 60 μs` | 理论近似，ns-3 中实际自动计算 |

需要注意：`15523200 B` 是 A-MPDU 聚合上限，不应直接当作每次实际发送长度。理论中更稳妥的写法是：

$$
L_{\mathrm{payload},j}^{\mathrm{agg}}\approx10500N_{\mathrm{MPDU},j}\ \mathrm{B},
$$

并保留 $N_{\mathrm{MPDU},j}$ 作为变量。

---

## 6. PHY rate 的计算

本节只考虑最简单情景：一个 ONT、一个 AP、一个 STA、无外部干扰。候选接入点只有 $j\in\{\mathrm{ONT},\mathrm{AP}\}$。

聚合吞吐公式中的 $R_j(d_j)$ 表示 STA 与候选接入点之间的 PHY rate。由于当前代码是下行 TCP，因此更直观地看，是 ONT/AP 到 STA 的无线 PHY rate；从路径损耗角度看，它仍由 STA 与 ONT/AP 的距离决定。

### 6.1 距离计算

在当前最小场景中，可以令：

| 节点 | 典型坐标 |
|---|---:|
| ONT | `(0, 0)` |
| AP | `(10, 0)` |
| STA | `(x_STA, y_STA)` |

两个候选无线链路距离分别为：

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

### 6.2 距离到 PHY rate 的链条

无外部干扰时，距离通过路径损耗、接收功率和 SNR 间接影响 PHY rate：

$$
d_{\mathrm{STA}-j}\rightarrow PL_j(d_{\mathrm{STA}-j})\rightarrow P_{r,j}(d_{\mathrm{STA}-j})\rightarrow SNR_j(d_{\mathrm{STA}-j})\rightarrow MCS_j\rightarrow R_j.
$$

路径损耗模型可写为：

$$
PL_j(d_{\mathrm{STA}-j})=PL_0+10\alpha\log_{10}\left(\frac{d_{\mathrm{STA}-j}}{d_0}\right).
$$

其中 $d_0$ 是参考距离，通常可取 `1 m`；$PL_0$ 是参考距离处的路径损耗；$\alpha$ 是路径损耗指数。

接收功率为：

$$
P_{r,j}(d_{\mathrm{STA}-j})=P_t+G_t+G_{r,j}-PL_j(d_{\mathrm{STA}-j}).
$$

无外部干扰时，信噪比为：

$$
SNR_j(d_{\mathrm{STA}-j})=P_{r,j}(d_{\mathrm{STA}-j})-N_j.
$$

噪声功率可近似写成：

$$
N_j=-174+10\log_{10}(B_j)+NF_j\qquad \mathrm{(dBm)}.
$$

其中 `-174 dBm/Hz` 是室温下的热噪声功率谱密度，$B_j$ 需要以 Hz 为单位。若 $B_j=160\ \mathrm{MHz}$ 且采用 $NF_j=7\ \mathrm{dB}$，则：

$$
N_j\approx-174+10\log_{10}(160\times10^6)+7\approx-84.96\ \mathrm{dBm}.
$$

MCS 可用 SNR 门限建模：

$$
MCS_j(d_{\mathrm{STA}-j})=\max_{m:\ SNR_j(d_{\mathrm{STA}-j})\ge\Gamma_m}m.
$$

最后得到：

$$
R_j(d_{\mathrm{STA}-j})=R_{MCS_j(d_{\mathrm{STA}-j})}(B_j,NSS_j,GI_j).
$$

在当前代码配置下，带宽、空间流和 GI 固定，因此：

$$
R_j(d_{\mathrm{STA}-j})=R_{MCS_j(SNR_j(d_{\mathrm{STA}-j}))}(160\ \mathrm{MHz},2,800\ \mathrm{ns}).
$$

### 6.3 频段为什么没有直接出现在 PHY rate 表达式中

表达式 $R_{MCS}(B,NSS,GI)$ 描述的是在给定 MCS、带宽、空间流数和保护间隔后的标称 PHY rate。2.4 GHz、5 GHz、6 GHz 的差异主要通过路径损耗进入，即通过 $PL(d,f)$ 影响接收功率、SNR 和 MCS。当前仿真固定为 5 GHz，因此模型中暂不把频率作为变量；若后续比较 2.4/5/6 GHz，需要将 $R_j(d_j)$ 扩展为 $R_j(d_j,f_j)$。

---

## 7. 路径吞吐与应用层供给限制

### 7.1 路径吞吐

若 STA 直接接入 ONT，下行路径为 ONT 到 STA：

$$
S_{\mathrm{path,ONT}}^{\mathrm{agg}}=S_{\mathrm{ONT}\to\mathrm{STA}}^{\mathrm{agg}}(d_{\mathrm{STA-ONT}}).
$$

若 STA 接入 AP，下行路径为 ONT 先经有线链路到 AP，再由 AP 通过 Wi-Fi 发给 STA：

$$
S_{\mathrm{path,AP}}^{\mathrm{agg}}=\min\left(C_{\mathrm{eth}},S_{\mathrm{AP}\to\mathrm{STA}}^{\mathrm{agg}}(d_{\mathrm{STA-AP}})\right).
$$

由于当前有线回传容量为 `10 Gbps`，通常远大于单条 Wi-Fi 链路吞吐，因此在最小场景中常可近似为：

$$
S_{\mathrm{path,AP}}^{\mathrm{agg}}\approx S_{\mathrm{AP}\to\mathrm{STA}}^{\mathrm{agg}}(d_{\mathrm{STA-AP}}).
$$

最终接入选择为：

$$
j^*=\arg\max_{j\in\{\mathrm{ONT},\mathrm{AP}\}}S_{\mathrm{path},j}^{\mathrm{agg}}.
$$

### 7.2 应用层供给限制

实际测得吞吐还受到应用层提供流量的限制，因此可以写成：

$$
S_{\mathrm{meas},j}=\min\left(S_{\mathrm{path},j},\Lambda_{\mathrm{app}}\right).
$$

批量脚本实际使用 `1` 条 TCP 流、单流应用速率 `20 Gbps`，因此：

$$
\Lambda_{\mathrm{app}}=1\times20\ \mathrm{Gbps}=20\ \mathrm{Gbps}.
$$

通常有：

$$
\Lambda_{\mathrm{app}}\gg S_{\mathrm{path},j}.
$$

因此当前最小场景可以近似看作饱和发送：

$$
S_{\mathrm{meas},j}\approx S_{\mathrm{path},j}.
$$

---

## 8. 传播时延与级联场景

### 8.1 单跳最小场景中的时延

代码中无线信道采用 `ConstantSpeedPropagationDelayModel`，有线 AP-ONT 链路设置了 `500 ns` 时延。对单跳最小场景而言，无线传播时延和有线回传时延都很小，但为了后续扩展到级联场景，建议保留时延项。

无线传播时延可写为：

$$
\tau_{\mathrm{wifi},j}=\frac{d_{\mathrm{STA}-j}}{c},
$$

其中 $c$ 为电磁波传播速度，近似取 $3\times10^8\ \mathrm{m/s}$。

严格写法可以在无线成功聚合传输时间中加入 DATA 和 Block ACK 的往返传播时延：

$$
\begin{aligned}
T_{\mathrm{succ},j}^{\mathrm{agg,delay}}
&=T_{\mathrm{AIFS}}+E[T_{\mathrm{bo}}]+T_{\mathrm{PHY},j}
+\frac{L_{\mathrm{PSDU},j}^{\mathrm{agg}}}{R_j(d_j)}
+T_{\mathrm{SIFS}}+T_{\mathrm{BA}}+2\tau_{\mathrm{wifi},j}.
\end{aligned}
$$

当前米级 Wi-Fi 场景中，例如距离为 `10 m` 时，单程无线传播时延约为 `33 ns`，往返约 `66 ns`，远小于 AIFS、随机退避、SIFS 等微秒级开销。因此在单跳吞吐主公式中可以忽略；但在级联跳数较多时，多个有线 / 无线传播时延会累积，需要在端到端时延模型中保留。

### 8.2 AP 路径中的有线时延

对于最小 AP 接入路径：

```text
ONT → AP → STA
```

除了 AP 到 STA 的无线服务时间，还存在 ONT 到 AP 的有线传输与传播时延。可以写成：

$$
D_{\mathrm{path,AP}}=\frac{L_{\mathrm{payload}}}{C_{\mathrm{eth}}}+\tau_{\mathrm{eth}}+T_{\mathrm{succ,AP}}^{\mathrm{agg,delay}}.
$$

其中当前代码典型值为：

$$
C_{\mathrm{eth}}=10\ \mathrm{Gbps},\qquad \tau_{\mathrm{eth}}=500\ \mathrm{ns}.
$$

如果采用保守的“端到端服务时间”模型，则 AP 路径的时延增强吞吐可写成：

$$
S_{\mathrm{path,AP}}^{\mathrm{delay}}=\frac{L_{\mathrm{payload}}}{D_{\mathrm{path,AP}}}.
$$

不过需要注意：对于连续 TCP 流，网络通常存在流水线传输，稳态吞吐更接近瓶颈链路容量；固定传播时延主要影响端到端 delay 和 TCP RTT，而不一定直接按上述端到端服务时间线性降低稳态吞吐。

### 8.3 级联场景的一般写法

若后续考虑多级 AP 级联，路径中可能包含多个无线跳和多个有线跳。设路径中有 $H$ 个无线跳、$K$ 个有线跳，则端到端时延可以写成：

$$
\begin{aligned}
D_{\mathrm{E2E}}
&=\sum_{h=1}^{H}T_{\mathrm{succ},h}^{\mathrm{agg,delay}}
+\sum_{k=1}^{K}\left(\frac{L_{\mathrm{payload}}}{C_{\mathrm{eth},k}}+\tau_{\mathrm{eth},k}\right).
\end{aligned}
$$

对应的保守服务时间吞吐为：

$$
\begin{aligned}
S_{\mathrm{E2E}}^{\mathrm{service}}
&=\frac{L_{\mathrm{payload}}}{D_{\mathrm{E2E}}}.
\end{aligned}
$$

如果采用稳态流水线视角，则端到端吞吐更接近路径中各跳容量的最小值：

$$
\begin{aligned}
S_{\mathrm{E2E}}^{\mathrm{bottleneck}}
&=\min\left(\min_h S_{\mathrm{wireless},h}^{\mathrm{agg}},\min_k C_{\mathrm{eth},k}\right).
\end{aligned}
$$

两种写法的区别是：

```text
保守服务时间模型：强调每个数据单元端到端经过所有跳的总耗时；
稳态瓶颈模型：强调连续流量流水线传输时由最慢链路决定吞吐。
```

在 TCP 场景中，级联时延还会增大 RTT。若 TCP 发送/接收窗口有限，吞吐还可能受到窗口限制：

$$
S_{\mathrm{TCP}}\le\frac{W_{\mathrm{TCP}}}{RTT}.
$$

因此，后续做级联分析时，建议同时保留：

```text
1. 每跳无线聚合服务时间；
2. 每段有线容量和有线时延；
3. 端到端 RTT / TCP 窗口限制；
4. 稳态瓶颈链路容量。
```

这样既能解释单跳场景下“时延影响很小”，也能扩展到级联场景下“跳数越多，端到端 delay 和 TCP RTT 越明显”的情况。

---

## 9. OBSS 与最小理论场景的区别

C++ 默认可以关闭 OBSS，但 Python 扫描脚本的默认参数会开启 OBSS1；批量脚本如果不显式关闭 OBSS，就会带一个 OBSS 干扰源。因此需要区分：

| 场景 | OBSS 设置 | 理论公式 |
|---|---:|---|
| 最小理论场景 | 关闭 | 使用 SNR |
| 批量扫描默认场景 | 可能开启 OBSS1 | 更严格应使用 SINR |

当前本文档讨论的是最小理论场景，因此无外部干扰，采用：

$$
SNR_j=P_{r,j}-N_j.
$$

如果后续考虑 OBSS 干扰，应改成：

$$
SINR_j=P_{r,j}-10\log_{10}\left(10^{N_j/10}+10^{I_j/10}\right).
$$

其中 $I_j$ 表示干扰功率。最小场景下可取 $I_j=0$。注意不能直接写成 `Pr - N - I`，因为噪声和干扰都是功率，必须先在线性域相加。

---

## 10. 单帧模型是聚合模型的特例

如果令：

$$
N_{\mathrm{MPDU},j}=1,\qquad N_{\mathrm{MSDU},j}=1,\qquad T_{\mathrm{BA}}\rightarrow T_{\mathrm{ACK}},
$$

则聚合模型退化为单帧模型。两者结构都是：

```text
吞吐 = 有效负载 / 总服务时间
```

区别在于，单帧模型中每个 1500 B 数据包都要承担一次固定开销；聚合模型中一次信道竞争可以发送多个数据单元，固定开销被聚合负载分摊，因此更适合解释 802.11be 高吞吐仿真结果。

---

## 11. 小结

当前最小场景可以按三层理解：

1. **无线聚合服务能力**：由 AIFS、退避、PHY 前导码、聚合帧发送时间、SIFS 和 Block ACK 决定；
2. **路径瓶颈**：AP 路径需要考虑 AP-ONT 有线回传容量，当前 `10 Gbps` 通常不是瓶颈；
3. **应用层供给**：批量脚本实际供给约 `20 Gbps`，通常足以把 Wi-Fi 链路打满，因此可近似看作饱和发送。

如果只分析单跳最小场景，传播时延对吞吐影响很小；如果后续分析多级 AP 级联，则应该保留无线传播时延、有线链路时延和 TCP RTT / 窗口限制，因为这些因素会随跳数增加而累积。

因此，接入选择的核心仍然是比较两条无线链路：

```text
ONT → STA 链路，由 d_STA-ONT 决定
AP  → STA 链路，由 d_STA-AP 决定
```

在无外部干扰、有线回传不构成瓶颈、应用层供给充足的条件下，STA 更应该接入能够提供更大聚合吞吐的节点。

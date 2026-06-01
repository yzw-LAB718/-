# 多帧聚合吞吐模型说明

本文档说明当前 Wi-Fi Mesh 最小场景下的多帧聚合吞吐模型。为了避免公式在 GitHub 中不渲染，关键数学表达式均使用 SVG 公式图片呈现。

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

对候选接入点 <img alt="j in ONT AP" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20j%5Cin%5C%7B%5Cmathrm%7BONT%7D%2C%5Cmathrm%7BAP%7D%5C%7D" />，一次成功的聚合发送过程可写为：

<p align="center"><img alt="aggregate process" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20%5Cmathrm%7BAIFS%7D%5Crightarrow%5Cmathrm%7BBackoff%7D%5Crightarrow%5Cmathrm%7BPHY%5C%20Preamble%7D%5Crightarrow%5Cmathrm%7BAggregated%5C%20Data%7D%5Crightarrow%5Cmathrm%7BSIFS%7D%5Crightarrow%5Cmathrm%7BBlock%5C%20ACK%7D" /></p>

一次成功聚合传输时间为：

<p align="center"><img alt="Tsucc agg" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20T_%7B%5Cmathrm%7Bsucc%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%3DT_%7B%5Cmathrm%7BAIFS%7D%7D%2BE%5BT_%7B%5Cmathrm%7Bbo%7D%7D%5D%2BT_%7B%5Cmathrm%7BPHY%7D%2Cj%7D%2B%5Cfrac%7BL_%7B%5Cmathrm%7BPSDU%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%7D%7BR_j%28d_j%29%7D%2BT_%7B%5Cmathrm%7BSIFS%7D%7D%2BT_%7B%5Cmathrm%7BBA%7D%7D" /></p>

其中：

| 符号 | 含义 |
|---|---|
| <img alt="T_AIFS" src="https://latex.codecogs.com/svg.image?T_%7B%5Cmathrm%7BAIFS%7D%7D" /> | 发送前的监听等待时间 |
| <img alt="E_T_bo" src="https://latex.codecogs.com/svg.image?E%5BT_%7B%5Cmathrm%7Bbo%7D%7D%5D" /> | 平均随机退避时间 |
| <img alt="T_PHY_j" src="https://latex.codecogs.com/svg.image?T_%7B%5Cmathrm%7BPHY%7D%2Cj%7D" /> | PHY 前导码和 PHY 头开销 |
| <img alt="L_PSDU_agg" src="https://latex.codecogs.com/svg.image?L_%7B%5Cmathrm%7BPSDU%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D" /> | 聚合后在空口上传输的总 PSDU 比特数 |
| <img alt="R_j_d_j" src="https://latex.codecogs.com/svg.image?R_j%28d_j%29" /> | STA 与接入点 `j` 之间的 PHY rate |
| <img alt="T_SIFS" src="https://latex.codecogs.com/svg.image?T_%7B%5Cmathrm%7BSIFS%7D%7D" /> | 聚合数据帧与 Block ACK 之间的短帧间隔 |
| <img alt="T_BA" src="https://latex.codecogs.com/svg.image?T_%7B%5Cmathrm%7BBA%7D%7D" /> | Block ACK 发送时间 |

---

## 3. 聚合负载建模

设一次 A-MPDU 中包含 <img alt="N_MPDU_j" src="https://latex.codecogs.com/svg.image?N_%7B%5Cmathrm%7BMPDU%7D%2Cj%7D" /> 个 MPDU，每个 MPDU 内部通过 A-MSDU 聚合 <img alt="N_MSDU_j" src="https://latex.codecogs.com/svg.image?N_%7B%5Cmathrm%7BMSDU%7D%2Cj%7D" /> 个 MSDU。若每个 MSDU 的有效负载为 <img alt="L_MSDU" src="https://latex.codecogs.com/svg.image?L_%7B%5Cmathrm%7BMSDU%7D%7D" />，则一次聚合中真正承载的有效数据量为：

<p align="center"><img alt="Lpayload agg" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7Bpayload%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%3DN_%7B%5Cmathrm%7BMPDU%7D%2Cj%7DN_%7B%5Cmathrm%7BMSDU%7D%2Cj%7DL_%7B%5Cmathrm%7BMSDU%7D%7D" /></p>

空口上传输的 PSDU 总长度可以写成：

<p align="center"><img alt="Lpsdu agg" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7BPSDU%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%3DL_%7B%5Cmathrm%7Bpayload%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%2BL_%7B%5Cmathrm%7Boh%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D" /></p>

其中 <img alt="L_oh" src="https://latex.codecogs.com/svg.image?L_%7B%5Cmathrm%7Boh%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D" /> 表示聚合相关开销，包括 MAC 头、FCS、A-MPDU delimiter、A-MSDU subframe header 和 padding 等。

如果只考虑 A-MPDU、不考虑 A-MSDU，可以令：

<p align="center"><img alt="N_MSDU_1" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20N_%7B%5Cmathrm%7BMSDU%7D%2Cj%7D%3D1" /></p>

如果退化为单帧模型，可以令：

<p align="center"><img alt="single frame limits" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20N_%7B%5Cmathrm%7BMPDU%7D%2Cj%7D%3D1%2C%5Cquad%20N_%7B%5Cmathrm%7BMSDU%7D%2Cj%7D%3D1%2C%5Cquad%20T_%7B%5Cmathrm%7BBA%7D%7D%5Crightarrow%20T_%7B%5Cmathrm%7BACK%7D%7D" /></p>

---

## 4. 聚合吞吐表达式

聚合情况下，单链路吞吐定义为一次成功聚合发送中真正传输的有效数据量除以一次成功聚合发送所需时间：

<p align="center"><img alt="S agg def" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20S_j%5E%7B%5Cmathrm%7Bagg%7D%7D%28d_j%29%3D%5Cfrac%7BL_%7B%5Cmathrm%7Bpayload%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%7D%7BT_%7B%5Cmathrm%7Bsucc%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%7D" /></p>

若 <img alt="L_MSDU" src="https://latex.codecogs.com/svg.image?L_%7B%5Cmathrm%7BMSDU%7D%7D" /> 以 Byte 为单位，则需要乘 8 转成 bit。于是可写成：

<p align="center"><img alt="S agg byte" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20S_j%5E%7B%5Cmathrm%7Bagg%7D%7D%28d_j%29%3D%5Cfrac%7B8N_%7B%5Cmathrm%7BMPDU%7D%2Cj%7DN_%7B%5Cmathrm%7BMSDU%7D%2Cj%7DL_%7B%5Cmathrm%7BMSDU%7D%7D%7D%7BT_%7B%5Cmathrm%7BAIFS%7D%7D%2B%5Cfrac%7BCW_%7B%5Cmin%7D%7D%7B2%7D%5Csigma%2BT_%7B%5Cmathrm%7BPHY%7D%2Cj%7D%2B%5Cfrac%7B8%5Cleft%28N_%7B%5Cmathrm%7BMPDU%7D%2Cj%7DN_%7B%5Cmathrm%7BMSDU%7D%2Cj%7DL_%7B%5Cmathrm%7BMSDU%7D%7D%2BL_%7B%5Cmathrm%7Boh%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%5Cright%29%7D%7BR_j%28d_j%29%7D%2BT_%7B%5Cmathrm%7BSIFS%7D%7D%2BT_%7B%5Cmathrm%7BBA%7D%7D%7D" /></p>

这个式子描述的是无线链路的聚合服务能力。实际测得吞吐还可能受到有线回传和应用层供给限制，因此后文会再加路径瓶颈和应用层供给两层。

---

## 5. 多帧聚合与仿真的典型取值

### 5.1 最小理论场景建议取值

| 类别 | 参数 | 最小场景建议取值 | 说明 |
|---|---|---:|---|
| 节点 | 拓扑 | `1 ONT + 1 AP + 1 STA` | 不考虑 AP2 |
| 方向 | TCP 方向 | 下行 | ONT 向 STA 发送 |
| 接入选择 | 候选接入点 | `ONT`、`AP` | STA 二选一接入 |
| 回传 | AP-ONT | 有线 | AP 通过 ONT 连接外部网络 |
| 回传速率 | <img alt="C_eth" src="https://latex.codecogs.com/svg.image?C_%7B%5Cmathrm%7Beth%7D" /> | `10 Gbps` | 通常不是瓶颈 |
| 回传时延 | <img alt="tau_eth" src="https://latex.codecogs.com/svg.image?%5Ctau_%7B%5Cmathrm%7Beth%7D" /> | `500 ns` | 吞吐主公式中可忽略 |
| 外部干扰 | OBSS | 关闭 | 最小理论场景 |
| 无线标准 | `WifiStandard` | `802.11be` | 代码配置 |
| 频段 | band | `5 GHz` | `BAND_5GHZ` |
| 信道号 | `ChannelNumber` | `50` | 代码配置 |
| 信道带宽 | <img alt="B" src="https://latex.codecogs.com/svg.image?B" /> | `160 MHz` | 进入 PHY rate |
| 空间流数 | <img alt="NSS" src="https://latex.codecogs.com/svg.image?NSS" /> | `2` | 进入 PHY rate |
| Guard Interval | <img alt="GI" src="https://latex.codecogs.com/svg.image?GI" /> | `800 ns` | 进入 PHY rate |
| 速率控制 | `RemoteStationManager` | `IdealWifiManager` | 自动选择 MCS / PHY rate |
| 发射功率 | <img alt="P_t" src="https://latex.codecogs.com/svg.image?P_t" /> | `20 dBm` | 进入接收功率计算 |
| 路损模型 | Loss Model | `LogDistancePropagationLossModel` | 距离影响 SNR |
| 传播时延 | Delay Model | `ConstantSpeedPropagationDelayModel` | 米级场景下传播时延很小 |
| RTS/CTS | `RtsCtsThreshold` | `999999` | 基本不触发 RTS/CTS |
| A-MPDU | `BE_MaxAmpduSize` | `15523200 B` | 聚合上限，不代表每次都发满 |
| A-MSDU | `BE_MaxAmsduSize` | `11398 B` | 聚合上限 |
| MSDU / MSS | <img alt="L_MSDU" src="https://latex.codecogs.com/svg.image?L_%7B%5Cmathrm%7BMSDU%7D%7D" /> | `1500 B` | 应用包 / TCP segment |
| 应用供给 | <img alt="Lambda_app" src="https://latex.codecogs.com/svg.image?%5CLambda_%7B%5Cmathrm%7Bapp%7D" /> | `20 Gbps` | 批量脚本实际：1 流 × 20 Gbps |
| 统计窗口 | <img alt="T_test" src="https://latex.codecogs.com/svg.image?T_%7B%5Cmathrm%7Btest%7D" /> | `4 s` | 批量脚本实际值 |
| 预热时间 | <img alt="T_prewarm" src="https://latex.codecogs.com/svg.image?T_%7B%5Cmathrm%7Bprewarm%7D" /> | `1 s` | 批量脚本实际值 |
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

<p align="center"><img alt="N_MSDU_approx_7" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20N_%7B%5Cmathrm%7BMSDU%7D%7D%5Capprox%5Cleft%5Clfloor%5Cfrac%7B11398%7D%7B1500%7D%5Cright%5Crfloor%3D7" /></p>

因此，每个 MPDU 内通过 A-MSDU 承载的有效负载可近似为：

<p align="center"><img alt="payload per mpdu" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7Bpayload%2Cper%20MPDU%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%5Capprox7%5Ctimes1500%3D10500%5C%20%5Cmathrm%7BB%7D%3D84000%5C%20%5Cmathrm%7Bbit%7D" /></p>

推荐理论推导中取：

| 参数 | 推荐取值 | 说明 |
|---|---:|---|
| <img alt="L_MSDU" src="https://latex.codecogs.com/svg.image?L_%7B%5Cmathrm%7BMSDU%7D%7D" /> | `1500 B` | 与代码一致 |
| <img alt="N_MSDU" src="https://latex.codecogs.com/svg.image?N_%7B%5Cmathrm%7BMSDU%7D%7D" /> | `1 ~ 7`，常用近似 `7` | 由 A-MSDU 上限决定 |
| <img alt="N_MPDU_j" src="https://latex.codecogs.com/svg.image?N_%7B%5Cmathrm%7BMPDU%7D%2Cj%7D" /> | 保留为变量 | 由队列、MCS、PPDU 时长、Block ACK 和调度过程决定 |
| <img alt="T_AIFS" src="https://latex.codecogs.com/svg.image?T_%7B%5Cmathrm%7BAIFS%7D%7D" /> | `43 μs` | BE 业务常用近似 |
| <img alt="sigma" src="https://latex.codecogs.com/svg.image?%5Csigma" /> | `9 μs` | 5 GHz OFDM 类 Wi-Fi 常用时隙 |
| <img alt="CWmin" src="https://latex.codecogs.com/svg.image?CW_%7B%5Cmin%7D" /> | `15` | BE 常用最小竞争窗口 |
| <img alt="Ebo" src="https://latex.codecogs.com/svg.image?E%5BT_%7B%5Cmathrm%7Bbo%7D%7D%5D" /> | `67.5 μs` | 由 `15/2 × 9 μs` 得到 |
| <img alt="T_SIFS" src="https://latex.codecogs.com/svg.image?T_%7B%5Cmathrm%7BSIFS%7D%7D" /> | `16 μs` | 常用近似 |
| <img alt="T_BA" src="https://latex.codecogs.com/svg.image?T_%7B%5Cmathrm%7BBA%7D%7D" /> | `30 ~ 60 μs` | 理论近似，ns-3 中实际自动计算 |

需要注意：`15523200 B` 是 A-MPDU 聚合上限，不应直接当作每次实际发送长度。理论中更稳妥的写法是：

<p align="center"><img alt="Lpayload typical" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7Bpayload%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%5Capprox%2010500N_%7B%5Cmathrm%7BMPDU%7D%2Cj%7D%5C%20%5Cmathrm%7BB%7D" /></p>

并保留 <img alt="N_MPDU_j" src="https://latex.codecogs.com/svg.image?N_%7B%5Cmathrm%7BMPDU%7D%2Cj%7D" /> 作为变量。

---

## 6. PHY rate 的计算

本节只考虑最简单情景：一个 ONT、一个 AP、一个 STA、无外部干扰。候选接入点只有：

<p align="center"><img alt="j in ONT AP only" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20j%5Cin%5C%7B%5Cmathrm%7BONT%7D%2C%5Cmathrm%7BAP%7D%5C%7D" /></p>

聚合吞吐公式中的 <img alt="R_j" src="https://latex.codecogs.com/svg.image?R_j%28d_j%29" /> 表示 STA 与候选接入点之间的 PHY rate。由于当前代码是下行 TCP，因此更直观地看，是 ONT/AP 到 STA 的无线 PHY rate；从路径损耗角度看，它仍由 STA 与 ONT/AP 的距离决定。

### 6.1 距离计算

在当前最小场景中，可以令：

| 节点 | 典型坐标 |
|---|---:|
| ONT | `(0, 0)` |
| AP | `(10, 0)` |
| STA | `(x_STA, y_STA)` |

两个候选无线链路距离分别为：

<p align="center"><img alt="d sta ont" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20d_%7B%5Cmathrm%7BSTA-ONT%7D%7D%3D%5Csqrt%7Bx_%7B%5Cmathrm%7BSTA%7D%7D%5E2%2By_%7B%5Cmathrm%7BSTA%7D%7D%5E2%7D" /></p>

<p align="center"><img alt="d sta ap" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20d_%7B%5Cmathrm%7BSTA-AP%7D%7D%3D%5Csqrt%7B%28x_%7B%5Cmathrm%7BSTA%7D%7D-10%29%5E2%2By_%7B%5Cmathrm%7BSTA%7D%7D%5E2%7D" /></p>

### 6.2 距离到 PHY rate 的链条

无外部干扰时，距离通过路径损耗、接收功率和 SNR 间接影响 PHY rate：

<p align="center"><img alt="rate chain" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20d_%7B%5Cmathrm%7BSTA-%7Dj%7D%5Crightarrow%20PL_j%28d_%7B%5Cmathrm%7BSTA-%7Dj%7D%29%5Crightarrow%20P_%7Br%2Cj%7D%28d_%7B%5Cmathrm%7BSTA-%7Dj%7D%29%5Crightarrow%20SNR_j%28d_%7B%5Cmathrm%7BSTA-%7Dj%7D%29%5Crightarrow%20MCS_j%5Crightarrow%20R_j" /></p>

路径损耗模型可写为：

<p align="center"><img alt="path loss" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20PL_j%28d_%7B%5Cmathrm%7BSTA-%7Dj%7D%29%3DPL_0%2B10%5Calpha%5Clog_%7B10%7D%5Cleft%28%5Cfrac%7Bd_%7B%5Cmathrm%7BSTA-%7Dj%7D%7D%7Bd_0%7D%5Cright%29" /></p>

其中 <img alt="d0" src="https://latex.codecogs.com/svg.image?d_0" /> 是参考距离，通常可取 `1 m`；<img alt="PL0" src="https://latex.codecogs.com/svg.image?PL_0" /> 是参考距离处的路径损耗；<img alt="alpha" src="https://latex.codecogs.com/svg.image?%5Calpha" /> 是路径损耗指数。当前代码没有单独手动设置这些值，而是由 ns-3 的 `LogDistancePropagationLossModel` 默认参数处理。

接收功率为：

<p align="center"><img alt="received power" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20P_%7Br%2Cj%7D%28d_%7B%5Cmathrm%7BSTA-%7Dj%7D%29%3DP_t%2BG_t%2BG_%7Br%2Cj%7D-PL_j%28d_%7B%5Cmathrm%7BSTA-%7Dj%7D%29" /></p>

无外部干扰时，信噪比为：

<p align="center"><img alt="snr" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20SNR_j%28d_%7B%5Cmathrm%7BSTA-%7Dj%7D%29%3DP_%7Br%2Cj%7D%28d_%7B%5Cmathrm%7BSTA-%7Dj%7D%29-N_j" /></p>

噪声功率可近似写成：

<p align="center"><img alt="noise" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20N_j%3D-174%2B10%5Clog_%7B10%7D%28B_j%29%2BNF_j%5Cquad%5Cmathrm%7B%28dBm%29%7D" /></p>

其中 `-174 dBm/Hz` 是室温下的热噪声功率谱密度，<img alt="B_j" src="https://latex.codecogs.com/svg.image?B_j" /> 需要以 Hz 为单位。当前 <img alt="B_j=160MHz" src="https://latex.codecogs.com/svg.image?B_j%3D160%5C%20%5Cmathrm%7BMHz%7D" /> 且暂不考虑接收机噪声系数时：

<p align="center"><img alt="noise 160" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20N_j%5Capprox-174%2B10%5Clog_%7B10%7D%28160%5Ctimes10%5E6%29%5Capprox-92%5C%20%5Cmathrm%7BdBm%7D" /></p>

MCS 可用 SNR 门限建模：

<p align="center"><img alt="mcs" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20MCS_j%28d_%7B%5Cmathrm%7BSTA-%7Dj%7D%29%3D%5Cmax%5Cleft%5C%7Bm%3ASNR_j%28d_%7B%5Cmathrm%7BSTA-%7Dj%7D%29%5Cge%5CGamma_m%5Cright%5C%7D" /></p>

最后得到：

<p align="center"><img alt="rate" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20R_j%28d_%7B%5Cmathrm%7BSTA-%7Dj%7D%29%3DR_%7BMCS_j%28d_%7B%5Cmathrm%7BSTA-%7Dj%7D%29%7D%5Cleft%28B_j%2CNSS_j%2CGI_j%5Cright%29" /></p>

在当前代码配置下，带宽、空间流和 GI 固定，因此：

<p align="center"><img alt="rate current" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20R_j%28d_%7B%5Cmathrm%7BSTA-%7Dj%7D%29%3DR_%7BMCS_j%28SNR_j%28d_%7B%5Cmathrm%7BSTA-%7Dj%7D%29%29%7D%5Cleft%28160%5C%20%5Cmathrm%7BMHz%7D%2C2%2C800%5C%20%5Cmathrm%7Bns%7D%5Cright%29" /></p>

### 6.3 频段为什么没有直接出现在 PHY rate 表达式中

表达式 <img alt="R_MCS" src="https://latex.codecogs.com/svg.image?R_%7BMCS%7D%28B%2CNSS%2CGI%29" /> 描述的是在给定 MCS、带宽、空间流数和保护间隔后的标称 PHY rate。2.4 GHz、5 GHz、6 GHz 的差异主要通过路径损耗进入，即通过 <img alt="PL_df" src="https://latex.codecogs.com/svg.image?PL%28d%2Cf%29" /> 影响接收功率、SNR 和 MCS。当前仿真固定为 5 GHz，因此模型中暂不把频率作为变量；若后续比较 2.4/5/6 GHz，需要将 <img alt="Rj" src="https://latex.codecogs.com/svg.image?R_j%28d_j%29" /> 扩展为 <img alt="Rjf" src="https://latex.codecogs.com/svg.image?R_j%28d_j%2Cf_j%29" />。

---

## 7. 路径吞吐与应用层供给限制

### 7.1 路径吞吐

若 STA 直接接入 ONT，下行路径为 ONT 到 STA：

<p align="center"><img alt="path ont" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20S_%7B%5Cmathrm%7Bpath%2CONT%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%3DS_%7B%5Cmathrm%7BONT%5Cto%20STA%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%28d_%7B%5Cmathrm%7BSTA-ONT%7D%7D%29" /></p>

若 STA 接入 AP，下行路径为 ONT 先经有线链路到 AP，再由 AP 通过 Wi-Fi 发给 STA：

<p align="center"><img alt="path ap" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20S_%7B%5Cmathrm%7Bpath%2CAP%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%3D%5Cmin%5Cleft%28C_%7B%5Cmathrm%7Beth%7D%2CS_%7B%5Cmathrm%7BAP%5Cto%20STA%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%28d_%7B%5Cmathrm%7BSTA-AP%7D%7D%29%5Cright%29" /></p>

由于当前有线回传容量为 `10 Gbps`，通常远大于单条 Wi-Fi 链路吞吐，因此在最小场景中常可近似为：

<p align="center"><img alt="path ap approx" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20S_%7B%5Cmathrm%7Bpath%2CAP%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%5Capprox%20S_%7B%5Cmathrm%7BAP%5Cto%20STA%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%28d_%7B%5Cmathrm%7BSTA-AP%7D%7D%29" /></p>

最终接入选择为：

<p align="center"><img alt="select" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20j%5E%5Cstar%3D%5Carg%5Cmax_%7Bj%5Cin%5C%7B%5Cmathrm%7BONT%7D%2C%5Cmathrm%7BAP%7D%5C%7D%7D%20S_%7B%5Cmathrm%7Bpath%2Cj%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D" /></p>

### 7.2 应用层供给限制

实际测得吞吐还受到应用层提供流量的限制，因此可以写成：

<p align="center"><img alt="measured throughput" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20S_%7B%5Cmathrm%7Bmeas%2Cj%7D%7D%3D%5Cmin%5Cleft%28S_%7B%5Cmathrm%7Bpath%2Cj%7D%7D%2C%5CLambda_%7B%5Cmathrm%7Bapp%7D%7D%5Cright%29" /></p>

批量脚本实际使用 `1` 条 TCP 流、单流应用速率 `20 Gbps`，因此：

<p align="center"><img alt="lambda app" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20%5CLambda_%7B%5Cmathrm%7Bapp%7D%7D%3D1%5Ctimes20%5C%20%5Cmathrm%7BGbps%7D%3D20%5C%20%5Cmathrm%7BGbps%7D" /></p>

通常有：

<p align="center"><img alt="lambda large" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20%5CLambda_%7B%5Cmathrm%7Bapp%7D%7D%5Cgg%20S_%7B%5Cmathrm%7Bpath%2Cj%7D%7D" /></p>

因此当前最小场景可以近似看作饱和发送：

<p align="center"><img alt="meas approx" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20S_%7B%5Cmathrm%7Bmeas%2Cj%7D%7D%5Capprox%20S_%7B%5Cmathrm%7Bpath%2Cj%7D%7D" /></p>

---

## 8. 传播时延是否需要进入吞吐公式

代码中无线信道采用 `ConstantSpeedPropagationDelayModel`，有线 AP-ONT 链路设置了 `500 ns` 时延。严格写法可以在无线成功传输时间中加入传播时延：

<p align="center"><img alt="Tsucc delay" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20T_%7B%5Cmathrm%7Bsucc%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%3DT_%7B%5Cmathrm%7BAIFS%7D%7D%2BE%5BT_%7B%5Cmathrm%7Bbo%7D%7D%5D%2BT_%7B%5Cmathrm%7BPHY%7D%2Cj%7D%2B%5Cfrac%7BL_%7B%5Cmathrm%7BPSDU%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%7D%7BR_j%28d_j%29%7D%2BT_%7B%5Cmathrm%7BSIFS%7D%7D%2BT_%7B%5Cmathrm%7BBA%7D%7D%2B2%5Ctau_%7B%5Cmathrm%7Bwifi%7D%2Cj%7D" /></p>

其中：

<p align="center"><img alt="tau wifi" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20%5Ctau_%7B%5Cmathrm%7Bwifi%7D%2Cj%7D%3D%5Cfrac%7Bd_%7B%5Cmathrm%7BSTA-%7Dj%7D%7D%7Bc%7D" /></p>

但是在室内米级 Wi-Fi 场景中，无线传播时延通常只有几十 ns；有线 `500 ns` 也只有 `0.5 μs`。它们远小于 AIFS、随机退避、SIFS 等微秒级 MAC 开销。因此，在吞吐主公式中可以忽略传播时延；如果研究端到端 delay 或 TCP RTT，再单独保留这些时延项。

---

## 9. OBSS 与最小理论场景的区别

C++ 默认可以关闭 OBSS，但 Python 扫描脚本的默认参数会开启 OBSS1；批量脚本如果不显式关闭 OBSS，就会带一个 OBSS 干扰源。因此需要区分：

| 场景 | OBSS 设置 | 理论公式 |
|---|---:|---|
| 最小理论场景 | 关闭 | 使用 SNR |
| 批量扫描默认场景 | 可能开启 OBSS1 | 更严格应使用 SINR |

当前本文档讨论的是最小理论场景，因此无外部干扰，采用：

<p align="center"><img alt="snr no interference" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20SNR_j%3DP_%7Br%2Cj%7D-N_j" /></p>

如果后续考虑 OBSS 干扰，应改成：

<p align="center"><img alt="sinr" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20SINR_j%3DP_%7Br%2Cj%7D-%2810%5Clog_%7B10%7D%2810%5E%7BN_j%2F10%7D%2B10%5E%7BI_j%2F10%7D%29%29" /></p>

其中 <img alt="I_j" src="https://latex.codecogs.com/svg.image?I_j" /> 表示干扰功率。最小场景下可取 <img alt="I zero" src="https://latex.codecogs.com/svg.image?I_j%3D0" />。

---

## 10. 单帧模型是聚合模型的特例

如果令：

<p align="center"><img alt="single special" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20N_%7B%5Cmathrm%7BMPDU%7D%2Cj%7D%3D1%2C%5Cquad%20N_%7B%5Cmathrm%7BMSDU%7D%2Cj%7D%3D1%2C%5Cquad%20T_%7B%5Cmathrm%7BBA%7D%7D%5Crightarrow%20T_%7B%5Cmathrm%7BACK%7D%7D" /></p>

则聚合模型退化为单帧模型。两者结构都是：

```text
吞吐 = 有效负载 / 总服务时间
```

区别在于，单帧模型中每个 1500 B 数据包都要承担一次固定开销；聚合模型中一次信道竞争可以发送多个数据单元，固定开销被聚合负载分摊，因此更适合解释 802.11be 高吞吐仿真结果。

---

## 11. 小结

当前最小场景可以按三层理解：

1. **无线聚合服务能力**：由 AIFS、退避、PHY 前导码、聚合帧发送时间、SIFS 和 Block ACK 决定；
2. **路径瓶颈**：AP 路径需要考虑 AP-ONT 有线回传容量，但当前 `10 Gbps` 通常不是瓶颈；
3. **应用层供给**：批量脚本实际供给约 `20 Gbps`，通常足以把 Wi-Fi 链路打满，因此可近似看作饱和发送。

因此，接入选择的核心仍然是比较两条无线链路：

```text
ONT → STA 链路，由 d_STA-ONT 决定
AP  → STA 链路，由 d_STA-AP 决定
```

在无外部干扰、有线回传不构成瓶颈、应用层供给充足的条件下，STA 更应该接入能够提供更大聚合吞吐的节点。
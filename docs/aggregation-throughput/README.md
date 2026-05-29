# 多帧聚合吞吐模型说明

本文档单独说明当前 Wi-Fi Mesh 吞吐分析中，**单帧模型**与 **多帧聚合模型**的关系，并给出更贴近当前 ns-3 仿真的 A-MPDU/A-MSDU 聚合吞吐表达式。

当前最小场景为：

```text
1 个 STA + 1 个 AP + 1 个 ONT
AP 与 ONT 之间采用有线连接
无外部干扰
无多 STA 竞争
```

STA 的目标是：在某个位置上，判断连接 AP 还是连接 ONT 能获得更大的上行吞吐。

---

## 1. 单帧模型与聚合模型的区别

前面的基础理论模型采用的是 **单帧 DATA + ACK** 结构，即一次信道竞争成功后，只发送一个 DATA 帧，然后接收一个普通 ACK。

单帧过程可以表示为：

```text
AIFS / DIFS
+ 随机退避
+ PHY 前导码
+ 一个 DATA 帧
+ SIFS
+ ACK
```

该模型适合解释 CSMA/CA 的基本时序，但它不一定能贴合当前 ns-3 的实际吞吐。原因是当前仿真使用的是 802.11be 高速 Wi-Fi 配置，并且默认允许 A-MPDU/A-MSDU 聚合。

聚合过程更接近：

```text
AIFS / DIFS
+ 随机退避
+ PHY 前导码
+ 聚合数据帧 A-MPDU / A-MSDU
+ SIFS
+ Block ACK
```

因此，单帧模型和聚合模型在结构上类似，都是“有效负载 / 总服务时间”，但二者在数值上可能差别很大。聚合模型中，一次信道竞争可以发送多个数据单元，AIFS、退避、PHY 前导码、SIFS 和 ACK 等固定开销被更多数据分摊，所以吞吐通常显著高于单帧模型。

---

## 2. 多帧聚合传输时间

对候选接入点 <img alt="j in ONT AP" src="https://latex.codecogs.com/svg.image?j%5Cin%5C%7B%5Cmathrm%7BONT%7D%2C%5Cmathrm%7BAP%7D%5C%7D" />，一次成功的聚合发送过程可写为：

<p align="center"><img alt="aggregate transmission process" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20%5Cmathrm%7BAIFS%7D%5Crightarrow%5Cmathrm%7BBackoff%7D%5Crightarrow%5Cmathrm%7BPHY%5C%20Preamble%7D%5Crightarrow%5Cmathrm%7BAggregated%5C%20Data%7D%5Crightarrow%5Cmathrm%7BSIFS%7D%5Crightarrow%5Cmathrm%7BBlock%5C%20ACK%7D" /></p>

一次成功聚合传输时间为：

<p align="center"><img alt="T_succ_agg" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20T_%7B%5Cmathrm%7Bsucc%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%3DT_%7B%5Cmathrm%7BAIFS%7D%7D%2BE%5BT_%7B%5Cmathrm%7Bbo%7D%7D%5D%2BT_%7B%5Cmathrm%7BPHY%7D%2Cj%7D%2B%5Cfrac%7BL_%7B%5Cmathrm%7BPSDU%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%7D%7BR_j%28d_j%29%7D%2BT_%7B%5Cmathrm%7BSIFS%7D%7D%2BT_%7B%5Cmathrm%7BBA%7D%7D" /></p>

其中：

| 符号 | 含义 |
|---|---|
| <img alt="T_AIFS" src="https://latex.codecogs.com/svg.image?T_%7B%5Cmathrm%7BAIFS%7D%7D" /> | 发送前的监听等待时间 |
| <img alt="E_T_bo" src="https://latex.codecogs.com/svg.image?E%5BT_%7B%5Cmathrm%7Bbo%7D%7D%5D" /> | 平均随机退避时间 |
| <img alt="T_PHY_j" src="https://latex.codecogs.com/svg.image?T_%7B%5Cmathrm%7BPHY%7D%2Cj%7D" /> | PHY 前导码和 PHY 头开销 |
| <img alt="L_PSDU_agg" src="https://latex.codecogs.com/svg.image?L_%7B%5Cmathrm%7BPSDU%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D" /> | 聚合后在空口上传输的总 PSDU 比特数 |
| <img alt="R_j_d_j" src="https://latex.codecogs.com/svg.image?R_j%28d_j%29" /> | STA 到接入点 <img alt="j" src="https://latex.codecogs.com/svg.image?j" /> 的 PHY rate，随距离变化 |
| <img alt="T_SIFS" src="https://latex.codecogs.com/svg.image?T_%7B%5Cmathrm%7BSIFS%7D%7D" /> | 聚合数据帧与 Block ACK 之间的短帧间隔 |
| <img alt="T_BA" src="https://latex.codecogs.com/svg.image?T_%7B%5Cmathrm%7BBA%7D%7D" /> | Block ACK 发送时间 |

---

## 3. 聚合负载建模

设一次 A-MPDU 中包含 <img alt="N_MPDU_j" src="https://latex.codecogs.com/svg.image?N_%7B%5Cmathrm%7BMPDU%7D%2Cj%7D" /> 个 MPDU，每个 MPDU 中进一步通过 A-MSDU 聚合 <img alt="N_MSDU_j" src="https://latex.codecogs.com/svg.image?N_%7B%5Cmathrm%7BMSDU%7D%2Cj%7D" /> 个 MSDU。若每个 MSDU 的有效负载为 <img alt="L_MSDU" src="https://latex.codecogs.com/svg.image?L_%7B%5Cmathrm%7BMSDU%7D%7D" />，则一次聚合中真正承载的有效数据量为：

<p align="center"><img alt="L_payload_agg" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7Bpayload%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%3DN_%7B%5Cmathrm%7BMPDU%7D%2Cj%7DN_%7B%5Cmathrm%7BMSDU%7D%2Cj%7DL_%7B%5Cmathrm%7BMSDU%7D%7D" /></p>

聚合帧在空口上传输的总长度不仅包括有效负载，还包括 MAC 头、FCS、A-MPDU delimiter、A-MSDU subframe header、padding 等额外开销。因此可以统一写成：

<p align="center"><img alt="L_PSDU_agg" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7BPSDU%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%3DL_%7B%5Cmathrm%7Bpayload%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%2BL_%7B%5Cmathrm%7Boh%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D" /></p>

若进一步展开聚合开销，可以写为：

<p align="center"><img alt="L_PSDU_agg_detail" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7BPSDU%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%3DN_%7B%5Cmathrm%7BMPDU%7D%2Cj%7D%5Cleft%5BL_%7B%5Cmathrm%7Bdelim%7D%7D%2BL_%7B%5Cmathrm%7BMAC%7D%7D%2BL_%7B%5Cmathrm%7BFCS%7D%7D%2BN_%7B%5Cmathrm%7BMSDU%7D%2Cj%7D%5Cleft%28L_%7B%5Cmathrm%7BMSDU%7D%7D%2BL_%7B%5Cmathrm%7Bsubhdr%7D%7D%2BL_%7B%5Cmathrm%7Bpad%7D%7D%5Cright%29%5Cright%5D" /></p>

如果只考虑 A-MPDU，不考虑 A-MSDU，可以令：

<p align="center"><img alt="N_MSDU_equals_1" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20N_%7B%5Cmathrm%7BMSDU%7D%2Cj%7D%3D1" /></p>

如果完全退化为单帧模型，可以令：

<p align="center"><img alt="single_frame_limits" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20N_%7B%5Cmathrm%7BMPDU%7D%2Cj%7D%3D1%2C%5Cquad%20N_%7B%5Cmathrm%7BMSDU%7D%2Cj%7D%3D1%2C%5Cquad%20T_%7B%5Cmathrm%7BBA%7D%7D%5Crightarrow%20T_%7B%5Cmathrm%7BACK%7D%7D" /></p>

---

## 4. 聚合吞吐表达式

聚合情况下，单链路吞吐定义为一次成功聚合发送中真正传输的有效数据量除以一次成功聚合发送所需时间：

<p align="center"><img alt="S_agg_def" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20S_j%5E%7B%5Cmathrm%7Bagg%7D%7D%28d_j%29%3D%5Cfrac%7BL_%7B%5Cmathrm%7Bpayload%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%7D%7BT_%7B%5Cmathrm%7Bsucc%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%7D" /></p>

代入成功传输时间：

<p align="center"><img alt="S_agg_full" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20S_j%5E%7B%5Cmathrm%7Bagg%7D%7D%28d_j%29%3D%5Cfrac%7BL_%7B%5Cmathrm%7Bpayload%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%7D%7BT_%7B%5Cmathrm%7BAIFS%7D%7D%2BE%5BT_%7B%5Cmathrm%7Bbo%7D%7D%5D%2BT_%7B%5Cmathrm%7BPHY%7D%2Cj%7D%2B%5Cfrac%7BL_%7B%5Cmathrm%7BPSDU%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%7D%7BR_j%28d_j%29%7D%2BT_%7B%5Cmathrm%7BSIFS%7D%7D%2BT_%7B%5Cmathrm%7BBA%7D%7D%7D" /></p>

平均退避时间可近似为：

<p align="center"><img alt="E_backoff" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20E%5BT_%7B%5Cmathrm%7Bbo%7D%7D%5D%3D%5Cfrac%7BCW_%7B%5Cmin%7D%7D%7B2%7D%5Csigma" /></p>

因此，聚合吞吐可写成：

<p align="center"><img alt="S_agg_final" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20S_j%5E%7B%5Cmathrm%7Bagg%7D%7D%28d_j%29%3D%5Cfrac%7BL_%7B%5Cmathrm%7Bpayload%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%7D%7BT_%7B%5Cmathrm%7BAIFS%7D%7D%2B%5Cfrac%7BCW_%7B%5Cmin%7D%7D%7B2%7D%5Csigma%2BT_%7B%5Cmathrm%7BPHY%7D%2Cj%7D%2B%5Cfrac%7BL_%7B%5Cmathrm%7Bpayload%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%2BL_%7B%5Cmathrm%7Boh%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%7D%7BR_j%28d_j%29%7D%2BT_%7B%5Cmathrm%7BSIFS%7D%7D%2BT_%7B%5Cmathrm%7BBA%7D%7D%7D" /></p>

这个式子是当前 ns-3 仿真场景下更合理的单链路吞吐表达式。

---

## 5. 多帧聚合典型取值

### 5.1 代码中已经明确的默认配置

当前 ns-3 代码默认开启 A-MPDU 和 A-MSDU，并给出了较大的 BE 业务聚合上限。典型配置如下：

| 参数 | 含义 | 代码默认值 |
|---|---|---:|
| `g_enableAmpdu` | 是否启用 A-MPDU | `true` |
| `g_enableAmsdu` | 是否启用 A-MSDU | `true` |
| `BE_MaxAmpduSize` | BE 业务最大 A-MPDU 聚合尺寸 | `15523200 B` |
| `BE_MaxAmsduSize` | BE 业务最大 A-MSDU 聚合尺寸 | `11398 B` |
| `g_payloadSize` | 应用层包大小 | `1500 B` |
| `TcpSocket::SegmentSize` | TCP segment 大小 | `1500 B` |
| Wi-Fi 标准 | 仿真使用的 Wi-Fi 标准 | `802.11be` |
| 速率控制 | PHY rate / MCS 选择方式 | `IdealWifiManager` |
| 信道带宽 | Wi-Fi 信道带宽 | `160 MHz` |
| 空间流数 | MIMO 空间流数 | `2` |
| GI | 保护间隔 | `800 ns` |
| 发射功率 | STA/AP 发射功率 | `20 dBm` |
| RTS/CTS threshold | RTS/CTS 触发阈值 | `999999` |
| AP-ONT 有线速率 | 有线回传容量 | `10 Gbps` |

因此，在理论建模中可以先取：

<p align="center"><img alt="L_MSDU_1500" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7BMSDU%7D%7D%3D1500%5C%20%5Cmathrm%7BB%7D" /></p>

<p align="center"><img alt="A_MSDU_max" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7BAMSDU%2Cmax%7D%7D%5Capprox11398%5C%20%5Cmathrm%7BB%7D" /></p>

<p align="center"><img alt="A_MPDU_max" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7BAMPDU%2Cmax%7D%7D%5Capprox15523200%5C%20%5Cmathrm%7BB%7D" /></p>

需要注意的是，<img alt="A_MPDU_max_inline" src="https://latex.codecogs.com/svg.image?L_%7B%5Cmathrm%7BAMPDU%2Cmax%7D%7D" /> 是代码允许的聚合上限，不代表每次空口发送都会聚合到这个大小。实际聚合长度还会受队列数据量、Block ACK、PPDU 时长、MCS、链路质量和调度过程影响。

### 5.2 由 1500 B 数据包推导的 A-MSDU 典型数量

由于应用层包和 TCP segment 都是 1500 B，而 A-MSDU 最大尺寸约为 11398 B。若先忽略 A-MSDU 子帧头和 padding，一个 A-MSDU 中最多可以容纳约 7 个 1500 B 的 MSDU：

<p align="center"><img alt="N_MSDU_approx_7" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20N_%7B%5Cmathrm%7BMSDU%7D%7D%5Capprox%5Cleft%5Clfloor%5Cfrac%7B11398%7D%7B1500%7D%5Cright%5Crfloor%3D7" /></p>

因此，每个 MPDU 中通过 A-MSDU 承载的有效负载可近似为：

<p align="center"><img alt="payload_per_mpdu" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7Bpayload%2Cper%20MPDU%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%5Capprox7%5Ctimes1500%3D10500%5C%20%5Cmathrm%7BB%7D%3D84000%5C%20%5Cmathrm%7Bbit%7D" /></p>

如果考虑 A-MSDU subframe header 和 padding，更严谨的写法是：

<p align="center"><img alt="N_MSDU_precise" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20N_%7B%5Cmathrm%7BMSDU%7D%7D%3D%5Cleft%5Clfloor%5Cfrac%7BL_%7B%5Cmathrm%7BAMSDU%2Cmax%7D%7D%7D%7BL_%7B%5Cmathrm%7BMSDU%7D%7D%2BL_%7B%5Cmathrm%7Bsubhdr%7D%7D%2BL_%7B%5Cmathrm%7Bpad%7D%7D%7D%5Cright%5Crfloor" /></p>

但在第一阶段理论推导中，取 <img alt="N_MSDU_7_inline" src="https://latex.codecogs.com/svg.image?N_%7B%5Cmathrm%7BMSDU%7D%7D%5Capprox7" /> 是一个可接受的近似。

### 5.3 推荐用于理论推导的典型组合

建议把 A-MSDU 内部聚合数量先取为典型值，把 A-MPDU 层聚合数量保留为变量：

| 参数 | 推荐取值 | 说明 |
|---|---:|---|
| <img alt="L_MSDU" src="https://latex.codecogs.com/svg.image?L_%7B%5Cmathrm%7BMSDU%7D%7D" /> | `1500 B` | 与代码中的应用包/TCP segment 一致 |
| <img alt="N_MSDU" src="https://latex.codecogs.com/svg.image?N_%7B%5Cmathrm%7BMSDU%7D%7D" /> | `1 ~ 7`，常用近似 `7` | 由 A-MSDU 最大尺寸决定 |
| <img alt="N_MPDU_j" src="https://latex.codecogs.com/svg.image?N_%7B%5Cmathrm%7BMPDU%7D%2Cj%7D" /> | 保留为变量 | 受队列、MCS、PPDU 时长、调度影响 |
| <img alt="T_BA" src="https://latex.codecogs.com/svg.image?T_%7B%5Cmathrm%7BBA%7D%7D" /> | `30 ~ 60 μs` | 理论近似；ns-3 实际自动计算 |
| <img alt="T_AIFS" src="https://latex.codecogs.com/svg.image?T_%7B%5Cmathrm%7BAIFS%7D%7D" /> | `43 μs` | 普通 BE 业务常用近似 |
| <img alt="sigma" src="https://latex.codecogs.com/svg.image?%5Csigma" /> | `9 μs` | 5 GHz OFDM 类 Wi-Fi 常用时隙 |
| <img alt="CW_min" src="https://latex.codecogs.com/svg.image?CW_%7B%5Cmin%7D" /> | `15` | BE 常用最小竞争窗口 |
| <img alt="E_T_bo" src="https://latex.codecogs.com/svg.image?E%5BT_%7B%5Cmathrm%7Bbo%7D%7D%5D" /> | `67.5 μs` | 由 `CW_min=15` 和 `σ=9 μs` 得到 |
| <img alt="T_SIFS" src="https://latex.codecogs.com/svg.image?T_%7B%5Cmathrm%7BSIFS%7D%7D" /> | `16 μs` | 常用近似 |
| <img alt="R_j_d_j" src="https://latex.codecogs.com/svg.image?R_j%28d_j%29" /> | 不写死 | 由距离、SNR、MCS 和 `IdealWifiManager` 决定 |

于是可以写成：

<p align="center"><img alt="L_payload_typical" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7Bpayload%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%5Capprox%20N_%7B%5Cmathrm%7BMPDU%7D%2Cj%7D%5Ctimes10500%5C%20%5Cmathrm%7BB%7D" /></p>

或者保留一般形式：

<p align="center"><img alt="L_payload_general" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7Bpayload%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%3DN_%7B%5Cmathrm%7BMPDU%7D%2Cj%7DN_%7B%5Cmathrm%7BMSDU%7D%2Cj%7DL_%7B%5Cmathrm%7BMSDU%7D%7D" /></p>

其中：

<p align="center"><img alt="N_MSDU_le_7" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20N_%7B%5Cmathrm%7BMSDU%7D%2Cj%7D%5Cle7" /></p>

<p align="center"><img alt="L_PSDU_limit" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20L_%7B%5Cmathrm%7BPSDU%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%5Cle15523200%5C%20%5Cmathrm%7BB%7D" /></p>

如果 <img alt="L_MSDU_inline" src="https://latex.codecogs.com/svg.image?L_%7B%5Cmathrm%7BMSDU%7D%7D" /> 用 Byte 表示，则吞吐公式中需要乘 8 转换为 bit：

<p align="center"><img alt="S_agg_with_bytes" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20S_j%5E%7B%5Cmathrm%7Bagg%7D%7D%28d_j%29%3D%5Cfrac%7B8N_%7B%5Cmathrm%7BMPDU%7D%2Cj%7DN_%7B%5Cmathrm%7BMSDU%7D%2Cj%7DL_%7B%5Cmathrm%7BMSDU%7D%7D%7D%7BT_%7B%5Cmathrm%7BAIFS%7D%7D%2B%5Cfrac%7BCW_%7B%5Cmin%7D%7D%7B2%7D%5Csigma%2BT_%7B%5Cmathrm%7BPHY%7D%2Cj%7D%2B%5Cfrac%7B8%5Cleft%28N_%7B%5Cmathrm%7BMPDU%7D%2Cj%7DN_%7B%5Cmathrm%7BMSDU%7D%2Cj%7DL_%7B%5Cmathrm%7BMSDU%7D%7D%2BL_%7B%5Cmathrm%7Boh%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%5Cright%29%7D%7BR_j%28d_j%29%7D%2BT_%7B%5Cmathrm%7BSIFS%7D%7D%2BT_%7B%5Cmathrm%7BBA%7D%7D%7D" /></p>

这个典型取值组合的核心含义是：A-MSDU 层可以先按每个 MPDU 约 7 个 1500 B 数据单元估算，而 A-MPDU 层的 MPDU 数量不建议直接取满上限，应保留为变量或通过 ns-3 trace / 仿真结果拟合得到。

---

## 6. Rj(dj) 的计算

聚合吞吐公式中的 <img alt="R_j_d_j" src="https://latex.codecogs.com/svg.image?R_j%28d_j%29" /> 表示 STA 到候选接入点 <img alt="j" src="https://latex.codecogs.com/svg.image?j" /> 的 PHY rate。它不是固定常数，而是由 STA 与候选接入点之间的距离、路径损耗、接收信号质量和速率控制共同决定。

在理论模型中，可以把计算链条写成：

<p align="center"><img alt="Rj_calculation_chain" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20d_j%5Crightarrow%20PL_j%28d_j%29%5Crightarrow%20P_%7Br%2Cj%7D%28d_j%29%5Crightarrow%20SNR_j%28d_j%29%5Crightarrow%20MCS_j%28d_j%29%5Crightarrow%20R_j%28d_j%29" /></p>

### 6.1 距离计算

对于候选接入点 <img alt="j in ONT AP1 AP2" src="https://latex.codecogs.com/svg.image?j%5Cin%5C%7B%5Cmathrm%7BONT%7D%2C%5Cmathrm%7BAP1%7D%2C%5Cmathrm%7BAP2%7D%5C%7D" />，STA 到节点 <img alt="j" src="https://latex.codecogs.com/svg.image?j" /> 的距离为：

<p align="center"><img alt="distance_general" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20d_j%3D%5Csqrt%7B%28x_%7B%5Cmathrm%7BSTA%7D%7D-x_j%29%5E2%2B%28y_%7B%5Cmathrm%7BSTA%7D%7D-y_j%29%5E2%7D" /></p>

当前代码中的典型节点位置为：

| 节点 | 典型坐标 |
|---|---:|
| ONT | `(0, 0)` |
| AP1 | `(10, 0)` |
| AP2 | `(10, 8)` |
| STA | 扫描脚本通过 `staX`、`staY` 指定 |

因此可进一步写成：

<p align="center"><img alt="d_ont" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20d_%7B%5Cmathrm%7BONT%7D%7D%3D%5Csqrt%7Bx_%7B%5Cmathrm%7BSTA%7D%7D%5E2%2By_%7B%5Cmathrm%7BSTA%7D%7D%5E2%7D" /></p>

<p align="center"><img alt="d_ap1" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20d_%7B%5Cmathrm%7BAP1%7D%7D%3D%5Csqrt%7B%28x_%7B%5Cmathrm%7BSTA%7D%7D-10%29%5E2%2By_%7B%5Cmathrm%7BSTA%7D%7D%5E2%7D" /></p>

<p align="center"><img alt="d_ap2" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20d_%7B%5Cmathrm%7BAP2%7D%7D%3D%5Csqrt%7B%28x_%7B%5Cmathrm%7BSTA%7D%7D-10%29%5E2%2B%28y_%7B%5Cmathrm%7BSTA%7D%7D-8%29%5E2%7D" /></p>

### 6.2 路径损耗和接收功率

仿真中使用 `LogDistancePropagationLossModel`，理论上可写成对数距离路径损耗模型：

<p align="center"><img alt="path_loss" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20PL_j%28d_j%29%3DPL_0%2B10%5Calpha%5Clog_%7B10%7D%5Cleft%28%5Cfrac%7Bd_j%7D%7Bd_0%7D%5Cright%29" /></p>

接收功率为：

<p align="center"><img alt="received_power" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20P_%7Br%2Cj%7D%28d_j%29%3DP_t%2BG_t%2BG_%7Br%2Cj%7D-PL_j%28d_j%29" /></p>

其中，代码中明确给出的发射功率为：

<p align="center"><img alt="Pt_20" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20P_t%3D20%5C%20%5Cmathrm%7BdBm%7D" /></p>

天线增益、参考损耗、参考距离和路径损耗指数没有在当前代码中单独手动赋值，因此理论分析中可先保留为参数，或者理解为由 ns-3 的默认信道模型内部处理。

### 6.3 SNR 计算

无外部干扰的最小场景下，信噪比可写成：

<p align="center"><img alt="snr" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20SNR_j%28d_j%29%3DP_%7Br%2Cj%7D%28d_j%29-N_j" /></p>

噪声功率可近似写为：

<p align="center"><img alt="noise_power" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20N_j%3D-174%2B10%5Clog_%7B10%7D%28B_j%29%2BNF_j%5Cquad%5Cmathrm%7B%28dBm%29%7D" /></p>

当前代码中信道带宽为：

<p align="center"><img alt="B_160" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20B_j%3D160%5C%20%5Cmathrm%7BMHz%7D" /></p>

若暂不考虑接收机噪声系数，即令 <img alt="NF_zero" src="https://latex.codecogs.com/svg.image?NF_j%3D0" />，则热噪声近似为：

<p align="center"><img alt="thermal_noise_160" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20N_j%5Capprox-174%2B10%5Clog_%7B10%7D%28160%5Ctimes10%5E6%29%5Capprox-92%5C%20%5Cmathrm%7BdBm%7D" /></p>

实际 ns-3 中噪声、误码和解码过程由 PHY 模块内部处理，代码没有把每个测试点的 <img alt="SNR" src="https://latex.codecogs.com/svg.image?SNR" /> 直接输出到 CSV。

### 6.4 MCS 与 PHY rate

理论上，MCS 可用 SNR 门限表示为：

<p align="center"><img alt="mcs_threshold" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20MCS_j%28d_j%29%3D%5Cmax%5Cleft%5C%7Bm%3ASNR_j%28d_j%29%5Cge%5CGamma_m%5Cright%5C%7D" /></p>

然后 PHY rate 为：

<p align="center"><img alt="rate_distance" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20R_j%28d_j%29%3DR_%7BMCS_j%28d_j%29%7D%5Cleft%28B_j%2CNSS_j%2CGI_j%5Cright%29" /></p>

当前代码中与速率计算直接相关的典型取值为：

| 参数 | 代码典型值 | 作用 |
|---|---:|---|
| Wi-Fi 标准 | `802.11be` | 决定可用 MCS、前导码和 PHY 结构 |
| 速率控制器 | `IdealWifiManager` | 根据链路条件自动选择 MCS / PHY rate |
| <img alt="B_j" src="https://latex.codecogs.com/svg.image?B_j" /> | `160 MHz` | 信道带宽 |
| <img alt="NSS_j" src="https://latex.codecogs.com/svg.image?NSS_j" /> | `2` | 空间流数 |
| <img alt="GI_j" src="https://latex.codecogs.com/svg.image?GI_j" /> | `800 ns` | 保护间隔 |
| <img alt="P_t" src="https://latex.codecogs.com/svg.image?P_t" /> | `20 dBm` | 发射功率 |
| 传播损耗模型 | `LogDistancePropagationLossModel` | 将距离映射为路径损耗 |

因此，面向当前仿真的 <img alt="Rj_final" src="https://latex.codecogs.com/svg.image?R_j%28d_j%29" /> 可以概括为：

<p align="center"><img alt="Rj_code_mapping" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20R_j%28d_j%29%3DR_%7BMCS_j%28SNR_j%28d_j%29%29%7D%5Cleft%28160%5C%20%5Cmathrm%7BMHz%7D%2C2%2C800%5C%20%5Cmathrm%7Bns%7D%5Cright%29" /></p>

其中 <img alt="MCS_j" src="https://latex.codecogs.com/svg.image?MCS_j" /> 不需要手动指定，而是由 `IdealWifiManager` 在仿真过程中根据链路条件自动选择。

### 6.5 需要注意的点

当前代码没有直接输出每个点的 <img alt="SNR_j" src="https://latex.codecogs.com/svg.image?SNR_j" />、<img alt="MCS_j" src="https://latex.codecogs.com/svg.image?MCS_j" /> 和 <img alt="R_j" src="https://latex.codecogs.com/svg.image?R_j" />，最终 CSV 主要输出的是端到端吞吐和各跳吞吐。因此，理论模型中的 <img alt="R_j_d_j" src="https://latex.codecogs.com/svg.image?R_j%28d_j%29" /> 更适合作为中间变量，用来解释“距离改变为什么会导致吞吐改变”。如果后续想精确拟合仿真结果，需要通过 ns-3 trace 额外记录每次发送所选 MCS、PHY rate 和接收端 SNR。

---

## 7. AP 与 ONT 的路径吞吐

若 STA 直接连接 ONT，则路径吞吐为：

<p align="center"><img alt="path_ont_agg" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20S_%7B%5Cmathrm%7Bpath%7D%2C%5Cmathrm%7BONT%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%3DS_%7B%5Cmathrm%7BSTA%7D%5Cto%5Cmathrm%7BONT%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%28d_%7B%5Cmathrm%7BONT%7D%7D%29" /></p>

若 STA 连接 AP，则路径吞吐为：

<p align="center"><img alt="path_ap_agg" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20S_%7B%5Cmathrm%7Bpath%7D%2C%5Cmathrm%7BAP%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%3D%5Cmin%5Cleft%28S_%7B%5Cmathrm%7BSTA%7D%5Cto%5Cmathrm%7BAP%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%28d_%7B%5Cmathrm%7BAP%7D%7D%29%2CC_%7B%5Cmathrm%7Beth%7D%7D%5Cright%29" /></p>

在当前最小场景中，AP 到 ONT 是有线回传。如果有线回传容量远大于 Wi-Fi 接入链路吞吐，即：

<p align="center"><img alt="backhaul_not_bottleneck" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20C_%7B%5Cmathrm%7Beth%7D%7D%5Cgg%20S_%7B%5Cmathrm%7BSTA%7D%5Cto%5Cmathrm%7BAP%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%28d_%7B%5Cmathrm%7BAP%7D%7D%29" /></p>

则：

<p align="center"><img alt="path_ap_approx" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20S_%7B%5Cmathrm%7Bpath%7D%2C%5Cmathrm%7BAP%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%5Capprox%20S_%7B%5Cmathrm%7BSTA%7D%5Cto%5Cmathrm%7BAP%7D%7D%5E%7B%5Cmathrm%7Bagg%7D%7D%28d_%7B%5Cmathrm%7BAP%7D%7D%29" /></p>

因此，最小场景下的接入点选择为：

<p align="center"><img alt="access_selection_agg" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20j%5E%5Cstar%3D%5Carg%5Cmax_%7Bj%5Cin%5C%7B%5Cmathrm%7BONT%7D%2C%5Cmathrm%7BAP%7D%5C%7D%7D%20S_%7B%5Cmathrm%7Bpath%7D%2Cj%7D%5E%7B%5Cmathrm%7Bagg%7D%7D" /></p>

---

## 8. 单帧模型是聚合模型的特例

如果令 <img alt="N_MPDU_1" src="https://latex.codecogs.com/svg.image?N_%7B%5Cmathrm%7BMPDU%7D%2Cj%7D%3D1" />、<img alt="N_MSDU_1" src="https://latex.codecogs.com/svg.image?N_%7B%5Cmathrm%7BMSDU%7D%2Cj%7D%3D1" />，并将 Block ACK 替换为普通 ACK，即：

<p align="center"><img alt="single_frame_special_case" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20N_%7B%5Cmathrm%7BMPDU%7D%2Cj%7D%3D1%2C%5Cquad%20N_%7B%5Cmathrm%7BMSDU%7D%2Cj%7D%3D1%2C%5Cquad%20T_%7B%5Cmathrm%7BBA%7D%7D%5Crightarrow%20T_%7B%5Cmathrm%7BACK%7D%7D" /></p>

则聚合模型退化为单帧模型：

<p align="center"><img alt="single_frame_model" src="https://latex.codecogs.com/svg.image?%5Cdisplaystyle%20S_j%28d_j%29%3D%5Cfrac%7BL_%7B%5Cmathrm%7Bpayload%7D%7D%7D%7BT_%7B%5Cmathrm%7BAIFS%7D%7D%2B%5Cfrac%7BCW_%7B%5Cmin%7D%7D%7B2%7D%5Csigma%2BT_%7B%5Cmathrm%7BPHY%7D%2Cj%7D%2B%5Cfrac%7BL_%7B%5Cmathrm%7BMAC%7D%7D%2BL_%7B%5Cmathrm%7Bpayload%7D%7D%7D%7BR_j%28d_j%29%7D%2BT_%7B%5Cmathrm%7BSIFS%7D%7D%2BT_%7B%5Cmathrm%7BACK%7D%7D%7D" /></p>

因此，单帧模型和聚合模型在结构上相同，都是：

```text
吞吐 = 有效负载 / 总服务时间
```

但数值上并不相同。单帧模型中，每个 1500 B 数据包都要承担一次固定开销；聚合模型中，一次信道竞争可以发送多个数据单元，固定开销被聚合负载分摊，因此更适合解释 802.11be 高吞吐仿真结果。

---

## 9. 结论

当前理论分析可以分两层使用：

1. **单帧 baseline**：用于说明 CSMA/CA 基本机制，以及距离如何通过 SNR、MCS、PHY rate 影响吞吐；
2. **多帧聚合模型**：用于贴近当前 ns-3 中 802.11be 的实际仿真结果。

如果只是做机制推导，可以先使用单帧模型；如果要和当前仿真结果对齐，应优先采用多帧聚合模型。

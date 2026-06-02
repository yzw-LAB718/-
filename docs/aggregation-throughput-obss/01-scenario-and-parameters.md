# 01. OBSS 干扰场景与参数

本文档对应当前带 OBSS 干扰的 Wi-Fi Mesh 仿真场景。相较于无 OBSS 场景，主要变化是：主链路接收端除了受到热噪声影响，还会受到同频 OBSS AP 的干扰，同时 OBSS 业务会占用一部分信道时间。

---

## 1. 节点与坐标

当前拓扑中的固定节点为：

| 节点 | 坐标 | 说明 |
|---|---:|---|
| ONT | $(0,0,0)$ | 固定节点 |
| AP1 | $(10,0,0)$ | 固定节点 |
| AP2 | $(10,8,0)$ | 固定节点 |
| STA | $(x_{\mathrm{STA}},y_{\mathrm{STA}},0)$ | 扫描网格中心 |
| OBSS AP | $(4,4,0)$ | 干扰源 AP |
| OBSS STA | $(5,4,0)$ | 干扰源 STA |

候选接入点可写为：

$$
j\in\{\mathrm{ONT},\mathrm{AP1},\mathrm{AP2}\}.
$$

如果只讨论最小场景，可以退化为：

$$
j\in\{\mathrm{ONT},\mathrm{AP1}\}.
$$

---

## 2. 主链路距离

STA 到各候选接入点的距离为：

$$
\begin{aligned}
d_{\mathrm{STA-ONT}}&=\sqrt{x_{\mathrm{STA}}^2+y_{\mathrm{STA}}^2},\\
d_{\mathrm{STA-AP1}}&=\sqrt{(x_{\mathrm{STA}}-10)^2+y_{\mathrm{STA}}^2},\\
d_{\mathrm{STA-AP2}}&=\sqrt{(x_{\mathrm{STA}}-10)^2+(y_{\mathrm{STA}}-8)^2}.
\end{aligned}
$$

为避免 $d=0$ 或近场距离导致对数路损异常，实际代入路径损耗时使用：

$$
d_{\mathrm{eff}}=\max(d,d_0),\qquad d_0=1\ \mathrm{m}.
$$

---

## 3. 干扰链路距离

OBSS AP 到目标 STA 的距离为：

$$
\begin{aligned}
d_{\mathrm{OBSSAP-STA}}
&=\sqrt{(x_{\mathrm{STA}}-4)^2+(y_{\mathrm{STA}}-4)^2}.
\end{aligned}
$$

如果后续考虑 OBSS STA 的 ACK 或上行反馈造成的干扰，可补充：

$$
\begin{aligned}
d_{\mathrm{OBSSSTA-STA}}
&=\sqrt{(x_{\mathrm{STA}}-5)^2+(y_{\mathrm{STA}}-4)^2}.
\end{aligned}
$$

当前简化模型主要考虑 OBSS AP 下行 UDP 数据帧对目标 STA 的干扰，因此默认使用 $d_{\mathrm{OBSSAP-STA}}$。

---

## 4. 无线与业务参数

当前仿真中的主要无线配置为：

| 参数 | 当前值 | 说明 |
|---|---:|---|
| Wi-Fi 标准 | 802.11be | EHT / Wi-Fi 7 |
| 速率控制 | IdealWifiManager | 根据链路质量选择 MCS |
| 信道带宽 | $160\ \mathrm{MHz}$ | 主链路 PHY rate 计算使用 |
| 空间流数 | $2$ | 主链路 PHY rate 计算使用 |
| Guard Interval | $800\ \mathrm{ns}$ | 主链路 PHY rate 计算使用 |
| 信道号 | $50$ | 同频 OBSS 干扰 |
| 发射功率 | $20\ \mathrm{dBm}$ | 主链路与干扰链路近似同功率 |
| 路损模型 | LogDistancePropagationLossModel | 对数距离损耗 |
| 传播时延模型 | ConstantSpeedPropagationDelayModel | 固定传播速度 |
| RTS/CTS 阈值 | $999999$ | 基本不触发 RTS/CTS |
| A-MPDU 上限 | $15523200\ \mathrm{B}$ | 开启 A-MPDU 聚合 |
| A-MSDU 上限 | $11398\ \mathrm{B}$ | 开启 A-MSDU 聚合 |

主业务配置为：

| 参数 | 当前值 | 说明 |
|---|---:|---|
| 业务方向 | 下行 TCP | ONT 向 STA 发送 |
| TCP 流数 | $1$ | 单流 TCP |
| 单流速率 | $20\ \mathrm{Gbps}$ | 应用层供给 |
| 包长 / MSS | $1500\ \mathrm{B}$ | TCP 负载大小 |
| RcvBufSize | $2\ \mathrm{MB}$ | TCP 接收缓冲区 |
| SndBufSize | $2\ \mathrm{MB}$ | TCP 发送缓冲区 |
| TCP Timestamp | false | 关闭 TCP 时间戳 |

有线回程配置为：

| 参数 | 当前值 | 说明 |
|---|---:|---|
| 链路模型 | CsmaHelper | 有线回程使用 CSMA |
| 数据率 | $20\ \mathrm{Gbps}$ | 有线链路最大传输速率 |
| 时延 | $500\ \mathrm{ns}$ | 固定链路时延 |

---

## 5. OBSS 干扰业务参数

OBSS 干扰业务为：

| 参数 | 当前值 | 说明 |
|---|---:|---|
| 频段 | 同频 | 与主链路同信道竞争 |
| 业务方向 | 下行 UDP | OBSS AP 向 OBSS STA 发送 |
| 流数 | $1$ | 单个干扰源 OBSS1 |
| OnTime | 常开 | Constant = $10^9$ |
| OffTime | $0$ | 连续发送 |
| OBSS 应用速率 | $150\ \mathrm{Mbps}$ | 显式配置的 OBSS 聚合发送速率 |
| UDP 负载 | $1472\ \mathrm{B}$ | 干扰包长 |
| 理想占空比 | $10\%-20\%$ | 实际约 $10\%-15\%$ |

理论中用 OBSS 占空比表示信道占用：

$$
\rho_{\mathrm{OBSS}}\approx0.10\sim0.15.
$$

---

## 6. 相比无 OBSS 模型需要修改的变量

无 OBSS 场景中，链路质量用 $SNR_j$ 表示；带 OBSS 后需要改为 $SINR_j$：

$$
SNR_j\quad\Longrightarrow\quad SINR_j.
$$

因此以下变量都需要相应修改：

| 无 OBSS 写法 | OBSS 写法 | 说明 |
|---|---|---|
| $SNR_j$ | $SINR_j$ | 加入干扰功率 |
| $MCS_j(SNR_j)$ | $MCS_j(SINR_j)$ | MCS 由 SINR 决定 |
| $R_j(SNR_j)$ | $R_j(SINR_j)$ | PHY rate 由 SINR 间接决定 |
| $S_j^{\mathrm{agg}}$ | $S_j^{\mathrm{agg,OBSS}}$ | 吞吐加入干扰占空修正 |
| $j^*=\arg\max S_j^{\mathrm{agg}}$ | $j^*=\arg\max S_j^{\mathrm{agg,OBSS}}$ | 按干扰修正后吞吐选接入点 |

---

## 7. 需要重新计算的量

加入 OBSS AP / STA 后，需要重新计算：

$$
\begin{aligned}
d_{\mathrm{OBSSAP-STA}},\qquad
I_{\mathrm{OBSS}},\qquad
SINR_j,\qquad
MCS_j,\qquad
R_j(SINR_j),\qquad
S_j^{\mathrm{agg,OBSS}}.
\end{aligned}
$$

其中 $j$ 为候选接入点。
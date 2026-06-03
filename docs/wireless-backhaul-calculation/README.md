# 无线回传场景下的吞吐计算要点

本文档专门总结 **ONT–AP 为无线回传** 时，当前 Wi-Fi Mesh 理论吞吐模型的计算流程、关键假设和推荐公式。

> 公式渲染说明：本文档所有多行公式均使用 `aligned` 写法，避免 GitHub Markdown 把单独的 `=` 或 `\approx` 误判为标题分隔线，导致 LaTeX 渲染失败。

---

## 1. 场景假设

最小理论场景为：

```text
1 个 ONT + 1 个 AP + 1 个 STA
ONT 与 AP 之间采用无线连接
STA 可以选择接入 ONT 或 AP
业务方向为下行 TCP
无 OBSS 外部干扰
无多 STA 竞争
```

因此，下行路径有两种：

```text
STA 接入 ONT：ONT → STA
STA 接入 AP ：ONT → AP → STA
```

其中：

```text
ONT → STA：无线接入链路
ONT → AP ：无线回传链路
AP  → STA：无线接入链路
```

在该场景中，**STA 接入 ONT** 是单跳无线；**STA 接入 AP** 是双跳无线，即一跳无线回传加一跳无线接入。

---

## 2. 无线回传场景的核心变化

有线回传时，STA 接入 AP 的路径吞吐通常可近似为 AP 到 STA 的一跳无线吞吐：

$$
\begin{aligned}
S_{\mathrm{path,AP,wired}}^{\mathrm{agg}}
&\approx S_{\mathrm{AP}\to\mathrm{STA}}^{\mathrm{agg}}.
\end{aligned}
$$

但当 ONT–AP 变成无线回传后，STA 接入 AP 的路径为：

```text
ONT → AP → STA
```

同一个数据单元需要占用两次无线空口：

```text
第一次：ONT → AP
第二次：AP  → STA
```

因此 AP 路径不能再只计算 AP–STA 一跳，而应计算两段无线服务时间之和：

$$
\begin{aligned}
T_{\mathrm{path,AP,wireless}}^{\mathrm{agg}}
&\approx T_{\mathrm{ONT}\to\mathrm{AP}}^{\mathrm{agg}}
+T_{\mathrm{AP}\to\mathrm{STA}}^{\mathrm{agg}}.
\end{aligned}
$$

这就是无线回传和有线回传的核心区别。

---

## 3. 单跳无线聚合服务时间

对任意无线链路 $u\to v$，一次成功聚合传输时间写为：

$$
\begin{aligned}
T_{u\to v}^{\mathrm{agg}}
&=T_{\mathrm{AIFS}}+E[T_{\mathrm{bo}}]+T_{\mathrm{PHY}}
+\frac{L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_{u\to v}}
+T_{\mathrm{SIFS}}+T_{\mathrm{BA}}.
\end{aligned}
$$

这里默认 $L_{\mathrm{PSDU}}^{\mathrm{agg}}$ 使用 bit 表示。若 $L_{\mathrm{PSDU}}^{\mathrm{agg}}$ 使用 Byte 表示，则应写成：

$$
\begin{aligned}
T_{u\to v}^{\mathrm{agg}}
&=T_{\mathrm{AIFS}}+E[T_{\mathrm{bo}}]+T_{\mathrm{PHY}}
+\frac{8L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_{u\to v}}
+T_{\mathrm{SIFS}}+T_{\mathrm{BA}}.
\end{aligned}
$$

当前理论近似中取：

$$
\begin{aligned}
T_{\mathrm{PHY}}&\approx52\ \mu s,\\
T_{\mathrm{AIFS}}&=43\ \mu s,\\
E[T_{\mathrm{bo}}]&=\frac{CW_{\min}}{2}\sigma=\frac{15}{2}\times9\ \mu s=67.5\ \mu s,\\
T_{\mathrm{SIFS}}&=16\ \mu s,\\
T_{\mathrm{BA}}&=45\ \mu s.
\end{aligned}
$$

定义单跳固定开销：

$$
\begin{aligned}
T_{\mathrm{fixed}}
&=T_{\mathrm{AIFS}}+E[T_{\mathrm{bo}}]+T_{\mathrm{PHY}}+T_{\mathrm{SIFS}}+T_{\mathrm{BA}}.
\end{aligned}
$$

代入上述取值：

$$
\begin{aligned}
T_{\mathrm{fixed}}
&=43+67.5+52+16+45\\
&=223.5\ \mu s.
\end{aligned}
$$

---

## 4. 无线链路速率计算

无线回传场景下，需要分别计算不同无线链路的 PHY rate。

若 STA 接入 ONT，只需要计算 $R_{\mathrm{ONT}\to\mathrm{STA}}$。若 STA 接入 AP，则需要同时计算 $R_{\mathrm{ONT}\to\mathrm{AP}}$ 和 $R_{\mathrm{AP}\to\mathrm{STA}}$。

每条无线链路都遵循：

$$
\begin{aligned}
d_{u\to v}
&\rightarrow PL_{u\to v}(d_{u\to v})
\rightarrow P_{r,u\to v}(d_{u\to v})
\rightarrow SNR_{u\to v}
\rightarrow MCS_{u\to v}
\rightarrow R_{u\to v}.
\end{aligned}
$$

无 OBSS 干扰时：

$$
\begin{aligned}
SNR_{u\to v}&=P_{r,u\to v}-N.
\end{aligned}
$$

MCS 可写为：

$$
\begin{aligned}
MCS_{u\to v}
&=\max_{m:\ SNR_{u\to v}\ge\Gamma_m}m.
\end{aligned}
$$

当前配置下：

$$
\begin{aligned}
R_{u\to v}&=R_{MCS_{u\to v}}(160\ \mathrm{MHz},2,800\ \mathrm{ns}).
\end{aligned}
$$

---

## 5. 距离计算

在最小场景中，可令：

| 节点 | 坐标 |
|---|---:|
| ONT | `(0, 0)` |
| AP | `(10, 0)` |
| STA | `(x_STA, y_STA)` |

则：

$$
\begin{aligned}
d_{\mathrm{ONT}\to\mathrm{STA}}
&=\sqrt{x_{\mathrm{STA}}^2+y_{\mathrm{STA}}^2},\\
d_{\mathrm{AP}\to\mathrm{STA}}
&=\sqrt{(x_{\mathrm{STA}}-10)^2+y_{\mathrm{STA}}^2},\\
d_{\mathrm{ONT}\to\mathrm{AP}}
&=10\ \mathrm{m}.
\end{aligned}
$$

实际代入路径损耗时，建议使用有效距离：

$$
\begin{aligned}
d_{\mathrm{eff}}&=\max(d,d_0),\qquad d_0=1\ \mathrm{m}.
\end{aligned}
$$

---

## 6. 聚合负载取值

当前简化模型中可取：

$$
\begin{aligned}
N_{\mathrm{MSDU}}&=7,\\
N_{\mathrm{MPDU}}&=64,\\
L_{\mathrm{MSDU}}&=1500\ \mathrm{B}.
\end{aligned}
$$

因此一次聚合中的有效负载为：

$$
\begin{aligned}
L_{\mathrm{payload}}^{\mathrm{agg}}
&=N_{\mathrm{MPDU}}N_{\mathrm{MSDU}}L_{\mathrm{MSDU}}\\
&=64\times7\times1500\\
&=672000\ \mathrm{B}\\
&=5376000\ \mathrm{bit}.
\end{aligned}
$$

若单个 A-MPDU 子单元长度近似取：

$$
\begin{aligned}
L_{\mathrm{AMPDU,sub}}&=10648\ \mathrm{B},
\end{aligned}
$$

则聚合后 PSDU 总长度为：

$$
\begin{aligned}
L_{\mathrm{PSDU}}^{\mathrm{agg}}
&=N_{\mathrm{MPDU}}L_{\mathrm{AMPDU,sub}}\\
&=64\times10648\\
&=681472\ \mathrm{B}\\
&=5451776\ \mathrm{bit}.
\end{aligned}
$$

理论推导时也可以保留 $N_{\mathrm{MPDU}}$ 为变量：

$$
\begin{aligned}
L_{\mathrm{payload}}^{\mathrm{agg}}&\approx10500N_{\mathrm{MPDU}}\ \mathrm{B},\\
L_{\mathrm{PSDU}}^{\mathrm{agg}}&\approx10648N_{\mathrm{MPDU}}\ \mathrm{B}.
\end{aligned}
$$

---

## 7. STA 接入 ONT：单跳无线路径

STA 接入 ONT 时，路径为：

```text
ONT → STA
```

因此路径吞吐为：

$$
\begin{aligned}
S_{\mathrm{path,ONT}}^{\mathrm{agg}}
&=S_{\mathrm{ONT}\to\mathrm{STA}}^{\mathrm{agg}}.
\end{aligned}
$$

展开为：

$$
\begin{aligned}
S_{\mathrm{path,ONT}}^{\mathrm{agg}}
&\approx
\frac{L_{\mathrm{payload}}^{\mathrm{agg}}}
{T_{\mathrm{fixed}}+\dfrac{L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_{\mathrm{ONT}\to\mathrm{STA}}}}.
\end{aligned}
$$

其中 $L_{\mathrm{payload}}^{\mathrm{agg}}$ 和 $L_{\mathrm{PSDU}}^{\mathrm{agg}}$ 均使用 bit 表示。

---

## 8. STA 接入 AP：双跳无线路径

STA 接入 AP 时，路径为：

```text
ONT → AP → STA
```

第一跳无线回传时间为：

$$
\begin{aligned}
T_{\mathrm{ONT}\to\mathrm{AP}}^{\mathrm{agg}}
&=T_{\mathrm{fixed}}+
\frac{L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_{\mathrm{ONT}\to\mathrm{AP}}}.
\end{aligned}
$$

第二跳无线接入时间为：

$$
\begin{aligned}
T_{\mathrm{AP}\to\mathrm{STA}}^{\mathrm{agg}}
&=T_{\mathrm{fixed}}+
\frac{L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_{\mathrm{AP}\to\mathrm{STA}}}.
\end{aligned}
$$

若回传和接入使用同一个无线信道，且按半双工 / NSTR 近似建模，则两段空口时间需要相加：

$$
\begin{aligned}
T_{\mathrm{path,AP}}^{\mathrm{agg}}
&\approx T_{\mathrm{ONT}\to\mathrm{AP}}^{\mathrm{agg}}
+T_{\mathrm{AP}\to\mathrm{STA}}^{\mathrm{agg}}.
\end{aligned}
$$

因此 AP 路径吞吐为：

$$
\begin{aligned}
S_{\mathrm{path,AP,wireless}}^{\mathrm{agg}}
&\approx
\frac{L_{\mathrm{payload}}^{\mathrm{agg}}}
{2T_{\mathrm{fixed}}
+\dfrac{L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_{\mathrm{ONT}\to\mathrm{AP}}}
+\dfrac{L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_{\mathrm{AP}\to\mathrm{STA}}}}.
\end{aligned}
$$

如果长度使用 Byte 表示，应把两个数据发送时间项写成：

$$
\begin{aligned}
\frac{8L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_{\mathrm{ONT}\to\mathrm{AP}}},
\qquad
\frac{8L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_{\mathrm{AP}\to\mathrm{STA}}}.
\end{aligned}
$$

---

## 9. 等效容量写法

也可以先定义两个单跳聚合吞吐：

$$
\begin{aligned}
C_1&=S_{\mathrm{ONT}\to\mathrm{AP}}^{\mathrm{agg}},\\
C_2&=S_{\mathrm{AP}\to\mathrm{STA}}^{\mathrm{agg}}.
\end{aligned}
$$

如果两跳使用同一信道、单 radio、半双工近似，则端到端吞吐更接近串行服务模型：

$$
\begin{aligned}
S_{\mathrm{path,AP,wireless}}^{\mathrm{agg}}
&\approx
\frac{1}{\dfrac{1}{C_1}+\dfrac{1}{C_2}}.
\end{aligned}
$$

如果两跳速率相同，即 $C_1=C_2=C$，则：

$$
\begin{aligned}
S_{\mathrm{path,AP,wireless}}^{\mathrm{agg}}
&\approx\frac{C}{2}.
\end{aligned}
$$

这说明同信道双跳无线回传即使每一跳链路都很好，端到端吞吐也常常会接近单跳的一半。

---

## 10. 与双信道 / 双 radio 情况的区别

如果 ONT–AP 和 AP–STA 使用不同信道，或者 AP 具有两个独立 radio，可以一边接收回传、一边发送接入流量，则稳态流水线吞吐可以近似写成：

$$
\begin{aligned}
S_{\mathrm{path,AP,dual}}^{\mathrm{agg}}
&\approx\min(C_1,C_2).
\end{aligned}
$$

但当前理论模型若对应同信道单 radio / 半双工场景，应优先使用串行服务时间相加模型：

$$
\begin{aligned}
S_{\mathrm{path,AP,wireless}}^{\mathrm{agg}}
&\approx
\frac{1}{\dfrac{1}{C_1}+\dfrac{1}{C_2}}.
\end{aligned}
$$

---

## 11. 应用层供给限制

实际测得的 TCP goodput 还受到应用层供给速率限制：

$$
\begin{aligned}
S_{\mathrm{meas},j}
&=\min\left(S_{\mathrm{path},j}^{\mathrm{agg}},\Lambda_{\mathrm{app}}\right).
\end{aligned}
$$

批量脚本实际使用：

$$
\begin{aligned}
\Lambda_{\mathrm{app}}&=1\times20\ \mathrm{Gbps}=20\ \mathrm{Gbps}.
\end{aligned}
$$

通常：

$$
\begin{aligned}
\Lambda_{\mathrm{app}}&\gg S_{\mathrm{path},j}^{\mathrm{agg}},
\end{aligned}
$$

因此最小理论场景中可近似认为：

$$
\begin{aligned}
S_{\mathrm{meas},j}&\approx S_{\mathrm{path},j}^{\mathrm{agg}}.
\end{aligned}
$$

---

## 12. 接入选择

无线回传场景下，接入选择应比较：

$$
\begin{aligned}
S_{\mathrm{path,ONT}}^{\mathrm{agg}}
\qquad \text{and} \qquad
S_{\mathrm{path,AP,wireless}}^{\mathrm{agg}}.
\end{aligned}
$$

即：

$$
\begin{aligned}
j^*
&=\arg\max
\left\{
S_{\mathrm{path,ONT}}^{\mathrm{agg}},
S_{\mathrm{path,AP,wireless}}^{\mathrm{agg}}
\right\}.
\end{aligned}
$$

其中：

$$
\begin{aligned}
S_{\mathrm{path,ONT}}^{\mathrm{agg}}
&\approx
\frac{L_{\mathrm{payload}}^{\mathrm{agg}}}
{T_{\mathrm{fixed}}+\dfrac{L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_{\mathrm{ONT}\to\mathrm{STA}}}},\\
S_{\mathrm{path,AP,wireless}}^{\mathrm{agg}}
&\approx
\frac{L_{\mathrm{payload}}^{\mathrm{agg}}}
{2T_{\mathrm{fixed}}
+\dfrac{L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_{\mathrm{ONT}\to\mathrm{AP}}}
+\dfrac{L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_{\mathrm{AP}\to\mathrm{STA}}}}.
\end{aligned}
$$

因此，STA 是否应该接入 AP 不仅取决于 AP–STA 链路，还取决于 ONT–AP 无线回传链路。

---

## 13. 传播时延项

若需要保留传播时延，可写：

$$
\begin{aligned}
\tau_{u\to v}&=\frac{d_{u\to v}}{c},\qquad
c\approx3\times10^8\ \mathrm{m/s}.
\end{aligned}
$$

AP 双跳路径时延可写为：

$$
\begin{aligned}
D_{\mathrm{path,AP,wireless}}
&=T_{\mathrm{ONT}\to\mathrm{AP}}^{\mathrm{agg}}
+T_{\mathrm{AP}\to\mathrm{STA}}^{\mathrm{agg}}
+\tau_{\mathrm{ONT}\to\mathrm{AP}}
+\tau_{\mathrm{AP}\to\mathrm{STA}}.
\end{aligned}
$$

不过米级 Wi-Fi 场景中传播时延为纳秒级，而 MAC/PHY 开销为微秒级，因此吞吐主公式中通常可以忽略传播时延。

---

## 14. 如果考虑误包率和重传

若低 SNR、OBSS 或冲突导致误包率不可忽略，可为每一跳引入成功概率：

$$
\begin{aligned}
P_{\mathrm{succ},1}&=1-PER_{\mathrm{ONT}\to\mathrm{AP}},\\
P_{\mathrm{succ},2}&=1-PER_{\mathrm{AP}\to\mathrm{STA}}.
\end{aligned}
$$

更稳妥的重传期望服务时间写法为：

$$
\begin{aligned}
T_{\mathrm{path,AP}}^{\mathrm{eff}}
&\approx
\frac{T_{\mathrm{ONT}\to\mathrm{AP}}^{\mathrm{agg}}}{P_{\mathrm{succ},1}}
+\frac{T_{\mathrm{AP}\to\mathrm{STA}}^{\mathrm{agg}}}{P_{\mathrm{succ},2}}.
\end{aligned}
$$

于是：

$$
\begin{aligned}
S_{\mathrm{path,AP}}^{\mathrm{eff}}
&\approx
\frac{L_{\mathrm{payload}}^{\mathrm{agg}}}{T_{\mathrm{path,AP}}^{\mathrm{eff}}}.
\end{aligned}
$$

在最小无 OBSS、链路质量较好场景中，可以先令：

$$
\begin{aligned}
P_{\mathrm{succ},1}&\approx1,\qquad
P_{\mathrm{succ},2}\approx1.
\end{aligned}
$$

---

## 15. 推荐计算流程小结

无线回传场景下，推荐按以下流程计算：

```text
1. 给定 STA 坐标 (x_STA, y_STA)
2. 计算 d_ONT-STA、d_AP-STA、d_ONT-AP
3. 由每条无线链路距离计算 SNR
4. 由 SNR 选择 MCS
5. 由 MCS 得到 R_ONT→STA、R_ONT→AP、R_AP→STA
6. 代入聚合长度 L_payload^agg 和 L_PSDU^agg
7. 计算 S_path,ONT
8. 计算 S_path,AP,wireless
9. 若考虑应用层供给，取 min(S_path,j, Lambda_app)
10. 选择吞吐更大的接入对象
```

核心公式为：

$$
\begin{aligned}
S_{\mathrm{path,ONT}}^{\mathrm{agg}}
&\approx
\frac{L_{\mathrm{payload}}^{\mathrm{agg}}}
{T_{\mathrm{fixed}}+\dfrac{L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_{\mathrm{ONT}\to\mathrm{STA}}}},\\
S_{\mathrm{path,AP,wireless}}^{\mathrm{agg}}
&\approx
\frac{L_{\mathrm{payload}}^{\mathrm{agg}}}
{2T_{\mathrm{fixed}}
+\dfrac{L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_{\mathrm{ONT}\to\mathrm{AP}}}
+\dfrac{L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_{\mathrm{AP}\to\mathrm{STA}}}}.
\end{aligned}
$$

如果取：

$$
\begin{aligned}
T_{\mathrm{fixed}}&\approx223.5\ \mu s,
\end{aligned}
$$

则 AP 双跳路径固定开销为：

$$
\begin{aligned}
2T_{\mathrm{fixed}}&\approx447\ \mu s.
\end{aligned}
$$

因此无线回传下，AP 路径比有线回传多承担一整跳无线服务时间。

# 03. OBSS 场景下的聚合吞吐与接入选择

本文档说明在存在 OBSS AP / STA 干扰时，聚合吞吐和接入选择规则如何从无 OBSS 模型扩展而来。

---

## 1. 固定聚合长度

当前简化理论模型中直接取：

$$
\begin{aligned}
N_{\mathrm{MSDU},j}&=7,
\\
N_{\mathrm{MPDU},j}&=64,
\\
L_{\mathrm{MSDU}}&=1500\ \mathrm{B}.
\end{aligned}
$$

因此一次聚合中的有效负载长度为：

$$
\begin{aligned}
L_{\mathrm{payload},j}^{\mathrm{agg}}
&=64\times7\times1500
\\
&=672000\ \mathrm{B}
\\
&=5376000\ \mathrm{bit}.
\end{aligned}
$$

聚合后 PSDU 总长度取：

$$
\begin{aligned}
L_{\mathrm{PSDU},j}^{\mathrm{agg}}
&=681472\ \mathrm{B}
\\
&=5451776\ \mathrm{bit}.
\end{aligned}
$$

聚合开销为：

$$
\begin{aligned}
L_{\mathrm{oh},j}^{\mathrm{agg}}
&=681472-672000
\\
&=9472\ \mathrm{B}
\\
&=75776\ \mathrm{bit}.
\end{aligned}
$$

在后续吞吐公式中，直接使用：

$$
L_{\mathrm{payload},j}^{\mathrm{agg}}=5376000\ \mathrm{bit},
\qquad
L_{\mathrm{PSDU},j}^{\mathrm{agg}}=5451776\ \mathrm{bit}.
$$

---

## 2. 无 OBSS 聚合吞吐基线

若不考虑 OBSS 干扰，则一次成功聚合传输时间为：

$$
\begin{aligned}
T_{\mathrm{succ},j}^{\mathrm{agg}}
&=T_{\mathrm{AIFS}}+E[T_{\mathrm{bo}}]+T_{\mathrm{PHY},j}
+\frac{L_{\mathrm{PSDU},j}^{\mathrm{agg}}}{R_j}
+T_{\mathrm{SIFS}}+T_{\mathrm{BA}}.
\end{aligned}
$$

由于这里 $L_{\mathrm{PSDU},j}^{\mathrm{agg}}$ 已经使用 bit，因此不再额外乘 8。

无 OBSS 聚合吞吐为：

$$
\begin{aligned}
S_j^{\mathrm{agg}}
&=\frac{L_{\mathrm{payload},j}^{\mathrm{agg}}}{T_{\mathrm{succ},j}^{\mathrm{agg}}}
\\
&=\frac{5376000}
{T_{\mathrm{AIFS}}+E[T_{\mathrm{bo}}]+T_{\mathrm{PHY},j}
+\dfrac{5451776}{R_j}
+T_{\mathrm{SIFS}}+T_{\mathrm{BA}}}.
\end{aligned}
$$

---

## 3. OBSS 的两个影响

OBSS 干扰对主链路有两个主要影响。

### 3.1 同时发送时降低链路质量

无 OBSS 时：

$$
SNR_j\rightarrow MCS_j\rightarrow R_j.
$$

有 OBSS 时：

$$
SINR_j\rightarrow MCS_j\rightarrow R_j.
$$

因此：

$$
R_j=R_j(SINR_j).
$$

### 3.2 占用信道时间

OBSS 业务会占用一部分信道时间。当前仿真中 OBSS 理想占空比约为 $10\%-20\%$，实际约为 $10\%-15\%$。理论中可取：

$$
\rho_{\mathrm{OBSS}}\approx0.10\sim0.15.
$$

---

## 4. 简化 OBSS 吞吐模型

最简单的 OBSS 修正方式是对吞吐乘上可用信道比例：

$$
S_j^{\mathrm{agg,OBSS}}
\approx
(1-\rho_{\mathrm{OBSS}})S_j^{\mathrm{agg}}(SINR_j).
$$

代入固定聚合长度后：

$$
\begin{aligned}
S_j^{\mathrm{agg,OBSS}}
&\approx
(1-\rho_{\mathrm{OBSS}})
\frac{5376000}
{T_{\mathrm{AIFS}}+E[T_{\mathrm{bo}}]+T_{\mathrm{PHY},j}
+\dfrac{5451776}{R_j(SINR_j)}
+T_{\mathrm{SIFS}}+T_{\mathrm{BA}}}.
\end{aligned}
$$

其中：

$$
R_j(SINR_j)=R_{MCS_j}(160\ \mathrm{MHz},2,800\ \mathrm{ns}).
$$

MCS 选择为：

$$
MCS_j=\max\{m:SINR_j\ge\Gamma_m\}.
$$

---

## 5. 等效服务时间写法

也可以把 OBSS 占用看成服务时间被拉长：

$$
T_{\mathrm{succ},j}^{\mathrm{agg,OBSS}}
\approx
\frac{T_{\mathrm{succ},j}^{\mathrm{agg}}(SINR_j)}{1-\rho_{\mathrm{OBSS}}}.
$$

于是：

$$
\begin{aligned}
S_j^{\mathrm{agg,OBSS}}
&=\frac{5376000}{T_{\mathrm{succ},j}^{\mathrm{agg,OBSS}}}
\\
&\approx
(1-\rho_{\mathrm{OBSS}})
\frac{5376000}{T_{\mathrm{succ},j}^{\mathrm{agg}}(SINR_j)}.
\end{aligned}
$$

这与上一节的简化吞吐模型等价。

---

## 6. 考虑 CCA / 退避影响的写法

在 ns-3 中，OBSS 不只是降低 SINR，还可能让主链路感知到信道忙，从而增加等待和退避时间。因此更贴近仿真的写法是：

$$
\begin{aligned}
S_j^{\mathrm{agg,OBSS}}
&\approx
\frac{5376000}
{T_{\mathrm{AIFS}}+E[T_{\mathrm{bo}}]_{\mathrm{OBSS}}+T_{\mathrm{PHY},j}
+\dfrac{5451776}{R_j(SINR_j)}
+T_{\mathrm{SIFS}}+T_{\mathrm{BA}}}.
\end{aligned}
$$

其中：

$$
E[T_{\mathrm{bo}}]_{\mathrm{OBSS}}\ge E[T_{\mathrm{bo}}].
$$

简化近似中可以取：

$$
E[T_{\mathrm{bo}}]_{\mathrm{OBSS}}
\approx
\frac{E[T_{\mathrm{bo}}]}{1-\rho_{\mathrm{OBSS}}}.
$$

如果只需要快速估计，使用第 4 节的整体乘 $(1-\rho_{\mathrm{OBSS}})$ 更直观。

---

## 7. AP 路径与有线回传

若 STA 接入 ONT，下行路径为：

```text
ONT → STA
```

路径吞吐为：

$$
S_{\mathrm{path,ONT}}^{\mathrm{agg,OBSS}}
=S_{\mathrm{ONT}\to\mathrm{STA}}^{\mathrm{agg,OBSS}}.
$$

若 STA 接入 AP1，下行路径为：

```text
ONT → AP1 → STA
```

路径吞吐为：

$$
S_{\mathrm{path,AP1}}^{\mathrm{agg,OBSS}}
=\min\left(C_{\mathrm{eth}},S_{\mathrm{AP1}\to\mathrm{STA}}^{\mathrm{agg,OBSS}}\right).
$$

若 STA 接入 AP2，下行路径为：

```text
ONT → AP2 → STA
```

路径吞吐为：

$$
S_{\mathrm{path,AP2}}^{\mathrm{agg,OBSS}}
=\min\left(C_{\mathrm{eth}},S_{\mathrm{AP2}\to\mathrm{STA}}^{\mathrm{agg,OBSS}}\right).
$$

当前有线回传容量为：

$$
C_{\mathrm{eth}}=20\ \mathrm{Gbps}.
$$

通常大于单条 Wi-Fi 链路实际吞吐，因此常可近似：

$$
S_{\mathrm{path,AP1}}^{\mathrm{agg,OBSS}}
\approx S_{\mathrm{AP1}\to\mathrm{STA}}^{\mathrm{agg,OBSS}},
$$

$$
S_{\mathrm{path,AP2}}^{\mathrm{agg,OBSS}}
\approx S_{\mathrm{AP2}\to\mathrm{STA}}^{\mathrm{agg,OBSS}}.
$$

---

## 8. 应用层供给限制

当前主业务为单流下行 TCP：

$$
\Lambda_{\mathrm{app}}=1\times20\ \mathrm{Gbps}=20\ \mathrm{Gbps}.
$$

实际测得吞吐可以写成：

$$
S_{\mathrm{meas},j}
=\min\left(S_{\mathrm{path},j}^{\mathrm{agg,OBSS}},\Lambda_{\mathrm{app}}\right).
$$

通常：

$$
\Lambda_{\mathrm{app}}\gg S_{\mathrm{path},j}^{\mathrm{agg,OBSS}},
$$

因此当前场景可近似认为应用层供给充足：

$$
S_{\mathrm{meas},j}\approx S_{\mathrm{path},j}^{\mathrm{agg,OBSS}}.
$$

---

## 9. 接入选择规则

最终接入选择应按 OBSS 修正后的路径吞吐进行比较：

$$
j^*=\arg\max_{j\in\{\mathrm{ONT},\mathrm{AP1},\mathrm{AP2}\}}
S_{\mathrm{path},j}^{\mathrm{agg,OBSS}}.
$$

如果只讨论最小二选一场景，则：

$$
j^*=\arg\max_{j\in\{\mathrm{ONT},\mathrm{AP1}\}}
S_{\mathrm{path},j}^{\mathrm{agg,OBSS}}.
$$

---

## 10. 当前 OBSS 理论模型总结

最终推荐使用：

$$
\begin{aligned}
S_j^{\mathrm{agg,OBSS}}
&\approx
(1-\rho_{\mathrm{OBSS}})
\frac{5376000}
{T_{\mathrm{AIFS}}+E[T_{\mathrm{bo}}]+T_{\mathrm{PHY},j}
+\dfrac{5451776}{R_j(SINR_j)}
+T_{\mathrm{SIFS}}+T_{\mathrm{BA}}}.
\end{aligned}
$$

其中：

$$
\rho_{\mathrm{OBSS}}\approx0.10\sim0.15,
$$

$$
R_j(SINR_j)=R_{\max\{m:SINR_j\ge\Gamma_m\}}(160\ \mathrm{MHz},2,800\ \mathrm{ns}).
$$

接入选择为：

$$
j^*=\arg\max_j S_{\mathrm{path},j}^{\mathrm{agg,OBSS}}.
$$
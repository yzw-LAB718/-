# 有线回传场景下的吞吐计算要点

本文档专门总结 **ONT–AP 为有线回传** 时，当前 Wi-Fi Mesh 理论吞吐模型的计算流程、关键假设和推荐公式。

---

## 1. 场景假设

最小理论场景为：

```text
1 个 ONT + 1 个 AP + 1 个 STA
ONT 与 AP 之间采用有线连接
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
ONT → STA：无线链路
AP  → STA：无线链路
ONT → AP ：有线回传链路
```

在该场景中，**STA 接入 ONT** 是单跳无线；**STA 接入 AP** 是一段有线回传加一跳无线接入。

---

## 2. 有线回传场景的核心简化

因为当前代码中的有线回传速率通常取：

$$
C_{\mathrm{eth}}=10\ \mathrm{Gbps},
$$

有线时延通常取：

$$
\tau_{\mathrm{eth}}=500\ \mathrm{ns},
$$

而单条 Wi-Fi 链路的实际 TCP goodput 通常低于有线回传容量，所以在最小理论场景中，一般可以认为：

$$
C_{\mathrm{eth}} \gg S_{\mathrm{wifi}}.
$$

因此，当 STA 接入 AP 时，路径瓶颈通常不在 ONT–AP 有线段，而在 AP–STA 无线段。

也就是说，有线回传场景下常用近似为：

$$
S_{\mathrm{path,AP}}^{\mathrm{agg}}
\approx
S_{\mathrm{AP}\to\mathrm{STA}}^{\mathrm{agg}}.
$$

这也是有线回传和无线回传公式差异最大的地方：

```text
有线回传：AP 路径主要计算 AP → STA 一跳无线吞吐
无线回传：AP 路径需要计算 ONT → AP 和 AP → STA 两跳无线空口时间
```

---

## 3. 第一步：计算候选无线链路距离

在最小场景中，可令：

| 节点 | 坐标 |
|---|---:|
| ONT | `(0, 0)` |
| AP | `(10, 0)` |
| STA | `(x_STA, y_STA)` |

若 STA 接入 ONT，需要计算：

$$
d_{\mathrm{STA-ONT}}=\sqrt{x_{\mathrm{STA}}^2+y_{\mathrm{STA}}^2}.
$$

若 STA 接入 AP，需要计算：

$$
d_{\mathrm{STA-AP}}=\sqrt{(x_{\mathrm{STA}}-10)^2+y_{\mathrm{STA}}^2}.
$$

实际代入路径损耗时，建议使用有效距离：

$$
d_{\mathrm{eff}}=\max(d,d_0),\qquad d_0=1\ \mathrm{m}.
$$

---

## 4. 第二步：由距离计算 SNR

无 OBSS 干扰时，链路质量计算链条为：

$$
d_{\mathrm{STA}-j}
\rightarrow
PL_j(d_{\mathrm{STA}-j})
\rightarrow
P_{r,j}(d_{\mathrm{STA}-j})
\rightarrow
SNR_j(d_{\mathrm{STA}-j}).
$$

其中：

$$
j\in\{\mathrm{ONT},\mathrm{AP}\}.
$$

当前理论近似中，对数距离路径损耗可写为：

$$
PL_j(d)=PL_0+10\alpha\log_{10}\left(\frac{d}{d_0}\right).
$$

若采用当前文档中的默认近似：

$$
PL_0=46.6777\ \mathrm{dB},\qquad \alpha=3,\qquad d_0=1\ \mathrm{m},
$$

则：

$$
PL_j(d)=46.6777+30\log_{10}(d).
$$

接收功率为：

$$
P_{r,j}(d)=P_t+G_t+G_r-PL_j(d).
$$

若：

$$
P_t=20\ \mathrm{dBm},\qquad G_t=0\ \mathrm{dB},\qquad G_r=0\ \mathrm{dB},
$$

则：

$$
P_{r,j}(d)=20-PL_j(d).
$$

噪声功率为：

$$
N=-174+10\log_{10}(B)+NF.
$$

当前配置中：

$$
B=160\ \mathrm{MHz},\qquad NF=7\ \mathrm{dB},
$$

因此：

$$
N\approx-84.96\ \mathrm{dBm}.
$$

无 OBSS 时：

$$
SNR_j(d)=P_{r,j}(d)-N.
$$

代入上述默认值，可得到：

$$
SNR_j(d)\approx58.28-30\log_{10}(d).
$$

---

## 5. 第三步：由 SNR 得到 MCS 和 PHY rate

当前代码使用 `IdealWifiManager`，因此 MCS 会根据链路条件自动选择。理论上可写为：

$$
MCS_j(d)=\max_{m:\ SNR_j(d)\ge\Gamma_m}m.
$$

其中 $\Gamma_m$ 是 MCS $m$ 对应的 SNR 门限。

在当前配置下：

$$
B=160\ \mathrm{MHz},\qquad NSS=2,\qquad GI=800\ \mathrm{ns}.
$$

所以 PHY rate 可写成：

$$
R_j(d)=R_{MCS_j(d)}(160\ \mathrm{MHz},2,800\ \mathrm{ns}).
$$

对于有线回传场景，需要分别得到：

$$
R_{\mathrm{ONT}\to\mathrm{STA}}
=R_{MCS_{\mathrm{ONT}\to\mathrm{STA}}}(d_{\mathrm{STA-ONT}})},
$$

$$
R_{\mathrm{AP}\to\mathrm{STA}}
=R_{MCS_{\mathrm{AP}\to\mathrm{STA}}}(d_{\mathrm{STA-AP}})}.
$$

注意：有线回传场景下，一般不需要计算 ONT–AP 的无线 PHY rate，因为 ONT–AP 是有线链路。

---

## 6. 第四步：聚合负载取值

当前简化理论模型中，可取：

$$
N_{\mathrm{MSDU},j}=7,
$$

$$
N_{\mathrm{MPDU},j}=64,
$$

$$
L_{\mathrm{MSDU}}=1500\ \mathrm{B}.
$$

因此一次聚合中的有效负载为：

$$
\begin{aligned}
L_{\mathrm{payload},j}^{\mathrm{agg}}
&=N_{\mathrm{MPDU},j}N_{\mathrm{MSDU},j}L_{\mathrm{MSDU}} \\
&=64\times7\times1500 \\
&=672000\ \mathrm{B} \\
&=5376000\ \mathrm{bit}.
\end{aligned}
$$

若单个 A-MPDU 子单元长度近似取：

$$
L_{\mathrm{AMPDU,sub},j}=10648\ \mathrm{B},
$$

则聚合后 PSDU 总长度为：

$$
\begin{aligned}
L_{\mathrm{PSDU},j}^{\mathrm{agg}}
&=N_{\mathrm{MPDU},j}L_{\mathrm{AMPDU,sub},j} \\
&=64\times10648 \\
&=681472\ \mathrm{B} \\
&=5451776\ \mathrm{bit}.
\end{aligned}
$$

聚合开销为：

$$
\begin{aligned}
L_{\mathrm{oh},j}^{\mathrm{agg}}
&=L_{\mathrm{PSDU},j}^{\mathrm{agg}}-L_{\mathrm{payload},j}^{\mathrm{agg}} \\
&=681472-672000 \\
&=9472\ \mathrm{B} \\
&=75776\ \mathrm{bit}.
\end{aligned}
$$

理论推导时也可以保留 $N_{\mathrm{MPDU},j}$ 为变量，写成：

$$
L_{\mathrm{payload},j}^{\mathrm{agg}}\approx10500N_{\mathrm{MPDU},j}\ \mathrm{B},
$$

$$
L_{\mathrm{PSDU},j}^{\mathrm{agg}}\approx10648N_{\mathrm{MPDU},j}\ \mathrm{B}.
$$

---

## 7. 第五步：单跳无线聚合服务时间

对任意候选接入点 $j$，一跳无线聚合服务时间为：

$$
\begin{aligned}
T_{\mathrm{succ},j}^{\mathrm{agg}}
&=T_{\mathrm{AIFS}}
+E[T_{\mathrm{bo}}]
+T_{\mathrm{PHY},j}
+\frac{L_{\mathrm{PSDU},j}^{\mathrm{agg}}}{R_j(d_j)}
+T_{\mathrm{SIFS}}
+T_{\mathrm{BA}}.
\end{aligned}
$$

若 $L_{\mathrm{PSDU},j}^{\mathrm{agg}}$ 用 bit 表示，上式可直接使用。

若 $L_{\mathrm{PSDU},j}^{\mathrm{agg}}$ 用 Byte 表示，则应写成：

$$
\begin{aligned}
T_{\mathrm{succ},j}^{\mathrm{agg}}
&=T_{\mathrm{AIFS}}
+E[T_{\mathrm{bo}}]
+T_{\mathrm{PHY},j}
+\frac{8L_{\mathrm{PSDU},j}^{\mathrm{agg}}}{R_j(d_j)}
+T_{\mathrm{SIFS}}
+T_{\mathrm{BA}}.
\end{aligned}
$$

当前理论模型中建议取：

$$
T_{\mathrm{AIFS}}=43\ \mu s,
$$

$$
E[T_{\mathrm{bo}}]=\frac{CW_{\min}}{2}\sigma=\frac{15}{2}\times9\ \mu s=67.5\ \mu s,
$$

$$
T_{\mathrm{PHY},j}\approx52\ \mu s,
$$

$$
T_{\mathrm{SIFS}}=16\ \mu s.
$$

仓库目前尚未固定 $T_{\mathrm{BA}}$ 的精确数值。理论计算中可先使用：

$$
T_{\mathrm{BA}}\approx40\ \mu s,
$$

或做敏感性分析：

$$
T_{\mathrm{BA}}\in[30,60]\ \mu s.
$$

若取 $T_{\mathrm{BA}}=40\ \mu s$，则固定开销为：

$$
\begin{aligned}
T_{\mathrm{fixed}}
&=T_{\mathrm{AIFS}}+E[T_{\mathrm{bo}}]+T_{\mathrm{PHY}}+T_{\mathrm{SIFS}}+T_{\mathrm{BA}} \\
&=43+67.5+52+16+40 \\
&=218.5\ \mu s.
\end{aligned}
$$

于是单跳无线吞吐可近似写成：

$$
S_j^{\mathrm{agg}}(d_j)
\approx
\frac{L_{\mathrm{payload},j}^{\mathrm{agg}}}
{T_{\mathrm{fixed}}+\dfrac{L_{\mathrm{PSDU},j}^{\mathrm{agg}}}{R_j(d_j)}}.
$$

其中 $L_{\mathrm{payload},j}^{\mathrm{agg}}$ 和 $L_{\mathrm{PSDU},j}^{\mathrm{agg}}$ 均使用 bit 表示。

---

## 8. 第六步：路径吞吐计算

### 8.1 STA 接入 ONT

STA 接入 ONT 时，路径为：

```text
ONT → STA
```

因此：

$$
S_{\mathrm{path,ONT}}^{\mathrm{agg}}
=
S_{\mathrm{ONT}\to\mathrm{STA}}^{\mathrm{agg}}(d_{\mathrm{STA-ONT}}).
$$

展开为：

$$
S_{\mathrm{path,ONT}}^{\mathrm{agg}}
\approx
\frac{L_{\mathrm{payload}}^{\mathrm{agg}}}
{T_{\mathrm{fixed}}+\dfrac{L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_{\mathrm{ONT}\to\mathrm{STA}}}}.
$$

### 8.2 STA 接入 AP

STA 接入 AP 时，路径为：

```text
ONT → AP → STA
```

其中 ONT–AP 是有线回传，AP–STA 是无线接入。因此路径吞吐为：

$$
S_{\mathrm{path,AP}}^{\mathrm{agg}}
=
\min\left(
C_{\mathrm{eth}},
S_{\mathrm{AP}\to\mathrm{STA}}^{\mathrm{agg}}(d_{\mathrm{STA-AP}})
\right).
$$

由于当前常有：

$$
C_{\mathrm{eth}}=10\ \mathrm{Gbps}\gg S_{\mathrm{AP}\to\mathrm{STA}}^{\mathrm{agg}},
$$

所以可近似为：

$$
S_{\mathrm{path,AP}}^{\mathrm{agg}}
\approx
S_{\mathrm{AP}\to\mathrm{STA}}^{\mathrm{agg}}(d_{\mathrm{STA-AP}}).
$$

展开为：

$$
S_{\mathrm{path,AP}}^{\mathrm{agg}}
\approx
\frac{L_{\mathrm{payload}}^{\mathrm{agg}}}
{T_{\mathrm{fixed}}+\dfrac{L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_{\mathrm{AP}\to\mathrm{STA}}}}.
$$

---

## 9. 第七步：应用层供给限制

实际测得的 TCP goodput 还受到应用层供给速率限制。可写成：

$$
S_{\mathrm{meas},j}
=
\min\left(S_{\mathrm{path},j}^{\mathrm{agg}},\Lambda_{\mathrm{app}}\right).
$$

批量脚本实际使用：

$$
\Lambda_{\mathrm{app}}=1\times20\ \mathrm{Gbps}=20\ \mathrm{Gbps}.
$$

由于通常有：

$$
\Lambda_{\mathrm{app}}\gg S_{\mathrm{path},j}^{\mathrm{agg}},
$$

所以最小理论场景中可近似认为：

$$
S_{\mathrm{meas},j}\approx S_{\mathrm{path},j}^{\mathrm{agg}}.
$$

---

## 10. 第八步：接入选择

有线回传场景中，STA 接入选择可写为：

$$
j^*=\arg\max_{j\in\{\mathrm{ONT},\mathrm{AP}\}}S_{\mathrm{path},j}^{\mathrm{agg}}.
$$

即比较：

$$
S_{\mathrm{path,ONT}}^{\mathrm{agg}}
\approx
\frac{L_{\mathrm{payload}}^{\mathrm{agg}}}
{T_{\mathrm{fixed}}+\dfrac{L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_{\mathrm{ONT}\to\mathrm{STA}}}},
$$

和：

$$
S_{\mathrm{path,AP}}^{\mathrm{agg}}
\approx
\frac{L_{\mathrm{payload}}^{\mathrm{agg}}}
{T_{\mathrm{fixed}}+\dfrac{L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_{\mathrm{AP}\to\mathrm{STA}}}}.
$$

因为两者的固定开销和聚合长度相同，所以在最简化情况下，接入选择基本由无线 PHY rate 决定：

$$
R_{\mathrm{ONT}\to\mathrm{STA}}
\quad \text{vs.} \quad
R_{\mathrm{AP}\to\mathrm{STA}}.
$$

但更严格地说，仍应比较路径吞吐 $S_{\mathrm{path},j}^{\mathrm{agg}}$，而不是只比较距离或 SNR。

---

## 11. 有线回传场景下不需要做的事情

在 ONT–AP 为有线回传时，理论模型中通常不需要：

```text
1. 计算 ONT → AP 的无线 SNR；
2. 计算 ONT → AP 的无线 MCS；
3. 将 ONT → AP 和 AP → STA 两段无线空口时间相加；
4. 对 AP 路径额外乘以同信道双跳折损；
5. 将 AP 路径吞吐写成双跳无线 harmonic mean。
```

这些是无线回传场景才需要考虑的问题。

---

## 12. 推荐计算流程小结

有线回传场景下，推荐按以下流程计算：

```text
1. 给定 STA 坐标 (x_STA, y_STA)
2. 计算 d_STA-ONT 和 d_STA-AP
3. 由距离计算 SNR_ONT 和 SNR_AP
4. 由 SNR 选择 MCS
5. 由 MCS 得到 R_ONT→STA 和 R_AP→STA
6. 代入聚合长度 L_payload^agg 和 L_PSDU^agg
7. 计算 S_path,ONT 和 S_path,AP
8. 若考虑应用层供给，取 min(S_path,j, Lambda_app)
9. 选择吞吐更大的接入点
```

核心公式为：

$$
\boxed{
S_{\mathrm{path,ONT}}^{\mathrm{agg}}
\approx
\frac{L_{\mathrm{payload}}^{\mathrm{agg}}}
{T_{\mathrm{fixed}}+\dfrac{L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_{\mathrm{ONT}\to\mathrm{STA}}}}
}
$$

$$
\boxed{
S_{\mathrm{path,AP}}^{\mathrm{agg}}
\approx
\frac{L_{\mathrm{payload}}^{\mathrm{agg}}}
{T_{\mathrm{fixed}}+\dfrac{L_{\mathrm{PSDU}}^{\mathrm{agg}}}{R_{\mathrm{AP}\to\mathrm{STA}}}}
}
$$

其中：

$$
T_{\mathrm{fixed}}\approx218.5\ \mu s
$$

是采用：

$$
T_{\mathrm{BA}}\approx40\ \mu s
$$

时的理论近似值。如果希望保守处理，可令：

$$
T_{\mathrm{fixed}}\in[210,240]\ \mu s.
$$

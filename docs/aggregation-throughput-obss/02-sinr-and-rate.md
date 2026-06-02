# 02. OBSS 场景下的 SINR、MCS 与 PHY rate

本文档说明在存在 OBSS AP / STA 干扰时，候选接入点链路的 SINR、MCS 和 PHY rate 如何建模。

---

## 1. 原无干扰链条

无 OBSS 干扰时，链路质量链条为：

$$
\begin{aligned}
d_{\mathrm{STA}-j}
&\rightarrow SNR_j
\rightarrow MCS_j
\rightarrow R_j.
\end{aligned}
$$

其中：

$$
j\in\{\mathrm{ONT},\mathrm{AP1},\mathrm{AP2}\}.
$$

无干扰时可写为：

$$
SNR_j=P_{r,j}-N.
$$

这里所有功率均使用 dBm 或 dB 表示。

---

## 2. 加入 OBSS 后的链条

有 OBSS 同频干扰后，需要改成：

$$
\begin{aligned}
d_{\mathrm{STA}-j}
&\rightarrow P_{r,j}
\rightarrow SINR_j
\rightarrow MCS_j
\rightarrow R_j.
\end{aligned}
$$

其中：

$$
SINR_j=P_{r,j}-P_{N+I}.
$$

这里 $P_{N+I}$ 是噪声功率和干扰功率在线性域相加后再转回 dBm 的结果。

---

## 3. 主链路接收功率

候选接入点 $j$ 到目标 STA 的接收功率为：

$$
\begin{aligned}
P_{r,j}(d_{\mathrm{STA}-j})
&=P_{t,j}+G_{t,j}+G_{r,\mathrm{STA}}-PL_j(d_{\mathrm{STA}-j}).
\end{aligned}
$$

当前简化模型中可取：

$$
P_{t,j}=20\ \mathrm{dBm},\qquad G_{t,j}=0\ \mathrm{dB},\qquad G_{r,\mathrm{STA}}=0\ \mathrm{dB}.
$$

路径损耗采用对数距离模型：

$$
\begin{aligned}
PL_j(d)
&=PL_0+10\alpha\log_{10}\left(\frac{d_{\mathrm{eff}}}{d_0}\right),
\\
d_{\mathrm{eff}}&=\max(d,d_0),
\\
d_0&=1\ \mathrm{m}.
\end{aligned}
$$

若按 ns-3 默认 LogDistancePropagationLossModel 近似，可取：

$$
PL_0\approx46.6777\ \mathrm{dB},\qquad \alpha=3.
$$

于是：

$$
PL_j(d)\approx46.6777+30\log_{10}(d_{\mathrm{eff}}).
$$

---

## 4. 噪声功率

噪声功率为：

$$
N=-174+10\log_{10}(B)+NF.
$$

其中 $B$ 使用 Hz，当前：

$$
B=160\times10^6\ \mathrm{Hz}.
$$

若取接收机噪声系数：

$$
NF=7\ \mathrm{dB},
$$

则：

$$
\begin{aligned}
N
&=-174+10\log_{10}(160\times10^6)+7
\\
&\approx -84.96\ \mathrm{dBm}.
\end{aligned}
$$

---

## 5. OBSS 干扰功率

OBSS AP 到目标 STA 的距离为：

$$
\begin{aligned}
d_{\mathrm{OBSSAP-STA}}
&=\sqrt{(x_{\mathrm{STA}}-4)^2+(y_{\mathrm{STA}}-4)^2}.
\end{aligned}
$$

OBSS AP 在目标 STA 处产生的干扰功率为：

$$
\begin{aligned}
I_{\mathrm{OBSS}}
&=P_{t,\mathrm{OBSS}}+G_{t,\mathrm{OBSS}}+G_{r,\mathrm{STA}}-PL(d_{\mathrm{OBSSAP-STA}}).
\end{aligned}
$$

当前简化模型中可取：

$$
P_{t,\mathrm{OBSS}}=20\ \mathrm{dBm},\qquad G_{t,\mathrm{OBSS}}=0\ \mathrm{dB},\qquad G_{r,\mathrm{STA}}=0\ \mathrm{dB}.
$$

因此：

$$
I_{\mathrm{OBSS}}\approx20-PL(d_{\mathrm{OBSSAP-STA}}).
$$

---

## 6. 噪声与干扰的合成功率

注意：噪声和干扰都是功率，不能直接在 dBm 中相加。需要先转到线性域：

$$
\begin{aligned}
P_{N+I}
&=10\log_{10}\left(10^{N/10}+10^{I_{\mathrm{OBSS}}/10}\right).
\end{aligned}
$$

因此：

$$
\begin{aligned}
SINR_j
&=P_{r,j}-10\log_{10}\left(10^{N/10}+10^{I_{\mathrm{OBSS}}/10}\right).
\end{aligned}
$$

这个式子是 OBSS 场景相对无 OBSS 场景最重要的修改。

---

## 7. 各候选接入点的 SINR

### 7.1 STA 接入 ONT

主链路为：

```text
ONT → STA
```

接收功率为：

$$
P_{r,\mathrm{ONT}}=20-PL(d_{\mathrm{STA-ONT}}).
$$

因此：

$$
\begin{aligned}
SINR_{\mathrm{ONT}}
&=P_{r,\mathrm{ONT}}
-10\log_{10}\left(10^{N/10}+10^{I_{\mathrm{OBSS}}/10}\right).
\end{aligned}
$$

### 7.2 STA 接入 AP1

主链路为：

```text
AP1 → STA
```

接收功率为：

$$
P_{r,\mathrm{AP1}}=20-PL(d_{\mathrm{STA-AP1}}).
$$

因此：

$$
\begin{aligned}
SINR_{\mathrm{AP1}}
&=P_{r,\mathrm{AP1}}
-10\log_{10}\left(10^{N/10}+10^{I_{\mathrm{OBSS}}/10}\right).
\end{aligned}
$$

### 7.3 STA 接入 AP2

主链路为：

```text
AP2 → STA
```

接收功率为：

$$
P_{r,\mathrm{AP2}}=20-PL(d_{\mathrm{STA-AP2}}).
$$

因此：

$$
\begin{aligned}
SINR_{\mathrm{AP2}}
&=P_{r,\mathrm{AP2}}
-10\log_{10}\left(10^{N/10}+10^{I_{\mathrm{OBSS}}/10}\right).
\end{aligned}
$$

同一 STA 位置下，三个候选接入点面对的 $I_{\mathrm{OBSS}}$ 相同，因为接收端都是同一个 STA；不同的是主链路接收功率 $P_{r,j}$。

---

## 8. MCS 选择

无 OBSS 时，MCS 可写为：

$$
MCS_j=\max\{m:SNR_j\ge\Gamma_m\}.
$$

有 OBSS 时，需要改为：

$$
MCS_j=\max\{m:SINR_j\ge\Gamma_m\}.
$$

其中 $\Gamma_m$ 是第 $m$ 个 MCS 的 SINR / SNR 门限。若使用 IdealWifiManager，可理解为由目标 BER 反推得到的门限。

---

## 9. PHY rate 计算

当前无线配置固定为：

$$
B=160\ \mathrm{MHz},\qquad NSS=2,
\qquad GI=800\ \mathrm{ns}.
$$

因此：

$$
R_j(SINR_j)=R_{MCS_j}(160\ \mathrm{MHz},2,800\ \mathrm{ns}).
$$

也可以写成：

$$
R_j(SINR_j)=R_{\max\{m:SINR_j\ge\Gamma_m\}}(160\ \mathrm{MHz},2,800\ \mathrm{ns}).
$$

---

## 10. 当前模型需要重新计算的结果

对每个 STA 网格点，需要计算：

$$
\begin{aligned}
d_{\mathrm{STA-ONT}},\quad
d_{\mathrm{STA-AP1}},\quad
d_{\mathrm{STA-AP2}},\quad
d_{\mathrm{OBSSAP-STA}},
\end{aligned}
$$

然后计算：

$$
\begin{aligned}
P_{r,\mathrm{ONT}},\quad
P_{r,\mathrm{AP1}},\quad
P_{r,\mathrm{AP2}},\quad
I_{\mathrm{OBSS}},\quad
SINR_{\mathrm{ONT}},\quad
SINR_{\mathrm{AP1}},\quad
SINR_{\mathrm{AP2}}.
\end{aligned}
$$

最后得到：

$$
\begin{aligned}
MCS_{\mathrm{ONT}},\quad
MCS_{\mathrm{AP1}},\quad
MCS_{\mathrm{AP2}},\quad
R_{\mathrm{ONT}},\quad
R_{\mathrm{AP1}},\quad
R_{\mathrm{AP2}}.
\end{aligned}
$$
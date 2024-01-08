# MOS小信号模型

## Notations

![notations](img/mos-notations.png)

## Understanding the concept of small signal model

### Transient response

![transient](img/mos-transient-response.png)

$$
v_{in} = v_a \sin \omega t
$$

$$
v_{out} = -i_dR_d = -\mu_n C_{ox} \frac{W}{L} (V_{GS} - V_{TH})v_a \sin \omega t
$$

Amplification factor: $\mu_n C_{ox} \frac{W}{L} (V_{GS} - V_{TH})$

## Transconductance of MOS

$$
g_m = \mu_n C_{ox} \frac{W}{L} (V_{GS} - V_{TH}) = \sqrt{2 \mu_n C_{ox} \frac{W}{L} I_D} = \frac{2I_D}{V_{GS}- V_{TH}}
$$

### Three elements for calculating $g_m$

* There are only two independent elements in $W/L$, $I_D$ and $V_{GS} - V_{TH}$.
* Any one can be derived from the other two.

![three](img/three-elements.png)

## Ideal small signal model of MOS

![basic](img/basic-small.png)

## Second order effect

### Body effect

当$V_{S}$大于$V_{B}$时，会发生什么？$V_{TH}$变大。

$$
V_{T H}=V_{T H 0}+\gamma\left(\sqrt{\left|2 \Phi_F+V_{S B}\right|}-\sqrt{\left|2 \Phi_F\right|}\right)
$$

$\gamma$ is called the body effect coefficient, 跟工艺有关。

#### Influence of body effect on small signal model: $g_{mb}$

$g_{mb}$: body transconductance

$$
g_{mb} = \frac{\partial I_D}{\partial V_{SB}} 
$$

### Channel length modulation effect

* A NMOS operating in the saturation region with large enough length
  * $I_D$ is independent of $V_{DS}$.
  * MOS成为理想电流源。
  * Infinite output resistance(因为两端电压变化后，电流却不变)

The influence of $V_{DS}$ on $I_D$ is called **channel length modulation effect**.

$$
I_D \approx \frac{1}{2} \mu_n C_{o x} \frac{W}{L}\left(V_{G S}-V_{T H}\right)^2\left(1+\lambda V_{D S}\right)
$$

沟道长度调制效应的结果是，即使在饱和区，漏极电流（$I_D$）依然轻微依赖于漏极电压（$V_{DS}$）

定义finite output resistance $r_o$, $g_{ds} = \frac{1}{r_o}$

$$
r_{o}=\frac{1}{\lambda I_{D}}
$$

Important note:

**MOS small-signal output resistance can be changed by adjusting bias current $I_D$,
or transistor length $L$.**
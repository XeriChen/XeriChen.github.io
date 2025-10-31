+++
date = '2025-10-31T21:41:19+08:00'
draft = false
title = 'phy'
+++

{{< katex >}}

## 1，

简答什么是施主、施主电离；什么是受主、受主电离；分别画出浅施主能级和浅受主能级。

### 答

施主杂质：在半导体晶体中电离时，能够释放电⼦⽽产⽣导电电⼦并形成正电中⼼的杂质称为施主杂质。

施主电离：电⼦脱离杂质原⼦的束缚成为导电电⼦的过程称为杂质电离。

受主杂质：在半导体晶体中电离时，能够接受电⼦⽽产⽣导电空⽳并形成负电中⼼的杂质称为受主杂质。

受主电离：空⽳挣脱受主杂质束缚的过程称为受主电离。

浅施主能级和浅受主能级

![image-20250923201617653](https://raw.githubusercontent.com/XeriChen/picPicture/main/myPicture/2025/09/23/1758629777788.png)

![image-20250923201531476](https://raw.githubusercontent.com/XeriChen/picPicture/main/myPicture/2025/09/23/1758629738588.png)



## 2，

推导 $p_0$，$n_i$, $E_i$。

### 答

#### 状态密度

**自由电子能量**
$$
\varepsilon = \frac{\hbar^2 k^2}{2m}
$$

**法一**
$$
\sum_k{\rightarrow \frac{V}{\left( 2\pi \right) ^3}\int{4\pi k^2dk}}=\frac{V}{\left( 2\pi \right) ^3}\int{4\pi k\frac{m}{\hbar ^2}d\varepsilon}=\frac{V}{\left( 2\pi \right) ^3}\int{4\pi \frac{\sqrt{2m\varepsilon}}{\hbar}\frac{m}{\hbar ^2}d\varepsilon}=\int{g(\varepsilon )d\varepsilon}
$$

$$
g(\varepsilon )=2\times g(\varepsilon )=\frac{V}{\pi ^2\hbar ^3}\sqrt{2m^3\varepsilon}
$$

**法二**

简并度2，$k$ 空间密度 $\rho_k$ ，三维球体积 $V_k$
$$
Z=2 \rho_k V_k=2\times\frac{V}{\left( 2\pi \right) ^3}\times \frac{4\pi}{3}\left(\frac{2m\varepsilon}{\hbar^2}\right)^{3/2}
$$
$$
g(\varepsilon)=\frac{dZ}{d\varepsilon}=\frac{V}{2\pi^2}\frac{(2m)^{3/2}}{\hbar^3}(\varepsilon)^{1/2}
$$

#### 玻尔兹曼分布

电子费米分布
$$
f(E)=\frac{1}{1+e^{\frac{E-E_F}{k_0T}}}
$$
空穴
$$
1-f(E)=\frac{1}{1+e^{\frac{E_F-E}{k_0T}}}
$$

当 $E-E_F \gg k_0 T$ 转化为玻尔兹曼分布
$$
f(E)=e^{\frac{-E+E_F}{k_0T}}
$$

$$
1-f(E)=e^{\frac{E-E_F}{k_0T}}
$$

#### 空⽳浓度

⾮简并半导体的价带中的空⽳浓度 $p_0$ 为
$$
p_0=\int_{E_{\mathrm{v}}^{\prime}}^{E_{\mathrm{v}}}{\left[ 1-f(E) \right] \frac{g_{\mathrm{v}}(E)}{V}\mathrm{d}E}
$$
解得
$$
\begin{aligned}
p_0&=2\left( \frac{m_{\mathrm{p}}^{*}\:k_0T}{2\pi \hbar ^2} \right) ^{3/2}\exp \left( \frac{E_{\mathrm{v}}-E_{\mathrm{F}}}{k_0T} \right) 
\\
&=N_{\mathrm{v}}\exp \left( \frac{E_{\mathrm{v}}-E_{\mathrm{F}}}{k_0T} \right)
\end{aligned}
$$
#### 本征载流⼦浓度

令 $n_0=p_0$ 本征载流⼦浓度 $n_i$ 为
$$
n_{\mathrm{i}}=n_0=p_0=\left( N_{\mathrm{c}}N_{\mathrm{v}} \right) ^{1/2}\exp \left( -\frac{E_{\mathrm{g}}}{2k_0T} \right)
$$
本征半导体的费⽶能级
$$
E_{\mathrm{i}}=E_{\mathrm{F}}=\frac{E_{\mathrm{c}}+E_{\mathrm{v}}}{2}+\frac{3k_0T}{4}\ln \frac{m_{\mathrm{p}}^{*}}{m_{\mathrm{n}}^{*}}
$$


## 3，

有效状态密度 $N_\mathrm{c}$ , $N_V$ 的物理意义？

### 答

> 导带的有效状态密度 $N_\mathrm{c}$ : 按照玻⽿兹曼分布函数, 电⼦在导带内占据量⼦态的概率随量⼦态具有的能量升⾼⽽迅速下降, 可以近似认为导带中的所有量⼦态都集中在导带底 $E_c$ 附近, 因此 , 导带的有效状态密度也就是导带的有效能级密度。
>

导带的有效状态密度 $N_\mathrm{c}$ 理解为把导带中所有量⼦态都集中在导带底 $E_c$ 处 , ⽽它的状态密度为 $N_c$，则导带中的电⼦浓度是 $N_c$ 中有电⼦占据的量⼦态数。
$$
n_0=N_cf(E_c)
$$
导带的有效状态密度 $N_\mathrm{v}$ ，可以理解为把导带中所有量⼦态都集中在导带底 $E_v$ 处 , ⽽它的状态密度为 $N_v$
$$
p_0=N_v f(E_v)
$$

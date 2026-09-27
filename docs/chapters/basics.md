# IBL
## 总览
Image-Based Lighting（IBL）使用 environment map 表示来自远处环境的入射辐射度，并据此计算表面的环境光照。

对于 diffuse BRDF，环境光积分可以直接整理为只与表面法线有关的 irradiance；而对于 specular BRDF，积分还依赖观察方向、roughness、Fresnel 等参数，无法直接预计算为一张普通的 environment map。

下面从渲染方程出发，依次推导 diffuse IBL、specular IBL 的 Monte Carlo estimator，以及实时渲染中常用的 Split Sum 近似，并说明 prefiltered environment map 和 BRDF LUT 分别是如何得到的。

## 渲染方程
在表面某一点上

$$
L_{o}(V)=L_{emissive}(V)+\int_\Omega L_i(L)f(L,V)(N\cdot L) dL
$$
## 环境光
**将入射辐射度 $L_i$ 分为直接光照和环境光照，这里只考虑后者，用$E(L)$表示**

$$
L_{o,env}(V)=\int_\Omega E(L)f(L,V)(N\cdot L) dL
$$

**将环境照明表示为位于无穷远处的方向辐射场，则$E(L)$与所在点无关只与方向$L$有关，记录在environment map上**
## Diffuse BRDF
将 BRDF 分为 diffuse 和 specular 两部分。先考虑 diffuse BRDF

$$f_{diffuse}(L,V)=\frac{c}{\pi}$$

$c$是材质的diffuse albedo（漫反射率），则

$$
L_{o,env,diffuse}=\int_\Omega E(L)\frac{c}{\pi}(N\cdot L)dL=\frac{c}{\pi}\int_\Omega E(L)(N\cdot L)dL
$$

对于给定的 environment map，上式不再依赖材质参数和观察方向，只依赖表面法线 $N$。因此可以预先计算 $E_{\mathrm{irr}}(N)$，并用 cubemap 存储，这就是 irradiance map

## Specular BRDF
对于Cook-Torrance微表面模型

$$
f_s(L,V)=\frac{D(H)F(V,H)G(L,V,H)}{4(N\cdot V)(N\cdot L)}
$$


$$
L_{o,env,specular}(V)=\int_\Omega E(L)\frac{D(H)F(V,H)G(L,V,H)}{4(N\cdot V)}dL
$$

## 蒙特卡洛积分
对于给定的 $V,N,F_0,\alpha$ 和环境函数 $E$，将 specular IBL 的 integrand 记作

$$g(L)=E(L)f_s(L,V;N,F_0,\alpha)(N\cdot L)$$

此时唯一的积分变量是入射方向 \(L\)。如果 \(g\) 是非负标量函数，零方差 importance sampling 对应的理想 PDF 满足

$$p^*(L)\propto g(L)$$

但这个 PDF 同时受到环境光和完整 BRDF 的影响，既难以构造，也难以直接采样。因此实际选择一个容易采样、并能够捕获 specular lobe 主要形状的 proposal PDF $q(L)$。

$$
q(L|V,N,\alpha)
$$

用q作为pdf，进行蒙特卡洛积分，

$$
L_o(V)\approx\frac{1}{M}\sum_{k=1}^M\frac{E(L_k)D(h_k)F(V,h_k)G(L_k,V,h_k)}{4(N\cdot V)\times q(L_k|V,N,\alpha)}
$$

那么如何找到这样的 $q$ 呢
## 对H采样
$D(H)\,d\omega_H$ 描述单位宏观表面上，法线方向落在 $d\omega_H$ 内的微表面面积；$D(H)$ 是面积关于方向的密度，不能直接当作随机方向 $H$ 的 PDF。如果直接以 $D(H)$ 为方向采样权重，需要先将它归一化：

$$
p_H^{(D)}(H)=\frac{D(H)}{\int_{\Omega^+}D(\omega)\,d\omega}
$$

**这个归一化形式不好处理**：NDF 已知的归一化关系并不是 $\int_{\Omega^+}D(H)\,d\omega_H=1$，所以分母不能直接消掉。若继续采用这种采样方式，还需要额外计算 $\int D\,d\omega$，并为归一化后的 $p_H^{(D)}$ 构造采样方法。

为了避开这项额外归一化，改为**按微表面沿宏观法线 $N$ 的投影面积采样**。NDF 天然满足投影归一化关系：

$$
\int_{\Omega^+}D(H)(N\cdot H)\,d\omega_H=1
$$

因此，$D(H)$ 乘上面积投影因子 $N\cdot H$，就得到关于方向立体角 $d\omega_H$ 归一化的 PDF：

$$
p_H(H|N,\alpha)=D(H;N,\alpha)(N\cdot H)
$$

这个分布对应的随机实验是：在单位宏观表面的投影区域内均匀选一个点，找到投影到该点的微表面面元，将该面元的法线作为随机变量 $H$。

某个方向的面元贡献的投影面积是其真实面积乘以 $N\cdot H$，因此 $H$ 落在方向微元 $d\omega_H$ 内的概率为 $D(H)(N\cdot H)\,d\omega_H$。这里的 $H$ 是**按投影面积选中面元后得到的法线方向**。

$$
q_L(L|V,N,\alpha)=\frac{D(H;N,\alpha)(N\cdot H)}{4(V\cdot H)}
$$

将$q_L$代入上面的蒙特卡洛积分，

$$
\begin{aligned}L_o(V)&\approx\frac{1}{M}\sum_{k=1}^M\frac{E(L_k)DFG}{4(N\cdot V)\times q_L(L_k|V,N,\alpha)}\\&\approx\frac{1}{M}\sum_{k=1}^M[\frac{E(L_k)DFG}{4(N\cdot V)}\times\frac{4(V\cdot H_k)}{D(N\cdot H_k)}]\\&\approx\frac{1}{M}\sum_{k=1}^ME(L_k)\frac{FG(V\cdot H_k)}{(N\cdot V)(N\cdot H_k)}\end{aligned}
$$

## Split Sum
至此可以引入 Split Sum 近似，将环境光项和BRDF项拆开

$$
L_{o}(V)\approx(\frac{1}{M}\sum_{k=1}^ME(L_k))\times(\frac{1}{M}\sum_{k=1}^M\frac{FG(V\cdot H_k)}{(N\cdot V)(N\cdot H_k)})
$$

Split Sum 的本质，是把同一个采样分布下“环境辐射 × BRDF 权重”的期望，近似成两个期望的乘积，从而把环境和材质响应分开预计算
## 环境光项
采样得到的 $L_k$ 由 $V$、$N$ 和 $\alpha$ 决定，因此环境项可以记为`

$$
P(V,N,\alpha)\approx \frac{1}{M}\sum_{k=1}^ME(L_k)
$$

但该环境项仍同时依赖 V、N 和 α，无法直接存入二维方向索引的 cubemap

进一步采用 N=V=R 的近似，其中 R 表示预滤波 cubemap 当前查询的反射方向

$$P'(R,\alpha)=P(R,R,\alpha)\approx \frac{1}{M}\sum_{k=1}^ME(L_k)$$

该近似用 N=V=R 时的采样分布代替实际 N、V 下的分布。其误差在 grazing angle 处尤其明显，grazing angle 时真实 lobe 会更拉长、偏斜，而预滤波 cubemap 里用的是 \(N=V\) 的近似形状。

## 具体采样方法
下面把前面构造的半角向量分布 $p_H(H)=D(H)(N\cdot H)$ 落实为采样过程：先生成半角向量 $H$，再结合观察方向 $V$ 得到入射方向 $L$。对于各向同性 GGX，NDF 为

$$
D(H)=\frac{\alpha^2}{\pi[(N\cdot H)^2(\alpha^2-1)+1]^2}
$$

代入投影面积采样的 PDF，得到 $H$ 关于立体角的方向密度：

$$
p_H(H|N,\alpha)=\frac{\alpha^2(N\cdot H)}{\pi[(N\cdot H)^2(\alpha^2-1)+1]^2}
$$

取两个独立的均匀随机数 $\xi_1,\xi_2\sim U(0,1)$，按以下四步从这个分布生成 $L$：

1. **求 $\theta$**。各向同性使方位角 $\phi$ 均匀分布，因此取 $\phi=2\pi\xi_1$。在以 $N$ 为极轴的球坐标中，$d\omega_H=\sin\theta\,d\theta\,d\phi$。将方向密度对方位角积分后，极角的累积分布满足

    $$
    F_\theta(\theta)=\int_0^\theta\frac{2\alpha^2\cos t\sin t}{[\cos^2t(\alpha^2-1)+1]^2}\,dt=\xi_2
    $$

    令 $x=\sin^2t$，则 $dx=2\sin t\cos t\,dt$，上式化为

    $$
    \begin{aligned}
    F_\theta(\theta)
    &=\int_0^{\sin^2\theta}\frac{\alpha^2}{[(1-x)(\alpha^2-1)+1]^2}\,dx\\
    &=\left.\frac{x}{\alpha^2+(1-\alpha^2)x}\right|_0^{\sin^2\theta}\\
    &=\frac{\sin^2\theta}{\alpha^2+(1-\alpha^2)\sin^2\theta}
    \end{aligned}
    $$

    令累积概率等于 $\xi_2$ 并反解，得到

    $$
    \sin^2\theta=\frac{\alpha^2\xi_2}{1+(\alpha^2-1)\xi_2}
    $$

2. **构造局部 $H$**。把 $N$ 作为局部坐标系的 $z$ 轴，用采样得到的 $\theta$ 和 $\phi$ 写出单位半角向量：

    $$
    H_{\mathrm{local}}=(\sin\theta\cos\phi,\ \sin\theta\sin\phi,\ \cos\theta)
    $$

3. **转到世界空间**。以切线 $T$、副切线 $B$ 和法线 $N$ 构造正交基，将局部向量的三个分量分别放到这三个轴上：

    $$
    H=(\sin\theta\cos\phi)T+(\sin\theta\sin\phi)B+(\cos\theta)N
    $$

4. **根据 $V,H$ 得到 $L$**。把 $V$ 视为从表面指向观察者的方向，将 $-V$ 绕世界空间的 $H$ 反射，得到入射方向：

    $$
    L=2(V\cdot H)H-V
    $$

    若 $N\cdot L\leq 0$，该方向不属于表面上半球，不参与此处的环境光积分。


## BRDF项
Split Sum 的第二项只包含 BRDF 权重，与环境贴图无关。它相当于在环境辐射恒为 1 时积分镜面 BRDF：

$$
I_{\mathrm{BRDF}}(N,V,\alpha,F_0)
=\int_{\Omega^+}f_s(L,V)(N\cdot L)\,d\omega_L
$$

采用 Schlick Fresnel，并记 $F_c=(1-V\cdot H)^5$，则

$$
F(V,H)=F_0+(1-F_0)F_c=F_0(1-F_c)+F_c
$$

BRDF 的其余部分不依赖 $F_0$，因此可以把它从积分中分离出来，将结果写成一个关于 $F_0$ 的线性形式：

$$
I_{\mathrm{BRDF}}=F_0\,A(N\cdot V,\alpha)+B(N\cdot V,\alpha)
$$

这里的 $A$ 和 $B$ 分别是 $F_0$ 的系数与常数项。实际预计算时仍使用前面从 GGX 分布得到的样本 $H_k,L_k$，而不直接求解连续积分。为缩短求和式，定义每个有效样本的几何权重和 Fresnel 系数：

$$
G_{\mathrm{vis},k}
=\frac{G(L_k,V,H_k)(V\cdot H_k)}{(N\cdot V)(N\cdot H_k)}
$$

$$
F_{c,k}=(1-V\cdot H_k)^5
$$

于是两个预计算量分别为

$$
\begin{aligned}
A(N\cdot V,\alpha)
&\approx\frac{1}{M}\sum_{k=1}^{M}(1-F_{c,k})G_{\mathrm{vis},k},\\
B(N\cdot V,\alpha)
&\approx\frac{1}{M}\sum_{k=1}^{M}F_{c,k}G_{\mathrm{vis},k}.
\end{aligned}
$$

若某次采样得到 $N\cdot L_k\leq 0$，该样本的权重按 0 计。由于 BRDF 各向同性，$A$ 和 $B$ 只需以 $N\cdot V$ 和粗糙度参数 $\alpha$ 为输入，可预先存入一张双通道的 2D BRDF LUT。运行时，将它与前面的预滤波环境光 $P'(R,\alpha)$ 组合：

$$
\begin{aligned}
L_{o,\mathrm{env,specular}}(V)
\approx P'(R,\alpha)\cdot\bigl[F_0A(N\cdot V,\alpha)+B(N\cdot V,\alpha)\bigr].
\end{aligned}
$$



## 总结
至此，diffuse 和 specular 两部分的环境光积分都被转化为了适合实时计算的预计算表示。

Diffuse IBL 将环境光在半球上的 cosine-weighted 积分存入 irradiance map。Specular IBL 则通过 GGX importance sampling 构造 Monte Carlo estimator，再利用 Split Sum 将环境项与 BRDF 项近似分离：前者进一步采用 N=V=R 的近似，预计算为不同 roughness 下的 prefiltered environment map；后者利用 Fresnel 对 F0 的线性形式，将结果预计算为一张由 roughness 和 N·V 索引的 2D BRDF LUT。

因此，运行时不需要重新计算完整的环境光积分，只需要查询这些预计算结果并组合得到最终的 IBL。整个过程的核心，是通过选择合适的采样分布和降维近似，将原本依赖多个变量的积分转化为可以实时查询的低维表示。

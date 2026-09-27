# IBL

## 总览

Image-Based Lighting（IBL）用 environment map 记录远处环境从各个方向射来的辐射度，再把它代入渲染方程，计算表面的环境光照。

漫反射积分可以整理成**只随表面法线变化的环境照度**；镜面反射还受观察方向和材质参数影响，需要进一步采样和近似，才能预计算供实时渲染使用。

> [!NOTE] 阅读路线
> 先从渲染方程中取出环境光项，再处理 Diffuse IBL。Specular IBL 从 Monte Carlo 估计出发，构造 Half-Vector 采样分布，用 Split Sum 分离环境项和 BRDF 项，最后得到预滤波环境贴图与 BRDF Integration LUT。

## 渲染方程与环境光

设 $N$ 为表面法线，$V$ 为从表面指向观察者的方向，$L$ 为从表面指向入射光源的方向，$\Omega^+$ 为 $N$ 上方的半球。在表面某一点，渲染方程写为

$$
L_o(V)=L_e(V)+\int_{\Omega^+}L_i(L)f(L,V)(N\cdot L)\,d\omega_L
$$

其中 $d\omega_L$ 是入射方向的立体角微元。本文只讨论环境光，用 $E(L)$ 表示其入射辐射度。把环境视为无穷远的方向辐射场后，**$E(L)$ 只依赖方向 $L$**，可由 environment map 存储。于是

$$
L_{o,\mathrm{env}}(V)=\int_{\Omega^+}E(L)f(L,V)(N\cdot L)\,d\omega_L
$$

下面分别计算 BRDF 中的漫反射项和镜面反射项。

## Diffuse IBL

对 Lambert 漫反射，设 $c$ 为材质的漫反射反照率（diffuse albedo），有

$$f_{\mathrm{diffuse}}(L,V)=\frac{c}{\pi}$$

将 $f_{\mathrm{diffuse}}=c/\pi$ 代入环境光积分，材质项可以移到积分外。剩下的环境照度**只依赖表面法线 $N$**，因此可**预先计算并存入 irradiance map**：

$$
E_{\mathrm{irr}}(N)=\int_{\Omega^+}E(L)(N\cdot L)\,d\omega_L
$$

运行时按 $N$ 查询这张贴图，再乘上材质的漫反射项：

$$
L_{o,\mathrm{env,diffuse}}=\frac{c}{\pi}E_{\mathrm{irr}}(N)
$$

## Specular IBL

镜面部分采用 Cook-Torrance 微表面模型。令 $H=(L+V)/\lVert L+V\rVert$ 为半角向量；$D$、$F$、$G$ 分别是法线分布函数（NDF）、Fresnel 项和几何遮蔽项：

$$
f_s(L,V)=\frac{D(H)F(V,H)G(L,V,H)}{4(N\cdot V)(N\cdot L)}
$$

代入环境光积分并约去 $N\cdot L$，得到

$$
L_{o,\mathrm{env,specular}}(V)
=\int_{\Omega^+}E(L)\frac{D(H)F(V,H)G(L,V,H)}{4(N\cdot V)}\,d\omega_L
$$

以下用 $F_0$ 表示法线入射时的 Fresnel 反射率，用 $\alpha$ 表示 GGX NDF 的粗糙度参数；若材质使用感知粗糙度 $r$，UE4 采用 $\alpha=r^2$。这个积分**同时依赖环境、视角和材质**。我们先给出采样估计，再把其中的**环境响应与 BRDF 响应分开预计算**。

### Monte Carlo 估计

固定 $V$、$N$、$F_0$、$\alpha$ 和环境函数 $E$。先看单个颜色通道，把镜面环境光积分的被积函数记作

$$g(L)=E(L)f_s(L,V;N,F_0,\alpha)(N\cdot L)$$

此时只有 $L$ 是积分变量。对非负的 $g$，理想的重要性采样密度应满足

$$p^*(L)\propto g(L)$$

但这个密度既依赖环境光，又依赖完整 BRDF，**难以预先构造和采样**。实际改用只依赖 $V$、$N$、$\alpha$ 的密度 $q(L\mid V,N,\alpha)$，使样本集中于镜面反射波瓣附近。从 $q$ 生成 $M$ 个入射方向后，估计式为

$$
L_{o,\mathrm{env,specular}}(V)\approx\frac{1}{M}\sum_{k=1}^M\frac{g(L_k)}{q(L_k\mid V,N,\alpha)}
$$

下一步不直接采样 $L$：先采样微表面法线 $H$，再由反射关系得到 $L$，从而构造 $q$。

### 构造 Half-Vector 采样分布

$D(H)\,d\omega_H$ 描述单位宏观面积上，法线方向落在 $d\omega_H$ 内的微表面面积。它是面积密度，**不能直接当作随机方向 $H$ 的 PDF**。如果坚持以 $D(H)$ 为采样权重，必须另算归一化常数：

$$
p_H^{(D)}(H)=\frac{D(H)}{\int_{\Omega^+}D(\omega)\,d\omega}
$$

NDF 已知的归一化关系并不是 $\int_{\Omega^+}D(H)\,d\omega_H=1$，因此不能省去分母。这里改为**按微表面沿宏观法线 $N$ 的投影面积采样**，直接利用 NDF 满足的关系：

$$
\int_{\Omega^+}D(H)(N\cdot H)\,d\omega_H=1
$$

微表面面元在 $N$ 方向的投影面积是其面积乘以 $N\cdot H$。因此，**按投影面积选面元**时，$H$ 关于立体角的方向 PDF 为

$$
p_H(H|N,\alpha)=D(H;N,\alpha)(N\cdot H)
$$

> [!NOTE] 随机变量 $H$ 的含义
> 在单位宏观表面的投影区域内均匀选一个点，找到对应的微表面面元，并把它的法线记为随机变量 $H$。
>
> 方向落在 $d\omega_H$ 内的面元贡献的投影面积是 $D(H)(N\cdot H)\,d\omega_H$，也就是抽到这一方向的概率。

给定 $V$，由 $H$ 反射得到 $L=2(V\cdot H)H-V$。对有效的反射方向，立体角变换的雅可比为 $d\omega_L=4(V\cdot H)\,d\omega_H$，所以入射方向的 PDF 是

$$
q_L(L|V,N,\alpha)=\frac{D(H;N,\alpha)(N\cdot H)}{4(V\cdot H)}
$$

当 $D$ 取 GGX NDF 时，$p_H$ 就是本文采用的 **GGX Half-Vector 重要性采样分布**，而 $q_L$ 是它诱导出的入射方向分布。样本跟随 GGX 的 $D$ 项，更多地落在镜面反射波瓣附近；$F$ 和 $G$ 不在采样 PDF 中，仍保留在后面的样本权重里。具体生成 $H$ 的方法见「GGX Half-Vector 采样」一节。

将 $q_L$ 代入 Monte Carlo 估计式，$D(H_k)$ 与 PDF 中的对应因子抵消：

$$
\frac{D(H_k)}{4q_L(L_k\mid V,N,\alpha)}
=\frac{V\cdot H_k}{N\cdot H_k}
$$

因此，每个有效样本只需乘上环境辐射度和余下的 BRDF 权重 $W_k$：

$$
W_k=\frac{F(V,H_k)G(L_k,V,H_k)(V\cdot H_k)}{(N\cdot V)(N\cdot H_k)}
$$

镜面环境光的估计式便成为

$$
L_{o,\mathrm{env,specular}}(V)\approx\frac{1}{M}\sum_{k=1}^ME(L_k)W_k
$$

### Split Sum 近似

刚得到的求和式是同一批样本上 $E(L_k)W_k$ 的平均值。为使环境与材质可以分别预计算，Split Sum 将它**近似为两个平均值的乘积**。令

$$
\bar E=\frac{1}{M}\sum_{k=1}^M E(L_k),
\qquad
\bar W=\frac{1}{M}\sum_{k=1}^M W_k
$$

则

$$
L_{o,\mathrm{env,specular}}(V)\approx\bar E\,\bar W
$$

> [!IMPORTANT] Split Sum 是近似
> $\overline{E W}$ 通常不等于 $\bar E\,\bar W$：这一步忽略了环境辐射与 BRDF 权重随方向变化时的相关性。当环境辐射为常数时，拆分是精确的。Prefiltered Environment Map 近似 $\bar E$，BRDF Integration LUT 近似 $\bar W$。

### Prefiltered Environment Map

第一项 $\bar E$ 是在 GGX 采样方向上对环境辐射度求平均。样本方向由 $V$、$N$ 和 $\alpha$ 决定，因此先记为

$$
P(V,N,\alpha)\approx \frac{1}{M}\sum_{k=1}^ME(L_k)
$$

但 **$P$ 同时依赖 $V$、$N$ 和 $\alpha$**；若直接存储，无法只用一张按方向与粗糙度查询的 cubemap 表示。

于是令 $R=2(N\cdot V)N-V$ 为镜面反射方向，并在预计算时**近似取 $N=V=R$**。对每个 $R$ 和 $\alpha$ 生成相应的 $L_k$，得到

$$
P'(R,\alpha)=P(R,R,\alpha)
\approx\frac{1}{M}\sum_{k=1}^{M}E(L_k)
$$

这样可以把不同粗糙度的结果存入 cubemap 的不同 mip 级别。上式保留 Split Sum 推导中的简单平均；Karis 给出的预滤波实现还会对有效样本按 $N\cdot L_k$ 加权，并除以权重之和。具体的 GGX 样本生成方法见下一节。

> [!WARNING] 掠射角误差
> 这里用 $N=V=R$ 的分布代替真实 $N$、$V$ 对应的分布。掠射角附近真实的反射波瓣会拉长、偏斜，而预滤波 cubemap 无法保留这种形状，因此误差更明显。

### GGX Half-Vector 采样

上一节构造了 Half-Vector 的方向 PDF；这里以各向同性 GGX 为例，给出实际的生成步骤。先按 $p_H(H)=D(H)(N\cdot H)$ 生成 $H$，再由 $V$ 和 $H$ 求 $L$。GGX NDF 为

$$
D(H)=\frac{\alpha^2}{\pi[(N\cdot H)^2(\alpha^2-1)+1]^2}
$$

代入投影面积采样的 PDF，得到 $H$ 关于立体角的方向密度：

$$
p_H(H|N,\alpha)=\frac{\alpha^2(N\cdot H)}{\pi[(N\cdot H)^2(\alpha^2-1)+1]^2}
$$

取两个独立的均匀随机数 $\xi_1,\xi_2\sim U(0,1)$，按以下四步生成样本方向：

1. **求 $\theta$**。各向同性使方位角 $\phi$ 均匀分布，因此取 $\phi=2\pi\xi_1$。在以 $N$ 为极轴的球坐标中，$d\omega_H=\sin\theta\,d\theta\,d\phi$。对方位角积分，极角的累积分布为

    $$
    F_\theta(\theta)=\int_0^\theta\frac{2\alpha^2\cos t\sin t}{[\cos^2t(\alpha^2-1)+1]^2}\,dt
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

    最后令 $F_\theta(\theta)=\xi_2$ 并反解，得到

    $$
    \sin^2\theta=\frac{\alpha^2\xi_2}{1+(\alpha^2-1)\xi_2}
    $$

2. **构造局部 $H$**。令局部坐标系的 $z$ 轴对齐 $N$，用刚得到的 $\theta$ 和 $\phi$ 写出单位半角向量：

    $$
    H_{\mathrm{local}}=(\sin\theta\cos\phi,\ \sin\theta\sin\phi,\ \cos\theta)
    $$

3. **转到世界空间**。用切线 $T$、副切线 $B$ 和法线 $N$ 构造正交基，把局部向量的三个分量映射到世界空间：

    $$
    H=(\sin\theta\cos\phi)T+(\sin\theta\sin\phi)B+(\cos\theta)N
    $$

4. **根据 $V,H$ 得到 $L$**。将入射到微表面的方向 $-V$ 绕 $H$ 反射，得到从表面指向环境的采样方向：

    $$
    L=2(V\cdot H)H-V
    $$

    若 $V\cdot H\leq 0$ 或 $N\cdot L\leq 0$，该样本对积分的贡献按 0 计。生成 Prefiltered Environment Map 时取 $N=V=R$；生成 BRDF Integration LUT 时使用 LUT 当前对应的 $N\cdot V$ 和粗糙度。

### BRDF Integration LUT

Split Sum 的第二项 $\bar W$ **只包含 BRDF 权重，与环境贴图无关**。把环境辐射设为常数 1，就得到它所对应的连续积分：

$$
I_{\mathrm{BRDF}}(N,V,\alpha,F_0)
=\int_{\Omega^+}f_s(L,V)(N\cdot L)\,d\omega_L
$$

为使这个积分能够存入二维 LUT，采用 Schlick Fresnel。记 $F_c=(1-V\cdot H)^5$，则

$$
F(V,H)=F_0+(1-F_0)F_c=F_0(1-F_c)+F_c
$$

由于 $D$ 和 $G$ 都不依赖 $F_0$，代入积分后可以把 $F_0$ 提到外面，得到**关于 $F_0$ 的线性形式**：

$$
I_{\mathrm{BRDF}}=F_0\,A(N\cdot V,\alpha)+B(N\cdot V,\alpha)
$$

其中 $A$ 是 $F_0$ 的系数，$B$ 是常数项。预计算仍使用前述 GGX 样本 $H_k,L_k$，对积分作数值估计。为把每个样本的权重写得简洁，定义

$$
G_{\mathrm{vis},k}
=\frac{G(L_k,V,H_k)(V\cdot H_k)}{(N\cdot V)(N\cdot H_k)}
$$

$$
F_{c,k}=(1-V\cdot H_k)^5
$$

因为 $W_k=F(V,H_k)G_{\mathrm{vis},k}$，两个通道的估计式分别为

$$
\begin{aligned}
A(N\cdot V,\alpha)
&\approx\frac{1}{M}\sum_{k=1}^{M}(1-F_{c,k})G_{\mathrm{vis},k},\\
B(N\cdot V,\alpha)
&\approx\frac{1}{M}\sum_{k=1}^{M}F_{c,k}G_{\mathrm{vis},k}.
\end{aligned}
$$

对于不满足 $V\cdot H_k>0$ 或 $N\cdot L_k>0$ 的样本，求和项按 0 计。由于所用 BRDF 各向同性，$A$、$B$ 对 $N$ 和 $V$ 的依赖只通过 $N\cdot V$ 表达，因此可按 $(N\cdot V,\alpha)$ 预计算成**双通道的 2D BRDF LUT**。

> [!TIP] 运行时查询
> 用 $(N\cdot V,\alpha)$ 查询 LUT，得到 $A$、$B$；用 $(R,\alpha)$ 查询预滤波环境贴图，得到 $P'(R,\alpha)$。运行时将两者组合：
>
> $$
> \begin{aligned}
> L_{o,\mathrm{env,specular}}(V)
> &\approx P'(R,\alpha)\\
> &\quad\cdot\bigl[F_0A(N\cdot V,\alpha)+B(N\cdot V,\alpha)\bigr].
> \end{aligned}
> $$

## 总结

Diffuse IBL 将只依赖法线方向的环境照度存入 irradiance map。Specular IBL 先用 GGX Half-Vector 重要性采样估计积分，再用 Split Sum 将环境辐射与 BRDF 权重分开：环境项经过 $N=V=R$ 近似存入 Prefiltered Environment Map；BRDF 项借助 Schlick Fresnel 对 $F_0$ 的线性形式存入 BRDF Integration LUT。

运行时分别查询 irradiance map、预滤波环境贴图和 BRDF LUT，即可组合出环境漫反射与环境镜面反射，**无需逐像素重新计算完整的环境光积分**。

## 参考文献

- Brian Karis，[*Real Shading in Unreal Engine 4*](https://cdn2.unrealengine.com/Resources/files/2013SiggraphPresentationsNotes-26915738.pdf)，SIGGRAPH 2013 课程 *Physically Based Shading in Theory and Practice* 讲义，Epic Games，2013。

# IBL

## 总览

Image-Based Lighting（IBL）用 environment map 记录远处环境从各个方向射来的辐射度，再把它代入渲染方程，计算表面的环境光照。

漫反射积分可以整理成**只随表面法线变化的环境照度**；镜面反射还受观察方向和材质参数影响，需要进一步采样和近似，才能预计算供实时渲染使用。

> [!NOTE] 阅读路线
> 先从渲染方程中取出环境光项，再处理 [Diffuse IBL](#diffuse-ibl)。[Specular IBL](#specular-ibl) 从 Monte Carlo 估计出发，构造 Half-Vector 分布并给出 GGX 采样步骤，随后用 [Split Sum](#split-sum) 分离环境项和 BRDF 项，得到预滤波环境贴图与 BRDF Integration LUT。

## 渲染方程与环境光

设 $n$ 为表面法线，$\omega_o$ 为从表面指向观察者的方向，$\omega_i$ 为从表面指向入射光源的方向，$\Omega^+$ 为 $n$ 上方的半球。在表面某一点，渲染方程写为

<a id="eq-rendering"></a>

$$
L_o(\omega_o)=L_e(\omega_o)+\int_{\Omega^+}L_i(\omega_i)f(\omega_i,\omega_o)(n\cdot \omega_i)\,d\omega_i
\tag{1}\label{eq:rendering}
$$

[渲染方程](#eq-rendering)中的入射辐射度 $L_i$ 包含直接光和环境光。本文只讨论后者。若把环境视为位于无穷远处的辐射场，环境辐射度便只随入射方向变化，可记为 $E(\omega_i)$ 并存入 environment map。只取积分中由 $E(\omega_i)$ 产生的出射辐射度，得到

<a id="eq-environment"></a>

$$
L_{o,\mathrm{env}}(\omega_o)=\int_{\Omega^+}E(\omega_i)f(\omega_i,\omega_o)(n\cdot \omega_i)\,d\omega_i
\tag{2}\label{eq:environment}
$$

下面分别计算 BRDF 中的漫反射项和镜面反射项。

## Diffuse IBL

对 Lambert 漫反射，设 $c$ 为材质的漫反射反照率（diffuse albedo），有

$$f_{\mathrm{diffuse}}(\omega_i,\omega_o)=\frac{c}{\pi} \tag{3}$$

将 $f_{\mathrm{diffuse}}=c/\pi$ 代入[环境光积分](#eq-environment)，材质项可以移到积分外。剩下的环境照度**只依赖表面法线 $n$**，因此可**预先计算并存入 irradiance map**：

<a id="eq-irradiance"></a>

$$
E_{\mathrm{irr}}(n)=\int_{\Omega^+}E(\omega_i)(n\cdot \omega_i)\,d\omega_i
\tag{4}\label{eq:irradiance}
$$

这个积分相当于对环境图做余弦加权的半球滤波。滤波会抑制环境中的高频变化，使 $E_{\mathrm{irr}}(n)$ 随法线方向缓慢变化，因此 **irradiance map 通常可以用比原环境图低得多的分辨率存储**。

生成贴图时，对每个待存储的法线方向 $n$，在其上半球按余弦权重进行重要性采样：

<a id="eq-cosine-pdf"></a>

$$
p_{\mathrm{diffuse}}(\omega_i\mid n)=\frac{n\cdot\omega_i}{\pi}
\tag{5}\label{eq:cosine-pdf}
$$

按[余弦加权分布](#eq-cosine-pdf)取得 $M$ 个方向后，采样 PDF 与[环境照度积分](#eq-irradiance)中的余弦因子相消，得到

$$
E_{\mathrm{irr}}(n)\approx\frac{\pi}{M}\sum_{k=1}^{M}E(\omega_{i,k})
\tag{6}\label{eq:irradiance-estimate}
$$

运行时按 $n$ 采样 irradiance map，再乘上材质的漫反射项：

$$
L_{o,\mathrm{env,diffuse}}=\frac{c}{\pi}E_{\mathrm{irr}}(n)
\tag{7}\label{eq:diffuse-runtime}
$$

## Specular IBL

镜面部分采用 Cook-Torrance 微表面模型。令 $h=(\omega_i+\omega_o)/\lVert \omega_i+\omega_o\rVert$ 为半角向量；$D$、$F$、$G$ 分别是法线分布函数（NDF）、Fresnel 项和几何遮蔽项：

<a id="eq-microfacet-brdf"></a>

$$
f_s(\omega_i,\omega_o)=\frac{D(h)F(\omega_o,h)G(\omega_i,\omega_o,h)}{4(n\cdot \omega_o)(n\cdot \omega_i)}
\tag{8}\label{eq:microfacet-brdf}
$$

将[微表面 BRDF](#eq-microfacet-brdf) 代入[环境光积分](#eq-environment)，约去 $n\cdot \omega_i$，得到

$$
L_{o,\mathrm{env,specular}}(\omega_o)
=\int_{\Omega^+}E(\omega_i)\frac{D(h)F(\omega_o,h)G(\omega_i,\omega_o,h)}{4(n\cdot \omega_o)}\,d\omega_i
\tag{9}\label{eq:specular-integral}
$$

以下用 $F_0$ 表示法线入射时的 Fresnel 反射率，用 $\alpha$ 表示 GGX NDF 的粗糙度参数；若材质使用感知粗糙度 $r$，UE4 采用 $\alpha=r^2$。这个积分**同时依赖环境、视角和材质**。我们先给出采样估计，再把其中的**环境响应与 BRDF 响应分开预计算**。

### Monte Carlo 估计

固定 $\omega_o$、$n$、$F_0$、$\alpha$ 和环境函数 $E$。先看单个颜色通道，把镜面环境光积分的被积函数记作

$$g(\omega_i)=E(\omega_i)f_s(\omega_i,\omega_o;n,F_0,\alpha)(n\cdot \omega_i) \tag{10}$$

此时只有 $\omega_i$ 是积分变量。对非负的 $g$，理想的重要性采样密度应满足

$$p^*(\omega_i)\propto g(\omega_i) \tag{11}$$

但这个密度既依赖环境光，又依赖完整 BRDF，**难以预先构造和采样**。实际改用只依赖 $\omega_o$、$n$、$\alpha$ 的密度 $q(\omega_i\mid \omega_o,n,\alpha)$，使样本集中于镜面反射波瓣附近。从 $q$ 生成 $M$ 个入射方向后，估计式为

<a id="eq-mc-estimate"></a>

$$
L_{o,\mathrm{env,specular}}(\omega_o)\approx\frac{1}{M}\sum_{k=1}^M\frac{g(\omega_{i,k})}{q(\omega_{i,k}\mid \omega_o,n,\alpha)}
\tag{12}\label{eq:mc-estimate}
$$

下一步不直接采样 $\omega_i$：先采样微表面法线 $h$，再由反射关系得到 $\omega_i$，从而构造 $q$。

### 构造 Half-Vector 采样分布

$D(h)\,d\omega_h$ 描述单位宏观面积上，法线方向落在 $d\omega_h$ 内的微表面面积。它是面积密度，**不能直接当作随机方向 $h$ 的 PDF**。如果坚持以 $D(h)$ 为采样权重，必须另算归一化常数：

$$
p_h^{(D)}(h)=\frac{D(h)}{\int_{\Omega^+}D(\omega)\,d\omega}
\tag{13}
$$

NDF 已知的归一化关系并不是 $\int_{\Omega^+}D(h)\,d\omega_h=1$，因此不能省去分母。这里改为**按微表面沿宏观法线 $n$ 的投影面积采样**，直接利用 NDF 满足的关系：

$$
\int_{\Omega^+}D(h)(n\cdot h)\,d\omega_h=1
\tag{14}\label{eq:ndf-projection}
$$

微表面面元在 $n$ 方向的投影面积是其面积乘以 $n\cdot h$。因此，**按投影面积选面元**时，$h$ 关于立体角的方向 PDF 为

<a id="eq-half-vector-pdf"></a>

$$
p_h(h\mid n,\alpha)=D(h;n,\alpha)(n\cdot h)
\tag{15}\label{eq:half-vector-pdf}
$$

> [!NOTE] 随机变量 $h$ 的含义
> 在单位宏观表面的投影区域内均匀选一个点，找到对应的微表面面元，并把它的法线记为随机变量 $h$。
>
> 方向落在 $d\omega_h$ 内的面元贡献的投影面积是 $D(h)(n\cdot h)\,d\omega_h$，也就是抽到这一方向的概率。

给定 $\omega_o$，由 $h$ 反射得到 $\omega_i=2(\omega_o\cdot h)h-\omega_o$。对有效的反射方向，立体角变换的雅可比为 $d\omega_i=4(\omega_o\cdot h)\,d\omega_h$，所以入射方向的 PDF 是

<a id="eq-incident-pdf"></a>

$$
q(\omega_i\mid\omega_o,n,\alpha)=\frac{D(h;n,\alpha)(n\cdot h)}{4(\omega_o\cdot h)}
\tag{16}\label{eq:incident-pdf}
$$

当 $D$ 取 GGX NDF 时，$p_h$ 就是本文采用的 **GGX Half-Vector 重要性采样分布**，而 $q$ 是它诱导出的入射方向分布。样本跟随 GGX 的 $D$ 项，更多地落在镜面反射波瓣附近；$F$ 和 $G$ 不在采样 PDF 中，仍保留在后面的样本权重里。

将[入射方向的 PDF](#eq-incident-pdf) 代入[Monte Carlo 估计](#eq-mc-estimate)，$D(h_k)$ 与 PDF 中的对应因子抵消：

$$
\frac{D(h_k)}{4q(\omega_{i,k}\mid \omega_o,n,\alpha)}
=\frac{\omega_o\cdot h_k}{n\cdot h_k}
\tag{17}
$$

因此，每个有效样本只需乘上环境辐射度和余下的 BRDF 权重 $W_k$：

$$
W_k=\frac{F(\omega_o,h_k)G(\omega_{i,k},\omega_o,h_k)(\omega_o\cdot h_k)}{(n\cdot \omega_o)(n\cdot h_k)}
\tag{18}\label{eq:sample-weight}
$$

镜面环境光的估计式便成为

<a id="eq-specular-estimate"></a>

$$
L_{o,\mathrm{env,specular}}(\omega_o)\approx\frac{1}{M}\sum_{k=1}^ME(\omega_{i,k})W_k
\tag{19}\label{eq:specular-estimate}
$$

下一节给出从 GGX 分布实际生成 $h$ 和 $\omega_i$ 的步骤。

### GGX Half-Vector 采样

上一节构造了 Half-Vector 的方向 PDF；这里以各向同性 GGX 为例，给出实际的生成步骤。先按 $p_h(h)=D(h)(n\cdot h)$ 生成 $h$，再由 $\omega_o$ 和 $h$ 求 $\omega_i$。GGX NDF 为

<a id="eq-ggx-ndf"></a>

$$
D(h)=\frac{\alpha^2}{\pi[(n\cdot h)^2(\alpha^2-1)+1]^2}
\tag{20}\label{eq:ggx-ndf}
$$

将[GGX NDF](#eq-ggx-ndf) 代入[Half-Vector 的投影面积采样分布](#eq-half-vector-pdf)，得到 $h$ 关于立体角的方向密度：

$$
p_h(h\mid n,\alpha)=\frac{\alpha^2(n\cdot h)}{\pi[(n\cdot h)^2(\alpha^2-1)+1]^2}
\tag{21}\label{eq:ggx-pdf}
$$

取两个独立的均匀随机数 $\xi_1,\xi_2\sim U(0,1)$，按以下四步生成样本方向：

1. **求 $\theta$**。各向同性使方位角 $\phi$ 均匀分布，因此取 $\phi=2\pi\xi_1$。在以 $n$ 为极轴的球坐标中，$d\omega_h=\sin\theta\,d\theta\,d\phi$。对方位角积分，极角的累积分布为

    $$
    F_\theta(\theta)=\int_0^\theta\frac{2\alpha^2\cos t\sin t}{[\cos^2t(\alpha^2-1)+1]^2}\,dt
    \tag{22}
    $$

    令 $x=\sin^2t$，则 $dx=2\sin t\cos t\,dt$，上式化为

    $$
    \begin{aligned}
    F_\theta(\theta)
    &=\int_0^{\sin^2\theta}\frac{\alpha^2}{[(1-x)(\alpha^2-1)+1]^2}\,dx\\
    &=\left.\frac{x}{\alpha^2+(1-\alpha^2)x}\right|_0^{\sin^2\theta}\\
    &=\frac{\sin^2\theta}{\alpha^2+(1-\alpha^2)\sin^2\theta}
    \end{aligned}
    \tag{23}
    $$

    最后令 $F_\theta(\theta)=\xi_2$ 并反解，得到

    $$
    \sin^2\theta=\frac{\alpha^2\xi_2}{1+(\alpha^2-1)\xi_2}
    \tag{24}\label{eq:ggx-theta}
    $$

2. **构造局部 $h$**。令局部坐标系的 $z$ 轴对齐 $n$，用刚得到的 $\theta$ 和 $\phi$ 写出单位半角向量：

    $$
    h_{\mathrm{local}}=(\sin\theta\cos\phi,\ \sin\theta\sin\phi,\ \cos\theta)
    \tag{25}
    $$

3. **转到世界空间**。用切线 $T$、副切线 $B$ 和法线 $n$ 构造正交基，把局部向量的三个分量映射到世界空间：

    $$
    h=(\sin\theta\cos\phi)T+(\sin\theta\sin\phi)B+(\cos\theta)n
    \tag{26}
    $$

4. **根据 $\omega_o,h$ 得到 $\omega_i$**。将入射到微表面的方向 $-\omega_o$ 绕 $h$ 反射，得到从表面指向环境的采样方向：

    $$
    \omega_i=2(\omega_o\cdot h)h-\omega_o
    \tag{27}\label{eq:reflection-direction}
    $$

    若 $\omega_o\cdot h\leq 0$ 或 $n\cdot \omega_i\leq 0$，该样本对积分的贡献按 0 计。

由此得到的方向样本可用于[镜面环境光的 Monte Carlo 估计](#eq-specular-estimate)。下面再用 Split Sum 将环境辐射与 BRDF 权重分开预计算。

### Split Sum 近似

[镜面环境光估计](#eq-specular-estimate)对同一批样本的 $E(\omega_{i,k})W_k$ 求平均。为使环境与材质可以分别预计算，Split Sum 将它**近似为两个平均值的乘积**。令

$$
\bar E=\frac{1}{M}\sum_{k=1}^M E(\omega_{i,k}),
\qquad
\bar W=\frac{1}{M}\sum_{k=1}^M W_k
\tag{28}
$$

则

$$
L_{o,\mathrm{env,specular}}(\omega_o)\approx\bar E\,\bar W
\tag{29}\label{eq:split-sum}
$$

> [!IMPORTANT] Split Sum 是近似
> $\overline{E W}$ 通常不等于 $\bar E\,\bar W$：这一步忽略了环境辐射与 BRDF 权重随方向变化时的相关性。当环境辐射为常数时，拆分是精确的。[Prefiltered Environment Map](#prefiltered-environment-map) 近似 $\bar E$，[BRDF Integration LUT](#brdf-integration-lut) 近似 $\bar W$。

### Prefiltered Environment Map

Split Sum 的环境项 $\bar E$ 是在 GGX 采样方向上对环境辐射度求平均。样本方向由 $\omega_o$、$n$ 和 $\alpha$ 决定，因此先记为

$$
P(\omega_o,n,\alpha)\approx \frac{1}{M}\sum_{k=1}^ME(\omega_{i,k})
\tag{30}
$$

但 **$P$ 同时依赖 $\omega_o$、$n$ 和 $\alpha$**；若直接存储，一张以方向和粗糙度为索引的 cubemap 无法表示这些结果。

于是令 $R=2(n\cdot \omega_o)n-\omega_o$ 为镜面反射方向，并在预计算时**近似取 $n=\omega_o=R$**。对每个 $R$ 和 $\alpha$ 生成相应的 $\omega_{i,k}$，得到

$$
P'(R,\alpha)=P(R,R,\alpha)
\approx\frac{1}{M}\sum_{k=1}^{M}E(\omega_{i,k})
\tag{31}\label{eq:prefiltered-map}
$$

这样可以把不同粗糙度的结果存入 cubemap 的不同 mip 级别。上式保留 Split Sum 推导中的简单平均；Karis 给出的预滤波实现还会对有效样本按 $n\cdot \omega_{i,k}$ 加权，并除以权重之和。

> [!WARNING] 掠射角误差
> 这里用 $n=\omega_o=R$ 的分布代替真实 $n$、$\omega_o$ 对应的分布。掠射角附近真实的反射波瓣会拉长、偏斜，而预滤波 cubemap 无法保留这种形状，因此误差更明显。

### BRDF Integration LUT

Split Sum 的 BRDF 项 $\bar W$ **只包含 BRDF 权重，与环境贴图无关**。把环境辐射设为常数 1，就得到它所对应的连续积分：

$$
I_{\mathrm{BRDF}}(n,\omega_o,\alpha,F_0)
=\int_{\Omega^+}f_s(\omega_i,\omega_o)(n\cdot \omega_i)\,d\omega_i
\tag{32}\label{eq:environment-brdf}
$$

为使这个积分能够存入二维 LUT，采用 Schlick Fresnel。记 $F_c=(1-\omega_o\cdot h)^5$，则

$$
F(\omega_o,h)=F_0+(1-F_0)F_c=F_0(1-F_c)+F_c
\tag{33}
$$

由于 $D$ 和 $G$ 都不依赖 $F_0$，代入积分后可以把 $F_0$ 提到外面，得到**关于 $F_0$ 的线性形式**：

$$
I_{\mathrm{BRDF}}=F_0\,A(n\cdot \omega_o,\alpha)+B(n\cdot \omega_o,\alpha)
\tag{34}\label{eq:brdf-linear}
$$

其中 $A$ 是 $F_0$ 的系数，$B$ 是常数项。预计算仍使用前述 GGX 样本 $h_k,\omega_{i,k}$，对积分作数值估计。为把每个样本的权重写得简洁，定义

$$
G_{\mathrm{vis},k}
=\frac{G(\omega_{i,k},\omega_o,h_k)(\omega_o\cdot h_k)}{(n\cdot \omega_o)(n\cdot h_k)}
\tag{35}
$$

$$
F_{c,k}=(1-\omega_o\cdot h_k)^5
\tag{36}
$$

因为 $W_k=F(\omega_o,h_k)G_{\mathrm{vis},k}$，两个通道的估计式分别为

$$
\begin{aligned}
A(n\cdot \omega_o,\alpha)
&\approx\frac{1}{M}\sum_{k=1}^{M}(1-F_{c,k})G_{\mathrm{vis},k},\\
B(n\cdot \omega_o,\alpha)
&\approx\frac{1}{M}\sum_{k=1}^{M}F_{c,k}G_{\mathrm{vis},k}.
\end{aligned}
\tag{37}\label{eq:brdf-lut}
$$

对于不满足 $\omega_o\cdot h_k>0$ 或 $n\cdot \omega_{i,k}>0$ 的样本，求和项按 0 计。由于所用 BRDF 各向同性，$A$、$B$ 对 $n$ 和 $\omega_o$ 的依赖只通过 $n\cdot \omega_o$ 表达，因此可按 $(n\cdot \omega_o,\alpha)$ 预计算成**双通道的 2D BRDF LUT**。

> [!TIP] 运行时组合
> 预滤波环境贴图已在预计算阶段生成。运行时以 $R$ 为采样方向、按粗糙度 $\alpha$ 选取相应的 mip 级别，从贴图中读取 $P'(R,\alpha)$；再以 $(n\cdot \omega_o,\alpha)$ 从预计算的 BRDF LUT 中读取 $A$、$B$。将两部分组合：
>
> $$
> L_{o,\mathrm{env,specular}}(\omega_o)\approx P'(R,\alpha)\bigl[F_0A(n\cdot \omega_o,\alpha)+B(n\cdot \omega_o,\alpha)\bigr]
> \tag{38}\label{eq:specular-runtime}
> $$

## 总结

Diffuse IBL 将只依赖法线方向的环境照度存入 irradiance map。Specular IBL 先用 GGX Half-Vector 重要性采样估计积分，再用 Split Sum 将环境辐射与 BRDF 权重分开：环境项经过 $n=\omega_o=R$ 近似存入 Prefiltered Environment Map；BRDF 项借助 Schlick Fresnel 对 $F_0$ 的线性形式存入 BRDF Integration LUT。

运行时采样 irradiance map 和预滤波环境贴图，再读取 BRDF LUT，即可组合出环境漫反射与环境镜面反射，**无需逐像素重新计算完整的环境光积分**。

## 参考文献

- Brian Karis，[*Real Shading in Unreal Engine 4*](https://cdn2.unrealengine.com/Resources/files/2013SiggraphPresentationsNotes-26915738.pdf)，SIGGRAPH 2013 课程 *Physically Based Shading in Theory and Practice* 讲义，Epic Games，2013。
- Ravi Ramamoorthi、Pat Hanrahan，[*An Efficient Representation for Irradiance Environment Maps*](https://graphics.stanford.edu/papers/envmap/envmap.pdf)，SIGGRAPH 2001。

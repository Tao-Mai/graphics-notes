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

$$f_{diffuse}(p,l,v)=\frac{c(p)}{\pi}$$

$c(p)$是p点材质的diffuse albedo（漫反射率），则

$$
L_{o,env,diffuse}=\int_\Omega E(L)\frac{c(p)}{\pi}(N\cdot L)dL=\frac{c(p)}{\pi}\int_\Omega E(L)(N\cdot L)dL
$$

对于给定的 environment map，上式不再依赖材质参数和观察方向，只依赖表面法线 \(N\)。因此可以预先计算 \(E_{\mathrm{irr}}(N)\)，并用 cubemap 存储，这就是 irradiance map

## Specular BRDF
对于Cook-Torrance微表面模型

$$
f_s(L,V)=\frac{D(H)F(V,H)G(L,V,H)}{4(N\cdot V)(N\cdot L)}
$$


$$
L_{o,env,specular}(V)=\int_\Omega E(L)\frac{D(H)F(V,H)G(L,V,H)}{4(N\cdot V)}dL
$$

## 蒙特卡洛积分
对于给定的 \(V,N,F_0,\alpha\) 和环境函数 \(E\)，将 specular IBL 的 integrand 记作

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
下面给出各向同性 GGX 分布的具体采样方法

$$D(H)=\frac{\alpha^2}{\pi[(N\cdot H)^2(\alpha^2-1)+1]^2}$$

所以

$$
p_H(H)=\frac{\alpha^2(N\cdot H)}{\pi[(N\cdot H)^2(\alpha^2-1)+1]^2}
$$

从$\xi\sim U(0,1)$生成两个随机数$\xi_1,\xi_2$，**方位角均匀分布**，因此$\phi=2\pi\xi_1$，而$$F_\theta(\theta)=\int_0^\theta\frac{2\alpha^2\cos t\sin t}{[\cos^2t(\alpha^2-1)+1]^2}dt=\xi_2$$
$x=\sin^2 t, dx=2\sin t\cos tdt$，

$$\begin{aligned}F_\theta(\theta)&=\int_0^{\sin^2\theta} \frac{\alpha^2}{[(1-x)(\alpha^2-1)+1]^2}dx\\&=\frac{x}{\alpha^2+(1-\alpha^2)x}|_{0}^{\sin^2\theta}\\&=\frac{\sin^2\theta}{\alpha^2+(1-\alpha^2)\sin^2\theta}\end{aligned}$$

解$F_\theta(\theta)=\xi_2$，得到$$\sin^2\theta=\alpha^2\xi_2+(1-\alpha^2)\xi_2\sin^2\theta,\sin^2\theta=\frac{\alpha^2\xi_2}{1+(\alpha^2-1)\xi_2}$$随后结合$\phi$就可以得到局部坐标的$(x,y,z)$，再根据N转到世界坐标系，再得到世界坐标系下的L
TODO：拆成“求 θ → 构造局部 H → 转世界空间 → 根据 V、H 得到 L”四步

## BRDF项
先考察 BRDF 项的连续积分形式，

$$
F(v,h)=F_0+(1-F_0)(1-v\cdot h)^5
$$

$$
\int_\Omega f(l,v)\cos\theta_ldl=F_0\int_\Omega\frac{f(l,v)}{F(v,h)}(1-(1-v\cdot h)^5)\cos\theta_ldl+\int_\Omega\frac{f(l,v)}{F(v,h)}(1-v\cdot h)^5\cos\theta_ldl
$$

由于各向同性和旋转对称性，对 $N,V$ 的依赖可以压缩成 $N\cdot V$，则只依赖于roughness和$\cos\theta_v$，并且都在0到1之内，于是可以用一张2d的texture记录$F_0$的scale和bias
**这里的积分只是理论分析，实际计算依然是由上面的采样离散项来算，离线采样多次**

$$F_0(\frac{1}{M}\sum_{k=1}^M\frac{[1-(1-V\cdot H_k)^5]G(V\cdot H_k)}{(N\cdot V)(N\cdot H_k)})+(\frac{1}{M}\sum_{k=1}^M\frac{(1-V\cdot H_k)^5G(V\cdot H_k)}{(N\cdot V)(N\cdot H_k)})=F_0A+B$$



## 总结
至此，diffuse 和 specular 两部分的环境光积分都被转化为了适合实时计算的预计算表示。

Diffuse IBL 将环境光在半球上的 cosine-weighted 积分存入 irradiance map。Specular IBL 则通过 GGX importance sampling 构造 Monte Carlo estimator，再利用 Split Sum 将环境项与 BRDF 项近似分离：前者进一步采用 N=V=R 的近似，预计算为不同 roughness 下的 prefiltered environment map；后者利用 Fresnel 对 F0 的线性形式，将结果预计算为一张由 roughness 和 N·V 索引的 2D BRDF LUT。

因此，运行时不需要重新计算完整的环境光积分，只需要查询这些预计算结果并组合得到最终的 IBL。整个过程的核心，是通过选择合适的采样分布和降维近似，将原本依赖多个变量的积分转化为可以实时查询的低维表示。




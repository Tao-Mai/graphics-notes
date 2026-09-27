# IBL
**计算环境光照**
## 渲染方程
在表面某一点上

$$
L_{o}(V)=L_{emissive}(V)+\int_\Omega L_i(L)f(L,V)(N\cdot L) dL
$$
## 环境光
$L_i$可以分为**环境光和普通光源，现在只考虑环境光，用$E(L)$表示**

$$
L_{o,env}(V)=\int_\Omega E(L)f(L,V)(N\cdot L) dL
$$

**将环境光建模为无限远光源，则$E(L)$与所在点无关只与方向$L$有关，记录在environment map上**
## Diffuse BRDF
将BRDF分为diffuse和specular，其中diffuse BRDF为

$$
f_{diffuse}(p,l,v)=\frac{c(p)}{\pi}
$$

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

但这个 PDF 同时受到环境光和完整 BRDF 的影响，既难以构造，也难以直接采样。因此实际选择一个容易采样、并能够捕获 specular lobe 主要形状的 proposal PDF $q(L)$

对 Cook–Torrance BRDF，变化最尖锐的部分通常来自 GGX NDF，因此先由 $D(H)$ 构造 half vector $H$ 的采样分布
该形状受多个参数控制，且不好算分布函数。所以可以选择一个容易采样，且大致贴近g形状的proposal PDF，只是估计。

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

**这个归一化形式不好处理**：NDF 已知的归一化关系并不是 $\int_{\Omega^+}D(H)\,d\omega_H=1$，所以分母不能直接消掉。沿着这条路，还得额外计算 $\int D\,d\omega$，并为归一化后的 $p_H^{(D)}$ 构造采样方法。

为了避开这项额外归一化，改为**按微表面沿宏观法线 $N$ 的投影面积采样**。NDF 天然满足投影归一化关系：

$$
\int_{\Omega^+}D(H)(N\cdot H)\,d\omega_H=1
$$

因此，$D(H)$ 乘上面积投影因子 $N\cdot H$，就得到关于方向立体角 $d\omega_H$ 归一化的 PDF：

$$
p_H(H|N,\alpha)=D(H;N,\alpha)(N\cdot H)
$$

这个分布对应的随机实验是：在单位宏观表面的投影区域内均匀选一个点，找到投影到该点的微表面面元，将该面元的法线作为随机变量 $H$。某个方向的面元贡献的投影面积是其真实面积乘以 $N\cdot H$，因此 $H$ 落在方向微元 $d\omega_H$ 内的概率为 $D(H)(N\cdot H)\,d\omega_H$。这里的 $H$ 是**按投影面积选中面元后得到的法线方向**。

$$
q_L(L|V,N,\alpha)=\frac{D(H;N,\alpha)(N\cdot H)}{4(V\cdot H)}
$$

将其代入上面的蒙特卡洛积分，

$$
\begin{aligned}L_o(V)&\approx\frac{1}{M}\sum_{k=1}^M\frac{E(L_k)DFG}{4(N\cdot V)\times q(L_k|V,N,\alpha)}\\&\approx\frac{1}{M}\sum_{k=1}^M[\frac{E(L_k)DFG}{4(N\cdot V)}\times\frac{4(V\cdot H_k)}{D(N\cdot H_k)}]\\&\approx\frac{1}{M}\sum_{k=1}^ME(L_k)\frac{FG(V\cdot H_k)}{(N\cdot V)(N\cdot H_k)}\end{aligned}
$$

## Split Sum
现在才开始使用Split Sum，将环境光项和BRDF项拆开

$$
L_{o}(V)\approx(\frac{1}{M}\sum_{k=1}^ME(L_k))\times(\frac{1}{M}\sum_{k=1}^M\frac{FG(V\cdot H_k)}{(N\cdot V)(N\cdot H_k)})
$$

Split Sum 的本质，是把同一个采样分布下“环境辐射 × BRDF 权重”的期望，近似成两个期望的乘积，从而把环境和材质响应分开预计算
## 环境光项
由于$L_k|V,N,\alpha$，则可以用

$$
P(V,N,\alpha)\approx \frac{1}{M}\sum_{k=1}^ME(L_k)
$$

预计算环境光项，但这是三元函数

近似N=V，并记为R，则

$$P'(R,\alpha)=P(R,R,\alpha)\approx \frac{1}{M}\sum_{k=1}^ME(L_k)$$

也就是说将$N，V$时反射光线$L_k$周围的分布近似成了N=V=R时周围的分布
尤其 grazing angle 时真实 lobe 会更拉长、偏斜，而预滤波 cubemap 里用的是 \(N=V\) 的近似形状。


**GGX importance sample**，选择$D(H)$作为相似的分布。对H采样

各向同性GGX，

$$
D(H)=\frac{\alpha^2}{\pi[(N\cdot H)^2(\alpha^2-1)+1]^2}
$$

所以

$$
p_H(H)=\frac{\alpha^2(N\cdot H)}{\pi[(N\cdot H)^2(\alpha^2-1)+1]^2}
$$

从$\xi\sim U(0,1)$生成两个随机数$\xi_1,\xi_2$，显然$\phi=2\pi\xi_2$，而$$F_\theta(\theta)=\int_0^\theta\frac{2\alpha^2\cos t\sin t}{[\cos^2t(\alpha^2-1)+1]^2}dt=\xi_1$$
$x=\sin^2 t, dx=2\sin t\cos tdt$，

$$\begin{aligned}F_\theta(\theta)&=\int_0^{\sin^2\theta} \frac{\alpha^2}{[(1-x)(\alpha^2-1)+1]^2}dx\\&=\frac{x}{\alpha^2+(1-\alpha^2)x}|_{0}^{\sin^2\theta}\\&=\frac{\sin^2\theta}{\alpha^2+(1-\alpha^2)\sin^2\theta}\end{aligned}$$

解$F_\theta(\theta)=\xi_1$，得到$$\sin^2\theta=\alpha^2\xi_1+(1-\alpha^2)\xi_1\sin^2\theta,\sin^2\theta=\frac{\alpha^2\xi_1}{1+(\alpha^2-1)\xi_1}$$随后结合$\phi$就可以得到局部坐标的$(x,y,z)$，再根据N转到世界坐标系，再得到世界坐标系下的L


## BRDF项
先从积分角度看，

$$
F(v,h)=F_0+(1-F_0)(1-v\cdot h)^5
$$

$$
\int_\Omega f(l,v)\cos\theta_ldl=F_0\int_\Omega\frac{f(l,v)}{F(v,h)}(1-(1-v\cdot h)^5)\cos\theta_ldl+\int_\Omega\frac{f(l,v)}{F(v,h)}(1-v\cdot h)^5\cos\theta_ldl
$$

由于各向同性和旋转对称性，对 $N,V$ 的依赖可以压缩成 $N\cdot V$，则只依赖于roughness和$\cos\theta_v$，并且都在0到1之内，于是可以用一张2d的texture记录$F_0$的scale和bias
**这里的积分只是理论分析，实际计算依然是由上面的采样离散项来算，离线采样多次**

$$F_0(\frac{1}{M}\sum_{k=1}^M\frac{[1-(1-V\cdot H_k)^5]G(V\cdot H_k)}{(N\cdot V)(N\cdot H_k)})+(\frac{1}{M}\sum_{k=1}^M\frac{(1-V\cdot H_k)^5G(V\cdot H_k)}{(N\cdot V)(N\cdot H_k)})=F_0A+B$$








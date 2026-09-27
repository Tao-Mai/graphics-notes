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

其中$\int_\Omega E(L)(N\cdot L)dL$只和$\Omega$范围有关，其取决于法线$N$，所以可以预计算为irradiance map

## Specular BRDF
对于Cooker-Torrance微表面模型

$$
f_s(L,V)=\frac{D(H)F(V,H)G(L,V,H)}{4(N\cdot V)(N\cdot L)}
$$


$$
L_{o,env,specular}(V)=\int_\Omega E(L)\frac{D(H)F(V,H)G(L,V,H)}{4(N\cdot V)}dL
$$

## 蒙特卡洛积分
设被积函数为$g(L;V,N,F_0,\alpha)[E]=E(L)f_s(L;V,N,F_0,\alpha)(N\cdot L)$，五个函数参数，会决定函数分布，一个积分变量$L$
理论上的最优pdf也应该是一个五参数，单变量的函数，$p(l|V,N,F_0,\alpha,E)$，条件概率密度函数，后面是条件，输出的是概率。其中

$$
p^*(l)\propto g(l)
$$


该形状受多个参数控制，且不好算分布函数。所以可以选择一个容易采样，且大致贴近g形状的proposal PDF

$$
q(L|V,N,\alpha)
$$

**大致形状贴近的意思是，选定$V,N,\alpha$后，q的形状已确定，而剩余的$F_0,E$任意变化，$p^*$和q的形状也相差不远，只是估计，不是严谨计算。**
用q作为pdf，进行蒙特卡洛积分，

$$
L_o(V)\approx\frac{1}{M}\sum_{k=1}^M\frac{E(L_k)D(h_k)F(V,h_k)G(L_k,V,h_k)}{4(N\cdot V)\times q(L_k|V,N,\alpha)}
$$

那么如何找到这样的q呢，

根据$D(H)$构造一个投影面积下的H分布，得到概率密度函数

$$
p_H(H|N,\alpha)=D(H;N,\alpha)(N\cdot H)
$$

**相当于先从单位投影面积上采样一个点，再去判断该点所在微表面的法线，和真实微表面法线分布有一点不同**
不过本身也就是在近似g，用$D(H;\alpha)(N\cdot H)$也没啥关系，且只会影响方差。

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
L_{o}(V)\approx\frac{1}{M}\sum_{k=1}^ME(L_k)+\frac{1}{M}\sum_{k=1}^M\frac{FG(V\cdot H_k)}{(N\cdot V)(N\cdot H_k)}
$$

## 环境光项
由于$L_k|V,N,\alpha$，则可以用

$$
P(V,N,\alpha)\approx \frac{1}{M}\sum_{k=1}^ME(L_k)
$$

预计算环境光项，但这是三元函数

近似N=V，并记为R，则$$P'(R,\alpha)=P(R,R,\alpha)\approx \frac{1}{M}\sum_{k=1}^ME(L_k)$$



也就是说将$N，V$时反射光线$L_k$周围的分布近似成了N=V=L时周围的分布
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

**积分结果和n无关，只依赖于roughness和$\cos\theta_v$，并且都在0到1之内**，于是可以用一张2d的texture记录$F_0$的scale和bias
**这里的积分只是理论分析，实际计算依然是由上面的采样离散项来算，离线采样多次**
$$F_0(\frac{1}{M}\sum_{k=1}^M\frac{[1-(1-V\cdot H_k)^5]G(V\cdot H_k)}{(N\cdot V)(N\cdot H_k)})+(\frac{1}{M}\sum_{k=1}^M\frac{(1-V\cdot H_k)^5G(V\cdot H_k)}{(N\cdot V)(N\cdot H_k)})=F_0A+B$$











# IBL
**计算环境光照**
## 渲染方程
在点p处

$$
L_{o}(p,v)=L_{emissive}(p,v)+\int_\Omega L(p,l)f(p,l,v)\cos\theta_l dl
$$


## 环境光
$L(p,l)$可以分为**环境光和普通光源，现在只考虑环境光**

$$
L_{o,env}(p,v)=\int_\Omega L'_{env}(p,l)f(p,l,v)\cos\theta_l dl
$$

将**环境光建模为无限远光源，则$L'_{env}(p,l)$和p点无关只和方向$l$有关，用$L_{env}(l)$表示，记录在environment map上**
## Diffuse BRDF
将BRDF分为diffuse和specular，其中diffuse BRDF为

$$
f_{diffuse}(p,l,v)=\frac{c(p)}{\pi}
$$

$c(p)$是p点材质的diffuse albedo（漫反射率），则

$$
L_{o,env,diffuse}(p)=\int_\Omega L_{env}(l)\frac{c(p)}{\pi}\cos\theta_ldl=\frac{c(p)}{\pi}\int_\Omega L_{env}(l)\cos\theta_ldl
$$

其中$\int_\Omega L_{env}(l)\cos\theta_ldl$只和$\Omega$范围有关，其取决于法线，所以可以预计算为irradiance map

## Specular BRDF
先固定p点来分析，


$$
f_s(l,v)=\frac{D(h)F(v,h)G(l,v,h)}{4\cos\theta_l\cos\theta_v}
$$




$$
L_{o,env,specular}(v)=\int_\Omega L_{env}(l)\frac{D(h)F(v,h)G(l,v,h)}{4\cos\theta_l\cos\theta_v}\cos\theta_ldl=\int_\Omega L_{env}(l)\frac{D(h)F(v,h)G(l,v,h)}{4\cos\theta_v}dl
$$


该函数受n，v，$\alpha$，$F_0$决定
## 蒙特卡洛积分
**重要性采样**

$$
L_{o,env,specular}(v)\approx\frac{1}{N}\sum_{k=1}^NL_{env}(l_k)\frac{D(h_k)F(v,h_k)G(l_k,v,h_k)}{4\cos\theta_v}
$$


设被积函数为$g(l;V,N,F_0,\alpha,E)$，五个函数参数，会决定函数分布，一个积分变量$l$
理论上的最优pdf也应该是一个五参数，单变量的函数，$p(l|V,N,F_0,\alpha,E)$，条件概率密度函数，后面是条件，输出的是概率。其中

$$
p^*(l)\propto g(l)
$$


该形状受多个参数控制，且不好算分布函数。所以可以采样一个容易采样，且大致贴近g形状的proposal PDF

$$
q(l|V,N,\alpha)
$$

**大致形状贴近的意思是，选定$V,N,\alpha$后，q的形状已确定，而剩余的$F_0,E$任意变化，$p^*$和q的形状也相差不远，只是估计，不是严谨计算。**
用q作为pdf，进行蒙特卡洛积分，

$$
L_o(V)=\frac{1}{M}\sum_{k=1}^M\frac{E(L_k)D(h_k)F(V,h_k)G(L_k,V,h_k)}{4\cos\theta_V\times q(L_k|V,N,\alpha)}
$$

那么如何找到这样的q呢，

先根据$D(H)$来考虑一下H的分布，真实的H概率密度为

$$
p_H(H)=\frac{D(H)}{\int_\Omega D(H)d\omega}
$$

因为$D(H)$是朝向H的微表面相对于投影面积的面积密度，所以要统计一下总微表面面积归一化。

如果按投影面积来对H进行采样的话，

$$
p_H(H|N,\alpha)=D(H;\alpha)(N\cdot H)
$$

**相当于先从单位投影面积上采样一个点，再去判断该点所在微表面的法线，和真实微表面法线分布有一点误差**

比如微表面形成一个半球时，$D(H)=\frac{1}{\pi}$，理论上应该所有方向等概率，$p_H(H)=\frac{1}{2\pi}$，但如果用投影面积的话

$$
p_H(H)=\frac{N\cdot H}{\pi}
$$

**离法线近的概率偏高，离法线远的概率偏低**

不过用$p_H(H|N,\alpha)=D(H;\alpha)(N\cdot H)$也只是函数分布上有误差，顶多会使蒙特卡洛采样方差会变高，并且用D本来就是近似了



$$
q_L(L|V,N,\alpha)=\frac{D(H;N,\alpha)(N\cdot H)}{4(V\cdot H)}
$$


粗糙度比较低时，L的概率密度就在反射方向上附近比较高。所以确定$V,N,\alpha$后就可以对L采样了

将其代入上面的蒙特卡洛积分，

$$
L_o(V)=\frac{1}{M}\sum_{k=1}^M\frac{E(L_k)D(h_k)F(V,h_k)G(L_k,V,h_k)}{4\cos\theta_V\times q(L_k|V,N,\alpha)}=\frac{1}{M}\sum_{k=1}^MEFG\frac{V\cdot H_k}{(N\cdot V)(N\cdot H_k)}
$$


## split-sum


$$
L_{o}(V)\approx\frac{1}{M}\sum_{k=1}^ME(L_k)+\frac{1}{M}\sum_{k=1}^MF_kG_{vis,k}
$$



## 左侧项
近似N=V，并记为R，所以第一项可以写为

$$
P(R,\alpha)\approx \frac{1}{M}\sum_{k}E(L_k)
$$

也就是说将$N，V$时反射光线$L_k$周围的分布近似成了N=V=L时周围的分布
尤其 grazing angle 时真实 lobe 会更拉长、偏斜，而预滤波 cubemap 里用的是 \(N=V\) 的近似形状。


**GGX importance sample**，选择$D(h)$作为相似的分布。对H采样

各向同性GGX，

$$
D(H)=\frac{\alpha^2}{\pi[(N\cdot H)^2(\alpha^2-1)+1]^2}
$$

所以

$$
p_H(H)=\frac{\alpha^2(N\cdot H)}{\pi[(N\cdot H)^2(\alpha^2-1)+1]^2}
$$





其中$h_k$也是$l_k$决定的，所以相当于是先确定n，v，$\alpha$后，根据被积函数对l的分布来重要性采样，则

$$
L_k\sim p(l|n,v,\alpha)
$$





## 右侧项
先从积分角度看，

$$
F(v,h)=F_0+(1-F_0)(1-v\cdot h)^5
$$

$$
\int_\Omega f(l,v)\cos\theta_ldl=F_0\int_\Omega\frac{f(l,v)}{F(v,h)}(1-(1-v\cdot h)^5)\cos\theta_ldl+\int_\Omega\frac{f(l,v)}{F(v,h)}(1-v\cdot h)^5\cos\theta_ldl
$$

**积分结果和n无关，只依赖于roughness和$\cos\theta_v$，并且都在0到1之内**，于是可以用一张2d的texture记录$F_0$的scale和bias
**这里的积分只是理论分析，实际计算依然是由上面的采样离散项来算，离线超高采样**

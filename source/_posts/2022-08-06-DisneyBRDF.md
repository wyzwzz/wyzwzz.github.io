---
title: Disney Principled BRDF实现笔记
date: 2022-08-06 14:13:11
tags:
---

目前自己的离线光追渲染器已经实现了PT、SPPM、BDPT三种渲染器，然而材质系统只有可怜的Phong和Glass，场景表现力严重不足。虽然目前场景文件的导入还都内嵌在代码中，这确实十分的丑陋、不优雅，不过下一步准备用OpenGL+ImGui写个简单的界面，支持自定义场景，包括添加物体、修改材质、选择渲染方法、场景预览等常见的功能。Disney Principled BRDF由迪士尼在2012的sigraphic上提出，在之后2015年的sigraphics又将其扩展到了BSDF（本文不涉及），本文主要记录实现Disney BRDF(12)过程中涉及到的公式和代码，以及一些自己的理解，图形学入门中，如有错误，欢迎指正。

<!-- more -->

## Disney BRDF参数概览
![disney brdf params](disney_brdf.png)
如上图所示，Disney BRDF一共有11项参数（+1基础颜色）用于调整材质外观，分别为：
    1.baseColor，材质的基础颜色，即反射率(albedo)，可以是常数或者由纹理提供。
    2.subsurface，次表面散射系数，用于控制材质的漫反射项向次表面散射靠拢的程度，默认值为0。
    3.metallic，金属度，用于控制材质的外观向金属靠拢的程度，默认值为0。
    3.specular，镜面度，用于控制非金属微表面镜面反射项的大小，由菲涅尔项进行插值，默认值为0.5。
    4.specularTint，用于控制镜面反射光的颜色向基本颜色靠拢的程度，越小镜面光表现为白色，默认值为0。
    5.roughness，用于控制材质表面的粗糙程度，默认值为0.5。
    6.anisotropic，各向异性，用于控制材质镜面反射的非对称程度，默认值为0。
    7.sheen，模拟纺织物边缘的明亮效果，即绒毛效果，默认值为0，
    8.sheenTint，用于控制sheen分量颜色向基本颜色靠拢的程度，默认值为0.5。
    9.clearcoat，模拟清漆的效果，类似于镀了一层膜的效果，默认值为0，
    10.clearcoatGloss，用于控制清漆的光滑程度，默认值为1
上述所有参数的范围都在[0,1]之间，在该范围内对上述参数的任意取值和组合都是合法的，可以得到特定的材质效果，并且可以实现不同材质之间的线性过渡，而且对于美术十分友好。以上的参数其实可以分为两大类，漫反射和镜面项，下面分别具体这两部分的公式和推导。

## 漫反射
常说的漫反射项是光线进入到物体内部经过多次散射再从物体表面射出的部分，在实时渲染领域一般将其视为低频信息，因此会采用一些很粗糙的模型，比如lambert模型，直接将漫反射部分视作常数，这其实是很迷惑人的，以前使用Bling Phong模型，虽说它是个经验模型，能work就行，但很容易误导人。而次表面散射则是相对尺度更大的漫反射，如果光线进入物体表面后从离入射点很近的地方出射，或者说在一个像素范围内，那么被视为漫反射，而如果这个距离更大，则需要引入次表面散射的概念，即光线从表面入射后，在物体内部经过多次散射，最后从物体表面射出，入射点和出射点的距离不能忽略。不管是漫反射还是次表面散射，都是考虑折射到物体内部那部分光的能量，因此很多材质模型会使用菲尼尔项计算折射光的能量比例作为漫反射分量的乘积因子，比如著名的金属工作流。
Disney BRDF的漫反射项不仅与菲涅尔项挂钩，还与物体表面的粗糙程度挂钩，这是考虑了掠射角时会出现逆射的现象，弥补了lambert模型边缘很暗的缺陷，因为物体越粗糙，在掠射角处，有更多的微平面是垂直于视线的，如下图。
![掠射角时的逆射](grazing_angle_retro.png)
考虑了菲涅尔项和逆射后，Disney BRDF的diffuse项公式如下：
$$
f_{diffuse} = \frac{baseColor}{\pi}(1+(F_{D90} - 1)(1 - \cos\theta_i)^5)
(1 + (F_{D90}-1)(1-\cos\theta_o)^5)
$$
$$
F_{D90} = 0.5 + 2 \cos^2\theta_d \cdot roughness
$$
其中$\theta_i$和$\theta_o$是入射方向$\omega_i$和出射方向$\omega_o$与宏观表面法向量$\omega_n$的夹角，$\theta_d$是入射方向$\omega_i$或出射方向$\omega_o$与half vector $\omega_h = (\omega_i + \omega_o)/2$的夹角。
对于次表面散射的计算，Disney BRDF采用从[Hanrahan-Krueger BRDF](https://cseweb.ucsd.edu/~ravir/6998/papers/p165-hanrahan.pdf)改进而来的公式：
$$
f_{subsurface} = 1.25\cdot \frac{baseColor}{\pi}\cdot (F_{ss}\cdot(
    \frac{1}{\cos\theta_i+\cos\theta_o}-0.5
) + 0.5)
$$
$$
F_{ss} = (1 + (F_{ss90} - 1)(1 - \cos\theta_i)^5)(1 + (F_{ss90} - 1)(1 - \cos\theta_o)^5)
$$
$$
F_{ss90} = \cos^2\theta_d \cdot roughness
$$
因为次表面散射和漫反射是同一个原理在不同尺度上的表现，所以Disney BRDF用subsurface参数来调整两者之间的过渡。另外这里的次表面散射用brdf来模拟，总体效果是使得漫反射更加平，十分trick，因此对于这一类材质的表现力十分有限，就当看看罢了。

## 镜面反射
### 微表面模型
Disney的镜面反射部分包括了表面镜面反射和清漆，均使用微表面模型。在微平面理论中，光线在物体表面的反射是完美的镜面反射，但因为宏观表面是由不同朝向的微观表面组成的，因此在宏观上会表现出粗糙。对于PBR材质，镜面反射一般使用Torrance-Sparrow微表面模型，其公式为：
$$
    f = F_r(\theta_d)\frac{D(\theta_h,\phi_h)G(\omega_i,\omega_o)}{4\cos\theta_i\cos\theta_o}
$$
这个公式懂得都懂，就不再详细介绍了。值得注意的是在迪士尼给出的源码中，把分母项$4\cos\theta_i\cos\theta_o$移到了$G(\omega_i,\omega_o)$中，但其实两者并没有联系，而且分母项主要是$D(\theta_h,\phi_h)$引入的。自己实现的时候，就不结合进去了，很容易搞混淆。
### 微表面法线分布
Disney BRDF中镜面项的法线分布函数使用了Generalized-Trowbridge-Reitz(简称GTR)函数，其公式为：
$$
D_{GTR}(\theta_h,\phi_h) = \frac{c}{
    (\sin^2\theta_h(
        \frac{\cos^2\phi}{\alpha^2_x} + \frac{\sin^2\phi}{\alpha^2_y}
    ) + \cos^2\theta_h)^{\gamma}
}
$$
其中c是归一化常数；$\theta_h$和$\phi_h$是半程向量分别与平面法向量和法线坐标系下x轴的夹角；$\alpha_x$和$\alpha_y$是衡量表面不同方向上的粗糙程度比例，当相等时即为常见的各向同性材质，否则为各向异性材质（实时渲染中用的比较少）；$\gamma$是用于调整曲线形状的参数，不同的值对应的曲线见下图。当$\gamma$取值为2时，即为常见的GGX函数（一般也会取各向同性）。在Disney BRDF中有两处用到，一处是镜面反射，使用了$\gamma = 2$下的各向异性公式，一处是清漆，使用了$\gamma = 1$下的各向同性公式。
![GTR](gtr.png)
其中的$\alpha_x$和$\alpha_y$和Disney BRDF的参数之间的对应关系为：
$$
    aspect = \sqrt{1-0.9 \cdot anisotropic}
$$
$$
\alpha_x = roughness^2 / aspect
$$
$$
\alpha_y = roughness^2 \cdot aspect
$$
Disney BRDF只会用到$\gamma=2$下各向异性的GTR，显然易见，各向异性的公式复杂得多，而且大多数情况下，使用各向同性已经足够了。要使用GTR的话，首先要求出常数c，一般是通过积分为1反算出来，而在各向同性的情况下，c可以求出在$\gamma$表示下的通用表达式，十分方便，包括采样公式(在离线渲染中，需要对brdf进行采样，一般对NDF进行重要性采样)。首先是各向同性条件下GTR的表达式为：
$$
D_{GTR}(\theta_h,\phi_h) = \frac{c}{
    (\sin^2\theta_h/\alpha^2 + \cos^2\theta_h)^{\gamma}
}
$$
对于各向同性的微表面分布，通过积分求解得到其通用表达式以及对应的采用公式：
$$
    D_{GTR}(\theta_h,\phi_h) = 
    \frac{(\gamma - 1)(\alpha^2 - 1)}{\pi(1-(\alpha^2)^{1-\gamma})}
    \frac{1}{(1 + (\alpha^2 - 1)\cos^2\theta_h)^\gamma}
$$
$$
    \phi_h = 2\pi\xi_1
$$
$$
    \cos\theta_h=\sqrt{
        \frac{1-((\alpha^2)^{1-\gamma}(1-\xi_2)+\xi_2)^{\frac{1}{1-\gamma}}}{1-\alpha^2}
    }
$$
当$\gamma$取1时，会存在奇异值，此时需要计算出$\gamma \rightarrow 1$下的极限表达式（简单用一下洛必达定理）：
$$
D_{GTR_1}(\theta_h,\phi_h) = 
\frac{\alpha^2 - 1}{\pi \ln\alpha^2}
\frac{1}{(1 + (\alpha^2 - 1)\cos^2\theta_h)}
$$
$$
\cos\theta_h=\sqrt{\frac{1-(\alpha^2)^{1-\xi_2}}{1-\alpha^2}}
$$
当$\gamma=2$时，即为熟悉的GGX，公式为：
$$
    D_{GTR_2}(\theta_h,\phi_h)=\frac{\alpha^2}{\pi}
    \frac{1}{(1+(\alpha^2-1)\cos^2\theta_h)^2}
$$
$$
    \cos\theta_h=\sqrt{\frac{1-\xi_2}{1+(\alpha^2-1)\xi_2}}
$$
Disney的文档中提到，当$\gamma = 3/2$时，等效于Henyey-Greenstein相位函数，应该不是等于吧...我推了一下得不到相等的条件，可能是我太菜了orz...

对于各向异性的$GTR_2$，其表达式和采样公式为：
$$
    D_{GTR_2aniso} = \frac{1}{\pi}\frac{1}{\alpha_x\alpha_y}\frac{1}
    {(\sin^2\theta_h(\frac{\cos^2\phi_h}{\alpha^2_x} + \frac{\sin^2\phi_h}{\alpha^2_y}) + \cos^2\theta_h)^2}
$$
$$
\tan\phi_h = \frac{\alpha_y}{\alpha_x}\tan(2\pi\xi_1)
$$
$$
    r = \sqrt{(\frac{\cos^2\phi_h}{\alpha^2_x} + \frac{\sin^2\phi_h}{\alpha^2_y})^{-1}}
$$
$$
\cos\theta_h=\sqrt{\frac{1-\xi_2}{
    1 + ((\frac{\cos^2\phi_h}{\alpha^2_x} + \frac{\sin^2\phi_h}{\alpha^2_y})^{-1} - 1)\xi_2
}} = \sqrt{\frac{1-\xi_2}{1 + (r^2-1)\xi_2}}
$$

$$
    \sin\phi_h=\frac{\alpha_y\sin(2\pi\xi_1)}{r}
$$
$$
    \cos\phi_h=\frac{\alpha_x\cos(2\pi\xi_1)}{r}
$$
$$
    \tan\theta_h = r\sqrt{\frac{\xi_2}{1-\xi_2}}
$$

### 几何遮挡
Disney BRDF中镜面项的几何遮蔽函数G使用了Smith遮蔽函数：
$$
G(\omega_i,\omega_o)=G_1(\omega_i)G_1(\omega_o)
$$
$$
G_1(\omega) = \frac{1}{1 + \Lambda(\omega)}
$$
$$
\Lambda(\omega)=-0.5+0.5\sqrt{1 + (\alpha^2_x\cos^2\phi+\alpha^2_y\sin^2\phi)tan^2\theta}
$$

## Sheen
对于一些有绒毛的材质，从掠射角方向观察时，漫反射会更加明亮。在Disney BRDF中，sheen项正比于$(1-\cos\theta_d)^5$。

## 清漆clearcoat
现实中很多材质往往是多层单一材质的混合体，比如涂了油漆的木板，上面一层是薄薄的漆，通常比较光滑，而下面则是木板表面，主要表现为漫反射，如果想要渲染后这种材质，就必须要考虑层与层之间的折射和反射，这个过程可以模拟地很复杂，但即便是在离线渲染中，也很难接受。在Disney BRDF中则直接简单地额外添加一项clearcoat，与镜面项同样地采用微表面模型——Torrance-Sparrow模型，不过限制了G项使用固定粗糙度0.25，F项采用固定折射率1.5的电介质，D项则采用$\gamma=1$的各向同性GTR函数，粗糙度则通过参数中的clearcoatGloss重映射得到：
$$
    roughness_{clearcoat} = mix(0.1,0.001,clearcoatGloss)
$$

## 公式汇总
上述那么多项其实可以分为两部分相加，一部分是漫反射相关，一部分是镜面反射相关。漫反射相关包括了diffuse、subsurface和sheen，镜面反射相关包括了specular、clearcoat。因为只有金属才有漫反射现象，因此漫反射相关需要乘以系数(1-metallic)。为了更简洁地表达最终公式，首先设：
$$C       = baseColor$$
$$k_m     = metallic$$
$$k_{ss}  = subsurface$$
$$k_{s}   = specular$$
$$k_{st}  = specularTint$$
$$k_{r}   = roughness$$
$$k_{a}   = anisotropic$$
$$k_{sh}  = sheen$$
$$k_{sht} = sheenTint$$
$$k_{c}   = clearcoat$$
$$k_{cg}  = clearcoatGloss$$
并设入射方向为$\omega_i=(\phi_i,\theta_i)$，出射方向为$\omega_o=(\phi_o,\theta_o)$，半程向量half vector为$\omega_h=(\phi_h,\theta_h)$，$\omega_i$与$\omega_h$的入射夹角为$\theta_d$，那么Disney BRDF的公式为：
$$
f_{disney}(\omega_i,\omega_o) = (1-k_{m})(\frac{C}{\pi}mix(f_d(\omega_i,\omega_o),f_{ss}(\omega_i,\omega_o),k_{ss}) + f_{sh}(\omega_i,\omega_o))\\
+\frac
{F_{s}(\theta_d)G_{s}(\omega_i,\omega_o)D_s(\omega_h)}
{4\cos\theta_i\cos\theta_o}\\
+\frac{k_c}{4}
\frac
{F_{c}(\theta_d)G_{c}(\omega_i,\omega_o)D_c(\omega_h)}
{4\cos\theta_i\cos\theta_o}
$$

其中$f_d$为漫反射项：
$$
f_{diffuse} = \frac{C}{\pi}(1+(F_{D90} - 1)(1 - \cos\theta_i)^5)
(1 + (F_{D90}-1)(1-\cos\theta_o)^5)
$$
$$
F_{D90} = 0.5 + 2 \cos^2\theta_d \cdot k_r
$$
$f_{ss}$为次表面散射项：
$$
f_{subsurface} = 1.25\cdot \frac{C}{\pi}\cdot (F_{ss}\cdot(
    \frac{1}{\cos\theta_i+\cos\theta_o}-0.5
) + 0.5)
$$
$$
F_{ss} = (1 + (F_{ss90} - 1)(1 - \cos\theta_i)^5)(1 + (F_{ss90} - 1)(1 - \cos\theta_o)^5)
$$
$$
F_{ss90} = \cos^2\theta_d \cdot k_r
$$
$f_{sh}$为sheen项：
$$
f_{sh}(\omega_i,\omega_o) = mix(one,C_{tint},k_{sht}) \cdot k_{sh} \cdot (1-\cos\theta_d)^5
$$
$$
C_{tint} = \frac{C}{lum(C)}
$$
$$
lum(C) \approx 0.2126 * C.r + 0.7152 * C.g + 0.0722 * C.b 
$$
$F_s(\theta_d)$为镜面反射的frensel项，使用Schlick公式做了近似。类似于金属工作流中的处理，对于电介质而言，基础反射率约等于0.04，而对于导体而言，不同的导体有不同的反射率，使用金属度metallic对其进行插值：
$$
F_s(\theta_d) = C_s+(1 - C_s)(1-\cos\theta_d)^5
$$
$$
C_s = mix(0.08 * k_s * mix(one,C_{tint},k_{st}),C,k_m)
$$
$G_s$是镜面反射的遮蔽项，采用各向异性GGX对应的smith函数：
$$
G_s(\omega_i,\omega_o)=G_{s1}(\omega_i)G_{s1}(\omega_o)
$$
$$
G_{s1}(\omega) = \frac{1}{1 + \Lambda(\omega)}
$$
$$
\Lambda_s(\omega)=-0.5+0.5\sqrt{1 + (\alpha^2_x\cos^2\phi+\alpha^2_y\sin^2\phi)tan^2\theta}
$$
$D_s$是镜面反射的法线分布函数，采用各向异性的GGX函数：
$$
    D_{s} = \frac{1}{\pi}\frac{1}{\alpha_x\alpha_y}\frac{1}
    {(\sin^2\theta_h(\frac{\cos^2\phi_h}{\alpha^2_x} + \frac{\sin^2\phi_h}{\alpha^2_y}) + \cos^2\theta_h)^2}
$$
$$
\alpha_x = k^2_r / a
$$
$$
\alpha_y = k^2_r\cdot a
$$
$$
a = \sqrt{1-0.9k_a}
$$
$F_c$是清漆的fresnel项，采用折射率为1.5对应的基础反射率，对应的Schlick公式为：
$$
F_c(\theta_d) = 0.04 + 0.96(1-\cos\theta_d)^5
$$
$G_c$是清漆的几何遮蔽项，采用粗糙度为0.25的各向同性GGX对应的smith函数：
$$
G_c(\omega_i,\omega_o)=G_{c1}(\omega_i)G_{c1}(\omega_o)
$$
$$
G_{c1}(\omega) = \frac{1}{1 + \Lambda(\omega)}
$$
$$
\Lambda_c(\omega)=-0.5+0.5\sqrt{1 + \alpha^2tan^2\theta}
$$
$D_c$是清漆的法线分布函数，采用各向同性的GTR1函数,其中的粗糙度需要重映射：
$$
D_{c}(\theta_h,\phi_h) = 
\frac{\alpha^2 - 1}{\pi \ln\alpha^2}
\frac{1}{(1 + (\alpha^2 - 1)\cos^2\theta_h)}
$$
$$
\alpha = mix(0.1,0.01,k_{cg})
$$
综上，Disney BRDF公式的具体介绍完毕，可以在shader里实现，迪士尼开源了[代码](https://github.com/wdas/brdf/blob/main/src/brdfs/disney.brdf)。

## 实现效果
首先是固定金属度为1，不同粗糙度下的效果
![](dragon_metal.png)
可以看到随着粗糙度的增加，物体表面越来越暗，这是因为没有考虑粗糙表面下光线会多次折射，没有补上这部分的能量，不过Disney BRDF是在2012年提出的，后面实时渲染中可以采用KullaConty方法补上这部分能量，有兴趣的可以看看我实现的[Games202作业](https://github.com/wyzwzz/Games202-OpenGL)和[KullConty和IBL结合](https://github.com/wyzwzz/KullaConty-IBL)。

接下来是固定金属度为0，不同粗糙度和次表面散射下的效果
![](fullbody_none_metal.png)
之前说到了Disney BRDF中的次表面散射是是什么鸡肋的，从上图也可以看到，随着subsurface的增加，物体表面表现地更暗，我也不知道该怎么评价...其它的参数调整懒得搞了，到Disney BSDF实现后再多调试一下。

在实现时，需要对各部分进行重要性采样，因为sheen部分占据的能量很少，直接忽略就好，主要考虑漫反射、镜面反射、清漆。首先以$min(0.8,1-k_m)$的概率采样漫反射，使用cos加权的半球采样；如果未采样漫反射，那么以$1/(1+k_c/2)$的概率采样镜面项，使用各向异性的GGX采样；以最后的概率采样清漆，使用各向同性的GTR1采样。设漫反射、镜面和清漆采样对应的pdf分别为$p_d$、$p_s$、$p_c$，则最后的pdf根据全概率公式为：
$$
p = min(0.8,1-k_m)p_d + (1-min(0.8,1-k_m))(\frac{1}{1+k_c/2}p_s+\frac{k_c/2}{1+k_c/2}p_c)
$$

这之后迪士尼又推出了Disney Principled BSDF，支持了对透明和半透明的材质，并且将次表面散射从公式中分离，采用Normalized Diffusion BSSRDF，可以结合体介质模型。另外对GTR2的采样结合可视法向量获得更好的效果。这部分的实现待我再好好研究，加入了投射后公式的推导没有那么好理解了，代码中处理也需要更加注意...

## 参考
1.https://zhuanlan.zhihu.com/p/57771965

2.https://zhuanlan.zhihu.com/p/407007915

3.https://zhuanlan.zhihu.com/p/60977923

4.https://blog.selfshadow.com/publications/
s2012-shading-course/burley/s2012_pbs_disney_brdf_notes_v3.pdf

5.https://schuttejoe.github.io/post/disneybsdf/

6.http://shihchinw.github.io/2015/07/implementing-disney-principled-brdf-in-arnold.html

## 附录
上述的公式都直接给出了结果，不过推导一下，复习微积分还是很有必要的，以及巩固概率论。

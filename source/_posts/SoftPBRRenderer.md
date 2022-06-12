---
title: IBL的一些小结和实现
---

简单总结一些关于PBR和IBL的知识点，顺便贴一下自己的实现仓库
[软光栅实现IBL](https://github.com/wyzwzz/SoftIBLRenderer)

<!-- more -->

# 辐射度量学
一些定义

## Energy
一个光子携带的能量
$$Q = \frac{hc}{\lambda}$$
$h$是普朗克常量，$c$是真空中光速,$\lambda$是波长
## Flux
单位时间内光穿过某个区域的能量
$$\Phi = \lim^{}_{\Delta t \to 0}{\frac{\Delta Q}{\Delta t}} = \frac{dQ}{dt}$$

## 辐照度 Irradiance
单位时间通过单位面积的能量
$$E(p) =\lim^{}_{\Delta A \to 0}{\frac{\Delta \Phi(p)}{\Delta A}}
= \frac{d\Phi(p)}{dA^\bot}= \frac{d\Phi(p)}{dA \cos\theta}$$


## 辐射强度 Intensity
单位时间内通过单位立体角的能量
$$I = \lim^{}_{\Delta \omega \to 0}\frac{\Delta \Phi}{\Delta \omega}
=\frac{d\Phi}{d\omega}$$

## 辐射度 Radiance
单位时间内通过单位立体角单位面积的能量
也可以理解为从一点向某一个方向单位时间发出的能量
$$
L = \lim^{}_{\Delta \omega \to 0}\frac{\Delta E_{\omega}(p)}{\Delta \omega} 
= \frac{dE_{\omega}(p)}{d\omega}
= \frac{d\Phi}{d\omega dA^\bot}
= \frac{d\Phi}{d\omega dA \cos\theta}
$$

# 渲染方程
$$
L_{o}(p,\omega_{o}) = \int^{}_{\Omega}f_{r}(p,\omega_{i},\omega_{o})L_{i}(p,\omega_{i})
n \cdot \omega_{i}d\omega_{i}
$$

# 微表面模型
我们之前常说的漫反射和镜面反射现象其实是光线通过微表面模型反射后的一种统计学现象。
微表面模型认为，宏观上的一个平面，在微观上是由很多高低起伏的粗糙形状构成。
微表面上发生的反射都是完美的镜面发射，但是由于微平面的混乱，这些镜面反射会向着不同的方向发散开来，
产生分布范围更广泛的镜面反射，在宏观上表现为漫反射和glossy反射lobe，
而对于微平面也是十分光滑的平面来说，就会产生宏观上的镜面反射。
由于微平面小到无法逐像素区分，因此用一个粗糙度（Roughness）表示某个向量与微平面的平均法向量取向一致的概率，微平面的平均法向量就是常说的平面法向量，某个向量一般是入射光和发射光方向的中间向量。

## BRDF
BRDF定义为 单位面积上的 出射辐射度 与 入射辐照度 的比值：
$$
f_{r}(p,\omega_{i},\omega_{o}) = \frac{dL_{o}(p,\omega_{o})}{dE_{i}(p,\omega_{i})}
$$

目前一般实时使用的BRDF模型为Cook-Torrance BRDF模型，其分为漫反射和镜面反射两个部分
$$
f_{r} = k_{d}f_{lambert} + k_{s}f_{cook-torrance}
$$

光线在微表面发生碰撞时会产生折射和反射现象，其中折射是光线进入物体内部，两者的能量比满足菲涅尔方程，光线在物体内部折射时能量可能被吸收，同时路径也可能会改，也可能因为被介质散射（考虑体介质参与），最后又从物体表面射出，如果重新射出的表面点离原来的入射点很近，那么可以简单将其考虑为一种漫反射现象，一般是朝各个方向均匀射出，如果距离比较远，比如说超过一个像素范围，那么如果要达到好的渲染效果，需要进一步考虑BSSRDF反射模型。

上式中$kd$式入射光被折射的能量占比，而$ks$是反射部分的能量占比，根据能量守恒定律,$ks + kd = 1$
在Cook-Torrance BRDF模型中，$f_{d}$为常数，采用Lambertian漫反射模型：
$$
f_{lambert} = \frac{c}{\pi}
$$
$c$是表面的漫反射系数albedo，在0~1之间，这里考虑了物体对光线能量的吸收，分母的$\pi$是为了
考虑积分为1：
$$
\int^{2\pi}_{0}\int^{\frac{\pi}{2}}_{0}\cos\theta\sin\theta d\theta d\phi = \pi
$$

而镜面项的形式为：
$$
f_{cook-torrance} = \frac{DFG}{4(\omega_{o}\cdot n)(\omega_{i}\cdot n)}
$$

这个公式也可以通过推导得到：

假设 $\theta_{h}$ 是 $\omega_{i}$ 和 $\omega_{o}$ 与半程向量 $\omega_{h}$ 的夹角

假设 $\theta$ 是 $\omega_{h}$ 与 $n$ 的夹角

假设 $n$ 是平面的法向量， $a$ 是平面的粗糙度

假设 $\theta_{i}$ 是入射光与平面法向量的夹角，$\theta_{o}$ 是出射光与平面法向量的夹角

类似于$d\omega = \sin \theta d\theta d\phi$的关系，对于平面上的法向分布概率函数有以下关系：
$$
dA(\omega_{h}) = D(\omega_{h})d\omega_{h}dA
$$

D就是法线分布函数

这个点的能量通量为：
$$
d\Phi_{h} = dE_{h}dA(\omega_{h})
$$
出射的辐射度定义为：
$$
L(\omega_{o}) = \frac{d\Phi_{o}}{d\omega_{o}dA(\omega_{o})}
$$
根据菲涅尔定理有：
$$
d\Phi_{o} = F_{r}(\omega_{o})d\Phi_{h}
$$
代入替换一下 $d\Phi_{o}$ 到 $L(\omega_{o})$ 中：
$$
L(\omega_{o}) = \frac{F_{r}(\omega_{o})d\Phi_{h}}{d\omega_{o}dA(\omega_{o})}
$$
再替换当中的$\Phi_{h}$得到：
$$
L(\omega_{o}) = \frac{F_{r}(\omega_{o})dE_{h}dA(\omega_{h})}{d\omega_{o}dA(\omega_{o})}
$$
在这里需要注意的是$dA(\omega_{h})$与$dA(\omega_{o})$的计算公式是不一样的，
前者是需要通过微表面的法线分布计算得到，后者通过确定的出射方向得到，即：
$$
dA(\omega_{o}) = \cos \theta_{o} dA
$$
另外$dE_{h}$的定义为：
$$
dE_{h} = L_{i}d\omega_{i} \cos \theta_{h}
$$
然后再替换一下得到：
$$
L(\omega_{o}) = \frac{F_{r}L_{i}d\omega_{i} \cos \theta_{h}D(\omega_{h})d\omega_{h}dA}
{d\omega_{o}\cos \theta_{o} dA}
=\frac{F_{r}L_{i} D(\omega_{h})\cos \theta_{h}d\omega_{h}d\omega_{i}}
{\cos \theta_{o}d\omega_{o}}
$$

然后$w_{o}$ 和 $w_{h}$是有几何关系的，假设$\theta^{'}_{o}$是出射光与入射光的夹角，
那么以入射光为z轴下有如下关系 $\theta_{o} = 2 \cdot \theta_{h}$
（特殊的推导出一般情况,因为
 $d\omega_{h}$ 与 $ d\omega_{o}$ 之间的关系与坐标系的朝向无关）
 $$
\frac{d\omega_{h}}{d\omega_{o}} = \frac{\sin \theta_{h} d\theta_{h} d\phi_{h}}
{\sin \omega_{o}d\theta_{o} d\phi_{o}} = \frac
{\sin \theta_{h} d\theta_{h} d\phi_{h}}{\sin (2\theta_{h}) d (2\theta_{h}) d\phi_{o}}
=\frac{1}{4 \cos \theta_{h}}
$$
所以代入后，最后化简为：
$$
L(\omega_{o}) = \frac{F_r(\omega_{o})L_{i}(\omega_{i})D(\omega_{h})d\omega_{i}}
{4 \cos \theta_{o}}
=L_{i}(\omega_{i}) \cdot \frac{F_r(\omega_{o})D(\omega_{h})}
{4 \cos \theta_{o}} \cdot d\omega_{i}
$$

那么BRDF的定义为：
$$
L(\omega_{o}) = \frac{F_r(\omega_{o})G(\omega_{o},\omega_{i})D(\omega_{h})}
{4 \cos \theta_{o} \cos \theta_{i}}
$$
一般来说，$F_{r}$ 与 $\omega_{o},\omega_{h},F_{0}$有关，
$D$ 与 $\omega_{h},n,\alpha$ 有关，
$G$ 与 $n,\omega_{i},\omega_{o},k$ 有关 $k$ 是 $\alpha$ 针对直接光和IBL的重映射

而 $G$ 需要同时考虑 Geometry Shadow 和 Mask，前者是反射光被物体自遮挡，后者是物体挡住了人眼的实现，因此 $G$ 的定义为：
$$
G(n,\omega_{o},\omega_{i},\alpha) = G_{SchlickGGX}(n,\omega_{i},k(\alpha)) \cdot
G_{SchlickGGX}(n,\omega_{o},k(\alpha)) 
$$
其中：
$$
G_{SchlickGGX}(n,v,k) = \frac{n \cdot v}{(n \cdot v)(1 - k) + k}
$$
$$
k_{direct} = \frac{(\alpha + 1)^2}{8}
$$
$$
k_{IBL} = \frac{\alpha^2}{2}
$$

常用的法线分布函数D有Trowbrideg-ReitzGGX，定义为：
$$
NDF_{GGXTR}(n,h,\alpha) = \frac{\alpha^2}
{\pi((n \cdot h)^2(\alpha^2-1)+1)^2}
$$

关于电介质的折射系数n的测量，其实是根据可见光在电介质和真空中的相对速度求得的

$$
r_{\parallel} = \frac
{\eta_{t} \cos \theta_{i} - \eta_{i} \cos \theta_{t}}
{\eta_{t} \cos \theta_{i} + \eta_{i} \cos \theta_{t}}
$$

$$
r_{\bot} = \frac
{\eta_{i} \cos \theta_{i} - \eta_{t} \cos \theta_{t}}
{\eta_{i} \cos \theta_{i} + \eta_{t} \cos \theta_{t}}
$$

$$
F_{r} = \frac{1}{2}(r^2_{\parallel} + r^2_{\bot})
$$

导体的 折射指数 定义为复数，形式为 $\overline\eta = \eta + ik$ 公式就不放了

菲涅尔方程比较复杂，求解代价比较大，一般使用Fresnel-Schlick近似法求解：
$$
F_{Schlick}(h,v,F_{0}) = F_{0} + (1 - F_{0})(1-(h \cdot v))^5
$$

$F_{0}$ 使用折射指数计算的，但是导体的折射指数是复数，无法这么计算，所以把沿着法向入射的反射率作为 $F0$

## Cook-Torrance 渲染方程
$$
L_{o}(p,\omega_{o})=\int_{\Omega}
(k_{d}\frac{c}{\pi}+k_{s}\frac{DFG}{4(\omega_{o}\cdot n)(\omega_{i}\cdot n)})
L_{i}(p,\omega_{i})n \cdot \omega_{i}d\omega_{i}
$$

# IBL
## 漫反射项

$$
L_{d}(\omega_{o}) = \frac{c}{\pi}\int_{\Omega}k_{d}L_{i}(\omega_{i})n\cdot \omega_{i}d\omega_{i}
$$
因为 $k_{d}$ 并不是一个常数，需要对其进行近似简化：
$$
k_{d} = (1 - F(n\cdot \omega_{i}))(1 - metalness) \\
\approx (1-F_{roughness}(n\cdot \omega_{o}))(1-metalness)
=k^{*}_{d}
$$
其中的 $F_{roughness}(n\cdot \omega_{o})$的定义为：
$$
F_{roughness}(n\cdot \omega_{o}) = F_{0} + (max(1-roughness,F_{0}) - F_{0})(1-n\cdot \omega_{o})^5
$$

采用蒙特卡洛方法进行预计算，均匀采样每一个方向的$L_{i}$
$$
\int^{b}_{a}f(x)dx = \frac{b-a}{N} \sum^{N}_{i=1}f(X_{i})
$$

转换为：
$$
\int_{\Omega}\frac{1}{\pi}L_{i}(p,\omega_{i})d\omega_{i} \approx
\frac{\pi}{n1 \cdot n2}\sum^{n1}_{m=0} \sum^{n2}_{n=0}
L_{i}(p,\phi_{m},\theta_{n})\cos(\theta_{n})sin(\theta_{n})
$$
也可以用余弦为权重进行半球采样 $p(\omega_i) = \frac{\cos \theta}{\pi}$ ：
$$
\frac{1}{\pi}\int_{\Omega}L_{i}(\omega_{i})n\cdot \omega_{i}d\omega_{i} \approx
\frac{1}{\pi}\frac{1}{N}\sum^{N}_{i}\frac{L(\omega_{i})(n\cdot \omega_{i})}
{\frac{n\cdot \omega_{i}}{\pi}}
=\frac{1}{N}\sum^{N}_{i}L(\omega_{i})
$$

## 镜面项

 使用了split sum近似，将$L_{i}(\omega_{i})$与$f_{s}(\omega_{i},\omega_{o})n\cdot\omega_{i}$从积分项中分离，具体来说首先进行等价变换：
 $$
L_{s}(\omega_{o})=\int_{\Omega}f_{s}(\omega_{i},\omega_{o})L_{i}(\omega_{i})n\cdot \omega_{i}d\omega_{i}
=L_{c}(\omega_{o})\int_{\Omega}f_{s}(\omega_{i},\omega_{o})n\cdot \omega_{i}d\omega_{i}\\
L_{c}({\omega_{o}})=\frac
{\int_{\Omega}f_{s}(\omega_{i},\omega_{o})L_{i}(\omega_{i})n\cdot \omega_{i}d\omega_{i}}
{\int_{\Omega}f_{s}(\omega_{i},\omega_{o})n\cdot \omega_{i}d\omega_{i}}
 $$
### Prefilter
 之后的$L_{c}(\omega_{o})$ 的化简结合了使用法线分布函数进行重要性采样，忽略$F_{r}$项，替换掉$G$项，
 以及假设出射和反射方向都是宏观平面法向量方向（即假设所有反射的lobe都是差不多的，这样子会造成在掠射角
 grazing angles看表面时无法观察到拖长的反射），最后可以化简为：
 $$
L_{c}(\omega_{o}) \approx \frac{\sum^{N}_{i}L_{i}(\omega_{i})n\cdot \omega_{i}}
{\sum^{N}_{i}n\cdot \omega_{i}} 
 $$

 在实际计算过程中，如果只是从原始的环境贴图采样，计算出来的预过滤贴图在明亮区域周围会出现点状图案
 ```glsl
    prefiltered_color += texture(environmentMap,L).rgb * NdotL;
    total_weight += NdotL;
 ```
 可以将一个采样点实际代表的立体角大小和pdf结合考虑，估算应该从哪个lod采样，pdf越大，说明这个方向采样的概率越大，应该采样更高分辨率的贴图，pdf越小，应该采样更低分辨率的贴图，出现点状图案是因为采样没有很好的收敛，采样次数不够多，方差还较大，如果pdf小的时候采样高分辨率的贴图就会有很大概率引入较大的方差。
 ```glsl
    float D = DistributionGGX(N,H,roughness);
    float NdotH = max(dot(N, H), 0.f);
    float HdotV = max(dot(H, V), 0.f);
    float pdf = D * NdotH / (4.f * HdotV) + 0.0001f;

    float resolution = PrefilterMapSize;
    float sa_texel = 4.f * PI / (6.f * resolution *resolution);
    float sa_sample = 1.f / (float(SampleCount) * pdf + 0.0001f);

    float mip_level = roughness == 0.f ? 0.f : 0.5f * log2(sa_sample / sa_texel);

    prefiltered_color += textureLod(environmentMap, L, mip_level).rgb * NdotL;
    total_weight += NdotL;
 ```
 ### BRDF LUT
 原来的积分式子移掉$L_{c}(\omega_{o})$之后进行拆项以及带入具体的brdf定义：
 $$
\int_{\Omega}f_{s}(\omega_{i},\omega_{o})n\cdot\omega_{i}d\omega_{i}\\
=\int_{\Omega}f_{s}(\omega_{i},\omega_{o})\frac{F(\omega_{o},h)}{F(\omega_{o},h)}n\cdot\omega_{i}d\omega_{i}\\
=\int_{\Omega}\frac{f_{s}(\omega_{i},\omega_{o})}{F(\omega_{o},h)}(F_{0}+(1-F_{0})(1-\omega_{o}\cdot h)^5)n\cdot \omega_{i}d\omega_{i}\\
F_{0}\int_{\Omega}\frac{f_{s}(\omega_{i},\omega_{o})}{F(\omega_{o},h)}(1-(1-\omega_{o}\cdot h)^5)n\cdot\omega_{i}d\omega_{i}
+\int_{\Omega}\frac{f_{s}(\omega_{i},\omega_{o})}{F(\omega_{o},h)}(1-\omega_{o}\cdot h)^5n\cdot\omega_{i}d\omega_{i}\\
=F_{0}*scale+bias
 $$
这个积分的未知量维度为9维，包括$\omega_{o},n,F_{0}(由metalness和albedo决定),粗糙度\alpha$，一般假设
BRDF都是各向同性的，也就是说不在意$\phi$，只需要$\cos\theta$即$\omega_{o}与n的夹角cos值$，因此对于
预计算scale和bias来说只有两个维度：$(\omega_{o}\cdot{n})和\alpha$。因此可以将其预计算到一张两个通道的二维纹理当中，之后在最终渲染只需要查表就可以获取scale和bias的值。

实际离散计算积分的时候，需要对$scale$根据发现分布函数进行重要性采样，但是法线分布函数是关于$\omega_{h}$，需要转换到$\omega_{i}$，根据法线分布函数的定义，单位面积单位立体角微观表面朝向的百分比或概率，因此其满足如下积分等式，几何意义上可以解释为单位面积dA上所有方向的微观表面投影到该平面的面积和等于单位面积dA：
$$
\int_{\Omega}D(\omega_{h})n\cdot\omega_{h}d\omega_{h}dA = dA \Rightarrow 
\int_{\Omega}D(\omega_{h})n\cdot\omega_{h}d\omega_{h} = 1 \tag 1
$$
$$
\frac{d\omega_{h}}{d\omega_{i}} = \frac{1}{4(\omega_{h}\cdot\omega_{i})} \tag 2
$$
把(2)带入(1)得到：
$$
pdf(\omega_{i}) = \frac{\omega_{h}\cdot\omega_{n}}{4(\omega_{h}\cdot\omega_{i})}D(\omega_{h})
$$
因此对$scale$进行重要性采样的公式为：
$$
scale \approx \frac{1}{N}\sum^{N}_{i}
\frac
{
    \frac{
        D(\omega^{(k)}_{h})G(\omega_{o},\omega^{k}_{i})}{
        4(\omega_{o}\cdot n)(\omega^{(k)}_{i}\cdot n)
    }
    (1-(1-\omega_{o}\cdot\omega^{(k)}_{h})^5)(\omega^{(k)}_{i}\cdot n)
}
{p(\omega^{(k)}_{i})}\\
=\frac{1}{N}\sum^{N}_{i}
\frac
{
    \frac{
        D(\omega^{(k)}_{h})G(\omega_{o},\omega^{k}_{i})}{
        4(\omega_{o}\cdot n)(\omega^{(k)}_{i}\cdot n)
    }
    (1-(1-\omega_{o}\cdot\omega^{(k)}_{h})^5)(\omega^{(k)}_{i}\cdot n)
}
{\frac
{D(\omega^{(k)}_{h})(\omega^{(k)}_{h}\cdot n)}
{4(\omega_{o}\cdot\omega^{(k)}_{h})}}\\
=\frac{1}{N}\sum^{N}_{i}
\frac
{
    
    G(\omega_{o},\omega^{k}_{i})(\omega_{o}\cdot\omega^{(k)}_{h})
    (1-(1-\omega_{o}\cdot\omega^{(k)}_{h})^5)
}
{(\omega^{(k)}_{h}\cdot n)(\omega_{o}\cdot n)}
$$
类似的$bias$为：
$$
bias \approx
\frac{1}{N}\sum^{N}_{i}
\frac
{
    
    G(\omega_{o},\omega^{k}_{i})(\omega_{o}\cdot\omega^{(k)}_{h})
    (1-\omega_{o}\cdot\omega^{(k)}_{h})^5
}
{(\omega^{(k)}_{h}\cdot n)(\omega_{o}\cdot n)}
$$

BRDF的LUT计算代码如下：
```glsl
vec2 IntegrateBRDF(float NdotV, float roughness){
    vec3 V;
    V.x = sqrt(1.f - NdotV * NdotV);
    V.y = 0.f;
    V.z = NdotV;

    float A = 0.f;
    float B = 0.f;

    vec3 N = vec3(0.f,0.f,1.f);
    for(uint i = 0u; i < SampleCount; i++){
        vec2 Xi = Hammersley(i, SampleCount);
        vec3 H = ImportanceSampleGGX(Xi, N, roughness);
        vec3 L = normalize(2.f * dot(V, H) * H - V);

        float NdotL = max(L.z, 0.f);
        float NdotH = max(H.z, 0.f);
        float VdotH = max(dot(V, H), 0.f);
        if(NdotL > 0.f){
            float G = GeometrySmith(N, V, L, roughness);
            float G_Vis = (G * VdotH) / (NdotH * NdotV);
            float Fc = pow(1.f - VdotH, 5.f);

            A += (1.f - Fc) * G_Vis;
            B += Fc * G_Vis;
        }
    }
    A /= float(SampleCount);
    B /= float(SampleCount);
    return vec2(A, B);
}
```

## 最终计算

```glsl
vec3 F0 = mix(vec3(0.04),albedo,metalness);
vec3 F = fresnelSchlickRoughness(wo,normal,F0,roughness);
vec3 kS = F;
vec3 kD = (vec3(1) - kS) * (1 - metalness);

vec3 irradiance = texture(irradianceMap,n).rgb;

vec3 diffuse_color = kD * albedo * irradiance;

vec3 prefilteredColor = textureLod(prefilterMap,wi,roughness*MAX_LOD).rgb;
vec2 brdf = texture(brdfLUT,vec2(max(dot(n,wo),0.0),roughness)).rg;
vec3 specular_color = (F0 * brdf.x + brdf.y) * prefilteredColor;

vec3 color = diffuse_color + specular_color;

```



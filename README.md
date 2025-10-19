> [【从UnityURP开始探索游戏渲染】](https://github.com)**专栏-直达**

MipMap是Unity中用于优化纹理渲染的多级渐远纹理技术，其核心原理是通过预生成一系列分辨率递减的纹理金字塔（从原始尺寸逐级减半至1×1像素），根据物体与摄像机的距离动态选择合适层级的纹理进行采样。在URP（Universal Render Pipeline）中，该技术通过减少远处物体的纹理细节来提升渲染效率，同时避免因纹理缩小导致的摩尔纹和锯齿现象。

# **原理详解**

## ‌**纹理金字塔构建**‌：

* 原始纹理经过滤波处理生成多级缩略图，例如256×256的纹理会生成128×128、64×64等层级，每级分辨率递减50%。

## ‌**动态层级选择**‌：

* GPU根据像素在屏幕空间中的覆盖面积自动计算合适的Mip层级（公式为`level = log2(max(ddx,ddy))`，其中ddx/ddy为纹理坐标导数）。

## ‌**过滤技术配合**‌：

* ‌**双线性过滤**‌：在单一Mip层级内插值。
* ‌**三线性过滤**‌：在相邻两个Mip层级间插值，消除层级切换的突兀感。

# 构建 纹理金字塔

在Unity URP中，Mipmap纹理金字塔的构建是通过GPU逐级下采样实现的，其核心流程分为硬件自动生成和计算着色器手动生成两种方式。

## **硬件自动生成原理**

* ‌**基础纹理处理**‌：原始纹理（如2048×2048）经过双线性/三线性滤波处理，生成分辨率逐级减半的子纹理（1024×1024、512×512等），直至1×1像素。
* ‌**层级关系计算**‌：GPU根据屏幕像素覆盖率自动选择Mip层级，公式为：$lod=log2(max(\frac{\partial u}{\partial x},\frac{\partial v}{\partial y}))$其中偏导数通过纹理坐标微分计算。

## **计算着色器手动生成示例（Hi-Z技术）**

* 通过Compute Shader构建深度纹理的金字塔：
  + HiZFeature.cs

    ```
    |  |  |
    | --- | --- |
    |  | using UnityEngine; |
    |  | using UnityEngine.Rendering; |
    |  | using UnityEngine.Rendering.Universal; |
    |  |  |
    |  | public class HiZFeature : ScriptableRendererFeature { |
    |  | class HiZPass : ScriptableRenderPass { |
    |  | ComputeShader _hizShader; |
    |  | RenderTexture _hizPyramid; |
    |  |  |
    |  | public override void Execute(ScriptableRenderContext context, ref RenderingData data) { |
    |  | CommandBuffer cmd = CommandBufferPool.Get(); |
    |  | // 从深度纹理生成Mipmap金字塔 |
    |  | int width = _hizPyramid.width; |
    |  | int height = _hizPyramid.height; |
    |  | for (int mip = 0; width >= 8 || height >= 8; mip++) { |
    |  | cmd.SetComputeTextureParam(_hizShader, 0, "_HiZTexture", _hizPyramid, mip); |
    |  | cmd.DispatchCompute(_hizShader, 0, width/8, height/8, 1); |
    |  | width >>= 1; height >>= 1; |
    |  | } |
    |  | context.ExecuteCommandBuffer(cmd); |
    |  | CommandBufferPool.Release(cmd); |
    |  | } |
    |  | } |
    |  | } |
    ```
  + HiZ.compute

    ```
    |  |  |
    | --- | --- |
    |  | #pragma kernel CSMain_HiZ |
    |  | Texture2D<float> _DepthTexture; |
    |  | RWTexture2D<float> _HiZTexture; |
    |  |  |
    |  | [numthreads(8, 8, 1)] |
    |  | void CSMain_HiZ(uint3 id : SV_DispatchThreadID) { |
    |  | // 8x8区域取最大深度值作为下采样结果 |
    |  | float maxDepth = 0; |
    |  | for (int x = 0; x < 8; x++) { |
    |  | for (int y = 0; y < 8; y++) { |
    |  | maxDepth = max(maxDepth, _DepthTexture[id.xy * 8 + uint2(x,y)]); |
    |  | } |
    |  | } |
    |  | _HiZTexture[id.xy] = maxDepth; |
    |  | } |
    ```

### **关键步骤解析**

* ‌**层级生成逻辑**‌：每级Mipmap通过对上一级4个像素取平均值（颜色纹理）或最大值（深度纹理）生成，例如256×256纹理生成128×128层级时，每个新像素由原纹理2×2区域计算得出。
* ‌**过滤模式影响**‌：
  + ‌**双线性过滤**‌：在单层Mip内插值4个相邻纹素。
  + ‌**三线性过滤**‌：在相邻两层Mip间进行二次插值，消除层级切换突变。

## **应用场景对比**

| 生成方式 | 适用场景 | 性能开销 |
| --- | --- | --- |
| 硬件自动生成 | 常规颜色纹理 | 低 |
| 计算着色器生成 | 深度纹理/特殊效果（如Hi-Z） | 中高 |

通过调整`RenderTexture.useMipMap`属性和`GenerateMipMaps`标志位可控制生成行为。手动生成更适合需要自定义下采样规则的场景，如遮挡剔除优化.

# 实现动态层级选择

在Unity URP中，Mipmap的动态层级选择是通过GPU硬件自动计算纹理坐标的导数（ddx/ddy）实现的.

## **动态选择原理**

* ‌**导数计算**‌$lod=log2(max(∥\frac{\partial u}{\partial x}∥,∥\frac{\partial v}{\partial y}∥))$

  GPU通过片元着色器中的纹理坐标偏导数（`ddx(uv)`和`ddy(uv)`）确定屏幕像素覆盖的纹理区域大小。当物体距离摄像机越远，UV坐标变化率越大，导数值越高。

  *示例*：若屏幕像素覆盖4×4纹素区域，则Mip层级计算公式为：

  此时计算结果为2（因4=2²），选择Mip Level 2的纹理。
* ‌**层级插值**‌

  启用三线性过滤时，GPU会在计算出的层级（如2.3级）相邻两层（Level 2和Level 3）之间进行插值，消除突变感。
* ‌**LOD偏移控制**‌

  URP通过`Texture2D.mipMapBias`参数手动调整层级偏移，正值使纹理更模糊（强制使用更高层级），负值保留细节（偏向低级）。

## **实现示例**

以下Shader代码演示手动控制Mip层级的两种方式：

```
|  |  |
| --- | --- |
|  | hlsl |
|  | // 方式1：通过tex2Dlod硬编码层级 |
|  | fixed4 frag(v2f i) : SV_Target { |
|  | float4 uv = float4(i.uv, 0, _ManualLod); // _ManualLod为自定义层级 |
|  | return tex2Dlod(_MainTex, uv); |
|  | } |
|  |  |
|  | // 方式2：根据距离动态计算层级 |
|  | float3 worldPos = mul(unity_ObjectToWorld, v.vertex).xyz; |
|  | float dist = distance(_WorldSpaceCameraPos, worldPos); |
|  | float lod = log2(dist * _LodScale); // _LodScale控制距离敏感度 |
|  | fixed4 col = tex2Dlod(_MainTex, float4(i.uv, 0, lod)); |
```

## **层级选择优化**

* ‌**Hi-Z技术**‌

  对深度纹理构建Mip金字塔时，每层级存储最大深度值（而非颜色平均值），加速遮挡剔除。

  *流程*：

  + 生成深度纹理Mipmap时，通过Compute Shader对4×4区域取最大值下采样。
  + 渲染时根据层级深度快速判断可见性。
* ‌**ShaderGraph节点**‌

  URP的`Calculate Level Of Detail`节点提供两种模式：

  + ‌**Clamped**‌：限制在纹理最大层级内，避免越界。
  + ‌**Unclamped**‌：允许超出现有层级，需配合边界处理。

## **性能与质量平衡**

* ‌**过模糊问题**‌：通过`QualitySettings.masterTextureLimit`全局限制最高层级。
* ‌**锐化需求**‌：禁用Mipmap或使用`Unlit` Shader避免自动插值。

典型应用场景包括地形渲染（远山使用高Mip层级）和动态LOD系统（根据物体重要性调整`mipMapBias`）

# **解决的问题**

## ‌**性能优化**‌：

* 减少显存带宽占用，远处物体使用低分辨率纹理降低GPU负载。

## ‌**视觉质量**‌：

* 消除远景的像素闪烁（Texture Aliasing）和锯齿，提升平滑度。

## ‌**缓存命中率**‌：

* 低分辨率纹理占用更少缓存空间，提高采样效率。

# **使用场景与限制**

* ‌**适用场景**‌：
  + 3D开放世界地形（如远山、建筑）。
  + 动态缩放的物体（如角色模型在远距离时）。
* ‌**不适用场景**‌：
  + 2D像素游戏（需保持锐利像素风格）。
  + UI元素（通常无需动态缩放）。
* ‌**限制**‌：
  + 内存开销增加33%（存储多级纹理）。
  + 可能引起远处纹理过度模糊（需调整Mip Bias参数）。

# **具体示例**

* ‌**Shader实现**‌：

  在URP Shader中，可通过`tex2Dlod`函数手动指定Mip层级（`float4`参数的w分量控制层级）。例如：

  ```
  |  |  |
  | --- | --- |
  |  | hlsl |
  |  | fixed4 frag(v2f i) : SV_Target { |
  |  | return tex2Dlod(_MainTex, float4(i.uv, 0, _MipLevel)); |
  |  | } |
  ```

  调整`_MipLevel`可观察不同层级的模糊效果.
* ‌**Unity编辑器演示**‌：

  + 导入纹理后勾选`Generate Mip Maps`，观察Inspector面板中的Mipmap预览滑块（0-10级）。
  + 对比开启/关闭Mipmap的相同纹理，远处物体在开启时会自然模糊，关闭则出现像素噪点。

通过权衡内存与性能，Mipmap在URP中成为优化大规模场景渲染的关键技术之一

---

> [【从UnityURP开始探索游戏渲染】](https://github.com):[逗猫机场](https://doucat6.com)**专栏-直达**
> （欢迎*点赞留言*探讨，更多人加入进来能更加完善这个探索的过程，🙏）

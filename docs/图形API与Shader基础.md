# 图形 API 与 Shader 基础

> 定位：OpenGL／Vulkan 的对象与一帧流程，GLSL／HLSL 的资源接口、矩阵、采样和 Compute。按通用基础与常见实现问题筛选，保留关键代码，省去冗长窗口初始化和 API 名称清单。共 18 题。

[总目录](面试题库总目录.md) · [图形学基础](图形学基础.md) · [硬件与优化](硬件移动端与渲染优化.md) · [高级渲染](高级渲染技术.md)

## 目录

| 模块 | 题号 |
| --- | --- |
| [一、图形 API 与绘制对象](#一、图形 API 与绘制对象) | 1–7 |
| [二、着色语言与数据接口](#二、着色语言与数据接口) | 8–13 |
| [三、常用实现与排查](#三、常用实现与排查) | 14–18 |

## 一、图形 API 与绘制对象

### 1. OpenGL 与 Vulkan 的核心差异是什么？

OpenGL 以当前上下文状态和命令调用组织绘制，驱动承担较多资源与执行管理；Vulkan 显式组织命令缓冲、管线、资源绑定、内存及同步，给应用更多控制，也要求更严格的生命周期设计。

OpenGL 仍有显式同步和内存可见性规则，Vulkan 也仍有状态和规范定义的隐式行为。两者都可能异步执行，不能说 OpenGL 每次 draw 都等 GPU 完成，或 Vulkan 自动比 OpenGL 快。

Direct3D 11／12、Metal、OpenGL ES、WebGL／WebGPU 也有各自资源与平台模型。先理解“资源、管线、命令、同步、呈现”五类职责，再映射到各 API，而非死背版本时间表。

### 2. OpenGL Context 与共享上下文有哪些限制？

调用通常要求线程持有当前上下文；同一上下文不能同时在多个线程作为 current，但可按窗口系统规则解绑后迁移，并非永远固定在创建线程。

共享组可共享纹理、buffer 和部分其他对象，VAO／FBO 等容器对象通常不共享；共享资源不代表共享全部状态。跨上下文读写仍需同步和适用的可见性处理，CPU 互斥锁不自动等价于 GPU 工作已完成。

Core Profile 去掉部分旧接口，Compatibility 保留相关兼容能力；OpenGL ES 的能力另按版本查询，不把桌面 Core 的所有功能套到 ES。

### 3. VAO、VBO、EBO 与索引绘制有什么关系？

VBO／EBO 是按用途称呼的 buffer，分别承载顶点属性与索引；VAO 保存属性格式、步长、偏移、启用状态及相关 buffer 绑定，不复制顶点数据。

传统 `glVertexAttribPointer` 设置时记录当前 ARRAY_BUFFER 来源，ELEMENT_ARRAY_BUFFER 绑定属于 VAO 状态。`glDrawArrays` 按顶点区间，`glDrawElements` 按索引取顶点；索引类型、数量与字节偏移要一致。

索引可复用完全相同的顶点属性组合，UV 接缝或硬法线两侧可能仍需拆顶点。实例化再通过 instance ID／实例属性区分对象，减少提交和几何存储，但不免除每实例的变换和可见性成本。

### 4. 纹理、纹理单元、采样器与 FBO 分别是什么？

| 对象 | 职责 |
| --- | --- |
| Texture | 像素／纹素数据、维度、层级和格式 |
| Texture Unit | OpenGL 中供 shader sampler 引用的绑定槽 |
| Sampler | 过滤、寻址、LOD 等采样规则 |
| FBO | 组织颜色、深度、模板等渲染附件 |

Sampler uniform 常保存纹理单元编号，不是 texture object ID。离屏纹理可作为 FBO 附件，再供下一 Pass 采样；FBO 本身不存储颜色内容。

Renderbuffer 可作为附件而不能直接通过普通 sampler 采样，但不保证一定比同格式纹理省内存。检查 FBO 完整性、附件尺寸／样本规则、draw buffers 与 viewport；不要无同步地把正在写入的同一图像当输入反馈采样。

### 5. 一次 Draw 为什么需要明确状态？

绘制需要匹配顶点输入、程序接口、资源、目标及深度／模板／混合／剔除等状态。一个对象显示错误常来自之前绘制遗留的状态，而不是几何数据本身。

深度测试与写入分开；正面绕序不等于法线方向；透明常保留深度测试、关闭深度写入并远到近排序。Pipeline 对象可打包多种状态，动态状态仍可随命令变化。

改变 Blend／Depth 不要求必然结束 render pass：可在兼容的同一实例中切换管线。逻辑上的“不透明阶段／透明阶段”不等于 API 上必须两个独立 render pass。

### 6. Vulkan 的对象层次和每帧流程怎样描述？

Instance 连接 API 环境；Physical Device 表示可查询的设备能力；Device 是启用相应能力后的逻辑设备；Queue 承接提交，Command Pool 提供命令缓冲。

Surface 关联窗口系统；查询其与设备的兼容性后创建 Swapchain。典型每帧是：确认当前帧资源可复用 → acquire 图像 → 记录／复用命令 → submit → present。

帧槽索引与 swapchain 图像索引不一定相同，在途资源、信号量和图像的所有权应分开管理。窗口变化可能要求重建相关资源；OUT_OF_DATE 与 SUBOPTIMAL 的处理策略也不同，最小化窗口时需处理零尺寸。

### 7. 同步、PSO、Descriptor 等为什么不在这里重复展开？

这些是跨 API 的资源生命周期核心，详见[硬件与优化](硬件移动端与渲染优化.md)：内存与上传第 7–10 题、同步第 11–15 题、命令／描述符／Pipeline 第 16–20 题。

OpenGL 里 `glFlush` 推进命令处理但不等于执行完成，`glFinish` 等待先前命令完成；Fence 可标记更细的完成点，等待它仍可能阻塞。Shader storage／image 写后的消费还要选择适当 memory barrier；Fence 与缓存可见性不是一句“用锁即可”能替代。

Vulkan 图形与计算是不同管线类型，不把 Compute 当作同一个普通 graphics pipeline 的阶段；RT 与 Ray Query 见[高级渲染](高级渲染技术.md)第 8–12 题。

## 二、着色语言与数据接口

### 8. GLSL 与 HLSL 哪些语法要能互相读懂？

| 含义 | GLSL | HLSL |
| --- | --- | --- |
| 浮点向量 | vec2／vec3／vec4 | float2／float3／float4 |
| 顶点位置输出 | gl_Position | SV_Position |
| 片元颜色输出 | out ＋ location | SV_Target |
| 插值数据 | in／out 与相应接口位置 | 结构体成员＋语义，如 TEXCOORD0 |
| 常量数据 | uniform block | cbuffer |
| 纹理采样 | texture(sampler, uv) | texture.Sample(samplerState, uv) |
| 可写数据 | SSBO／image | RWStructuredBuffer／RWTexture 等 UAV |
| 插值与限制 | mix／clamp | lerp／clamp，saturate 限制到 0–1 |

GLSL 的 OpenGL、OpenGL ES 与 Vulkan 使用规则并不完全相同；HLSL 也能服务多种后端，不等于只支持 Direct3D。示例须标明目标环境，不能把 GL 的独立 uniform 布局原样视为 Vulkan 接口。

### 9. uniform、UBO、SSBO、cbuffer 的布局怎样对齐？

常量缓冲适合帧／对象参数，storage buffer 适合较大且可读写的数据；容量、对齐和访问支持由 API 与设备约束，不固定为某个“16 KB”容量。

`std140`、`std430` 与 HLSL cbuffer 打包规则不同，vec3、矩阵、数组和结构体常有 padding。CPU 结构体不能只看字段顺序一样就直接 memcpy；要对照字节 offset、stride、矩阵存储和绑定范围，必要时用反射确认。GPU bool 的表示也不必匹配 C++ bool。

参考：[Microsoft：常量缓冲打包](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-packing-rules)。

### 10. 行主序、列主序和矩阵乘法顺序有什么区别？

存储顺序决定矩阵数据在内存如何排列，乘法约定决定变换怎样作用。列向量常写 `P*V*M*p`；HLSL 用 `mul(M,p)`／`mul(p,M)` 明确乘法侧，普通 `*` 的矩阵语义不能直接套 GLSL。

HLSL 常量矩阵默认打包通常为 column-major，可由声明／设置改变；原材料“默认行主序”不准确。CPU 上传、shader 声明和数学约定须成套匹配，不要为修画面随意加转置。

参考：[Microsoft：矩阵运算与存储](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-per-component-math)。

### 11. smooth、flat 与 noperspective 有何区别？

smooth 按透视校正规则插值，flat 从规定的 provoking vertex 取值，noperspective 在屏幕空间线性插值。整数 ID 等数据不应当成连续颜色插值，阶段两端的类型与限定也须兼容。

`gl_FragCoord`／片元 `SV_Position` 与顶点输出的裁剪坐标含义不同；屏幕位置、UV、视图空间深度和设备深度要区分。像素中心与原点约定依 API／状态，避免硬编码“所有后端 Y 都向上”。

### 12. texture、texelFetch 与显式 LOD 怎么选？

texture 通常按归一化坐标走采样器过滤／寻址和适用的隐式 LOD；texelFetch 用整数纹素位置，按相应重载提供 LOD 或 sample index，不做普通过滤，越界不能指望 sampler wrap 救场。HLSL 对应常见的 Sample、Load、SampleLevel、SampleGrad。

后处理精确读取邻居可用整数位置；纹理缩放需要过滤时按采样器读取。非一致分支里的隐式导数、Compute 中没有普通片元 quad 导数的情境，需要按目标能力选择显式梯度／LOD。Mip 尺度与 TBN 原理见[基础篇](图形学基础.md)。

### 13. mediump、highp、half 是否代表固定硬件位宽？

ESSL 精度限定描述最低精度／范围要求，不保证每个设备严格使用某个固定物理位宽；片元默认精度和 highp 支持须按 ES 版本判断，不能统一说“移动片元默认 mediump”。

HLSL 有 half、最小精度类型以及受目标支持的原生 16 位类型等，含义依目标与设置。大世界坐标、深度和敏感累积保留足够精度，局部颜色／方向可测量后降低；“法线一定低精度够用”也不成立。

## 三、常用实现与排查

### 14. 如何写一个空间一致的基础光照片段？

以下为 OpenGL GLSL 450 的片元示意，配套顶点阶段输出世界位置、正确变换的世界法线和 UV；宿主提供资源与光照参数。它演示经验点光照，完整阴影、曝光和材质流程另行处理。

```glsl
#version 450 core
layout(location=0) in vec3 worldPos;
layout(location=1) in vec3 worldNormal;
layout(location=2) in vec2 uv;
layout(location=0) out vec4 outColor;
layout(binding=0) uniform sampler2D baseMap;
uniform vec3 cameraPos;
uniform vec3 lightPos;
uniform vec3 lightIntensity;

void main() {
    vec3 N = normalize(worldNormal); // 输入须非零
    vec3 delta = lightPos - worldPos;
    float r2 = max(dot(delta, delta), 1e-6);
    vec3 L = delta * inversesqrt(r2);
    vec3 toEye = cameraPos - worldPos;
    vec3 V = toEye * inversesqrt(max(dot(toEye, toEye), 1e-6));
    vec3 sumDir = L + V;
    vec3 H = sumDir * inversesqrt(max(dot(sumDir, sumDir), 1e-6));
    float diffuse = max(dot(N, L), 0.0);
    float specular = diffuse > 0.0
        ? pow(max(dot(N, H), 0.0), 32.0) : 0.0;
    vec3 albedo = texture(baseMap, uv).rgb; // 按颜色格式正确解码
    outColor = vec4((albedo * diffuse + vec3(0.04) * specular)
                    * lightIntensity / r2, 1.0);
}
```

视线必须是 `cameraPos−worldPos`，不是直接 normalize(cameraPos)。非均匀缩放的法线用模型线性变换逆转置；昂贵的公共矩阵可预计算后传入。

### 15. 切线空间法线最容易漏掉什么？

先解码法线纹理，在同一空间构造 T、B、N；正交化 `T=normalize(T−dot(T,N)N)`，再按手性计算 `B=sign×cross(N,T)`。sign 通常来自切线属性，并需考虑变换是否镜像。

最后 `normalize(T*n.x + B*n.y + N*n.z)` 得到着色法线。贴图可能是压缩双通道、绿色方向约定也可能不同，应使用对应解码方式；只写 `B=cross(N,T)` 会在镜像 UV 上出问题。原理及逆转置见基础篇与数学篇。

### 16. Compute Shader 的线程编号、边界和同步怎样处理？

总线程范围由工作组大小与 dispatch 组数决定，常向上取整分组，因此要检查末尾越界。下面为 HLSL 粒子核心，宿主保证 count 不超过 buffer 实际容量：

```hlsl
struct Particle { float4 position; float4 velocity; };
RWStructuredBuffer<Particle> particles : register(u0);
cbuffer SimParams : register(b0) {
    float3 gravity;
    float deltaTime;
    uint count;
    float3 padding;
};
[numthreads(64, 1, 1)]
void CSMain(uint3 id : SV_DispatchThreadID) {
    uint i = id.x;
    if (i >= count) return;
    Particle p = particles[i];
    p.velocity.xyz += gravity * deltaTime;
    p.position.xyz += p.velocity.xyz * deltaTime;
    if (p.position.y < 0.0) {
        p.position.y = 0.0;
        p.velocity.y = abs(p.velocity.y) * 0.8;
    }
    particles[i] = p;
}
```

dispatch 组数为 `ceil(count/64)`，64 是示例选择，不是设备最大线程数。重力先更新速度再更新位置，不能把加速度直接当速度平移。

组内 shared／groupshared 协作须正确使用组屏障，并保证所需线程都到达；不能在部分线程提前 return 后再要求全组屏障。普通组屏障不协调所有工作组，跨 dispatch 或后续绘制读取需 API 层依赖。GLSL SSBO 的无长度数组只能放在允许的末尾位置，可用粒子结构数组，不能在同一块里放两个非末尾无长度数组。

参考：[Microsoft：numthreads](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/sm5-attributes-numthreads)。

### 17. 全屏后处理和 GPU 剔除如何避免多余回读？

全屏三角形／quad 读取上一 Pass 的场景纹理，按尺寸算邻域步长 `1/textureSize`，输出到另一个目标。要区分输入与输出的尺寸、动态分辨率、MSAA resolve、边界寻址和线性／显示编码。

GPU 剔除可以直接写 indirect 参数及计数供绘制消费，不必每帧读回 CPU；需要 CPU 统计时可延迟异步回读。参数布局、buffer usage、写到读取的同步和在途帧存储仍须正确，见硬件篇第 22 题。

### 18. 黑屏、花屏、闪烁怎样有序排查？

先确认 draw／dispatch 实际发生且输出到预期目标；再检查输入顶点、索引、资源绑定、常量字节布局和范围。用固定颜色排除纹理／光照，依次观察 UV、法线、深度，再检查矩阵、viewport、scissor、绕序、深度与混合状态。

帧间才发生的问题优先看资源复用、同步、生命周期和历史缓冲；单帧数值错误优先看空间、精度及格式。RenderDoc 或厂商工具可查看中间附件与管线状态，API 验证信息帮助定位契约错误；性能分析还要把 CPU 提交、GPU 执行和呈现等待分开。

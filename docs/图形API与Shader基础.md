# 图形 API 与 Shader 面试题

> 共 19 题。按主题分类，答案直接放在题目后。

[返回题库总目录](面试题库总目录.md)

## 目录

| 模块 | 题号 |
| --- | --- |
| [一、OpenGL 与 Vulkan](#%E4%B8%80%E3%80%81OpenGL%20%E4%B8%8E%20Vulkan) | 1–8 |
| [二、GLSL 与 HLSL](#%E4%BA%8C%E3%80%81GLSL%20%E4%B8%8E%20HLSL) | 9–14 |
| [三、实现与排查](#%E4%B8%89%E3%80%81%E5%AE%9E%E7%8E%B0%E4%B8%8E%E6%8E%92%E6%9F%A5) | 15–19 |

## 一、OpenGL 与 Vulkan

### 1. OpenGL 和 Vulkan 有什么区别？

**答案：**

OpenGL 通过当前上下文状态组织绘制，驱动管理较多资源与执行细节；Vulkan 使用显式命令缓冲、管线、描述符、内存和同步，让应用更精细控制工作。

Vulkan 更适合明确的多线程录制与复杂调度，但代码和生命周期管理更复杂。两者都可异步执行，OpenGL 也有显式同步要求；实际性能取决于使用方式，不是换 API 就自动更快。

### 2. OpenGL 上下文和共享上下文有什么作用？

**答案：**

上下文保存绘制状态，调用 OpenGL 时通常需要线程绑定当前上下文。同一上下文不能同时在多个线程作为 current，可以按窗口系统规则迁移。

共享组可共享纹理、buffer 等资源，但 VAO／FBO 等容器状态通常不共享。跨上下文使用资源需要正确的 GPU 同步，不能只靠 CPU 锁保证先前绘制已经完成。

### 3. VAO、VBO 和 EBO 有什么关系？

**答案：**

VBO 保存顶点属性，EBO 保存索引，VAO 保存如何从 buffer 读取属性的配置，包括格式、步长、偏移和绑定。

绑定 VAO 后，DrawArrays 按顶点区间绘制，DrawElements 按索引绘制。索引可复用相同属性组合，但 UV 接缝／硬法线可能需要拆顶点。传统 VertexAttribPointer 设置时记录 buffer 来源，EBO 绑定属于 VAO 状态。

### 4. 纹理单元和 Sampler 是什么？

**答案：**

纹理对象保存数据，纹理单元是 shader sampler 引用的绑定槽，Sampler 描述过滤、寻址、LOD 等规则。

OpenGL 的 sampler uniform 通常设置纹理单元编号，不是 texture object ID。HLSL 常把 Texture 与 SamplerState 分开声明；同一张纹理可用不同采样状态访问。

### 5. FBO 是什么？如何进行离屏渲染？

**答案：**

FBO 将颜色、深度和模板附件组成渲染目标，本身不存储图像。绑定 FBO 后设置视口并绘制，即可把结果写到纹理或 renderbuffer 附件。

结果纹理随后可用于阴影、后处理、反射或 G-buffer 读取。检查附件完整性、格式、尺寸／样本及 draw buffers；需要后续普通采样时使用纹理，不能直接采样 renderbuffer。

### 6. glFlush、glFinish 和 Fence Sync 有什么区别？

**答案：**

glFlush 推进命令处理但不等待全部执行完成；glFinish 阻塞主机直到先前工作完成；Fence 在命令流放一个完成标记，允许查询或等待某一位置。

频繁 glFinish 会破坏 CPU／GPU 重叠，通常优先使用更精确的资源回收信号。Shader storage／image 写后读取还要满足相应内存可见性规则。

### 7. Vulkan 的基本对象和一帧流程是什么？

**答案：**

Instance 连接 API 环境，Physical Device 用于查询设备能力，Device 启用需要的队列和特性，Command Pool 分配命令缓冲，Queue 接受提交；Surface 与 Swapchain 连接窗口呈现。

一帧通常是：确认帧资源可复用 → acquire 交换链图像 → 记录／复用绘制命令 → submit → present。帧槽和交换链图像索引分别管理，窗口尺寸变化时按结果重建相关资源。

### 8. PSO、Descriptor 和同步在一次绘制中各负责什么？

**答案：**

PSO／Pipeline 定义程序和渲染状态；Descriptor 指向 Shader 使用的资源；命令缓冲记录绑定、绘制和复制；同步保证前序写入可被后续合法读取，并管理资源复用。

完整实现还需正确的目标附件、viewport、深度／混合状态。具体 barrier、内存和 Tile 优化见[硬件题库](硬件移动端与渲染优化.md)。

## 二、GLSL 与 HLSL

### 9. GLSL 和 HLSL 的常见类型、接口有什么对应关系？

**答案：**

| 用途 | GLSL | HLSL |
| --- | --- | --- |
| 浮点向量 | vec2／vec3／vec4 | float2／float3／float4 |
| 顶点位置输出 | gl_Position | SV_Position |
| 片元颜色输出 | out ＋ location | SV_Target |
| 阶段间数据 | in／out | 结构体成员＋语义 |
| 常量数据 | uniform block | cbuffer |
| 纹理采样 | texture | Texture.Sample |
| 可写数据 | SSBO／image | RWStructuredBuffer／RWTexture |

GLSL 的 GL／ES／Vulkan 环境存在接口差异，HLSL 也能服务不同后端。示例要明确运行环境。

### 10. UBO、SSBO 和 cbuffer 有什么区别？如何对齐 CPU 数据？

**答案：**

UBO／cbuffer 适合只读的帧、对象和材质参数；SSBO／storage buffer 可承载较大、可写的数组数据，用于粒子、剔除等。

CPU 和 GPU 要逐字段匹配 offset、数组 stride、矩阵布局和绑定范围。std140、std430、HLSL cbuffer 打包规则不同，vec3 和数组常有填充，不能只看字段名字一致就直接复制结构体。

### 11. 矩阵的行主序、列主序和乘法顺序有什么区别？

**答案：**

主序决定内存排列，向量乘在哪侧决定数学变换顺序。列向量常写 `P×V×M×p`，HLSL 用 `mul(M,p)` 或 `mul(p,M)` 表达。

GLSL 矩阵运算与 HLSL 普通 `*` 的语义不能直接混用；HLSL 常量矩阵默认打包通常为 column-major，也可显式指定。CPU 上传和 Shader 声明要保持一致。

### 12. smooth、flat 和 noperspective 有什么区别？

**答案：**

smooth 进行透视校正插值，适用于一般 UV 等属性；flat 不插值，使用规定顶点的值，适合整数 ID；noperspective 在屏幕空间线性插值。

插值法线仍需归一化。顶点输出的裁剪位置与片元输入的屏幕位置含义不同，不能直接混用。

### 13. texture、texelFetch、textureLod 和 textureGrad 有什么区别？

**答案：**

texture 按采样器进行过滤、寻址和适用的自动 LOD；texelFetch 按整数纹素坐标读取，不做普通过滤；textureLod 显式指定层级；textureGrad 提供梯度控制层级与过滤。

HLSL 对应常见的 Sample、Load、SampleLevel 和 SampleGrad。精确读 G-buffer 可用 Load／texelFetch；缩放纹理需过滤时用采样器接口。整数读取越界不能依赖普通寻址模式修正。

### 14. highp、mediump、half 应怎么选？

**答案：**

位置、深度、大范围 UV 和敏感累积通常需要较高精度；颜色、局部方向等可在验证误差后使用较低精度，以减少寄存器或运算成本。

ESSL 限定描述最低精度要求，HLSL 的 half／最小精度／原生 16 位类型也依目标设置，不能把它们全部视为固定物理位宽。相机相对坐标可减少大世界数值精度问题。

## 三、实现与排查

### 15. 如何在 Shader 中计算 Blinn–Phong 光照？

**答案：**

把位置、法线、光源和相机统一到世界或视图空间，先求方向，再计算漫反射与高光：

```glsl
vec3 N = normalize(worldNormal);
vec3 L = normalize(lightPos - worldPos);
vec3 V = normalize(cameraPos - worldPos);
vec3 H = normalize(L + V);
float diffuse = max(dot(N, L), 0.0);
float specular = diffuse > 0.0
    ? pow(max(dot(N, H), 0.0), shininess) : 0.0;
vec3 color = ambient + attenuation * lightColor
             * (albedo * diffuse + specColor * specular);
```

这是方向非退化时的公式示例。法线在非均匀缩放下用逆转置变换；V 是“表面指向相机”，不是直接 normalize(cameraPos)。

### 16. 切线空间法线如何转换到世界空间？

**答案：**

先对 T 与 N 正交化，利用手性构造副切线 B，再组合纹理中的三个分量：

```glsl
vec3 N = normalize(worldNormal);
vec3 T = normalize(worldTangent - dot(worldTangent, N) * N);
vec3 B = tangentSign * cross(N, T);
vec3 nt = texture(normalMap, uv).xyz * 2.0 - 1.0;
vec3 result = normalize(T * nt.x + B * nt.y + N * nt.z);
```

手性须考虑镜像 UV／变换。压缩法线贴图可能只保存两个分量，应使用匹配的解码；绿色通道方向也要与烘焙工具一致。

### 17. Compute Shader 如何更新粒子？线程数和同步要注意什么？

**答案：**

每线程处理一个粒子，先按加速度更新速度，再更新位置。宿主按组大小向上取整 dispatch，尾组线程检查越界。

```hlsl
struct Particle { float4 position; float4 velocity; };
RWStructuredBuffer<Particle> particles : register(u0);
cbuffer SimParams : register(b0) {
    float3 gravity; float deltaTime;
    uint count; float3 padding;
};
[numthreads(64, 1, 1)]
void CSMain(uint3 id : SV_DispatchThreadID) {
    if (id.x >= count) return;
    Particle p = particles[id.x];
    p.velocity.xyz += gravity * deltaTime;
    p.position.xyz += p.velocity.xyz * deltaTime;
    particles[id.x] = p;
}
```

组数为 `ceil(count/64)`，宿主保证 count 不超过容量。组内协作需让相关线程都到达组屏障，组屏障不协调全部工作组；后续绘制读取需 API 层同步。

### 18. 全屏后处理怎么实现？GPU 剔除结果需要回读 CPU 吗？

**答案：**

全屏三角形或 quad 读取上一 Pass 场景纹理，对邻域采样并写到另一目标。注意输入尺寸、UV、边界寻址以及颜色空间，不要无约束地同时读写同一图像。

GPU 剔除可直接写 indirect 参数和计数供绘制消费，无需逐帧回读。只有 CPU 确实需要结果时再考虑异步统计；立即回读等待可能破坏流水线。

### 19. 黑屏或画面错误应如何排查？

**答案：**

按数据流检查：绘制是否执行、目标是否正确 → 顶点／索引与属性布局 → Shader 资源和常量 → 矩阵、viewport、scissor → 剔除、深度、混合状态。

先输出固定颜色，再依次可视化 UV、法线和深度，逐步恢复光照。跨帧闪烁优先查同步和资源复用；RenderDoc 等工具可查看单帧中间结果，验证信息帮助定位 API 使用错误。

## 参考资料

- [Microsoft：HLSL 矩阵](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-per-component-math)
- [Microsoft：常量缓冲布局](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-packing-rules)

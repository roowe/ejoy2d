# ejoy2d 图形渲染入门 · 番外篇：CPU 的数据怎么到 GPU？GPU 会哪些基本操作？

> 定位：这是[第一讲](01-第一讲-基础心智模型.md)与[第二讲](02-第二讲-精灵树与动画.md)之间的**桥梁篇**。
> 第一讲讲了帧缓冲、三角形、shader、合批"是什么"，第二讲讲了引擎怎么组织精灵和批次；这一篇回答两个更底层的问题：
>
> 1. 我在 C 里准备的数组、图片、shader 代码，**到底怎么跑到 GPU 那边去的**？
> 2. GPU 接到命令后，**按哪些固定步骤**把数据变成像素？
>
> 涉及文件：`lib/render/render.c`、`lib/shader.c`、`lib/renderbuffer.c`、`ejoy2d/shader.lua`。

---

目录

1. [两台隔着"桥"的处理器](#1-两台隔着桥的处理器)
2. [运过桥的四类东西（GL 对象与操作对照表）](#2-运过桥的四类东西gl-对象与操作对照表)
3. [数据是"复制"过去的，不是传指针](#3-数据是复制过去的不是传指针)
4. [glDrawElements 是"工单"，不是当场画图](#4-gldrawelements-是工单不是当场画图)
5. [GPU 的六个固定动作（渲染流水线）](#5-gpu-的六个固定动作渲染流水线)
6. [GPU 的编程观：你不写 for，它并行跑无数份](#6-gpu-的编程观你不写-for它并行跑无数份)
7. [ejoy2d 一帧的数据走向总图](#7-ejoy2d-一帧的数据走向总图)
8. [为什么这正好解释了"合批"](#8-为什么这正好解释了合批)
9. [术语表](#9-术语表)
10. [自测题与答案](#10-自测题与答案)
11. [代码位置索引](#11-代码位置索引)

---

## 1. 两台隔着"桥"的处理器

CPU 和 GPU 是**两个独立的计算核心，各有各的内存**。你 C 代码里的数组、指针、变量，GPU 一个都看不见。两者之间只能通过**显卡驱动提供的 OpenGL 接口**通信：

```
┌──────────────── CPU + 内存 ────────────────┐        ┌──────────── GPU + 显存 VRAM ───────────┐
│                                            │        │                                          │
│  RS->vb[] 顶点数组（renderbuffer.h）        │  glBufferData   ─▶  VBO / IBO（显存里的缓冲）    │
│  sample.1.ppm 图片像素                      │  glTexImage2D   ─▶  纹理对象（可采样的图片）      │
│  shader.lua 里的 GLSL 字符串                │  glCompile/glLink ─▶ Shader Program（GPU 程序） │
│  少量常量（缩放/颜色矩阵）                  │  glUniform      ─▶  uniform（程序参数）          │
│                                            │  glVertexAttribPointer ─▶ VAO（怎么读字节）     │
│                                            │  glDrawElements ─▶  一道"开工工单"（命令队列）   │
│                                            │        │                                          │
└────────────────────────────────────────────┘        │        成千上万个小核心并行处理             │
                                                      │                  │                       │
                                                      │                  ▼                       │
                                                      │          帧缓冲 framebuffer（一张图）       │
                                                      │                  │ swapbuffers           │
                                                      └──────────────────┼───────────────────────┘
                                                                         ▼ 屏幕
```

三个必须建立的认知：

1. **GPU 不认识你的内存，只认识你"运过桥"的对象。** 顶点要先复制进 VBO，图片要先复制进纹理，shader 要先编译上传。
2. **`glDrawElements` 不是当场画图，而是递一张工单。** 它把命令塞进驱动的命令队列就返回，GPU 在另一边稍后批量、异步地执行。
3. **过桥很慢，GPU 干活极快。** 数据传输、状态切换、下工单都有不便宜的固定开销；GPU 用几千个小核心并行算顶点和像素，单个反而极廉价。

> Mac（Apple Silicon）上 CPU/GPU 物理上共用内存，但 OpenGL 的**逻辑模型**不变：照样要创建 buffer/texture 对象、照样走驱动，不能当成能直接共享指针。

---

## 2. 运过桥的四类东西（GL 对象与操作对照表）

| 你要给 GPU 什么 | GL 对象 | 关键 GL 操作 | ejoy2d 位置 |
| --- | --- | --- | --- |
| 一块装**顶点/索引**的显存 | VBO / IBO（Buffer） | `glGenBuffers`、`glBufferData` | render.c:155 建缓冲、:158 静态 IBO、:180 动态 VBO |
| 一张可按坐标采样的**图片** | 纹理 Texture | `glGenTextures`、`glTexImage2D` | render.c:614、:737 |
| 一段在 GPU 上跑的**程序** | Shader Program | `glCreateShader/glShaderSource/glCompileShader`、`glCreateProgram/glLinkProgram`、`glUseProgram` | render.c:213、:315、:457 |
| 程序运行时的**常量参数** | uniform | `glGetUniformLocation`、`glUniform*` | render.c:1055、:1067 |
| "**怎么解读 VBO 字节**"的说明书 | VAO（顶点数组对象） | `glGenVertexArrays`、`glVertexAttribPointer`、`glBindVertexArray` | render.c:329、:569、:549 |
| 下工单开跑 | — | `glDrawElements` | render.c:1022 |

两类东西的上传时机不同，这点很重要：

- **启动时传一次、之后常驻显存**：纹理（ppm 图片）、编译好的 shader、静态 IBO；
- **每批发车时重传**：当前这批矩形的 VBO 顶点（`render_buffer_update` → `glBufferData`，render.c:180）。

---

## 3. 数据是"复制"过去的，不是传指针

`glBufferData(target, 字节数, CPU指针, 用法)` 做的事：驱动在显存里开一块区域，把你 CPU 数组里的字节**拷贝**进去。从此 GPU 用的是那份副本，你改原数组不会影响已上传的数据（要再传一次）。

- IBO 内容永不变，用 `GL_STATIC_DRAW`，启动传一次（render.c:158）；
- VBO 每批都变，用 `GL_DYNAMIC_DRAW` 反复重传（render.c:180）。每批重新 `glBufferData` 指定一次存储叫 **buffer orphaning（缓冲孤立）**：驱动干脆另给一块新显存，避免停下来等 GPU 读完旧数据，减少 CPU/GPU 互相等待。

图片走的是纹理专用通道：`glTexImage2D`（render.c:737）把像素传进纹理对象，之后片元着色器用 `texture2D(纹理, uv)` 按坐标取色。

shader 源码（`ejoy2d/shader.lua` 里的字符串）也要"过桥"，但它过的是**编译通道**：源码字符串 → `glShaderSource` → `glCompileShader` 编译成 GPU 指令 → 挂到 program 上 `glLinkProgram` 链接 → `glUseProgram` 切换当前程序。这在启动时做一次。

---

## 4. glDrawElements 是"工单"，不是当场画图

理解异步命令队列是这一篇的关键：

```
CPU 线程：  glBufferData(运料) → glUseProgram/glBindTexture(设置) → glDrawElements(递工单) → 立刻返回继续干别的
                                                              │
驱动维护的命令队列（缓冲）  ──────────────────────────────────▶  GPU 稍后按顺序执行
```

工单上写着："用当前绑定的 **shader、VBO、IBO、纹理、uniform、混合状态**，从第 `from` 个索引开始，画 `ni` 个索引（三角形）"。

ejoy2d 里这张工单是（render.c:1022）：

```c
glDrawElements(GL_TRIANGLES, ni /*=6*矩形数*/, GL_UNSIGNED_SHORT, offset);
```

因为工单只记录"当前状态"，所以**一旦递出去就不能改了**；想换纹理/shader，必须先把上一批画完（`rs_commit`），再设置新状态、递新工单——这就是第二讲 B"冻结元组"的硬件根源。

---

## 5. GPU 的六个固定动作（渲染流水线）

`glDrawElements` 一响，GPU 按一条**固定流水线**处理。其中只有两步运行你写的程序（顶点着色器、片元着色器），其余都是 GPU 固化的硬件单元：

```
VBO（顶点）+ IBO（索引）+ 当前纹理/uniform/状态
│
① 输入装配 Input Assembler（固化硬件）
│     按 IBO 的索引从 VBO 取顶点，每 3 个拼成一个三角形
│     （ejoy2d 的 0 1 2 / 0 2 3 在这里被解释成两个三角形）
│
② 顶点着色器 Vertex Shader（你写的，每个顶点跑 1 次，大量并行）
│     输入：这一个顶点的 position / texcoord / color / additive
│     干活：算出它在 NDC 的位置（ejoy2d 里只是 position+(-1,+1)）
│     输出：gl_Position，以及往下传的 varying（uv、颜色）
│
③ 光栅化 Rasterization（固化硬件）
│     看三角形盖住了哪些像素格子，每个格子生成一个"片元(fragment)"
│     自动把三个顶点的 uv、颜色在三角形内部做线性插值，发给每个片元
│
④ 片元着色器 Fragment Shader（你写的，每个片元跑 1 次，海量并行）
│     按插值来的 uv 用 texture2D 采纹理，乘顶点色、加发光色
│     输出这一个片元的 RGBA
│
⑤ 逐片元测试与混合（固化硬件）
│     裁剪测试 scissor、深度测试、alpha 混合……
│     ejoy2d 的 2D 主要用混合：新像素和帧缓冲里已有像素按 blend 公式合成
│
⑥ 写帧缓冲 + 上屏（固化）
│     结果写进 framebuffer 这张图；一帧结束 swapbuffers 把它送到屏幕
▼
屏幕看到画面
```

①③⑤⑥ 是 GPU 硬件固定好的，你只能开关/设参数；②④ 是两个"可编程插槽"，你用 GLSL 填代码。

---

## 6. GPU 的编程观：你不写 for，它并行跑无数份

这是从 CPU 思维转到 GPU 思维最容易卡住的地方：

- **你不写"遍历所有顶点/像素"的循环。** 你只写"**一个**顶点怎么处理"和"**一个**片元怎么处理"两段小程序。GPU 把同一段程序在几千个小核心上对几万个顶点、几百万像素**同时各跑一份**（这种模式叫 SIMT）。
- **顶点着色器和片元着色器之间不能直接通信**，也互不知道对方存在。它们唯一的纽带是 `varying` 变量：顶点阶段写，光栅化阶段自动插值，片元阶段读（ejoy2d 里就是 `v_texcoord / v_color / v_additive`）。
- **同一次 draw 里所有顶点共享同一套状态**：同一个 shader、同一张纹理、同一组 uniform、同一种混合。想让不同矩形用不同纹理/shader，就只能拆成多道工单（多个 draw call）。
- 顶点着色器**逐顶点**跑（一个 quad 跑 4 次）；片元着色器**逐被覆盖的片元**跑（数量约等于三角形盖住的像素数）。

---

## 7. ejoy2d 一帧的数据走向总图

```
【启动阶段：只做一次，常驻显存】
  sample.1.ppm 像素 ── glTexImage2D ──▶ 纹理对象
  shader.lua 的 GLSL ── 编译/链接 ──▶ Shader Program
  索引 0 1 2 / 0 2 3 … ── glBufferData(STATIC) ──▶ IBO

【每一帧】
  CPU：递归精灵树（第二讲 A），把成百上千个矩形算成 4 顶点，
       攒进内存数组 RS->vb（renderbuffer.c:19）  ← 此刻还在 CPU 内存，GPU 完全不知道
       │
       │  攒满 1024 / 换纹理 / 换 shader / 帧末  → rs_commit（shader.c:201）
       ▼
  过桥（每趟一车）：
       glBufferData(DYNAMIC)  把这批顶点复制进 VBO   render.c:180
       glDrawElements         递工单（异步，不等）    render.c:1022
            │
            ▼  GPU 流水线：
            ①装配三角形 → ②顶点着色器(每顶点) → ③光栅化插值
            → ④片元着色器(每片元，采已在显存的纹理) → ⑤混合 → ⑥写帧缓冲
       重复若干趟……
  帧末：shader_flush 运走最后一批（shader.c:314）
       swapbuffers：帧缓冲 → 屏幕
```

对照一个角点的坐标命运（详见第二讲 B）：本地角点经定点矩阵、`screen_trans` 变成 NDC 前的 (1,-1)，顶点着色器加 (-1,+1) 得到 NDC (0,0)——**矩阵在 CPU 算，最后的加法在 GPU 的第②步做**。

---

## 8. 为什么这正好解释了"合批"

把这一篇和第二讲 B 合起来，合批的理由就彻底通了：

- **过桥和驱动有固定成本**：每次 `glBufferData`、每次切换状态、每道 `glDrawElements` 工单都有开销，和你画几个三角形关系不大；
- **GPU 并行极廉价**：一道工单画 2 个三角形和画 6000 个三角形，对它差别很小；
- 所以优化方向是：**让尽量多的矩形共用同一状态（同 shader/纹理/混合），攒满一车、只过一次桥、只下一道工单**。
- 打断合批的，正是"工单状态必须改变"的时刻：换纹理、换 shader、换混合、改 uniform、车装满、帧末——这些就是第二讲 B 的"冻结元组"判据。

---

## 9. 术语表

| 术语 | 一句话 |
| --- | --- |
| 显存 VRAM | GPU 自己的内存；CPU 不能直接用指针访问 |
| 驱动 / GL 上下文 | CPU 与 GPU 之间的桥，GL 调用在这里变成命令 |
| VBO / IBO | 显存里装顶点 / 索引的缓冲对象 |
| 纹理 Texture | 显存里的图片，可用 uv 坐标采样 |
| Shader Program | 编译链接后运行在 GPU 的程序（VS+FS） |
| uniform | 一次 draw 内所有顶点/片元共享的常量参数 |
| VAO | 录制"绑哪个 VBO/IBO + 属性怎么读"的状态对象 |
| glBufferData | 把 CPU 内存复制进 GPU 缓冲 |
| glDrawElements | 下一道按索引画三角形的工单（异步） |
| 命令队列 | 驱动缓存 GL 命令、交给 GPU 异步执行 |
| 输入装配 | 按索引取顶点、拼三角形的固化阶段 |
| 光栅化 | 三角形 → 片元，并对 varying 插值的固化阶段 |
| 片元 fragment | 一个候选像素，带颜色/深度等信息 |
| 帧缓冲 framebuffer | GPU 输出的那张图 |
| SIMT | GPU 把同一段小程序并行跑在海量数据上的模式 |
| buffer orphaning | 每批重指定缓冲存储，换取不等待 GPU |
| STATIC_DRAW / DYNAMIC_DRAW | 缓冲"一次上传" / "频繁重传"的用法提示 |

---

## 10. 自测题与答案

**Q1. CPU 数组里的顶点，GPU 能直接用指针读吗？要怎样才能给 GPU 用？**

答：不能。CPU/GPU 内存分离，必须用 `glBufferData` 把顶点字节复制进显存里的 VBO 对象，GPU 才能读到。

**Q2. `glDrawElements` 调用返回时，画面已经画好了吗？**

答：没有。它只是往驱动命令队列递了一道工单就返回，GPU 在另一边异步执行；真正上屏要等帧末 `swapbuffers`。

**Q3. 哪些资源启动时传一次就行，哪些要每帧/每批重传？**

答：纹理、编译好的 shader、静态 IBO 启动传一次并常驻显存；动态 VBO（当前这批矩形顶点）每批 `rs_commit` 时用 `glBufferData(DYNAMIC_DRAW)` 重传。

**Q4. GPU 流水线六个动作里，哪两个是你写的程序？其余是什么？**

答：②顶点着色器和④片元着色器是可编程的（写 GLSL）；①输入装配、③光栅化、⑤测试与混合、⑥写帧缓冲是固化硬件单元，只能开关或设参数。

**Q5. 为什么 shader 里不用 for 循环遍历所有顶点/像素？**

答：GPU 是 SIMT 并行模型：你只写"一个顶点/一个片元怎么算"，GPU 用几千个小核心对全部顶点、全部片元同时各跑一份，不需要也不应该由你写循环。

**Q6. 顶点着色器里的 uv，靠什么到达片元着色器？**

答：靠 `varying` 变量。顶点阶段写出后，光栅化阶段在三角形内部对它做线性插值，每个片元拿到自己位置上的 uv，片元阶段再用它采纹理。

**Q7. 为什么一次 draw 不能让这批矩形一半用纹理 A、一半用纹理 B（不用多纹理技术）？**

答：一次 draw 的所有片元共享同一个绑定纹理和同一套状态；工单一旦交出无法中途换纹理。要混用就得拆成两道工单（两个 draw call），或把图拼进同一张纹理图集。

**Q8. 为什么"合批、减少 draw call"能提速？**

答：数据传输、状态切换、下工单都要过桥、有固定开销，而 GPU 并行画大量三角形很便宜。让更多矩形同状态攒成一批，就能减少过桥次数和工单数（draw call），把 GPU 的并行能力喂饱。

---

## 11. 代码位置索引

| 主题 | 位置 |
| --- | --- |
| 建缓冲 / 上传顶点（CPU→GPU 复制） | `lib/render/render.c`（`render_buffer_create` 140、`glBufferData` STATIC 158、DYNAMIC 180） |
| 上传图片为纹理 | `lib/render/render.c`（`render_texture_create` 610、`glTexImage2D` 737） |
| 编译/链接/切换 shader | `lib/render/render.c`（`compile` 210、`render_shader_create` 310、`glUseProgram` 457） |
| 设置 uniform | `lib/render/render.c`（`render_shader_setuniform` 1064） |
| VAO / 顶点属性解读 | `lib/render/render.c`（glGenVertexArrays 329、`apply_va` 544、`glVertexAttribPointer` 569） |
| 下工单 glDrawElements | `lib/render/render.c`（`render_draw` 1004，调用 1022） |
| 攒批 / 发车 | `lib/shader.c`（`renderbuffer_add` 调用在 285、`rs_commit` 201、`shader_flush` 314）、`lib/renderbuffer.c:19` |
| 顶点/四边形内存结构 | `lib/renderbuffer.h:9` |
| 真正的 VS/FS 源码 | `ejoy2d/shader.lua`（sprite_vs 29、sprite_fs 14） |

---

> **关联讲义**
>
> - 概念入门：[01-第一讲-基础心智模型](01-第一讲-基础心智模型.md)
> - CPU 侧怎么组织数据：[02-第二讲-精灵树与动画](02-第二讲-精灵树与动画.md)
> - 过桥与状态切换的工程细节：[03-第二讲B-合批与渲染抽象层](03-第二讲B-合批与渲染抽象层.md)
> - 下一讲（第二讲 C）：文字 label 的动态字体纹理、粒子 anchor、离屏渲染 render target。

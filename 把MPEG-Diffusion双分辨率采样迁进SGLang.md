# 把 MPEG-Diffusion 的双分辨率采样迁进 SGLang：一次 2 倍加速的移植复盘

> 一篇纯技术复盘：怎么把一个「论文级」的视频生成加速方法，落到一个已经生产化的推理框架里；哪些地方真省了算力，哪些地方是幻觉，以及为什么最后速度达标、画质不达标。
> 文中机器地址、内部路径、服务端口、业务项目均已脱敏。

## TL;DR

- 目标：MiniMax-H3 系 DiT，2×A100-80G，TP2、INT8（convrot）量化、**6 步蒸馏**采样；单条 768×1344 / 158 帧 / 6.58 秒的视频，端到端 **163.5 秒**。
- 先量化：整个请求 **160.4 秒是纯 GPU kernel**，denoise 占 93.7%，其中 **FlashAttention 占 60%（96.4 秒）**；denoise 段 GPU 利用率 **99.8%**，调度层没有空泡。
- 第一次尝试换 attention kernel（SageAttention）：attention 只快 **9.7%**，且输出出现系统性偏移（PSNR 27 dB / SSIM 0.84），回退。
- 真正的杠杆是 **MPEG-Diffusion 的双分辨率采样**（two-track）：denoise **139.5 → 59.9 秒（2.33×）**，端到端 **163.5 → 80.2 秒（2.04×）**，且已经等于该配置的理论天花板。
- 但画质低于基线（PSNR 21.9 dB / SSIM 0.709），结论是**机制验证通过、不可上生产**。

---

## 一、先把时间轴拆开：profiling 的方法与三个坑

### 1.1 三个坑

**坑一：分段计时会低报。**
框架自带 stage 计时（`[...] finished in N seconds`）和 `perf_dump`，但在未开启 stage 同步时是「启动侧」的时间：日志报 denoise 139.5 秒，torch profiler 的 GPU kernel 时间是 **150.3 秒**，差 8%。

**坑二：第 0 步的数字是假的。**
`perf_dump` 会给出「step 0 = 14.0 秒、其余每步 25.0 秒」的台阶。trace 显示**每一步都是 24.87 秒**——第 0 步那个 14 秒只是 CPU 抢跑、GPU 队列还没被填满时的测量假象。三次运行完全复现。

**坑三：tqdm 的 `it/s` 是累计平均。**
日志里每步耗时「15 → 22 → 24 → 25 → 26 秒」看着像热降频，其实是平均值在收敛；`nvidia-smi` 同期读数 1.29–1.41 GHz、约 400 W、低于 65 ℃，没有降频。

**结论：判断加速只看 GPU kernel 时间，其次看端到端 wall；不信 stage 计时，不信 tqdm。**

### 1.2 采集方式

框架支持「按请求」开 profiler，不用重启服务：在请求体里加 `profile: true`、`profile_all_stages: true`、`num_profiled_timesteps: -1`，trace 会落在服务进程工作目录的 `logs/<request-id>-full_stages-global-rank0.trace.json.gz`（这次约 51 MB gz）。

分析 trace 用的是三个小脚本：

| 脚本 | 作用 |
|---|---|
| `summarize_trace.py` | 按 kernel 名聚合总时长与调用次数，打印 top-N；统计 memcpy/memset |
| `summarize_trace2.py` | 把 kernel 归因到 pipeline 各 stage（利用 `STAGE xxx` 的 record_function 区间），再按「每 50 次 attention 调用」切开 denoise 的 6 步 |
| `inspect_shapes.py` | 取 kernel 的 grid/block/寄存器、以及 aten 算子的输入维度，用来反推 token 数 |

### 1.3 结果

**Stage 拆分（同一请求）：**

| stage | stage 计时 | GPU kernel | 利用率 |
|---|---|---|---|
| 文本编码 | 2.35 s | ~0.6 s | — |
| 参考图编码 | 2.22 s | 2.19 s | 95 % |
| latent / timestep 准备 | 0.02 s | 0.00 s | — |
| **denoise（6 步）** | 139.5 s | **150.3 s** | **99.8 %** |
| VAE 解码 | 8.10 s | 7.92 s | 97.4 % |
| 合计 | 163.9 s | 160.4 s | — |

整个请求只有 **0.3 秒**的 GPU 空泡。这一条直接排除了「减少调度开销」类优化：CUDA Graph、算子融合调度、文本编码缓存（只占 0.6 秒）、VAE 调 tile（只占 4.9%）都不值得动。

**denoise 内部（每步 24.87 秒）：**

| kernel | 每步 | 占 denoise | 每步调用数 | 备注 |
|---|---|---|---|---|
| FlashAttention2 bf16（`FlashAttnFwdSm80`） | 16.07 s | 64.6 % | 50（每层 1 次） | 单次约 318 ms，head_dim 128 |
| FP32 cuBLAS（`ampere_sgemm_128x64_tn`） | 2.67 s | 10.7 % | 344 | 单次 5.6 ms |
| INT8 cutlass GEMM | 2.62 s | 10.5 % | 196 | 4 个主 GEMM × 50 层，约 40 % INT8 峰值 |
| FP32 cuBLAS（`ampere_sgemm_32x128_tn`） | 0.89 s | 3.6 % | 148 | |
| elementwise add / direct_copy | ~1.0 s | 4.0 % | ~380 | 访存型 |
| NCCL AllReduce bf16 RING | 0.47 s | 1.9 % | ~100 | 每层 2 次，4.7 ms/次 |
| INT8 激活量化（convrot64） | 0.47 s | 1.9 % | ~200 | 动态量化 |
| AdaLN 索引调制（gate / scale-shift） | 0.36 s | 1.4 % | ~200 | |
| SwiGLU / fused qknorm+rope | 0.15 s | 0.6 % | ~100 | 已融合 |

**全程 top kernel（160.4 秒口径）：**

| 时间 | 占比 | 次数 | kernel |
|---|---|---|---|
| 96.41 s | 60.1 % | 302 | `FlashAttnFwdSm80` |
| 16.36 s | 10.2 % | 2114 | `ampere_sgemm_128x64_tn`（FP32） |
| 16.05 s | 10.0 % | 1209 | cutlass INT8 GEMM |
| 5.46 s | 3.4 % | 906 | `ampere_sgemm_32x128_tn`（FP32） |
| 3.60 s | 2.4 % | 1200 | elementwise add |
| 2.81 s | 1.8 % | 604 | NCCL AllReduce bf16 |
| 2.40 s | 1.6 % | 1214 | direct copy |
| 2.07 s | 1.3 % | 900 | elementwise add（另一版） |
| 1.83 s | 1.2 % | 907 | `quantize_int8_rowwise_convrot64` |
| 0.99 s | 0.6 % | 324 | cudnn SDPA（参考图编码用） |
| 0.97 s | 0.6 % | 302 | `quantize_int8_rowwise_convrot64`（512 组） |
| 0.78 s | 0.5 % | 600 | `_indexed_gate_bf16` |
| 0.51 s | 0.3 % | 600 | `_rmsnorm_indexed_scale_shift` |
| 0.51 s | 0.3 % | 302 | `act_and_mul`（SwiGLU） |
| 0.38 s | 0.3 % | 300 | `fused_qknorm_rope` |

两个反直觉的点：

1. **attention 是唯一大头**：每层一次、50 层 × 6 步 = 300 次，单次 318 ms。序列是打包的 `[text | audio | video]`，158 帧在 768×1344 的画布上大约 2 万个视频 token（head_dim 128、每卡 28 头），attention 的代价是 token 数的平方。
2. **模型主体已经 INT8，但仍有 13.6% 的时间跑在 FP32 GEMM 上。** 量化只覆盖了 4 个主 GEMM，AdaLN 之类的小投影还在 FP32。这是一块独立的、确定性的收益（把 FP32 GEMM 换成 bf16 预计省 8–10%），但不在本文主线里。

---

## 二、第一次尝试：换 attention kernel，失败

### 2.1 为什么选它

attention 占 60%，最直觉的动作就是换更快的注意力内核。SageAttention 的思路是 QK 用 INT8、PV 用 FP16/FP8，在 A100 上只有 INT8-QK 这条路径可用（没有 FP8 张量核）。

### 2.2 安装过程（三次失败）

| 尝试 | 失败原因 | 修法 |
|---|---|---|
| 1 | 从源码直连克隆超时（约 40 KB/s） | 改为在本地下载 release 包再传上去；实测本机到源站 9.4 MB/s，传回推理机约 4.9 MB/s |
| 2 | 编译报错：仓库硬编码 `-std=c++17`，而当前 PyTorch 的头文件要求 C++20 | 用 `CXX_APPEND_FLAGS` / `NVCC_APPEND_FLAGS` 追加 `-std=c++20` |
| 3 | 报「CUDA compiler and CUDA toolkit headers are incompatible」 | 实测环境的 runtime 头是 CUDA **13.0**、nvcc 是 **13.4**（pip 安装的组件版本不一致）；用 CCCL 官方逃生口 `-DCCCL_DISABLE_CTK_COMPATIBILITY_CHECK` 放行 |

另外两个环境细节：只编目标架构（sm80）可以显著缩短编译时间；编译需要 `ninja`，它不在默认 PATH 里。

### 2.3 结果

| | FA2 | SageAttention | 差异 |
|---|---|---|---|
| attention kernel 总时间 | 96.41 s | 87.09 s | **−9.7 %** |
| 单次 attention | 319 ms | 288 ms | |
| denoise（GPU） | 150.3 s | 142.7 s | −5.0 % |
| 全流程 kernel | 160.4 s | 152.9 s | −4.7 % |
| 端到端 wall | 163.5 s | 160.2 s | 噪声内 |
| 其它 kernel | — | 基本不变 | FP32 GEMM 16.5 s、INT8 GEMM 15.8 s、NCCL 2.9 s |

A100 上的 INT8-QK 只值 9.7%，原因很朴素：head_dim 128、长序列下 FA2 本身效率已经不差，而 Sage 的量化本身有开销，PV 仍然是 FP16。

### 2.4 决定性的对照：不是快了，是变了

| 对照 | PSNR | SSIM(Y) |
|---|---|---|
| 同一条请求跑两次 FA | **inf（逐字节一致）** | 1.000 |
| FA ↔ SageAttention | **27.20 dB** | **0.838** |

而且逐帧看，**第一帧就已经差到 30 dB**，不是逐步累积——说明 INT8-QK 是每一步都存在的系统性偏移，被 6 步的大跨度更新放大。中间帧最差掉到 27.5 dB。

为 5% 的 denoise 收益丢掉确定性，不划算，回退。**教训：在已经蒸馏到 6 步的模型上，attention 的数值精度是画质账本的一部分，不是可以单独 A/B 的性能开关。**

---

## 三、真正的杠杆：双分辨率采样

### 3.1 一句话原理

MPEG-Diffusion（Mixed-Resolution Video Generation）把采样拆成两段：**先在半分辨率上把整帧采样到接近收敛，再只在需要细节的区域（人脸等）用全分辨率 token 续采样**。

### 3.2 结构

```
阶段 1（switch 之前）：整帧按半分辨率跑，token 只有 1/4

  ┌─ 中间层（block 25）的 LR 隐状态 → 一格 Linear → mask（区域）
  ▼
阶段 2：LR 的 clean 估计放大 2×、重新加噪，拆成两条轨道

  序列 = [ text | audio | LR 全网格 | HR 只在 mask 内的格子 ]
                   │              │
  LR query ─只看─► text/audio/LR keys
  HR query ─看──► text/audio + 一整张 HR 网格的 K/V
                          │            │
                      M 内=自己的 K/V  M 外= f_up(LR 的 K/V)
                                            ↑
                              每层一张 rank-128 线性映射（约 32 MB）
                              把 1 个 LR 的 K/V 展开成 2×2 四格
                              （去掉「复制 key」造成的菱形网格纹）
```

四个件：

1. **双轨序列** —— `[text, audio, LR 全网格, HR 只在 mask 内的格子]`。
2. **双分辨率 attention** —— LR query 只看 LR/text/audio（这部分就是「原生模型跑在低分辨率」，不需要任何额外权重）；HR query 看 text/audio + 一整张 HR 网格的 K/V，mask 内用 HR 自己的，mask 外用 `f_up` 展开的 LR 的。
3. **switch** —— 在某一步用中间层的隐状态跑一个线性 region head 得到 mask；把 LR 的 clean 估计放大 2 倍、**重新加一次噪声**，进入双轨阶段。
4. **收尾** —— LR 上采样、HR 按 mask 贴回（羽化）、VAE 解码。

关键洞察：**视频 DiT 的 attention 代价是 token 数的平方，而大多数像素并不需要全分辨率。** 把不重要的区域从「全分辨率」降到「四分之一 token」，省的是平方项。

### 3.3 参考实现的实测（作者那套独立渲染器）

用官方发布的权重（pruned int8 DiT 20.97 GB + fp16 VAE 5.21 GB + `f_up` 映射 32 MB + region head 22 KB），22 帧 1344×768、20 步、switch 16：

| 变体 | 时间（含 VAE 解码） | 平均 token | 区域占比 |
|---|---|---|---|
| 全分辨率基线 | 75.09 s | 7056 | — |
| two-track | **31.21 s** | 2446 | 35.4 % |

**2.41×**（作者自己报 1.94×）。这条独立栈能跑通，说明方法本身没问题——问题出在「换成我们这套栈 + 6 步」之后。

顺带记录下载侧的实测（国内网络下）：直连源站超时，镜像单流约 10 MB/s、8 并发 27 MB/s，用 `huggingface_hub` 实测 21 GB 用了 452 秒（**46 MB/s**）、5.2 GB 用了 234 秒（22 MB/s）。裸 `curl -r` 分块会被镜像 429 拒绝并拼出 0 字节文件，用 hub 客户端才稳。

### 3.4 和我们设定的关键差异：σ 对齐，不是步数对齐

同一套 shift=12 的 σ 路径，两边离散化密度不同：

**6 步（我们的蒸馏模型）**，knots 在 t 上等分：

| 步 | σ 区间 | 跨度 |
|---|---|---|
| 0 | 1.000 → 0.973 | 很小 |
| 1 | 0.973 → 0.923 | 小 |
| 2 | 0.923 → 0.800 | 中 |
| 3 | 0.800 → 0.571 | **大** |
| 4 | 0.571 → 0.333 | **大** |
| 5 | 0.333 → 0.000 | **大** |

**20 步（原方法）**：σ = 1.0 → 0.9956 → 0.9908 → 0.9855 → 0.9796 → 0.973 → 0.9655 → 0.9571 → 0.9474 → 0.9362 → 0.9231 → 0.9076 → 0.8889 → 0.8660 → 0.8372 → 0.800 → **0.750（switch）** → 0.6792 → 0.5714 → 0.3871 → 0。

同一条路径，20 步是 6 步的 3.3 倍细。而 switch 在 **σ=0.75**，正好落在我们第 3→4 步之间 → **6 步版本 = 3 步 LR + 3 步双轨**。（一开始我按步数比例估成「5 LR + 1 HR」，那是错的；必须按 σ 对齐。）

---

## 四、移植：四步走，每步一个验收门

目标框架的实现有一个很好的接缝：`Attention._attend()` 拿到的是**归一化之前、RoPE 之前**的 q/k/v——正是双分辨率 attention 需要的原料。

另外框架里有个环境变量可以关掉融合的 `qknorm+rope` 内核、走可拦截路径。**先验证这个开关「输出逐字节一致」**，等于拿到接管 norm+RoPE 的许可，才开始往上叠。

### 4.1 第 1 步：attention 数学（parity 测试）

新写一个模块，四个件：

| 件 | 作用 |
|---|---|
| `TwoTrackPlan` | mask → `hr_index` / `cell_order`（token-major ↔ HR 栅格的置换） |
| `UpMaps` | 加载 `f_up`（50 层、rank 128），`expand()` 把 K/V 展开成 2×2 格 |
| `TwoTrackContext` | 一次前向所需的行区间、HR 网格位置、映射、层号、真实行数 |
| `two_track_attend()` | LR/文本/音频走原路径；HR 查询对 `[text, audio, HR 网格]` 单独做一次 attention |

**验收方式是纯 tensor 的 parity 测试**（随机权重、随机输入、几秒钟跑完）：

| 用例 | 期望 | 实测 |
|---|---|---|
| A 全 mask + copy 映射 | 与全分辨率 attention 逐位相等 | max\|diff\| = **0.0** |
| B 全 mask + 零 `f_up` | 同上（验证映射路径的接线） | **0.0** |
| C 部分 mask | 未选中格回落到 LR 的 K/V | **0.0** |
| D LR/文本/音频行 | 不受影响 | **0.0** |
| E 部分 mask + 非零 `f_up` | 与参考实现等价（验证 head/cell 顺序） | **0.0** |

### 4.2 第 2 步：布局（从已有 packed 布局派生）

不新造打包器，而是**从已有的 packed 布局派生**：

- 前缀（text / 参考图块 / audio）**原样保留**，只把视频段换成「LR 网格 + mask 内的 HR 格」；
- LR 与 HR 的位置都用框架自己的 `_video_t_grid` / `_axis_from_sqrt_area` 生成——H3 的 RoPE 按 `sqrt(area)` 归一，两个分辨率天然共用一个坐标系；
- 视频行必须是被打包序列的**最后一段**（代码里有断言）。

26 项不变量测试（plain 与「带参考块」两种布局各 13 项），覆盖：

| 检查 | 结果 |
|---|---|
| LR token 数 = `latent_t × (h/4) × (w/4)` | PASS |
| 全 mask 时 HR = 4 × LR | PASS |
| `used = 前缀 + LR + HR`、`seq_len` 64 对齐 | PASS |
| LR 行位置 = 半分辨率网格 | PASS |
| HR 网格位置 = 全分辨率网格 | PASS |
| text/audio 行位置与原生一致 | PASS |
| `token_tags`：视频 0、padding −1 | PASS |
| 部分 mask 的 HR 行 = 命中格、按栅格序（例如 `[150, 151, 162, 163]`） | PASS |
| HR 首格位置与 `(t, u, v)` 对应 | PASS |

`[150, 151, 162, 163]` 这种结果正是容易写错的地方——一个 LR token 的 2×2 格在栅格序里不连续，现在有测试锁住了。

### 4.3 第 3 步：采样循环

| 阶段 | 做什么 |
|---|---|
| 阶段一 | 半分辨率、`switch` 步；用框架自己的 `DenoiseBranch` 管静态布局与缓冲区；在最后一步抓「更新前」的状态与速度场 |
| switch | 算 mask（第一版是固定框，后接 region head）；LR 的 clean 估计 ×2；`(1-σ)·start + σ·noise` |
| 阶段二 | 装 context，跑剩下几步；两条轨道（LR 行 + HR 行）同步更新 |
| 收尾 | HR 写回全网格 → LR 上采样 → `inside·hr + (1-inside)·lr_up` → 交回框架解码 |

### 4.4 第 4 步：region head

在 block 25 挂一个抓取钩子（注意：**阶段一没有装双轨 context，所以光在双轨 attention 里抓是抓不到的**，必须在分发钩子的最前面独立抓），拿到 LR 阶段最后一个视频隐状态，跑「无仿射 LayerNorm + 一格 Linear」得到每 token 分数，按论文规则生成 mask：

```
smooth = 3×3 空间平均 + 时间轴 [1,2,1]/4
mask   = s > median(s) + α · (p99(s) − median(s))
```

实测 α = 0.5 时选中 15.7 %（偏小），α = 0.35 时选中 **24.6 %**。region head 自带的元数据是 `block=25, hidden=5376`，权重只有 5376 个数（21 KB）。

---

## 五、移植过程中九个会静默出错的坑

按「症状 → 根因 → 修法」记，每一个都花了至少一次重启。

| # | 症状 | 根因 | 修法 |
|---|---|---|---|
| 1 | 全 mask 正确、部分 mask 全错 | 网格先把 `index_copy` 再 `index_select`，顺序反了；全覆盖时因为全部被覆盖而看不出 | 先排成栅格（`index_select(cell_order)`），再覆盖选中格（`index_copy(hr_index)`） |
| 2 | 同上，且 head 与格出现串位 | copy 分支 `expand` 后直接 reshape，布局成了 token→head→cell | `expand` 后 `transpose(1,2)`，保证 token→cell→head |
| 3 | HR 网格的 K 是「已旋转」的，结果离谱 | `_attend` 的 RoPE 是**就地**写入，LR 那一支已经把 K 旋掉了 | 在 LR 分支之前 clone 出「旋转前」的 LR 视频 K |
| 4 | `mask [17984] 与 tensor [9324] 不匹配` | 条件行有两个视图：`img_pos`（打包序列行号）与 `update_mask`（视频行序号，条件行在前） | 布局里两套都按各自语义生成：`img_pos = [条件行行号..., 目标行行号...]`，`update_mask` 长度 = 视频行数 |
| 5 | `index_copy: indices 13024 vs source 13036` | 行区间用了 `seq_len`（64 对齐后）而不是 `used`（真实行数），多出 **12 行 padding** 混进 HR 分支 | 所有行区间用 `used` 截断，padding 行输出置零 |
| 6 | **illegal memory access**，两个 rank 一起挂 | 前缀那次 attention 的 `cu_seqlens` 用整条序列的 `[0, used, seq_len]` 改了末位，varlen 内核读到不存在的行 | 前缀单独传 `[0, stop, stop]` |
| 7 | 蒙版区域解码成**一块均匀的灰**，且不崩 | HR 网格用了**序列的** RoPE 表去索引网格位置（那张表是「每行对应的旋转」，不是「按位置查表」），网格 key 变 NaN（实测 6.4 万个 NaN），NaN 一路传到 VAE | 用网格自己的 `build_rope_cache`，把返回的 `(cos_sin, positions)` 成对存进 context |
| 8 | HR 轨道从第一步就是错的 | `x0 = x + σ·v` 里用了**更新后**的状态；蒸馏采样是 renoise 的，循环走完一步状态已被重写 | 在 `model_forward` 里抓「更新前」的 x 与同一时刻的 v |
| 9 | 蒙版外是噪声、蒙版内是糊的 | 收尾用 switch 时刻的 LR 估计（半成品），而不是阶段二结束后的 LR | 收尾从阶段二的返回里取 LR 行重新反打包 |

另外三个小坑：

- **`--prompt` 是贪婪参数**（`nargs="+"`），放在命令行中间会吃掉后面的 flag；
- **dataclass 字段顺序**：给有默认值的字段插在无默认值字段前面，会直接 `TypeError: non-default argument follows default argument`；
- **offline 路径缺 `SGLANG_JIT_CACHE_DIR` 会触发重新编译**，而编译需要 `ninja`，不在 PATH 里就静默退出（worker exit code 1）。

---

## 六、实测：2.04×，以及为什么这就是天花板

### 6.1 结果

| 口径 | 基线 | box mask | region head |
|---|---|---|---|
| wall | 163.5 s | 85.2 s（1.92×） | 80.2 s（**2.04×**） |
| denoise（stage） | 139.5 s | 69.3 s | — |
| denoise（循环内计时） | — | 65.2 s | 59.9 s（**2.33×**） |
| LR 单步 | 25.0 s | 6.9 s（3.6×） | 6.9 s |
| HR 单步 | — | 14.8 s | 13.0 s（1.9×） |
| 区域占比 | — | 35.7 % | 24.6 % |
| PSNR / SSIM(Y) | — | 22.1 dB / 0.712 | 21.9 dB / 0.709 |

拆分（region head 配置）：LR 3 步 20.8 s → switch 0.0 s → HR 3 步 39.1 s → 收尾 <0.1 s。

### 6.2 天花板推导

- LR 步的 token 是 1/4，attention 是平方项 → 理论 4×，实测 **3.6×**，基本吃满（剩下的固定开销是 norm/MLP/通信）。
- HR 步的 token 是 `(LR + 区域内 HR) / 全帧 ≈ 0.61`。但 attention 只占一步的 64.6%，剩下 35%（MLP/GEMM/量化/elementwise/NCCL）**按行数线性走**，所以 HR 步的地板 ≈ `0.61 × 25 ≈ 15.3 s`。实测 14.8 s，已经贴地。
- 6 步的总账：`(3 × 0.25 + 3 × 0.61) / 6 ≈ 0.43` → 上限 **2.33×**，实测 2.33×。

**顺带一个对照**：原作者 20 步设定报 1.94×，我们 2.04×。不是方法变好了，而是 6 步里 LR 占了 50% 的步数（20 步里 LR 占 16 步但基数大、HR 又要 4–6 步）。**倍率高低取决于「LR 步的占比」，不取决于实现优劣。**

### 6.3 数据落盘的形态

每次实验都写一份 JSON，字段固定，便于以后做对照表：

```json
{
 "variant": "region_head_v3+maps",
 "config": {"switch_sigma": 0.75, "switch_step": 3, "lr_steps": 3, "hr_steps": 3,
            "maps": true, "region_head": "region_head_v3", "alpha": 0.35, "region_share": 0.246},
 "wall_s": 80.2, "denoise_stage_s": null, "loop_total_s": 59.91,
 "breakdown_s": {"lr_steps": 20.8, "switch": 0.0, "hr_steps": 39.11, "compose": 0.0},
 "per_step_s": {"lr": 6.93, "hr": 13.04},
 "quality": {"psnr_y_db": 21.87, "ssim_y": 0.7091, "frames": 158,
             "verdict": "usable pipeline, below baseline"},
 "perf_dump": "<perf dump 路径>", "output": "<输出 mp4 路径>"
}
```

---

## 七、为什么这个方式在我们的设定下不行

三层原因，从重到轻。

### 7.1 6 步蒸馏没有余量（结构性）

two-track 的前提是「LR 阶段把构图和整体质量做出来」。20 步时 LR 有 16 步；我们只有 3 步，而且每步的 σ 跨度巨大（0.8 → 0.571 → 0.333 → 0）。

结果是：LR 在第 3 步交出来的 clean 估计还是个**80% 噪声的粗坯**，把它放大 2 倍当 HR 的起点，等于用半成品当先验；HR 只有 3 步，来不及重画。表现就是我们看到的——区域连贯但发虚。

（试过把 switch 提前到 σ=0.973 给 HR 5 步：耗时升到 100 秒，画质没有本质改善——因为问题不只在步数，也在下面两层。）

### 7.2 关键的 fine-tune 权重没公开（权重层）

作者在模型卡里写得很清楚：`f_up` 映射只解决「复制 key 的网格纹」；**让全分辨率轨道提前介入的是那个 rank-16 LoRA，而它没有发布**。

没有它，模型推理时看到「同一条序列里同时存在两种分辨率 token」是训练分布外的输入，两条轨道的速度场不一致，HR 区域会向条件均值收敛——也就是发虚、偏灰，而不是出细节。只用 maps 等于用了半套方法。

### 7.3 只移植了 attention，没移植配套件（移植层）

- 起点用 latent 的三线性 ×2，不是论文的 `U`（VAE 解码 → 双三次放大 → 再编码）。插值**不可能造出高频**，起点就注定是软的。
- `f_down`、学习式上采样 `U`、混合器 `G`（论文 eqs. 5–6）都没移植，收尾是贴回 + 羽化。
- maps 是在 base 模型上拟合的，我们跑的是叠了 LoRA 的蒸馏模型，映射本身有偏差。

### 7.4 一个可以复用的判断

如果团队的模型是 **20 步级别**、并且愿意移植论文的完整配套件 → 这条路值得投入。
如果是 **6 步蒸馏 + 拿不到原作者权重** → 速度收益能拿到（2 倍是真的），但画质过不了自己那关；这时候更该去优化「每步的固定开销」（那 35–40% 的线性项：FP32 GEMM、量化、通信），而不是继续削减 token。

---

## 八、另外交付了一件东西：不依赖服务的 runner

整个链路平时挂在推理服务里，但做算法实验时不希望每次起服务。于是加了一个离线入口：

```
python3 -I two_track_runner.py req.json --out runs/offline \
  --maps up_maps_tt.safetensors --region region_head_tt.safetensors --alpha 0.35
```

它把请求 JSON 拆成 `sglang generate --config` + `--prompt`，设置开关注入后**进程内**跑完整条 pipeline（不监听端口），直接出 mp4。实测：total 75.1 秒、denoise 64.1 秒、输出 768×1344 / 158 帧。

这条路径上踩到的三个坑，都是「服务模式下永远不会遇到、换到离线就必炸」的：

1. **不传组件级量化就会 OOM**：全精度 DiT 62 GB + 文本编码器 65 GB，两张 80 GB 卡装不下。必须按组件量化到 INT8。
2. **`PATH` 里没有 `ninja` 和 CUDA 的 `bin`**：INT8 内核要 JIT 编译，直接 `FileNotFoundError: ninja`，worker 退出码 1。
3. **`--output-dir` 不存在**，正确的是 `--output-path` + `--output-file-name`；且 `--prompt` 必须放在最后。

---

## 九、可复用的三条工程习惯

1. **字节一致的回退门。** 每一步都要求「不设开关时，输出与基线逐字节一致」。这个模型是确定性的（同请求两次运行 PSNR = inf），所以这条门既强又可执行——它抓住了所有「不小心动了默认路径」的改动。整个迁移过程中，这条门一次都没被打破。
2. **parity 测试用随机张量、不用真模型。** 双分辨率 attention 的正确性，可以用构造的小张量在几秒内验证（全覆盖 + 恒等映射必须逐位相等；部分覆盖必须等于「未选中格回落到 LR 拷贝」），比跑一次真实推理快两个数量级。九个坑里有三个是它抓出来的。
3. **先量化再优化。** 如果先动手改内核，大概率会去优化那 13.6% 的 FP32 GEMM，而真正的 60% 在 attention 上——那个 60% 不是换 kernel 能解决的，是计算方法的问题。

---

## 十、结论

- **收益是真的**：同一套框架、同一批权重，双分辨率采样把 denoise 从 139.5 秒压到 59.9 秒（2.33×），端到端 2.04×，且已贴着该配置的理论上限。
- **代价也是真的**：6 步蒸馏拿不到足够步数，唯一能补的 fine-tune 权重未公开，配套的上采样器/混合器也需要额外移植。
- **方法本身没错**：同一批公开发布权重，在 20 步的独立渲染栈上实测 2.41×，画面可用。差异来自「步预算 + 权重 + 配套件」，不是来自算法。
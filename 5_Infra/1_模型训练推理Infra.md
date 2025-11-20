# 模型训练
- 大模型训练框架包括 deepspeed 和 megatron-llm
    - deepspeed 如何解决大模型训练的问题？详细可参考 [这里](https://zhangzhe.space/2024/08/21/ZeRO-Memory-Optimizations-Toward-Training-Trillion-Parameter-Models/)，简单一句话：数据并行（DP）下的优化器状态 / 梯度 / 参数分片，每个设备只存储全部的 1 / n（n 表示设备数），这样一个很大的模型就被切分到了不同的设备上，需要使用全部信息的时候就通过通信 reduce 聚合
    - megatron 靠什么加速？张量并行（TP）/ 流水线并行（PP）/ 专家并行（EP）/ 序列并行（SP）/ 上下文并行（CP）等的组合，将一个非常大的模型用各种方式正交的拆到不同的设备上，尽量做到计算和通信的 overlap，理论的训练速度上限更高
- 对于 2 - 10B 的 3D 生成大模型训练，最优方案是使用 DeepSpeed 训练框架做分布式训练，主要原因如下：
    - 工程复杂度低，deepspeed 的设计哲学之一就是从 pytorch 的迁移成本极低
    - 在 diffusion 领域，最具影响力的开源模型代码库是 huggingface 的 [diffusers](https://github.com/huggingface/diffusers)，diffusers 生态完全兼容 deepspeed；而 megatron 使用较为复杂，需要手写 tensor parallel，可能出现 2D/3D attention reshape 导致 Megatron TP 崩溃
    - 在 DiT 训练过程中，通常来说 activation 会非常占显存，deepspeed 的 zero + activation checkpoint 机制几乎是 diffusion 模型训练的标配，非常适合 2 - 10B 参数的模型
- 对于搜索 / 推荐 / 2D 等小模型的训练，如果需要分布式，也可以使用 deepspeed 或 megatron，通常 deepspeed 是以 python 安装包的形式使用，megatron 是通过容器使用
- deepspeed 结合 nvidia h100 可实现 fp8 低精度训练
# 模型推理
- 对于 GPT 类的 decoder only 的纯语言模型（搜索 / 推荐场景可能用这个），使用 vllm / sglang 推理框架通常可以得到较高的速度和较低的显存占用
    - vllm 靠什么加速？
        - PagedAttention：将 KV Cache 像“操作系统分页”一样管理
        - Continuous batching：普通服务器一次只能处理固定 batch，sequence 结束后 batch 会“空洞”。vLLM 动态把新请求插入正在运行的 batch GPU 每个 step 都填满，高并发 throughput 提升巨大（>10x 数量级提升）
        - 高度并行的 CUDA Kernel：底层算子 / 图优化
        - Zero-copy 体系结构：Python 端和 C++/CUDA 端共享张量，不做 copy，降低开销
    - sglang 靠什么加速？
        - RadixAttention（树注意力）：将 kv cache 树形结构存储，对于多分支推理（比如 ReAct、RAG）性能极强
        - 高效的 Context Caching：Prompt 共享（multi-turn cache）；Prefix caching（对 RAG/Chat 类任务极其重要）;复杂的 Branching decode 可以共享前缀 KV，不重复计算
        - 更轻量的 Runtime：rust 语言实现调度器，比 vllm 的 python 快
- 由上面可知目前主流的大模型推理框架仅仅用于语言模型，对于 diffusion 模型并不适用（没有 kv cache），因此对于 diffusion 模型，通常有两种方案
    - TensorRT / TensorRT-LLM ：nvidia 官方提供的工具链和推理引擎，工具链做了量化 / 图优化等，支持 pytorch / onnx 格式输入，可做到极致的性能
    - CUDA / Triton kernel：自己手写 CUDA 或 Triton（CUDA 的 python 封装 API），除非遇到非常定制化且 TRT 支持不好才需要
# 性能分析
Profiling 过程的目标是明确 3 个核心问题：
- 瓶颈在哪里？
    - GPU 利用率低？
    - Memory Bandwidth 打满？
    - Kernel 太多/太碎？
    - CPU 成为瓶颈？
    - 通信（NCCL）拖慢训练？

- 瓶颈的原因是什么？
    - 算子实现？
    - Batch size 太小？
    - DataLoader 过慢？
    - 参数同步不均匀？

- 优化是否有效？
    - 调整 Batch、TF32、BF16、FlashAttention、KV cache 是否提升 GPU 执行效率。
## NVIDIA GPU Profiling 的工具选择
### Nsight Systems / Nsight Compute（最权威）
- Nsight Systems (nsys)
    - 解决：
        - GPU 利用率低？
        - CPU/GPU pipeline 不平衡？
        - DataLoader 是否拖慢？
        - NCCL 通信等待？
    - 优势：全局视角（timeline），剖析端到端瓶颈

- Nsight Compute (ncu)
    - 解决：
        - 单个 Kernel 性能是否 optimal？
        - TensorCore 是否使用？
        - Memory bandwidth 是否打满？
        - Warp divergence 是否严重？
    - 优势：针对某个 kernel 深度分析
### PyTorch Profiler（最灵活）
- 解决：
    - 哪个 PyTorch operator 慢？
    - op → kernel 映射关系
    - FLOPS、TFLOPS 利用率
    - GPU 计算 vs CPU 调度
- 优势：直接集成 Torch + Tensorboard
### NVIDIA DCGM / nvtop（简单监控）
- 查看 GPU：
    - 显存占用
    - SM 利用率
    - PCIe 读写
    - 温度功耗
- 用于快速排查基本问题。

### NCCL debug 工具
- 用于分布式训练瓶颈分析。
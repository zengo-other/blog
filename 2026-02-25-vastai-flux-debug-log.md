# Vast.ai 上调试 FLUX.1-schnell + LoRA 的踩坑实录（2026-02-25）

今天把 `runpod-test` 这个镜像在 Vast.ai 上做了一轮完整实测，目标很简单：

- 跑通 `black-forest-labs/FLUX.1-schnell`
- 加载 `Heartsync/Flux-NSFW-uncensored` LoRA
- 生成 3 张图
- 跑完立刻停机，控制成本

结果是能跑通，但中间踩了不少坑。这里把关键故障、排查思路和最终可复用配置记录下来，给后面自己省时间。

---

## 1. 一开始的问题：实例在跑，但任务没跑

第一波误判是把 Vast.ai 的 `instance running` 当成了“脚本在执行”。

后面固定用三重确认：

```bash
# 1) 进程是否存在
ps -ef | grep run_real_test.py | grep -v grep

# 2) 日志是否前进
tail -n 100 /app/run_real_test.log

# 3) 产物是否增长
ls -1 /app/out/vast_real_*.png 2>/dev/null | wc -l
```

这个比看面板状态靠谱得多。

---

## 2. 关键坑位汇总

### 坑 A：模型 gated / token 问题

`FLUX.1-schnell` 是 gated 模型，没 token 直接 401。

症状：

- `Cannot access gated repo`
- `401 Unauthorized`

处理：

- 明确在运行命令前注入 `HF_TOKEN`
- 不依赖“容器环境可能自动传进去”，直接命令行显式传值

---

### 坑 B：磁盘不够（首次下载模型）

初期给了 20~25GB 磁盘，下载到一半直接报空间不足。

症状：

- `Not enough free disk space`
- 下载中断，后续加载失败

处理：

- FLUX + 相关权重缓存建议直接给 **80GB+**（实测 100/120GB更稳）

---

### 坑 C：依赖缺失

镜像里并不总是有运行 FLUX/LoRA 需要的所有依赖。

遇到过：

- `sentencepiece` / `protobuf` 缺失（tokenizer 初始化失败）
- `PEFT backend is required`（LoRA 加载失败）

处理：

```bash
pip install --no-cache-dir sentencepiece protobuf peft
```

---

### 坑 D：GPU 架构兼容性（5060 Ti + 旧 torch）

试过 5060 Ti（sm_120），但镜像里 `torch==2.4.0`，编译能力不匹配。

症状：

- `sm_120 is not compatible with current PyTorch installation`

处理：

- 换回兼容性稳定的卡（如 RTX 3090）
- 或升级到支持该架构的 torch/cuda 组合

---

### 坑 E：24GB 显存 OOM

即使用 3090（24GB），直接 `pipe.to("cuda")` 也会 OOM。

症状：

- `torch.OutOfMemoryError`（加载/推理阶段显存爆）

处理（最关键）：

把 `handler.py` 里加载后放置设备的逻辑改为 CPU offload：

```python
if torch.cuda.is_available():
    pipe.enable_model_cpu_offload()
else:
    pipe.to("cpu")
```

这一步后，24GB 卡可跑通（速度会慢一点，但成本更可控）。

---

## 3. 最终跑通配置（可复用）

- **GPU**：RTX 3090（24GB）
- **磁盘**：120GB
- **依赖**：`sentencepiece protobuf peft`
- **策略**：`enable_model_cpu_offload()`
- **参数**：`512x512`, `steps=8`, `guidance=3.5`, `num_images=1`（循环三次）
- **结果**：成功生成 3 张图（`vast_real_1/2/3.png`）
- **收尾**：生成后销毁实例，停止计费

---

## 4. 成本和效率经验

1. **先做资源估算再租机**：
   - 只看“便宜卡”容易在磁盘、显存、架构上反复踩坑，反而更贵。
2. **先跑 smoke，再跑真实模型**：
   - 先验证链路，再烧下载和推理成本。
3. **监控不要只看面板**：
   - 一定看 `ps + log + out` 三件套。
4. **跑完立刻停机**：
   - 生成完马上 `destroy instance`，这是最直接的省钱动作。

---

## 5. 下一步优化（TODO）

- 把依赖写进镜像，避免每次手动 `pip install`
- 在 `handler.py` 中增加显存自适应策略（自动选择 offload）
- 增加结构化日志，便于远程监控和失败重试
- 把 “进程/日志/产物” 做成统一健康检查脚本

---

这次最大的收获不是“终于跑通了”，而是把**失败路径**走完整了：
从 token、磁盘、依赖、架构到显存，后续再部署同类模型会快很多。
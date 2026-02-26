# XTTS-v2 本地试跑记录（macOS, 2026-02-26）

今天把 `coqui/XTTS-v2` 在本地完整跑通，目标是：

- 本地加载模型
- 中文文本 + 参考音色合成语音
- 记录踩坑与稳定方案

对应代码仓库：
- https://github.com/zengo-other/xtts-v2-local-runner

---

## 结论先说

**能在本地跑通。**

最终成功产物：
- `out/xtts_v2_zh_test.wav`

但首次跑需要处理 3 个典型兼容坑。

---

## 环境

- 机器：macOS (Apple Silicon)
- Python：3.11（用 `uv` 创建）
- 主包：`TTS==0.22.x`

---

## 踩坑与修复

### 1) 首次加载模型卡在 TOS 交互

XTTS-v2 首次下载会要求确认 Coqui 许可（y/n）。

- 现象：脚本在 `ask_tos()` 处报 EOF 或阻塞
- 处理：手动/管道输入 `y`

---

### 2) transformers 版本不兼容

默认安装链路里可能拉到 `transformers 5.x`，与当前 TTS/XTTS 代码不兼容。

- 现象：`ImportError`（`stream_generator` 等）
- 处理：锁版本

```bash
transformers<5
tokenizers<0.21
```

---

### 3) PyTorch 2.6+ 的 weights_only 默认行为

`torch.load` 默认 `weights_only=True` 导致 checkpoint 读取失败。

- 现象：`_pickle.UnpicklingError: Weights only load failed`
- 处理：设置环境变量

```bash
export TORCH_FORCE_NO_WEIGHTS_ONLY_LOAD=1
```

---

### 4) 缺少 torchcodec

- 现象：`TorchCodec is required for load_with_torchcodec`
- 处理：

```bash
pip install torchcodec
```

---

## 最小可复现步骤

```bash
uv venv --python 3.11 .venv
source .venv/bin/activate
uv pip install "TTS>=0.22,<0.23" torchcodec "transformers<5" "tokenizers<0.21"

# 首次模型许可确认需要 y
export TORCH_FORCE_NO_WEIGHTS_ONLY_LOAD=1
python run_xtts_v2_zh.py \
  --text "同学们，今天我们来学习平均分。把十二个苹果平均分给三个小朋友，每人分到四个。" \
  --speaker-wav ./samples/speaker.wav \
  --out ./out/xtts_v2_zh_test.wav
```

---

## 经验总结

1. AI 语音模型本地部署，**先锁依赖版本**，再跑业务逻辑。
2. 首次下载/许可交互一定要纳入自动化脚本设计（至少给明确提示）。
3. 遇到报错优先看“默认行为变化”（例如 torch 2.6 的 `weights_only`）。
4. 跑通后把流程沉淀为仓库，后续复用成本会非常低。

---

如果后续要做成“批量中文配音”，下一步可以加：
- 多段文本批处理
- 多参考音色切换
- 输出格式与采样率统一（16k/24k）
- 自动降噪与响度归一化。
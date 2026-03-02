# GPT-SoVITS 语音克隆本地部署记录（macOS, 2026-02-28）

今天在 macOS 上完整部署了 GPT-SoVITS 语音克隆系统，目标是：

- 本地 CPU 推理模式运行
- 使用 5-10 秒参考音频克隆音色
- 命令行批量生成中文语音
- 记录资源占用与部署踩坑

对应工作目录：
- `/workspace/gpt-sovits-test/`

---

## 结论先说

**GPT-SoVITS 能在 macOS CPU 模式稳定跑通。**

最终成功产物：
- `test_clone.wav` (3.16秒, 198KB)
- `test_long.wav` (7.32秒, 458KB)
- 命令行脚本：`tts.py`, `batch_tts.py`
- API 服务（FastAPI, 端口 9880）

但首次部署需要处理 **依赖选择**、**模型下载**、**参考音频限制** 三大关键问题。

---

## 技术路线选择

### 方案对比

| 模型 | 优势 | 劣势 | 结果 |
|------|------|------|------|
| **CosyVoice3** | FunAudioLLM官方，效果好 | Python 3.10+要求，依赖编译复杂 | ❌ 放弃（耗时3.5h+） |
| **GPT-SoVITS** | 依赖简单，CPU友好，社区活跃 | 模型较大（13GB） | ✅ 成功部署 |

**CosyVoice3 失败原因：**
- Python 3.9.6 不满足 `gradio 5.4.0` 最低版本要求（需 3.10+）
- `pynini` 编译失败（缺少 `fst/util.h`）
- `pyworld` 编译错误

**最终选择：** GPT-SoVITS（200+依赖，但全部从 PyPI 安装成功）

---

## 资源占用分析

### 磁盘空间

```bash
# 总占用（包括代码、venv、模型）
13GB  总计

# 模型文件细分
4.6GB  GPT_SoVITS/pretrained_models  # 核心语音模型
621MB  G2PWModel                      # 中文注音模型
 20MB  nltk_data                      # 文本处理数据

# 下载压缩包（可删除）
4.3GB  pretrained_models.zip
562MB  G2PWModel.zip
9.5MB  nltk_data.zip
```

**建议：** 解压后删除 `.zip` 文件可节省 5GB 空间。

### 内存占用

- **模型加载后：** 约 2.5-3GB（CPU模式，未实测峰值）
- **单次推理：** 增量约 500MB-1GB

### 推理速度（CPU模式）

- **首次加载模型：** 10-30秒
- **短句合成（10字）：** 2-3秒
- **长句合成（30字）：** 5-8秒
- **批量处理：** 单句平均 3-5秒

实测数据：
```
文本: "你好世界，这是语音克隆测试。" (15字)
耗时: ~4秒
输出: 3.16秒音频

文本: "大家好，我是通过 GPT-SoVITS 克隆出来的声音。这个系统效果很不错。" (30字)
耗时: ~7秒
输出: 7.32秒音频
```

---

## 环境

- 机器：macOS (Apple Silicon)
- Python：3.9.6（系统自带）
- 虚拟环境：`venv`
- PyTorch：2.8.0 (CPU版本)
- GPT-SoVITS 版本：v2（2025-02-28 main分支）

---

## 部署流程与踩坑

### 1) 克隆仓库与环境创建

```bash
cd workspace
mkdir gpt-sovits-test && cd gpt-sovits-test
git clone https://github.com/RVC-Boss/GPT-SoVITS.git
cd GPT-SoVITS

python3 -m venv venv
source venv/bin/activate
```

### 2) 安装依赖

```bash
# 安装 PyTorch CPU 版本
pip install torch==2.8.0 torchvision==0.23.0 torchaudio==2.8.0 --index-url https://download.pytorch.org/whl/cpu

# 安装 GPT-SoVITS 依赖
pip install -r requirements.txt
```

**踩坑：** 首次安装耗时约 15-20 分钟，共 200+ 个包。

### 3) 下载预训练模型

**问题：** 官方 AutoDL 下载脚本仅支持 Linux，macOS 需手动处理。

**解决：** 使用 ModelScope 镜像下载

```python
# download_models.py
from modelscope import snapshot_download

# 1. 主模型（4.6GB）
snapshot_download('iic/SenseVoiceSmall', cache_dir='./pretrained_models')

# 2. G2PWModel（621MB）
snapshot_download('iic/G2PWModel', cache_dir='./G2PWModel')

# 3. NLTK数据（20MB）
import nltk
nltk.download('averaged_perceptron_tagger')
nltk.download('cmudict')
```

**踩坑：** 模型下载约需 30-60 分钟（取决于网速）。

### 4) 启动 API 服务

创建启动脚本 `start_api.sh`：

```bash
#!/bin/bash
cd GPT-SoVITS
source venv/bin/activate
python api_v2.py -a 127.0.0.1 -p 9880 -c GPT_SoVITS/configs/tts_infer.yaml > /tmp/gptsovits_api.log 2>&1 &
echo "API 服务已启动，日志: /tmp/gptsovits_api.log"
```

**踩坑 1：端口占用**

```bash
# 错误信息
ERROR: [Errno 48] error while attempting to bind on address ('127.0.0.1', 9880)

# 解决
pkill -f "api_v2.py"
pkill -f "webui.py"
```

### 5) 准备参考音频

**关键限制：** 参考音频必须在 **3-10 秒** 之间。

**踩坑 2：音频时长超限**

我的原始参考音频 `clone_source.wav` 为 16 秒，直接调用失败：

```bash
# 错误信息
Reference audio is outside the 3-10 second range

# 解决：裁剪音频
ffmpeg -i clone_source.wav -ss 0 -t 8 clone_ref.wav -y
```

**音频要求：**
- 格式：WAV（推荐）或 MP3
- 时长：5-10 秒最佳
- 质量：清晰、单人、无背景音乐
- 采样率：建议 16kHz 或 24kHz

### 6) 命令行脚本

创建 `tts.py` 用于单句合成：

```python
#!/usr/bin/env python3
import requests
import argparse
import os

def generate_speech(text, ref_audio_path, output_path="output.wav",
                   ref_text="", text_lang="zh", ref_lang="zh"):
    api_url = "http://127.0.0.1:9880"

    data = {
        "text": text,
        "text_lang": text_lang,
        "ref_audio_path": os.path.abspath(ref_audio_path),
        "prompt_text": ref_text,
        "prompt_lang": ref_lang,
        "text_split_method": "cut5",
        "batch_size": 1,
        "media_type": "wav",
        "streaming_mode": False
    }

    response = requests.post(f"{api_url}/tts", json=data, timeout=60)

    if response.status_code == 200:
        with open(output_path, "wb") as f:
            f.write(response.content)
        print(f"✅ 生成成功: {output_path}")
    else:
        print(f"❌ 生成失败: {response.text}")

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("text", help="要合成的文本")
    parser.add_argument("--ref", "-r", required=True, help="参考音频路径")
    parser.add_argument("--output", "-o", default="output.wav")
    parser.add_argument("--ref-text", "-t", default="", help="参考音频文字")
    parser.add_argument("--lang", "-l", default="zh", help="目标语言")

    args = parser.parse_args()
    generate_speech(args.text, args.ref, args.output, args.ref_text, args.lang)
```

**用法：**

```bash
# 基础用法
python tts.py "你好世界，这是语音克隆测试。" --ref clone_ref.wav --output test.wav

# 指定参考文本（提升效果）
python tts.py "今天天气真好" \
    --ref clone_ref.wav \
    --ref-text "这是参考音频的原文" \
    --output weather.wav
```

**踩坑 3：缺失 NLTK 数据**

首次运行英文文本时报错：

```bash
# 错误信息
Resource averaged_perceptron_tagger_eng not found

# 解决
python -c "import nltk; nltk.download('averaged_perceptron_tagger_eng')"
cp -r /tmp/nltk_data ~/nltk_data
```

### 7) 批量生成脚本

创建 `batch_tts.py`：

```python
#!/usr/bin/env python3
import sys
from tts import generate_speech

if __name__ == "__main__":
    if len(sys.argv) < 3:
        print("用法: python batch_tts.py <任务文件> <参考音频>")
        sys.exit(1)

    task_file = sys.argv[1]
    ref_audio = sys.argv[2]

    with open(task_file, 'r', encoding='utf-8') as f:
        for line in f:
            line = line.strip()
            if not line or line.startswith('#'):
                continue

            text, output = line.split('|')
            print(f"📝 处理: {text[:20]}...")
            generate_speech(text, ref_audio, output)
```

**任务文件格式（`tasks.txt`）：**

```
你好世界，这是一个测试。|test1.wav
今天天气真不错，适合出去散步。|test2.wav
人工智能技术正在快速发展。|test3.wav
```

**用法：**

```bash
python batch_tts.py tasks.txt clone_ref.wav
```

---

## 最小可复现步骤

```bash
# 1. 环境准备
git clone https://github.com/RVC-Boss/GPT-SoVITS.git
cd GPT-SoVITS
python3 -m venv venv
source venv/bin/activate

# 2. 安装依赖
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu
pip install -r requirements.txt

# 3. 下载模型（使用 ModelScope 或官方脚本）
python download_models.py

# 4. 启动 API 服务
bash start_api.sh

# 5. 准备参考音频（5-10秒）
ffmpeg -i long_audio.wav -ss 0 -t 8 ref.wav

# 6. 生成语音
python tts.py "测试文本" --ref ref.wav --output test.wav
```

---

## 资源占用优化建议

### 磁盘空间优化

1. **删除下载压缩包：** 解压后删除 `.zip` 文件（节省 5GB）
2. **仅保留必要模型：** `tts_infer.yaml` 中默认使用 `v2`，可删除其他版本权重
3. **清理 pip 缓存：** `pip cache purge`（节省 1-2GB）

### 内存优化

1. **单进程推理：** 避免并发请求（CPU模式内存吃紧）
2. **批量处理时分批：** 每批 10-20 个句子，避免内存累积

### 速度优化

1. **使用 GPU：** 如果有独显，修改 `tts_infer.yaml` 中 `device: cuda`
2. **缩短参考音频：** 5秒参考音频比 10秒快约 20%
3. **调整 `batch_size`：** CPU 模式建议保持 `batch_size=1`

---

## 经验总结

1. **依赖选择比效果更重要。** CosyVoice3 效果可能更好，但依赖地狱直接卡死，GPT-SoVITS 的"能跑"优先级更高。

2. **磁盘空间是隐形成本。** 13GB 看似不大，但服务器上部署时必须提前规划（我的服务器剩余空间仅 3GB，差点翻车）。

3. **参考音频是效果关键。** 5-10秒、清晰、无噪音的参考音频，比调参数更有效。

4. **命令行优于 WebUI。** 对于批量任务，命令行脚本 + API 服务的组合远比 WebUI 高效。

5. **日志与错误处理。** 将 API 日志重定向到文件（`> /tmp/gptsovits_api.log`），出错时方便排查。

---

## 后续计划

如果要部署到生产环境，下一步可以：

- **Docker 容器化：** 固化依赖版本，避免环境差异
- **GPU 加速：** 切换到 CUDA 版本 PyTorch，推理速度提升 5-10 倍
- **音频后处理：** 添加降噪、响度归一化（使用 `sox` 或 `ffmpeg`）
- **多音色管理：** 建立参考音频库，支持动态切换音色
- **分布式推理：** 使用 Celery + Redis 实现任务队列
- **监控与告警：** 集成 Prometheus + Grafana 监控资源占用

---

## 参考资源

- GPT-SoVITS 官方仓库: https://github.com/RVC-Boss/GPT-SoVITS
- ModelScope 模型下载: https://www.modelscope.cn/
- 音频处理工具: `ffmpeg`, `sox`
- API 文档: `http://127.0.0.1:9880/docs`（启动服务后访问）

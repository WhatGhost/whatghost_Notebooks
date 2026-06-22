# ComfyUI 安装与运行 LTX-2.3 文生视频教程

在 AMD GPU(ROCm)服务器上从零安装 ComfyUI,跑通官方 LTX-2.3 文生视频示例。
参考:<https://docs.comfy.org/tutorials/video/ltx/ltx-2-3>

环境:Ubuntu 24.04 + ROCm 7.2 + PyTorch 2.9(HIP)+ AMD gfx1100(48GB 显存)。

---

## 0. 前置


配置国内源(可选):

```bash
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
pip config set global.trusted-host pypi.tuna.tsinghua.edu.cn
```

---

## 1. 安装 ComfyUI（若已安装可跳过）

```bash
git clone https://github.com/comfyanonymous/ComfyUI.git
cd ComfyUI
```

安装依赖(剔除 torch 三件套,避免覆盖已装好的 ROCm 版 PyTorch):

```bash
pip install $(grep -viE '^(torch|torchvision|torchaudio)\s*$' requirements.txt)
```

> LTX-2.3 所需节点已内置在新版 ComfyUI,无需安装自定义节点。

---

## 2. 下载模型 & 放置位置

使用 BF16 全精度版本(兼容性最好)。共 5 个文件:

| 模型 | 文件名 | 放置目录 | 大小 |
|---|---|---|---|
| 主扩散模型 | `ltx-2.3-22b-dev.safetensors` | `models/checkpoints/` | 46GB |
| 文本编码器 | `gemma_3_12B_it.safetensors` | `models/text_encoders/` | 24GB |
| 蒸馏 LoRA | `ltx_2.3_22b_distilled_1.1_lora_dynamic_fro09_avg_rank_111_bf16.safetensors` | `models/loras/` | 2.7GB |
| Gemma LoRA | `gemma-3-12b-it-abliterated_lora_rank64_bf16.safetensors` | `models/loras/` | 0.6GB |
| 空间超分模型 | `ltx-2.3-spatial-upscaler-x2-1.1.safetensors` | `models/latent_upscale_models/` | 1GB |

目录结构:

```
ComfyUI/models/
├── checkpoints/        ltx-2.3-22b-dev.safetensors
├── text_encoders/      gemma_3_12B_it.safetensors
├── loras/              ltx_2.3_22b_distilled_1.1_lora_dynamic_fro09_avg_rank_111_bf16.safetensors
│                       gemma-3-12b-it-abliterated_lora_rank64_bf16.safetensors
└── latent_upscale_models/  ltx-2.3-spatial-upscaler-x2-1.1.safetensors
```

用 ModelScope(魔搭)下载:

```bash
pip install modelscope
mkdir -p models/{checkpoints,text_encoders,loras,latent_upscale_models}

# 仓库根目录文件:直接落到目标目录
modelscope download --model Lightricks/LTX-2.3 ltx-2.3-22b-dev.safetensors --local_dir models/checkpoints
modelscope download --model Lightricks/LTX-2.3 ltx-2.3-spatial-upscaler-x2-1.1.safetensors --local_dir models/latent_upscale_models

# Comfy-Org 仓库的文件在 split_files/ 下,下载到目标目录后会带子路径,需拍平一次
modelscope download --model Comfy-Org/ltx-2 split_files/text_encoders/gemma_3_12B_it.safetensors --local_dir models/text_encoders
modelscope download --model Comfy-Org/ltx-2.3 split_files/loras/ltx_2.3_22b_distilled_1.1_lora_dynamic_fro09_avg_rank_111_bf16.safetensors --local_dir models/loras
modelscope download --model Comfy-Org/ltx-2 split_files/loras/gemma-3-12b-it-abliterated_lora_rank64_bf16.safetensors --local_dir models/loras

# 拍平 split_files 子目录
mv models/text_encoders/split_files/text_encoders/*.safetensors models/text_encoders/
mv models/loras/split_files/loras/*.safetensors models/loras/
rm -rf models/text_encoders/split_files models/loras/split_files
```

---

## 3. 下载官方example 工作流并修改

官方示例页:<https://docs.comfy.org/tutorials/video/ltx/ltx-2-3>
文生视频工作流下载地址:<https://raw.githubusercontent.com/Comfy-Org/workflow_templates/refs/heads/main/templates/video_ltx2_3_t2v.json>

```bash
mkdir -p user/default/workflows
curl -sL -o user/default/workflows/LTX-2.3_t2v.json \
  https://raw.githubusercontent.com/Comfy-Org/workflow_templates/refs/heads/main/templates/video_ltx2_3_t2v.json
```

官方模板默认用 fp8/fp4 量化模型，由于w7900不支持FP8, 改成 BF16模型:

```bash
sed -i \
  -e 's/ltx-2.3-22b-dev-fp8.safetensors/ltx-2.3-22b-dev.safetensors/g' \
  -e 's/gemma_3_12B_it_fp4_mixed.safetensors/gemma_3_12B_it.safetensors/g' \
  user/default/workflows/LTX-2.3_t2v.json
```

改动(git diff 形式),仅模型文件名,其它全部不变:

```diff
-              "ltx-2.3-22b-dev-fp8.safetensors"
+              "ltx-2.3-22b-dev.safetensors"
-              "gemma_3_12B_it_fp4_mixed.safetensors"
+              "gemma_3_12B_it.safetensors"
```

或者可以直接将改动好的workflow "comfyui_ltx\LTX-2.3_t2v_BF16.json" 放入user/default/workflows/文件夹中
---

## 4. 启动 ComfyUI 并运行

```bash
python3 main.py --listen 0.0.0.0 --port 8799
```

看到 `To see the GUI go to: http://0.0.0.0:8799` 即成功。浏览器打开 `http://<服务器IP>:8799`。

在网页里:

1. 左侧 **Workflows** 打开 `LTX-2.3_t2v`(或把 json 拖进网页)。
2. 点击 **Run** 运行。
3. 输出视频保存在 `ComfyUI/output/video/`。


---

## 预期结果

单卡约 3.5~4 分钟出片,输出 1280×704、25fps、约 5 秒

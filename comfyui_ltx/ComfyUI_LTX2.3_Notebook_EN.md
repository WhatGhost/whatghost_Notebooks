# ComfyUI Installation and LTX-2.3 Text-to-Video Tutorial

Install ComfyUI from scratch on an AMD GPU (ROCm) server and run the official LTX-2.3 text-to-video example.
Reference: <https://docs.comfy.org/tutorials/video/ltx/ltx-2-3>

Environment: Ubuntu 24.04 + ROCm 7.2 + PyTorch 2.9 (HIP) + AMD gfx1100 (48GB VRAM).

---

## 0. Prerequisites

Configure a China-based PyPI mirror (optional):

```bash
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
pip config set global.trusted-host pypi.tuna.tsinghua.edu.cn
```

---

## 1. Install ComfyUI (skip if already installed)

```bash
git clone https://github.com/comfyanonymous/ComfyUI.git
cd ComfyUI
```

Install dependencies, excluding the torch packages to avoid overwriting the already installed ROCm version of PyTorch:

```bash
pip install $(grep -viE '^(torch|torchvision|torchaudio)\s*$' requirements.txt)
```

> The nodes required for LTX-2.3 are already built into recent versions of ComfyUI, so no custom nodes are needed.

---

## 2. Download Models and Place Them in the Correct Directories

Use the BF16 full-precision versions for the best compatibility. There are 5 files in total:

| Model | File name | Target directory | Size |
|---|---|---|---|
| Main diffusion model | `ltx-2.3-22b-dev.safetensors` | `models/checkpoints/` | 46GB |
| Text encoder | `gemma_3_12B_it.safetensors` | `models/text_encoders/` | 24GB |
| Distilled LoRA | `ltx_2.3_22b_distilled_1.1_lora_dynamic_fro09_avg_rank_111_bf16.safetensors` | `models/loras/` | 2.7GB |
| Gemma LoRA | `gemma-3-12b-it-abliterated_lora_rank64_bf16.safetensors` | `models/loras/` | 0.6GB |
| Spatial upscaler model | `ltx-2.3-spatial-upscaler-x2-1.1.safetensors` | `models/latent_upscale_models/` | 1GB |

Directory structure:

```text
ComfyUI/models/
|-- checkpoints/             ltx-2.3-22b-dev.safetensors
|-- text_encoders/           gemma_3_12B_it.safetensors
|-- loras/                   ltx_2.3_22b_distilled_1.1_lora_dynamic_fro09_avg_rank_111_bf16.safetensors
|                            gemma-3-12b-it-abliterated_lora_rank64_bf16.safetensors
`-- latent_upscale_models/   ltx-2.3-spatial-upscaler-x2-1.1.safetensors
```

Download with ModelScope:

```bash
pip install modelscope
mkdir -p models/{checkpoints,text_encoders,loras,latent_upscale_models}

# Files in the repository root: download directly into the target directories.
modelscope download --model Lightricks/LTX-2.3 ltx-2.3-22b-dev.safetensors --local_dir models/checkpoints
modelscope download --model Lightricks/LTX-2.3 ltx-2.3-spatial-upscaler-x2-1.1.safetensors --local_dir models/latent_upscale_models

# Files in the Comfy-Org repositories are under split_files/.
# After downloading into the target directories, they will include subpaths and need to be flattened once.
modelscope download --model Comfy-Org/ltx-2 split_files/text_encoders/gemma_3_12B_it.safetensors --local_dir models/text_encoders
modelscope download --model Comfy-Org/ltx-2.3 split_files/loras/ltx_2.3_22b_distilled_1.1_lora_dynamic_fro09_avg_rank_111_bf16.safetensors --local_dir models/loras
modelscope download --model Comfy-Org/ltx-2 split_files/loras/gemma-3-12b-it-abliterated_lora_rank64_bf16.safetensors --local_dir models/loras

# Flatten the split_files subdirectories.
mv models/text_encoders/split_files/text_encoders/*.safetensors models/text_encoders/
mv models/loras/split_files/loras/*.safetensors models/loras/
rm -rf models/text_encoders/split_files models/loras/split_files
```

---

## 3. Download and Modify the Official Example Workflow

Official example page: <https://docs.comfy.org/tutorials/video/ltx/ltx-2-3>
Text-to-video workflow download URL: <https://raw.githubusercontent.com/Comfy-Org/workflow_templates/refs/heads/main/templates/video_ltx2_3_t2v.json>

```bash
mkdir -p user/default/workflows
curl -sL -o user/default/workflows/LTX-2.3_t2v.json \
  https://raw.githubusercontent.com/Comfy-Org/workflow_templates/refs/heads/main/templates/video_ltx2_3_t2v.json
```

The official template uses fp8/fp4 quantized models by default. Since the W7900 does not support FP8, change it to use the BF16 models:

```bash
sed -i \
  -e 's/ltx-2.3-22b-dev-fp8.safetensors/ltx-2.3-22b-dev.safetensors/g' \
  -e 's/gemma_3_12B_it_fp4_mixed.safetensors/gemma_3_12B_it.safetensors/g' \
  user/default/workflows/LTX-2.3_t2v.json
```

Changes in `git diff` format. Only the model file names are changed; everything else remains the same:

```diff
-              "ltx-2.3-22b-dev-fp8.safetensors"
+              "ltx-2.3-22b-dev.safetensors"
-              "gemma_3_12B_it_fp4_mixed.safetensors"
+              "gemma_3_12B_it.safetensors"
```

Alternatively, you can directly place the already modified workflow `comfyui_ltx\LTX-2.3_t2v_BF16.json` into the `user/default/workflows/` folder.

---

## 4. Start ComfyUI and Run the Workflow

```bash
python3 main.py --listen 0.0.0.0 --port 8799
```

When you see `To see the GUI go to: http://0.0.0.0:8799`, the server has started successfully. Open `http://<server-ip>:8799` in your browser.

In the web UI:

1. Click **Workflows** on the left sidebar.
2. Open `LTX-2.3_t2v`, or drag the JSON file into the web page.
3. Click **Run**.

<img width="938" height="438" alt="confyui_1" src="https://github.com/user-attachments/assets/0c930563-ee00-4c3e-87e6-faeea8836ebb" />

4. The output video is saved in `ComfyUI/output/video/`. This example generates a video like the one shown below.

<img width="807" height="437" alt="confyui_2" src="https://github.com/user-attachments/assets/fc41917c-377f-4436-b227-7b4ecf514588" />

---

## Expected Result

On a single GPU, generation takes about 3.5 to 4 minutes and outputs a video at 1280x704, 25fps, around 5 seconds long.

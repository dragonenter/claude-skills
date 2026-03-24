---
name: comfyui-civitai-research
description: Use when searching CivitAI for checkpoints, LoRAs, or upscalers for ComfyUI workflows. Use when analyzing CivitAI image generation parameters to reverse-engineer working configurations. Use when verifying model/LoRA compatibility before integration.
---

# ComfyUI CivitAI Model Research

## Overview
Systematic approach to finding, evaluating, and integrating CivitAI models into ComfyUI workflows. Covers API-based search, image parameter extraction, format compatibility verification, and proper installation.

## When to Use
- Need a checkpoint/LoRA for a specific art style (3D CG, 2.5D anime, semi-realistic, etc.)
- Found a good reference image on CivitAI, need to extract its exact generation parameters
- Downloaded a model, need to verify it's compatible before adding to workflow
- Need to find the right trigger words, CFG, sampler settings for a model

## Core Pattern

### 1. Search Models via API

```bash
# Search by model ID
curl -s https://civitai.com/api/v1/models/{MODEL_ID}

# Get specific version
curl -s https://civitai.com/api/v1/model-versions/{VERSION_ID}

# Download URL pattern
https://civitai.com/api/download/models/{VERSION_ID}
```

### 2. Extract Image Generation Parameters

```bash
# Get image metadata (NSFW content requires nsfw=X parameter)
curl -s "https://civitai.com/api/v1/images?imageId={IMAGE_ID}&nsfw=X&limit=1"
```

Key fields in response:
- `meta.prompt` / `meta.negativePrompt` — exact prompts used
- `meta.cfgScale`, `meta.sampler`, `meta.steps`, `meta.seed`
- `meta.clipSkip` — critical for Illustrious models (must be 2)
- `meta.resources` — checkpoint, LoRAs, upscalers with version IDs

### 3. Verify LoRA Format Compatibility

```python
from safetensors.torch import safe_open
with safe_open('model.safetensors', framework='pt') as f:
    keys = list(f.keys())
    has_diff_b = any('diff_b' in k for k in keys)
    fmt = 'lora_A/B' if any('lora_A' in k for k in keys) else 'lora_down/up'
    print(f'Keys: {len(keys)}, format: {fmt}, has_diff_b: {has_diff_b}')
```

**Compatible formats:** `lora_A/lora_B` and `lora_down/lora_up` (without `diff_b`)
**Incompatible:** `lora_down/lora_up` with `diff_b` keys — causes `'NoneType' has no attribute 'Params'`

### 4. Check Available Models via ComfyUI API

```bash
# List all available checkpoints
curl -s http://127.0.0.1:8188/object_info/CheckpointLoaderSimple | python3 -c "
import json, sys
data = json.load(sys.stdin)
for c in data['CheckpointLoaderSimple']['input']['required']['ckpt_name'][0]:
    print(c)"

# Verify a model was actually loaded (check execution history)
curl -s http://127.0.0.1:8188/history | python3 -c "
import json, sys
data = json.load(sys.stdin)
for pid in sorted(data.keys(), reverse=True)[:3]:
    prompt = data[pid].get('prompt', [None,None,None])[2]
    if prompt:
        for nid, node in prompt.items():
            if node.get('class_type') == 'CheckpointLoaderSimple':
                print(f'{pid[:12]}: {node[\"inputs\"][\"ckpt_name\"]}')"
```

### 5. Download with Authentication

```bash
# With CivitAI cookie for authenticated downloads
wget --header="Cookie: __Secure-civitai-token=YOUR_TOKEN" \
  -O model.safetensors "https://civitai.com/api/download/models/{VERSION_ID}"

# If behind proxy, bypass it
env no_proxy='*' wget ...

# Use nohup for large files
nohup env no_proxy='*' wget ... > download.log 2>&1 &
```

## Quick Reference

| Model Type | Install Path | ComfyUI Node |
|-----------|-------------|-------------|
| Checkpoint | `models/checkpoints/` | CheckpointLoaderSimple |
| LoRA | `models/loras/` | LoraLoader / LoraLoaderModelOnly |
| VAE | `models/vae/` | VAELoader |
| Upscaler | `models/upscale_models/` | UpscaleModelLoader |
| IPAdapter | `models/ipadapter/` | IPAdapterUnifiedLoader |
| CLIP Vision | `models/clip_vision/` | auto-loaded by IPAdapter |

## Common Mistakes

- **New model not showing in dropdown**: ComfyUI caches model list at startup. Must refresh model list in UI or restart ComfyUI.
- **Model loads but style unchanged**: Check `/history` API to confirm the new model was actually used, not cached old model.
- **FP8 models crash with `Params` error**: Requires `comfy_kitchen` package. Use non-FP8 models if unavailable.
- **LoRA has no effect**: Check trigger words on CivitAI page. Some LoRAs require specific trigger words in the prompt.
- **Wrong quality tags**: Illustrious models use `masterpiece, best quality, absurdres, newest`. Pony models use `score_9, score_8_up`. Don't mix them.

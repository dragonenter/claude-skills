---
name: comfyui-workflow-debugging
description: Use when ComfyUI workflows fail with errors like NoneType Params, prompt_outputs_failed_validation, required_input_missing, or model not found. Use when generated images have wrong style despite correct checkpoint. Use when API-submitted prompts silently fail.
---

# ComfyUI Workflow Debugging

## Overview
Systematic approach to diagnosing and fixing ComfyUI workflow failures. Covers model loading errors, node validation failures, API format mismatches, and style consistency issues.

## When to Use
- Workflow errors: `'NoneType' object has no attribute 'Params'`
- API errors: `prompt_outputs_failed_validation`
- Model loads but output style is wrong
- Node missing required inputs
- Workflows work in UI but fail via API

## Diagnostic Flowchart

```
Error occurs
  ├─ 'NoneType' Params → FP8 model + missing comfy_kitchen (see #1)
  ├─ prompt_outputs_failed_validation → Node validation error (see #2)
  ├─ model not found → Wrong preset or model not in directory (see #3)
  ├─ Style unchanged → Model not actually loaded / cached (see #4)
  └─ API 200 but no output → Prompt accepted but execution failed (see #5)
```

## Issue #1: FP8 Model Crash

**Error:** `'NoneType' object has no attribute 'Params'`

**Root cause:** `comfy_kitchen` not installed. `get_layout_class()` returns `None` for FP8 quantized models.

**Fix options:**
1. Install `comfy_kitchen`: `pip install comfy_kitchen`
2. Use non-FP8 models (e.g., Remix NSFW models instead of `fp8_scaled` variants)

**Diagnosis:**
```python
python3 -c "import comfy_kitchen; print('OK')"
```

## Issue #2: Node Validation Failure

**Error:** `prompt_outputs_failed_validation` with `node_errors`

**Diagnosis:** Read the full error response — it tells you exactly which node and which input is wrong.

```bash
curl -s -X POST http://127.0.0.1:8188/prompt \
  -H 'Content-Type: application/json' \
  -d '{"prompt": YOUR_PROMPT}' | python3 -m json.tool
```

**Common causes:**
- Node API parameters changed after ComfyUI update (e.g., QwenVL: `attention_backend` → `attention_mode`, `mode` → `preset_prompt`)
- Check current node parameters: `curl -s http://127.0.0.1:8188/object_info/NODE_TYPE`
- `LoadImage` can only load from `input/` directory, not `output/`
- Missing optional inputs that became required in newer versions

## Issue #3: Model Not Found

**Error:** `IPAdapter model not found` or similar

**Diagnosis:**
```bash
# Check what presets/models are available
curl -s http://127.0.0.1:8188/object_info/IPAdapterUnifiedLoader | python3 -c "
import json, sys
data = json.load(sys.stdin)
print(json.dumps(data, indent=2)[:2000])"

# Check what files exist
ls -la models/ipadapter/
```

**Fix:** Match preset name to available model files. `PLUS FACE (portraits)` requires `ip-adapter-plus-face_sdxl_vit-h.safetensors`. `PLUS (high strength)` requires a different file.

## Issue #4: Style Not Changing Despite New Checkpoint

**Diagnosis checklist:**
1. **File exists?** `ls -la models/checkpoints/path/to/model.safetensors`
2. **ComfyUI sees it?** Check API: `curl -s http://127.0.0.1:8188/object_info/CheckpointLoaderSimple`
3. **Actually loaded?** Check history: was the new model name in the executed prompt?
4. **Prompt format correct?** Different models need different quality tags (Illustrious vs Pony vs SDXL)
5. **Clip Skip correct?** Illustrious models require Clip Skip 2 (`CLIPSetLastLayer: -2`)
6. **CFG correct?** Illustrious: 3-5, Pony: 5-7

**Key insight:** If file was added while ComfyUI was running, the dropdown won't show it. Refresh model list in UI or restart ComfyUI.

## Issue #5: API Prompt Accepted But No Output

**Symptom:** API returns `prompt_id` and `200`, but `/history/` shows no output or shows error status.

**Diagnosis:**
```bash
# Check ComfyUI execution log
tail -50 nohup.out | grep -i "error\|exception\|invalid"

# Check queue status
curl -s http://127.0.0.1:8188/queue | python3 -m json.tool

# Check specific prompt result
curl -s http://127.0.0.1:8188/history/PROMPT_ID | python3 -m json.tool
```

**Common causes:**
- `invalid prompt` in log = validation passed at API level but failed at execution level
- Check if `node_errors` in history response is non-empty
- LoRA format incompatibility causes silent failure

## Quick Reference: API vs UI Workflow Differences

| Issue | UI Workflow | API Prompt |
|-------|-----------|-----------|
| Node connections | Visual links | `[source_node_id, output_index]` arrays |
| Widget values | Dropdown/input | Must match exact option strings |
| Missing optionals | Shows default | May cause validation error |
| `control_after_generate` | Frontend concept | Pass seed directly, ignore this field |
| File paths | Relative to model dirs | Same, but must exist in ComfyUI's scan |

## Common Mistakes

- **Proxy interference**: System HTTP proxy intercepts localhost requests. Use `env no_proxy='*'` or `urllib.request.ProxyHandler({})`.
- **Browser JS cache**: Modified `.js` files not loading. Add `?v=timestamp` to script src tags.
- **ConditioningConcat attention dilution**: Two conditionings concatenated via `ConditioningConcat` can have uneven attention weights. Use `StringConcatenate` + single `CLIPTextEncode` instead for consistent results.
- **Different pipelines = different style**: If one image uses IPAdapter and another doesn't, even with same prompt and seed, the output style differs. Keep pipeline identical across related generations.

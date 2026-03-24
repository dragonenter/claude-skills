---
name: comfyui-prompt-engineering
description: Use when writing positive/negative prompts for ComfyUI image or video generation. Use when switching checkpoints and need to adapt prompt format. Use when generated images don't match the intended style despite correct model setup. Use when building multi-view consistent character sheets.
---

# ComfyUI Prompt Engineering

## Overview
Prompt format varies dramatically by model ecosystem. Wrong format = wasted generation. This skill covers correct prompt structures for major ecosystems, style-specific techniques, and multi-view consistency strategies.

## When to Use
- Switching from one checkpoint to another
- Images look wrong despite correct model/LoRA setup
- Need to generate consistent multi-view character sheets
- Building batch prompt sets for systematic generation
- Adapting prompts from CivitAI examples to your workflow

## Model-Specific Prompt Formats

### Illustrious Ecosystem (IlluQuaint, Nova 3DCG, Nova Anime, etc.)

```
Positive: masterpiece, best quality, amazing quality, 4k, very aesthetic,
  high resolution, ultra-detailed, absurdres, newest,
  [character/scene description],
  BREAK,
  [lighting/post-processing]

Negative: modern, recent, old, oldest, cartoon, graphic, text, painting,
  crayon, graphite, abstract, glitch, deformed, mutated, ugly, disfigured,
  long body, lowres, bad anatomy, bad hands, missing fingers, extra digits,
  fewer digits, cropped, very displeasing, (worst quality, bad quality:1.2),
  sketch, jpeg artifacts, signature, watermark, username, simple background,
  conjoined, bad ai-generated
```

**Settings:** CFG 3-5, Sampler: euler_ancestral, Clip Skip: 2

### Pony Ecosystem

```
Positive: score_9, score_8_up, score_7_up, score_6_up, source_anime,
  [character/scene description]

Negative: score_4, score_5, 3d, jpeg artifacts, watermark, signature,
  worst quality, low quality
```

**Settings:** CFG 5-7, Sampler: Euler a, Clip Skip: 1-2

### Wan 2.2 Video (I2V with LoRA)

**NN Semi-realistic LoRA:**
```
Positive prefix: nns3m1, digital art, anime style, 2.5d, semi-realistic,
  gradient shading, glossy skin, anime eyes, soft shadows, cinematic lighting,
  [motion description]
```

**No style tags needed** — video models respond to motion descriptions, not quality tags.

## Style Control Techniques

### Push Toward 3D CG (on Illustrious)
```
Positive: + 3d, rendered, volumetric lighting, depth of field, rim lighting
Negative: + flat colors, flat shading, cel shading, lineart, sketch
```

### Push Toward 2.5D Semi-Realistic
```
Positive: + detailed skin, realistic lighting, gradient shading, soft shadows
Negative: + 3d, plastic, doll, flat colors, cel shading, ((photorealistic))
```

### Push Toward Anime
```
Positive: + anime, cel shading, anime eyes, source_anime
Negative: + ((photorealistic)), ((real person)), 3d render
```

### LoRA Trigger Words Matter

| LoRA | Trigger Word | Without It |
|------|-------------|-----------|
| NN Semi-realistic | `nns3m1` | LoRA effect minimal |
| FF7 Rebirth Style | `ffviirb` | Generic 3D look |
| Character CG | `character cg, black background` | Standard checkpoint output |
| AIイラストおじさん | None needed | Style LoRA, always active |

## Multi-View Character Sheet Strategy

### Problem
Same seed + different prompt ≠ same character. View angle tags (`front view` vs `side view`) change text conditioning enough to produce completely different characters.

### Solution: Consistent Pipeline

**With reference image (best consistency):**
1. All views use identical pipeline: same IPAdapter reference + same QwenVL analysis
2. Use `StringConcatenate` to merge QwenVL output + scene prompt into ONE text
3. Do NOT use `ConditioningConcat` — it splits attention weights unevenly across views

```
Pipeline per view:
LoadImage → QwenVL → StringConcatenate(appearance + scene) → CLIPTextEncode
                                                                    ↓
LoadImage → IPAdapter ──────────────────────────────────────→ KSampler
```

**Without reference image:**
1. Generate front view first (pure text)
2. Copy front output to `input/` directory
3. Use front image as IPAdapter reference for side/back views

### Three-View Prompt Template

```javascript
viewExtraTags: {
  front: ', nude, completely nude, naked, full body, standing, white background, nipples, pussy, navel',
  side: ', nude, completely nude, naked, full body, standing, white background, nipples, navel, bare skin',
  back: ', nude, completely nude, naked, full body, standing, white background, ass, bare back, bare butt',
}

viewNegExtra: ', (clothes:1.3), (dressed:1.3), swimsuit, bikini, underwear, shirt, skirt, pants, dress'
```

**Key:** Explicit `nude, completely nude, naked` in EVERY view's positive, plus weighted clothing terms in negative. Back view especially needs reinforcement.

## Character Description: Less is More

### Bad: Over-specified face (model conflicts)
```
oval face, soft features, small nose, thin eyebrows, gentle eyes,
brown eyes, droopy eyes, tareme, glossy lips, nude lipstick,
natural makeup, long eyelashes, light blush
```

### Good: Minimal face description
```
beautiful detailed face, japanese, black hair, long hair, brown eyes,
(large breasts:1.3), skindentation
```

### Best: Character name anchor (if model knows the character)
```
tifa lockhart, beautiful detailed face, brown eyes, long black hair
```

**Why:** Too many face attributes compete for attention. The model can't satisfy all constraints simultaneously, producing distorted faces. Character names give the model a single coherent reference.

## Faceless Male Pattern

For female-focused scenes with a male character:
```
faceless male, out of frame face, muscular arms, on top, between legs
```
or
```
faceless male, face out of frame, muscular arms, kneeling behind, grabbing hips
```

## Batch Prompt Generation Pattern

Structure: `[quality prefix] + [character] + [outfit] + [pose] + [expression] + [scene] + [camera] + BREAK + [lighting]`

Vary systematically across dimensions:
- **Poses:** missionary, spooning, doggy, cowgirl, standing, reverse cowgirl
- **Scenes:** bedroom, kitchen, hotel, bathroom, office, japanese room
- **Expressions:** shy/blush, pain/tears, calm/relaxed, pleasure/ahegao
- **Outfits:** lingerie, naked apron, bathrobe, OL uniform, school uniform, negligee

## Common Mistakes

- **Mixing model ecosystem tags**: Using `score_9` on Illustrious models or `masterpiece, best quality` on Pony models.
- **Ignoring Clip Skip**: Illustrious models MUST use Clip Skip 2. Default (1) produces visibly worse output.
- **Overloading prompts with style tags**: `3d, 3d render, cg artwork, unreal engine, octane render, volumetric lighting, ray tracing` — most of these are redundant. Keep it simple.
- **Copying prompts from different model**: A prompt that works on Nova 3DCG XL won't work on IlluQuaint without format adaptation.
- **ConditioningConcat for multi-view**: Splits attention unevenly between character description and scene text. Use StringConcatenate + single CLIPTextEncode instead.

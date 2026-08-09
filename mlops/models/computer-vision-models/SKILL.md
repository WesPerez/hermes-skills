---
name: computer-vision-models
description: "Computer vision foundation models: SAM (Segment Anything) for zero-shot image segmentation, CLIP for zero-shot image classification and text-image matching."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [computer-vision, segmentation, sam, clip, zero-shot, image-classification, vision-language]
---
# Computer Vision Models

Collection of computer vision foundation models for zero-shot image understanding. Covers **SAM** (Segment Anything) for image segmentation and **CLIP** for image classification and text-image matching. No fine-tuning required — these work out of the box.

## When to Use

- User wants to segment objects in images without training
- User wants to classify images by natural language descriptions
- User wants to find images matching a text query (semantic image search)
- User needs to build interactive annotation tools
- User wants to extract object masks or transparent-background images

## Reference Files

| File | Covers |
|------|--------|
| `references/advanced-usage.md` | SAM batching, fine-tuning, integration patterns |
| `references/troubleshooting.md` | SAM OOM, slow inference, poor mask quality, edge artifacts |
| `references/clip-applications.md` | CLIP advanced use cases: embedding caching, vector DB integration, recipe search |

---

## Part 1: SAM — Segment Anything Model

Zero-shot image segmentation from Meta AI. Trained on 1.1B masks across 11M images.

### Installation

```bash
# From GitHub
pip install git+https://github.com/facebookresearch/segment-anything.git
pip install opencv-python pycocotools matplotlib

# Or via HuggingFace Transformers
pip install transformers

# Download checkpoints (ViT-H 2.4GB, ViT-L 1.2GB, ViT-B 375MB)
wget https://dl.fbaipublicfiles.com/segment_anything/sam_vit_h_4b8939.pth
```

### Basic Usage (Point Prompt)

```python
import numpy as np
from segment_anything import sam_model_registry, SamPredictor

sam = sam_model_registry["vit_h"](checkpoint="sam_vit_h_4b8939.pth")
sam.to(device="cuda")

predictor = SamPredictor(sam)
predictor.set_image(image)  # Computes embeddings (once)

# Point prompt: 1 = foreground, 0 = background
masks, scores, logits = predictor.predict(
    point_coords=np.array([[500, 375]]),
    point_labels=np.array([1]),
    multimask_output=True
)
best_mask = masks[np.argmax(scores)]
```

### Using HuggingFace Transformers

```python
from transformers import SamModel, SamProcessor

model = SamModel.from_pretrained("facebook/sam-vit-huge")
processor = SamProcessor.from_pretrained("facebook/sam-vit-huge")

inputs = processor(image, input_points=[[[450, 600]]], return_tensors="pt")
with torch.no_grad():
    outputs = model(**inputs)
masks = processor.image_processor.post_process_masks(outputs.pred_masks.cpu(), ...)
```

### Automatic Mask Generation

```python
from segment_anything import SamAutomaticMaskGenerator

mask_generator = SamAutomaticMaskGenerator(sam)
masks = mask_generator.generate(image)

# Each mask: segmentation, bbox, area, predicted_iou, stability_score, point_coords
```

### Prompt Types

| Prompt | Description | Use Case |
|--------|-------------|----------|
| Point (foreground) | Click on object | Single object selection |
| Point (background) | Click outside object | Exclude regions |
| Bounding box | Rectangle around object | Larger objects |
| Previous mask | Low-res mask input | Iterative refinement |

### Model Variants

| Model | Checkpoint | Size | Speed | Accuracy |
|-------|-----------|------|-------|----------|
| ViT-H | `vit_h` | 2.4 GB | Slowest | Best |
| ViT-L | `vit_l` | 1.2 GB | Medium | Good |
| ViT-B | `vit_b` | 375 MB | Fastest | Good |

### Common Workflows

**Object extraction (transparent background):**
```python
masks, scores, _ = predictor.predict(
    point_coords=np.array([[500, 375]]),
    point_labels=np.array([1]),
    multimask_output=True
)
best_mask = masks[np.argmax(scores)]
rgba = np.zeros((*image.shape[:2], 4), dtype=np.uint8)
rgba[:, :, :3] = image
rgba[:, :, 3] = best_mask * 255
```

**Interactive annotation:**
```python
def on_click(event, x, y, flags, param):
    if event == cv2.EVENT_LBUTTONDOWN:
        masks, scores, _ = predictor.predict(
            point_coords=np.array([[x, y]]),
            point_labels=np.array([1]),
            multimask_output=True
        )
        display_mask(masks[np.argmax(scores)])
```

### Pitfalls — SAM

1. **Out of memory** — Use ViT-B model, reduce image size, or use half precision.
2. **Slow inference** — Use ViT-B, reduce `points_per_side` for automatic generation.
3. **Poor mask quality** — Try different prompts (box + points for precision).
4. **Edge artifacts** — Use `stability_score` filtering.
5. **Small objects missed** — Increase `points_per_side` for automatic mask generation.

---

## Part 2: CLIP — Contrastive Language-Image Pre-Training

Zero-shot image classification and text-image matching from OpenAI. Trained on 400M image-text pairs.

### Installation

```bash
pip install git+https://github.com/openai/CLIP.git
pip install torch torchvision ftfy regex tqdm
```

### Basic Usage (Zero-shot Classification)

```python
import torch
import clip
from PIL import Image

device = "cuda" if torch.cuda.is_available() else "cpu"
model, preprocess = clip.load("ViT-B/32", device=device)

image = preprocess(Image.open("photo.jpg")).unsqueeze(0).to(device)
text = clip.tokenize(["a dog", "a cat", "a bird"]).to(device)

with torch.no_grad():
    logits_per_image, _ = model(image, text)
    probs = logits_per_image.softmax(dim=-1).cpu().numpy()

for label, prob in zip(["a dog", "a cat", "a bird"], probs[0]):
    print(f"{label}: {prob:.2%}")
```

### Available Models

| Model | Params | Speed | Quality |
|-------|--------|-------|---------|
| RN50 | 102M | Fast | Good |
| ViT-B/32 | 151M | Medium | Better |
| ViT-B/16 | 151M | Slower | Better |
| ViT-L/14 | 428M | Slow | Best |

### Image-Text Similarity

```python
image_features = model.encode_image(image)
text_features = model.encode_text(text)
image_features /= image_features.norm(dim=-1, keepdim=True)
text_features /= text_features.norm(dim=-1, keepdim=True)
similarity = (image_features @ text_features.T).item()
```

### Semantic Image Search

```python
# Index images
image_embeddings = []
for img_path in image_paths:
    image = preprocess(Image.open(img_path)).unsqueeze(0).to(device)
    with torch.no_grad():
        embedding = model.encode_image(image)
        embedding /= embedding.norm(dim=-1, keepdim=True)
    image_embeddings.append(embedding)

# Query with text
text_input = clip.tokenize(["a sunset over the ocean"]).to(device)
with torch.no_grad():
    text_embedding = model.encode_text(text_input)
    text_embedding /= text_embedding.norm(dim=-1, keepdim=True)

similarities = (text_embedding @ torch.cat(image_embeddings).T).squeeze(0)
top_k = similarities.topk(3)
```

### Content Moderation

```python
categories = ["safe for work", "not safe for work", "violent content", "graphic content"]
text = clip.tokenize(categories).to(device)
with torch.no_grad():
    logits_per_image, _ = model(image, text)
    probs = logits_per_image.softmax(dim=-1)
print(f"Category: {categories[probs.argmax()]} ({probs.max():.2%})")
```

### Pitfalls — CLIP

1. **Not for fine-grained tasks** — Best for broad categories.
2. **Requires descriptive text** — Vague labels perform poorly.
3. **Biased on web data** — 400M web-crawled pairs encode real-world biases.
4. **No bounding boxes** — Whole image only; no spatial localization.
5. **Limited spatial understanding** — Poor at counting, position, and relative size.

---

## Verification Checklist

- [ ] SAM checkpoint downloaded and loads without errors
- [ ] Single point-prompt segmentation produces a valid mask
- [ ] Automatic mask generation returns at least one mask
- [ ] CLIP model loads and classifies an image into the correct top-3 categories
- [ ] Text-image similarity score is >0.5 for matching pairs

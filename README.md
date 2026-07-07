# Qwen-Image-Edit (GGUF) Image Edit — Modly Extension

Instruction-based image editing with Alibaba's **Qwen-Image-Edit**, loaded as a 4-bit **GGUF** transformer so it runs on consumer GPUs. Give it one image and a text instruction and it returns edited image(s) — strong at novel viewpoints (rotate an object to show its back/side), style changes, object add/remove, and background swaps.

**Apache 2.0 and ungated** — no HuggingFace login, no token, no license click. The download just works.

---

## VRAM reality (read this first)

Qwen-Image-Edit is a 20B model with a separate ~7B text encoder. The transformer is loaded as a low-bit GGUF, the text encoder is 4-bit, and model CPU offload swaps them on/off the GPU. The **Transformer Quant** is your main VRAM knob — model offload puts the whole transformer on the GPU during denoising, so it has to fit:

- **~12GB:** **Q3_K_M** (~9GB) — the default. Q2_K (~7GB) for more headroom.
- **~10GB:** **Q2_K** (~7GB).
- **16GB:** Q4_K_M (~12GB). **24GB+:** Q5_K_M / Q8_0, or Max Speed (no offload).
- **8GB:** only Q2_K, and only at lower resolution — expect it to be tight.
- **6GB:** not really feasible here — even the smallest quant (~7GB) is larger than the card, and the offload puts the transformer on the GPU as one piece. For genuine 6–8GB use you want the **Nunchaku** build (per-block offload down to ~3GB) — see Notes.

**System RAM matters as much as VRAM** — offloaded weights live there. 32GB+ is strongly recommended; with less you may hit out-of-memory or heavy swapping.

---

## Installation

1. Open Modly → **Extensions** → **Install from GitHub** and paste this repo URL.
2. Wait for setup (installs PyTorch, diffusers, the `gguf` loader, and bitsandbytes into an isolated venv).
3. Click **Download** on the Edit Image node. It pulls two ungated repos: the 4-bit GGUF transformer (`calcuis/qwen-image-edit-gguf`) and the decoder bundle (`callgg/image-edit-decoder`, which holds the text encoder + VAE).

If the model fails to load with a `from_single_file` error, your diffusers is too old for Qwen GGUF loading — update it in the extension venv (`pip install -U diffusers`) and retry. This is bleeding-edge; the first run on a given machine sometimes needs that bump.

---

## Usage (Workflows tab)

1. Drag an **Image** node and point it at the image you want to edit.
2. Drag an **Edit Image** node and type your instruction into its **Edit Instruction** field (e.g. `rotate the object 180 degrees to show the back`, `three-quarter side view`, `change the background to a sunset beach`).
3. Connect the **Image** node into the Edit Image node's image input.
4. Connect the **Edit Image** output into your **Preview Image** node.
5. Hit **Run**.

Number of Outputs = 1 emits a single image (same as the text-to-image node). Set 2–4 for multiple variations in one run (each uses a different seed); the node then emits a list of images.

### Prompt tips
Qwen-Image-Edit follows explicit instructions well and preserves the subject's identity. Describe the change directly, make one change at a time, and for viewpoints be specific ("rotate 90° left", "view from above").

---

## Parameters

| Parameter | Default | Notes |
|---|---|---|
| Memory Mode | Auto | Offload strategy (see below) |
| Transformer Quant | Q3_K_M | Main VRAM knob — pick to fit your GPU (see VRAM reality) |
| Steps | 40 | 40–50 is typical; higher = slower |
| CFG Scale | 4.0 | Prompt adherence (`true_cfg_scale`); ~2.5–4 works well |
| Number of Outputs | 1 | 1–4 variations per run |
| Seed | 0 | 0 = random each run; fixed = reproducible |

Changing the Transformer Quant downloads that GGUF file the next time you run (the ~15GB decoder bundle is shared and isn't re-downloaded).

### Memory Mode

| Mode | What it does | Rough VRAM |
|---|---|---|
| Auto / Offload | 4-bit text encoder + model CPU offload | depends on quant (≈ quant size + a few GB) |
| Max Speed | Everything on the GPU, no offload | big GPUs only (24GB+) |

Both modes 4-bit the text encoder via `bitsandbytes` (installed at setup). If it's missing, it falls back to the bf16 encoder, which needs a much larger GPU. Note: GGUF transformers are **not** compatible with per-layer (sequential) CPU offload, so that mode was removed — model offload is the working path.

---

## Notes

- The default transformer is **Q3_K_M** (~9GB), chosen to fit a 12GB GPU. Use the **Transformer Quant** dropdown to size up (Q4/Q5/Q8) or down (Q2_K). Quants come from the [GGUF repo](https://huggingface.co/calcuis/qwen-image-edit-gguf/tree/main).
- If you previously downloaded `qwen-image-edit-iq4_nl.gguf`, it's ~11.6GB and too big for 12GB — you can delete that file from the model folder to reclaim space.
- Want real speed and genuine 6–8GB support? **Nunchaku** ships 4-bit Qwen-Image-Edit with optimized CUDA kernels and per-block offload down to ~3GB. It's an extra, version-matched install, so it's left out of this build — ask for a Nunchaku variant if you want it.

---

## License

Extension code is **MIT** — see [LICENSE](LICENSE). The Qwen-Image-Edit weights it downloads are **Apache 2.0** (Alibaba's Qwen team), which permits commercial use. This is an independent community extension, not affiliated with Alibaba, Hugging Face, or Modly; each user is responsible for complying with the model license and for whatever they generate. Provided "as is", without warranty.

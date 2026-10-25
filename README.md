# Stable Diffusion GenAI Lab

A hands-on repo where I work through **Stable Diffusion** end to end: text-to-image, image-to-image, inpainting, ControlNet, LoRA fine-tuning, and text-to-audio, all with **Hugging Face Diffusers** on **Google Colab**. It also holds my study notes and an interview question bank.

I built it to learn the topic properly and to keep a record I can show. If you are learning too, feel free to fork it.

## What's inside

| Path | What it is |
|---|---|
| `notebooks/01_text_to_image.ipynb` | Load SDXL, then experiment with CFG, steps, samplers, negative prompts |
| `notebooks/02_img2img_inpaint_controlnet.ipynb` | Steering generation with an image, a mask, or edges |
| `notebooks/03_lora_and_speed.ipynb` | Memory/speed benchmarks and a DreamBooth-LoRA recipe |
| `notebooks/04_text_to_audio.ipynb` | Text-to-audio with Stable Audio Open |
| `src/sd_lab/` | Small reusable helper package used by the notebooks and scripts |
| `scripts/` | Command line image and audio generation |
| `app/gradio_app.py` | Simple web UI |
| `docs/01_concepts.md` | Concepts explained plainly |
| `docs/02_landscape_2026.md` | Where SD sits next to FLUX and others, with sources |
| `docs/03_interview_qa.md` | 37 interview questions with answers |
| `docs/04_troubleshooting.md` | Errors I hit and how I fixed them |

## Quick start (Colab)

1. Push this repo to your GitHub (see below) and change the clone URL in each notebook's first cell if your username or repo name differs.
2. Open a notebook from GitHub in Colab: `File > Open notebook > GitHub`.
3. `Runtime > Change runtime type > T4 GPU`.
4. Run the cells top to bottom.

Gated models (Stable Audio Open, SD 3.5) need you to accept the terms on their Hugging Face page and run `notebook_login()` with a token.

## Quick start (local)

```bash
git clone https://github.com/subbuworkspace/stable-diffusion-genai-lab.git
cd stable-diffusion-genai-lab
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

python scripts/generate_image.py "a lighthouse at dawn, watercolor" --seed 7
python scripts/generate_audio.py "gentle rain on a tin roof" --seconds 10
python app/gradio_app.py
```

You need an NVIDIA GPU for anything comfortable. Use `--low-vram` if memory is tight.

## Models used

| Task | Model | Notes |
|---|---|---|
| Text/img2img, ControlNet, LoRA | `stabilityai/stable-diffusion-xl-base-1.0` | Open, fits a T4 in fp16 |
| Inpainting | `diffusers/stable-diffusion-xl-1.0-inpainting-0.1` | |
| Text-to-audio | `stabilityai/stable-audio-open-1.0` | Gated, up to about 47 s, 44.1 kHz stereo |
| Upgrade path | Stable Diffusion 3.5 Medium/Large | Gated, heavier, quantise on small GPUs |

Always check each model's licence before reuse. Model weights are not covered by this repo's MIT licence.

## My results (fill in after running)

| Setup | Time (20 steps, 768px) | Peak VRAM |
|---|---|---|
| SDXL fp16, all on GPU | | |
| SDXL fp16 + CPU offload + VAE slicing | | |

Add your best 4-6 images in an `assets/` folder and link them here. Recruiters scroll for pictures.

## What I learned

- (write 3-5 honest lines here: what surprised you, what failed, what you would try next)

## Ideas for next steps

- Swap in SD 3.5 Medium with 4-bit quantisation and compare quality and speed against SDXL
- Train a small LoRA on a dataset I own and publish it on the Hub with a model card
- Add CLIP-score evaluation over a fixed prompt set
- Deploy the Gradio app as a Hugging Face Space

## Publishing this repo

```bash
cd stable-diffusion-genai-lab
git init
git add .
git commit -m "Initial commit: Stable Diffusion GenAI Lab"
git branch -M main
git remote add origin https://github.com/subbuworkspace/stable-diffusion-genai-lab.git
git push -u origin main
```

## Credits

Built on Hugging Face Diffusers, Transformers and PEFT, and on the Stable Diffusion, SDXL and Stable Audio Open models by Stability AI.

## Licence

Code: MIT. Models and generated outputs follow their own licences.

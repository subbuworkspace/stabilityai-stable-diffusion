# Stable Diffusion interview questions and answers

Answers are written the way I would say them out loud: short first, detail if they push. Where something changes fast (model names, licences), I say so, because that is also what a good candidate does.

**Sections:** [Basics](#basics) | [Architecture](#architecture) | [Sampling and guidance](#sampling-and-guidance) | [Fine-tuning](#fine-tuning) | [Practical / engineering](#practical--engineering) | [Beyond images](#beyond-images) | [Landscape and judgement](#landscape-and-judgement)

---

## Basics

**1. What is a diffusion model?**
A generative model that learns to reverse a gradual noising process. In training we add noise to real data and teach a network to predict that noise. To generate, we start from pure noise and denoise repeatedly until a sample appears.

**2. What makes Stable Diffusion "stable" or different from earlier diffusion models?**
It runs diffusion in a compressed **latent space** instead of pixel space. A VAE shrinks the image, the denoiser works on the small latent, and the VAE decoder brings it back. That cuts compute and memory massively, which is why it runs on consumer GPUs.

**3. Walk me through what happens when I type a prompt and press generate.**
The tokenizer and text encoder(s) turn the prompt into embeddings. We sample random latent noise from the seed. For N steps the denoiser predicts noise (or velocity) conditioned on the embeddings, and the scheduler uses that prediction to produce a slightly cleaner latent. Finally the VAE decoder converts the latent to an image.

**4. What is the difference between training and inference here?**
Training: take a clean latent, add noise at a random timestep, ask the model to predict the noise, minimise the error. Inference: start from noise and run the learned denoiser many times. Inference is iterative, which is why steps matter for speed.

**5. What is a latent, and why is it useful?**
A compact representation of the image produced by the VAE encoder. It keeps the semantic and structural information but drops redundant pixel detail, so every denoising step is far cheaper.

---

## Architecture

**6. What are the main components of Stable Diffusion?**
Text encoder(s), a denoising network (U-Net in SD 1.x/2.x/SDXL, a transformer in SD 3.x), a scheduler, and a VAE.

**7. Why did the field move from U-Net to transformers (DiT / MMDiT)?**
Transformers scale more predictably with data and parameters, and MMDiT lets text and image tokens attend to each other in the same blocks, which tends to improve prompt following and text rendering. SD 3 and FLUX both use this family of designs.

**8. How does the text prompt actually influence the image?**
Through **cross-attention** in U-Net models (image features attend to text embeddings), or joint attention over text and image tokens in MMDiT. Each denoising step is conditioned on those embeddings.

**9. SDXL uses two text encoders. SD 3.5 uses three. Why?**
Different encoders capture different things. CLIP encoders are aligned with images and are good at style and concepts. A big language model like T5-XXL understands longer, more complex prompts and helps with spelled-out text. Combining them improves understanding at the cost of memory, which is why people quantise or drop T5 on small GPUs.

**10. What does the VAE do and what happens if it is poor?**
It encodes and decodes between pixels and latents. A weak decoder limits fine detail, faces and text. Known fp16 overflow issues in SDXL's VAE can produce black images, which is why a fixed fp16 VAE is commonly used.

**11. What is a scheduler, and is it the same as the sampler?**
In Diffusers "scheduler" is the object that defines the noise schedule and the update rule from step *t* to *t-1*. In UIs people say "sampler". Same idea. It affects speed, sharpness and how many steps you need.

---

## Sampling and guidance

**12. Explain classifier-free guidance.**
Train (or use) a model that can predict noise both with and without the text condition. At inference, compute both and extrapolate: `uncond + w * (cond - uncond)`. A bigger `w` follows the prompt more strongly. Too large causes oversaturation and artefacts, too small gives vague, off-prompt images.

**13. How does a negative prompt work?**
It replaces the "unconditional" branch in classifier-free guidance. So the output is pushed away from the negative prompt and toward the positive one.

**14. What happens if I increase the number of steps?**
Usually quality improves quickly and then plateaus. Beyond roughly 20-40 steps with a good sampler (DPM++ family) you pay more time for very little gain. Some samplers, like ancestral ones, keep changing the image as steps increase because they inject fresh noise.

**15. Why does the same seed give the same image?**
The seed fixes the initial noise (and any noise added by ancestral samplers). With the same model, settings and hardware path, the process is deterministic. Different GPUs or library versions can still produce small differences.

**16. DDIM vs Euler vs DPM++: when would you use which?**
DDIM is deterministic and handy for inversion and editing. Euler is simple and robust. DPM++ 2M (often with Karras sigmas) gets good results in fewer steps, so it is a common default. Ancestral variants (Euler a) add randomness and give more variety but are less reproducible across step counts.

**17. What is a distilled / turbo model?**
A model trained to imitate a larger multi-step model in 1-8 steps (often without needing CFG). You gain a lot of speed and lose some diversity and fine control. SD 3.5 Large Turbo is an example from the SD 3.5 family.

**18. What is flow matching, and how is it different from classic diffusion training?**
Instead of predicting noise on a stochastic path, the model learns a velocity field that moves samples along a near-straight path from noise to data. Straighter paths mean fewer steps needed. SD 3.x and FLUX use this style of objective.

---

## Fine-tuning

**19. What is LoRA and why is it popular?**
Low-Rank Adaptation. Freeze the original weights and learn small low-rank matrices whose product is added to selected layers (usually attention). It trains a tiny fraction of the parameters, the output file is MBs, it fits on modest GPUs, and you can load, swap or blend LoRAs at inference.

**20. LoRA vs DreamBooth vs textual inversion vs full fine-tuning?**
Textual inversion learns a new token embedding only (cheapest, least expressive). DreamBooth teaches the model a specific subject from a few images, often implemented with LoRA for efficiency. LoRA is the parameter-efficient mechanism. Full fine-tuning updates everything, giving the most capacity but needing the most VRAM, data and care to avoid forgetting.

**21. What is "overfitting" in DreamBooth/LoRA and how do you spot it?**
The model reproduces training images or the subject appears in every output regardless of prompt, and it stops following other instructions. Spot it by testing prompts that change setting and style. Reduce steps or learning rate, use fewer or more varied images, lower rank, or add class-preservation images.

**22. What does ControlNet add?**
A trainable copy of the encoder that takes a spatial condition (edges, depth, pose, segmentation) and injects it into the frozen base model. It gives structural control without retraining the base.

**23. How would you build a model that generates in a specific brand style?**
Collect images I am licensed to use, caption them consistently, train a LoRA with a held-out prompt set for evaluation, check for overfitting, measure style consistency and prompt adherence, and version the LoRA with the dataset and settings. Also document licence and consent.

---

## Practical / engineering

**24. My generation runs out of GPU memory. What do I try, in order?**
Half precision (fp16/bf16), reduce resolution and batch size, VAE slicing/tiling, attention slicing or memory-efficient attention, model CPU offload, 8/4-bit quantisation for big transformers or the T5 encoder, then a smaller model.

**25. How do you speed up inference?**
Fewer steps with a better scheduler, a distilled/turbo model, `torch.compile`, memory-efficient attention, fp16/bf16, batching, caching text embeddings, and keeping the model resident instead of reloading it.

**26. How do you evaluate a text-to-image model?**
Automatic: CLIP score for text-image alignment, FID for distribution quality, newer learned preference metrics. Human: side-by-side preference tests, since automatic metrics miss a lot (hands, text, composition). For a product: a fixed prompt set, fixed seeds, and regression checks on every model change.

**27. Why do hands and text often look wrong?**
Hands are small, highly variable and under-represented in clear form. Text needs precise glyph-level structure that weaker text encoders and lower-resolution latents lose. Larger encoders (T5) and transformer backbones improved this, but it is still not perfect.

**28. How do you make results reproducible?**
Fix seed, model revision, scheduler and settings, library versions, dtype and device, and log them with the output. Even then, expect tiny differences across hardware.

**29. What is `from_pipe` / sharing components in Diffusers and why use it?**
It builds a different pipeline (say img2img) from an already-loaded one, reusing the same weights. It saves download time and VRAM compared with loading the model twice.

**30. How would you deploy a text-to-image service?**
Keep the model loaded on a GPU worker, queue requests, enforce limits on resolution and steps, cache repeated prompts if useful, add a safety filter and logging, and scale workers horizontally. For demos, Gradio on a Hugging Face Space is the quick route.

---

## Beyond images

**31. How can diffusion generate audio?**
Same pattern. An audio autoencoder compresses waveforms into latents, a diffusion model denoises those latents given a text embedding (Stable Audio Open uses a T5 encoder), and the decoder reconstructs stereo audio. Duration can be conditioned too.

**32. Is "Stable Diffusion" itself used for text-to-audio?**
Strictly no. Stable Diffusion is the image model family. Text-to-audio uses Stable Audio, which shares the latent-diffusion idea and is made by the same company. People loosely say "stable diffusion for audio", and in an interview it is good to gently make that distinction.

**33. Other uses of diffusion?**
Image editing and inpainting, super-resolution, video, 3D, molecule and protein design, robot policy learning, time-series imputation, and even text diffusion research models.

---

## Landscape and judgement

**34. How does Stable Diffusion compare with FLUX today?**
As of my last check (Oct 2026), FLUX models are widely reported to be ahead on out-of-the-box photorealism and prompt following, while Stable Diffusion keeps the largest ecosystem of community LoRAs, ControlNets and tooling, plus lighter models that run on small GPUs. Licences differ and change, so I check the model card before choosing for a commercial project.

**35. When would you pick an open model over a closed API?**
When I need customisation (LoRA, ControlNet), data privacy, predictable cost at volume, or offline use. I pick a closed API when quality and convenience matter more, for example text-heavy marketing images, and I do not want to run GPUs.

**36. What are the ethical and legal concerns?**
Training data consent and copyright, deepfakes and impersonation, bias in outputs, and misuse. Mitigations: licensed or consented data, safety filters, provenance and watermarking, clear usage policies, and honest documentation of limits.

**37. If you had one week on a new diffusion project, what would you do?**
Day 1: define the use case and an evaluation prompt set. Day 2: baseline with an open model and fixed seeds. Day 3-4: improve with prompting, a scheduler choice, then LoRA/ControlNet if needed. Day 5: evaluate against the baseline with humans and metrics. Day 6: optimise speed and memory. Day 7: package a demo and document limits and licences.

---

### Quick-fire round

- **Default CFG range for SDXL-style models?** Roughly 5-8.
- **What does `strength` do in img2img?** How much noise is added to the input image, so how much it can change.
- **What does the VAE downsample factor mean for a 1024px image?** A latent of 128x128 spatial size for 8x compression.
- **Which component handles "what the image should contain"?** The text encoder(s) plus cross/joint attention.
- **What file format should weights use?** `safetensors`, not pickle-based `.ckpt`, for safety.

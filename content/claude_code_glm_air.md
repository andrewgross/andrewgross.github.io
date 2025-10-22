Title: Running Claude Code with GLM Air
Date: 2025-09-08 9:51

Claude Code is fun and all, but I am curious how much is the model and how much is the prompting. It's also really expensive or locks you out. To this end, I wanted to try running Claude Code backed by a local model, named [GLM-Air-FP8](https://huggingface.co/zai-org/GLM-4.5-Air-FP8).  This took a bit of doing as a lot of the pieces aren't quite there yet.

### Running GLM-Air on vLLM

GLM-Air is not a small model, as its an MoE with around 106B parameters, 12B active.  However, its "small enough" to be run with less than 8xH100s, which a nice change of pace. It also punches above its weight pretty well, closing in on Sonnet for coding, at least by the evals.

To run it, I opted to go for dual RTX Pro 6000 Blackwell, as these have 96GB of VRAM each, allowing one to fit the model and a decent bit of context.  Z.ai even released a "native" FP8 version as an alternative to other forms of quantization. This makes more recently Nvidia GPUs ideal since they have native FP8 computation and can run things pretty quickly, in theory.  The reality is that most Blackwell stuff still has pretty spotty support, and the majority of support that is out there is for the B200 cards, which are actually coded a separate architecture anyways (`sm100` vs `sm120`).

Based on the HF repo, I opted to try using vLLM as I have had success with it in the past and it generally aims for broad support for models and relative easy of use, versus the absolute fastest implementation (it is still plenty fast).  In addition, it expects to host multiple streams of communication, which is ideal if you expect Claude Code to be running things in parallel (think SubAgents).

Getting vLLM running with FP8 and Blackwell support took jumping through some hoops. The final config looked something like this:

1. Download the CUDA Toolkit 12.9.1 toolkit as a [runfile](https://developer.nvidia.com/cuda-12-9-1-download-archive?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=24.04&target_type=runfile_local) (I never like using aptitude for this as it always seemed to update itself at the wrong time)
2. On the host OS install the drivers and CUDA toolkit.
3. Make sure you have `nvidia-container-toolkit` installed.
4. Realize you destroyed your CUDA envs on all of your docker containers and get very sad.
5. Copy the CUDA toolkit runfil into your docker container, but **only** install the toolkit, not the drivers.
6. Make sure things look sane with `nvcc` and `nvidia-smi` reporting CUDA 12.9.1, CUDA_HOME finding the right version etc. (this step elides about 3 evenings of work)
7. Install the latest vLLM
8. Realize it doesn't work as it can't seem to find any compatible CUDA kernels.
9. Spend a lot of time re-installing and recompiling vLLM, CUDA, torch, and `uv`.
10. Discover the `--torch-backend cu129` flag for `uv pip install`
11. Realize you have an old version of NCCL, 
12. Realize you need to undo the changes you made to your BIOS for the 4090 p2p drivers.
13. Realize you are a week into this project and haven't generated a single token locally.
14. Realize you saved a 100GB model file directly into your docker image.
15. Lookup how to prune docker images for the umteenth time.
15. Install Flash Attention properly for vLLM.
16. Get vLLM working and get.... *70 tok/s* for bs=1.

In the end of the saga, 70 tok/s wasn't bad, as it is actually on par with what the Anthropic API's usually push out for thier models. However, there was the lingering issue of GLM-4.5 adverting something called MTP. Essentially an in-built draft model you can use with speculative execution. vLLM claimed to support it but I couldn't figure out how to enable it or get it running (Edit: See this [issue](https://github.com/F1LM1/llama.cpp/pull/2) for details). The vLLM GLM-4.5 docs and the GLM-4.5 HF space didn't give instructions.  So why not throw out all of this work and try to get SGLang working instead?

### Running GLM-Air on SGLang

SGLang is definitely not quite as easy to work with as vLLM. It leans more towards the "this is intended to be used by serious hosting places" and doesn't seem to have quite the same coverage in models and quantizations. However, it can be pretty fast (5-15% by some estimations).  But, there was another bonus. The documentation from the Z.ai team explicitly mentioned how to enable MTP on SGLang. That could be interesting.

Getting SGLang running went even deeper than the vLLM rabbit hole, but at least we had CUDA mostly sorted at this point.

1. Install SGLang from the docs, realize it points you to a different version (0.5.2.rc2) than their latest release (0.5.1), which is not actually the latest release, thats 0.5.1.post3 (this matters later). At least they encourage you to use `uv`
2. Try running SGLang with the params mentioned 
3. Get a lot of CUDA errors.  The initial ones were the usual suspects, making sure you have all of your libraries installed properly, `python3-dev`, `nccl`, etc.  The later ones were where it really hurt.
4. Realize that you are getting to the CUDA graph capturing stage when it keeps blowing up, suggesting OOM.
5. Fiddle with `--mem-fraction-static` and `--cuda-graph-max-bs` endlessly to no avail.
6. Try using a much smaller model, like Qwen3-8B but get the same errors, hinting that its not really about VRAM.
7. Reread the error message to realize its complaining about [RMSNorm](https://github.com/sgl-project/sglang/issues/7249).
8. Spend a lot of quality time in Github issues and figure out you need to install a special `sgl-kernel` via a wheel, specifically for cuda 12.9 and python 3.10. This got some models, like Qwen3-8B running on SGLang.
9. Try to run GLM-4.5-Air-FP8 and get another set of errors when capturing the CUDA graph.
10. Discover that its because the Cutlass FP8 kernels aren't really working yet in non-B200 Blackwell cards in SGLang.
11. Discover we can fall back to Triton using the completely undocumented `USE_TRITON_W8A8_FP8_KERNEL` env var.
12. Run SGLang with out model and get tokens out! 70-80 tok/s! Weirdly, it blew up after one response but hey, progress.
13. See that the error mentions running out of memory `Failed a config with error out of resource: shared memory, Required: 131072, Hardware limit: 101376.`
14. Spend a lot more time futzing around with context lengths and cuda block sizes again with no luck.
15. Try enabling MTP anyways and get... **180 tok/s**.  Still not working reliably but thats a pretty heft carrot to keep chasing this down.
16. Discover [this blog post](https://www.jamesflare.com/tuning-fused-moe-triton/). Triton wants to you generate a config for your hardware so it knows what size operations it can use. I didn't have that, so it fell apart as soon as it got enough data to need a larger size.
17. Try to run the tuning function. Get all kinds of import errors.
18. Discover that `0.5.2rc2` moved around a lot of stuff for Triton configs and did not update the tuning scripts.
19. Roll back to `0.5.1` and discover other things are now broken in the tuning scripts.
20. Roll forward to `0.5.1.post3` (after a lot of Git Blame and following tags on Github) and get the tuning scripts to run.
21. Watch the tuning script run suspiciously fast but find no kernels.
22. Edit the tuning script to stop swalling exceptions and print the error and realize that CUDA broke again.
23. Properly reinstall `torch` and `triton` with the proper CUDA connection, and reinstall `0.5.1` from source.
24. Finally get the tuning job running.
25. Realize its going to take hours.
26. Write a blog post while you wait.
27. Discover that SGLang does have RTX 6000 Pro Blackwell Max-Q Workstation configs that its just not using for some reason.
28. Try not to cry
29. Cry a lot.
30. Finish generating your configs. Move them to the appropriate folder in SGLang. Run SGLang from your git checkout.
31. It works! GLM-Air-FP8 went from ~70 tok/s on 2x RTX6000 Pro to over 200 tok/s with MTP enabled at bs=1!
32. Create [PR](https://github.com/sgl-project/sglang/pull/10243) to SGLang with the configs. Watch it languish.

### Translating APIs

vLLM and SGLang are in the process of [adding](https://github.com/vllm-project/vllm/issues/21313) [support](https://github.com/sgl-project/sglang/pull/8149) for Anthropic API compatibility. I suspect driven by the popularity of Claude Code. In the mean time, I didn't want to fight all of my prior battles again but on a fork for API compat. Instead, I looked to LiteLLM. It has the capability to translate receive API calls in one format and translate them to another. In this case, it would "host" the Anthropic API and proxy it to the OpenAI API format for SGLang.  Overall it worked pretty well. Still need to investigate some things for tool calls, in particular web search.


### Actually Configuring Claude Code

* Make sure your models are aliased properly in LiteLLM.
* Its like two env vars to start
* No need to for a smaller model as GLM-Air is already faster than Haiku, just use it for everything.


# tl;dr

1. Clone a copy of SGLang from **at least** 0.5.1.post3 (newer is probably better)
2. Grab my copies of configs for MTP-enabled Triton FP8 kernels for RTX 6000 Pro Workstation cards or generate your own
3. Place them as seen in this [PR](https://github.com/sgl-project/sglang/pull/10243)
4. Compile / build SGLang as appropriate
5. Run your server (inside venv):  `USE_TRITON_W8A8_FP8_KERNEL=1 python3 -m sglang.launch_server   --model-path zai-org/GLM-4.5-Air-FP8   --tp-size 2   --tool-call-parser glm45    --reasoning-parser glm45   --speculative-algorithm EAGLE   --speculative-num-steps 3   --speculative-eagle-topk 1   --speculative-num-draft-tokens 4   --mem-fraction-static 0.94   --served-model-name glm-4.5-air   --host 0.0.0.0   --port 8000`
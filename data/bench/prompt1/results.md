# Prompt:
> In repository github.com/neptuneMagicSauce/catpi
> at commit 85ab294 (before the fix).
> Find the root cause of this bug. Do not fix it.
Full prompt in file prompt1.txt

# GPU: 4070 Ti Super 16 GB

> $ ollama ps
> SIZE     PROCESSOR          CONTEXT

### unsloth/`Devstral-Small-2-24B`-Instruct-2512-GGUF:IQ4_XS
> 19 GB    28%/72% CPU/GPU    65536`
opencode v1: found a root cause in 4 minutes
opencode v2: found the same thing in 13 minutes

### unsloth/`Qwen3.8-27B`-GGUF:UD-IQ3_S
> 15 GB    9%/91% CPU/GPU     65536
was close to the same root cause, looking at its thinking output
but did not finish after 90 minutes

### continuum-ai/`qwen3-coder-30b-a3b`-compacted-19b-256k:Q4_K_M
> 14 GB    1%/99% CPU/GPU     65536
found a generic, vague explanation, close to the same root cause
in 2 minutes

# mradermacher/`Qwen3.8-20B-Minitron`-i1-GGUF:Q4_K_M
> 13 GB    100% GPU           65536
failed: finished without an answer in 15 minutes

# mradermacher/`gemma-4-A4B-98e-v5-coder`-it-i1-GGUF:Q4_K_M
> 13 GB    100% GPU           65536
failed: found an incorrect cause in 2 minutes

# mradermacher/`Qwen3.8-14B-Instruct-Turbo`-i1-GGUF:Q4_K_M
> 10 GB    100% GPU           65536
failed: non-sensical output

# mradermacher/`Qwen3.8-14B`-Instruct-Turbo-GGUF:Q4_K_M
> 10 GB    100% GPU           65536
failed: non-sensical output

# mradermacher/`gpt-oss-coder-20b`-i1-GGUF
> 12 GB    100% GPU           65536
failed: model does not support tools

# mradermacher/`DeepSeek-Coder-V2`-Lite-Instruct-abliterated-i1-GGUF
>  11 GB    100% GPU          65536
failed: model does not support tools

# TeichAI/`gpt-oss-20b-gpt-5`-codex-distill-GGUF
failed: model does not support tools

# mradermacher/`GPT-OSS-20B-Code`-BF16-i1-GGUF
failed: model does not support tools

# `qwen3:14b`
> 12 GB    100% GPU           40960
failed: found an incorrect cause in 2 minutes

# KyleHessling1/`Qwopus-GLM`-18B-Merged-GGUF
> 12 GB    100% GPU           65536
found a different cause that may be more correct in 6 minutes
failed: confronted with the mistral analysis, it changed its mind

# unsloth/`GLM-4.7-Flash`-REAP-23B-A3B-GGUF
> 14 GB    1%/99% CPU/GPU     65536
failed: found an incorrect cause in 4 minutes

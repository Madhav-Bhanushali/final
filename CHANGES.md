# Changelog — benchmark harness & catalog fixes

Recent changes to `test.sh` (the loan-collection benchmark harness). Each commit is pushed to `server/main`; on the server run `git pull && bash test.sh`.

## Latest changes (this commit)

### 1. Prompt rewritten (fixes irrelevant / meta responses)
The old 14-rule prompt made small models reply *about the task* ("Sure, I understand the instructions. Here's the message you could send back...") or produce corporate boilerplate. The new prompt is short and direct:

- Declares the assistant's role, reference date, payment window (Aug 12–19) and pending amount (INR 25,000).
- Gives 7 concrete response rules that map 1:1 to the benchmark categories (confirm in-window dates, ask for a date ≤ Aug 19 when late, clarify vague/conflicting dates, redirect unrelated talk, stay polite on frustration, never invent penalties).
- Ends with `Customer message:` and the customer's actual message is appended verbatim (no shouting "CUSTOMER MESSAGE:" label).
- Removed "Respond ONLY with...", "Do not include JSON/markdown...", "Never expose chain-of-thought" meta-instructions that triggered meta-commentary.

Validated locally on `f3-7b` (was generic boilerplate → 10/10 PASS with relevant payment replies) and `ternary-8b` (10/10 PASS).

### 2. No context leak from previous chats (cache_prompt off by default)
`cache_prompt` is now **off by default**, so every test starts from a clean KV slot and the model cannot pick up context from an earlier customer chat. Add `--cache-prompt` to opt back into prefix reuse.

### 3. Catalog trimmed
- Removed `ds-r1-1.5b` (returned empty/looping outputs). Removed earlier: `bitnet-2b`, `f3-1b`, `f3-3b`, `f3-10b`, `mistral-7b`.
- Current catalog: `f3-7b llama3.1-8b qwen3-8b gemma3-12b gemma-3b ternary-8b`.

### 4. Ternary Bonsai server failure fixed
The old link pointed at `prism-ml/Ternary-Bonsai-8B-gguf` `Q2_0.gguf`, whose custom g128 quant failed to load on the llama.cpp fork (`tensor 'output_norm.weight' has offset ...`). The catalog now uses the llamacpp-compatible pack:

- Repo: `Minarut/Ternary-Bonsai-8B-GGUF-llamacpp-compatible`
- File: `Ternary-Bonsai-8B-TQ2_0.gguf` (native ternary TQ2_0, loads cleanly on the fork build — verified locally, 10/10 PASS)

The corrupt `models/standard/Ternary-Bonsai-8B-Q2_0.gguf` is removed locally; the new file auto-downloads on the server.

### 5. qwen3 reasoning disabled properly
`qwen3-8b` (and `ternary-8b`, same architecture) emit a `thinking` chain into `reasoning_content`, leaving `content` empty. Instead of the fragile "Do not think step by step" text hack, the request now sends `chat_template_kwargs: {"enable_thinking": false}`, which the qwen3 chat template honors by injecting an empty think marker block so the model answers directly.

### 6. UTF-8 robustness
Python output/file handling uses `encoding="utf-8"` and reconfigures stdout — models that emit the `₹` (U+20B9) character no longer crash the Windows-local run (`UnicodeEncodeError: cp1252`).

## Earlier changes (already on server)

- **Hallucinated-turn stopping**: every model's `stop[]` includes its real end-of-turn token plus the bare `"\nuser\n"`/`"\nassistant\n"` markers, so models that emit raw `user:`/`assistant:` lines instead of templated tokens halt cleanly at their real reply.
- **Coherence gate**: flags fabricated user/assistant/system turns, leaked prompt fragments, unsafe responses, template artifacts, and JSON/markdown output.
- **`start_server` fix**: status echoes to stderr so `$server_pid` is never contaminated.
- **In-model threading**: `THREADS` auto-detect, `-t/-tb`, `-b 4096 -ub 2048`, 60s default timeout, deterministic ports. No multi-model concurrency — parallelism is inside each single model server.
- **gemma-3b** added (`arcee-ai/gemma-3b-it-expanded`, GGUF from `mradermacher/gemma-3b-it-expanded-GGUF`).
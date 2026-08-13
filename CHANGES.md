# Changelog — benchmark harness & catalog fixes

Recent changes to `test.sh` (the loan-collection benchmark harness). Each commit is pushed to `server/main`; on the server run `git pull && bash test.sh`.

## Latest changes (this commit)

### 1. Output is plain text only — no JSON anywhere
The harness no longer writes `benchmark_results.json` / `.jsonl` / `.csv` and never dumps the raw OpenAI JSON envelope. The console shows **only the bot's reply text** (plus the model/test headers), and everything is recorded in a human-readable `benchmark_results.txt`:

```
MODEL: f3-7b
TEST: NORMAL_01 (Normal Payment)
User: I can pay on August 15.
Result: PASS  Coherent plain-text reply
Reply:
Thank you very much for confirming that we can proceed as planned!
```

### 2. System prompt from `sp.txt`
The prompt is read from `./sp.txt` so it can be edited without touching the script. A copy of the same prompt is embedded in `test.sh` as a fallback if the file is missing. Unicode dashes are normalized to ASCII hyphens on load so the request JSON is always valid (a raw `–` byte broke older servers).

### 3. Faster: cache_prompt ON by default
`cache_prompt` is now **on**: the shared prompt prefix stays in the KV cache, so each new test only evaluates its own customer message. Measured locally: prompt eval dropped from 5.4s → 0.19s per test. Verified **no cross-test context leakage** (the full prompt is always re-sent; the cache is only a speed shortcut). Pass `--no-cache-prompt` to force full re-evaluation.

### 4. Bounded generation
Per-model `Predict` capped at 64 tokens (was 128/256). Replies are 1-3 sentences, so the cap only prevents runaway generation — it does not cut off valid replies.

### 5. HTTP 000 no longer masks valid replies
curl can exit `000` (socket quirk) even after the server fully generated a reply and wrote it to the response file. The harness now always extracts `choices[0].message.content`, displays just that text, and grades the actual reply instead of auto-failing as "Timeout". Raw JSON is never printed.

### 6. Max threading
Server launched with `-t <all cores> -tb <all cores> -b 4096 -ub 2048`, single slot, so all parallelism happens inside the one model server.

## Earlier changes (already on server)

- **Prompt quality**: rewritten from the long 14-rule block to a short, direct prompt mapped 1:1 to the test categories (fixes "Sure, I understand the instructions..." meta-replies and corporate boilerplate). Validated locally: f3-7b and ternary-8b both 10/10.
- **Catalog trimmed**: removed `ds-r1-1.5b` (and earlier `bitnet-2b`, `f3-1b`, `f3-3b`, `f3-10b`, `mistral-7b`). Current: `f3-7b llama3.1-8b qwen3-8b gemma3-12b gemma-3b ternary-8b`.
- **Ternary Bonsai fixed**: switched to the llamacpp-compatible pack `Minarut/Ternary-Bonsai-8B-GGUF-llamacpp-compatible` → `Ternary-Bonsai-8B-TQ2_0.gguf` (the prism `Q2_0` g128 file failed to load on the fork).
- **qwen3 reasoning disabled** via `chat_template_kwargs: {"enable_thinking": false}` instead of a text hack.
- **Hallucinated-turn stopping**: every model's `stop[]` includes its real end-of-turn token plus bare `"\nuser\n"`/`"\nassistant\n"` markers.
- **Coherence gate**: flags fabricated user/assistant/system turns, leaked prompt fragments, unsafe responses, template artifacts, and JSON/markdown output.
- **`start_server` fix**: status echoes to stderr so `$server_pid` is never contaminated.
- **gemma-3b** added (`arcee-ai/gemma-3b-it-expanded`, GGUF from `mradermacher/gemma-3b-it-expanded-GGUF`).
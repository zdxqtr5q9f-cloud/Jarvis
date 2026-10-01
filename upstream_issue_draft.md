# Draft issue for isair/jarvis (not sent)

**Title:** Voice latency (~20 s/turn), audio queue overflow, fast-model reloads, and unrestricted `localFiles` — measurements on Apple Silicon (Ollama)

**Environment:** macOS 26, MacBook Pro M5 Max (128 GB), Jarvis desktop app 2.3.0 (asset id 575514360; code paths below checked against the 2.2.2 source, config schema v3), Ollama 0.32.14, chat `qwen3.8:27b` (GGUF; also tried the `-mlx` build), fast `gemma4:e2b`, Whisper `medium` (CPU, int8, 18 threads), Piper `ru_RU-irina-medium`, language Russian.

Everything below was measured from the Jarvis activity log plus the Ollama server log. Happy to split into separate issues if preferred.

## 1. Prefix cache never hits across user turns (cost: 5–13 s per turn)

The main chat request is ~3.4–3.6k prompt tokens. Ollama log (MLX runner):

```
cache miss  total=3367 matched=0  cached=0 left=3367     (turn 1)
cache hit   total=3576 matched=31 cached=2 left=3574     (turn 2: effectively a miss)
cache hit   total=4020 matched=3611 cached=3611 left=409 (tool-call step inside the same turn: works)
```

So the prompt diverges after ~31 tokens on every new user turn (something dynamic seems to be placed near the top of the system prompt: time, profile, memory, tool list). Prefill speed measured on the same machine: MLX build 259 tok/s (≈13 s for 3.4k tokens), GGUF 643 tok/s (≈5 s). Moving dynamic content to the end of the prompt would make the stable prefix cacheable and remove most of this cost.

## 2. Fast model is reloaded 2× per turn because `num_ctx` differs between call sites

`llm/ollama.py`: default `num_ctx=4096`, main agentic chat uses 8192, the warmup probe sets none. Ollama restarts the llama-server runner whenever `num_ctx` changes:

```
11:24:39 starting llama-server … -c 8192   (intent judge)
11:24:46 starting llama-server … -c 4096   (planner / other fast-tier call)
11:25:09 … -c 8192
11:25:16 … -c 4096
```

≈1.3 s per restart, 2–3 restarts per turn for `gemma4:e2b`. Using one `num_ctx` for all calls to the same model (or making it configurable) would avoid it. (The MLX runner ignores `num_ctx`, so it only affects GGUF/llama-server.)

## 3. Audio capture queue overflows (`maxsize=64`, hard-coded) and the assistant is deaf while it works

`listener.py`: `self._audio_q = queue.Queue(maxsize=64)`. Log during one turn:

```
Audio capture: queue full; 147 blocks dropped.   (intent → plan)
Audio capture: queue full; 780 blocks dropped.   (LLM generation, ~15 s)
Audio capture: queue full; 1569 blocks dropped.  (37 s turn with a tool call)
```

Also 1–40 blocks dropped every ~5 s while idle with Whisper medium on CPU (it was 50–326 with large-v3). The consumer thread seems blocked during LLM calls, so wake word / "stop" cannot be heard while the assistant is thinking, and the start of the next utterance is clipped (we observed a lost first word and a Whisper hallucinated tail). A larger/configurable queue, or doing LLM work off the audio thread, would help.

## 4. Web search fallback is ineffective for non-English users

- DuckDuckGo returns the bot-challenge page on every query for hours ("DuckDuckGo served a bot-challenge page").
- The Wikipedia fallback logs `Searching Wikipedia (en) for '<Russian query>'` and finds nothing, although the setup wizard text says it uses the host matching the language Whisper detected. There is no config key to force the language (`config.language` does not seem to feed it).
- Suggestion: pass `cfg.language` (or the query's language) to the fallback, and consider supporting a self-hosted SearXNG endpoint as a local/private option.

## 5. Planner chooses `webSearch` for requests that need no search

"Tell me a short joke" (Russian) → `Plan: 2 step(s) — webSearch query='короткие смешные шутки на русском'`, then a "search blocked, try later" reply, even with a system prompt that says to answer general questions directly and not to search. The plan comes from a separate call that does not see/obey `system_prompt`. Disabling `planner_enabled` fixed it and saved ~2 s per turn.

## 6. `localFiles` can write/append/delete anywhere in `$HOME` with no confirmation

`tools/builtin/local_files.py` supports `list/read/write/append/delete`; the only guard is `_resolve_safe` (path must be inside the home directory). I could not find a confirmation step or a config switch to disable the tool or make it read-only. For setups where voice recognition errors or the hot window can trigger an unintended command, a read-only mode and/or an allowlist of folders (and a confirmation requirement for write/delete) would be valuable. Note it was also selected for a "search my notes" request instead of an installed Obsidian MCP search tool, and then listed ~8 directories.

## 7. Minor

- `voice_device` as a numeric index breaks when AirPods connect (`Error opening InputStream: Invalid number of channels [PaErrorCode -9998]`); using a name substring works (supported in code but not mentioned in the UI/docs).
- Native crash 7 minutes after the 2.3.0 update: `-[NSEvent clickCount]` NSAssertionHandler (EXC_CRASH, Abort trap: 6) from Qt on a non-mouse event; `.ips` available.
- Qwen3.x thinking is correctly disabled by default (`think: false`); good.

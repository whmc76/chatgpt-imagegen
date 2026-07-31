---
name: chatgpt-imagegen
description: Generate or reference-edit raster images through the user's ChatGPT subscription with a local CLI that has explicit total and stall timeouts, progress events, bounded concurrency, and deterministic file saving. Use when Codex must create PNG/WebP assets, when the built-in imagegen tool is unavailable or remains unresponsive, when the user asks to use ChatGPT for image generation without an API key, or when reliable batch generation is more important than an inline image tool call. Do not use for public image-generation services, strict transparent backgrounds, guaranteed exact dimensions, or tasks better expressed as SVG/HTML/Mermaid.
---

# ChatGPT ImageGen

Generate images with the bundled CLI and verify the saved file. Prefer the tested `codex` backend for reliability; use the browser backend only when its prerequisites already pass.

## Preflight

1. Resolve this skill directory from the loaded `SKILL.md`; never guess it.
2. Run the read-only doctor before the first generation in a thread or after any backend failure:

   - Windows: `py -3 "<skill-dir>\scripts\chatgpt-imagegen" doctor`
   - macOS/Linux: `python3 "<skill-dir>/scripts/chatgpt-imagegen" doctor`

3. Require Python 3.10+ and a ChatGPT OAuth token from `codex login`. Never read, print, copy, or expose the token in `~/.codex/auth.json`.
4. Treat `web` as ready only when doctor reports `chrome-use` and a logged-in browser as ready. Do not install `chrome-use` or a browser extension without explicit user approval.

## Backend selection

- Use `--backend codex` by default. It is dependency-free, supports up to four internal slots, and has been live-verified on Windows. It consumes the Codex usage bucket.
- Use `--backend web` only when the user prefers ChatGPT conversation quota and doctor says the browser path is ready. It is serial and depends on the user's logged-in Chrome plus `chrome-use`; it does not use the Codex in-app browser.
- Use `--backend auto` only when the user explicitly wants web-first fallback behavior. Auto may change the quota bucket based on local readiness.
- Never retry a submitted web request on `codex` automatically; the first request may still finish and a duplicate could consume two quotas.

## Generate

1. Turn the request into a concise prompt describing subject, composition, style, mood, text, and exclusions. Preserve a specific user prompt instead of embellishing it.
2. Choose an output path inside the current workspace. Use the user's path when provided; otherwise use the project's existing asset directory, or `outputs/` in a projectless task. Do not overwrite an existing file unless requested.
3. Run one command with bounded waits:

   - Windows:
     `py -3 "<skill-dir>\scripts\chatgpt-imagegen" "<prompt>" --backend codex --size <size> --timeout 300 --stall-timeout 90 --quiet -o "<output>"`
   - macOS/Linux:
     `python3 "<skill-dir>/scripts/chatgpt-imagegen" "<prompt>" --backend codex --size <size> --timeout 300 --stall-timeout 90 --quiet -o "<output>"`

4. If the shell yields a running process, keep polling that exact process every 30-60 seconds. Report meaningful phase changes and never start a duplicate generation merely because the first call is still running.
5. On success, require exit code 0, a non-empty output file, and successful image decoding. Inspect it visually when `view_image` is available.
6. If the result is clearly wrong, make at most one targeted prompt correction. Do not spray variants without the user's approval.

Supported verified sizes are `1024x1024`, `1536x1024`, and `1024x1536`. The subscription backend may return a different square dimension even when `1024x1024` is requested; inspect and report the actual dimensions when exact geometry matters.

## Reference-guided generation

Pass up to four references with repeated `--ref "<path-or-url>"` arguments. Treat local files as data transmitted to ChatGPT; proceed only when the user's request authorizes uploading those specific files. This is generative reference editing, not pixel-preserving local retouching.

## Failure recovery

- The CLI already retries one incomplete Codex stream, transport interruption, or transient 5xx response inside the original total timeout budget. Do not immediately add another outer retry after that retry is exhausted.
- `stalled`: retry once only when no rate-limit message appeared, using `--timeout 420 --stall-timeout 150`. If it stalls again, stop and report the last phase.
- Total timeout after active progress: retry once with `--timeout 420`; do not retry indefinitely.
- HTTP 429 or account-limit banner: stop. Do not loop or switch backends immediately.
- HTTP 401/403: allow the CLI's one token refresh attempt. If it still fails, ask the user to run `codex login`.
- `no image returned`: revise the prompt once to explicitly say `Use the image_generation tool to render ...`.
- Missing `chrome-use`: keep using `codex`; mention the optional browser setup only if the user asks to avoid Codex quota.

## Throughput and limits

- Default to one request. For an explicitly requested batch, use at most two concurrent `codex` requests unless the user asks for higher throughput; the bundled semaphore prevents process collisions.
- Do not generate more than three exploratory variants without permission.
- Do not promise transparent output, `quality=high`, or exact output dimensions. Use the official Images API or another appropriate workflow when those are hard requirements.
- Do not use a personal ChatGPT subscription to power a public or customer-facing generation service.

## Provenance and maintenance

The bundled MIT-licensed CLI is derived from `leeguooooo/chatgpt-imagegen` 0.21.2 at commit `f47816d5560b1052f47a3cbfbe1a7c20aa2638a9`. This fork adds a Windows-safe concurrency lock using `msvcrt.locking`, retains `fcntl.flock` on POSIX, and retries one transient Codex stream interruption inside the original total timeout budget. Do not replace the script from upstream without reapplying or confirming equivalent fixes, then rerun doctor, a Windows lock-contention check, and a live generation.

---
title: "OpenWhispr on Windows with OpenRouter STT and transcript cleanup"
description: "A small FastAPI proxy that lets OpenWhispr talk to OpenRouter STT on Windows, with optional transcript cleanup and quiet background startup."
pubDate: 2026-05-16
updatedDate: 2026-05-16
tags: ["python", "fastapi", "openrouter", "windows", "speech-to-text"]
draft: false
language: "en"
---

## Context

I wanted a practical local speech-to-text path for OpenWhispr on Windows without depending on OpenAI's own transcription endpoint.

OpenWhispr can already send audio to an OpenAI-compatible API:

- `POST /v1/audio/transcriptions`
- `multipart/form-data`
- file upload plus model and a few optional fields

The problem was that the target model I wanted to use was on OpenRouter:

- STT model: `qwen/qwen3-asr-flash-2026-02-10`

That endpoint does not take multipart uploads. It expects JSON with base64-encoded audio. So the smallest useful thing was a local adapter that sits between the app and OpenRouter.

The repository is public here: [nayutalienx/openrouter-stt-proxy](https://github.com/nayutalienx/openrouter-stt-proxy).

## Problem

The protocol mismatch was the easy part to describe and the annoying part to automate.

OpenWhispr sends:

- multipart file upload;
- OpenAI-style route shape;
- a model name that the app expects to validate;
- optional language, prompt, temperature, and response format fields.

OpenRouter STT expects:

- `POST /api/v1/audio/transcriptions`;
- JSON body;
- `input_audio.data` as base64;
- `input_audio.format` inferred from the filename extension.

That already required a translation layer. But a raw STT result is often not the output I actually want from dictated text. For chat-style workflows, the useful result is usually:

- correct punctuation;
- fixed casing;
- obvious ASR errors cleaned up;
- minimal rewriting;
- no second manual editing pass.

So the proxy needed a second stage: optional cleanup after transcription, while still keeping the external interface stable for OpenWhispr.

## What I Tried

The first version stayed deliberately small:

- FastAPI;
- Uvicorn;
- `httpx` for upstream calls;
- `.env` configuration through `python-dotenv`;
- in-memory audio handling only;
- local bind on `127.0.0.1:8787`.

The request flow became:

1. Receive multipart audio from OpenWhispr.
2. Read the bytes without writing them to disk.
3. Infer format from the filename extension.
4. Base64-encode the payload.
5. Send the STT request to OpenRouter.
6. Normalize the response back to `{"text": "..."}`.

That solved the compatibility layer, but I did not stop there.

The second pass added optional transcript cleanup with:

- cleanup model: `deepseek-v4-flash` directly on DeepSeek API, or `deepseek/deepseek-v4-flash` through OpenRouter;
- low temperature;
- configurable failure policy;
- a conservative `punctuation` mode for grammar and punctuation fixes with minimal paraphrasing.

The current setup uses:

- OpenRouter for STT;
- DeepSeek API directly for grammar cleanup.

The proxy now supports:

- optional local bearer auth;
- `GET /health`;
- `GET /v1/models`;
- `POST /debug/cleanup` for testing the editor pass without uploading audio;
- quiet startup on Windows through `pythonw.exe`.

## What Failed

There were a few small but real dead ends.

The first one was FastAPI response typing. Returning different response classes from one endpoint is fine at runtime, but the initial type annotation tripped FastAPI's response model generation. The fix was to stop pretending that route-level response typing was useful there and declare `response_model=None`.

The second one was Windows process behavior. A scheduled task that launches `powershell.exe` and then `uvicorn` works, but it still feels like a console app. That is acceptable for a quick test, not for something that is supposed to sit quietly in the background every time the machine starts.

The current startup path uses `pythonw.exe` plus a tiny launcher script instead. That keeps the proxy alive without an extra visible console window and still writes logs to a local file.

The third one was not in the proxy itself but in testing. Debugging Russian cleanup prompts through Windows shells is a fast way to run into encoding edge cases. The live endpoint was fine; the surrounding test invocation needed more care than the actual API logic.

## Current Direction

The current version is intentionally boring in the good sense:

- one local HTTP proxy;
- one STT upstream call;
- one optional cleanup call;
- straightforward PowerShell and batch launchers;
- one background entrypoint for quiet Windows startup.

The useful part is not technical novelty. It is that the whole thing now behaves like a normal local integration instead of a fragile demo:

- OpenWhispr still talks to `/v1/audio/transcriptions`;
- OpenRouter STT stays behind the adapter;
- cleanup can use either OpenRouter or DeepSeek directly;
- cleanup is controlled through environment flags;
- failures can fall back to the raw transcript instead of breaking the whole dictation flow;
- secrets stay out of Git.

The repository also includes the operational details that are usually skipped and then rediscovered later:

- `.env.example`;
- local auth option;
- health and cleanup test scripts;
- README for Windows users;
- scheduled startup flow.

## Open Questions

- Whether the cleanup pass should eventually support chunked editing for very long dictations instead of skipping once the text exceeds the configured input threshold.
- Whether a second cleanup prompt should exist for languages other than Russian, or whether language-specific editing is better left disabled by default.
- Whether OpenWhispr users would prefer the proxy to expose more than one cleanup profile through separate models or separate endpoints.

Those are useful next problems, but they are not required for the current setup to be productive.

## Next Step

The immediate next step is simple: use it for real dictation over a few weeks and see where the rough edges are.

If the current behavior holds up, I would rather keep it small than turn it into a generic platform. Right now the value is exactly that it is a narrow, understandable tool:

- OpenWhispr in front;
- OpenRouter STT behind;
- optional cleanup after transcription, now with direct DeepSeek support for the editor pass;
- quiet Windows background startup;
- no unnecessary infrastructure.

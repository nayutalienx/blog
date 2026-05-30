---
title: "ClipGloss Overlay: global hotkey translation on Windows"
description: "A small Windows-first Python tool that grabs selected text, sends it to DeepSeek or OpenRouter, and shows the translation in a fast desktop overlay."
pubDate: 2026-05-30
updatedDate: 2026-05-30
tags: ["python", "windows", "translation", "tkinter", "deepseek", "openrouter"]
draft: false
language: "en"
---

## Context

I wanted a very narrow desktop tool:

1. Select text in any Windows app.
2. Press one shortcut.
3. See the translation immediately without opening a browser tab or pasting into a chat UI.

The repository is public here: [nayutalienx/clipgloss-overlay](https://github.com/nayutalienx/clipgloss-overlay).

The target use case is boring on purpose. It is for quick reading help while working in other apps, not for building a full translation workstation.

## Problem

The main problem was not translation quality. That part is easy once an API call exists.

The annoying part is the desktop behavior around it:

- capture the current selection from arbitrary Windows apps;
- trigger from a global hotkey;
- keep the interaction fast enough to feel local;
- show the result without a heavy GUI shell;
- avoid turning a small helper into another application window that has to be managed.

There is also a practical integration constraint. Different users may want different providers, so the tool should not hard-code one model vendor. It should be able to call DeepSeek directly or go through OpenRouter without changing the basic workflow.

## What I Tried

The current version stays deliberately small:

- Python;
- `httpx` for API calls;
- `python-dotenv` for configuration;
- `pywin32` plus a Windows global hotkey registration;
- `tkinter` for the overlay itself.

The request flow is simple:

1. Register a global shortcut.
2. When the shortcut fires, simulate `Ctrl+C`.
3. Read the selected text from the clipboard.
4. Send it to the configured translation model.
5. Show the result in a custom bottom-right overlay card.

The current app supports:

- DeepSeek API directly;
- OpenRouter as an alternative provider;
- configurable source and target language;
- background startup on Windows;
- optional copying of the translated text back to the clipboard;
- a small stack of overlay notifications instead of one blocking dialog.

The useful part is that the whole thing lives in one small repository with straightforward scripts:

- `.env.example` for configuration;
- `run.ps1` and `run.bat` for foreground startup;
- `start-background.ps1` and a tiny background runner for quiet startup;
- one main `app.py` file instead of a multi-layer desktop framework.

## What Failed

The first rough edge was hotkey selection.

There is no universally safe global shortcut on Windows. If the default is too common, another app already owns it. If it is too obscure, it becomes hard to remember. The project now defaults to `Ctrl+3`, but the real solution is not the exact shortcut. The real solution is making it configurable and handling registration failure clearly.

The second rough edge was the clipboard.

Using the clipboard is the most practical way to capture selected text across unrelated desktop apps, but it is not perfectly clean. Some applications delay clipboard updates, some selections are not text, and restoring previous clipboard state is only partly reliable when the old content was non-text data.

The third one was UI scope. It is easy to overbuild this kind of tool:

- settings windows;
- tray menus;
- history panels;
- provider dashboards.

None of that was required for the first useful version. The overlay itself was enough.

## Current Direction

The current direction is to keep ClipGloss Overlay narrow and reliable.

The app now behaves like a small background utility:

- start it once;
- leave it running;
- select text anywhere;
- press the shortcut;
- get the translated result as an overlay.

That is enough for the tool to be useful during normal reading or document work.

The design is intentionally conservative:

- no large desktop framework;
- no persistent translation database;
- no browser dependency at runtime;
- no secrets in the repository;
- no attempt to support every platform.

It is Windows-first because that is the environment I wanted to use it in, and the implementation matches that constraint directly instead of pretending to be cross-platform.

## Open Questions

- Whether the clipboard capture should eventually fall back to app-specific strategies for editors that do not behave well with synthetic `Ctrl+C`.
- Whether the overlay should support manual copy buttons or pinned results, or whether that would just make the tool slower and heavier.
- Whether the provider abstraction should stay limited to translation models that already fit an OpenAI-style chat completion flow.
- Whether a later version should add a second translation mode for more literal output versus more natural localized wording.

These are real follow-up questions, but they are not blocking for the current tool.

## Next Step

The next useful step is not a rewrite. It is usage.

I want a few weeks of normal daily use to answer the practical questions:

- Does `Ctrl+3` stay comfortable?
- Which apps fail to expose selected text cleanly?
- Is the overlay duration long enough?
- Is direct DeepSeek enough, or does OpenRouter become the default path more often?

If the current shape holds up, I would rather keep the project small than expand it into a generic desktop translation suite. Right now its value is exactly that it does one thing with very little ceremony.

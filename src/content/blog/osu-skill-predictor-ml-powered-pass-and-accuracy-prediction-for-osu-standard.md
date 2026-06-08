---
title: "osu-skill-predictor: ML-powered pass & accuracy prediction for osu! standard"
description: "Classical ML pipeline that predicts pass probability and accuracy for any player-vs-beatmap pairing in osu!, shipped as a real-time overlay."
pubDate: 2026-06-08
updatedDate: 2026-06-08
tags: ["ml", "osu", "python", "scikit-learn", "fastapi", "portfolio"]
draft: false
language: "en"
---

## Context

I play osu! standard. Like every player, I constantly wonder: can I pass this map? What accuracy would I get? osu! gives you a star rating, but that is a global difficulty metric — it says nothing about your current skill relative to it. A 5-star map is trivial for a 4-digit player and impossible for a 6-digit.

I wanted a tool that takes a handful of numbers — my pp, my accuracy, the map's star rating, BPM, AR, OD, CS, length, the mods I have on — and spits out a pass probability and an expected accuracy. That is the entire project.

The secondary goal was to build a complete, honest, production-shaped ML portfolio piece. No deep learning, no cloud dependencies, no overselling. Just the full lifecycle: data collection -> cleaning -> feature engineering -> model comparison -> serialization -> API -> real-time inference.

## Problem

Given a player-beatmap pair with mods, predict two things:

1. **Pass probability** (binary classification: pass or fail)
2. **Accuracy percentage** (regression: 0-100)

Constraints:

- No replay data, no sequence modeling, no computer vision. Only tabular features from API summaries.
- Must run locally, offline after initial dataset collection.
- Must work in real time during gameplay as an overlay.
- Classical ML only (scikit-learn). Intentionally no neural networks.

## What I Tried

### Data collection

Built a 1340-line collection script against the osu! API v2. The sampling strategy: top 100 countries -> 100 players per country -> truncated-normal sampling of local ranks -> 30 recent + 20 best scores per player -> deduplicate. This gave 184,229 cleaned rows across 9,999 unique players and 35,261 beatmaps. JSONL for resumable checkpoints, flattened to CSV for training.

### Feature engineering

The most impactful feature is **star comfort mapping**. For each pp bin in the training data, I computed the median star rating of passed maps. This gives `player_star_comfort_estimate` — the star level a player is "comfortable" with, inferred from their pp. The key interaction feature is **star gap**: `beatmap_star_rating - player_star_comfort_estimate`. Positive gap -> map is above comfort zone.

Other features: mod flags (HD, HR, DT/NC parsed from raw mod strings), length bucket (short/medium/long based on hit length), and raw numeric stats (BPM, AR, OD, CS, player accuracy, play count).

### Model comparison

Not just a single train/test split. Used **grouped holdout by `user_id`** to prevent data leakage — no player appears in both train and test. Also ran **grouped cross-validation** (StratifiedGroupKFold for classification, GroupKFold for regression).

Compared 3 classifiers and 3 regressors. The winners:

| Model | Type | CV Metric |
|---|---|---|
| RandomForestClassifier | Pass prediction | PR AUC: 0.995 |
| HistGradientBoostingRegressor | Accuracy prediction | MAE: 3.46 |

The classifiers all performed similarly well; RandomForest won on PR AUC by a narrow margin. For regression, HistGradientBoosting clearly outperformed RandomForest and Ridge.

### Real-time pipeline

Wired up the [tosu](https://github.com/tosuapp/tosu) memory reader to poll osu! live state (current beatmap, player stats, mods), enrich missing fields from the osu! API v2 with caching (15 min TTL for beatmaps, 24h for users), feed features into the loaded models, and display results in a tkinter always-on-top overlay window.

A single-page web UI (HTML/JS served by FastAPI) handles settings, status display, and shutdown. Everything packaged as a standalone Windows exe via PyInstaller, with tosu.exe bundled in.

## What Failed

- **Dimensionality of real data**. The dataset is a sample — ~10k players from a curated country-based crawl. It does not cover the full osu! player population. Predictions for extreme outliers (very new players, very high-ranked players) are less reliable than for mid-range players.
- **Snapshot bias**. Player features (pp, accuracy, play count) are current snapshots, not historical values at the time of each play. This adds noise for older scores.
- **Mod interactions are simplistic**. Only HD, HR, DT/NC are modeled as binary flags. Mod combinations (HDHR, HDDT, etc.) are not explicitly encoded, and things like Flashlight or EZ are ignored entirely.
- **No uncertainty calibration**. The classifier outputs probabilities, but I did not run proper calibration (Platt scaling, isotonic regression). The raw probability numbers are reasonably well-ordered but not strictly calibrated.

## Current Direction

The project is in **feature-complete MVP state**, tagged as `v0.1.0`. It does exactly what I set out to do and ships as a working standalone app. I am treating it as a done portfolio piece.

Architecture:

```
osu! client -> tosu (memory reader) -> LiveService (FastAPI) -> PredictionService
                                          |                        |
                                     Web UI (settings)      Canonical Models
                                          |                  (joblib artifacts)
                                     Tkinter overlay
                                    (pass% + acc% on screen)
```

## Open Questions

- Would the star comfort mapping benefit from being computed per-mod combination instead of globally?
- Is it worth adding a calibration step to the classifier probabilities?
- Could a lightweight online adaptation mechanism (exponential moving average of recent plays) improve predictions without full retraining?
- Does the overlay mode actually help players in practice, or is it just a fun gadget?

## Next Step

This project is done for now. If I return to it, the first thing I would tackle is proper probability calibration and expanding mod support beyond the basic three. But honestly, the most useful next step is just using it while playing and seeing if the predictions feel right.

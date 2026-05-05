# AGENTS.md

This file provides guidance to Codex and other coding agents when working with code in this repository. Use `AGENTS.md` for active instructions; `CLAUDE.md` files are legacy migration references only.

## What This Is

Trunk Reporter is a monorepo for a P25 trunked radio monitoring system. It ingests data from [trunk-recorder](https://github.com/robotastic/trunk-recorder) instances and provides a full stack: backend API, web dashboards, Discord bot, speech recognition, and audio quality research.

All repos live under the GitHub org at https://github.com/orgs/trunk-reporter/repositories. The cross-repo roadmap is at https://github.com/orgs/trunk-reporter/projects/1.

## Mission

Trunk Reporter turns raw trunk-recorder output into a richer radio monitoring, analysis, and transcription platform. The goal is not just a call log: users should be able to understand live and historical radio activity across systems, sites, talkgroups, units, affiliations, aliases, encryption changes, and transcriptions.

The core audience is trunk-recorder hobbyists, power users, public-safety radio analysts, and ML researchers. Casual users need setup simplicity, a reliable call list, and inline audio playback. Power users need unit tracking, affiliation analysis, multi-site P25 merging, investigative cross-reference tools, and transcription that works on real radio audio.

`tr-engine` is the center of gravity. It normalizes ingest from MQTT, file watch, HTTP upload, and plugin sidecars into a PostgreSQL-backed API, SSE stream, and live audio surface. `tr-dashboard` is the primary user-facing UI. The DVCF/AVCF plugins and SymbolStream capture cleaner source data from trunk-recorder. `imbe_asr`, `qwen3-asr-server`, and `p25_vocoder` push transcription and audio-quality research forward. `tr-stack`, `tr-docker`, and deployment docs exist to make the whole system practical to run.

Engineering priorities follow from that mission:

- Keep `tr-engine` reliable; data model, identity resolution, ingest, auth, and API regressions affect everything downstream.
- Keep setup and deployment simple enough for real hobbyist systems.
- Make multi-site P25 merging and system/site identity resolution correct.
- Improve transcription quality, especially by avoiding audio reconstruction artifacts through DVCF and IMBE-ASR.
- Keep docs accurate as auth, ingest modes, deployment paths, and ASR providers evolve.
- Treat Discord and GitHub feedback as the work queue, but keep fixes scoped, tested, and tied back to user impact.

## Architecture Overview

```
trunk-recorder (external)
  │
  ├─[MQTT]──────────────────────────────────────────────────────────┐
  ├─[filesystem/audio]───────────────────────────────────────────────│
  ├─[tr-plugin-dvcf]── .dvcf files (P25 raw codec frames) ──────────│
  └─[tr-plugin-avcf]── .avcf files (analog audio+metadata) ─────────│
                                                                     ▼
                                                              tr-engine (Go)
                                                           REST API + SSE + DB
                                                                     │
                    ┌────────────────────────────────────────────────┤
                    │                     │                          │
                    ▼                     ▼                          ▼
              tr-dashboard           tr-bot                   imbe_asr / qwen3-asr-server
          (React SPA frontend)  (Discord bot, RAG)          (transcription backends)
```

### Deployed Instance

Production runs on `gerty` (SSH: `ssh root@gerty`). See `tr-engine/AGENTS.md` for full deployment details.

| Service | URL |
|---------|-----|
| tr-dashboard (UI) | https://tr-dashboard.luxprimatech.com |
| tr-engine API | https://tr-engine.luxprimatech.com |
| tr-engine (Tailnet, preferred for dev) | http://gerty.pizzly-manta.ts.net:8080 |

## Subprojects

Each subproject may have its own `AGENTS.md` guidance; read it before working in that directory.

### tr-engine (Go)
Backend: MQTT ingestion, REST API (80+ endpoints), SSE event streaming, live audio via WebSocket, PostgreSQL, pluggable transcription, built-in web dashboards.

- **Build:** `bash build.sh` (injects version via ldflags)
- **Run:** `./tr-engine.exe` (auto-loads `.env`)
- **Health check:** `curl http://localhost:8080/api/v1/health`
- **API spec:** `openapi.yaml` is the source of truth — always update it when adding/changing endpoints
- **Schema:** `schema.sql` auto-applied on first run; incremental changes go in `internal/database/migrations.go`
- See `tr-engine/AGENTS.md` for full details on data model, ingest pipeline, auth modes, and deployment

### tr-dashboard (TypeScript/React)
Frontend SPA: React 19, TypeScript strict, Vite 7, Tailwind CSS v4, shadcn/ui, Zustand v5, React Router v7.

- **Dev server:** `npm run dev` (proxies `/api` to tr-engine)
- **Build:** `npm run build`
- **Type check:** `npm run lint`
- **Regen API types:** `npm run api:generate`
- No tests configured — lint is the primary quality check
- See `tr-dashboard/AGENTS.md` for store architecture, routing, and radio domain model

### tr-bot (TypeScript/Node.js)
Discord support bot using Claude (Haiku for routine, Sonnet for complex). RAG knowledge base from GitHub repos/issues/discussions.

- **Dev:** `npm install && npm run build && npm start`
- **Docker:** `docker compose up -d`
- See `tr-bot/README.md` for env vars and knowledge base config

### tr-plugin-dvcf (C++)
trunk-recorder plugin that taps `voice_codec_data()` to capture raw IMBE/AMBE frames before vocoder synthesis. Saves `.dvcf` sidecar files and/or publishes over MQTT. Digital calls only (P25, DMR, D-STAR, YSF).

- Format spec: `tr-plugin-dvcf/DVCF_SPEC.md` (SymbolStream v2 binary format)
- tr-engine's `handleDvcf` handler consumes these for IMBE ASR transcription

### tr-plugin-avcf (C++)
Analog companion to tr-plugin-dvcf. Wraps `.wav`/`.flac`/`.m4a` recordings with call metadata in the same SSSP v2 container format as `.dvcf`.

### symbolstream (C++)
SymbolStream Protocol v2 implementation — the binary container format used by both `.dvcf` and `.avcf` files.

### imbe_asr (Python/PyTorch)
ASR directly from P25 IMBE vocoder parameters — skips audio reconstruction entirely. 290M Conformer-CTC, 3.35% WER on LibriSpeech-IMBE with 5-gram KenLM.

- **Input:** 170-dim raw IMBE parameters per 20ms frame (not audio)
- **Inference:** `python -m src.inference --checkpoint checkpoints/best.pth --dvcf-file call.dvcf`
- **Watch mode:** `python -m src.inference --watch /path/to/dvcf/dir`
- Models on HuggingFace: `trunk-reporter/imbe-asr-*`
- See `imbe_asr/AGENTS.md` for full architecture, training pipeline, and data format

### qwen3-asr-server (Python/FastAPI)
OpenAI-compatible `/v1/audio/transcriptions` endpoint wrapping Qwen3-ASR 0.6B fine-tuned on P25 audio. Drop-in Whisper replacement. Supports word-level timestamps via Qwen3-ForcedAligner.

- **Start:** `./start.sh` or `python server.py`
- **Docker:** `docker compose -f docker-compose.gpu.yml up` (GPU) or `docker-compose.cpu.yml` (CPU)
- See `qwen3-asr-server/AGENTS.md` for config env vars and request flow

### p25_vocoder (Python/PyTorch)
Research project: neural vocoder replacing classical IMBE synthesis with Vocos (ConvNeXt backbone + ISTFTHead). 16kHz wideband output from 259-dim IMBE features. Two-phase training (reconstruction pretraining → GAN fine-tuning).

- See `p25_vocoder/AGENTS.md` for training commands and architecture

### tr-stack
Single `docker compose up` for the full P25 stack: trunk-recorder + tr-engine + tr-dashboard + imbe-asr + PostgreSQL + Mosquitto.

### tr-docker
Docker configuration variants for trunk-recorder builds.

## Data Flow: Transcription Pipeline

tr-engine supports four STT providers, selected via `STT_PROVIDER`:

| Provider | Input | Notes |
|----------|-------|-------|
| `whisper` | Audio (.m4a) | Self-hosted or cloud Whisper-compatible API |
| `elevenlabs` | Audio (.m4a) | ElevenLabs Scribe API |
| `deepinfra` | Audio (.m4a) | DeepInfra hosted Whisper |
| `imbe` | `.dvcf` codec frames | No audio reconstruction — tr-plugin-dvcf required |

When `STT_PROVIDER=imbe`, only `handleDvcf` enqueues transcription (not `handleAudio`). IMBE only covers P25 digital calls; analog calls on the same system get no transcription.

## P25 Radio Domain Model

The system/site hierarchy is central to the entire codebase:

- **System** = logical radio network, identified by `(sysid, wacn)` for P25 or `(instance_id, sys_name)` for conventional
- **Site** = recording point within a system (one trunk-recorder instance per site)
- Multiple sites monitoring the same P25 network auto-merge into one system in tr-engine

Talkgroups and units belong to systems (shared across all sites). Calls are tagged with `system_id` + `tgid`. Composite keys like `"system_id:tgid"` are used throughout the API and frontend stores.

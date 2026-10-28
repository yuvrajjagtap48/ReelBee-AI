# ReelBee AI

> Turn an educational idea into a reviewable, narrated vertical video.

ReelBee AI is a local TypeScript video-generation prototype. It plans storyboards with Gemini, creates speech with ElevenLabs, compiles captions and visuals with Remotion, and exports short-form videos in a consistent 9:16 format.

**Maintainer:** Yuvraj Jagtap
**Status:** Local prototype + Next.js UI foundation

> The video pipeline is usable locally. The web application is still a foundation: accounts, persistence, hosted rendering, and Instagram publishing are not implemented.

## Contents

- [Highlights](#highlights)
- [How it works](#how-it-works)
- [Requirements](#requirements)
- [Installation](#installation)
- [Configuration](#configuration)
- [Create a video](#create-a-video)
- [Review, revise, and resume](#review-revise-and-resume)
- [Fixture and studio workflows](#fixture-and-studio-workflows)
- [Web foundation](#web-foundation)
- [Testing and verification](#testing-and-verification)
- [Project structure](#project-structure)
- [Limitations](#limitations)
- [Documentation](#documentation)

## Highlights

- Gemini-assisted storyboard planning with schema validation
- Human review of storyboards and factual claims before narration
- ElevenLabs speech generation with timestamp alignment
- Deterministic Remotion rendering at **1080 × 1920**, **30 fps**
- Captions, diagrams, comparison panels, flow steps, and binary-search visuals
- Approval hashes to prevent accidental generation from an unreviewed plan
- Run locks, atomic artifacts, cache fingerprints, and resumable generation
- An isolated Next.js Cinema UI foundation in [`apps/web`](apps/web)
- TypeScript checks and automated pipeline regression tests

Verified narrated explainers currently cover the water cycle, RAM versus storage, and binary search. Detailed evidence is recorded in [`PROTOTYPE_STATUS.md`](PROTOTYPE_STATUS.md).

## How it works

```text
Topic or example
      |
      v
Gemini storyboard plan
      |
      v
Human review + approval hash
      |
      v
ElevenLabs narration + timestamps
      |
      v
Timeline, captions, and visuals
      |
      v
Remotion export + QA report
```

Each run is kept in `runs/<run-id>/` and can include the plan, review page, manifest, speech alignment, timeline, preview frames, MP4 export, and QA report.

## Requirements

- Node.js **24.x**. The repository version is pinned in [`.nvmrc`](.nvmrc).
- npm
- Gemini API access for planning and revisions
- ElevenLabs API access for narration
- Chromium or Chrome for Remotion, unless `BROWSER_EXECUTABLE` is configured

## Installation

```sh
npm ci
```

Run the standard checks after installation:

```sh
npm run check
```

To inspect whether local provider configuration is present without making provider requests:

```sh
npm run pipeline -- preflight
```

## Configuration

Copy [`.env.example`](.env.example) to `.env.local` when using Gemini or ElevenLabs:

```sh
Copy-Item .env.example .env.local
```

Never commit credentials. The root `.env.local`, generated runs, and media are ignored by Git.

| Variable | Required | Description |
| --- | --- | --- |
| `GEMINI_API_KEY` | For planning | Gemini API key |
| `GEMINI_MODEL` | No | Gemini model; configured default is `gemini-3.5-flash` |
| `ELEVENLABS_API_KEY` | For narration | ElevenLabs API key |
| `ELEVENLABS_MODEL` | No | Speech model; default is `eleven_multilingual_v2` |
| `ELEVENLABS_VOICE_ID_INDIAN_MALE` | No | Configured Indian male voice ID |
| `ELEVENLABS_VOICE_ID_INDIAN_FEMALE` | No | Configured Indian female voice ID |
| `BROWSER_EXECUTABLE` | No | Installed Chrome/Chromium executable |

The default test narrator is Daniel (`daniel-test`), a British English male voice. Indian library voices depend on account access and are not assumed to be available.

## Create a video

### 1. Create a storyboard plan

Use one of the supplied examples or your own topic file:

```sh
npm run pipeline -- plan \
  --run my-video \
  --topic-file examples/ram-storage.md
```

### 2. Review and approve it

Open the generated review page:

```text
runs/my-video/plan-review.html
```

Check the factual claims, scene order, narration, and visual choices. Then approve the exact hash printed by the plan command:

```sh
npm run pipeline -- approve \
  --run my-video \
  --plan-hash <printed-hash>
```

Schema validation checks structure, not factual truth. Do not skip human review.

### 3. Generate the export

```sh
npm run pipeline -- generate --run my-video
```

The finished video is written to:

```text
runs/my-video/output.mp4
```

Review the final audio and visuals before sharing the file.

## Review, revise, and resume

Create a new run from an existing run with an instruction file:

```sh
npm run pipeline -- revise \
  --from my-video \
  --run my-revision \
  --instruction-file examples/ram-storage-revision.md
```

Change the narrator without modifying the source run:

```sh
npm run pipeline -- change-voice \
  --from my-video \
  --run new-narrator \
  --voice daniel-test
```

Resume an interrupted run and reuse compatible cached work:

```sh
npm run pipeline -- resume --run my-video
npm run pipeline -- resume --run my-video --audio-only
npm run pipeline -- resume --run my-video --stills-only
```

Resume verifies cache fingerprints and audio hashes before reuse. A new export replaces the previous MP4 only after its checks pass. Do not remove a run lock while its process is active.

## Fixture and studio workflows

Import an authored plan:

```sh
npm run pipeline -- import-plan \
  --run authored-video \
  --plan-file fixtures/water-cycle.json
```

Generate and inspect the silent fixture in Remotion Studio:

```sh
npm run pipeline -- fixture --run silent-fixture
npm run studio
```

Both workflows require approval before generation. Fixture mode is silent and synthetically timed; it is useful for rendering checks, not narration evaluation.

## Web foundation

The Next.js application is isolated in [`apps/web`](apps/web). It does not expose the root `runs/` or `public/runs` directories.

Start the development server:

```sh
npm run dev
```

Visit:

- [http://localhost:3000](http://localhost:3000) — scaffold preview
- [http://localhost:3000/foundation](http://localhost:3000/foundation) — Cinema UI specimens

These pages are static development surfaces. They do not save data, authenticate users, or call providers.

Build and serve the web app locally:

```sh
npm run build
npm start
```

Prototype configuration belongs in the root `.env.local`; future web configuration belongs in `apps/web/.env.local`.

## Testing and verification

Run the normal validation suite:

```sh
npm run check
```

This runs:

1. Root TypeScript checking
2. Web workspace TypeScript checking
3. The existing pipeline test suite

Run lifecycle checks when changing recovery or renderer-failure behavior:

```sh
npm exec tsx scripts/verify-recovery.ts
npm exec tsx scripts/verify-failure-exit.ts
```

The checks cover contracts, cache reuse, lock release, renderer failures, export metadata, and caption/timeline bounds. They do not replace human review of factual accuracy, audio quality, or visual quality.

## Project structure

```text
.
├── apps/web/       Next.js application foundation
├── design/         Cinema UI direction and verification artifacts
├── examples/       Example topic and revision instructions
├── fixtures/       Authored plans for deterministic workflows
├── scripts/        Recovery and failure-exit verification
├── src/            CLI, providers, pipeline, and Remotion compositions
├── tests/          Pipeline, provider, and search tests
├── runs/           Local generated artifacts; never commit media
└── PROJECT_STATUS.md
```

## Limitations

- This is a local developer workflow, not a multi-user hosted service.
- Accounts, authentication, MongoDB persistence, cloud storage, queues, hosted rendering, billing, and Instagram integration are not implemented.
- AI output requires human factual review; no open-web fact checker is included.
- Strict speech normalization can reject provider responses that do not exactly match approved narration.
- Automatic speech retries, friendly cancellation UI, text-overflow measurement, and complete human review of every topic remain outstanding.
- Daniel is a temporary test narrator; Indian voice quality and account access remain deferred.

## Documentation

- [`PROJECT_STATUS.md`](PROJECT_STATUS.md) — implementation roadmap and acceptance evidence
- [`PROTOTYPE_STATUS.md`](PROTOTYPE_STATUS.md) — verified prototype outputs and remaining work
- [`PRD.md`](PRD.md) — product requirements
- [`SYSTEM_DESIGN.md`](SYSTEM_DESIGN.md) — system architecture
- [`API_DESIGN.md`](API_DESIGN.md) — API contracts
- [`DB_DESIGN.md`](DB_DESIGN.md) — database design
- [`PIPELINE_TECHNICAL_DESIGN.md`](PIPELINE_TECHNICAL_DESIGN.md) — pipeline details

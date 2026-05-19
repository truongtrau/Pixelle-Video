# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

### Run Web UI (Streamlit)
```bash
uv run streamlit run web/app.py
# Opens at http://localhost:8501
```

### Run REST API (FastAPI)
```bash
uv run python api/app.py
# Docs at http://localhost:8000/docs
# With options:
uv run python api/app.py --host 0.0.0.0 --port 8080 --reload
```

### Lint
```bash
uv run ruff check .
uv run ruff format .
```

### Install dependencies (uv manages the venv automatically)
```bash
uv sync
uv sync --extra dev   # includes pytest and ruff
```

### Config setup
```bash
cp config.example.yaml config.yaml
# Then fill in LLM API key, ComfyUI URL or RunningHub API key
```

There is no test suite in this project (no `tests/` directory exists).

## Architecture

### Entry points
Two independent front-ends share the same `pixelle_video` core:
- **Web UI** — `web/app.py` (Streamlit multi-page app, pages in `web/pages/`)
- **REST API** — `api/app.py` (FastAPI with async task queue)

### Core service singleton (`pixelle_video/service.py`)
`PixelleVideoCore` is instantiated as a module-level singleton `pixelle_video`. All code imports from it:
```python
from pixelle_video import pixelle_video
await pixelle_video.initialize()
result = await pixelle_video.generate_video(text="topic", pipeline="standard")
```
It holds all services as attributes (`llm`, `tts`, `media`, `video`, `frame_processor`, `persistence`, `history`) and a `pipelines` dict. **ComfyKit** (ComfyUI client) is lazily initialized and recreated on config change.

### Pipeline system (`pixelle_video/pipelines/`)
Pipelines are the top-level unit of video generation logic:
- `BasePipeline` — abstract, defines the `__call__(text, progress_callback, **kwargs)` interface
- `LinearVideoPipeline` — extends Base with the **Template Method pattern**: 8 lifecycle steps (`setup_environment` → `generate_content` → `determine_title` → `plan_visuals` → `initialize_storyboard` → `produce_assets` → `post_production` → `finalize`). State is passed via `PipelineContext`.
- `StandardPipeline` — the default; generates from topic (LLM writes narrations) or splits a fixed script; handles serial and parallel (RunningHub) frame processing
- `AssetBasedPipeline` — user uploads their own images/videos; LLM analyses them and writes narrations
- `CustomPipeline` — template for custom workflows

The web layer has its **own** pipeline wrappers in `web/pipelines/` (e.g. `DigitalHumanPipeline`, `I2VPipeline`, `ActionTransferPipeline`) that adapt Streamlit session state and call into the core.

### Per-frame processing (`pixelle_video/services/frame_processor.py`)
`FrameProcessor.__call__` orchestrates one frame: **TTS → Image/Video generation → HTML frame composition (Playwright) → MP4 segment (FFmpeg)**. Each step is optional based on template type and whether media already exists. The TTS audio duration drives the video segment length to ensure A/V sync.

### Service layer (`pixelle_video/services/`)
| Service | What it does |
|---|---|
| `LLMService` | OpenAI SDK calls; structured output via Pydantic `response_type` |
| `TTSService` | ComfyKit-based TTS (Edge-TTS, Index-TTS, etc.) |
| `MediaService` | ComfyKit-based image/video generation |
| `VideoService` | FFmpeg concatenation and BGM mixing (via `ffmpeg-python` + `moviepy`) |
| `FrameHTMLService` | Renders HTML templates with Playwright to PNG |
| `PersistenceService` | Saves task metadata and storyboard JSON to `output/` |
| `HistoryManager` | Reads saved tasks for the History page |

### Configuration (`pixelle_video/config/`)
`config_manager` is a global singleton that hot-reloads `config.yaml`. Services call `config_manager.config` to get current values, enabling config changes without restart. The schema is defined in `config/schema.py` as Pydantic models.

### ComfyUI workflow files (`workflows/`)
JSON workflow files for ComfyUI, split into two backends:
- `workflows/selfhost/` — for a locally running ComfyUI instance
- `workflows/runninghub/` — for the RunningHub cloud API

Workflows exist for TTS (`tts_edge.json`, `tts_index2.json`), image generation (`image_flux.json`, `image_qwen.json`, etc.), and video generation (`video_wan2.2.json`, etc.). The active workflow is selected in `config.yaml` under `comfyui.tts.default_workflow`, `comfyui.image.default_workflow`, `comfyui.video.default_workflow`.

### Video templates (`templates/`)
HTML templates organized by resolution (`1080x1920/`, `1080x1080/`, `1920x1080/`). Template filename prefix determines media requirements:
- `static_*.html` — text-only, no AI media needed (fastest, no ComfyUI dependency)
- `image_*.html` — requires AI-generated image per frame
- `video_*.html` — requires AI-generated video per frame

Templates are rendered by Playwright and saved as PNGs, then combined with audio into MP4 segments.

### API task queue (`api/tasks/`)
`TaskManager` runs an async background worker. Long video generation requests go to `/api/video/generate/async`, returning a `task_id`. Progress is polled via `GET /api/tasks/{task_id}`.

### LLM prompts (`pixelle_video/prompts/`)
Each prompt module (e.g. `topic_narration.py`, `image_generation.py`) contains the system/user prompt templates used by `LLMService`. These are the files to edit when tuning generation quality.

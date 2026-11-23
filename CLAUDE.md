# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Python pipeline that turns a YouTube video into short captioned "reels". It uses Azure Video Indexer for the transcript, OpenAI (Assistants API + file_search, `gpt-4o`) to pick quotable segments, moviepy/ffmpeg to cut clips, and Replicate to burn in captions. Artifacts are uploaded to Azure Blob Storage, and progress is tracked in a Firestore `videos` document.

## Commands

Run everything from the repo root. The code uses CWD-relative paths for `.env`, `./firebase-credentials.json` and `./videos/`.

```bash
pip install -r requirements.txt          # also needs the ffmpeg binary on PATH

python ./src/video-processor.py --videoUrl "<youtube url>"   # CLI, full pipeline
python run_api.py                        # FastAPI (uvicorn, reload) on :8000, POST /process-video

docker build -t video-processor .
docker run -v $(pwd)/videos:/app/videos -v $(pwd)/firebase-credentials.json:/app/firebase-credentials.json video-processor --videoUrl "<url>"
```

`ReelGenerator.postman_collection.json` has sample API requests.

Black, isort (profile black, line length 88), mypy and flake8 are configured in `pyproject.toml` and `requirements.txt`. There is **no test suite**: `pyproject.toml` points pytest at a nonexistent `tests/` directory with `--cov=reel_generator`.

## Required config

- `.env` with the keys listed in `Readme.md` (Azure storage SAS URL, Video Indexer + Entra app credentials, `OpenAIAPIKey`, `ReplicateApiKey`). `Config.from_env()` loads it via `dotenv_values`, not `os.environ`. `Consts.__post_init__` raises if the Video Indexer account name or resource group is missing.
- `firebase-credentials.json` (a Firebase service account) in the repo root.

## Architecture

The real code lives in `src/` (`src/lib/` is imported as a package). The root-level `cli.py`, `main.py` and `exceptions.py` are unused scaffolding: `cli.py` imports a nonexistent `.processor` module.

`VideoProcessor.process_video` (`src/lib/VideoProcessor.py`) runs the stages in order. Each stage writes files under `videos/<internalVideoId>/`, uploads them to blob storage and advances the Firestore `state`:

1. **Download** (`VideoDownloader`): pytubefix fetches the lowest-resolution progressive MP4, saves a thumbnail and converts the audio to MP3 with ffmpeg. It creates the Firestore doc (`state` 10, `status` "downloaded") and returns its ID, which is threaded through every later stage.
2. **Upload** (`VideoProcessor.upload_audio_video`): uploads the MP4 and MP3 to blob storage (`state` 40).
3. **Index** (`VideoIndexerClient` + `account_token_provider`): gets an ARM token, then a Video Indexer account token. It indexes the **MP3 blob URL** (not the video), excludes most vision models, polls until indexing finishes and saves `-insights.json`.
4. **Parse transcript** (`TranscriptParser`): merges Video Indexer transcript lines into chunks of about 60 seconds (`{start, end, text}`) and writes `-transcript.json` (`state` 60).
5. **Summarize** (`TranscriptSummarizer`): uploads the transcript to an OpenAI vector store and streams an assistant run that asks for 3–6 quotes as JSON. The raw reply is saved as `-summary.txt` (`state` 80).
6. **Generate reels** (`ReelGenerator`): pulls quotes out of the summary (`json` code blocks first, then a `**Quote:**` markdown regex fallback). It finds each quote in the sanitized, concatenated transcript text and uses the **enclosing 60s transcript chunk's** start/end as the clip bounds, so clips aren't trimmed to the exact quote. It cuts clips with moviepy, uploads them, captions them through Replicate (`tiktok_captions`), uploads the captioned versions and writes the `reels` array (`state` 100, `status` "processed").

`src/api.py` wraps the same synchronous pipeline in a single FastAPI endpoint. The request blocks for the whole run.

## Gotchas

- `FirebaseDbHelper` calls `firebase_admin.initialize_app` in its **class body**, and `TranscriptSummarizer` builds its config, OpenAI and blob clients there too. Importing these modules needs valid credentials and a `.env`, and they must be imported only once per process.
- Methods named `*_async` in `VideoIndexerClient` are ordinary blocking functions.
- Blob uploads use the file's basename as the blob name and skip the upload if that blob already exists.
- Errors are mostly caught, printed and swallowed (Firestore helpers return `None`, and each reel failure is skipped), so a failed stage often only shows up downstream.
- `src/video-processor.py` does `import pprint` and then calls `pprint(...)`, which raises a `TypeError` after the pipeline finishes.

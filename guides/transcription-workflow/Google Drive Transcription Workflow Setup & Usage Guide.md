This workflow automates the download and transcription of large, technical videos from Google Drive using `faster-whisper`. Transcripts are saved locally in the same directory where you run the script.

---

## Assumptions & Use Case

- You have **15–20 large video or audio files** stored on Google Drive.
- You can **view/download** these files (shared to your Google account).
- You have a **MacBook Air M2** (no CUDA / GPU, no Metal backend support).
- You want to produce accurate text transcripts for offline analysis.
- Workflow is optimized for **CPU-only transcription** using `faster-whisper`.

---

## 1. Folder Setup

**What & Why**: Create a clean project directory. Keeps all scripts, credentials, models, and transcripts in one place.

Example structure:

```
transcriber/
├── credentials.json        # Google Drive API credentials (downloaded from Google Cloud)
├── requirements_frozen.txt # Pinned Python dependencies
├── transcribe_from_links.py# The main transcription script
├── links.txt               # Text file with Google Drive video links (one per line)
├── .venv/                  # Python virtual environment
└── models/                 # Cached Whisper models (auto-downloaded)
```

---

## 2. System Prep (macOS, Homebrew)

**What & Why**: Ensures system dependencies are current. Installs FFmpeg, which is required to extract audio from video.

```bash
brew update
brew upgrade
brew outdated
brew cleanup
brew install ffmpeg
```

---

## 3. Python Virtual Environment

**What & Why**: Creates an isolated environment so Python dependencies don’t conflict with system packages. Keeps everything reproducible.

```bash
cd ~/transcriber
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements_frozen.txt
```

- `.venv` is your dedicated Python environment.
- `requirements_frozen.txt` locks package versions for consistency.

---

## 4. Requirements File

**What & Why**: Lists exact dependencies needed for transcription, API calls, and utilities. Pinned versions avoid breaking changes.

Example `requirements_frozen.txt`:

```
# Speech engine
faster-whisper==1.0.3
ctranslate2>=4.4,<5

# Google Drive API
google-api-python-client>=2.131,<3
google-auth>=2.29,<3
google-auth-oauthlib>=1.2,<2

# Utilities
tqdm>=4.66,<5
numpy>=1.26,<3
protobuf>=4.25,<5
```

---

## 5. Google Drive API Auth

**What & Why**: The script downloads files directly from Drive. You must authenticate once using OAuth2.

Steps:

1. Enable **Google Drive API** in [Google Cloud Console](https://console.cloud.google.com/).
1. Download `credentials.json` into your `transcriber/` folder.
1. On first run, a browser window will open. Sign in with your Google account.
1. A `token.json` will be created and cached for future runs.

---

## 6. Pre-download Whisper Models

**What & Why**: Downloads the transcription models once into the `models/` folder. Prevents repeated downloads and ensures offline availability.

We preload `small.en` and `medium.en` for balance between speed and accuracy on CPU.

```bash
FW_DOWNLOAD_ROOT=models python - <<'PY'
from faster_whisper import WhisperModel
for name in ("small.en","medium.en"):
    WhisperModel(name, device="cpu", compute_type="int8", download_root="models")
print("Done.")
PY
```

- **`device="cpu"`** → CPU-only (M2 Air has no CUDA/Metal backend).
- **`compute_type="int8"`** → Optimized quantization for speed on CPU.

---

## 7. Prepare Video Links

**What & Why**: The script takes a text file (`links.txt`) containing one Google Drive video link per line. These are the videos you want to transcribe.

Example `links.txt`:

```
https://drive.google.com/file/d/1hnwUvrVT-XX1IwNgwENKkRBKSFd_CrIN/view?usp=sharing
https://drive.google.com/file/d/1L0iAICGXRzsvwiBNjRnvAqVEp20ogpAK/view?usp=sharing
...
```

---

## 8. Run the Script

**What & Why**: Launches transcription. Saves transcripts locally and logs results in a CSV. Choose job settings based on machine power.

### Example single-job run (safe, less CPU load):

```bash
FW_DEVICE=cpu FW_COMPUTE=int8 FW_THREADS=6 FW_MODEL=small.en FW_DOWNLOAD_ROOT=models python transcribe_from_links.py --links links.txt --jobs 1
```

- **`FW_DEVICE=cpu`** → Force CPU-only.
- **`FW_COMPUTE=int8`** → Quantized inference for faster CPU transcription.
- **`FW_THREADS=6`** → Uses 6 CPU threads (reasonable for M2 Air).
- **`FW_MODEL=small.en`** → Use smaller/faster model unless files are very long.
- **`--jobs 1`** → Run one file at a time. Recommended for laptops.

### Example parallel run (⚠️ heavier load, not always stable on laptops):

```bash
FW_DEVICE=cpu FW_COMPUTE=int8 FW_THREADS=4 FW_MODEL=small.en FW_DOWNLOAD_ROOT=models python transcribe_from_links.py --links links.txt --jobs 2
```

- **`--jobs 2`** → Two files in parallel. May speed up batch runs but uses more memory/CPU.

---

## 9. Outputs

- **Transcripts**: Saved as `fileid_filename.txt` in your working directory.
- **Log CSV**: `processed_links.csv` tracks:
    - file ID
    - title
    - link
    - status (OK, skipped, error)
    - audio duration, size, elapsed time
    - transcript filename

---

# TL;DR 

Copy/paste these commands into a fresh folder:

```bash
brew install ffmpeg
mkdir transcriber && cd transcriber

# put credentials.json, requirements_frozen.txt, and transcribe_from_links.py here

python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements_frozen.txt

FW_DOWNLOAD_ROOT=models python - <<'PY'
from faster_whisper import WhisperModel
for name in ("small.en","medium.en"):
    WhisperModel(name, device="cpu", compute_type="int8", download_root="models")
print("Done.")
PY

echo "https://drive.google.com/file/d/.../view" > links.txt

FW_DEVICE=cpu FW_COMPUTE=int8 FW_THREADS=6 FW_MODEL=small.en FW_DOWNLOAD_ROOT=models python transcribe_from_links.py --links links.txt --jobs 1
```

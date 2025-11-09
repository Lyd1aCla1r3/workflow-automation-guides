# Appendix - Environment Variables & Common Errors

This appendix expands on environment configuration and troubleshooting for the **Google Drive → Faster-Whisper transcription workflow**.

---

## A. Environment Variable Cheat Sheet

| Variable | Default | Purpose | Notes / When to Change |
| --- | --- | --- | --- |
| `FW_DEVICE` | `cpu` | Target hardware backend (`cpu`, `metal`, `cuda`) | M2 Macbook Air users should stick with `cpu`. `metal` is supported only if Apple’s Metal backend is detected. |
| `FW_COMPUTE` | `int8` | Precision of model math (`int8`, `float16`, etc.) | `int8` is fastest on CPU. `float16` is often optimal on GPU/Metal if supported. |
| `FW_MODEL` | `medium.en` | Default transcription model size | `small.en` is faster but less accurate; `medium.en` balances accuracy and speed. |
| `FW_FALLBACK` | `small.en` | Backup model if default fails or audio >1hr | Prevents huge models from choking on long files. |
| `FW_THREADS` | system CPU count | Threads per transcription job | Script adjusts automatically for parallel jobs. Override only if you want manual control. |
| `FW_DOWNLOAD_ROOT` | `models` | Local directory for model cache | Keeps downloaded models persistent between runs. |
| `FW_BEAM` | `5` | Beam search size (decoding search breadth) | Larger = more accuracy, slower. 5 is a good compromise. |
| `FW_OUT_DIR` | `.` | Output directory for transcripts | Change if you want transcripts in a dedicated folder. |
| `FW_GLOSSARY` | `gravitee_glossary.txt` | File containing domain-specific terms | Helps technical accuracy by biasing the model. |
| `FW_NO_SPEECH` | `0.7` | Threshold for ignoring no-speech segments | Higher = fewer false positives. |
| `FW_LOGPROB` | `-1.0` | Log probability cutoff for rejecting segments | Avoids including very low-confidence text. |
| `FW_COMPRESS` | `2.4` | Compression ratio threshold | Helps detect hallucinated/garbled outputs. |
| `FW_CHUNK_SEC` | `600` | Chunk length in seconds (10 minutes) | Long audio is split to avoid memory/time drift. |

---

## B. Common Errors & Fixes

### 1. `ffmpeg not found`

**Cause:** FFmpeg is required to extract WAV audio from video.

**Fix (macOS):**

```bash
brew install ffmpeg
```

Verify with:

```bash
ffmpeg -version
```

---

### 2. `FileNotFoundError: Missing 'credentials.json'`

**Cause:** Google Drive API credentials missing.

**Fix:**

- Enable Google Drive API in your Google Cloud project.
- Download `credentials.json` and place it next to the script.
- First run will open a browser to authorize and create `token.json`.

---

### 3. `RefreshError: invalid_scope`

**Cause:** Token was generated with different scopes than script requires.

**Fix:**

```bash
rm token.json
python transcribe_from_links.py --links links.txt
```

Re-authenticate when prompted.

---

### 4. `Could not extract Google Drive file ID`

**Cause:** Invalid or malformed Drive link.

**Fix:** Ensure links are full format:

```
https://drive.google.com/file/d/<FILE_ID>/view?usp=sharing
```

Or provide the raw file ID.

---

### 5. Model download hangs or fails

**Cause:** Slow or interrupted internet; cache folder missing.

**Fix:**

- Check network connectivity.
- Ensure `FW_DOWNLOAD_ROOT` folder exists.
- Pre-download models with:

```bash
FW_DOWNLOAD_ROOT=models python - <<'PY'
from faster_whisper import WhisperModel
for name in ("small.en","medium.en"):
  WhisperModel(name, device="cpu", compute_type="int8", download_root="models")
print("Done.")
PY
```

---

### 6. `RuntimeError: Metal backend not available`

**Cause:** Attempting `FW_DEVICE=metal` on unsupported machine.

**Fix:** Use:

```bash
FW_DEVICE=cpu FW_COMPUTE=int8
```

---

### 7. Transcriptions contain duplicated or half-sentences

**Cause:** Overlap in chunk segmentation or duplicate boundaries.

**Fix:** Script uses **non-overlapping 10-minute chunks** + post-processing. If still seen:

- Lower `FW_CHUNK_SEC` (e.g., 300) for more precise segmentation.
- Ensure latest version of script with `tidy_segments()` fixes.

---

### 8. Process crashes with tuple unpack error

**Cause:** Result logging expected 9 values but got 8.

**Fix:** Ensure you’re on the fixed script version that always appends an empty `transcript_name` string on error/skip paths.

---

### 9. Script is very slow

**Cause:** Running large models on CPU with too many threads.

**Fix:**

- Use `FW_MODEL=small.en` for faster runs.
- Set `--jobs 1` or `--jobs 2` (CPU-bound tasks don’t scale well).
- Ensure `FW_THREADS` is auto-managed by script (don’t oversubscribe cores).

---

### 10. Output file clutter

**Cause:** Long filenames or ambiguous mapping.

**Fix:** Each transcript is saved as:

```
<drive_file_id>_<sanitized_title>.txt
```

Use `processed_links.csv` to map back to original links.

---

### 11. `SKIPPED_DOWNLOAD_FORBIDDEN`

**Cause:** Script lacks permission to access a file.

**Fix:** Ensure your Google account has at least **Viewer** access. Update sharing settings on the Drive file.

---

### 12. Out of memory on large files (>1GB)

**Cause:** Processing unsegmented giant WAVs.

**Fix:** Script automatically segments, but you can also:

- Use `FW_MODEL=small.en`.
- Reduce `FW_CHUNK_SEC` to split into smaller parts.
- Run fewer parallel jobs.

---

## C. Best Practices

- Keep ~15–20 video links in `links.txt`. Avoid 100+ at once.
- Use `small.en` model for speed; `medium.en` when accuracy is critical.
- Always activate your virtualenv (`source .venv/bin/activate`) before running.
- Keep `processed_links.csv` — it’s your audit trail.
- If re-running, existing transcripts are skipped automatically.

---

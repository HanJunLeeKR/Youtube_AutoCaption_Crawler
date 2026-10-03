# Youtube_AutoCaption_Crawler

A Jupyter notebook that takes a single YouTube channel, downloads **video metadata and
auto-generated subtitles (ASR)**, and organizes them into analysis-ready tables (CSV/XLSX)
and text files (TXT).

The default target is the channel 「안녕하세요원이입니다잘부탁드립니다」
(`@helloiamwoninicetomeetyou`). To collect another channel, just change the channel handle or ID.

- Video files are **not** downloaded — only subtitles and metadata.
- Comments are **not** collected. Only the per-video comment **count** (`comment_count`) is kept in the metadata.
- The collection is resumable: if it stops midway, re-running skips videos that were already downloaded.

---

## 1. Development Environment

| Item | Version / Details |
|---|---|
| OS | Windows 10/11 (also works on macOS and Linux) |
| Python | 3.11.8 |
| Runtime | Jupyter Notebook / JupyterLab / VS Code Jupyter |
| yt-dlp | 2026.08.19 |
| pandas | 3.0.5 |
| openpyxl | latest (for writing .xlsx) |

### Installation

```bash
pip install --upgrade yt-dlp pandas openpyxl jupyter
```

Step 0 of the notebook runs the same installation.

> **Always upgrade yt-dlp to the latest version before collecting.** It is updated frequently to
> keep up with changes on YouTube. With an outdated version, subtitles may come back empty or the
> channel listing may fail.

---

## 2. Usage

### 2-1. Basic run

1. Open `youtube_channel_collector.ipynb` in Jupyter.
2. Check and edit the values below in the **Step 1 (Settings) cell**.
3. Run the cells in order from top to bottom.
4. In Step 3, confirm that the printed **channel name and channel ID are the channel you intended**
   before continuing.

### 2-2. Main settings (`CONFIG`)

| Setting | Default | Description |
|---|---|---|
| `OUT_ROOT` | `./youtube_woni` | Output folder. Use a separate folder for each channel |
| `CHANNEL` | `@helloiamwoninicetomeetyou` | A handle, channel ID (`UC…`), or channel URL |
| `TABS` | `("videos",)` | Channel tabs to scan. `"streams"` (live) and `"shorts"` can be added |
| `PERIOD_MODE` | `"latest"` | `"latest"` = most recent N videos / `"date"` = a date range |
| `MAX_VIDEOS` | `5` | Number of videos in `latest` mode. Raise it to collect the whole channel |
| `START_DATE` · `END_DATE` | `"YYYYMMDD"` | Used only in `date` mode |
| `SUB_LANGS` | `"ko-orig,ko"` | Subtitle languages. `ko-orig` = original Korean auto-captions |
| `REQ_DELAY` | `3.0` | Delay between requests (seconds). Raise to 5–10 if a bot check appears |
| `COOKIES_FROM_BROWSER` | `None` | Set to `"chrome"` etc. when blocked by a bot check or age restriction |
| `TABLE_INCLUDE_TEXT` | `False` | If `True`, full subtitle text is put inside the tables (see the caution below) |

To use shorter folder and file names, edit `OVERRIDE`.

```python
OVERRIDE = {
    "MEDIA_DIR":  "WONI",      # 01_RAW/WONI/ · 03_TEXT/WONI/
    "SLUG":       "woni",      # file name prefix: woni_broadcasts.csv
    "MEDIA_NAME": "안원잘부",   # value of the media column in the tables
}
```

### 2-3. Steps

| Step | What it does | Network | Time |
|---|---|---|---|
| 0 | Environment setup (package install) | ○ | |
| 1 | Settings · preset | | |
| 2 | Define shared functions | | |
| 3 | Resolve the channel · create folders | ○ | |
| 4 | List videos | ○ | short |
| 5 | Collect metadata · subtitles | ○ | **long** |
| 6 | Collect sister programs (off by default) | ○ | |
| 7 | ASR proper-noun correction | | |
| 8–9 | Build tables + attach text file paths | | |
| 10 | Validation | | |

### 2-4. Collecting another channel

Change two lines in Step 1:

```python
CONFIG["CHANNEL"]  = "@target_channel_handle"
CONFIG["OUT_ROOT"] = os.path.join(os.getcwd(), "youtube_newchannel")
```

Also update the names in `OVERRIDE` for the new channel, or set them to `None`
(they are then generated automatically from the handle).

### 2-5. Reading the results

```python
import os
import pandas as pd

ROOT = r"...\youtube_woni"
df = pd.read_csv(os.path.join(ROOT, "02_METADATA", "woni_broadcasts.csv"),
                 encoding="utf-8-sig")

# The full subtitle text lives in txt files, not in the table
txt = open(os.path.join(ROOT, df.loc[0, "txt_full_corrected"]),
           encoding="utf-8-sig").read()
```

---

## 3. Output Structure

```
{OUT_ROOT}/
├ 01_RAW/{MEDIA_DIR}/                  Raw data (never modified by any later step)
│  ├ {YYYYMMDD}_{video_id}.info.json       Full yt-dlp metadata
│  ├ {YYYYMMDD}_{video_id}.ko-orig.json3   Original auto-captions (millisecond timestamps)
│  └ raw_xlsx/
│     ├ RAW_{MEDIA_NAME}_방송단위.csv/.xlsx    Video-level table (uncorrected subtitles)
│     └ RAW_{MEDIA_NAME}_챕터단위.csv/.xlsx    Chapter-level table (uncorrected subtitles)
├ 02_METADATA/                         Tables based on corrected subtitles (CSV is canonical)
│  ├ {SLUG}_broadcasts.csv/.xlsx           Video-level table
│  ├ {SLUG}_chapters.csv/.xlsx             Chapter-level table
│  └ {SLUG}_metadata.xlsx                  Both tables + a missing_days sheet
├ 03_TEXT/{MEDIA_DIR}/                 Subtitle text for analysis (4 files per video)
│  ├ {stem}_full.txt                       Timestamped version (before correction)
│  ├ {stem}_flat.txt                       Plain-text version (before correction)
│  ├ {stem}_full_corrected.txt             Timestamped version (after correction)
│  └ {stem}_flat_corrected.txt             Plain-text version (after correction)
└ 04_LOG/
   ├ {SLUG}_collection_log.json            Collection record (channel info · settings · video list)
   ├ asr_correction_log.xlsx               ASR correction replacement log
   ├ asr_correction_dict.json              Correction dictionary used
   └ upload_date_cache.json                Cache of upload-date lookups
```

`{stem}` has the form `{YYYYMMDD}_{SLUG}_{video_id}` (e.g. `20260828_woni_GJuMbGXXedY`).

All CSV and TXT files are saved as **UTF-8 with BOM (`utf-8-sig`)**, so Korean text displays
correctly in Excel. Specify `encoding="utf-8-sig"` when reading them with pandas as well.

---

## 4. Variable Descriptions

### 4-1. Video-level table — `02_METADATA/{SLUG}_broadcasts.csv`

One row per video. **This is the main table for analysis.**

| Variable | Format | Description |
|---|---|---|
| `date` | `YYYY-MM-DD` | Video date. Under the `generic` preset this equals the upload date |
| `weekday` | 월–일 (Mon–Sun) | Day of the week of `date`, in Korean |
| `video_id` | string (11 chars) | Unique YouTube video ID |
| `title` | string | Video title (original Korean title; auto-translation is prevented) |
| `url` | URL | Video URL |
| `upload_date` | `YYYYMMDD` | YouTube upload date |
| `duration_sec` | integer | Video length in seconds |
| `duration_hms` | `HH:MM:SS` | Video length |
| `view_count_at_collect` | integer | View count. **A snapshot at collection time**; it changes if you re-collect |
| `like_count` | integer | Like count. Blank if the channel hides it |
| `comment_count` | integer | Comment count shown by YouTube. May be rounded; treat as approximate |
| `auto_caption_ko` | 0/1 | Whether Korean auto-captions exist |
| `manual_caption` | string | Language codes of manually uploaded subtitles (`\|`-separated; blank if none) |
| `n_caption_lines` | integer | Number of subtitle lines (events) |
| `n_chars` | integer | Total subtitle characters (based on the uncorrected original) |
| `n_chapters` | integer | Number of chapters. 0 for videos without chapters |
| `transcript_file` | file name | File name of the timestamped subtitle txt |
| `flat_file` | file name | File name of the plain-text subtitle txt |
| `raw_info` | file name | File name of the raw metadata json in `01_RAW` |
| `raw_subtitle` | file name | File name of the raw subtitle file in `01_RAW`. Blank if no subtitles |
| `collected_at` | `YYYY-MM-DD HH:MM:SS` | Collection timestamp |
| `text_len` | integer | Character count of the corrected plain-text subtitles |
| `txt_flat` | relative path | Plain text, before correction |
| `txt_full` | relative path | Timestamped, before correction |
| `txt_flat_corrected` | relative path | Plain text, after correction |
| `txt_full_corrected` | relative path | Timestamped, after correction — **usually the one to analyze** |

The `txt_*` paths are **relative to `OUT_ROOT`**, so they stay valid if the folder is moved.

### 4-2. Chapter-level table — `02_METADATA/{SLUG}_chapters.csv`

Rows exist only when the channel added chapters (a timeline) in the video description.
One row per chapter.

| Variable | Format | Description |
|---|---|---|
| `segment_id` | string | Unique chapter ID: `{prefix}_{YYYYMMDD}_{chapter no.}` (e.g. `WON_20260904_03`) |
| `date` | `YYYY-MM-DD` | Video date |
| `video_id` | string | ID of the parent video |
| `chapter_no` | integer | Chapter order within the video (starting at 1) |
| `chapter_title` | string | Chapter title |
| `start` · `end` | `HH:MM:SS` | Chapter start and end time |
| `start_sec` · `end_sec` | integer | Chapter start and end (seconds) |
| `duration_min` | float | Chapter length in minutes (1 decimal place) |
| `n_lines` | integer | Number of subtitle lines in the chapter |
| `n_chars` | integer | Number of subtitle characters in the chapter |
| `url_at` | URL | Link that jumps to the chapter start (`&t=…s`) |
| `transcript_file` | file name | Timestamped subtitle file of the parent video |
| `text_len` · `txt_*` | | Same as in 4-1 |

### 4-3. Raw tables — `01_RAW/{MEDIA_DIR}/raw_xlsx/`

Similar in content to the `02_METADATA` tables, but computed from the **uncorrected subtitles**.
If sister programs exist, they are stored in the same table as the main show.
Only variables not listed in 4-1 and 4-2 are shown here.

| Variable | Description |
|---|---|
| `media` | Media name (`MEDIA_NAME` in `OVERRIDE`) |
| `program` | Program name. Under `generic`, the channel name |
| `is_main_show` | 1 = main show, 0 = sister program. Always 1 under `generic` |

### 4-4. Subtitle text files — `03_TEXT/`

| File | Format |
|---|---|
| `*_full.txt` | A header (date, title, URL, view count, etc.) + chapter separators + one `HH:MM:SS utterance` per line. Use it to locate a quoted utterance in the video |
| `*_flat.txt` | A single line of utterances joined by spaces, without timestamps. For search and text mining |
| `*_corrected.txt` | Versions of the two files above with the ASR correction dictionary applied. Identical to the originals when the dictionary is empty |

`>>` in the subtitles is YouTube's auto-caption **speaker-change marker** and is kept as is.

### 4-5. `missing_days` sheet — `{SLUG}_metadata.xlsx`

Records days with no video in the period, for presets that assume one video per day
(`ONE_MAIN_PER_DAY=True`). Empty under `generic`.

| Variable | Description |
|---|---|
| `date` · `weekday` | Date and weekday with no video |
| `reason` | `주말` (weekend) or `미확인` (unconfirmed) |

### 4-6. ASR correction log — `04_LOG/asr_correction_log.xlsx`

| Sheet | Contents |
|---|---|
| `replacements` | One row per replacement: file (`source`), stage, variant, replacement, canonical form, confidence, rationale, position (`pos`), and 40 characters of context before and after |
| `dictionary` | The full correction dictionary with the actual number of applications |
| `preserved` | Similar expressions deliberately left unchanged, with reasons |
| `recall_effect` | Occurrences of each keyword before and after correction, with the rate of increase |

---

## 5. Cautions

- **Why full subtitles are not stored in Excel cells** — An Excel cell holds at most 32,767 characters.
  Long subtitles get truncated without warning, or the row structure breaks. The tables therefore keep
  only the length (`text_len`) and file paths, and the full text stays in the txt files. Opening a CSV
  in Excel and **saving it again** causes the same problem, so read CSVs with pandas or R.
- **Auto-captions misrecognize proper nouns.** When quoting an utterance, always go back to the video
  using the timestamp in `*_full.txt` and verify it.
- **The `lang=ko` option** — Adding `--extractor-args youtube:lang=ko` to the video metadata request
  causes `upload_date` and `like_count` to be missing. The notebook uses this option only for the
  channel listing and subtitle requests.
- **Bot checks** (`Sign in to confirm you're not a bot`) — Increase `REQ_DELAY` or set
  `COOKIES_FROM_BROWSER`.

## 6. What Not to Commit

The collected subtitles and metadata contain the channel owner's copyrighted work and the speech of
people appearing in the videos. **Publish the code only; do not commit the output folder (`OUT_ROOT`).**
Adding the output folder to `.gitignore` is recommended.

```gitignore
youtube_*/
.ipynb_checkpoints/
__pycache__/
```

## 7. Collection Ethics

This tool collects publicly available material for research purposes. Do not set the delay between
requests (`REQ_DELAY`) too low — it burdens the server and gets you blocked. Limit quotation and
publication of the collected material to what your research actually requires.

# spanish-transcriber

A GitHub Actions–powered pipeline that turns a Spanish-language YouTube video into a clean, side-by-side Spanish/English HTML transcript with speaker labels.

Each run downloads the audio, transcribes it with OpenAI Whisper, identifies who is speaking with `pyannote.audio` diarization, translates each segment to British English with DeepL, and commits the final HTML back into this repo under `transcripts/`.

## What this repo does

- Downloads audio from a YouTube URL using `yt-dlp`.
- Transcribes the Spanish audio using Whisper (`medium` model) with timestamps.
- Runs speaker diarization with `pyannote.audio` to figure out who said what.
- Translates each Spanish segment to English (`EN-GB`) using the official DeepL Python SDK.
- Renders a polished, readable HTML page with a two-column layout (Spanish on the left, English on the right), bold speaker names above each block, small muted timestamps, and alternating background colours per speaker.
- Commits the rendered HTML to the `transcripts/` folder of this repo.

## How to trigger the workflow

1. Open the repo on GitHub.
2. Go to the **Actions** tab.
3. Select the **Transcribe Spanish YouTube Video** workflow in the left sidebar.
4. Click **Run workflow** in the top right.
5. Fill in the inputs and click **Run workflow**.

### Workflow inputs

| Input | Required | Description |
| --- | --- | --- |
| `youtube_url` | yes | Full YouTube URL of the Spanish-language video to transcribe. |
| `speaker_names` | no | Comma-separated list of host names, e.g. `David,Héctor,Ignatius`. The names are mapped onto detected speakers in the order they first appear. If omitted, speakers are labelled `Host 1`, `Host 2`, `Host 3`, etc. |

When the workflow finishes, the new HTML transcript will appear as a fresh commit on the default branch under `transcripts/`.

## Required repo secrets

Before your first run, add these two secrets in **Settings → Secrets and variables → Actions → New repository secret**:

- `HF_TOKEN` — a Hugging Face access token (free account at [huggingface.co](https://huggingface.co)). Required by `pyannote.audio` to download the diarization model. You will also need to accept the model terms for [`pyannote/speaker-diarization-3.1`](https://huggingface.co/pyannote/speaker-diarization-3.1) and [`pyannote/segmentation-3.0`](https://huggingface.co/pyannote/segmentation-3.0) on your Hugging Face account.
- `DEEPL_API_KEY` — a DeepL API key from [deepl.com](https://www.deepl.com/pro-api). The free tier allows 500,000 characters per month, which is more than enough for a typical podcast episode.

The commit step uses GitHub's built-in `GITHUB_TOKEN`, so no extra token is needed for pushing the transcript.

### Optional but strongly recommended: `YOUTUBE_COOKIES`

GitHub-hosted runners use IP ranges that YouTube aggressively flags as bot traffic. Without cookies you will frequently see:

```
ERROR: [youtube] <id>: Sign in to confirm you're not a bot.
```

The reliable fix is to add a third secret called `YOUTUBE_COOKIES` containing a Netscape-format cookies file exported from a logged-in browser session.

How to export:

1. In a clean browser profile, sign in to YouTube. (Tip: use a private/incognito window so the cookies stay long-lived — closing the window before exporting can shorten their lifetime; just don't sign out.)
2. Install a "Get cookies.txt" extension (e.g. *Get cookies.txt LOCALLY* for Chrome or *cookies.txt* for Firefox).
3. Navigate to `https://www.youtube.com` and use the extension to export cookies for that site as a Netscape-format `.txt` file.
4. Open the file, copy its **entire contents** (including the `# Netscape HTTP Cookie File` header), and paste it as the value of a new repo secret called `YOUTUBE_COOKIES`.

Notes on cookies:

- They are written to a `cookies.txt` file inside the runner only for the duration of the job and never committed.
- Treat them like a password — anyone with these cookies can access your YouTube account. Use a dedicated/secondary Google account if you're cautious.
- Cookies expire. If you start seeing the bot-confirmation error again, repeat the export and update the secret.

## Where the transcripts live

Completed transcripts are written to the [`transcripts/`](./transcripts) folder of this repo. Each file is named after a slugified version of the video title, e.g. `transcripts/episodio-42-la-historia-de-espana.html`. Open the HTML file directly in a browser to read it.

## A note on speaker labels

Speaker diarization is unsupervised: `pyannote.audio` figures out that there are N distinct voices in the audio and assigns them anonymous labels. This means:

- Within a single episode, speaker labels are **consistent** — every line attributed to "Host 1" really is the same voice throughout that episode.
- Across episodes, speaker labels are **not** consistent unless you pass `speaker_names`. "Host 1" in one episode might be a different person from "Host 1" in another.
- If you pass `speaker_names` (e.g. `David,Héctor,Ignatius`), the names are assigned in the order each speaker first appears in the audio. This is usually correct for a podcast that always opens with the same intro, but it is worth a quick spot-check at the top of the file.

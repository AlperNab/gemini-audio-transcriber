# gemini-audio-transcriber

> **Production-grade audio transcription using Gemini 2.5 Flash/Pro** — with speaker diarization, multilingual support, structured JSON output, and built-in credit tracking. The open-source core powering [ToScribe](https://toscribe.to).

[![PyPI version](https://img.shields.io/pypi/v/gemini-audio-transcriber?style=flat)](https://pypi.org/project/gemini-audio-transcriber/)
[![Python](https://img.shields.io/badge/Python-3.10%2B-blue?style=flat&logo=python)](https://python.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Gemini](https://img.shields.io/badge/Gemini-2.5_Flash_%2F_Pro-4285F4?style=flat&logo=google)](https://ai.google.dev)

---

## Why this exists

Gemini 2.5 Flash has the cheapest audio token pricing of any frontier model, natively handles long audio files (up to 9.5 hours in one call), and produces accurate multilingual transcriptions with speaker attribution. But there's no clean Python library that wraps it properly for production use.

This is that library.

---

## Features

- **Speaker diarization** — who said what, with timestamps
- **Multilingual** — transcribe in original language or translate to English in one call
- **Structured output** — clean JSON with words, segments, speakers, confidence
- **Long audio** — Files API for >20MB, inline for small files (auto-detected)
- **Model pinning** — pin to `gemini-2.5-flash-002` for billing predictability
- **Credit tracking** — audio token counting with pre-deduction for SaaS billing
- **Async support** — `asyncio`-native, runs concurrent jobs cleanly
- **CLI included** — transcribe from terminal in one command

---

## Quickstart

```bash
pip install gemini-audio-transcriber
```

```python
from gemini_transcriber import Transcriber

t = Transcriber(api_key="YOUR_GEMINI_API_KEY")

result = t.transcribe(
    audio_path="interview.mp3",
    diarize=True,
    language="auto",          # auto-detect, or "ar", "en", "fr", etc.
    translate_to="en",        # optional — translate output to English
)

print(result.text)            # Full transcript as plain text
print(result.segments)        # List of {speaker, start, end, text}
print(result.speakers)        # ["Speaker 1", "Speaker 2"]
print(result.token_count)     # Audio tokens used (for billing)
```

### CLI usage

```bash
# Basic transcription
gemini-transcribe audio.mp3

# With diarization + translation
gemini-transcribe interview.mp4 --diarize --translate-to en

# Output as JSON
gemini-transcribe meeting.wav --format json --output meeting.json

# Use Pro model for higher accuracy
gemini-transcribe lecture.mp3 --model gemini-2.5-pro-002
```

---

## Output format

```json
{
  "text": "Hello, welcome to the show. Thanks for having me...",
  "language": "en",
  "duration_seconds": 3847.2,
  "token_count": 142081,
  "speakers": ["Speaker 1", "Speaker 2"],
  "segments": [
    {
      "speaker": "Speaker 1",
      "start": 0.0,
      "end": 4.2,
      "text": "Hello, welcome to the show."
    },
    {
      "speaker": "Speaker 2",
      "start": 4.8,
      "end": 7.1,
      "text": "Thanks for having me."
    }
  ],
  "words": [
    { "word": "Hello", "start": 0.0, "end": 0.4, "speaker": "Speaker 1" }
  ],
  "model": "gemini-2.5-flash-002",
  "cost_usd": 0.0113
}
```

---

## Supported formats

Audio: `mp3`, `wav`, `flac`, `aac`, `ogg`, `opus`, `m4a`, `webm`
Video: `mp4`, `avi`, `mov`, `mkv` (audio track extracted automatically)
Max duration: ~9.5 hours per call (Gemini's context window limit)
Max file size: Unlimited — large files automatically use the Files API

---

## Installation

```bash
# Basic install
pip install gemini-audio-transcriber

# With async support
pip install "gemini-audio-transcriber[async]"

# With video support (ffmpeg wrapper)
pip install "gemini-audio-transcriber[video]"
```

**Requires**: A Gemini API key from [Google AI Studio](https://aistudio.google.com)

---

## Advanced usage

### Async batch processing

```python
import asyncio
from gemini_transcriber import AsyncTranscriber

async def batch():
    t = AsyncTranscriber(api_key="YOUR_KEY")
    files = ["ep1.mp3", "ep2.mp3", "ep3.mp3"]
    
    results = await asyncio.gather(*[
        t.transcribe(f, diarize=True) for f in files
    ])
    
    for r in results:
        print(r.text[:200])

asyncio.run(batch())
```

### Credit system integration (SaaS billing)

```python
from gemini_transcriber import Transcriber, TokenEstimator

# Pre-estimate token cost before charging the user
estimator = TokenEstimator()
estimated_tokens = estimator.estimate(audio_path="file.mp3")
estimated_cost_usd = estimator.to_usd(estimated_tokens, model="flash")

# Reserve credits before transcription
# your_credit_system.reserve(user_id, estimated_tokens)

t = Transcriber(api_key="YOUR_KEY")
result = t.transcribe("file.mp3")

# Settle with actual tokens used
# your_credit_system.settle(user_id, result.token_count)
print(f"Actual tokens: {result.token_count}, cost: ${result.cost_usd:.4f}")
```

### Custom prompts / domain-specific vocabulary

```python
result = t.transcribe(
    "medical_lecture.mp3",
    system_prompt="""
    This is a medical lecture. Accurately transcribe medical terminology.
    Common terms: myocardial infarction, tachycardia, echocardiogram.
    """,
    diarize=True,
)
```

### Multilingual / Arabic

```python
# Transcribe Arabic audio, output Arabic
result = t.transcribe("arabic_interview.mp3", language="ar", diarize=True)

# Transcribe Arabic, translate to English
result = t.transcribe(
    "arabic_interview.mp3",
    language="ar",
    translate_to="en",
    diarize=True
)
```

---

## Pricing reference (as of 2025)

| Model | Audio tokens per second | Cost per hour |
|-------|------------------------|---------------|
| `gemini-2.5-flash-002` | ~32 tokens/sec | ~$0.041 |
| `gemini-2.5-pro-002` | ~32 tokens/sec | ~$0.22 |

Use `result.token_count` and `result.cost_usd` to track actual costs.

---

## How it works

1. **File size check**: Files >20MB are uploaded via the [Files API](https://ai.google.dev/gemini-api/docs/files) and referenced by URI. Smaller files are sent inline as base64.
2. **Model call**: A structured prompt instructs Gemini to return JSON with transcript, segments, speaker labels, and word-level timestamps.
3. **Post-processing**: Output is validated against a Pydantic schema, speaker labels are normalized, and overlapping segments are resolved.
4. **Token counting**: Actual audio token usage is extracted from the response metadata for billing accuracy.

---

## Contributing

Issues and PRs welcome. This is the open-source foundation of [ToScribe](https://toscribe.to) — production bug reports are especially appreciated.

```bash
git clone https://github.com/AlperNab/gemini-audio-transcriber
cd gemini-audio-transcriber
pip install -e ".[dev]"
pytest
```

Please read [CONTRIBUTING.md](CONTRIBUTING.md) before submitting a PR.

---

## Related projects

- **[ToScribe](https://toscribe.to)** — The full SaaS built on this library: web UI, user auth, credit system, Zapier integration
- **[shopify-mcp-server](https://github.com/AlperNab/shopify-mcp-server)** — MCP server for Shopify store management with Claude

---

## License

MIT © [Alper Nabil Gabra Zakher](https://github.com/AlperNab)

---

<div align="center">

⭐ **If this saves you from building your own Gemini audio wrapper, please star it**

</div>

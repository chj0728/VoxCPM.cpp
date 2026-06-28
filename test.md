# VoxCPM.cpp

## x86_64 Platform

- Operating System: WSL2 + Ubuntu 22.04.3 LTS + x86_64
- GPU: GeForce RTX 3060 Laptop GPU
- CUDA: 13.3

## Build

### CPU Build

```bash
cmake -B build
cmake --build build
```

### CUDA Build

Enable the ggml CUDA backend at configure time only if you want to run with `--backend cuda`:

```bash
cmake -B build-cuda \
  -DVOXCPM_CUDA=ON \
  -DVOXCPM_BUILD_BENCHMARK=OFF \
  -DVOXCPM_BUILD_TESTS=OFF \
  -DCMAKE_CUDA_ARCHITECTURES=86
cmake --build build-cuda
```

If you want to keep both CPU and CUDA builds, use separate build directories such as `build` and `build-cuda`.

Important:

- `-DVOXCPM_CUDA=ON` is only needed when you want to use `--backend cuda`.
- CPU-only and Vulkan builds do not need CUDA enabled.
- `-DCMAKE_CUDA_ARCHITECTURES=89` is only an example for RTX 40-series GPUs.
- You should set `-DCMAKE_CUDA_ARCHITECTURES` to match your own GPU architecture.
- Common values:
  - `86` for many RTX 30-series GPUs
  - `89` for many RTX 40-series GPUs

If you are unsure, check your GPU model first instead of copying `89` blindly.

check your GPU architecture with:

```bash
nvidia-smi --query-gpu=compute_cap --format=csv
```

## Inference

### Basic CPU Inference

```bash
 ./build/examples/voxcpm_tts \
  --model-path ./models/voxcpm-0.5b-q4_k-audiovae-f16.gguf \
  --prompt-audio ./examples/tai_yi_xian_ren.wav \
  --prompt-text "对，这就是我，万人敬仰的太乙真人。" \
  --text "大家好，我现在正在大可奇奇体验AI科技。" \
  --output ./out_cpu.wav \
  --backend cpu \
  --threads 8
```

### CUDA Inference

```bash
./build-cuda/examples/voxcpm_tts \
  --model-path ./models/voxcpm-0.5b-q4_k-audiovae-f16.gguf \
  --prompt-audio ./examples/tai_yi_xian_ren.wav \
  --prompt-text "对，这就是我，万人敬仰的太乙真人。" \
  --text "大家好，我现在正在大可奇奇体验AI科技。" \
  --output ./out_cuda.wav \
  --backend cuda \
  --threads 8 \
  --inference-timesteps 10 \
  --cfg-value 2.0
```

## OpenAI-Compatible TTS Server

`voxcpm-server` now exposes a single-port HTTP API for:

- `POST /v1/voices`
- `GET /v1/voices/{id}`
- `DELETE /v1/voices/{id}`
- `POST /v1/audio/speech`

### Full Endpoint List

#### `GET /healthz`

Health check.

Example response:

```json
{
  "status": "ok"
}
```

#### `POST /v1/voices`

Registers a reusable voice entry by uploading:

- multipart field `id`: required, unique voice id
- multipart field `text`: required, transcript for the reference audio
- multipart file `audio`: required, reference audio file

Success response: `201 Created`

Returned JSON fields:

- `id`
- `prompt_text`
- `prompt_audio_length`
- `sample_rate`
- `patch_size`
- `feat_dim`
- `created_at`
- `updated_at`

#### `GET /v1/voices/{id}`

Returns metadata for a previously registered voice id.

Success response: `200 OK`

Returned JSON fields:

- `id`
- `prompt_text`
- `prompt_audio_length`
- `sample_rate`
- `patch_size`
- `feat_dim`
- `created_at`
- `updated_at`

#### `DELETE /v1/voices/{id}`

Deletes a registered voice id.

Success response: `200 OK`

Example response:

```json
{
  "id": "taiyi",
  "deleted": true
}
```

#### `POST /v1/audio/speech`

Synthesizes speech from text using a registered voice id.

JSON request fields:

- `model`: required string, must match the configured `--model-name`
- `input`: required string, 1 to 4096 characters
- `voice`: required
  - string voice id, for example `"taiyi"`
  - or object form `{ "id": "taiyi" }`
- `response_format`: optional, defaults to `mp3`
  - supported values: `mp3`, `opus`, `flac`, `wav`, `pcm`
- `speed`: optional float, range `0.25` to `4.0`
- `stream_format`: optional, `audio` or `sse`
- `instructions`: accepted for compatibility, but non-empty values currently return an error

Response behavior:

- `stream_format=audio` or omitted:
  - returns raw audio bytes
  - `Content-Type` matches `response_format`
- `stream_format=sse`:
  - returns `text/event-stream`
  - each `audio.delta` event contains a self-contained chunk encoded with the requested `response_format`
  - emits:
    - `event: audio.delta`
    - `event: audio.completed`

Server-side output rate:

- pass `--output-sample-rate HZ` to `voxcpm-server` to resample synthesized audio before it is encoded
- if omitted, the server uses the model's AudioVAE output rate
- for OpenAI-compatible `pcm` responses, set `--output-sample-rate 24000` if your client expects 24 kHz PCM
- the same override also applies to `wav`, `mp3`, and `opus`

Queue behavior:

- one synthesis request runs at a time per server process
- additional requests wait in a bounded queue controlled by `--max-queue`
- when the queue is full, the server returns `503`

Supported output formats:

- `mp3`: `audio/mpeg`
- `opus`: `audio/ogg; codecs=opus`
- `flac`: `audio/flac`
- `wav`: `audio/wav`
- `pcm`: `application/octet-stream`

Build-time support:

- `VOXCPM_ENABLE_MP3=ON|OFF`
  - toggles MP3 support
  - the runtime tries the native encoder first and falls back to `ffmpeg` if that path cannot initialize
- `VOXCPM_ENABLE_OPUS=ON|OFF`
  - toggles the Opus path, which uses an Ogg Opus `ffmpeg` fallback when enabled
  - if `ffmpeg` is unavailable when CMake configures the build, support is disabled and `/v1/audio/speech` returns `501` for `response_format=opus`

Example outputs:

- `speech.mp3`
- `speech.opus`

These options default to `ON`.

### Start The Server

The server auto-creates `--voice-dir` if it does not exist.

CUDA example:

```bash
./build-cuda/examples/voxcpm-server \
  --host 127.0.0.1 \
  --port 8080 \
  --model-path ./models/voxcpm-0.5b-q4_k-audiovae-f16.gguf \
  --model-name voxcpm-0.5b \
  --threads 8 \
  --backend cuda \
  --voice-dir ./runtime/voices \
  --max-queue 8 \
  --max-decode-steps 512 \
  --output-sample-rate 24000 \
  --disable-auth
```

Use `--max-decode-steps` when serving long text. If omitted or set to `0`, the server keeps the conservative per-backend default decode budget.

CPU example:

```bash
./build/examples/voxcpm-server \
  --host 127.0.0.1 \
  --port 8080 \
  --model-path ./models/voxcpm-0.5b-q4_k-audiovae-f16.gguf \
  --model-name voxcpm-0.5b \
  --threads 8 \
  --backend cpu \
  --voice-dir ./runtime/voices \
  --max-queue 8 \
  --max-decode-steps 512 \
  --output-sample-rate 24000 \
  --disable-auth
```

### Get Health Status

```bash
curl -X GET http://127.0.0.1:8080/healthz
```

### Get Registered Voices

```bash
curl -X GET http://127.0.0.1:8080/v1/voices/{id}

curl -X GET http://127.0.0.1:8080/v1/voices/taiyi
```

### Delete A Voice

```bash
curl -X DELETE http://127.0.0.1:8080/v1/voices/taiyi
```

### Register A Voice

```bash
curl -X POST http://127.0.0.1:8080/v1/voices \
  -F "id=taiyi" \
  -F "text=对，这就是我，万人敬仰的太乙真人。" \
  -F "audio=@./examples/tai_yi_xian_ren.wav"
```

Example response:

```json
{"created_at":"2026-06-28T09:47:35Z",
"feat_dim":64,
"id":"taiyi",
"patch_size":2,
"prompt_audio_length":84,
"prompt_text":"对，这就是我，万人敬仰的太乙真人。",
"sample_rate":16000,
"updated_at":"2026-06-28T09:47:35Z"
}
```

### Synthesize Speech

```bash
curl -X POST http://127.0.0.1:8080/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{
    "model": "voxcpm-0.5b",
    "input": "大家好，我现在正在大可奇奇体验AI科技。",
    "voice": "taiyi",
    "response_format": "wav",
    "speed": 1.0,
    "stream_format": "audio"
  }' \
  --output ./voxcpm_taiyi.wav
```

MP3 example:

```bash
curl -X POST http://127.0.0.1:8080/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{
    "model": "voxcpm-0.5b",
    "input": "大家好，我现在正在大可奇奇体验AI科技。",
    "voice": "taiyi",
    "response_format": "mp3",
    "stream_format": "audio"
  }' \
  --output ./voxcpm_taiyi.mp3
```

Opus example:

```bash
curl -X POST http://127.0.0.1:8080/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{
    "model": "voxcpm-0.5b",
    "input": "大家好，我现在正在大可奇奇体验AI科技。",
    "voice": "taiyi",
    "response_format": "opus",
    "stream_format": "audio"
  }' \
  --output ./voxcpm_taiyi.opus
```

stream pcm example:

```bash
curl -X POST http://127.0.0.1:8080/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{
    "model": "voxcpm-0.5b",
    "input": "大家好，我现在正在大可奇奇体验AI科技。",
    "voice": "taiyi",
    "response_format": "pcm",
    "stream_format": "audio"
  }' \
  --output ./voxcpm_taiyi.pcm
```

## Jetson Orin Platform

部署 Orin 时建议这样压延迟：

```bash
sudo nvpmodel -m 0
sudo jetson_clocks
```

并且：
CMake CUDA 架构用 Orin 的 87
服务端启动加 --backend cuda --output-sample-rate 24000
客户端 sample_rate 与服务端输出采样率保持一致
继续使用 response_format=pcm + stream_format=audio
首次请求前做一次 warmup，避免首轮图/缓存初始化进入用户感知延迟

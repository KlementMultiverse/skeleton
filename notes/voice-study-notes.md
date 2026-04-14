# Level 17: Real-Time Voice AI

> **Audience:** Python backend developers.
> **Goal:** Build production voice AI systems with sub-500ms end-to-end latency.
> **Stack:** FastAPI, WebSockets, Ultravox, Silero VAD, Redis, Twilio.

---

## Table of Contents

1. [Latency Budget Decomposition](#part-1--latency-budget-decomposition)
2. [Audio Formats and WebSocket Streaming](#part-2--audio-formats-and-websocket-streaming)
3. [Ultravox Integration](#part-3--ultravox-integration)
4. [VAD and Interruption Handling](#part-4--vad-voice-activity-detection)
5. [Session State Management](#part-5--session-state-management)
6. [Telephony Integration (Twilio)](#part-6--telephony-integration-twilio)
7. [Production Architecture](#part-7--production-architecture)

---

## Part 1 — Latency Budget Decomposition

### The Postal Analogy

Think of a voice exchange like a courier service between two cities. Every leg of the journey — pickup, sorting, transport, delivery — takes time. If the total journey must complete in 600ms, and you know the truck (network RTT) takes 50ms each way, you have 500ms left to split between writing the letter (LLM) and printing it (TTS). You cannot make the truck faster; you optimize the letter-writing.

That is latency budget decomposition: **name every leg, measure it, allocate budgets, and optimize the legs you control.**

### The Five Components

```
User stops speaking
        │
        ▼
┌───────────────┐
│  VAD silence  │  ~300–500ms  ← end-of-speech detection delay
│  detection    │
└───────┬───────┘
        │
        ▼
┌───────────────┐
│     STT       │  ~50–150ms   ← finalize transcript
│  finalization │
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  LLM TTFT     │  ~100–400ms  ← first token from LLM
│               │
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  TTS TTFA     │  ~40–130ms   ← first audio byte from TTS
│               │
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  Network RTT  │  ~20–50ms    ← WebRTC; 150–700ms for PSTN
│               │
└───────┬───────┘
        │
        ▼
User hears first audio
```

**Realistic totals:**
- WebRTC browser: 510–1,230ms worst case → optimize to **300–500ms**
- PSTN phone call: 660–1,930ms → phone is fundamentally harder

### Budget Allocation Strategy

| Component | Unconstrained | Optimized Target | Notes |
|---|---|---|---|
| VAD silence detection | 500ms | 300ms | Tune `silence_duration_ms` |
| STT finalization | 150ms | 80ms | Use Deepgram streaming |
| LLM TTFT | 400ms | 150ms | Haiku/Groq; streaming |
| TTS TTFA | 130ms | 60ms | Cartesia Sonic, ElevenLabs Turbo |
| Network RTT | 50ms | 30ms | Edge deployment |
| **Total** | **1,230ms** | **620ms** | Goal: keep under 600ms |

**Key insight:** Allocate the most tolerance to LLM TTFT because it is the hardest to shrink — model inference is compute-bound. Squeeze VAD, TTS, and STT instead.

### Measuring Each Component Independently

```python
from __future__ import annotations

import time
from dataclasses import dataclass, field
from typing import Callable


@dataclass
class LatencyBudget:
    """Tracks per-component latency for one voice turn."""

    call_id: str
    vad_silence_start_ms: float = 0.0
    vad_speech_end_ms: float = 0.0      # VAD fires "speech ended"
    stt_start_ms: float = 0.0
    stt_final_ms: float = 0.0           # transcript finalized
    llm_request_ms: float = 0.0
    llm_first_token_ms: float = 0.0     # TTFT measured here
    tts_request_ms: float = 0.0
    tts_first_audio_ms: float = 0.0     # TTFA measured here
    audio_delivered_ms: float = 0.0     # first audio byte sent to client

    def record(self, event: str) -> None:
        now_ms = time.monotonic() * 1_000
        setattr(self, event, now_ms)

    def report(self) -> dict[str, float]:
        return {
            "vad_latency_ms": self.vad_speech_end_ms - self.vad_silence_start_ms,
            "stt_latency_ms": self.stt_final_ms - self.stt_start_ms,
            "llm_ttft_ms": self.llm_first_token_ms - self.llm_request_ms,
            "tts_ttfa_ms": self.tts_first_audio_ms - self.tts_request_ms,
            "total_e2e_ms": self.audio_delivered_ms - self.vad_speech_end_ms,
        }
```

Usage in a voice handler:

```python
import asyncio
import logging
from typing import AsyncIterator

import httpx

logger = logging.getLogger(__name__)


async def handle_voice_turn(
    call_id: str,
    audio_bytes: bytes,
    llm_client: httpx.AsyncClient,
) -> AsyncIterator[bytes]:
    budget = LatencyBudget(call_id=call_id)
    budget.record("vad_speech_end_ms")

    # STT
    budget.record("stt_start_ms")
    transcript = await transcribe(audio_bytes)
    budget.record("stt_final_ms")

    # LLM (streaming)
    budget.record("llm_request_ms")
    first_token_recorded = False
    async for token in stream_llm(transcript, llm_client):
        if not first_token_recorded:
            budget.record("llm_first_token_ms")
            first_token_recorded = True
        yield token

    report = budget.report()
    logger.info(
        "latency_report call_id=%s vad=%.0fms stt=%.0fms llm_ttft=%.0fms "
        "tts_ttfa=%.0fms total=%.0fms",
        call_id,
        report["vad_latency_ms"],
        report["stt_latency_ms"],
        report["llm_ttft_ms"],
        report["tts_ttfa_ms"],
        report["total_e2e_ms"],
    )
```

### Why Sub-500ms Matters Perceptually

Human conversational response time averages **200–250ms**. Research on turn-taking shows:
- < 200ms: feels slightly rushed but natural
- 200–500ms: indistinguishable from human conversation
- 500–1,000ms: noticeable pause; user wonders if they were heard
- > 1,000ms: user assumes failure; repeats themselves or hangs up

The 500ms target is not arbitrary — it is the boundary between "AI assistant" and "laggy phone tree."

### Streaming as the Primary Latency Tool

Without streaming, you wait for complete text before starting TTS. With streaming, TTS begins on the first sentence fragment:

```
Without streaming:
  LLM: [generating 200ms] → complete text → TTS: [converting 300ms] → first audio at 500ms

With streaming:
  LLM: first token at 150ms → TTS: first audio at 210ms → user hears audio at 240ms
  LLM: continues generating → TTS: converts in parallel
```

The TTS hears `"The weather in"` and starts synthesizing before `"San Francisco is"` arrives. This overlap is the core optimization.

---

## Part 2 — Audio Formats and WebSocket Streaming

### The Format Analogy

Audio formats are like water containers: PCM16 is a bucket (large, nothing lost, heavy); Opus is a sealed bottle (smaller, pressurized, needs opening); mulaw is an old clay jug (telephone era, specific shape).

### PCM16 — The Raw Format

**PCM16 (Pulse-Code Modulation, 16-bit signed integers)** is uncompressed audio. Every sample is a 16-bit integer representing the air pressure at that instant.

Key parameters:
- **Sample rate:** how many samples per second
  - 8,000 Hz: telephone quality (intelligible, sounds like a call)
  - 16,000 Hz: standard voice AI (clear speech, compact size)
  - 24,000 Hz: high quality (natural prosody)
  - 44,100 Hz: CD quality (unnecessary for voice AI)
- **Channels:** mono (1) for voice AI; stereo doubles the data
- **Bit depth:** 16-bit standard = 65,536 possible values per sample

**Data size formula:**
```
bytes_per_second = sample_rate × channels × (bit_depth / 8)

16kHz mono PCM16: 16,000 × 1 × 2 = 32,000 bytes/sec = 250 kbps
24kHz mono PCM16: 24,000 × 1 × 2 = 48,000 bytes/sec = 375 kbps
```

**Frame size for real-time:**
```python
SAMPLE_RATE = 16_000       # 16kHz
FRAME_DURATION_MS = 20     # 20ms frame — standard for real-time voice
BYTES_PER_SAMPLE = 2       # PCM16 = 2 bytes per sample

FRAME_SIZE_BYTES = int(SAMPLE_RATE * FRAME_DURATION_MS / 1_000) * BYTES_PER_SAMPLE
# = int(16000 * 0.020) * 2 = 320 * 2 = 640 bytes per frame
```

### Opus — The Compressed Format

Opus compresses audio to **32–64 kbps** vs PCM16's **256 kbps** at 16kHz. It is the native codec for WebRTC and is excellent for browser-to-server streaming.

- Frame size: **20ms** standard (can be 2.5, 5, 10, 20, 40, 60ms)
- Latency: ~26.5ms total algorithmic delay (codec + framing)
- Quality: near-transparent at 64kbps for voice

```python
# Install: pip install pyogg opuslib
import opuslib

encoder = opuslib.Encoder(sample_rate=16_000, channels=1, application="voip")
decoder = opuslib.Decoder(sample_rate=16_000, channels=1)

def encode_pcm_to_opus(pcm_bytes: bytes) -> bytes:
    """Encode 20ms of PCM16 to Opus."""
    # pcm_bytes must be exactly frame_size * channels * 2 bytes
    return encoder.encode(pcm_bytes, frame_size=320)

def decode_opus_to_pcm(opus_bytes: bytes) -> bytes:
    """Decode Opus frame back to PCM16."""
    return decoder.decode(opus_bytes, frame_size=320)
```

### mulaw / alaw — Telephony Legacy

**mulaw (G.711 μ-law)** and **alaw (G.711 A-law)** are the legacy telephony codecs. Every phone call you make uses one of these. They compress 16-bit PCM to 8-bit using logarithmic encoding.

- Sample rate: **8,000 Hz** (hardwired into the PSTN standard)
- Bandwidth: **64 kbps** (8-bit × 8,000 samples/sec)
- Twilio Media Streams use mulaw exclusively

```python
import audioop  # stdlib — converts between PCM and mulaw


def pcm16_to_mulaw(pcm_bytes: bytes) -> bytes:
    """Convert PCM16 at 16kHz to mulaw at 8kHz for Twilio."""
    # Step 1: downsample 16kHz → 8kHz
    downsampled, _ = audioop.ratecv(pcm_bytes, 2, 1, 16_000, 8_000, None)
    # Step 2: convert PCM16 → mulaw (8-bit)
    return audioop.lin2ulaw(downsampled, 2)


def mulaw_to_pcm16(mulaw_bytes: bytes) -> bytes:
    """Convert Twilio mulaw at 8kHz to PCM16 at 16kHz."""
    # Step 1: decode mulaw → PCM16 at 8kHz
    pcm_8k = audioop.ulaw2lin(mulaw_bytes, 2)
    # Step 2: upsample 8kHz → 16kHz for STT models
    pcm_16k, _ = audioop.ratecv(pcm_8k, 2, 1, 8_000, 16_000, None)
    return pcm_16k
```

### FastAPI WebSocket Endpoint for Audio Streaming

```python
from __future__ import annotations

import asyncio
import base64
import json
import logging
from typing import Any

from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from starlette.websockets import WebSocketState

logger = logging.getLogger(__name__)
app = FastAPI()


@app.websocket("/voice/stream")
async def voice_stream(websocket: WebSocket) -> None:
    """
    WebSocket endpoint for raw PCM16 audio streaming.
    Clients send binary PCM16 frames; server sends back binary PCM16 audio.
    """
    await websocket.accept()
    session_id: str | None = None

    try:
        # First message must be JSON with auth token and config
        init_msg = await websocket.receive_json()
        session_id = await authenticate_ws(init_msg, websocket)
        if session_id is None:
            return

        audio_buffer = bytearray()
        FRAME_BYTES = 640  # 20ms at 16kHz PCM16

        async for message in _receive_messages(websocket):
            if isinstance(message, bytes):
                audio_buffer.extend(message)
                # Process in 20ms frames
                while len(audio_buffer) >= FRAME_BYTES:
                    frame = bytes(audio_buffer[:FRAME_BYTES])
                    audio_buffer = audio_buffer[FRAME_BYTES:]
                    await process_audio_frame(frame, session_id, websocket)
            elif isinstance(message, str):
                event = json.loads(message)
                await handle_control_event(event, session_id, websocket)

    except WebSocketDisconnect:
        logger.info("ws_disconnect session_id=%s", session_id)
    except Exception:
        logger.error("ws_error session_id=%s", session_id, exc_info=True)
    finally:
        if websocket.client_state != WebSocketState.DISCONNECTED:
            await websocket.close()


async def _receive_messages(
    websocket: WebSocket,
) -> Any:
    """Yield messages until disconnect. Handles binary and text frames."""
    while True:
        try:
            # receive() returns {"type": "websocket.receive", "bytes": ..., "text": ...}
            raw = await websocket.receive()
            if raw["type"] == "websocket.disconnect":
                return
            if raw.get("bytes"):
                yield raw["bytes"]
            elif raw.get("text"):
                yield raw["text"]
        except WebSocketDisconnect:
            return


async def authenticate_ws(
    init_msg: dict[str, Any],
    websocket: WebSocket,
) -> str | None:
    """Validate auth token from initial handshake message."""
    token = init_msg.get("token")
    if not token:
        await websocket.send_json({"error": "missing_token"})
        await websocket.close(code=4001)
        return None
    session_id = await validate_token(token)
    if not session_id:
        await websocket.send_json({"error": "invalid_token"})
        await websocket.close(code=4001)
        return None
    return session_id
```

### Connection Lifecycle

```
Client                         Server
  │                               │
  │── HTTP GET /voice/stream ─────►│
  │   (Upgrade: websocket)         │
  │◄─ 101 Switching Protocols ─────│
  │                               │
  │── {"token": "...", ──────────►│  ← auth + config
  │    "sample_rate": 16000}       │
  │◄─ {"status": "ready",  ───────│  ← session confirmed
  │    "session_id": "..."}        │
  │                               │
  │── [binary PCM16 frame] ──────►│  ← 640 bytes (20ms)
  │── [binary PCM16 frame] ──────►│
  │── [binary PCM16 frame] ──────►│
  │                               │
  │◄─ [binary PCM16 audio] ───────│  ← TTS response audio
  │◄─ [binary PCM16 audio] ───────│
  │◄─ {"event": "transcript",  ───│  ← interleaved text events
  │    "text": "Hello!"}           │
  │                               │
  │── {"event": "end_session"} ──►│
  │◄─ 1000 Normal Closure ────────│
```

### Backpressure in WebSocket Audio Streams

Backpressure occurs when the producer (audio frames arriving) is faster than the consumer (LLM processing). Without backpressure, you buffer indefinitely and run out of memory.

```python
import asyncio


class AudioStreamProcessor:
    """
    Manages backpressure with a bounded queue.
    If the queue fills, we drop frames rather than OOM.
    """

    MAX_QUEUE_FRAMES = 50  # 50 × 20ms = 1 second of buffering max

    def __init__(self) -> None:
        self._queue: asyncio.Queue[bytes] = asyncio.Queue(
            maxsize=self.MAX_QUEUE_FRAMES
        )
        self._dropped_frames = 0

    async def push_frame(self, frame: bytes) -> None:
        """Non-blocking push. Drops oldest frame if queue is full."""
        if self._queue.full():
            try:
                self._queue.get_nowait()  # drop oldest
                self._dropped_frames += 1
                if self._dropped_frames % 50 == 0:
                    logger.warning(
                        "audio_backpressure dropped_total=%d", self._dropped_frames
                    )
            except asyncio.QueueEmpty:
                pass
        await self._queue.put(frame)

    async def consume_frames(self) -> bytes:
        """Block until a frame is available."""
        return await self._queue.get()
```

---

## Part 3 — Ultravox Integration

### What Ultravox Does

Ultravox is a real-time voice AI platform that collapses the traditional STT → LLM → TTS pipeline into a single model. Instead of:

```
Audio → Whisper (STT) → GPT-4 (LLM) → ElevenLabs (TTS) → Audio
         [100ms]          [200ms]          [80ms]
```

Ultravox processes audio directly:

```
Audio → Ultravox model → Audio
         [~200ms total]
```

This eliminates STT and TTS latency floors because the model understands speech natively (no transcription step) and generates audio tokens directly.

**Architecture:** You create a "call" via REST API, receive a WebSocket URL, connect to it, and stream audio bidirectionally. Ultravox handles VAD, transcription, LLM reasoning, and TTS internally.

### API Flow

```
1. POST /api/calls  →  { joinUrl: "wss://..." }
2. WebSocket connect to joinUrl
3. Stream PCM16 binary frames →
4. ← Receive mixed binary (TTS audio) + text (events) messages
5. Handle events: transcripts, agent responses, VAD signals
6. Close WebSocket when done
```

### Creating a Call

```python
from __future__ import annotations

import logging
import os

import httpx
from pydantic import BaseModel

logger = logging.getLogger(__name__)

ULTRAVOX_API_KEY = os.environ["ULTRAVOX_API_KEY"]
ULTRAVOX_BASE_URL = "https://api.ultravox.ai"


class UltravoxCallConfig(BaseModel):
    system_prompt: str
    voice: str = "Mark"
    model: str = "fixie-ai/ultravox-70B"
    input_sample_rate: int = 16_000   # Hz — match your audio source
    output_sample_rate: int = 16_000  # Hz — what you'll receive
    client_buffer_size_ms: int = 60   # ms — tradeoff: lower = less latency, more underflow risk


class UltravoxCall(BaseModel):
    call_id: str
    join_url: str


async def create_ultravox_call(
    config: UltravoxCallConfig,
    http_client: httpx.AsyncClient,
) -> UltravoxCall:
    """
    Create an Ultravox call session.
    Returns the WebSocket URL to connect to.
    """
    payload = {
        "systemPrompt": config.system_prompt,
        "model": config.model,
        "voice": config.voice,
        "medium": {
            "serverWebSocket": {
                "inputSampleRate": config.input_sample_rate,
                "outputSampleRate": config.output_sample_rate,
                "clientBufferSizeMs": config.client_buffer_size_ms,
            }
        },
    }

    response = await http_client.post(
        f"{ULTRAVOX_BASE_URL}/api/calls",
        headers={"X-API-Key": ULTRAVOX_API_KEY},
        json=payload,
        timeout=10.0,
    )
    response.raise_for_status()
    data = response.json()

    return UltravoxCall(
        call_id=data["callId"],
        join_url=data["joinUrl"],
    )
```

### WebSocket Streaming to Ultravox

```python
from __future__ import annotations

import asyncio
import json
import logging
from collections.abc import AsyncIterator
from dataclasses import dataclass, field
from enum import Enum
from typing import Any

import websockets
from websockets.client import WebSocketClientProtocol

logger = logging.getLogger(__name__)


class UltravoxEventType(str, Enum):
    TRANSCRIPT_PARTIAL = "transcript"          # intermediate transcript (may change)
    TRANSCRIPT_FINAL = "transcript"            # final transcript (use is_final flag)
    AGENT_RESPONSE_START = "agent_response"    # agent about to speak
    VAD_SPEECH_START = "vad_speech_started"
    VAD_SPEECH_END = "vad_speech_stopped"
    PLAYBACK_CLEAR = "playback_clear_buffer"   # interrupt: clear local TTS buffer
    SESSION_END = "session_ended"


@dataclass
class UltravoxEvent:
    event_type: str
    data: dict[str, Any]
    raw: str


@dataclass
class UltravoxSession:
    call_id: str
    join_url: str
    # Outbound: frames to send to Ultravox
    send_queue: asyncio.Queue[bytes] = field(
        default_factory=lambda: asyncio.Queue(maxsize=100)
    )
    # Inbound: TTS audio received from Ultravox
    audio_queue: asyncio.Queue[bytes] = field(
        default_factory=lambda: asyncio.Queue(maxsize=200)
    )
    # Inbound: events (transcripts, VAD, etc.)
    event_queue: asyncio.Queue[UltravoxEvent] = field(
        default_factory=lambda: asyncio.Queue(maxsize=50)
    )
    _connected: bool = False
    _ws: WebSocketClientProtocol | None = None


async def run_ultravox_session(session: UltravoxSession) -> None:
    """
    Connect to Ultravox and run bidirectional audio streaming.
    Handles send and receive concurrently.
    """
    try:
        async with websockets.connect(
            session.join_url,
            ping_interval=20,
            ping_timeout=10,
            max_size=10 * 1024 * 1024,  # 10MB max message
        ) as ws:
            session._ws = ws
            session._connected = True
            logger.info("ultravox_connected call_id=%s", session.call_id)

            # Run sender and receiver concurrently
            async with asyncio.TaskGroup() as tg:
                tg.create_task(_send_audio(ws, session))
                tg.create_task(_receive_messages(ws, session))

    except websockets.exceptions.ConnectionClosed as exc:
        logger.warning(
            "ultravox_disconnected call_id=%s code=%s reason=%s",
            session.call_id,
            exc.code,
            exc.reason,
        )
    except Exception:
        logger.error(
            "ultravox_error call_id=%s", session.call_id, exc_info=True
        )
    finally:
        session._connected = False


async def _send_audio(
    ws: WebSocketClientProtocol,
    session: UltravoxSession,
) -> None:
    """
    Drain the send queue and ship audio frames to Ultravox.
    Send 20ms of audio every 20ms — Ultravox relies on consistent timing
    for VAD and interruption detection.
    """
    while True:
        try:
            # Block up to 100ms waiting for a frame
            frame = await asyncio.wait_for(session.send_queue.get(), timeout=0.1)
            await ws.send(frame)
        except asyncio.TimeoutError:
            # No frame available — send silence to keep timing consistent
            silence = b"\x00" * 640  # 20ms of PCM16 silence at 16kHz
            await ws.send(silence)
        except websockets.exceptions.ConnectionClosed:
            return


async def _receive_messages(
    ws: WebSocketClientProtocol,
    session: UltravoxSession,
) -> None:
    """
    Receive mixed binary (TTS audio) and text (events) from Ultravox.
    Binary → audio_queue
    Text (JSON) → event_queue
    """
    async for message in ws:
        if isinstance(message, bytes):
            # TTS audio — s16le PCM at output_sample_rate
            if not session.audio_queue.full():
                await session.audio_queue.put(message)
            else:
                logger.warning(
                    "ultravox_audio_queue_full call_id=%s dropping=%d bytes",
                    session.call_id,
                    len(message),
                )
        elif isinstance(message, str):
            try:
                data = json.loads(message)
                event = UltravoxEvent(
                    event_type=data.get("type", "unknown"),
                    data=data,
                    raw=message,
                )
                if not session.event_queue.full():
                    await session.event_queue.put(event)
            except json.JSONDecodeError:
                logger.warning(
                    "ultravox_malformed_event call_id=%s raw=%r",
                    session.call_id,
                    message[:200],
                )
```

### Event Handling

```python
async def handle_ultravox_events(session: UltravoxSession) -> None:
    """
    Process events from Ultravox event queue.
    Run this as a separate task alongside audio streaming.
    """
    while session._connected or not session.event_queue.empty():
        try:
            event = await asyncio.wait_for(session.event_queue.get(), timeout=0.5)
        except asyncio.TimeoutError:
            continue

        event_type = event.data.get("type", "")
        logger.debug(
            "ultravox_event call_id=%s type=%s", session.call_id, event_type
        )

        if event_type == "transcript":
            text = event.data.get("text", "")
            is_final = event.data.get("final", False)
            role = event.data.get("role", "user")  # "user" or "agent"
            if is_final:
                logger.info(
                    "transcript_final call_id=%s role=%s text=%r",
                    session.call_id,
                    role,
                    text,
                )
                await on_transcript_final(session.call_id, role, text)

        elif event_type == "playback_clear_buffer":
            # User interrupted the agent — clear local TTS buffer
            logger.info("barge_in_detected call_id=%s", session.call_id)
            await on_barge_in(session)

        elif event_type == "vad_speech_started":
            logger.debug("vad_speech_start call_id=%s", session.call_id)

        elif event_type == "vad_speech_stopped":
            logger.debug("vad_speech_stop call_id=%s", session.call_id)

        elif event_type == "session_ended":
            logger.info("session_ended call_id=%s", session.call_id)
            break


async def on_barge_in(session: UltravoxSession) -> None:
    """
    Handle user interruption: drain the local audio output queue.
    This prevents queued TTS audio from playing after the user started speaking.
    """
    drained = 0
    while not session.audio_queue.empty():
        try:
            session.audio_queue.get_nowait()
            drained += 1
        except asyncio.QueueEmpty:
            break
    logger.info(
        "barge_in_cleared call_id=%s drained_frames=%d", session.call_id, drained
    )
```

### Full FastAPI Integration

```python
from __future__ import annotations

import asyncio
import logging

import httpx
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()
logger = logging.getLogger(__name__)

# Shared HTTP client — reuse across requests
_http_client: httpx.AsyncClient | None = None


@app.on_event("startup")
async def startup() -> None:
    global _http_client
    _http_client = httpx.AsyncClient(timeout=30.0)


@app.on_event("shutdown")
async def shutdown() -> None:
    if _http_client:
        await _http_client.aclose()


@app.websocket("/voice/{user_id}")
async def voice_endpoint(websocket: WebSocket, user_id: str) -> None:
    """
    Client WebSocket endpoint. Connects to Ultravox and proxies audio.
    Client sends PCM16 binary; server sends PCM16 binary back.
    """
    await websocket.accept()

    # 1. Create Ultravox call
    config = UltravoxCallConfig(
        system_prompt=f"You are a helpful assistant. User ID: {user_id}.",
    )
    try:
        call = await create_ultravox_call(config, _http_client)
    except httpx.HTTPError:
        logger.error("ultravox_create_failed user_id=%s", user_id, exc_info=True)
        await websocket.close(code=1011)
        return

    session = UltravoxSession(call_id=call.call_id, join_url=call.join_url)

    # 2. Start Ultravox session in background
    async with asyncio.TaskGroup() as tg:
        tg.create_task(run_ultravox_session(session))
        tg.create_task(handle_ultravox_events(session))
        tg.create_task(_proxy_client_to_ultravox(websocket, session))
        tg.create_task(_proxy_ultravox_to_client(websocket, session))


async def _proxy_client_to_ultravox(
    websocket: WebSocket,
    session: UltravoxSession,
) -> None:
    """Forward audio frames from client WebSocket → Ultravox send queue."""
    try:
        async for message in _receive_messages(websocket):
            if isinstance(message, bytes):
                if not session.send_queue.full():
                    await session.send_queue.put(message)
    except WebSocketDisconnect:
        pass
    finally:
        session._connected = False


async def _proxy_ultravox_to_client(
    websocket: WebSocket,
    session: UltravoxSession,
) -> None:
    """Forward TTS audio from Ultravox → client WebSocket."""
    while session._connected or not session.audio_queue.empty():
        try:
            audio_chunk = await asyncio.wait_for(
                session.audio_queue.get(), timeout=0.5
            )
            await websocket.send_bytes(audio_chunk)
        except asyncio.TimeoutError:
            continue
        except WebSocketDisconnect:
            break
```

---

## Part 4 — VAD (Voice Activity Detection)

### The Bouncer Analogy

VAD is a bouncer at the door of your STT pipeline. Its job is simple: **"Is someone speaking right now?"** A good bouncer lets speech through immediately and holds silence back. A slow bouncer makes the whole queue wait.

Two types of bouncers:
1. **Energy-based (fast, dumb):** Is the audio loud? → probably speech. Fails in noisy environments.
2. **Model-based (slower, smart):** Does this sound like human speech patterns? → Silero VAD. Works with background noise.

### Energy-Based VAD

```python
import audioop
import struct


def energy_vad(
    pcm_bytes: bytes,
    threshold_rms: int = 500,
    sample_width: int = 2,
) -> bool:
    """
    Simple energy-based VAD. Returns True if frame contains speech.

    threshold_rms: RMS amplitude threshold. Typical values:
      - 300: sensitive (picks up quiet speech, also some noise)
      - 500: balanced
      - 800: conservative (misses quiet speech)
    """
    if len(pcm_bytes) < sample_width:
        return False
    rms = audioop.rms(pcm_bytes, sample_width)
    return rms > threshold_rms


# Calibration helper
def calibrate_threshold(background_frames: list[bytes], multiplier: float = 2.0) -> int:
    """
    Compute a threshold based on background noise RMS.
    Record 1-2 seconds of silence, pass the frames here.
    """
    rms_values = [audioop.rms(f, 2) for f in background_frames]
    avg_background_rms = sum(rms_values) / len(rms_values)
    return int(avg_background_rms * multiplier)
```

### Silero VAD — Model-Based

Silero VAD is the production standard for real-time voice activity detection. It uses a small LSTM model that processes 30ms chunks in under **1ms on CPU**.

```python
# Install: pip install silero-vad torch torchaudio
from __future__ import annotations

import asyncio
import logging
import struct
from typing import Callable

import torch

logger = logging.getLogger(__name__)


class SileroVAD:
    """
    Silero VAD wrapper for real-time voice activity detection.
    Processes PCM16 frames and emits speech_start / speech_end events.
    """

    SAMPLE_RATE = 16_000
    FRAME_SAMPLES = 512          # 32ms at 16kHz — Silero minimum chunk
    SPEECH_THRESHOLD = 0.5       # probability above this = speech
    SILENCE_THRESHOLD = 0.35     # probability below this = silence
    END_OF_SPEECH_FRAMES = 15    # 15 × 32ms = 480ms of silence before declaring end

    def __init__(self) -> None:
        self._model, utils = torch.hub.load(
            repo_or_dir="snakers4/silero-vad",
            model="silero_vad",
            force_reload=False,
            onnx=True,   # ONNX is faster on CPU
        )
        self._model.eval()
        self._is_speaking = False
        self._silence_frame_count = 0
        self._on_speech_start: Callable[[], None] | None = None
        self._on_speech_end: Callable[[], None] | None = None

    def set_callbacks(
        self,
        on_speech_start: Callable[[], None],
        on_speech_end: Callable[[], None],
    ) -> None:
        self._on_speech_start = on_speech_start
        self._on_speech_end = on_speech_end

    def process_frame(self, pcm_bytes: bytes) -> float:
        """
        Process one PCM16 frame. Returns speech probability 0.0–1.0.
        Call this for every 32ms frame of incoming audio.
        """
        if len(pcm_bytes) < self.FRAME_SAMPLES * 2:
            return 0.0

        # Convert PCM16 bytes → float32 tensor in [-1.0, 1.0]
        samples = struct.unpack(f"{self.FRAME_SAMPLES}h", pcm_bytes[:self.FRAME_SAMPLES * 2])
        tensor = torch.tensor(samples, dtype=torch.float32) / 32768.0
        tensor = tensor.unsqueeze(0)  # [1, samples]

        with torch.no_grad():
            prob = self._model(tensor, self.SAMPLE_RATE).item()

        self._update_state(prob)
        return prob

    def _update_state(self, prob: float) -> None:
        if not self._is_speaking:
            if prob >= self.SPEECH_THRESHOLD:
                self._is_speaking = True
                self._silence_frame_count = 0
                logger.debug("vad_speech_start prob=%.3f", prob)
                if self._on_speech_start:
                    self._on_speech_start()
        else:
            if prob < self.SILENCE_THRESHOLD:
                self._silence_frame_count += 1
                if self._silence_frame_count >= self.END_OF_SPEECH_FRAMES:
                    self._is_speaking = False
                    self._silence_frame_count = 0
                    logger.debug("vad_speech_end silent_frames=%d", self.END_OF_SPEECH_FRAMES)
                    if self._on_speech_end:
                        self._on_speech_end()
            else:
                self._silence_frame_count = 0  # reset on speech
```

### Barge-In Detection and Interruption Handling

Barge-in is the most important UX feature in voice AI. Without it, users must wait for the agent to finish speaking before they can respond — like being stuck on hold.

```python
from __future__ import annotations

import asyncio
import logging
from dataclasses import dataclass, field

logger = logging.getLogger(__name__)


@dataclass
class BargeInController:
    """
    Manages barge-in: detects when user speaks while agent is playing audio
    and immediately cancels TTS playback.
    """

    call_id: str
    _agent_speaking: bool = False
    _barge_in_lock: asyncio.Lock = field(default_factory=asyncio.Lock)
    _tts_task: asyncio.Task | None = None

    def set_agent_speaking(self, speaking: bool) -> None:
        self._agent_speaking = speaking

    async def on_user_speech_start(self) -> None:
        """
        Called by VAD when user begins speaking.
        If agent is currently playing audio, cancel it immediately.
        """
        if not self._agent_speaking:
            return

        async with self._barge_in_lock:
            if self._tts_task and not self._tts_task.done():
                self._tts_task.cancel()
                logger.info("barge_in_interrupt call_id=%s", self.call_id)
                try:
                    await self._tts_task
                except asyncio.CancelledError:
                    pass
            self._agent_speaking = False

    def set_tts_task(self, task: asyncio.Task) -> None:
        self._tts_task = task
        self._agent_speaking = True


async def play_tts_with_barge_in(
    audio_frames: list[bytes],
    controller: BargeInController,
    send_func: Callable[[bytes], Awaitable[None]],
) -> None:
    """
    Play TTS audio frames with barge-in cancellation support.
    If user starts speaking, this coroutine gets cancelled.
    """
    task = asyncio.current_task()
    if task:
        controller.set_tts_task(task)

    try:
        for frame in audio_frames:
            await send_func(frame)
            # Small yield to allow barge-in check
            await asyncio.sleep(0.001)
    finally:
        controller.set_agent_speaking(False)
```

### End-of-Speech Detection: How Long to Wait?

The right silence timeout depends on context:

| Scenario | Silence Duration | Why |
|---|---|---|
| Single word response | 300ms | Short, crisp turns |
| Answering a question | 500ms | User may pause mid-sentence |
| Dictation | 800ms | Longer natural pauses |
| Phone calls | 600ms | Network jitter masks short silences |

```python
class EndOfSpeechDetector:
    """
    Adapts silence timeout based on conversation context.
    Longer turns get more generous silence detection.
    """

    BASE_SILENCE_MS = 400

    def __init__(self, vad: SileroVAD) -> None:
        self._vad = vad
        self._speech_frames_in_turn = 0
        self._last_speech_ms: float = 0.0

    def adaptive_silence_threshold(self) -> int:
        """Returns silence duration in ms to wait before declaring end-of-speech."""
        if self._speech_frames_in_turn < 5:
            return 300  # Short turn — quick cutoff
        elif self._speech_frames_in_turn < 20:
            return 500  # Medium turn — standard
        else:
            return 700  # Long turn — user is explaining something; be patient
```

---

## Part 5 — Session State Management

### The Whiteboard Analogy

A voice conversation is like two people at a whiteboard. Every message adds to what's on the board. If you erase the board between each sentence, the next speaker has to repeat everything. Session state is the whiteboard — it persists the context that makes conversation coherent.

Voice sessions are **more stateful than text sessions** because:
1. Audio cannot be re-read — if context is lost, the user must repeat themselves aloud
2. Turns are shorter — the model must remember more from earlier turns
3. Intent can span multiple turns — "book a table" → "for 4 people" → "this Saturday"

### Redis for Fast Session Reads

```python
from __future__ import annotations

import json
import logging
from dataclasses import dataclass, field
from typing import Any

import redis.asyncio as aioredis

logger = logging.getLogger(__name__)

REDIS_URL = "redis://localhost:6379"
SESSION_TTL_SECONDS = 1_800  # 30 minutes hard timeout
IDLE_TTL_SECONDS = 120       # 2 minutes idle timeout (refreshed on each turn)


@dataclass
class VoiceSessionState:
    session_id: str
    user_id: str
    call_id: str | None = None
    turn_count: int = 0
    current_intent: str | None = None
    slot_values: dict[str, Any] = field(default_factory=dict)
    conversation_history: list[dict[str, str]] = field(default_factory=list)
    metadata: dict[str, Any] = field(default_factory=dict)

    def to_json(self) -> str:
        return json.dumps({
            "session_id": self.session_id,
            "user_id": self.user_id,
            "call_id": self.call_id,
            "turn_count": self.turn_count,
            "current_intent": self.current_intent,
            "slot_values": self.slot_values,
            "conversation_history": self.conversation_history[-20:],  # keep last 20 turns
            "metadata": self.metadata,
        })

    @classmethod
    def from_json(cls, data: str) -> "VoiceSessionState":
        d = json.loads(data)
        return cls(**d)


class VoiceSessionStore:
    """
    Dual-layer session store: Redis for speed, PostgreSQL for durability.
    Pattern: read from Redis, fall back to PostgreSQL on cache miss.
    """

    def __init__(self, redis_client: aioredis.Redis) -> None:
        self._redis = redis_client

    def _key(self, session_id: str) -> str:
        return f"voice:session:{session_id}"

    async def get(self, session_id: str) -> VoiceSessionState | None:
        raw = await self._redis.get(self._key(session_id))
        if raw is None:
            return None
        try:
            return VoiceSessionState.from_json(raw)
        except (json.JSONDecodeError, TypeError):
            logger.error(
                "session_parse_error session_id=%s", session_id, exc_info=True
            )
            return None

    async def save(
        self,
        state: VoiceSessionState,
        idle_refresh: bool = True,
    ) -> None:
        key = self._key(state.session_id)
        serialized = state.to_json()
        ttl = IDLE_TTL_SECONDS if idle_refresh else SESSION_TTL_SECONDS
        await self._redis.setex(key, ttl, serialized)

    async def delete(self, session_id: str) -> None:
        await self._redis.delete(self._key(session_id))

    async def add_turn(
        self,
        session_id: str,
        role: str,
        text: str,
    ) -> VoiceSessionState | None:
        """Atomically append a conversation turn to the session."""
        state = await self.get(session_id)
        if state is None:
            return None
        state.conversation_history.append({"role": role, "content": text})
        state.turn_count += 1
        await self.save(state)
        return state
```

### LangGraph with AsyncPostgresSaver

For multi-turn agents that need durable state with graph-based conversation flow:

```python
# Install: pip install langgraph langgraph-checkpoint-postgres
from __future__ import annotations

import logging
from typing import Annotated, TypedDict

from langchain_core.messages import AIMessage, HumanMessage
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
from langgraph.graph import END, START, StateGraph
from langgraph.graph.message import add_messages

logger = logging.getLogger(__name__)


class VoiceAgentState(TypedDict):
    messages: Annotated[list, add_messages]
    current_intent: str | None
    slot_values: dict
    turn_count: int


def build_voice_agent_graph() -> StateGraph:
    """Build a LangGraph voice agent with persistent state."""

    graph = StateGraph(VoiceAgentState)

    async def intent_classifier(state: VoiceAgentState) -> dict:
        last_msg = state["messages"][-1].content if state["messages"] else ""
        # Classify intent from latest user utterance
        intent = await classify_intent(last_msg)
        return {
            "current_intent": intent,
            "turn_count": state.get("turn_count", 0) + 1,
        }

    async def slot_filler(state: VoiceAgentState) -> dict:
        # Extract slot values from conversation history
        slots = await extract_slots(
            state["messages"],
            intent=state.get("current_intent"),
        )
        return {"slot_values": slots}

    async def response_generator(state: VoiceAgentState) -> dict:
        response_text = await generate_response(
            messages=state["messages"],
            intent=state.get("current_intent"),
            slots=state.get("slot_values", {}),
        )
        return {"messages": [AIMessage(content=response_text)]}

    graph.add_node("classify_intent", intent_classifier)
    graph.add_node("fill_slots", slot_filler)
    graph.add_node("generate_response", response_generator)

    graph.add_edge(START, "classify_intent")
    graph.add_edge("classify_intent", "fill_slots")
    graph.add_edge("fill_slots", "generate_response")
    graph.add_edge("generate_response", END)

    return graph


async def create_persistent_voice_agent(db_url: str):
    """Create a voice agent with PostgreSQL-backed state persistence."""
    async with await AsyncPostgresSaver.from_conn_string(db_url) as checkpointer:
        await checkpointer.setup()
        graph = build_voice_agent_graph().compile(checkpointer=checkpointer)
        return graph


async def run_voice_turn(
    graph,
    session_id: str,
    user_text: str,
) -> str:
    """Process one voice turn. State is automatically persisted."""
    config = {"configurable": {"thread_id": session_id}}
    state = await graph.ainvoke(
        {"messages": [HumanMessage(content=user_text)]},
        config=config,
    )
    last_ai_msg = next(
        (m for m in reversed(state["messages"]) if isinstance(m, AIMessage)),
        None,
    )
    return last_ai_msg.content if last_ai_msg else ""
```

### Session Expiry Patterns

```python
import asyncio
import logging

logger = logging.getLogger(__name__)


class SessionExpiryManager:
    """
    Manages idle timeout (2 min) and hard timeout (30 min) for voice sessions.
    Idle timeout resets on each turn. Hard timeout is absolute.
    """

    IDLE_TIMEOUT_S = 120
    HARD_TIMEOUT_S = 1_800

    def __init__(self, session_store: VoiceSessionStore) -> None:
        self._store = session_store
        self._idle_timers: dict[str, asyncio.TimerHandle] = {}
        self._hard_timers: dict[str, asyncio.TimerHandle] = {}

    def register_session(self, session_id: str) -> None:
        loop = asyncio.get_event_loop()
        self._hard_timers[session_id] = loop.call_later(
            self.HARD_TIMEOUT_S,
            self._expire_session,
            session_id,
            "hard_timeout",
        )
        self._reset_idle(session_id)

    def on_turn(self, session_id: str) -> None:
        """Call this on every user utterance to reset idle timer."""
        self._reset_idle(session_id)

    def _reset_idle(self, session_id: str) -> None:
        if session_id in self._idle_timers:
            self._idle_timers[session_id].cancel()
        loop = asyncio.get_event_loop()
        self._idle_timers[session_id] = loop.call_later(
            self.IDLE_TIMEOUT_S,
            self._expire_session,
            session_id,
            "idle_timeout",
        )

    def _expire_session(self, session_id: str, reason: str) -> None:
        logger.info("session_expired session_id=%s reason=%s", session_id, reason)
        asyncio.create_task(self._async_expire(session_id))

    async def _async_expire(self, session_id: str) -> None:
        await self._store.delete(session_id)
        self._idle_timers.pop(session_id, None)
        self._hard_timers.pop(session_id, None)
```

### Session Handoff: Voice to Text

```python
async def handoff_voice_to_text(
    voice_session_id: str,
    session_store: VoiceSessionStore,
) -> str:
    """
    Transfer voice session context to a text chat session.
    Returns a text session ID that includes full conversation history.
    """
    voice_state = await session_store.get(voice_session_id)
    if voice_state is None:
        raise VoiceSessionNotFoundError(f"Session {voice_session_id!r} not found")

    text_session_id = f"text:{voice_state.user_id}:{voice_session_id[:8]}"
    text_state = VoiceSessionState(
        session_id=text_session_id,
        user_id=voice_state.user_id,
        call_id=None,
        turn_count=voice_state.turn_count,
        current_intent=voice_state.current_intent,
        slot_values=voice_state.slot_values,
        conversation_history=voice_state.conversation_history,
        metadata={**voice_state.metadata, "transferred_from_voice": True},
    )
    await session_store.save(text_state, idle_refresh=False)
    return text_session_id
```

---

## Part 6 — Telephony Integration (Twilio)

### The Telephone Exchange Analogy

Twilio is a telephone exchange that lives in the cloud. When someone calls your Twilio number, Twilio answers, then asks your server: "Someone called — what should I do?" Your server replies in TwiML (a tiny XML dialect) saying "connect them to my WebSocket." From then on, audio flows between the caller's phone and your server over a WebSocket, in mulaw 8kHz format — because that is what every phone on earth speaks.

### TwiML and Media Streams Setup

```python
from __future__ import annotations

import logging
import os

from fastapi import FastAPI, Request, Response

app = FastAPI()
logger = logging.getLogger(__name__)

WEBSOCKET_HOST = os.environ["WEBSOCKET_HOST"]  # e.g. "wss://your-server.com"


@app.post("/twiml/inbound")
async def twiml_inbound(request: Request) -> Response:
    """
    Twilio calls this when an inbound call arrives.
    We respond with TwiML instructing Twilio to connect audio to our WebSocket.
    """
    twiml = f"""<?xml version="1.0" encoding="UTF-8"?>
<Response>
    <Say voice="alice">Please hold while we connect you.</Say>
    <Connect>
        <Stream url="{WEBSOCKET_HOST}/twilio/stream" track="both_tracks">
            <Parameter name="caller" value="{{{{CallSid}}}}"/>
        </Stream>
    </Connect>
</Response>"""
    return Response(content=twiml, media_type="text/xml")
```

### WebSocket Endpoint for Twilio Media Streams

```python
from __future__ import annotations

import asyncio
import base64
import json
import logging
from typing import Any

from fastapi import WebSocket, WebSocketDisconnect

logger = logging.getLogger(__name__)


@app.websocket("/twilio/stream")
async def twilio_media_stream(websocket: WebSocket) -> None:
    """
    Receives Twilio Media Stream audio (mulaw 8kHz base64-encoded).
    Responds with AI-generated audio (mulaw 8kHz base64-encoded).
    """
    await websocket.accept()

    stream_sid: str | None = None
    call_sid: str | None = None
    session: UltravoxSession | None = None

    try:
        while True:
            raw = await websocket.receive_text()
            message = json.loads(raw)
            event = message.get("event")

            if event == "connected":
                logger.info("twilio_connected")

            elif event == "start":
                stream_sid = message["streamSid"]
                call_sid = message["start"]["callSid"]
                logger.info(
                    "twilio_stream_start stream_sid=%s call_sid=%s",
                    stream_sid,
                    call_sid,
                )
                # Create Ultravox session configured for mulaw input/output
                session = await _start_twilio_voice_session(call_sid)

            elif event == "media" and session is not None:
                # Decode base64 mulaw audio from Twilio
                mulaw_payload = base64.b64decode(message["media"]["payload"])
                # Convert mulaw 8kHz → PCM16 16kHz for Ultravox
                pcm16_frame = mulaw_to_pcm16(mulaw_payload)
                if not session.send_queue.full():
                    await session.send_queue.put(pcm16_frame)

                # If Ultravox has audio to play back, send it to Twilio
                while not session.audio_queue.empty():
                    tts_pcm = session.audio_queue.get_nowait()
                    await _send_audio_to_twilio(websocket, stream_sid, tts_pcm)

            elif event == "dtmf":
                digit = message["dtmf"]["digit"]
                logger.info(
                    "dtmf_received call_sid=%s digit=%s", call_sid, digit
                )
                await handle_dtmf(call_sid, digit)

            elif event == "stop":
                logger.info("twilio_stream_stop call_sid=%s", call_sid)
                break

    except WebSocketDisconnect:
        logger.info("twilio_ws_disconnect call_sid=%s", call_sid)
    except Exception:
        logger.error("twilio_error call_sid=%s", call_sid, exc_info=True)
    finally:
        if session:
            session._connected = False


async def _send_audio_to_twilio(
    websocket: WebSocket,
    stream_sid: str,
    pcm16_bytes: bytes,
) -> None:
    """Convert PCM16 16kHz to mulaw 8kHz and send to Twilio."""
    mulaw_audio = pcm16_to_mulaw(pcm16_bytes)
    payload = base64.b64encode(mulaw_audio).decode("utf-8")
    await websocket.send_json({
        "event": "media",
        "streamSid": stream_sid,
        "media": {"payload": payload},
    })


async def _clear_twilio_audio_buffer(
    websocket: WebSocket,
    stream_sid: str,
) -> None:
    """Send clear event to Twilio to cancel buffered audio (for barge-in)."""
    await websocket.send_json({
        "event": "clear",
        "streamSid": stream_sid,
    })


async def _send_mark_to_twilio(
    websocket: WebSocket,
    stream_sid: str,
    mark_name: str,
) -> None:
    """
    Send a mark — Twilio will echo this back when the audio before
    this mark finishes playing. Useful for knowing when TTS is done.
    """
    await websocket.send_json({
        "event": "mark",
        "streamSid": stream_sid,
        "mark": {"name": mark_name},
    })
```

### DTMF Handling

```python
import logging
from enum import Enum

logger = logging.getLogger(__name__)


class DTMFDigit(str, Enum):
    ZERO = "0"
    ONE = "1"
    TWO = "2"
    THREE = "3"
    FOUR = "4"
    FIVE = "5"
    SIX = "6"
    SEVEN = "7"
    EIGHT = "8"
    NINE = "9"
    STAR = "*"
    HASH = "#"


DTMF_MENU: dict[str, str] = {
    "1": "sales",
    "2": "support",
    "3": "billing",
    "0": "operator",
}


async def handle_dtmf(call_sid: str, digit: str) -> None:
    """
    Process DTMF digit press.
    Example: phone tree navigation via keypad.
    """
    destination = DTMF_MENU.get(digit)
    if destination:
        logger.info(
            "dtmf_route call_sid=%s digit=%s destination=%s",
            call_sid,
            digit,
            destination,
        )
        await route_call(call_sid, destination)
    else:
        logger.info("dtmf_unrecognized call_sid=%s digit=%s", call_sid, digit)
```

### Call Lifecycle Events

```python
from fastapi import Form


@app.post("/twilio/status")
async def call_status_callback(
    CallSid: str = Form(...),
    CallStatus: str = Form(...),
    Duration: str | None = Form(None),
) -> Response:
    """
    Twilio posts call status updates here.
    StatusCallback URL must be configured in Twilio console or TwiML.
    """
    logger.info(
        "call_status call_sid=%s status=%s duration=%s",
        CallSid,
        CallStatus,
        Duration,
    )

    if CallStatus == "completed":
        await on_call_completed(CallSid, int(Duration or "0"))
    elif CallStatus == "failed":
        await on_call_failed(CallSid)
    elif CallStatus == "no-answer":
        await on_call_no_answer(CallSid)

    return Response(content="", status_code=204)


async def on_call_completed(call_sid: str, duration_seconds: int) -> None:
    """Persist call record, release session resources."""
    logger.info(
        "call_completed call_sid=%s duration_s=%d", call_sid, duration_seconds
    )
    # Archive session to PostgreSQL for GDPR-compliant retention
    await archive_call_session(call_sid, duration_seconds)
```

---

## Part 7 — Production Architecture

### WebSocket Scaling: The Sticky Session Problem

WebSockets are **stateful connections**. If you have 3 backend instances and a user's WebSocket is on instance 1, all their messages must continue going to instance 1. A load balancer that round-robins requests will break the connection.

**Solution: Sticky sessions (IP hash or cookie-based load balancing)**

```nginx
# nginx.conf — sticky sessions for WebSocket
upstream voice_backend {
    ip_hash;  # same client IP always hits same backend
    server backend1:8000;
    server backend2:8000;
    server backend3:8000;
}

server {
    listen 443 ssl;

    location /voice/ {
        proxy_pass http://voice_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 3600s;   # hold WebSocket open for 1 hour
        proxy_send_timeout 3600s;
    }
}
```

**Redis pub/sub for broadcast across instances:**

```python
import json
import logging

import redis.asyncio as aioredis

logger = logging.getLogger(__name__)


class VoiceEventBus:
    """
    Publish/subscribe for voice events across multiple backend instances.
    Example use: admin broadcasts "end all calls" to all instances.
    """

    CHANNEL_PREFIX = "voice:events:"

    def __init__(self, redis_url: str) -> None:
        self._pub_client = aioredis.from_url(redis_url)
        self._sub_client = aioredis.from_url(redis_url)

    async def publish(self, session_id: str, event: dict) -> None:
        channel = f"{self.CHANNEL_PREFIX}{session_id}"
        await self._pub_client.publish(channel, json.dumps(event))

    async def subscribe(self, session_id: str):
        """Async generator yielding events for a session."""
        pubsub = self._sub_client.pubsub()
        channel = f"{self.CHANNEL_PREFIX}{session_id}"
        await pubsub.subscribe(channel)
        try:
            async for message in pubsub.listen():
                if message["type"] == "message":
                    yield json.loads(message["data"])
        finally:
            await pubsub.unsubscribe(channel)
            await pubsub.close()
```

### Jitter Buffer

Packets over the internet arrive out of order and with variable delay. A jitter buffer holds frames for a short window (20–80ms) to smooth playback.

```python
import asyncio
import heapq
import logging
import time

logger = logging.getLogger(__name__)


class JitterBuffer:
    """
    Simple jitter buffer for smoothing packet arrival.
    Holds frames for BUFFER_MS before releasing them.
    """

    BUFFER_MS = 40  # 40ms buffer — tradeoff: higher = smoother but more latency
    FRAME_MS = 20

    def __init__(self) -> None:
        self._heap: list[tuple[float, int, bytes]] = []  # (timestamp, seq, frame)
        self._seq_counter = 0
        self._play_cursor: float = 0.0

    def push(self, frame: bytes, timestamp_ms: float | None = None) -> None:
        ts = timestamp_ms if timestamp_ms is not None else time.monotonic() * 1_000
        heapq.heappush(self._heap, (ts, self._seq_counter, frame))
        self._seq_counter += 1

    def pop(self) -> bytes | None:
        """Return next frame if it has been buffered long enough."""
        if not self._heap:
            return None
        now_ms = time.monotonic() * 1_000
        oldest_ts, _, frame = self._heap[0]
        if now_ms - oldest_ts >= self.BUFFER_MS:
            heapq.heappop(self._heap)
            return frame
        return None

    def drain(self) -> list[bytes]:
        """Return all buffered frames, ordered by timestamp."""
        frames = [frame for _, _, frame in sorted(self._heap)]
        self._heap.clear()
        return frames
```

### Rate Limiting Concurrent Calls

```python
from __future__ import annotations

import asyncio
import logging

import redis.asyncio as aioredis

logger = logging.getLogger(__name__)

MAX_CONCURRENT_CALLS_PER_USER = 2
MAX_CONCURRENT_CALLS_PER_TENANT = 50


class CallRateLimiter:
    """
    Enforces per-user and per-tenant concurrent call limits using Redis counters.
    Uses atomic INCR + EXPIRE pattern.
    """

    def __init__(self, redis_client: aioredis.Redis) -> None:
        self._redis = redis_client

    async def acquire(self, user_id: str, tenant_id: str) -> bool:
        """
        Atomically increment call counters.
        Returns False if limit exceeded (do not start the call).
        """
        user_key = f"calls:active:user:{user_id}"
        tenant_key = f"calls:active:tenant:{tenant_id}"

        async with self._redis.pipeline(transaction=True) as pipe:
            try:
                await pipe.watch(user_key, tenant_key)
                user_count = int(await pipe.get(user_key) or 0)
                tenant_count = int(await pipe.get(tenant_key) or 0)

                if user_count >= MAX_CONCURRENT_CALLS_PER_USER:
                    logger.warning(
                        "call_limit_user user_id=%s current=%d", user_id, user_count
                    )
                    return False
                if tenant_count >= MAX_CONCURRENT_CALLS_PER_TENANT:
                    logger.warning(
                        "call_limit_tenant tenant_id=%s current=%d",
                        tenant_id,
                        tenant_count,
                    )
                    return False

                pipe.multi()
                pipe.incr(user_key)
                pipe.expire(user_key, 3_600)  # safety TTL — 1 hour
                pipe.incr(tenant_key)
                pipe.expire(tenant_key, 3_600)
                await pipe.execute()
                return True

            except aioredis.WatchError:
                return False  # retry caller should handle this

    async def release(self, user_id: str, tenant_id: str) -> None:
        """Decrement call counters when call ends."""
        async with self._redis.pipeline() as pipe:
            pipe.decr(f"calls:active:user:{user_id}")
            pipe.decr(f"calls:active:tenant:{tenant_id}")
            await pipe.execute()
```

### Audio Recording Consent and GDPR

```python
from __future__ import annotations

import logging
from dataclasses import dataclass
from datetime import datetime, timezone
from enum import Enum

logger = logging.getLogger(__name__)


class RecordingConsent(str, Enum):
    GRANTED = "granted"
    DENIED = "denied"
    NOT_COLLECTED = "not_collected"


@dataclass
class CallRecord:
    call_id: str
    user_id: str
    tenant_id: str
    started_at: datetime
    ended_at: datetime | None
    duration_seconds: int
    recording_consent: RecordingConsent
    recording_s3_key: str | None  # None if consent denied or not recorded
    transcript_s3_key: str | None
    gdpr_erasure_requested: bool = False
    gdpr_erasure_at: datetime | None = None


# GDPR: audio data retention — 90 days maximum for most jurisdictions
AUDIO_RETENTION_DAYS = 90
TRANSCRIPT_RETENTION_DAYS = 365  # text transcripts may have longer retention


async def request_gdpr_erasure(call_id: str, db) -> None:
    """
    Mark call record for erasure. A background job handles actual deletion.
    Cryptographic erasure: delete the DEK from the KMS, not the ciphertext.
    """
    await db.execute(
        """
        UPDATE call_records
        SET gdpr_erasure_requested = TRUE,
            gdpr_erasure_at = NOW()
        WHERE call_id = :call_id
        """,
        {"call_id": call_id},
    )
    logger.info("gdpr_erasure_requested call_id=%s", call_id)
    # Actual S3 deletion happens in a scheduled background job
    # that respects the retention policy and any legal hold flags
```

### Monitoring: Key Metrics

```python
from __future__ import annotations

import time
from dataclasses import dataclass, field

from prometheus_client import Counter, Gauge, Histogram

# Prometheus metrics
CALL_DURATION = Histogram(
    "voice_call_duration_seconds",
    "Duration of voice calls",
    ["tenant_id"],
    buckets=[10, 30, 60, 120, 300, 600, 1800],
)

E2E_LATENCY = Histogram(
    "voice_e2e_latency_ms",
    "End-to-end latency per turn (VAD end → first audio byte)",
    ["tenant_id"],
    buckets=[100, 200, 300, 400, 500, 750, 1000, 2000],
)

ACTIVE_CALLS = Gauge(
    "voice_active_calls",
    "Current number of active voice calls",
    ["tenant_id"],
)

DROPOUT_RATE = Counter(
    "voice_connection_drops_total",
    "Number of unexpected WebSocket disconnections",
    ["tenant_id", "reason"],
)

WER_METRIC = Histogram(
    "voice_word_error_rate",
    "Word Error Rate from STT (sampled calls with transcripts)",
    buckets=[0.0, 0.05, 0.10, 0.15, 0.20, 0.30, 0.50, 1.0],
)


@dataclass
class CallMetricsCollector:
    tenant_id: str
    call_id: str
    _start_time: float = field(default_factory=time.monotonic)

    def __post_init__(self) -> None:
        ACTIVE_CALLS.labels(tenant_id=self.tenant_id).inc()

    def record_turn_latency(self, latency_ms: float) -> None:
        E2E_LATENCY.labels(tenant_id=self.tenant_id).observe(latency_ms)

    def record_wer(self, wer: float) -> None:
        WER_METRIC.observe(wer)

    def on_disconnect(self, reason: str = "normal") -> None:
        duration = time.monotonic() - self._start_time
        CALL_DURATION.labels(tenant_id=self.tenant_id).observe(duration)
        ACTIVE_CALLS.labels(tenant_id=self.tenant_id).dec()
        if reason != "normal":
            DROPOUT_RATE.labels(
                tenant_id=self.tenant_id, reason=reason
            ).inc()
```

### Alerting Thresholds for Voice AI

| Metric | Warning | Critical | Action |
|---|---|---|---|
| E2E latency P95 | > 500ms | > 1,000ms | Scale LLM, check network |
| E2E latency P99 | > 800ms | > 2,000ms | Investigate bottleneck |
| Active calls | > 80% capacity | > 95% capacity | Scale horizontally |
| Connection dropout rate | > 2% | > 5% | Check WebSocket infra |
| WER (Word Error Rate) | > 10% | > 20% | Check STT config, audio quality |
| Call duration P99 | > 15 min | > 30 min | Check for stuck sessions |

```python
# Grafana alert rule (YAML)
ALERT_RULES = """
groups:
  - name: voice_ai
    rules:
      - alert: VoiceLatencyHigh
        expr: histogram_quantile(0.95, voice_e2e_latency_ms) > 500
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Voice P95 latency {{ $value }}ms exceeds 500ms"

      - alert: VoiceDropoutRateHigh
        expr: rate(voice_connection_drops_total[5m]) / rate(voice_active_calls[5m]) > 0.02
        for: 3m
        labels:
          severity: critical
        annotations:
          summary: "Voice dropout rate {{ $value | humanizePercentage }} over 2%"
"""
```

### Connection Pooling for Ultravox API

```python
from __future__ import annotations

import asyncio
import logging
from contextlib import asynccontextmanager

import httpx

logger = logging.getLogger(__name__)

# Shared client — created at startup, reused across all requests
_ultravox_client: httpx.AsyncClient | None = None


def get_ultravox_client() -> httpx.AsyncClient:
    """Dependency injection accessor for the shared Ultravox HTTP client."""
    if _ultravox_client is None:
        raise RuntimeError("Ultravox client not initialized — call setup_clients() first")
    return _ultravox_client


@asynccontextmanager
async def lifespan(app):
    """FastAPI lifespan handler — set up and tear down shared clients."""
    global _ultravox_client
    _ultravox_client = httpx.AsyncClient(
        base_url="https://api.ultravox.ai",
        headers={"X-API-Key": os.environ["ULTRAVOX_API_KEY"]},
        timeout=httpx.Timeout(connect=5.0, read=30.0, write=10.0, pool=5.0),
        limits=httpx.Limits(
            max_connections=100,
            max_keepalive_connections=20,
        ),
    )
    logger.info("ultravox_client_initialized")
    yield
    await _ultravox_client.aclose()
    logger.info("ultravox_client_closed")
```

---

## Quick Reference: Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Creating httpx.AsyncClient per request | New TCP+TLS conn per call, +200ms | Create once at startup, inject |
| Sending audio in large chunks | High latency (buffering), VAD fails | Send 20ms frames on a 20ms timer |
| Not handling `playback_clear_buffer` | Agent keeps talking after barge-in | Drain audio queue on this event |
| `audioop.ratecv` direction wrong | Garbled audio to Twilio | mulaw is 8kHz input, 16kHz output |
| No silence TTL on Redis sessions | Memory leak over time | Always set `EXPIRE` |
| Sticky sessions missing | WebSocket drops randomly | Configure `ip_hash` in nginx |
| Not draining event queue on disconnect | Events lost, session leaks | Check `not session.event_queue.empty()` |
| Fixed window rate limiting | 2x burst at boundary | Use sliding window with Redis |

---

## References

- [Ultravox WebSocket Documentation](https://docs.ultravox.ai/apps/websockets)
- [Twilio Media Streams WebSocket Messages](https://www.twilio.com/docs/voice/media-streams/websocket-messages)
- [Silero VAD GitHub](https://github.com/snakers4/silero-vad)
- [Voice AI Latency Budget Analysis — Chanl Blog](https://www.channel.tel/blog/voice-ai-pipeline-stt-tts-latency-budget)
- [Pipecat Voice AI Framework](https://modal.com/blog/low-latency-voice-bot)
- [Ultravox WebSocket Example Client](https://github.com/fixie-ai/ultravox-websocket-example-client)

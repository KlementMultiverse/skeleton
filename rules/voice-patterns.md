---
description: Real-time voice AI rules — latency budgets, WebSocket audio, Ultravox, VAD, session state, telephony.
paths: ["*.py", "*.toml", "websocket*", "voice*", "audio*"]
---

# Voice AI Patterns — Production Rules

---

## Latency Budget (rules 1–10)

1. ALWAYS decompose end-to-end latency into exactly five named components before optimizing: VAD silence detection, STT finalization, LLM TTFT, TTS TTFA, and network RTT — optimizing "latency" without this decomposition means guessing which leg to fix.

2. ALWAYS measure each latency component independently using `time.monotonic()` wrapped tightly around that component only — `time.time()` is not monotonic and can jump on NTP sync; `time.monotonic()` never goes backward.

3. NEVER set a voice latency SLO above 600ms end-to-end for interactive conversation — above 600ms users perceive a pause and begin repeating themselves; above 1,000ms they assume the system failed.

4. ALWAYS stream LLM output token-by-token to TTS rather than buffering the full response — streaming reduces first-audio latency from (TTFT + full generation time) to (TTFT + first-sentence generation time), typically a 300–800ms reduction.

5. ALWAYS allocate the largest tolerance in your latency budget to LLM TTFT because it is compute-bound and hardest to shrink — VAD, STT, and TTS all have sub-100ms achievable targets; LLM inference rarely goes below 100ms.

6. NEVER use `time.sleep()` or `asyncio.sleep()` to pace audio frame sending — use a real-time clock reference instead; sleep drift accumulates over long calls and causes VAD timing failures in the remote platform.

7. ALWAYS log latency breakdowns per turn at INFO level with component labels — `vad=Xms stt=Xms llm_ttft=Xms tts_ttfa=Xms total=Xms` — without this, production regressions are invisible until P99 alerts fire.

8. ALWAYS choose WebRTC transport over PSTN for latency-sensitive applications when both are available — PSTN adds 150–700ms of carrier switching overhead before AI processing begins, consuming the entire latency budget before your code runs.

9. NEVER route a voice turn through more than three sequential network hops — each hop adds RTT; a chain of client → your server → STT service → LLM API → TTS service → your server → client has four hops and will exceed 600ms even with fast individual services.

10. ALWAYS set `max_tokens` on every LLM call in a voice pipeline — an unbounded generation produces minutes of TTS audio that the user cannot interrupt without barge-in support; voice responses should be limited to 100–200 tokens.

---

## Audio Formats and Streaming (rules 11–22)

11. ALWAYS use PCM16 (s16le) as your internal audio format throughout the processing pipeline — convert to PCM16 on ingress and from PCM16 on egress; mixed formats in the middle cause silent arithmetic errors that produce garbled audio.

12. ALWAYS match sample rates at every conversion boundary — passing 24kHz audio to a component expecting 16kHz produces audio that plays at the wrong pitch and speed; assert `sample_rate == expected` at every function entry point that processes audio.

13. NEVER use `audioop.ratecv` direction backwards for Twilio — mulaw from Twilio is 8kHz and must be upsampled to 16kHz for STT; PCM16 to send back to Twilio must be downsampled from 16kHz to 8kHz before `lin2ulaw` encoding.

14. ALWAYS send audio frames at the exact cadence of the frame size — for 20ms PCM16 at 16kHz, send 640 bytes every 20ms; Ultravox and Silero VAD rely on consistent timing for VAD and interruption detection; burst-and-wait delivery degrades both.

15. NEVER accumulate audio into a single large buffer before processing — process in 20ms frames as they arrive; buffering adds exactly `buffer_duration` to your latency floor and cannot be recovered downstream.

16. ALWAYS compute frame size in bytes from first principles: `frame_bytes = int(sample_rate * frame_duration_ms / 1000) * channels * bytes_per_sample` — never hardcode frame sizes as magic numbers without this formula as a comment.

17. ALWAYS use `struct.unpack` to convert PCM16 bytes to samples, not `numpy.frombuffer` in async hot paths — numpy import and GIL interaction adds latency spikes; `struct.unpack` is pure C and predictable.

18. NEVER base64-encode audio for internal WebSocket transport — base64 inflates bytes by 33%; use binary WebSocket frames (send `bytes`, not `str`) for all PCM16 audio between your own services; base64 is only required for Twilio Media Streams which are JSON-wrapped.

19. ALWAYS validate the byte length of incoming audio frames before processing — a partial frame from network fragmentation passed to Silero VAD or a codec encoder will raise an error or silently produce garbage; assert `len(frame) == expected_bytes` and buffer until complete.

20. ALWAYS use Opus at 16kHz mono for browser-to-server streaming when WebRTC is not available — Opus delivers near-transparent quality at 32kbps, 8x smaller than PCM16 at 16kHz; PCM16 is appropriate only for server-to-server where bandwidth is not constrained.

21. NEVER use `audioop` functions with a `sample_width` argument that does not match the actual PCM encoding — `audioop.rms(data, 2)` means 2-byte (16-bit) samples; passing `1` for mulaw returns nonsense RMS values that break energy-based VAD.

22. ALWAYS apply a 50–100ms of silence padding to audio before sending to STT models — most STT models have a warm-up period and clip the first syllable without leading silence; prepend `b"\x00" * padding_bytes` to the audio chunk.

---

## WebSocket Connection Management (rules 23–32)

23. ALWAYS accept the WebSocket connection before any async operations that might fail — call `await websocket.accept()` as the first statement; if you perform DB lookups before accepting, the browser will time out the upgrade handshake while you work.

24. ALWAYS authenticate via the first JSON message after handshake, not via query parameters — query parameters appear in server access logs in plaintext; the first message is encrypted by TLS and not logged by default.

25. NEVER store authentication state in WebSocket connection scope variables without validating token expiry on every turn — a token valid at connection time may expire during a long call; validate expiry at least every 5 minutes for long-running sessions.

26. ALWAYS use `asyncio.TaskGroup` (Python 3.11+) to run sender and receiver tasks concurrently — if either task fails, `TaskGroup` cancels the other; using `asyncio.gather` without `return_exceptions=False` lets one task silently fail while the other runs forever.

27. ALWAYS set `ping_interval=20` and `ping_timeout=10` on WebSocket connections to remote voice platforms — without keepalive pings, load balancers and NAT gateways silently drop idle connections after 60–300 seconds; voice calls with quiet periods will disconnect unexpectedly.

28. NEVER call `await websocket.receive()` from multiple concurrent coroutines — the websockets library is not thread-safe on the receive path; use a single receiver coroutine and distribute messages via an `asyncio.Queue`.

29. ALWAYS handle `WebSocketDisconnect` and `websockets.exceptions.ConnectionClosed` separately from general exceptions — disconnect is normal; log at INFO, clean up resources; unexpected exceptions should log at ERROR with `exc_info=True`.

30. ALWAYS drain the send queue gracefully on disconnect — when the client disconnects, attempt to send any queued audio frames with a short timeout (100ms) before closing; abrupt closure leaves partial audio frames in flight that produce audible glitches.

31. NEVER use `proxy_read_timeout` below 3,600 seconds in nginx for voice WebSockets — the default nginx proxy timeout is 60 seconds; a 5-minute call will be dropped at the proxy before the WebSocket client or server decide to close.

32. ALWAYS include a bounded `asyncio.Queue` between the audio source and the WebSocket sender with a maximum of 50–100 frames — an unbounded queue means an LLM processing slowdown causes the queue to grow without limit; when the queue is full, drop the oldest frame rather than blocking the audio input pipeline.

---

## Ultravox Integration (rules 33–44)

33. ALWAYS create a single `httpx.AsyncClient` at application startup with the Ultravox API key in the default headers and reuse it across all calls — creating a new client per call opens a new TCP+TLS connection each time, adding 50–200ms overhead and depleting file descriptors under load.

34. NEVER expose the Ultravox `callId` to end users in URLs or WebSocket messages — the `callId` can be used to probe call metadata; treat it as an internal identifier, store it server-side, and map it from a user-facing session ID.

35. ALWAYS send silence frames (640 bytes of `\x00` at 16kHz PCM16) when there is no user audio to send — Ultravox relies on a continuous audio stream for VAD timing; gaps in the audio stream cause the VAD to misfire end-of-speech events.

36. ALWAYS set `clientBufferSizeMs` to 60ms for interactive use cases — lower values (< 40ms) cause audio underflow and choppy output on networks with > 20ms jitter; higher values (> 200ms) add perceptible latency; 60ms is the documented default and a good starting point.

37. NEVER handle binary and text WebSocket messages from Ultravox with the same code path — binary messages are raw s16le PCM audio that must be forwarded directly to the audio output; text messages are JSON events that must be parsed; conflating them causes JSON parsing errors on audio bytes or silent audio corruption.

38. ALWAYS handle the `playback_clear_buffer` event from Ultravox immediately — this event means the user interrupted (barged in) while the agent was speaking; drain your local audio output queue within the same event loop iteration to stop playback.

39. ALWAYS check the `final` flag on `transcript` events before acting on the transcript text — partial transcripts (non-final) are intermediate guesses that may change; only final transcripts should trigger downstream actions like slot filling or DB writes.

40. NEVER retry a failed Ultravox call creation with the same `systemPrompt` if the error is a 400 — a 400 means the request was malformed (prompt too long, invalid voice name, unsupported model); fix the payload before retrying; retrying the same 400-producing request wastes quota.

41. ALWAYS reconnect to Ultravox with exponential backoff on WebSocket `ConnectionClosed` errors with codes 1001–1006 — these codes indicate transient failures (network drop, server restart); do not reconnect on code 1000 (normal closure) or 4xxx (authentication failure).

42. ALWAYS store the Ultravox `joinUrl` in your session store immediately after call creation — the `joinUrl` is single-use and time-limited; if your WebSocket connection drops before connecting, you cannot retrieve the URL again; persist it so reconnection attempts can use it within the time window.

43. NEVER run Ultravox WebSocket connections from a browser or mobile client directly — Ultravox server WebSocket is designed for server-to-server communication; TCP-based audio has head-of-line blocking that makes it unsuitable for browser use; use Ultravox client SDKs with WebRTC for browser clients.

44. ALWAYS configure `outputSampleRate` to match your downstream audio output requirements — if your client expects 24kHz for high-quality playback, set `outputSampleRate=24000`; mismatched sample rates cause pitch and speed distortion without any error message.

---

## VAD and Interruption Handling (rules 45–54)

45. ALWAYS use Silero VAD over energy-based VAD for any deployment with background noise — energy-based VAD fires on keyboard sounds, music, and HVAC noise; Silero VAD distinguishes human speech with < 1ms processing time per 32ms frame at under 0.5% CPU on a single core.

46. NEVER share a single Silero VAD model instance across concurrent calls — the Silero model maintains LSTM hidden state between frames; sharing an instance across calls means frames from call A corrupt the VAD state for call B; instantiate one `SileroVAD` per active call.

47. ALWAYS tune end-of-speech silence duration to conversation context — use 300ms for short command-and-response turns, 500ms for conversational replies, and 700ms for free-form dictation; a single global threshold causes either premature cutoff (too short) or frustrating delays (too long).

48. ALWAYS implement barge-in by cancelling the TTS playback task via `asyncio.Task.cancel()` — do not set a flag and check it in a loop; task cancellation is immediate and deterministic; flag-based approaches leave up to one frame of unwanted audio playing after the flag is set.

49. NEVER cancel the Ultravox WebSocket connection on barge-in — barge-in means the user interrupted the agent's speech; cancel local TTS audio playback and drain the audio output queue; the Ultravox session itself should remain connected so the new user utterance is captured.

50. ALWAYS feed silence frames to the VAD during barge-in response (while agent audio is muted) — if you stop sending audio to the VAD, it will miss the speech-end event and hold the STT pipeline open indefinitely; continue feeding audio to VAD even while suppressing audio output.

51. ALWAYS debounce VAD speech-start events with a minimum of two consecutive speech frames (64ms at 32ms frames) — a single frame of noise above the threshold triggers false speech-start events; requiring two consecutive frames eliminates single-frame transients with no perceptible delay.

52. NEVER disable VAD and rely solely on PTT (push-to-talk) for production voice AI — PTT requires the user to hold a button, which is incompatible with hands-free use cases and voice-first UX; VAD enables natural turn-taking; use PTT only as an accessibility fallback.

53. ALWAYS reset the Silero VAD internal state (`model.reset_states()`) between conversation turns — residual LSTM state from the previous utterance biases VAD predictions for the next utterance; reset at the start of each new turn to ensure clean detection.

54. ALWAYS log VAD speech-start and speech-end events at DEBUG level with a timestamp offset from turn start — `vad_speech_end offset_ms=320`; these logs are essential for diagnosing premature cutoff complaints ("the bot cut me off") from production users.

---

## Session State Management (rules 55–64)

55. ALWAYS store voice session state in Redis with a TTL of `max(idle_timeout, expected_turn_duration)` — never store it in application memory; when the instance handling the call restarts, the next instance must be able to resume; in-memory state is lost on restart.

56. ALWAYS use two TTLs per voice session: an idle TTL (2 minutes, reset on each turn) and a hard TTL (30 minutes, set once) — the idle TTL prevents zombie sessions from leaked connections; the hard TTL prevents sessions from living indefinitely if the idle TTL is never reset due to a bug.

57. NEVER use Redis `SET` without `EX` for voice session keys — sessions without TTLs accumulate indefinitely; a Redis instance handling 1,000 calls per hour with no TTL accumulates 720,000 sessions per month; always pass `ex=ttl_seconds`.

58. ALWAYS trim conversation history to the last N turns before storing in Redis — storing unbounded history exhausts Redis memory and increases serialization time per turn; keep the last 20 turns for context and store the full history in PostgreSQL for compliance.

59. ALWAYS serialize session state as JSON, not pickle — pickle state stored in Redis cannot be deserialized if your Python class definition changes between deploys; JSON survives schema evolution as long as you handle missing keys gracefully with `.get()`.

60. ALWAYS use `WATCH` + `MULTI`/`EXEC` (optimistic locking) when updating session state from multiple concurrent tasks — if the send task and the event handler task both write session state, one will silently overwrite the other without locking; use the Redis transaction pattern.

61. ALWAYS include a `schema_version` field in serialized session state — when you change the session schema (add a field, rename a key), increment the version; read code must handle both old and new versions during the deployment window.

62. NEVER look up session state by `user_id` alone — one user may have multiple concurrent sessions (two browser tabs, voice and text simultaneously); always key sessions by `session_id` and maintain a secondary index `user:sessions:{user_id}` → set of session IDs.

63. ALWAYS persist session state to PostgreSQL at session end (call completed or 30-minute hard timeout) — Redis is a cache, not a database of record; session history needed for billing, compliance, or support tickets must survive a Redis flush or cluster failure.

64. ALWAYS propagate `session_id` as a contextvar, not a function parameter, in async voice handlers — a session ID threaded through 10 function signatures is noise; `contextvars.ContextVar("session_id")` makes it available to any function in the call stack without parameter pollution.

---

## Production and Monitoring (rules 65–80)

65. ALWAYS configure sticky sessions (nginx `ip_hash` or cookie-based affinity) for any deployment with more than one backend instance — WebSocket connections are stateful; a load balancer that round-robins requests will route mid-call messages to a different instance that has no session state, causing immediate disconnection.

66. ALWAYS instrument every voice turn with a Prometheus histogram for end-to-end latency with buckets at 100, 200, 300, 400, 500, 750, 1000, and 2000ms — a histogram without fine-grained buckets below 500ms cannot distinguish "meeting SLO" from "barely missing SLO".

67. ALWAYS track active call count as a Prometheus Gauge, not a Counter — a Counter only goes up; active calls go up (call starts) and down (call ends); use `gauge.inc()` on start and `gauge.dec()` in the `finally` block of the call handler.

68. ALWAYS set a P95 latency alert at 500ms and a P99 alert at 1,000ms — P50 and mean latency hide tail latency problems that affect the worst-served users; the users who experience > 1,000ms latency are the most likely to report the AI as broken.

69. NEVER alert on per-call dropout events — individual disconnections are noise; alert on `rate(connection_drops[5m]) / rate(call_starts[5m]) > 0.02` — a 2% dropout rate over 5 minutes is a systemic problem; a single drop is normal network behavior.

70. ALWAYS record Word Error Rate (WER) on a sampled set of production calls by comparing STT transcripts to ground-truth transcripts from a QA set — WER above 15% degrades intent recognition accuracy enough to cause call failures; monitor WER as a quality metric, not just latency.

71. ALWAYS implement a circuit breaker on the Ultravox call creation API — if call creation fails 5 times in 60 seconds, open the circuit and return a graceful degradation response ("our voice AI is temporarily unavailable") rather than queuing hundreds of retries that will overwhelm the API on recovery.

72. NEVER store raw audio recordings on the application server filesystem — use S3 or equivalent object storage; the application server must be stateless to allow horizontal scaling; audio files are large (1 minute of PCM16 at 16kHz ≈ 1.9MB) and will fill local disk quickly.

73. ALWAYS implement recording consent collection before starting a call recording — log consent with a timestamp, the method of collection (IVR prompt, web checkbox), and the specific consent text shown; regulators require proof that consent was informed and explicit.

74. ALWAYS set audio recording retention to 90 days maximum unless a legal hold requires longer — audio recordings contain PII; GDPR and CCPA require data minimization; automated deletion after 90 days reduces breach surface and compliance risk.

75. ALWAYS implement per-user and per-tenant concurrent call limits using Redis atomic counters — without limits, a single tenant can exhaust all Ultravox connections and degrade service for all other tenants; enforce before creating the call, not after.

76. ALWAYS add a jitter buffer of 20–80ms before audio playback on the client side — internet packet arrival has natural variance of ±20ms; without a jitter buffer, audio playback stalls every few seconds when a packet arrives 25ms late; 40ms buffer eliminates this completely.

77. ALWAYS use structured logging with `call_id`, `user_id`, `tenant_id`, and `turn_number` on every log line in the voice handler — voice calls generate hundreds of log lines; without these identifiers, a support ticket about "the call at 2pm" requires manual correlation across thousands of unrelated log lines.

78. NEVER deploy a voice AI endpoint without load testing it at 2x expected peak concurrent calls — WebSocket connections hold open file descriptors and event loop resources; a production spike to 500 concurrent calls on a server load-tested only to 100 will cause event loop starvation and cascading timeouts.

79. ALWAYS implement a GDPR audio erasure endpoint that deletes the encryption key (DEK) from your KMS rather than deleting the S3 object — cryptographic erasure makes the ciphertext permanently unreadable without modifying backup files or audit logs; physical deletion of S3 objects does not remove data from versioned buckets or cross-region replicas.

80. ALWAYS test barge-in behavior under network degradation (introduce 100ms artificial latency and 1% packet loss) before production launch — barge-in relies on timely VAD event delivery; under degraded conditions the `playback_clear_buffer` event may arrive 200ms late, causing 200ms of unwanted audio after the user starts speaking; test and tune your queue drain timing under realistic conditions.

---

## Acoustic Echo Cancellation

81. ALWAYS apply acoustic echo cancellation (AEC) on the audio path before sending to STT — without AEC, the agent hears its own TTS playback as user speech, triggering false VAD events and sending the agent's own words back to the LLM as a new user turn; gate the microphone input entirely during TTS playback using your `is_agent_speaking` flag and resume only after `playback_clear_buffer` is received.

82. NEVER rely solely on energy-gating to suppress echo during agent speech — energy gating cannot distinguish a loud user voice from the agent's own audio reflected back; use a proper AEC implementation (`webrtc-audio-processing` on the server) that maintains a reference signal of outbound audio and subtracts it from inbound capture.

---

## Noise Suppression

83. ALWAYS apply noise suppression before VAD, not after — running VAD on raw noisy audio causes HVAC hum and keyboard clicks to trigger continuous speech-start events; use `noisereduce` (spectral gating) or `rnnoise` for real-time per-frame suppression at under 2ms per 10ms frame.

84. NEVER apply noise suppression at aggression level above 2 (on a 0-3 scale) for voice pipelines — levels 3+ introduce audible musical noise artifacts and suppress consonants, raising WER by 5-15%; tune to the lowest level that eliminates the dominant noise in your target environment.

---

## Sample Rate Conversion

85. ALWAYS use `libsamplerate` (via `samplerate` Python bindings) or `resampy` with the `kaiser_best` kernel for sample rate conversion — Python's `audioop.ratecv` uses linear interpolation which introduces aliasing artifacts audible above 4kHz; high-quality resampling filters add under 1ms of latency per 20ms frame.

86. NEVER resample audio more than once in a pipeline — each resample pass introduces quantization error; if your pipeline converts 48kHz browser audio to 16kHz for STT and then 8kHz for PSTN output, do the 48kHz-to-8kHz conversion in a single pass.

---

## Packet Loss Concealment

87. ALWAYS implement packet loss concealment (PLC) for UDP-based audio transport by repeating or interpolating the previous frame when a gap is detected — a 20ms silent gap degrades STT accuracy by 8-12%; the Opus codec provides built-in PLC via `opus_decode` with a null input, which should be preferred over silence insertion.

88. ALWAYS track packet sequence numbers on every inbound RTP or WebRTC audio frame to detect gaps — do not assume frames arrive in order; a sequence number jump of N means N-1 frames were lost and must be concealed before forwarding to the VAD or STT pipeline.

---

## Audio Normalization

89. ALWAYS apply RMS normalization before sending to STT — target -18 to -12 dBFS; audio from a distant microphone may arrive at -40 dBFS, causing STT underperformance because models are trained on speech near -18 dBFS; compute `rms = math.sqrt(sum(s**2 for s in samples) / len(samples))` and scale by `target_rms / (rms + 1e-9)`.

90. NEVER normalize audio frames individually in a 20ms window — RMS over 320 samples has high variance; compute normalization gain over a rolling 500ms window and apply it with a gain ramp of no more than 3dB per frame to prevent audible pumping artifacts.

---

## WebRTC vs WebSocket Transport

91. ALWAYS use WebRTC (not WebSocket) for browser-to-server audio when round-trip latency must stay below 200ms — WebSocket is TCP-based and suffers head-of-line blocking where a single lost packet stalls all subsequent packets; WebRTC uses SRTP over UDP with its own jitter buffer and PLC, bypassing TCP retransmission entirely.

92. NEVER use WebRTC for server-to-server voice pipeline segments — WebRTC's ICE negotiation, DTLS handshake, and SRTP overhead are designed for untrusted peer-to-peer networks; for trusted server-to-server audio, a plain WebSocket with TLS and binary frames has lower setup latency and simpler failure modes.

---

## WebRTC ICE/STUN/TURN

93. ALWAYS provide both a STUN server and a TURN server in your WebRTC ICE configuration — STUN alone fails for clients behind symmetric NAT (corporate firewalls, 4G carriers), which account for ~15-20% of real-world network topologies; without TURN those clients silently fail to establish audio.

94. NEVER use a public STUN server (stun.l.google.com) in production — public STUN servers have no SLA, log connection metadata, and have caused outages when rate-limited; run your own coturn instance with HMAC-based ephemeral credentials and a 24-hour TTL.

---

## WebSocket Compression

95. ALWAYS disable `permessage-deflate` on WebSocket connections used for audio transport — PCM16 and Opus are already incompressible so deflate adds 0-2% size reduction while imposing 0.5-2ms of CPU latency per frame; disable via `websockets.connect(..., compression=None)`.

96. ALWAYS enable `permessage-deflate` on WebSocket connections used for JSON control messages (transcripts, tool calls, session events) — control messages compress 60-80% with no perceptible latency cost because they are not on the real-time audio path.

---

## WebSocket Close Codes

97. ALWAYS map WebSocket close codes to distinct retry behaviors: 1000/1001 (normal closure) → clean up session, no retry; 1002-1006 (protocol/network errors) → exponential backoff reconnection starting at 500ms; 4000-4099 (application auth failures) → never retry without first refreshing credentials.

98. NEVER log WebSocket close code 1000 at WARN or ERROR level — code 1000 is normal closure expected at the end of every call; logging it at elevated severity fills dashboards with noise; log at DEBUG with reason field; reserve WARN for codes 1001-1006 and ERROR for 4xxx.


---

## Telephony — Twilio Media Streams

99. ALWAYS fan out a Twilio Media Stream to multiple consumers by reading each inbound frame once and writing it to each downstream consumer's queue — Twilio sends a single stream to one `<Stream url="">` endpoint; never re-establish a second Twilio stream to split consumers because that doubles PSTN audio cost.

100. NEVER expose your SIP signaling port (5060 UDP/TCP) to the public internet without IP allowlisting to your SIP trunk provider's CIDR ranges — SIP scanners continuously probe for open 5060 ports and brute-force credentials within minutes; unauthenticated SIP registration leads to toll fraud charges that can exceed $10,000 in hours.

---

## Telephony — SIP and Routing

101. ALWAYS terminate SIP trunks at a media gateway (FreeSWITCH, Asterisk, or hosted SBC) rather than writing raw SIP parsing in application code — SIP state machines handle re-INVITE, CANCEL race conditions, and RTP negotiation that take years to implement correctly; your application should receive clean audio via WebSocket or RTP from the gateway.

102. ALWAYS configure your SBC to transcode inbound G.711 ulaw/alaw to PCM16 before forwarding to your AI pipeline — most STT and VAD models expect PCM16; transcoding at the SBC boundary keeps your pipeline codec-agnostic.

103. ALWAYS provision phone numbers in the same region as your voice AI processing servers — a US phone number handled by a server in EU adds 150-200ms of transatlantic RTT to every audio packet before your VAD even runs.

104. ALWAYS store phone number → tenant → routing configuration in a database with a 60-second cache TTL — routing changes must take effect within one cache TTL without a redeploy; hardcoded routing requires a deploy and causes downtime during number transfers.

---

## Telephony — Agent Handoff

105. ALWAYS implement warm transfer by placing the AI session on hold, establishing a consultation leg to the human agent, announcing caller context, and only then bridging the call — cold transfer drops the caller with no context; the human agent must re-ask for information already provided, destroying the CX benefit of the AI front-end.

106. ALWAYS pass a structured context payload to the human agent at transfer time: `caller_name`, `verified_intent`, `sentiment_score`, `last_3_turns_summary`, and `session_id` — use CTI screen-pop via your contact center platform's API; the session_id links the agent desktop to the full call transcript for compliance review.

107. NEVER mark a warm transfer complete until you receive acknowledgment from the destination agent leg — if the agent leg is not answered within 30 seconds, return the caller to the AI or queue rather than dropping the call.

---

## Call Recording Consent

108. ALWAYS collect explicit call recording consent before recording when callers may be in US two-party consent states (CA, FL, IL, MD, MA, MI, MT, NV, NH, OR, PA, WA) — one-party-consent disclaimers are insufficient; per-violation statutory damages are $2,500-$5,000 under state wiretapping statutes.

109. ALWAYS treat any call involving an EU or UK caller as requiring GDPR-compliant explicit opt-in consent for recording — legitimate interest is not a valid basis; record the consent event with a timestamp in your audit log and honor withdrawal mid-call by stopping recording and deleting the partial recording within 72 hours.

110. ALWAYS play a jurisdiction-specific consent disclosure based on the caller's area code before starting recording — maintain a mapping of area codes to two-party consent states and route callers from those states to a two-party consent IVR branch; log which branch was taken per call.

---

## Horizontal WebSocket Scaling

111. ALWAYS use Redis pub/sub or Redis Streams to fan out messages across horizontally scaled voice instances — when a REST API call updates a session on instance B but the WebSocket is held by instance A, publish the update to a Redis channel keyed by `session_id`; without this, cross-instance message delivery silently fails.

112. ALWAYS use Redis Streams (`XADD`/`XREAD`) rather than pub/sub for durable cross-instance delivery — pub/sub drops messages when no subscriber is listening (e.g., during a brief reconnect); Redis Streams persist messages with a consumer group enabling the reconnecting instance to replay missed events.

113. ALWAYS prefer cookie-based sticky sessions over IP-hash load balancing for voice WebSocket deployments — IP-hash fails for callers behind CGNAT where millions of mobile users share a single public IP; all CGNAT users hash to the same backend creating a hot-spot; cookie-based affinity routes each session independently of source IP.

---

## Voice Security

114. ALWAYS enforce per-tenant concurrent session limits with Redis `INCR` before creating the session and `DECR` in the `finally` block — using a PostgreSQL count query introduces a TOCTOU race where two simultaneous call-start requests both read count=N and both proceed; only Redis atomic `INCR` is race-free.

115. ALWAYS set a hard TTL on Redis concurrent-session counters equal to your maximum call duration as a safety net — if `DECR` is never called due to a process crash, the counter will be permanently inflated blocking all future calls for that tenant; the TTL ensures self-healing within one maximum call duration.

116. ALWAYS include a monotonically increasing sequence number and timestamp in every WebSocket audio frame metadata envelope — reject frames where `abs(frame_timestamp - server_time) > 5000ms` or `sequence_number <= last_seen_sequence`; this detects audio stream replay attacks.

117. NEVER authenticate a voice WebSocket session using only a long-lived static API key embedded in the URL — use short-lived (5-minute TTL) signed tokens with a `jti` claim marked used on first connection, preventing session hijacking via token replay.

118. NEVER log SRTP master key material, DTLS certificates, or ICE credentials at any log level — these allow decryption of the entire audio stream during their validity window; treat them identically to JWT signing secrets.

119. ALWAYS apply real-time PII redaction to STT transcripts before writing to any persistent store — use a regex-first pass for credit cards/SSNs/phones followed by an NER model for names and addresses; regex alone misses spoken-out PII; NER alone has >100ms latency; layer both.

120. ALWAYS redact PII from transcripts before sending to LLM context or RAG pipelines — a transcript containing a caller's SSN passed to an LLM API leaves your trust boundary and may appear in provider logs or fine-tuning datasets.

---

## HIPAA Compliance for Voice AI

121. ALWAYS classify voice transcripts, call recordings, and session metadata as Protected Health Information (PHI) when the caller is a patient and the topic is health-related — a transcript of "John Smith called about his diabetes medication" is PHI even if the recording is deleted.

122. ALWAYS execute a Business Associate Agreement (BAA) with every vendor processing voice data before going live — this includes telephony provider, STT provider, LLM provider, and object storage; operating without BAAs exposes you to HIPAA breach penalties up to $1.9M per violation category per year.

---

## Voice Minute Billing

123. ALWAYS bill voice calls using per-second duration rounded up to the nearest second and record three timestamps per call: `initiated_at`, `answered_at`, `ended_at` — bill on `ended_at - answered_at`; ring time before answer should never be passed to your customer as call duration.

124. ALWAYS reconcile your internal call duration records against your telephony provider's CDRs nightly — clock skew between your server and the carrier can produce systematic billing discrepancies of 1-3 seconds per call; at 100K calls per day that is 100K-300K seconds of unreconciled billing; alert when duration diverges by more than 2 seconds per call.


---

## STT Model Selection

125. ALWAYS evaluate STT providers on three axes before selecting: accuracy (WER on your domain vocabulary), latency (time to final transcript), and cost per minute — Deepgram Nova-2 leads on latency and cost; Whisper large-v3 leads on accuracy for non-English/technical vocabulary but requires self-hosting for real-time; benchmarking on your actual call audio is mandatory, synthetic benchmarks diverge significantly.

126. NEVER use OpenAI Whisper's hosted API for real-time voice pipelines — it is a batch endpoint with no streaming, adds 2-6 seconds of latency for a 10-second utterance; use Deepgram streaming, AssemblyAI streaming, or self-hosted faster-whisper for sub-500ms TTFT.

127. ALWAYS configure STT provider keyword boosting for domain-specific vocabulary — product names, medical terms, and proper nouns are systematically undertranscribed; Deepgram `keywords`, AssemblyAI `word_boost`, and Google STT `speech_contexts` allow term-level boosting.

---

## Streaming vs Batch STT

128. ALWAYS use streaming STT for interactive voice AI and batch STT only for post-call analytics and compliance transcription — streaming returns a partial transcript in 200-400ms enabling LLM processing to begin before the user finishes speaking.

129. NEVER act on partial/interim STT transcripts for downstream operations — partial transcripts change word-by-word as the model updates its hypothesis; only the `is_final=true` transcript (Deepgram) or `FinalTranscript` (AssemblyAI) should trigger downstream logic.

130. ALWAYS implement a streaming STT fallback to batch on connection failure — if the streaming WebSocket drops mid-utterance, buffer the remaining audio and submit to the batch endpoint; a call that produces no transcript is worse than a delayed one.

---

## STT Confidence Scores

131. ALWAYS read per-word confidence scores and flag turns where any content word (noun, verb, number) has confidence below 0.7 — low-confidence content words indicate the STT guessed; acting on them without confirmation causes errors attributed to the LLM rather than STT.

132. ALWAYS implement a confidence-gated re-prompt: when turn-level mean confidence is below 0.65, respond with a targeted clarification rather than proceeding with a potentially wrong transcript — distinguish between "low confidence on a specific slot" (re-prompt for that slot) and "low confidence across the whole utterance" (full re-prompt).

133. NEVER treat a zero-word transcript or filler-word-only transcript ("um", "uh") as a valid turn requiring a full LLM response — detect this at the STT output layer, respond with a brief acknowledgment, and extend the VAD timeout for the next turn.

---

## Word Error Rate

134. ALWAYS measure WER as `(S + D + I) / N` using `jiwer` with both `wer()` and `mer()` (Match Error Rate) — `mer` is more meaningful for short utterances; never report just one metric.

135. ALWAYS build a domain-specific WER test set of at least 50 utterances from your actual user population before selecting a STT provider — WER on studio audio underestimates production WER by 5-20 percentage points for conversational AI.

136. ALWAYS run automated WER regression tests on every change to STT configuration — a keyword boost weight that is too high degrades WER on non-boosted words; require WER delta < +1% absolute vs baseline before merging configuration changes.

---

## TTS Voice Cloning and Custom Voices

137. NEVER deploy a cloned voice without explicit written consent from the voice talent and legal review — the EU AI Act and US state laws (Illinois BIPA, Tennessee ELVIS Act) impose strict consent requirements and statutory damages for unauthorized voice cloning.

138. ALWAYS maintain a fallback to a stock TTS voice when your custom voice API endpoint is unavailable — custom voice endpoints have lower SLA than standard TTS endpoints; silence because a custom model is down is a worse user experience than a standard voice.

---

## Prosody and SSML

139. ALWAYS use SSML `<break>` tags at clause boundaries rather than inserting literal commas to control pacing — a comma added to control TTS pacing corrupts the transcript and downstream NLP; `<break time="300ms"/>` inserts a pause without altering the text fed to slot filling.

140. ALWAYS use SSML `<prosody rate="slow">` for number sequences, codes, and confirmations — TTS reading "your confirmation code is AB-4729" at default rate is frequently misheard; slowing rate by 20% on confirmation values reduces re-ask rates measurably.

---

## Turn-Taking Detection

141. ALWAYS implement prosodic end-of-turn detection as a second signal alongside VAD silence — falling intonation and reduced speaking rate at a clause boundary are stronger predictors of turn completion than silence alone; use Ultravox's built-in end-of-turn model or `pyannote/brouhaha` to reduce premature cutoffs by 30-40%.

142. ALWAYS distinguish between backchannel utterances ("mm-hmm", "yeah") and genuine turn-taking attempts — a backchannel while the agent is speaking should not trigger a full barge-in; detect 1-3 word utterances under 500ms with energy below speech baseline and suppress them from VAD turn-start logic.

143. NEVER treat all silences longer than your VAD timeout as end-of-turn — users with speech disorders and elderly users have longer inter-word pauses; implement a brief filler ("I'm listening...") rather than cutting them off at the first silence.

---

## Conversation Design

144. ALWAYS write a welcome message under 15 words that states the AI's role and the primary action available — users hang up within 5 seconds if they cannot determine what the bot can do; the welcome message is the highest-impact copy in the entire conversation.

145. ALWAYS implement a three-tier error recovery strategy: first re-prompt with a more specific prompt, second offer a menu of alternatives, third offer human escalation — abandoning the caller at any tier without escalation is unacceptable.

146. ALWAYS design re-prompt responses shorter and more directive than the initial prompt — a re-prompt longer than the original question adds cognitive load during confusion; narrow scope, don't expand it.

---

## Intent Recognition and Slot Filling

147. ALWAYS validate extracted slot values against domain-specific constraints before passing to business logic — a date slot must be parsed and checked for past/future validity; use Pydantic validators on the LLM's structured output to catch type errors and impossible values before they reach your DB.

148. ALWAYS implement slot confirmation for high-stakes slots (account numbers, amounts, dates) by reading the value back before committing — "I'll transfer $500 from checking. Is that correct?" prevents errors caused by STT misrecognition of critical values.

---

## Multi-Language Support

149. ALWAYS detect the caller's language from the first utterance before routing to any STT model — use Deepgram `detect_language=true` or AssemblyAI language detection on the first 3 seconds; routing a Spanish caller to an English STT model produces near-zero transcription accuracy.

150. ALWAYS provision separate STT model instances per language — per-language models have 15-30% lower WER than multilingual models; the latency and cost difference is negligible.

151. NEVER hard-code language routing in application code — store language-to-STT-model-to-TTS-voice mappings in configuration; adding a new language must not require a code deployment.

---

## Accessibility

152. ALWAYS tune VAD end-of-speech silence timeout to at least 700ms for all voice AI deployments — users with speech disorders (dysarthria, stuttering) and elderly users have longer inter-word pauses; a 300ms timeout cuts off 20-30% of these users mid-sentence.

153. ALWAYS provide a DTMF touch-tone fallback for every voice intent when deploying on telephony — callers with speech disorders, strong accents, or high ambient noise can use digit-press alternatives without changing your intent pipeline.

154. NEVER evaluate WER exclusively on native-speaker audio — heavy accents account for 30-40% of WER variance in real deployments; assemble a WER test set representing your actual caller demographics including non-native speakers and elderly speakers.

---

## Load Testing and Quality Metrics

155. ALWAYS load test voice WebSocket endpoints with a 3-stage ramp before production launch — stage 1: ramp from 0 to target concurrent sessions over 2 minutes; stage 2: sustain target load for 5 minutes; stage 3: ramp down; use k6 with a custom WebSocket script that sends real PCM16 audio frames at the correct 20ms interval and asserts `connection_setup_time < 800ms` at P95.

156. ALWAYS measure RTF (Real-Time Factor) per session — `RTF = processing_time / audio_duration`; RTF > 1.0 means the pipeline cannot keep up in real time and audio will buffer; alert when P95 RTF > 0.8 to catch degradation before RTF exceeds 1.0 under load.

157. ALWAYS track MOS (Mean Opinion Score) via a POLQA/VISQOL automated tool on a sample of recorded calls — MOS < 3.5 (out of 5.0) indicates perceptible quality degradation; correlate MOS drops with packet loss rate and jitter buffer depth to identify root cause.

158. ALWAYS categorize call drops into exactly 4 Prometheus counter buckets — `call_drops_total{reason="network_timeout"}`, `call_drops_total{reason="vad_silence_timeout"}`, `call_drops_total{reason="server_error"}`, `call_drops_total{reason="client_disconnect"}`; a single undifferentiated `call_drops_total` counter is unactionable when diagnosing production incidents.

---

## Testing Without Real Audio

159. ALWAYS test voice pipelines with synthetic PCM16 WAV fixtures generated from a script — use `scipy.io.wavfile.write` or `ffmpeg -f lavfi -i "sine=frequency=440:duration=5" -ar 16000 -ac 1 -f s16le` to generate known-content audio; tests that require a real microphone are not runnable in CI.

160. ALWAYS use TTS-generated audio as ground-truth test input for STT integration tests — generate test audio with a deterministic TTS voice, transcribe with the STT under test, measure WER against the known TTS input string; this gives reproducible STT quality regression tests without human recording sessions.

161. ALWAYS test WebSocket voice endpoints in integration tests by starting a real uvicorn server on a random port and connecting with a WebSocket client — do not test via ASGI transport; WebSocket upgrade negotiation and binary frame handling behave differently in transport-layer vs real TCP tests.

---

## Chaos and Reliability Testing

162. ALWAYS run chaos tests for voice pipelines using toxiproxy to simulate packet loss and latency spikes — inject 2% packet loss and 200ms jitter on the WebSocket connection and verify that call quality degrades gracefully (RTF stays < 1.0, PLC activates) rather than the session crashing with an unhandled exception.

163. ALWAYS test the LLM timeout path explicitly — use `tc netem delay 5000ms` to delay the LLM response beyond your 3-second TTFT SLO and assert that the system plays a hold phrase (`"One moment..."`) and does not drop the call.

---

## Buffer Management

164. ALWAYS detect audio buffer underrun by monitoring the jitter buffer depth metric — a jitter buffer depth below 40ms means the playback thread is consuming faster than the network delivers; insert Packet Loss Concealment (PLC) comfort noise (RFC 3389 CN payload) rather than silence or stuttering artifacts; silence artifacts cause callers to believe the call dropped.

165. ALWAYS generate comfort noise during buffer underruns using the CN payload matching the active codec — for PCM16 use a low-level white noise at -60 dBFS; for Opus embed a `ComfortNoise()` frame; do not use silence (0-value samples) which triggers end-of-speech detection in VAD and causes premature turn-taking.

---

## Silence and Dead Air Handling

166. ALWAYS implement a tiered re-engagement strategy for long silences — tier 1 (5 seconds of VAD silence): play a low-latency prompt (`"Are you still there?"`); tier 2 (15 seconds): play a longer re-engagement phrase and signal the LLM to ask an open-ended question; tier 3 (30 seconds): gracefully terminate the call with a goodbye message and log `call_drops_total{reason="vad_silence_timeout"}`; do not terminate at tier 1.

167. ALWAYS suppress the tier-1 silence re-engagement prompt when the LLM is actively generating — check LLM streaming state before firing; a user pausing to listen to a long LLM response appears as VAD silence and must not trigger a re-engagement interruption.

---

## Crosstalk and Multi-Speaker Handling

168. NEVER route crosstalk (two people talking simultaneously detected by VAD) identically to single-speaker barge-in — crosstalk requires a distinct handler: pause the TTS, wait 500ms for the dominant speaker to emerge, then resume STT transcription; treating crosstalk as a simple barge-in event causes the pipeline to commit to a partial, noise-contaminated transcript.

169. ALWAYS log crosstalk events as a separate `voice_crosstalk_detected_total` Prometheus counter distinct from `barge_in_total` — crosstalk rate > 5% of sessions indicates a conference room or multi-person setup that needs a different UX flow (speaker diarization, round-robin turn management).

---

## Reconnection Strategy

170. ALWAYS generate a short-lived `reconnect_token` (UUID, 60-second TTL, stored in Redis) at WebSocket connect time — on disconnect, the client presents the token to reconnect and resume the existing session state rather than starting a new session; without this, a network blip destroys the conversation context and the user must re-identify.

171. ALWAYS distinguish planned drops (session.end() called by application) from unplanned drops (WebSocket close code 1006 / ABNORMAL_CLOSURE) in reconnection logic — planned drops must invalidate the `reconnect_token` immediately; unplanned drops must keep the token alive for its full TTL to allow transparent reconnection.

172. ALWAYS resume LangGraph checkpoint state on reconnection using `thread_id` from the reconnect token — restore the full conversation history and LLM context so the user does not need to repeat prior turns; reconnection without state restoration is indistinguishable from starting a new session.

---

## Security and Fraud Prevention

173. ALWAYS check the STIR/SHAKEN `verstat` parameter on every inbound PSTN call and log its attestation level (A/B/C or `TN-Validation-Failed`) as a Prometheus label on `voice_calls_total` — treat `TN-Validation-Failed` or absent verstat as an elevated-risk signal; do not reject the call, but suppress privileged actions (account changes, payments) and route to a human agent or enforce step-up authentication before proceeding, because over 90% of Tier-1 carrier traffic now carries A-level attestation and anything lower is a credible spoofing indicator.

174. NEVER allow a voice AI agent to execute an irreversible action (payment, cancellation, account number change, password reset) without first reading back the key data to the caller and requiring an explicit spoken or DTMF confirmation — the read-back must include the specific value ("I will transfer $250 to account ending in 4471 — say 'confirm' or press 1 to proceed"), not a generic "shall I continue?", because confirmation prompts without data content do not catch transcription errors in spoken numeric input.

175. ALWAYS apply DTMF-collection mode (pause-and-resume audio redaction or silent DTMF passthrough) when collecting PAN, CVV, or expiry date over a voice channel, and configure your recording pipeline to suppress or replace those DTMF tones with silence before writing to any storage system — transmitting cardholder data over a recorded voice channel places the entire recording infrastructure in PCI-DSS scope under v4.0; DTMF isolation is the standard mechanism for descoping agents and call recording systems from CDE requirements.

176. ALWAYS integrate a passive voice liveness detector (spectral artifact analysis for neural vocoder signatures, absence of organic breath patterns) on any voice authentication flow before comparing to an enrolled voiceprint — generative voice cloning now replicates pitch, timbre, and prosody well enough to pass legacy speaker verification systems; liveness detection must run before the biometric comparison; treat a liveness score below 0.75 (or provider-recommended threshold) as an authentication failure regardless of voiceprint match score.

---

## Audio Quality and Normalization

177. ALWAYS normalize TTS audio output to -16 LUFS (integrated loudness, ITU-R BS.1770-4) for stereo delivery and -19 LUFS for mono telephone delivery before transmission — TTS engines produce inconsistent loudness across voices and providers; a provider failover mid-call produces a jarring loudness jump that callers interpret as a system fault; use `pyloudnorm` or `ffmpeg loudnorm` on each synthesized utterance before packetization.

178. ALWAYS transcode AMR-WB (16 kHz, variable bitrate) and EVS inbound mobile audio to 16 kHz linear PCM before feeding to your STT pipeline — most cloud STT providers accept PCM or Opus but not AMR-WB/EVS RTP payloads directly; failing to transcode silently degrades WER because the STT model receives malformed input it treats as noise.

---

## Scalability and Deployment

179. ALWAYS maintain a warm pool of at least two pre-initialized voice server instances per availability zone — voice WebSocket servers have a cold-start cost of 2–8 seconds when loading TTS model weights on GPU; a cold start on the first call of a burst window exceeds the entire session latency budget; configure `minReplicas=2` in HPA and accept the idle cost in exchange for eliminating cold-start tail latency.

180. ALWAYS set `terminationGracePeriodSeconds` to at least 600 seconds in Kubernetes pod specs for voice WebSocket servers, and implement a drain handler that stops accepting new WebSocket upgrades (return `503` on the readiness probe) while allowing existing sessions to complete — the default 30-second grace period is shorter than most call durations and causes in-flight calls to be forcibly disconnected on rolling deploys.

181. NEVER route voice traffic to a secondary STT provider using only error-rate as the failover trigger — also failover when rolling WER degrades more than 15 percentage points from the 24-hour baseline, because a provider can return `200 OK` with plausible but inaccurate transcripts during partial outages; instrument `voice_stt_confidence_p10` and alert when P10 confidence drops below 0.6 over a 5-minute window.

---

## Observability

182. ALWAYS emit a per-call flame graph trace with spans covering STT decode, VAD decision, LLM TTFT, TTS first-byte, and WebSocket send for every call that exceeds the P95 latency SLO — use OpenTelemetry with `gen_ai.*` span attributes for LLM stages and custom spans for audio stages; without per-call flame graphs, tail-latency investigation degrades to log correlation across disparate systems.

183. ALWAYS define four voice-specific SLO targets before go-live: (1) conversation turn latency P95 < 800ms end-to-end, (2) STT WER < 8% on domain vocabulary test set, (3) TTS first-audio-byte P99 < 400ms, and (4) call drop rate < 0.5% — generic HTTP latency SLOs do not capture voice quality dimensions; without explicit voice SLOs there is no objective criterion for triggering provider failover.

184. ALWAYS track TTS character cost per call as `voice_tts_characters_total{provider,voice_id,call_id}` and alert when per-call character count exceeds 3x the 30-day P95 for that call type — verbose-looping TTS output (a known LLM failure mode) can generate 10,000+ characters per turn on a prompt regression, multiplying TTS spend by 100x before the monthly invoice surfaces the anomaly.

---

## Compliance and Data Residency

185. ALWAYS store EU-origin voice recordings and transcripts exclusively on storage infrastructure physically located within the EEA when serving GDPR-regulated callers, and configure STT provider EU data residency endpoints — GDPR Article 44 prohibits cross-border data transfers without an adequacy decision; even ephemeral transcripts written to a US-region buffer during streaming STT constitute a transfer; document residency configuration in a DPIA.

---

## Conversation Design and Disambiguation

186. ALWAYS implement a disambiguation branch that triggers when STT confidence falls below 0.75 on an utterance controlling a branching decision — offer at most two explicit options read back to the caller rather than re-asking an open question; open re-prompts on low-confidence input cause recursive misrecognition loops where each retry degrades further due to frustration-induced speech changes.

187. NEVER allow a voice session to accept more than 3 consecutive low-confidence utterances (confidence < 0.65) before offering a transfer to a human agent or an asynchronous fallback (SMS, callback) — track `session.consecutive_low_confidence_count` in session state, reset on high-confidence utterance, and trigger escalation at threshold 3; the 3-strike threshold is the industry-converged default from production IVR tuning data.

---

## WebRTC Operations

188. ALWAYS handle ICE restart on network change by detecting `iceconnectionstate === "failed"` or `"disconnected"` and calling `restartIce()` on the RTCPeerConnection within 2 seconds — log the restart attempt as `webrtc_ice_restart_total{reason="network_change"}` and tear down the session if ICE fails to recover within 10 seconds after restart.

---

## SSML Safety and TTS Edge Cases

189. ALWAYS sanitize user-supplied text before passing it to any TTS SSML pipeline — strip or escape all `<`, `>`, `&`, `"`, and `'` characters, then validate the resulting SSML against the provider's schema before synthesis; a user who supplies `</prosody><audio src="...">` in a freeform input field can inject arbitrary audio or alter prosody tags (SSML injection attack).

190. ALWAYS normalize numbers, dates, currencies, and ordinals to spoken form before TTS synthesis using a language-aware library (`num2words` with locale) — never pass raw tokens like `$1,234.56`, `1st`, `2024-03-15` directly to the TTS engine; raw numeric tokens produce inconsistent pronunciation across providers.

---

## LangGraph Voice Patterns

191. ALWAYS execute non-blocking tool calls (DB lookups, CRM reads) in a LangGraph parallel node running concurrently with TTS playback of a bridging phrase — use `asyncio.gather` with a hard 3-second timeout; if the tool call exceeds the TTS duration plus 1 second, play a second bridging phrase and extend the timeout once before yielding a fallback response.

---

## Codec and DTMF Handling

192. ALWAYS handle codec negotiation failure — when SDP offer/answer results in no common codec, terminate the session with close code 4415, log `codec_negotiation_failure_total{offered_codecs,supported_codecs}`, and send the client a JSON error body specifying the supported codec list.

193. NEVER store DTMF digit sequences (PINs, card numbers) in voice session state beyond the immediate validation call — once digits are validated, overwrite the buffer with zeros, exclude from LangGraph state snapshots, and ensure the Redis key containing digits has TTL ≤ 60 seconds; log the event as `dtmf_digits_collected` with length only, never the digit values.

194. ALWAYS implement a DTMF inter-digit timeout of 3 seconds and a max-digits limit per collection prompt — fire collection complete on timeout expiry or on reaching max-digits; log both paths separately (`dtmf_collection_timeout` vs `dtmf_collection_complete`) to distinguish deliberate entry from abandoned collection.

---

## Accessibility

195. ALWAYS send a simultaneous SMS or WebSocket transcript channel to callers who request accessibility accommodations or are detected as hearing-impaired — emit each TTS utterance's text to the parallel channel within 500ms of audio playback start; if SMS, batch to ≤160 characters and rate-limit to one SMS per turn.

---

## Event Sourcing and Audit

196. ALWAYS store voice session events (`utterance_started`, `barge_in_detected`, `tool_call_dispatched`, `transfer_initiated`, `session_ended`) as an append-only event log in PostgreSQL with columns `(session_id, event_type, sequence_number, payload JSONB, occurred_at TIMESTAMPTZ)` — never update or delete rows; the `sequence_number` must be a monotonically increasing integer per session enforced at the application layer, not a DB sequence, to survive reconnections without gaps.

---

## Regression Testing and Audio Provenance

197. ALWAYS maintain a golden audio regression suite of at least 30 reference calls covering accent diversity, background noise, DTMF sequences, and edge-case vocabulary — run it automatically against any TTS voice change, STT model upgrade, or VAD threshold adjustment; block changes when `delta_MOS < -0.1` or `delta_WER > +2%`.

198. ALWAYS embed an inaudible psychoacoustic watermark in all AI-generated TTS audio before delivery — use a frequency-domain spread-spectrum technique that survives 8 kHz telephony resampling; the watermark payload must encode at minimum `session_id`, `utterance_sequence`, and `generation_timestamp`; never expose watermark key material in logs.

---

## SIP Trunk Health

199. ALWAYS send SIP OPTIONS keepalive messages to each SIP trunk peer on a 30-second interval and treat two consecutive non-responses as a trunk failure — remove the failed trunk from the active routing pool, emit `sip_trunk_health{trunk_id,state="down"}`, and reroute in-flight calls to a secondary trunk via warm-transfer rather than dropping them; restore only after three consecutive successful OPTIONS responses.




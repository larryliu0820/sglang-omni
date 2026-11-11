# ASR long-recording measurements

The opt-in benchmark measures native Qwen3-ASR through HTTP or its realtime WebSocket, and Parakeet through HTTP. It reports transcription quality alongside memory and elapsed time.

## Run

Build the native runtime with sglang_omni_mlx/native/scripts/build_runtime.sh, install ci/requirements.lock into a Python 3.12 environment, and provision the frozen corpus and Qwen3-ASR-0.6B-4bit using ci/provision.py. Then run this from the repository root:

```bash
python -m sglang_omni_mlx.native.ci.benchmark_long_audio \
  --runtime-bin "$NATIVE_RUNTIME_BIN" \
  --data-root "$CI_DATA_ROOT" \
  --model mlx-community/Qwen3-ASR-0.6B-4bit \
  --output "$ASR_RESULTS_DIRECTORY" \
  --mode realtime --minutes 5 10 30 60 --languages en zh --speed 1 \
  -- --cache-limit-mb 512
```

--output must be a new directory. Flags after -- go to the server and are recorded in metadata.json. Voxt starts the server with --cache-limit-mb 512, so pass that flag to measure what the app runs; without it the server's default 2 GiB limit applies. This command paces 210 minutes of audio live. A speed greater than one is an accelerated input stress measurement: refresh decodes may be coalesced, and its accuracy and finalization latency are not equivalent to live dictation.

Use --mode http separately for HTTP requests. Alone it sends one whole-file request with the server's default budget of 10 tokens per audio second. This is a maximum-length stress measurement, not what Voxt sends: a repetition loop runs until that budget, so the time, memory and error rate of such a run depend on the budget. Start with five minutes before testing longer whole-file requests. Add --chunk-seconds 30 to send whole clips in requests of at most 30 seconds, each with the budget of a Voxt Final (28 tokens per audio second plus 64, within 256 to 8192). This is close to the 20 to 45 second pieces Voxt sends during a long recording, except that the cuts fall on clip boundaries instead of detected pauses.

Each case starts a dedicated server, warms it with a short clip, sends the long recording and transcribes the warmup clip again. The post-session transcript must match warmup exactly. Disconnection is covered by test_a_disconnected_stream_stops_its_decode in ci/test_server_api.py.

The benchmark reads the dedicated server's macOS physical footprint, the kernel's record of its peak footprint and its resident memory every 250 ms. By default it kills only that server once the peak exceeds 8 GiB; change --max-footprint-gib explicitly to test another budget. If the sampler cannot read or record measurements, it stops that server rather than silently running without the guard. This is a soft guard, not a hard memory limit: a large allocation can exceed the threshold before the next read. Do not rely on it to prevent system memory pressure.

## Parakeet final transcription

Parakeet's automatic whole-recording mode (omit chunk_duration, or use 1199 or more) keeps audio up to 30 seconds whole and splits longer audio at the quietest 100 ms in the last five seconds before each 30-second limit. The ranges are consecutive, without overlap or discarded samples. Searching only before the limit bounds the encoder's quadratic attention allocation. This follows the low-energy boundary selection used by Cohere Transcribe and MOSS, but unlike their bidirectional searches it never exceeds the chunk cap. It requires no VAD checkpoint.

Explicit chunk_duration values from 2 to 300 seconds retain fixed windows with one-second overlap and token alignment/merging. These are compatibility and diagnostic settings, not the recommended final-transcription policy: short mid-speech windows can decode to blanks, and long explicit windows consume substantially more memory. The HTTP stream field selects SSE transport, not the internal chunking algorithm or incremental recognition.

The Parakeet process limits MLX's freed-buffer cache to 256 MiB for both server and CLI. This is not a total memory limit: weights, active encoder arrays, the uploaded audio and queued requests also consume memory. The CLI clears its cache between files, like Qwen3-ASR, and reports peak_active_bytes, active_bytes and cache_bytes. These MLX counters exclude some process allocations; use the benchmark's kernel physical footprint to assess a smaller Mac. Avoid unbounded request concurrency, and keep headroom for macOS and other applications. A measured footprint on a larger Mac is not a guarantee against OOM on every smaller Mac.

Provision mlx-community/parakeet-tdt-0.6b-v3 with ci/provision_models.py, then run the same guarded benchmark (Parakeet v3 has no Chinese or realtime endpoint):

```bash
python -m sglang_omni_mlx.native.ci.benchmark_long_audio \
  --runtime-bin "$NATIVE_RUNTIME_BIN" \
  --data-root "$CI_DATA_ROOT" \
  --model-kind parakeet --model mlx-community/parakeet-tdt-0.6b-v3 \
  --output "$PARAKEET_RESULTS_DIRECTORY" \
  --mode http --languages en --minutes 1 5 15 30 --max-footprint-gib 3
```

This deliberately sends each whole recording in one request, exercising the server's internal 30-second splitting. Do not add --chunk-seconds when measuring that path. The guard records startup, transcription and recovery and fails if the sampled kernel peak exceeds the supplied budget; it does not impose a hard allocator limit. Compare the same seed and frozen corpus before and after a policy change.

The original parakeet-tdt-0.6b-v3.json golden explicitly requests five-second overlapping windows to retain its recorded Swift parity outputs. The separate parakeet-tdt-0.6b-v3-final.json golden requests automatic mode and records the changed final policy. Preserve that distinction rather than replacing the old outputs or weakening their tolerance. API regressions are in ci/test_model_servers.py; checkpoint-free boundary and coverage tests are in native/tests/audio_chunk_test.cpp and run in Mac CI.

### Validated configuration

Measured on Apple M4 Max with 64 GiB unified memory, macOS 27.0.1 and the pinned MLX/MLX Metal 0.32.3 runtime. The checkpoint is the revision and file hashes in ci/model_pins.json. English WER on the frozen 392-clip corpus changed from 23.844% with the original five-second policy to 2.601% with automatic pause-aligned chunks. The 64 short English clips were unchanged; medium clips improved from 20.214% to 2.009% and long clips from 31.851% to 2.865%. Explicit five-second decoding matched the original binary on all 392 outputs on this chip. Chinese and mixed-language metrics remain poor because this checkpoint does not support Chinese.

The command above, with the default seed 2888, gave the following matched-input comparison. Peak footprint includes startup and warmup, is the kernel process footprint (not MLX active memory or RSS), and uses GiB (2^30 bytes).

| Requested recording | Original five-second WER | Automatic final WER | Automatic peak footprint |
| --- | ---: | ---: | ---: |
| 1 minute | 16.23% | 2.09% | 2.10 GiB |
| 5 minutes | 23.23% | 2.86% | 2.18 GiB |
| 15 minutes | 40.75% | 7.66% | 2.33 GiB |
| 30 minutes | 38.97% | 5.43% | 2.64 GiB |

All four automatic cases passed the 3 GiB sampled guard, exited normally and reproduced the warmup transcript after the long request. A separate persistent-server run also completed 73 queued clip requests and recovered after a disconnected request under the same guard. The CLI's 73-file invocation kept its measured freed-buffer cache below 256 MiB. The source recording length changes the clip composition and cut positions, so WER is not monotonic in duration. These figures do not measure natural meetings, every supported language or worst-case acoustic content.

No physical 8 GiB Mac or system-wide low-memory-pressure test was performed. The 30-second encoder cap and cache bound avoid the prior unbounded CLI buffer retention, but cannot guarantee that an arbitrary upload, explicit 300-second window, concurrent queue or other applications will fit a smaller Mac. Use automatic mode, bounded client concurrency and the guarded benchmark on the target machine rather than inferring an OOM guarantee from this larger host.

## Input and scoring

The recording concatenates complete short and medium clips of one language from the frozen corpus. A seeded permutation uses every clip before reusing the pool. Longer recordings necessarily reuse clips; the manifest records their order and the recording's hash. These are synthetic concatenations, not natural hour-long meetings.

HTTP requests explicitly use the voxt_swift audio layout and both stopping flags. Realtime sessions use the server's own defaults for layout and stopping rules, which can differ between revisions. Do not treat HTTP versus realtime scores as a controlled same-layout comparison. Voxt shows realtime text only as a preview and inserts the text of its HTTP requests, so realtime error rates are not what the app inserts. This tool does not calculate an isolated-short-clip baseline or directly measure natural continuous meetings; differences across durations also include changing clip composition and boundary effects.

The final clip is kept whole, so actual duration is at least the requested duration and may exceed it by one clip. Every reference word remains aligned with audio actually sent. The same seed gives nested recording prefixes across durations. English WER and Chinese CER count the tokens of ci/transcript_tokens.py, which the golden checker also uses, so Chinese keeps embedded Latin words as units. Both missing tails and repeated text count as errors.

## Interpret results

- metadata.json records hardware, OS, source revision, working-tree state, executable hash, checkpoint file hashes, benchmark dependency versions, HTTP request fields, realtime defaults and the memory guard. The MLX version loaded by the separate native server is not discovered automatically; record it independently.
- Each input manifest records the exact source sequence and expected transcript. Each result stores the hypothesis, error numerator and denominator, timing, memory, the recovery observation and server_exit_code. An exit before shutdown, a nonzero exit during shutdown or a shutdown timeout marks the case as failed. Realtime runs also save timestamped protocol events.
- The memory trace spans startup, warmup, the long session and recovery. Compare equivalent phases; an encoder allocation or stored transcript is not by itself a leak. The warm baseline, post-session and post-recovery values separate those phases. peak_footprint_bytes is the kernel's exact peak at the end of the session. It only grows, so it also covers startup and warmup, and it equals warmup_peak_footprint_bytes when the session set no new peak.
- HTTP results include real_time_factor, the request wall time divided by audio duration: a 6-second request for 60 seconds of audio has a factor of 0.1. In realtime mode, wall time includes the time spent sending audio at the selected speed, while the server may process audio concurrently. Finalization time is the wait from sending the last audio frame to receiving the completed transcript; it can include processing audio that accumulated earlier.
- A measured status means the protocol, scoring and recovery completed. Review the error rate, generated-token finish reason and memory trace.

Compare before/after builds with identical weights, MLX version, input hashes, pacing and hardware. Keep existing API tests and corpus goldens as correctness gates; this benchmark is an opt-in measurement, not a timing-sensitive unit-test gate. Its own tests are in native/tests/test_long_audio_benchmark.py and run against the stub server.

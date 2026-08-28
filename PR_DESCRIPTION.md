**Title:** Fix short voice commands ("stop") often not being detected by HA's local VAD

**Summary:**
Short commands (e.g. a single word like "stop") spoken immediately after wake frequently weren't detected by Home Assistant's local VAD (`stt-vad-start` never fired), causing the interaction to run the full 15-second pipeline timeout before HA's STT engine eventually recovered the transcript on its own via its own independent end-pointing. This PR fixes it by priming the audio stream before real audio starts and running the same auto-gain/noise-suppression HA core already supports for this purpose, but never wires up for satellite pipelines.

**Root cause (two separate issues, both fixed here):**

1. **VAD warm-up collision.** HA's local VAD (`assist_pipeline.vad.VoiceCommandSegmenter`) scores speech via `pymicro_vad`, which has a hardcoded warm-up requirement (`INPUT_FEATURES = 74` — 74 consecutive 10ms chunks, i.e. 740ms) before it can output a valid probability at all; every chunk before that returns an internal "not enough context" sentinel treated as silence. A short command spoken immediately after wake is often entirely over before that window elapses, so the VAD model never gets a real chance to score it.

   Confirmed directly: pulled real `debug_recording_dir` WAV captures where "stop" produced no `stt-vad-start` and ran the full 15s timeout, then replayed the actual `pymicro_vad` model locally against them. In each case the word was finished (~0.5-0.6s in) before the model's first valid output (~0.76s in).

2. **Marginal VAD confidence for short/quiet utterances.** Even once past the warm-up, `VoiceCommandSegmenter` requires ~300ms of cumulative speech-probability >0.2 before registering onset. "Stop" is short and consonant-heavy, and how confidently the model scores a given utterance varies noticeably — several real recordings never crossed the 0.2 threshold at all, or crossed it too briefly. HA core already has a documented fix for exactly this: the STT engine's `audio_processing` declares `prefers_auto_gain_enabled`/`prefers_noise_reduction_enabled`, intended to trigger `pyspeex_noise`-based AGC/noise suppression before VAD scoring (see `assist_pipeline/audio_enhancer.py`). But `assist_satellite/entity.py`'s `async_accept_pipeline_from_satellite()` hardcodes `AudioSettings(silence_seconds=...)` and never forwards those preferences for satellite-driven pipelines — so this boost never happens for `voice_satellite`-based satellites regardless of what the STT engine asks for.

**Fix:**
Both issues are addressed in `audio_stream()` inside `async_run_pipeline` (only when `start_stage == "stt"`, so the wake-word server-side path is untouched):
- Prime the stream with ~800ms of digital silence before the first real chunk, so `pymicro_vad`'s warm-up completes before real audio starts.
- Run the same `pyspeex_noise.AudioProcessor` (auto-gain + noise suppression) HA core already supports, applied directly here since we're upstream of the gap in `assist_satellite/entity.py`.
- Re-chunk incoming audio into exact 320-byte (10ms) frames before processing — `Process10ms()` requires an exact frame, but incoming websocket binary messages aren't guaranteed to be that size (the client may batch multiple 10ms frames per network message). HA core's own pipeline re-chunks for exactly this reason (`assist_pipeline.vad.chunk_samples`, used in `process_enhance_audio`); missing this step here produced audibly choppy/garbled audio in testing.

**Testing performed:**
- Verified the warm-up fix by replaying the real `pymicro_vad` model against captured recordings with and without injected silence — recordings that never fired `stt-vad-start` fired cleanly once primed.
- Verified the AGC/noise-suppression fix the same way: ran it against 5 real previously-failing recordings offline, 3 flipped from fail to pass, 0 regressions on already-passing recordings.
- Deployed live and ran 5 consecutive real voice interactions ("Alexa" + immediate "stop", no pause, no restarts between attempts): 5/5 passed, consistent ~700ms detection windows (`stt-vad-start` ~1.3-1.6s, `stt-vad-end` ~0.7s later), correct transcription every time.
- Caught and fixed two library binding mismatches along the way (`pymicro_vad`/`pyspeex_noise`'s actual installed API differs from what's shown in some published docs — this PR's code matches the versions HA core itself pins: `pymicro-vad==1.0.1`, `pyspeex-noise==1.0.2`).

**Changes:**
- `manifest.json`: add `pyspeex-noise` to `requirements` (already an implicit dependency via HA core's `assist_pipeline`, but should be declared explicitly here since we import it directly).
- `assist_satellite.py`: add `from pyspeex_noise import AudioProcessor` import; rewrite `audio_stream()` as described above.

**Caveats / open questions for discussion:**
- `AudioProcessor(4000, -30)` (auto-gain, noise-suppression) and the 800ms silence duration are hardcoded to values that worked well in testing on one device/environment — might be worth exposing as config options rather than constants, especially since HA core's own `AudioSettings` already has a 0-31 / 0-4 scale for these that could be mirrored here.
- The silence-priming duration (800ms) has some slack above the actual 740ms model requirement; could be tuned down further if useful.
- Happy to adjust scope/approach based on maintainer preference — this was arrived at empirically against real hardware, not from reading `pymicro_vad`/`pyspeex_noise` docs alone (their actual installed APIs differ from what's published in a couple of places, noted above).

# MindWeave Safety Monitor — Technical Spec (MVP)

## Overview
A 10 Hz loop evaluates HR/HRV and EEG quality to trigger either (a) scene softening + facilitator alert (Caution) or (b) system Auto‑Pause (Hard Pause). EEG is **not** used for hard safety in MVP.

## Inputs
- HR (bpm) at 1 Hz; RR intervals (ms) via Polar H10 (BLE).
- RMSSD computed over rolling 30 s window; updated each second.
- EEG bandpower quality flags (optional), signal quality 0–1.

## Baselines
- Pre‑session 5‑min seated baseline for HR_mean and RMSSD_median.
- Per‑user Z‑scores maintained with 60 s EMA in-session.

## Thresholds (MVP defaults)
Caution if any holds true:
- RMSSD_drop = (baseline_RMSSD - RMSSD_now) / baseline_RMSSD > 0.30 for ≥20 s
- HR_now - baseline_HR > 25 bpm for ≥15 s

Hard Pause if any holds true:
- RMSSD_drop > 0.50 for ≥10 s
- HR_now ≥ 120 bpm for ≥20 s
- panic_gesture == true OR safeword_detected == true

## Pseudocode
loop @ 10Hz:
  update_metrics()
  if panic_gesture or safeword:
      trigger_hard_pause(reason="user_signal")
  elif (rmssd_drop > 0.50 and duration >= 10s) or (hr >= 120 and duration >= 20s):
      trigger_hard_pause(reason="biometric")
  elif (rmssd_drop > 0.30 and duration >= 20s) or (hr - baseline_hr > 25 and duration >= 15s):
      trigger_caution(reason="biometric")
  else:
      clear_alerts()

function trigger_caution(reason):
  set_scene_modulation(soften=True)
  notify_facilitator(level="caution", reason=reason)
  log_event(type="safety_caution", reason=reason, ts=now())

function trigger_hard_pause(reason):
  freeze_scene()
  fade_to_neutral_grey()
  play_voiceover("safety_exit_v1")
  open_facilitator_intercom()
  require_manual_resume = True
  log_event(type="safety_hard_pause", reason=reason, ts=now())

## Signal Quality Degradation
- If RR stream missing >5 s → mark HRV invalid; do not compute RMSSD; show red badge.
- If EEG quality <0.3 for >10 s → disable EEG‑driven modulation; continue session; alert facilitator.

## Event Logging (JSONL)
{"ts": 1712345678.123, "type": "safety_caution", "reason": "rmssd_drop_30", "rmssd": 18.4, "hr": 102}
{"ts": 1712345701.456, "type": "safety_hard_pause", "reason": "user_signal", "rmssd": 16.1, "hr": 88}

## Resume Conditions
- Facilitator presses RESUME on dashboard after client is oriented and confirms readiness.
- On resume: ramp visuals/audio over 10 s; do not immediately re‑enter deepeners.

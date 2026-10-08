# AWS EC2 Incident Postmortem — CPU Spike (Simulated)

**Author:** Piyush Awasthi
**Date:** 7 October 2026
**Environment:** AWS Free Tier — EC2 (Amazon Linux, 2 vCPU), Apache (httpd), CloudWatch
**Severity:** SEV-3 — single instance, lab environment, no customer impact
**Status:** Resolved

---

## 1. Summary

A CPU exhaustion event was simulated on a single EC2 instance (`web01`) using the `stress` utility to validate CloudWatch's CPU monitoring and alarm configuration. Early short runs pushed CPU as high as ~100%, but the `web01-high-cpu` alarm stayed in OK state, because its rule requires three consecutive breaching datapoints over 15 minutes and the short bursts never satisfied it. A sustained 20-minute run (`stress --cpu 2 --timeout 1200`) then triggered the alarm, confirming the alarm was configured correctly and simply needed a load pattern matching its evaluation window.

---

## 2. Timeline

| Time (UTC) | Event |
|---|---|
| ~04:00–06:30 | Baseline — instance idle, CPUUtilization around 0.33% |
| ~06:35–07:00 | Short stress runs (180 s and 400 s) produce intermittent spikes (peaks of ~60% and ~100%) with drops back toward baseline in between. Never three consecutive datapoints above 70%, so the alarm stays **OK** |
| ~07:03 | Sustained run started: `stress --cpu 2 --timeout 1200` (start time approximate, inferred from the CloudWatch graph) |
| 07:05–07:15 | Three consecutive 5-minute datapoints at ~100% (07:15 datapoint: 99.996%) |
| ~07:17 | `web01-high-cpu` alarm is **In alarm** (screenshot taken 12:47 IST) |
| ~07:23 | The 1200-second timeout expires and the `stress` processes exit on their own; CPU falls back to baseline (approximate) |
| 07:25 | `ps aux` shows no `stress` process still running |
| By 07:27 | Alarm back to **OK** (screenshot taken 12:57 IST) |
| 07:29–07:30 | A further 180-second run (PIDs 37677–37679) is started and inspected. `top` at 07:30:08 shows both workers near 100% CPU |

*IST times from the workstation clock were converted to UTC (IST = UTC+5:30) to match CloudWatch and the server.*

---

## 3. Detection

- **What alerted you?** The `web01-high-cpu` alarm changing to "In alarm" in CloudWatch, with the CPUUtilization graph confirming the sustained breach.
- **What metric/threshold fired?** `CPUUtilization > 70 for 3 datapoints within 15 minutes`.
- **Time to detect:** About 14 minutes from the start of the sustained load (~07:03) to the alarm (~07:17). This delay is built into the configuration: with default 5-minute metrics, three consecutive breaching datapoints take roughly 15 minutes to accumulate.

![web01-high-cpu alarm in In alarm state](screenshots/cpu-1-alarm-in-alarm.png)

![CPUUtilization datapoint at 99.996 percent](screenshots/cpu-2-metric-peak.png)

---

## 4. Investigation

- **Tools/commands used:**
  - CloudWatch alarm and metric graphs (3-hour view) — confirmed the metric was spiking and when the alarm changed state
  - `ps aux | grep stress` — confirmed the `stress` parent process and its worker processes, and later that none were left running
  - `top` — confirmed real-time CPU usage per process

![top output showing two stress workers near 100 percent CPU](screenshots/cpu-4-top-output.png)

- **What `top` showed (07:30:08):** two `stress` workers at 99.7% and 99.3% CPU, and overall `%Cpu(s): 100.0 us` with 0.0 idle. That is a fully saturated 2-vCPU instance, so the test genuinely exhausted the CPU. Memory was not under pressure (306 MiB free, no swap used), so the load was purely CPU-bound.

- **What was ruled out:** During the early short runs, the alarm staying OK was **not** caused by the stress test failing to load the CPU. The graph shows genuine spikes up to ~100%, and the alarm's state timeline stays green throughout them. It also was not a security group or instance-level problem.

- **Root cause identified:** The alarm's evaluation period (`3 datapoints within 15 minutes`, meaning CPU must stay above 70% across three consecutive 5-minute samples) did not match the early short, intermittent runs. The load was real but not sustained long enough in an unbroken window to satisfy the alarm's M-of-N condition. This is a common alarm-tuning pitfall: a metric can clearly breach a threshold without the alarm ever firing, if the evaluation window doesn't match the failure pattern being monitored.

---

## 5. Resolution

- **Action taken:** Ran the load as a longer, continuous test (`stress --cpu 2 --timeout 1200`) so it spanned multiple full 5-minute evaluation windows without interruption. No changes were made to the alarm's threshold or datapoints-to-alarm setting, and the existing 3-of-3/15-minute rule fired correctly once the load matched it.
- **Recovery:** The load ended when the 1200-second timeout expired (~07:23), with no manual intervention needed. CPU returned to baseline and the alarm returned to OK by 07:27.
- **Verification:** Confirmed in CloudWatch that the alarm went OK → In alarm → OK, and via `ps aux` that no `stress` process remained at 07:25.
- **Time to resolve:** Approximately 10 minutes from the alarm firing (~07:17) to it clearing (by 07:27).

![web01-high-cpu alarm back in OK state](screenshots/cpu-3-alarm-recovered.png)

---

## 6. Impact

- **Who/what was affected:** Single lab EC2 instance only; no external users or production systems involved.
- **Duration of degraded state:** About 20 minutes of sustained CPU saturation (~07:03–07:23), plus the earlier short test runs. All of it was intentional.

---

## 7. Prevention / Follow-up Actions

- [ ] Match the "datapoints to alarm" evaluation window to the failure pattern being monitored — a brief spike-and-recover pattern needs a shorter or lower-datapoint-count alarm than a sustained-load scenario.
- [ ] Add a secondary, faster alarm (e.g., 1 datapoint over 1 minute) alongside the sustained-load alarm to catch brief spikes. Production environments often layer alarms at different sensitivities for the same metric.
- [ ] Reduce detection latency: the current rule took ~14 minutes to fire. Enabling detailed (1-minute) monitoring would let the same 3-datapoint rule fire in about 3 minutes, at a small additional cost.
- [ ] Remember that "metric breached threshold" and "alarm fired" are not the same thing — conflating them is an easy mistake under incident pressure.
- [ ] Confirm the SNS email subscription is active before each test run, so alerts arrive in real time rather than requiring manual console checks.

---

## 8. Lessons Learned

This exercise didn't go as initially planned — the early runs clearly spiked CPU, but the alarm never fired, which taught the more valuable lesson. CloudWatch alarms don't trigger on a single breach; they require a sustained pattern matching the configured evaluation period. The sustained run confirmed the alarm configuration was correct all along — it just needed a matching load pattern. It also showed that a 3-of-3 rule on 5-minute metrics means roughly 15 minutes of detection delay, a trade-off between fewer false alarms and faster notification. For real support work, alarm tuning is not a one-time setup step: a "failed" test is sometimes a mismatch between the scenario and the alarm's intended failure mode, which is itself useful diagnostic information.

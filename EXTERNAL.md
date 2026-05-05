# External Context: Users, Community, and Wants

## Who Uses This

**Core audience**: hobbyists and researchers running [trunk-recorder](https://github.com/robotastic/trunk-recorder) to monitor P25 trunked radio systems — mostly law enforcement, fire, EMS, public works, transit, and utilities. They want to do more with their data than a raw call log lets them.

**Tiers of engagement:**

| Type | What they want |
|------|---------------|
| Casual monitors | Simple call log, inline audio playback, talkgroup lookup |
| Power users | Unit tracking, affiliation analysis, cross-system correlation, transcription |
| Researchers / analysts | Network graphs, OTA alias capture, fleet identification, pattern analysis |
| ML / ASR community | IMBE-ASR models on HuggingFace; researchers interested in codec-domain ASR |

## Community Hub

The primary community is the **trunk-recorder Discord server**. The plugins-lobby channel is where plugin users (including tr-plugin-dvcf users) discuss issues, share configurations, and request features. Key community members regularly visible there: `taclane`, `tadscottsmith`, `natecarlson`, `.gofaster`, `simon0722`. The trunk-recorder maintainer (`robotastic_dc`) is also active.

tr-bot is our interface to this community: it answers support questions, analyzes debug reports, and opens GitHub issues on users' behalf.

## What Users Actually Want

### From direct Discord observation:

**Network / affiliation analysis** — Users are hungry for unit-to-talkgroup relationship mapping, even for units that never make voice calls (affiliation-only activity). The unit network graph in tr-engine/tr-dashboard directly addresses this. Specific asks:
- Track which units affiliated to which talkgroups over time
- Identify radios operating outside their home site (mobility inference)
- Surface denied affiliations as a distinct signal
- Color-coded idle/active/deregistered states

**OTA alias capture** — P25 systems broadcast radio aliases over-the-air. Users want these auto-populated as unit names ("350 OTA aliases in 24 hours" is not unusual for a busy system). tr-engine captures these via MQTT unit events.

**Multi-site without complexity** — Users run 4+ trunk-recorder instances monitoring the same P25 system from different locations. They want one unified view. tr-engine's auto-merge of P25 systems by `(sysid, wacn)` is a direct response to this pain.

**Transcription** — Multiple users have built Whisper/ASR pipelines (some using ICAD Transcribe, some Whisper, some Qwen3). The ask is: transcription that just works, without audio reconstruction artifacts. IMBE-ASR is the long-term answer.

**Hot-reload of talkgroup lists** — Users update RadioReference CSVs frequently as agencies add/rename talkgroups. Having to restart trunk-recorder to pick up changes is a pain. tr-engine's CSV writeback + in-memory updates partially address this.

**RadioReference integration** — Users manually track changes on radioreference.com and update CSVs. There's no clean programmatic way to get change notifications. Periodically scraping or subscribing to RR is a recurring community request.

**Multi-upload simultaneously** — Users want to send calls to rdio-scanner, OpenMHz, and Broadcastify at the same time. tr-engine's HTTP upload endpoint is compatible with rdio-scanner and OpenMHz upload formats.

### Deeper wants (not always explicitly stated):

- **Investigative tools** — Correlate an unknown unit ID with known IDs by looking at shared talkgroup usage patterns. The unit network graph and cross-reference features in tr-dashboard exist because of this.
- **Encryption detection** — Knowing which talkgroups are encrypted vs. plaintext, and being notified when previously plaintext TGs go encrypted, is a safety/awareness concern.
- **Live audio without a separate scanner** — The IRC Radio Live and Scanner pages, plus live audio streaming via WebSocket, address wanting browser-based monitoring without dedicated hardware.
- **Mobile-friendly** — Many users monitor from phones. The Scanner page is explicitly mobile-optimized.

## Existing Tools / Ecosystem

Understanding where tr-engine fits vs. alternatives:

| Tool | Role | Relationship |
|------|------|-------------|
| **rdio-scanner** | Self-hosted call archive + player, older-style UI | tr-engine accepts its upload format; some users migrate from it |
| **OpenMHz** | Cloud-hosted call sharing | tr-engine accepts its upload format |
| **Broadcastify Calls** | Cloud call archiving/streaming | Many users upload to it alongside tr-engine |
| **Trunk Recorder** | The capture layer (external dependency) | tr-engine is downstream of it; we don't fork TR, we plugin to it |
| **ICAD Transcribe** | Another transcription solution some users run | Competes with tr-engine's transcription pipeline |
| **Trunking Recorder** | Another TR-compatible backend | Alternative to tr-engine |
| **Unitrunker** | Older Windows-based trunking analyzer | Some users are migrating off it |

Users frequently run multiple tools simultaneously (e.g., uploading to both tr-engine and rdio-scanner from the same TR instance). Compatibility with the upload formats of existing tools is a retention factor.

## What "Good" Looks Like to Users

From observed feedback, users most consistently praise:
- **Setup simplicity** — `curl | sh` installs and zero-config system discovery reduce the barrier to entry
- **Real data, real fast** — seeing actual calls, unit names, and affiliations appear within minutes of connecting TR
- **The network graph** — frequently cited as a differentiator; users use it to identify unknown units and agency structures
- **Transcription quality** — particularly for fire/EMS where understanding the dispatch matters

The most common friction points:
- Auth/token confusion on first setup
- Multi-site P25 systems producing duplicate system entries before auto-merge kicks in
- Talkgroup CSVs being out of date (RadioReference lag)
- Understanding what `system_id` vs. `sysid` vs. `short_name` means

## ML / Research Community (HuggingFace)

The IMBE-ASR models on HuggingFace (`trunk-reporter/imbe-asr-*`) attract a different audience: researchers interested in codec-domain speech recognition, people working on P25 audio quality, and developers building public-safety transcription pipelines. This community interacts via HuggingFace model cards and GitHub issues on the imbe_asr repo, not through the Discord.

Key things this community wants:
- ONNX exports (already provided: fp32, int8, uint8)
- Clear WER benchmarks with reproducible eval scripts
- Integration with standard inference frameworks
- A drop-in Whisper-compatible server (qwen3-asr-server)

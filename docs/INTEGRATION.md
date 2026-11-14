# SCADA integration: MQTT and Modbus TCP

Both outputs are off by default. Turn them on with **MQTT / Modbus…** in the app. Settings are
saved in `%APPDATA%\KG5ZZB-SCADA-RF-Link-Monitor\settings.json` (the MQTT password is encrypted
for the Windows user with DPAPI). Both are covered by automated tests: the MQTT test publishes to
a real in-process MQTTnet broker, and the Modbus test uses an independent client library
(FluentModbus) against the app's server.

## MQTT

Publishing runs from a background queue (2000 messages, oldest dropped when full), so a slow or
missing broker never blocks the radio. Every network step has a 3 s timeout; the client
reconnects with back-off (1 s doubling to 30 s). Payloads are UTF-8 JSON.

| Topic | Retained | QoS | When | Content |
|---|---|---|---|---|
| `{prefix}/{site}/status` | yes | 1 | connect, clean shutdown, and as the **last will** | `{"state":"online"}` or `"offline"` |
| `{prefix}/{site}/summary` | yes | 0 | every check (2 s) | overall state, headline, active alarm count, unexpected carriers |
| `{prefix}/{site}/channel/{channelId}/state` | yes | 1 | every check | severity, alarm bits and names, one-line summary |
| `{prefix}/{site}/channel/{channelId}/metrics` | no | 0 | every check | live measurements (below) |
| `{prefix}/{site}/alarms` | no | 1 | each alarm raised or cleared | the alarm event |

Defaults: prefix `kg5zzb/scada-rf`, site `site1`. `/`, `+`, `#` and spaces in the site or channel
ID are replaced with `_`. Channel IDs are `ch1`, `ch2`, ... in the order of the channel list.

**metrics** example:
```json
{"timeUtc":"2026-05-01T12:00:00Z","channelId":"ch4","frequencyMhz":952.5,"ready":true,
 "burstsPerMinute":212.4,"dutyCycle":0.231,"rssiDbfs":-31.2,"snrMeanDb":21.7,"snrWeakestDb":13.9,
 "noiseFloorDbfs":-49.8,"noiseRiseDb":0.2,"unansweredPollFraction":0.287,"secondsSinceLastBurst":0.4,
 "anomalyScore":1.9134,"anomalyLimit":0.6012}
```
Until a channel has 20 s of data, `ready` is `false` and the measurements are `null`.

**alarms** example:
```json
{"id":4,"timeUtc":"2026-05-01T12:01:22Z","channelId":"ch4","channelName":"MAS pipeline remotes",
 "kind":"raised","code":"unansweredPolls","severity":"alarm","title":"About 29% of polls get no reply",
 "explanation":"The master keeps asking but some RTUs do not answer. ..."}
```
`kind` is `raised` or `cleared`; `severity` is `ok`, `watch`, `alarm` or `noData`. The same line
format is appended to `%APPDATA%\KG5ZZB-SCADA-RF-Link-Monitor\alarms.jsonl`.

The topic layout is plain JSON over MQTT. It is **not** Sparkplug B (which needs Protobuf
payloads and birth/death certificates); use an MQTT Engine "custom namespace" or Node-RED to map
it into tags.

## Modbus TCP

The app is a Modbus TCP **server** (slave). Port 502 by default (configurable); any unit ID is
accepted. Function codes: **03** Read Holding Registers, **04** Read Input Registers, **06** Write
Single Register, **16** Write Multiple Registers (max 125 registers per read, 123 per write).
Other function codes return exception 01, addresses outside the map exception 02, bad
quantities or values exception 03. Up to 16 clients at once; idle connections close after 120 s.

Addresses below are **0-based protocol addresses** (Ignition's Modbus driver with "zero-based
addressing" off, or most SCADA tools, show them +1: input register 0 = `3x00001`).
The input registers (FC04) and holding registers (FC03) hold the **same values**; use whichever
your driver prefers. Signed values are 16-bit two's complement (`short`).

### Header (registers 0-19)
| Addr | Meaning | Units / values |
|---|---|---|
| 0 | Heartbeat: +1 every check (2 s) | counter, wraps at 65535 |
| 1 | Overall status | 0 OK, 1 WATCH, 2 ALARM, 3 no data yet |
| 2 | Number of channels | 0-32 |
| 3 | Active alarms | count |
| 4 | Unexpected carriers right now | count |
| 5 | Anomaly model loaded | 1 yes, 0 no (rules only) |
| 6 | New alarm since last acknowledge | 1 / 0 (latched) |
| 7 | Learn this site: seconds left | 0 when not learning |
| 8 | Register map version | 100 = 1.00 |
| 10 | **Command (write):** acknowledge alarms | write 1: clears register 6 and the app's "new alarms" flag |
| 11 | **Command (write):** start Learn this site | write 1: learns for 120 s |

Only registers 10 and 11 are writable (values 0 or 1); writing anywhere else returns exception 02.

### Channel blocks
Channel *i* (0-based, in the channel list order) starts at **100 + 20·i** (up to 32 channels).

| Offset | Meaning | Units / scaling |
|---|---|---|
| +0 | Status | 0 OK, 1 WATCH, 2 ALARM, 3 no data yet |
| +1 | Active alarm bits | 1 silent, 2 interference, 4 continuous signal, 8 unanswered polls, 16 weak signal, 32 anomaly model, 64 traffic change |
| +2 | Frequency, high word | kHz as uint32 = (+2 << 16) + (+3) |
| +3 | Frequency, low word | |
| +4 | Transmissions per minute | x10 |
| +5 | Duty cycle (busy) | per mille (0-1000) |
| +6 | Mean RSSI | dBFS x10, signed |
| +7 | Mean SNR | dB x10, signed |
| +8 | Noise floor | dBFS x10, signed |
| +9 | Noise floor rise against usual | dB x10, signed |
| +10 | Anomaly score / limit | x1000 (1000 = at the limit) |
| +11 | Polls without reply | per mille |
| +12 | Seconds since last transmission | s |
| +13 | Transmissions in the 20 s window | count |
| +14 | Weakest-10% SNR | dB x10, signed |
| +15 | Traffic change log2(now / usual) | x1000, signed |
| +16-19 | reserved | 0 |

Example: in a SCADA package (Ignition or any other), add a Modbus TCP device pointing at the PC
running the app, then map tags to input registers. With one-based addressing, the overall status
(protocol address 1) is input register 2 and the first channel's status (protocol address 100)
is input register 101.

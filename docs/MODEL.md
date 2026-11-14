# Link-health anomaly model: model card


## What it does
Every 2 seconds the app describes each channel's last 20 seconds with 12 numbers (below).
A small autoencoder squeezes those 12 numbers through a 4-number bottleneck and rebuilds them.
It learned only from healthy polling traffic, so traffic that it cannot rebuild well is
unusual. The rebuild error is the **anomaly score**; above the limit the channel goes to WATCH,
above 3x the limit to ALARM. The features with the largest error become the plain-language
explanation ("whether polls get answers is lower than normal"). It runs on the CPU inside the
app (ONNX Runtime) in microseconds and needs no internet.

The app also runs plain threshold rules (silent channel, noise rise, busy channel, unanswered
polls, weak signal, traffic change). The model adds the patterns that fixed limits miss.

## Data
- **Source:** a C# dataset generator models polled telemetry channels: a master polling 2-10
  RTUs in turn (poll 15-55 ms, replies 25-145 ms, turnaround 8-55 ms, retries 0-2, normal reply
  loss 0-3%, RTU links 8-32 dB SNR with slow fading), 12.5 or 25 kHz channels. The channel is
  listened to the way the app does it (0.4 s dwells, sharing the dongle with 0-4 other tuning
  positions). Each window is measured by the app's own `ChannelTracker` from per-frame channel
  power; a test checks this frame-level model against the full I/Q + FFT path.
- **Training set:** seed 1, 16000 healthy windows (13600 train / 2400 validation).
  The model never sees a fault while training.
- **Test set:** seed 2, 11000 windows: 4000 healthy and
  7000 with one of seven faults, never used for training or for the limit.

## Features (per channel, per 20 s window)
1. log10(1 + bursts/s)
2. duty cycle
3. log10(1 + mean burst ms) / 3
4. mean SNR / 30 dB
5. weakest-10% SNR / 30 dB
6. noise floor rise / 10 dB
7. band floor rise / 10 dB
8. paired (answered) fraction
9. gap variation (CV) / 3
10. traffic change log2 / 4
11. weakest SNR change / 10 dB
12. paired fraction change

## Model
`12 -> 24 -> 4 -> 24 -> 12` autoencoder, tanh, 832 parameters. Standardisation (with a
minimum scale of 0.8 dB for the two noise-rise features and 1 dB for the SNR change) is part of the exported graph.
Mean squared reconstruction error, Adam (lr 2e-3), batch 256, early stopping on validation loss.
**Anomaly limit:** 99th percentile of the validation score = 0.7780.
"Learn this site" can raise a channel's limit to 1.25x the 99th percentile of its own scores.

## Results on the held-out test set
| Detector | Precision | Recall | F1 | False-alarm rate on healthy windows |
|---|---|---|---|---|
| **Autoencoder** | **0.995** | **0.692** | **0.817** | **0.006** |
| Threshold rules (baseline) | 0.999 | 0.498 | 0.665 | 0.001 |
| Either one flags | 0.995 | 0.693 | 0.817 | 0.006 |

Autoencoder ROC AUC: **0.919**.

The threshold baseline flags a window when the channel is silent, the channel or band noise is
up 6 dB or more, the traffic halves or doubles against the usual level, or the channel is busy
80% of the time or more: the fixed limits a SCADA engineer would set by hand.

### Detection rate per condition
| Condition | Test windows | Autoencoder | Threshold rules | Both |
|---|---|---|---|---|
| Healthy traffic | 4000 | false alarms 0.6% | 0.1% | 0.6% |
| Broadband interference (band noise +3 to +15 dB) | 1000 | 100.0% | 75.6% | 100.0% |
| In-channel noise interference | 1000 | 97.4% | 63.7% | 97.4% |
| Steady carrier on the channel | 1000 | 100.0% | 100.0% | 100.0% |
| One RTU stops answering | 1000 | 29.6% | 1.6% | 29.7% |
| Whole channel silent (master down) | 1000 | 100.0% | 100.0% | 100.0% |
| One or two RTUs fade 8-18 dB | 1000 | 11.2% | 0.6% | 11.2% |
| Babbling RTU (random extra bursts) | 1000 | 46.5% | 7.4% | 46.5% |

## Reading the numbers
- Faults that move one number a lot (a silent channel, a 10 dB noise rise) are caught by both.
- The model's advantage is on faults that change the *shape* of the traffic without crossing a
  fixed limit: one RTU out of several going quiet (fewer answered polls, a different rhythm),
  a fading link, mild interference, a babbling radio.

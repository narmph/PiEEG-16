# Plan: Reconfiguring PiEEG-16 for Surface EMG

**Status:** proposal, not yet implemented
**Hardware:** PiEEG-16 (2 × TI ADS1299) on Raspberry Pi 5, kernel 6.6.x, `gpiod==1.5.4`
**Scope:** acquisition, storage, validation, and offline/online processing of surface EMG. Using EMG to drive an actuator (e.g., exo control) is out of scope and needs its own safety review.

> **Verification note.** Register bit fields, gain codes, and test-signal specs below come from memory of the ADS1299 datasheet (TI SBAS499). Check each against the datasheet before use. Phase 0 includes bench tests that confirm the most important ones (scaling, sample rate) empirically. Items marked **[HW?]** depend on PiEEG-16 board wiring, which this repo does not document; check them against the schematic.

---

## 1. Summary

The EEG code works, but four of its settings make it unsuitable for EMG:

| # | Issue | Current | EMG requirement |
|---|-------|---------|-----------------|
| 1 | Sample rate | 250 SPS (`CONFIG1 = 0x96`); the ADS1299 sinc filter's −3 dB point is ≈ 0.26·f_DR ≈ 65 Hz | ≥ 1 kSPS, preferably 2 kSPS (sEMG band ≈ 20–450 Hz) |
| 2 | Montage | All channels referenced to one SRB1 electrode (`MISC1 = 0x20`) | Bipolar (differential) pair per muscle **[HW?]** |
| 3 | Software filters | 1 Hz HP + 10 Hz LP (variable names swapped), hard-coded to 250 samples | 20–450 Hz BP, optional notch, rectify + envelope |
| 4 | Acquisition loop | Single-threaded Python: polls DRDY, filters, and plots in the same loop | Dedicated acquisition process that writes raw data; plotting decoupled; dropped samples detected |

Bugs that affect EMG and EEG alike, found during review (details in §7):

- The µV scaling appears to be **2× too small**, and sign extension subtracts 2²⁴−2 instead of 2²⁴.
- `command_line/utils.py` writes gain codes into the wrong CHnSET bits. For example, `gain=24` actually selects the BIAS_DRP mux input at gain 1.
- `pieeg_to_csv.py` samples on a wall-clock timer instead of DRDY.

**Approach:** build a small, self-contained `emg/` package next to the existing scripts rather than editing them. The upstream EEG scripts stay untouched, so pulling from `pieeg-club/PiEEG-16` stays conflict-free. The shared bug fixes can be offered upstream as a separate PR.

---

## 2. Target ADC configuration

All values apply to both chips unless noted. Every register is written from a single config object (§3) instead of the 3+ copy-pasted blocks that exist today.

| Reg | Name | Current | Proposed | Rationale |
|-----|------|---------|----------|-----------|
| 0x01 | CONFIG1 | `0x96` (250 SPS) | `0x94` (1 kSPS) or `0x93` (2 kSPS) | Bandwidth. DR bits [2:0]: `100`=1k, `011`=2k. Start at 1 kSPS; move to 2 kSPS only if Phase 2 timing tests pass. |
| 0x02 | CONFIG2 | `0xD4` | `0xD4` (unchanged) | Configures the internal test signal. It is only applied when CHnSET.MUX = `101`; Phase 0 uses it. |
| 0x03 | CONFIG3 | `0xFF` | `0xEC` *(verify)* | Internal reference buffer on, bias reference internal, bias buffer on. BIAS_MEAS and LOFF_SENS off unless needed. |
| 0x04 | LOFF | `0x00` | `0x00` for recording; DC lead-off for the pre-trial check | See §5.3 (electrode contact check). |
| 0x05–0x0C | CHnSET | `0x00` (gain 1, normal input) | Used channels: `0x40` (gain 8) or `0x50` (gain 12). Unused: `0x81` (powered down, inputs shorted) | Gain 1 has the highest input-referred noise. Gain 12 gives ≈ ±375 mV of range, enough for sEMG plus electrode DC offset. Bits: [7] PD, [6:4] GAIN, [3] SRB2, [2:0] MUX. |
| 0x0D | BIAS_SENSP | `0x00` | Bit set for each used channel | Drives the bias (driven-ground) electrode from the measured channels' common mode. This improves CMRR and 60 Hz rejection. |
| 0x0E | BIAS_SENSN | `0x00` | Bit set for each used channel (bipolar only) | Same as above. |
| 0x0F/0x10 | LOFF_SENSP/N | `0x00` | `0x00` for recording | Enabled only during the contact check. |
| 0x14 | GPIO | `0x80` | `0x80` (unchanged) | Board-specific. Keep it until the board's use of the pin is known **[HW?]**. |
| 0x15 | MISC1 | `0x20` (SRB1 → all INxN) | `0x00` for bipolar; `0x20` for single-ended fallback | See §2.1. |
| 0x17 | CONFIG4 | `0x00` | `0x00` | Continuous conversion, lead-off comparators off. |

### 2.1 Montage decision **[HW?]**

- **Preferred: bipolar.** Each muscle uses INxP and INxN, with roughly 20 mm inter-electrode spacing per SENIAM (Hermens et al., 2000; unverified recall, check before citing). This requires the PiEEG-16 connectors to expose the INxN pins.
  - **Action:** check the schematic or probe the board. If INxN is tied to SRB1 in hardware, bipolar is impossible without board modification.
- **Fallback: single-ended.** Keep `MISC1 = 0x20` and place the reference electrode on an electrically quiet bony site, such as the patella or a malleolus. Expect more crosstalk and more motion artifact.
  - Pseudo-bipolar recordings are possible offline: put two electrodes over each muscle on separate channels and subtract. Common-mode rejection is worse than true differential input, but it is usable.
- Either way, place a bias electrode on a bony landmark away from the recorded muscles.

### 2.2 Inter-chip clock **[HW?]**

`CONFIG1.CLK_EN = 0` on both chips, so each runs its own internal oscillator. The two 8-channel banks therefore drift relative to each other, and their START commands are issued milliseconds apart.

Options, in order of preference:

1. If the board routes CLK from chip 1 to chip 2 and lets chip 2 select an external clock (the CLKSEL pin), set chip 1 `CLK_EN = 1` (`CONFIG1 = 0xB4`/`0xB3`). Then start both chips with the hardware START pin if it is accessible.
2. Otherwise, read each chip on its own DRDY, timestamp each frame with its own kernel timestamp, and align the banks offline.
3. Alternatively, record time-critical muscles on a single chip (≤ 8 channels).

Phase 0 measures the actual drift (§6, test 0.6).

---

## 3. Software architecture

New directory, leaving the existing scripts untouched:

```
emg/
├── __init__.py
├── ads1299.py        # Register map, opcodes, two-chip SPI driver, frame decode
├── config.py         # EMGConfig dataclass (+ YAML/JSON load), validation
├── acquire.py        # CLI recorder: DRDY-synchronous, raw codes → disk
├── live_view.py      # Separate process: decimated raw + envelope plots
├── processing.py     # Offline/online filters, envelope, normalisation
├── io.py             # Raw file reader/writer, sidecar metadata, CSV export
├── tools/
│   ├── check_regs.py     # Read back all registers, verify device ID
│   ├── test_signal.py    # Internal test signal / shorted-input noise runs
│   └── timing_bench.py   # Measures loop timing and drop rate at a given fs
└── configs/
    ├── default_bipolar_1k.yaml
    └── single_ended_1k.yaml
```

### 3.1 `ads1299.py`: driver

- Register addresses and opcodes as named constants, defined once. The magic numbers currently scattered across 5 files go away.
- `ADS1299Pair` class owns:
  - both `spidev` handles;
  - the GPIO19 CS line for chip 2;
  - both DRDY lines (GPIO26, GPIO13). The pins come from the existing scripts; confirm them against the board.
- Methods:
  - `reset()`, `write_reg()`, `read_reg()`
  - `apply(config)`, which writes then reads back every register and raises on mismatch
  - `start()`, `stop()`, `read_frame()`
- `read_frame()` returns raw signed 24-bit counts as `int32[16]` plus both 3-byte status words. **No µV conversion in the hot path.**
- Status-word check on **both** chips (the top nibble should be `0b1100`). The current GUI checks only chip 2.
- Device ID check at startup: read `ID` (0x00). The low bits identify the ADS1299 variant; check the expected value in the datasheet. Fail loudly on a mismatch, for example a board that isn't seated.
- `gpiochip` path is configurable. The current scripts use a mix of `"gpiochip4"`, `"0"`, and `/dev/gpiochip0`, and the correct one depends on the Pi 5 kernel version (gpiochip0 on 6.6.51).
- SPI clock raised from 1 MHz to 4 MHz (`command_line/utils.py` already uses 4 MHz). Test 8 MHz in Phase 2. The datasheet allows a much higher SCLK, but check the ribbon/trace signal integrity.

### 3.2 `config.py`

```python
@dataclass
class EMGConfig:
    fs: int = 1000                      # 250, 500, 1000, 2000
    gain: int = 12                      # 1, 2, 4, 6, 8, 12, 24
    montage: str = "bipolar"            # "bipolar" | "single_ended"
    channels: dict[int, str]            # {1: "TA_R", 2: "SOL_R", ...}; others powered down
    bias_channels: list[int] | None     # default: all used channels
    spi_hz: int = 4_000_000
    gpiochip: str = "/dev/gpiochip0"
    clock_mode: str = "independent"     # "independent" | "chip1_master"
```

- Derives every register value (CONFIG1 DR bits, CHnSET with the gain correctly shifted into bits [6:4], BIAS_SENSP/N bitmasks, MISC1) from these fields, with no hand-typed hex in scripts.
- Validates inputs, e.g. that fs is a legal DR value and the gain is in the allowed set.
- Saved verbatim into each recording's metadata.

### 3.3 `acquire.py`: recorder

Design goals: never lose a sample silently, and never bake processing into the raw data.

- **Process model:** the acquisition process does only DRDY wait → SPI read → append to a preallocated buffer. A writer thread flushes buffered chunks to disk, and live view gets data through `multiprocessing.shared_memory` (a ring buffer). No matplotlib, no scipy, and no per-sample `print` in the acquisition process.
- **DRDY handling:** use gpiod line **events** (falling edge) where the pinned 1.5.4 API supports them, because events carry kernel timestamps. Busy-polling, as `pieeg16_drdy_only.py:wait_for_drdy` does, is the fallback. Benchmark both in Phase 2.
- **Timing/drop detection:** store a `t_ns` timestamp for every frame (the kernel event timestamp, or `time.monotonic_ns()` taken right after DRDY).
  - Offline, flag any inter-frame gap > 1.5/fs.
  - Online, keep a running count of late frames: a frame is late if DRDY had already gone low again before the read finished. Print the count once per second.
  - The ADS1299 has no hardware sample counter, so host timestamps are the only check.
- **File format:** a raw binary file of `int32` counts, shape (N, 16), plus `int64` timestamps and the status words, with a **JSON sidecar** holding:
  - the full `EMGConfig`;
  - register readback;
  - the scale factor (µV/count);
  - channel → muscle labels;
  - subject/session IDs;
  - git commit hash;
  - drop statistics.

  Size at 2 kSPS: about 128 kB/s for the ADC data, or ≈ 0.5 GB/hour. Storing counts plus the scale factor means a scaling mistake can be fixed after the fact. `io.py` provides `load()` → (t, µV array, meta) and a CSV/MAT exporter for MATLAB users.
  - Never rewrite the whole file per block. `Save_data_OS.py` currently rewrites the full `.xlsx` every 250 samples, which is O(N²) and stalls acquisition.
- **CLI:** `python -m emg.acquire --config configs/default_bipolar_1k.yaml --out data/S01_walk01 --duration 120`
- **Shutdown:** on SIGINT/SIGTERM, send STOP/SDATAC, flush, close the file, and write the final drop stats to the sidecar.

### 3.4 `live_view.py`

- Separate process that reads the shared-memory ring buffer.
- Plots per channel:
  - a decimated, band-passed raw trace (last ≈ 5 s);
  - a rolling RMS/envelope bar or trace;
  - a 60 Hz power indicator (useful for spotting a bad electrode).
- Blitting or pyqtgraph instead of `plt.pause` redraws of 16 axes.
- Must be able to run headless (latest commit) or over the network. A dropped frame in the viewer must never stall acquisition.

### 3.5 `processing.py`

Offline defaults (zero-phase):

| Step | Default | Notes |
|------|---------|-------|
| DC/offset removal | Mean subtraction or HP | Included in BP |
| Band-pass | 4th-order Butterworth, 20–450 Hz, `sosfiltfilt` | Upper edge must be < fs/2; at 1 kSPS, 450 Hz is close to the sinc roll-off (§6, test 0.4). Use second-order sections (SOS) for numerical stability. |
| Line noise | Off by default; optional 60 Hz + harmonics notch (Q ≈ 30) or spectrum interpolation | A notch removes real EMG power near 60 Hz. Prefer fixing the source (bias electrode, skin prep, cable dress). US mains = 60 Hz. |
| Rectification | Full-wave (`abs`) | |
| Envelope | 4th-order Butterworth LP, `sosfiltfilt`, 6 Hz (gait); configurable 3–10 Hz | Report the cutoff, since it affects timing and amplitude |
| Normalisation | % MVC, or % peak dynamic for gait | Store the normalisation trial ID in metadata |
| Onset detection (optional) | Threshold = rest mean + k·SD, with a minimum duration | Report k and the window |

Online (causal) variants of the same chain go in a separate function. It uses `sosfilt` with persistent state (`zi`) and documents its group delay at the envelope cutoff. This is required if EMG ever feeds a controller. Today's GUI calls `filtfilt` on 500-sample windows, which is non-causal and causes edge artifacts at every block boundary.

Reporting: follow the ISEK reporting standards (Merletti, 1999, *J. Electromyogr. Kinesiol.*; unverified recall, confirm the reference before citing). Report electrode type, size, and spacing; placement (SENIAM); skin prep; amplifier gain and CMRR; sample rate; and all filter parameters.

### 3.6 Dependencies

`requirements.txt` additions: `numpy` (currently implicit), `pyyaml`, optionally `pyqtgraph`. Keep `gpiod==1.5.4` pinned. Drop the `RPi.GPIO` import in `command_line/utils.py`, since RPi.GPIO does not support the Pi 5.

---

## 4. Implementation phases

Each phase ends with a pass/fail gate. Don't start the next phase until the gate passes.

### Phase 0: Hardware facts and bench validation (no human subject)

1. Get the PiEEG-16 schematic. Answer the **[HW?]** items: INxN breakout, CLK/CLKSEL/START routing between chips, BIAS electrode pin, use of the GPIO register, and whether ADS1299 GPIO pins are exposed (for sync, §5.2).
2. Write `ads1299.py` + `tools/check_regs.py` and run register read-back on both chips.
3. Run the bench tests in §6 (0.1–0.6).

**Gate:** the device ID is correct on both chips; scaling is confirmed against a known input; the measured fs is within the datasheet tolerance.

### Phase 1: Driver + config + recorder at 1 kSPS

1. Write `config.py` and `acquire.py`, recording to raw binary + JSON sidecar.
2. Write `io.py` `load()` and the CSV/MAT export.
3. Run a 30 min bench recording with the function generator connected and no plotting.

**Gate:** zero dropped frames in 30 min at 1 kSPS; an FFT of the recording shows the generator's tone at the right frequency and amplitude.

### Phase 2: Throughput hardening and 2 kSPS

1. Write `tools/timing_bench.py` and compare: gpiod events vs polling; SPI at 4 vs 8 MHz; `readbytes` vs `xfer2`.
2. Pin the acquisition process to an isolated core (`taskset`, optionally `isolcpus`), use `SCHED_FIFO` via `chrt` if permissions allow, and set the CPU governor to `performance`.
3. Record the drop rate with live view running and with Wi-Fi/SSH active.

**Gate:** zero drops in 30 min at the target fs **with live view running**. If Python can't sustain 2 kSPS, choose one of:

- stay at 1 kSPS and accept a ~400 Hz effective bandwidth;
- move the read loop to a small C extension or helper process;
- use one chip (8 channels) to halve the SPI load.

### Phase 3: Live view + processing

1. Write `live_view.py` and `processing.py`.
2. Add unit tests for `processing.py`:
   - a synthetic sine/noise input with a known envelope;
   - filter frequency response checked against the design;
   - causal filter delay matches the documented value.

**Gate:** the tests pass; live view runs for 30 min without affecting the drop rate.

### Phase 4: Human pilot (after IRB/safety sign-off, §5.1)

1. One or two lab members; Ag/AgCl gel electrodes; skin prep (shave, abrade, alcohol).
2. Muscles that are easy to check, e.g. tibialis anterior, medial gastrocnemius, vastus lateralis.
3. Protocol:
   - rest (noise floor);
   - MVC (3 × 5 s);
   - graded contractions;
   - treadmill walking at a fixed speed (motion artifact, envelope shape vs literature).
4. If the lab has a commercial EMG system (e.g., Delsys), record simultaneously from adjacent sites on one muscle and compare envelope correlation and onset timing.
5. Metrics:
   - rest RMS (µV);
   - SNR (MVC RMS / rest RMS);
   - 60 Hz power fraction;
   - drop count;
   - envelope cross-correlation and lag vs the reference system.

**Gate:** results meet the lab's acceptance criteria. Set those before the pilot, not after.

### Phase 5: Documentation and cleanup

1. Add an EMG section to `setup.md`: wiring diagram, electrode placement, config files, run commands.
2. Offer the shared-bug fixes (§7) upstream as a separate PR.

---

## 5. Experimental and safety considerations

### 5.1 Electrical safety

- **Battery power only while electrodes are on a person.** No mains-powered HDMI monitor, USB hub, or wall adapter connected to the Pi during recording. Run headless over Wi-Fi.
  - `setup.md` §3 currently describes connecting a monitor and a wall adapter. Add a warning there.
- PiEEG is not a medical device and has no IEC 60601 isolation rating. Any human recording needs IRB coverage that accounts for this.
- **Any wired connection to other lab equipment is an isolation hazard.** This includes trigger lines, a shared ground with mocap/force plates, and an exo controller. Use opto-isolated sync only.

### 5.2 Synchronisation with other lab systems

Gait work usually needs EMG aligned with mocap, force plates, and/or exo data. Options, best first:

1. **Trigger in the ADS1299 status word.** If the ADS1299 GPIO pins are broken out **[HW?]**, an opto-isolated TTL trigger into an ADS1299 GPIO appears in the status word of every frame, which makes it sample-accurate.
2. **Trigger on a spare ADC channel.** Record the opto-isolated trigger through a divider into a spare ADC channel. This is also sample-accurate, and costs one channel.
3. **Trigger on a Pi GPIO.** Use an opto-isolated trigger on a Pi GPIO with kernel event timestamps, mapped to frames by `t_ns`. Accuracy is ~sub-ms, limited by timestamp jitter.
4. **Network time / start-command sync only.** Not adequate for onset-timing analyses.

### 5.3 Electrode contact check

- Add `tools/contact_check.py`. It enables DC lead-off detection for about 2 s (LOFF current, then LOFF_SENSP/N), reads the LOFF_STATP/N bits from the status word, then disables lead-off and reconfigures for recording.
- This gives a pass/fail per electrode, not an impedance value. If impedance numbers are needed, use an external impedance meter.

---

## 6. Bench validation tests (Phase 0)

| Test | Setup | Pass criterion | Resolves |
|------|-------|----------------|----------|
| 0.1 Register readback | `check_regs.py` | Every written register reads back; device ID as expected | SPI/CS wiring, chip-2 CS on GPIO19 |
| 0.2 Shorted-input noise | CHnSET MUX=`001`, each gain, at 1 k / 2 kSPS, 60 s | Input-referred noise within the datasheet noise table for that gain/fs | Baseline noise floor; chooses the gain |
| 0.3 Internal test signal | CHnSET MUX=`101`, CONFIG2=`0xD4` | Measured square-wave amplitude matches the datasheet test-signal spec (≈ (V_REFP−V_REFN)/2.4 mV × CAL_AMP factor; verify definition) | **Confirms or refutes the 2× scaling error** |
| 0.4 Amplitude / frequency response | Function generator → attenuator (e.g. 1000:1) → differential input; 1 mVpp sine swept 5–500 Hz | Amplitude matches within ~2 % in band; roll-off near f_DR matches the sinc3 response; FFT peak at the generator frequency | Absolute scaling, effective bandwidth, **actual fs** |
| 0.5 Long-run drop test | Generator connected, 30 min at target fs | Zero timestamp gaps > 1.5/fs; zero phase jumps in the sine | Throughput |
| 0.6 Inter-chip sync | Same generator signal into ch1 (chip 1) and ch9 (chip 2), 30 min | Measure lag vs time via windowed cross-correlation; drift (ppm) and initial offset recorded | Whether inter-chip alignment is needed (§2.2) |

Use battery power for the Pi during noise tests (0.2) so they reflect real recording conditions.

---

## 7. Existing bugs to fix (affect EEG too)

| Location | Bug | Fix |
|----------|-----|-----|
| All scripts, e.g. `GUI/2.Graph_Gpio_D _1_5_4_OS.py` and `Save_data/command_line/utils.py:get_voltage` | µV = `1e6 * 4.5 * code / 16777215`. Datasheet LSB = 2·V_REF / gain / 2²⁴, so it should be `1e6 * 4.5 * code / (gain * 2**23)`. Current values are **2× too small** at gain 1 and ignore gain. | Use the correct formula; confirm with test 0.3/0.4 |
| Same | Sign extension subtracts `16777214`; it should subtract 2²⁴ = `16777216` (2-count error) | Use `code - (1 << 24) if code & 0x800000 else code` |
| `Save_data/command_line/utils.py:convert_gain` | Returns `0b000…0b110` written directly to CHnSET, but gain occupies bits [6:4]. So `gain=24` writes `0x06`: gain 1, MUX=`110` (BIAS_DRP), and **the electrodes aren't measured at all**. | `return code << 4` |
| `Save_data/command_line/pieeg_to_csv.py` | Reads on a wall-clock timer (default 10 Hz), not DRDY, so it reads frames asynchronously with the 250 SPS ADC and can catch mid-update data. It also reopens the CSV every sample. | Superseded by `emg/acquire.py` |
| `Save_data/command_line/utils.py` | Imports `RPi.GPIO` (not supported on Pi 5); uses the `gpiod.Chip` / `LINE_REQ_DIR_OUT` API, which appears to belong to a different gpiod binding than the pinned `gpiod==1.5.4` (`gpiod.chip` / `line_request`) used elsewhere | Superseded; verify if fixing in place |
| `GUI/2.Graph_Gpio_D _1_5_4_OS.py` | Validates the status header only for chip 2; waits only on chip 1 DRDY | Check both; see §2.2 |
| Same | `highcut = 1` is used as the HP cutoff and `lowcut = 10` as the LP cutoff (names swapped) | Rename |
| Same | Ch 7 axis limits use `data_for_graph_1` | Use `data_for_graph_7` |
| `Save_data/Save_data_OS.py` | Rewrites the entire `.xlsx` every 250 samples (O(N²); stalls acquisition on long runs) | Append-only binary/CSV |
| Multiple | gpiochip name is inconsistent (`"gpiochip4"`, `"0"`, `/dev/gpiochip0`); the correct value depends on Pi 5 kernel version | Config field |

---

## 8. Open questions (answers change the plan)

1. **Does the PiEEG-16 expose INxN?** This decides bipolar vs single-ended (§2.1).
2. **How many muscles, and which?** If ≤ 8, one chip avoids inter-chip sync entirely and halves the SPI load.
3. **Offline analysis only, or real-time use?** If EMG feeds an exo controller, latency, causal filtering, and a separate actuator-safety review become requirements.
4. **What must EMG be synchronised with,** and to what tolerance (§5.2)?
5. **Electrodes:** gel Ag/AgCl (recommended for validation) or dry? Dry electrodes raise contact impedance and motion artifact considerably.
6. **Is a reference EMG system available** for the Phase 4 comparison?

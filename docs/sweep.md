# Wideband sweep with the B210

CyberEther's **SDR Sweep** block retunes a radio in hops, transforms each capture and stitches the pieces into one spectrum, for example 70 MHz to 1 GHz in a single trace and waterfall. Upstream, the block only knows radios with a native CyberEther driver (HackRF, PlutoSDR, LimeSDR, bladeRF, Airspy, RTL-SDR, BirriaSDR). [Patch 3](../patches/0003-sdr-sweep-soapysdr-devices.patch) adds a SoapySDR backend, so the B210 can sweep through SoapyUHD.

## Result

Measured on the B210 clone and the same laptop as the main write-up, sweeping 70 MHz to 1 GHz with the default settings below:

| Measure | Value |
|---|---|
| Plan | 38 hops of 1229 samples at 61.44 MS/s, 18 604 bins of about 50 kHz |
| Sweep speed | 1.62 GHz/s, about 1.7 sweeps per second |
| Time per hop | about 15 ms, mostly UHD retune and burst start |
| Buffer Fill / Buffer Loss | 0 % / 0.34 %, decreasing (a few frames dropped at start-up) |
| Radio overflows | none (no `O`) |

## Run it

Apply patch 3 on top of patches 1 and 2, rebuild, then use the [launcher](../launcher/ce):

```bash
SERIAL=31B92DD ce scan   # use your own B210's serial
START=400000000 END=2500000000 RES=100000 SERIAL=31B92DD ce scan
```

`SoapySDRUtil --find="driver=uhd"` prints the serial. `START` and `END` are in Hz, `RES` is the bin width in Hz, `DWELL` is the number of captures averaged per hop and `SCANGAIN` is a fixed gain in dB. Automatic gain is avoided because the levels of different hops would no longer be comparable.

In the block itself, the device appears as `B210 <serial> (SoapySDR)`.

## How the hops are laid out

- **Usable width:** at 61.44 MS/s a capture spans 61.44 MHz. The block keeps the central 80 %, about 49 MHz, because the analog filter attenuates the edges.
- **Step:** the radio moves by half of that, about **24.5 MHz**, so every frequency is measured by two captures.
- **Stitching:** each capture is weighted by a cos² taper, strongest at the centre and zero at the edges. Two tapers shifted by half their width add up to a constant, so the stitched spectrum is flat, without steps at the seams.

This 50 % overlap is how the upstream block plans every radio. It halves the speed compared with 49 MHz steps, but avoids visible seams.

## Capture Rate and Settling Time

Both read **0** in the block. Here 0 means "let the driver choose", not "nothing".

**Capture Rate = 0 (Preferred)** uses the driver's preferred sweep rate. UHD only reports the rates of the B200's current master clock (16 MHz / n), so the backend probes a few higher rates when it opens the device and keeps the highest one UHD accepts: **61.44 MS/s**. The capture rate sets the width of each hop and therefore the number of hops:

| Capture Rate | Usable width | Step | Hops for 70 MHz–1 GHz |
|---|---|---|---|
| 61.44 MS/s (0) | ≈ 49 MHz | ≈ 24.5 MHz | 38 (measured) |
| 30.72 MS/s | ≈ 24.5 MHz | ≈ 12.3 MHz | ≈ 76 (estimated) |

A lower rate only slows the sweep: the spectral detail comes from **Resolution**, since the block sizes the FFT to match it. At 61.44 MS/s the capture exceeds the AD9361's ~56 MHz analog bandwidth, which the 80 % usable width already leaves out.

**Settling Time = 0** uses the driver's default of **1 ms**. After each retune, the samples taken while the synthesizer, DC offset and I/Q corrections settle are discarded: 61 440 samples at 61.44 MS/s, before keeping the 1229-sample capture (20 µs). Each hop is captured as one burst requested after the retune, so no sample predates it; the settling time only covers analog transients.

| Part of a hop | Time |
|---|---|
| UHD retune and burst start | most of the ~15 ms |
| Settling (discarded) | 1 ms |
| Capture | 0.02 ms |

Lowering the settling time therefore gains very little speed. Raise it, to 2 or 5 ms, if you see spurs or ghost signals at the seams, a noise floor that steps from one hop to the next, or a peak at the centre of each capture that stays.

## Port names

UHD names the two connectors of each channel `TX/RX` and `RX2`, and CyberEther shows these names. On the clone's case, the port marked **RX1** is UHD's `RX2` on channel A, the one the sweep listens to (confirmed with a 50 Ω termination). See the [README](../README.md#reproduce).

## Notes

- The SoapySDR backend only sweeps. For continuous streaming, keep using the Soapy block as in the main write-up.
- Radios with a native CyberEther driver are not listed a second time through SoapySDR.
- Do not set `master_clock_rate=61.44e6` in `uhd.conf`: SoapySDRUtil's probe then failed with `get_time: Timeout`. The backend's rate probe reaches 61.44 MS/s without it.
- Patch 3 is not yet proposed upstream.

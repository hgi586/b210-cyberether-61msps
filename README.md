# 61.44 MS/s with zero sample loss

Live spectrum observation at 61.44 MS/s with a USRP B210 clone, [CyberEther](https://github.com/luigifcruz/CyberEther) and a laptop GPU. A practical methodology for low-budget R&D.

Full write-up (PDF): [b210-cyberether-61msps-zero-loss.pdf](b210-cyberether-61msps-zero-loss.pdf)

| Result | Live spectrum observation |
|---|---|
| Sample rate | 61.44 MS/s complex (the AD9361 maximum) |
| Throughput | 491 MB/s of CF32 samples, steady |
| Buffer fill | about 13 % |
| Buffer loss / radio overflows | 0 % / 0 OVF over a 15-minute run |
| Hardware | Dell Precision 3560 laptop (i7-1185G7, Iris Xe, NVIDIA T500 2 GB), B210 clone on USB 3 |

The default settings lost between 2 % and 60 % of the samples. Each bottleneck below was found by profiling (`perf`, `top -H`), not by guessing.

## The pipeline

| Stage | Runs on | Data |
|---|---|---|
| B210 clone (AD9361 + FPGA image) | radio | sc16 over USB 3: 61.44 MS/s × 4 B = 245.76 MB/s |
| UHD 4.9 via SoapySDR / SoapyUHD | CPU | converted to CF32: 491.52 MB/s |
| CyberEther SoapySDR block | CPU | ring buffer, batches of FFT frames |
| Duplicate block (to CUDA) | CPU → GPU | pinned host memory, read by the GPU over PCIe |
| Spectrum analyzer (window, FFT, dB, scaling) | NVIDIA T500 (CUDA) | line plot + waterfall |
| Display (Vulkan) | Intel Iris Xe | GUI rendering |

Each chip does the job it is best at: the CPU feeds the USB link, the discrete GPU does the math, the integrated GPU draws the result.

## The measurement journey

| Step | Configuration | Fill | Loss | Throughput |
|---|---|---|---|---|
| 0 | Stock build: only 16 MHz / n rates accepted (16, 8, 5.33, 4 … MS/s) | low | 0 % | far below 61.44 MS/s |
| 1 | Rate fix, debug-optimized build, CPU, Batches 32 / Mult. 32 | 98 % | 10 % | — |
| 2 | CUDA, Duplicate with managed memory | 99 % | 60 % | — |
| 3 | Debug-optimized build, CPU, Batches 8 / Mult. 4 | 18–82 % | ≈ 3.6 % | 450–491 MB/s |
| 4 | Release + AVX-512 build, CPU, Batches 8 | 20–90 % | 11 % | 420–491 MB/s |
| 5 | Release + AVX-512 build, CPU, Batches 16 | — | ≈ 2 % | 470–491 MB/s |
| 6 | CUDA, Duplicate with pinned memory, 256 USB frames | 13 % | 0 % | 491 MB/s, 1 OVF burst in 8 min |
| **7** | **Same, 1024 USB frames** | **13 %** | **0 %** | **491 MB/s, 0 OVF in 15 min** |

## The fixes

1. **Let the sample rate through** ([patch 1](patches/0001-soapy-accept-sample-rates-outside-reported-ranges.patch)). UHD reports B200 rates as `master_clock / n` for the *current* clock but reconfigures the clock on demand. The device opened with a 16 MHz clock, and CyberEther rejected anything outside `16 MHz / n` (e.g. 5 MS/s, or anything above 16 MS/s), so 61.44 MS/s was unreachable. The patch only rejects non-finite or non-positive rates and lets the driver decide.
2. **Known-good FPGA image.** `UHD_IMAGES_DIR=$HOME/uhd_images` and `fpga=.../usrp_b210_fpga_github.bin` in the device string.
3. **Optimized build.** `meson setup build-release -Dbuildtype=release -Dnative=true -Dc_std=gnu17 -Dcpp_args=-fcx-limited-range`. `-march=native` enables AVX-512; `-fcx-limited-range` removes the slow NaN-safe complex multiply (`__mulsc3`).
4. **Batches must be a multiple of 16.** pocketfft vectorizes *across* transforms, in groups of 4 (SSE), 8 (AVX) or 16 (AVX-512). With 8 batches on an AVX-512 build every FFT ran scalar and loss rose from 3.6 % to 11 %. Batches = 16 fixed it.
5. **GPU for compute, Intel for display.** `--device-index` picks both the Vulkan and the CUDA device. Vulkan device 0 is the Intel iGPU and the T500 is CUDA device 0, so the default index gives exactly this split. A Duplicate block (`outputDevice: cuda`, `hostAccessible: true`) bridges the CPU source to the CUDA spectrum analyzer.
6. **Pinned memory instead of managed memory** ([patch 2](patches/0002-cuda-pinned-host-memory.patch)). Host-accessible CUDA buffers used `cudaMallocManaged`; pages migrated every frame (60 % loss). `cudaHostAlloc(..., cudaHostAllocMapped | cudaHostAllocPortable)` lets the CPU write at full speed and the GPU read over PCIe. Result: 0 % loss, GPU ≈ 36 % busy.
7. **Deeper USB receive queue.** `num_recv_frames=1024` (≈ 34 ms of samples instead of ≈ 8 ms with 256) absorbs short system hiccups.

## Reproduce

- Apply both patches to CyberEther (tested on commit `fa027fb8`, v1.19) and build as in fix 3.
- Use [flowgraph/b210-spectrum-cuda.yml](flowgraph/b210-spectrum-cuda.yml) (replace `/home/USER`), or the [launcher/ce](launcher/ce) script: `ce` (CPU) or `ce gpu` (CUDA). Settings can be overridden per run, e.g. `B=32 ce gpu`.

Practical notes:

- **Build memory:** use `-j 4`, `systemd-run --user --scope -p MemoryMax=10G` and zram; a full parallel build froze a 16 GB laptop.
- **CUDA on Ubuntu:** with `nvidia-cuda-toolkit`, a small directory of symlinks in the NVIDIA layout plus `CUDA_PATH` lets Meson find CUDA.
- **Live USB Ubuntu:** `cuInit` returned 999 until `nvidia-uvm` was loaded and its device node created (made permanent with a udev rule).
- **Counters:** CyberEther's Buffer Loss is cumulative since start; UHD prints `O` when the radio itself overflows.
- **Port names on the clone:** UHD names the two connectors of each channel `TX/RX` and `RX2`, and CyberEther shows these names. The clone's case numbers its ports by channel instead: the port marked **RX1** is UHD's `RX2` on channel A (antenna `RX2`, channel 0), confirmed with a 50 Ω termination. The case's RX2 port is presumably channel B's receive input (not tested).

## Caveats

- 61.44 MS/s exceeds the AD9361's ~56 MHz analog bandwidth (UHD warns): the band edges are attenuated.
- Verified on this clone, FPGA image and laptop, over 15 minutes.
- Patch 2 changes every host-accessible CUDA buffer and is an experiment; patch 1 is proposed upstream.
- The CPU-only path still loses about 2 % because the spectrum chain runs on one thread.

## What's next

The spectrum is only the beginning: raw IQ capture to SSD for offline analysis, real-time processing with custom GPU algorithms (detection, demodulation, classification), and replay of captures through the same flowgraphs.

## Acknowledgements

- [@ossdrlab](https://x.com/ossdrlab), maker of the USRP B210 clone.
- [@JamesMcPherson](https://x.com/JamesMcPherson), for the FPGA image that runs the clone without lost packets.
- [@luigifcruz](https://github.com/luigifcruz) and the CyberEther contributors, for the software that computes and displays the spectrum.
- Ettus Research / NI for UHD, and the Pothos project for SoapySDR.
- [Claude Code](https://claude.com/claude-code) (Anthropic), which did the installation, profiling, patches and this write-up together with the author.

## License

[MIT](LICENSE). The patches modify CyberEther, which is also MIT-licensed (© Luigi F. Cruz).

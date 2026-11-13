# LARC — PUP EN24C T2R4 24 GHz FMCW Radar

Capture and processing code for the **PUP EN24C T2R4** radar: 24 GHz FMCW, 2 Tx / 4 Rx, connected over USB through a Cypress FX2LP controller.

The vendor only ships a Windows MATLAB GUI. This repo keeps that GUI as the reference and adds a standalone C++ port, so the radar can be driven from a Raspberry Pi with no MATLAB on the device.

| Part | Runs on | Purpose |
|---|---|---|
| [Matlab src/](Matlab%20src/) | Windows + MATLAB | Vendor GUI, plus the same steps extracted into scripts. Reference implementation. |
| [linux/](linux/) | Raspberry Pi 4/5 (aarch64) or any Linux host | `pupradar_capture`: configure the radar and record raw IQ to disk. |
| [processing_src/](processing_src/) | MATLAB on a PC | Turn recordings into range profiles; compare Pi captures against GUI captures. |

There is no on-device signal processing. The Pi records, and MATLAB processes afterwards.

## Repository structure

```
LARC/
├── Matlab src/                  Windows acquisition (reference implementation)
│   ├── PUPradarGUI.m / .fig     Vendor GUI: init, configure, live plots, record to .mat
│   ├── run_GetComplexData.m     Scripted capture without the GUI
│   ├── run_Send_PLL_Sawtooth.m  Compute or send the chirp PLL registers (has a dry-run mode)
│   ├── makeRadarHandles.m, PUPradar_initiating.m, Send_Basic_Parameter.m,
│   │   Send_PLL_Sawtooth.m, GetComplexData.m     GUI steps extracted as plain functions
│   ├── *.mexw64                 Prebuilt USB wrappers (Cypress CyAPI)
│   └── SDR_USB_FW.hex           FX2LP firmware, loaded into RAM on every run
│
├── linux/                       C++17 capture tool (libusb-1.0)
│   ├── include/pupradar/, src/
│   │   ├── Protocol             Opcodes, command packets, PLL maths, sampling tables
│   │   ├── IntelHex             Firmware .hex parser
│   │   ├── IUsbBackend, UsbDevice   USB interface and its libusb implementation
│   │   ├── RadarSession         initialize / configure / captureIq
│   │   ├── IqWriter             Writes <out>.bin and <out>.json
│   │   ├── main.cpp             pupradar_capture CLI
│   │   └── board_info.cpp       pupradar_board_info connectivity check
│   ├── tests/                   Unit tests against a fake USB backend (no hardware)
│   ├── firmware/SDR_USB_FW.hex  Same firmware as in Matlab src/
│   ├── scripts/                 build_native.sh, build_cross.sh, deploy_pi.sh, 99-pupradar.rules
│   ├── toolchains/              aarch64 cross-compile toolchain file
│   └── CMakeLists.txt, Makefile
│
├── processing_src/              MATLAB post-processing
│   ├── process_pi.m             Load a Pi .bin into the GUI's sweep matrix, compute range profile
│   ├── process_matlab.m         Same for a GUI .mat recording
│   ├── process_pi_matlab.m      Script: overlay one Pi capture on one GUI capture
│   ├── process_pi_matlab_integration.m   Script: compare several Pi runs
│   ├── simulation_*.m           FMCW simulations (static, moving, background, sawtooth)
│   └── plotRangeProfile*.m, launchInteractive*.m, archive/   Older helpers
│
├── data/                        Recorded captures, about 1.8 GB
│   └── <DDMMYYYY>[_matlab|_pi]/     _matlab = GUI .mat files, _pi = .bin + .json pairs
│
└── docs/                        Datasheets, operating manual
    └── Mex src/                 MEX C++ sources, PupRadar_Protocol.docx, PupRadar_DataFormat.docx
```

`PUPradarGUI.m` and `docs/Mex src/` are the vendor reference for the USB protocol. Treat them as read-only.

## Prerequisites

| Task | You need |
|---|---|
| Run the MATLAB GUI or scripts | 64-bit Windows, MATLAB, and the Cypress USB driver bound to the radar (the `.mexw64` files call CyAPI) |
| Process captures | MATLAB R2021a or newer with Signal Processing Toolbox |
| Run unit tests | Linux or WSL with `cmake` 3.16+, `make`, `g++` (C++17) |
| Build the tools natively | The above plus `pkg-config` and `libusb-1.0-0-dev` |
| Cross-compile for the Pi | The above plus `g++-aarch64-linux-gnu` and `libusb-1.0-0-dev:arm64` (setup below) |
| Run on the Pi | Raspberry Pi OS 64-bit, `libusb-1.0-0`, SSH access for deployment |

Building with MSVC on Windows is not supported. Use WSL.

One-time cross-compile setup on Ubuntu 22.04 under WSL (replace `jammy` if you use another release):

```bash
sudo dpkg --add-architecture arm64
sudo sed -i -E 's|^deb (http.*)|deb [arch=amd64] \1|' /etc/apt/sources.list
echo 'deb [arch=arm64] http://ports.ubuntu.com/ubuntu-ports/ jammy main universe' \
    | sudo tee /etc/apt/sources.list.d/arm64.list
sudo apt-get update
sudo apt-get install -y cmake make pkg-config g++-aarch64-linux-gnu libusb-1.0-0-dev:arm64
```

## Development

### C++ capture tool

Run everything from `linux/`. The Makefile is a thin wrapper around CMake.

| Command | Result |
|---|---|
| `make tests` | Debug build of the unit tests, then `ctest`. Needs neither libusb nor hardware. |
| `make native` | Release build for the current machine, in `build-native/`. Also works on the Pi itself. |
| `make` or `make cross` | Release build for aarch64, in `build-arm64/`. |
| `make deploy PI=user@host` | Cross-compile, then copy to the Pi (see Deployment). |
| `make clean` | Remove all build directories. |

`scripts/build_native.sh` and `scripts/build_cross.sh` do the same builds without `make`. `build_native.sh` also runs the tests.

The tests replace the USB layer with `tests/FakeUsbBackend.hpp`, which records every call and replays scripted replies. They cover hex parsing, command packets, PLL register maths, the sampling tables, the bring-up and configure sequences, and the output files.

The C++ code is a port of the MATLAB sequence `PUPradar_initiating` → `Send_Basic_Parameter` → `Send_PLL_Sawtooth` → `GetComplexData`, and its comments cite the matching GUI lines. When you change protocol behaviour, check it against `Matlab src/` and `docs/Mex src/` first.

### MATLAB acquisition (Windows)

Open MATLAB in `Matlab src/` so the `.mexw64` files are on the path.

- `PUPradarGUI` starts the vendor GUI. Its Record button saves `ComplexDataTx1` / `ComplexDataTx2` to a `.mat` file.
- `run_GetComplexData` captures without the GUI. Edit the `makeRadarHandles(...)` call at the top to set the configuration.
- `run_Send_PLL_Sawtooth` with `DRY_RUN = true` prints the PLL registers for a chirp without touching hardware.

## Deployment (Raspberry Pi)

**1. Build and copy** from `linux/` on the dev machine:

```bash
make deploy PI=pi@raspberrypi.local
```

This copies `pupradar_capture`, `pupradar_board_info`, `firmware/SDR_USB_FW.hex` and the udev rule to `~/pupradar` on the Pi. Add `DEST=/some/path` to change the directory. (`scripts/deploy_pi.sh` is an older equivalent that copies only `pupradar_capture`.)

**2. Allow non-root USB access**, once per Pi:

```bash
sudo cp ~/pupradar/99-pupradar.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules && sudo udevadm trigger
```

**3. Check the link, then capture.** Run from the deploy directory, because the firmware path defaults to `firmware/SDR_USB_FW.hex` relative to where you are:

```bash
cd ~/pupradar
./pupradar_board_info          # downloads firmware, prints model and Tx/Rx count
./pupradar_capture --out /tmp/cap --receiver Rx4 --sampling-index 4 --fc-high 26.0e9
```

**4. Copy `/tmp/cap.bin` and `/tmp/cap.json` back** to the PC for processing, for example into `data/<date>_pi/`.

### `pupradar_capture` options

The options match the parameters of `makeRadarHandles()` in the MATLAB path.

| Option | Values | Default |
|---|---|---|
| `--out <basename>` | Required. Writes `<basename>.bin` and `<basename>.json` | |
| `--sweep-time-index` | 1–5 = 0.5 / 1 / 2 / 4 / 8 ms | 1 |
| `--transmitter` | `Tx1`, `Tx2`, `Tx1.Tx2` | `Tx1` |
| `--receiver` | `Rx1`…`Rx4`, `Rx1.Rx2`, `Rx3.Rx4`, `Rx1.Rx2.Rx3.Rx4` | `Rx1` |
| `--sampling-index` | 1–4 = base samples per sweep ÷ 1 / 2 / 4 / 8 | 1 |
| `--num-sweeps` | Sweeps to record | 128 |
| `--fc-low`, `--fc-high` | Chirp start and stop in Hz | 24.0e9, 24.25e9 |
| `--firmware` | Path to `SDR_USB_FW.hex` | `firmware/SDR_USB_FW.hex` |
| `--cmd-delay-us` | Pause after each command packet | 15000 |

Samples per sweep are derived, not set directly. The base is 1024 at 0.5 ms with one receiver; it doubles with each sweep-time step and halves each time the receiver count doubles.

### Output files

- **`<out>.bin`**: the raw sample stream as 16-bit little-endian words, I and Q interleaved. The first word of each sweep carries a tag (`0xC000` for Tx1, `0x8000` for Tx2); subtract the tag to get the 12-bit sample. The file holds about 40 sweeps more than requested, as margin.
- **`<out>.json`**: the capture settings (frequencies, bandwidth, sweep time, samples per sweep, Tx/Rx masks), byte and read counts, `sweep_markers_found`, board info and a UTC timestamp.

## Processing captures

In MATLAB, from `processing_src/`:

```matlab
[raw, I, Q, freqAxis, rangeAxis, avgSpectrum, ComplexDataTx1, info] = ...
    process_pi("cap.bin", 1024, 0.5e-3, 2e9, butterworth=true, trim=true);
plot(rangeAxis, avgSpectrum);
```

The three numbers are samples per sweep, sweep time in seconds and bandwidth in Hz. Take them from `samples_per_sweep_per_rx`, `sweep_time_s` and `bandwidth_hz` in the `.json`. `process_pi` handles Tx1 captures with a single receiver. `process_matlab` takes the same four positional arguments for a GUI `.mat` file.

`process_pi_matlab.m` and `process_pi_matlab_integration.m` are comparison scripts with hard-coded file paths at the top. Edit the paths before running them.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `0 sweep markers found`, or changing `--receiver` / `--transmitter` has no effect | The radar MCU drops command words that arrive too close together. Raise `--cmd-delay-us` and capture again. |
| `process_pi` warns about a sweep stride mismatch | The board has two processors. The firmware download resets only the FX2LP; the radar MCU keeps its last configuration. Power-cycle the board and capture again. |
| Capture fails while waiting for re-enumeration | Run `lsusb` and look for the Cypress entry (`04b4:…`). If its product ID is not `8613`, update `post_fw` in `linux/src/main.cpp` and `scripts/99-pupradar.rules`, then rebuild. |
| USB permission or access error | The udev rule is not installed, or the board was plugged in before the rules were reloaded. Replug it. |

Do not "fix" the command pacing by opening and closing the USB device around each command. On Linux, releasing the interface resets it and flushes the command before the MCU reads it. The delay stands in for the per-call overhead that the MATLAB MEX files have naturally. The full explanation is in the comments of `RadarSession::sendCommand`.

## Known limitations

- Sawtooth FMCW only. CW mode exists only in the MATLAB GUI.
- No on-device DSP.
- USB transfers are synchronous, and the whole capture is held in memory before it is written.

## Reference documents

| Document | Content |
|---|---|
| [docs/PUP_EN24C_T2R4_Manual.pdf](docs/PUP_EN24C_T2R4_Manual.pdf) | Operating manual |
| [docs/PUP_EN24C_T2R4_datasheet.pdf](docs/PUP_EN24C_T2R4_datasheet.pdf) | PUP EN24C T2R4 datasheet |
| [docs/PUP_ATN24C_T2R4_datasheet.pdf](docs/PUP_ATN24C_T2R4_datasheet.pdf) | PUP ATN24C T2R4 datasheet |
| [docs/Mex src/PupRadar_Protocol.docx](docs/Mex%20src/PupRadar_Protocol.docx) | USB command protocol |
| [docs/Mex src/PupRadar_DataFormat.docx](docs/Mex%20src/PupRadar_DataFormat.docx) | IQ packet and sweep-header layout |
| [docs/Mex src/CyAPI.pdf](docs/Mex%20src/CyAPI.pdf) | Cypress CyAPI reference used by the MEX sources |

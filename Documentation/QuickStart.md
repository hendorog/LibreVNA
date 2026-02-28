# LibreVNA Quick Start Guide

This guide walks you through installing the LibreVNA-GUI software, connecting to your LibreVNA hardware, and making your first measurement. For advanced features and detailed specifications, refer to the full [user manual](UserManual/manual.pdf).

## Table of Contents

- [1. Installing the GUI Application](#1-installing-the-gui-application)
- [2. Connecting the Hardware](#2-connecting-the-hardware)
- [3. Your First VNA Measurement](#3-your-first-vna-measurement)
- [4. Using the Signal Generator](#4-using-the-signal-generator)
- [5. Using the Spectrum Analyzer](#5-using-the-spectrum-analyzer)
- [6. Saving and Loading Your Work](#6-saving-and-loading-your-work)
- [7. Troubleshooting](#7-troubleshooting)

---

## 1. Installing the GUI Application

### Windows

1. Download the latest release from the [GitHub Releases page](https://github.com/jankae/LibreVNA/releases).
2. Unpack the zip file to a convenient location.
3. Run `LibreVNA-GUI.exe`. No driver installation is required.

### Ubuntu / Linux

1. Download the latest release from the [GitHub Releases page](https://github.com/jankae/LibreVNA/releases) and unpack it.
2. Install the required libraries:
   ```
   sudo apt install qt6-base-dev libqt6svg6
   ```
3. Install the udev rule so your user account can access the USB device:
   ```
   wget https://raw.githubusercontent.com/jankae/LibreVNA/master/Software/PC_Application/51-vna.rules
   sudo cp 51-vna.rules /etc/udev/rules.d
   ```
4. Reload the udev rules (or reboot):
   ```
   sudo udevadm control --reload-rules
   sudo udevadm trigger
   ```
5. Run the application:
   ```
   ./LibreVNA-GUI
   ```

### macOS

1. Download the latest release from the [GitHub Releases page](https://github.com/jankae/LibreVNA/releases).
   - macOS 14+: use the release with `latest` in the name.
   - macOS 13.7: use the release with `13.7` in the name.
2. Unpack the zip and move `LibreVNA-GUI.app` to your `/Applications` folder.
3. On the first launch, macOS will block the app. Open **System Settings > Privacy & Security** and allow the app to run.
4. Launch the app again.

---

## 2. Connecting the Hardware

### Physical Connection

1. Connect your LibreVNA to your computer with a USB cable. The device is powered entirely over USB, so no external power supply is needed.
2. Launch LibreVNA-GUI if it is not already running.

### Connecting in the GUI

1. Go to **Device > Update Device List** to scan for connected devices.
2. Go to **Device > Connect to** and select your device by its serial number.
3. Once connected, the status bar at the bottom of the window will display the device serial number, firmware version, and current status.

**Tip:** To connect automatically on startup, open **Window > Preferences**, and enable **Connect to first device**. The GUI will then connect to the first detected LibreVNA each time it launches.

### Firmware Updates

If the GUI detects that the connected device is running outdated firmware, it will prompt you to update. You can also manually trigger a firmware update via **Device > Firmware Update**. Follow the on-screen instructions and do not disconnect the device during the update.

---

## 3. Your First VNA Measurement

When the GUI opens, it starts in **VNA mode** by default. This is the primary mode for measuring S-parameters (reflection and transmission) of a device under test (DUT).

### Understanding the Default Display

The default view shows:
- **S11** (port 1 reflection) — typically displayed on a Smith chart or as log magnitude.
- **S21** (forward transmission from port 1 to port 2) — typically displayed as log magnitude.

Four S-parameters are available for a 2-port measurement: S11, S21, S12, and S22.

### Setting Up a Sweep

Use the sweep toolbar at the top of the window to configure:

| Parameter | Description | Default |
|-----------|-------------|---------|
| **Start / Stop** | Frequency range for the sweep | 1 MHz – 6 GHz |
| **Center / Span** | Alternative way to set the frequency range | — |
| **Points** | Number of measurement points across the sweep | 501 |
| **IF BW** | IF bandwidth — lower values reduce noise but slow down the sweep | 1 kHz |
| **Level** | Stimulus power at the test port | -10 dBm |
| **Averaging** | Number of sweeps averaged together | 1 |

Click **Full Span** to reset the sweep to the instrument's full range (100 kHz – 6 GHz).

### Running a Measurement

1. Connect your DUT between Port 1 and Port 2 (for a two-port measurement), or to Port 1 only (for a one-port measurement such as antenna impedance).
2. Adjust the frequency range to cover the frequencies of interest.
3. The sweep runs continuously by default. Use the **Run / Stop** button to pause or resume.
4. Use **Single** to capture exactly one sweep and then stop.

### Adding Markers

Right-click on a trace plot and select **Add Marker** to place a marker on the trace. Markers display the exact frequency and measured value at their position. You can drag markers along the trace or use marker search functions (e.g., find peak, find minimum) from the marker context menu.

### Performing a Calibration

Calibration removes systematic errors from the measurement system (cables, connectors, and the instrument itself). For most measurements, a SOLT (Short-Open-Load-Through) calibration is recommended:

1. Go to **Calibration** in the menu bar.
2. Select the calibration type (e.g., **SOLT**).
3. The calibration dialog will guide you through connecting each standard:
   - **Short** to Port 1
   - **Open** to Port 1
   - **Load** (50 Ohm termination) to Port 1
   - **Through** cable between Port 1 and Port 2
   - Repeat Short/Open/Load on Port 2 for a full 2-port calibration
4. For each standard, click **Measure** and wait for the measurement to complete.
5. Once all standards are measured, click **Apply** to activate the calibration.

The calibration status is shown in the status bar. You can save a calibration for later use via **Device > Save Calibration** and reload it with **Device > Load Calibration**.

**Tip:** If you have a LibreCAL electronic calibration module, the process is largely automated. Connect the LibreCAL and follow the on-screen prompts.

---

## 4. Using the Signal Generator

The Signal Generator mode turns the LibreVNA into a CW (continuous wave) signal source.

1. Switch to Signal Generator mode by clicking the **Generator** tab at the top of the window.
2. Set the desired output **Frequency** (100 kHz – 6 GHz).
3. Set the output **Level** (approximately -50 dBm to 0 dBm).
4. Enable the port(s) you want to output on.

The signal generator can also perform a frequency sweep: configure the **Span**, **Steps**, and **Dwell Time** to have the output step through a range of frequencies automatically.

---

## 5. Using the Spectrum Analyzer

The Spectrum Analyzer mode lets you view the frequency spectrum of signals at the input ports.

> **Note:** The LibreVNA is optimized as a VNA, not a dedicated spectrum analyzer. Spectrum analyzer measurements may contain images and aliases. Enable **Signal ID** to reduce these artifacts.

1. Switch to Spectrum Analyzer mode by clicking the **SA** tab.
2. Set the **Start** and **Stop** frequencies (or **Center** and **Span**).
3. Adjust the **RBW** (Resolution Bandwidth) — lower values give finer frequency resolution but slower sweeps.
4. Select the desired **Detector** type (Peak, Average, Sample, or RMS).
5. Connect the signal you want to observe to a port and the spectrum will display automatically.

### Tracking Generator

The spectrum analyzer includes a tracking generator feature. When enabled, one port outputs a stimulus signal that tracks the receiver's sweep frequency. This is useful for measuring filter responses or cable loss without switching to VNA mode:

1. In SA mode, enable the **Tracking Generator** from the toolbar.
2. Select the output port and power level.
3. Connect the DUT between the output and input ports.

---

## 6. Saving and Loading Your Work

### Setup Files

A setup file saves the complete state of the application — all modes, sweep settings, traces, calibrations, and display configuration.

- **File > Save Setup** — save the current state to a `.setup` file.
- **File > Load Setup** — restore a previously saved state.

### Exporting Traces

To export measurement data for use in other tools:

- **File > Export Traces** — save traces in Touchstone (`.s2p`) or other supported formats.

### Importing Traces

You can import previously saved Touchstone files to view and compare against live measurements:

- **File > Import Traces** — load a `.s2p` or other trace file.

### Saving Screenshots

- **File > Save Image** — save a screenshot of the current display as an image file.

---

## 7. Troubleshooting

### Device Not Detected

- **Windows:** No drivers should be needed. Try a different USB port or cable.
- **Linux:** Make sure the udev rule is installed (see [Installation](#ubuntu--linux)) and that you have reloaded the rules or rebooted.
- **macOS:** Ensure you have granted the security exception for the app.
- On all platforms, try **Device > Update Device List** to rescan.

### Status Bar Error Flags

The status bar may show warning indicators:

| Flag | Meaning | What to Do |
|------|---------|------------|
| **ADC Overload** | Input signal is too strong for the ADC | Reduce the signal level at the port or reduce the stimulus power |
| **Unlock** | A PLL has lost frequency lock | This can happen briefly during sweep startup. If persistent, check the reference clock or contact support |

### Sweep Appears Noisy

- Reduce the IF bandwidth (e.g., from 10 kHz to 1 kHz) for a cleaner trace at the cost of slower sweep speed.
- Increase the averaging count.
- Perform a calibration to remove systematic errors.

### Spectrum Analyzer Shows Unexpected Signals

The LibreVNA's spectrum analyzer mode can produce image and alias responses due to the hardware architecture. Enable **Signal ID** in the SA toolbar to filter out most of these artifacts.

### Further Help

- Check the [FAQ](FAQ.md) for common questions.
- Visit the [LibreVNA support group](https://groups.io/g/LibreVNA-support/) for community help.
- Report bugs or request features on [GitHub Issues](https://github.com/jankae/LibreVNA/issues).

# WARDOGS — Helicopter Analog Trigger Fix

An open-source, lightweight, zero-dependency Windows utility that restores smooth **0–255 proportional analog trigger control** (`LT` / `RT`) over helicopter collective in **WARDOGS** — **without** installing kernel drivers like `vJoy` or third-party suites like `Joystick Gremlin`.

---

## Why Does This Issue Happen in WARDOGS?

In recent updates of **WARDOGS**, the helicopter flight model (`BHHeli_Core` / `BHRotaryVehiclePawn`) separates collective input into two internal code paths:

1. **Standard Gamepad / Keyboard Mode (`bUseHOTASHelicopters=False`)**  
   Routes controller trigger inputs through digital button events (`PressCollectiveUp` / `ReleaseCollectiveUp` and `PressCollectiveDown` / `ReleaseCollectiveDown`) to mirror Keyboard & Mouse behavior. As soon as `LT` or `RT` crosses the trigger actuation threshold (~1–2 mm of travel), the game commands **100% collective**, making it impossible to hold 30%, 50%, or 70% collective on an Xbox/XInput controller.
2. **HOTAS Mode (`bUseHOTASHelicopters=True`)**  
   Uses `InputCollectiveAbsolute` via `[/Script/WDJoystick.WDJoystickSettings]`.

Crucially, `BHHeli_Core` passes collective commands through an internal first-order rotor smoothing filter (`LagFilter` / `BHInterpolatedRotaryMovementControlAssistCondition`).

---

## How This Tool Fixes It

### 1. High-Frequency Analog-to-PWM Modulator (Recommended — No vJoy Required)
- **500 Hz XInput Polling:** Reads raw 8-bit trigger values (`bLeftTrigger` and `bRightTrigger`, `0..255`) directly via Windows XInput (`xinput1_4.dll` / `xinput1_3.dll` / `xinput9_1_0.dll`).
- **Response Curve & Deadzone Shaping:** Applies configurable inner/outer deadzones, an exponential response curve (`1.0x` Linear to `3.0x` Fine Hover), and a maximum collective limiter.
- **Proportional Pulse-Width Modulation (PWM):** Modulates hardware-level keyboard scancodes (`SendInput` with `KEYEVENTF_SCANCODE`) at an adjustable carrier frequency (default **28 Hz / 36 ms period**). Because `BHHeli_Core` integrates collective input through its internal rotor `LagFilter`, a **50% trigger pull** generates a **50% duty-cycle pulse train**, holding the helicopter collective smoothly at **~50%**.
- **Patch A (PWM Trigger Guard):** With one click, the tool sets `LeftTriggerMinMultiplier=95.000000` and `RightTriggerMinMultiplier=95.000000` in your `GameUserSettings.ini`. This prevents *Wardogs* from firing the 100% binary trigger spike at 1–2 mm of travel, allowing the PWM modulator to govern the entire **0%–95%** analog range smoothly while **95%–100%** engages full native trigger lock.

### 2. Native DirectInput Z-Axis HOTAS Unlocker (Patch B — Optional Alternative)
- Standard Windows Xbox controller drivers expose combined `LT`/`RT` triggers on **DirectInput Axis 2 (Z-Axis)**.
- Clicking **Apply Patch B** enables `bUseHOTASHelicopters=True` along with the hidden `bCollectiveSelfCentering=True` setting inside `GameUserSettings.ini`.

---

## Quick Start Guide

1. **Launch `Wardogs_Heli_Trigger_Fix.exe`** (no installation or administrator rights required).
2. **Verify Config Path:**  
   In Section `03 // WARDOGS CONFIGURATION PATCHER`, the tool automatically detects your `%LOCALAPPDATA%\Wardogs\Saved\Config\WindowsClient\GameUserSettings.ini` on any Windows PC. If your config is in a custom location, click **`BROWSE...`** to select it.
3. **Apply Patch A (Recommended):**  
   Click **`APPLY PATCH A: PWM TRIGGER GUARD`** while *Wardogs* is closed (or before entering a match). The tool automatically creates a clean backup (`GameUserSettings.ini.heli_backup`) that you can restore anytime with **`RESTORE ORIGINAL INI BACKUP`**.
   *(Alternatively, you can simply unbind `LT`/`RT` from Helicopter Collective Up/Down inside the in-game controller settings so only the PWM keys control collective).*
4. **Match Your Collective Keys:**  
   In Section `02`, verify that **Collective Up Key (RT)** (default: `SPACEBAR`) and **Collective Down Key (LT)** (default: `LEFT CTRL`) match your in-game keyboard bindings for Helicopter Collective Up / Down.
5. **Fly & Toggle Anytime In-Game:**
   - Press **`F8`** on your keyboard or **`LB + RB + D-Pad UP`** on your controller to toggle the **Heli-PWM Modulator** ON or OFF (with an audible high/low confirmation beep) when switching between helicopters, ground vehicles, and infantry.

---

## Flight Presets Included

| Preset | PWM Carrier | Expo Curve | Inner Deadzone | Max Authority | Best For |
| :--- | :---: | :---: | :---: | :---: | :--- |
| **UH-72 Lakota (Gentle Hover)** | `28 Hz` | `1.80x` | `4% (10/255)` | `90%` | Smooth, predictable transport flying without upsetting the Lakota |
| **AH-6 Littlebird (Agile)** | `32 Hz` | `1.50x` | `2% (6/255)` | `100%` | Fast collective response with fine low-end modulation |
| **1:1 Linear Direct** | `28 Hz` | `1.00x` | `2% (5/255)` | `100%` | Unmodified linear 0–100% trigger-to-collective mapping |

---

## Building from Source (No Visual Studio Required)

This project is written in standard, dependency-free C# and compiles using the native Microsoft `.NET Framework` compiler (`csc.exe`) pre-installed on every Windows 10 and Windows 11 PC.

1. Double-click **`build.bat`**, or run the following command in PowerShell / Command Prompt:
   ```bat
   "%WINDIR%\Microsoft.NET\Framework64\v4.0.30319\csc.exe" /target:winexe /optimize+ /out:Wardogs_Heli_Trigger_Fix.exe /r:System.dll /r:System.Drawing.dll /r:System.Windows.Forms.dll src\WardogsHeliTriggerFix.cs
   ```
2. The standalone `Wardogs_Heli_Trigger_Fix.exe` binary (~32 KB) will be generated in the root folder.

---

## Repository Structure

```text
Wardogs-Heli-Trigger-Fix/
├── Wardogs_Heli_Trigger_Fix.exe   # Pre-compiled standalone Windows executable
├── build.bat                      # 1-click compiler script using built-in Windows csc.exe
├── README.md                      # Documentation & setup guide
├── LICENSE                        # MIT Open Source License
└── src/
    └── WardogsHeliTriggerFix.cs   # Full, documented C# source code
```

---

## License

Released under the [MIT License](LICENSE). Free to use, modify, audit, and share with the community.

* **VirusTotal Scan (`Wardogs_Heli_Trigger_Fix.exe`):** [View VirusTotal Report](https://www.virustotal.com/gui/file/b345cae2409db9207e6e60f147c18c7a7784ba64bc07575e54f0028d4dad6bbe) *(Clean across Microsoft Defender, Malwarebytes, BitDefender, Kaspersky, ESET, CrowdStrike, Avast, AVG, etc. Note: Like any unsigned indie input tool using `SendInput` / `GetAsyncKeyState` for hotkeys, a few generic heuristic engines may flag unsigned binaries — if preferred, simply inspect `src/WardogsHeliTriggerFix.cs` and run `build.bat` to compile the executable yourself locally).*
* **Fair Play & Anti-Cheat:** This utility does **not** hook into the game process, inject DLLs, or read/write game memory. It strictly uses standard Windows `XInputGetState` polling, standard user-mode keyboard inputs (`SendInput`), and standard `GameUserSettings.ini` settings — running completely cleanly without issues in testing.


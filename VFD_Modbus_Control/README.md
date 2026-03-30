# modbus
Instructions on how to implement modbus for a H100 VFD with UCCNC

# background

I found implementing modbus for a H100 VFD with UCCNC to be a bit of a pain. The H100 VFD has a complex modbus implementation, and UCCNC's modbus support is great but maybe non-intuitive. Additionally, the documentation for the H100 VFD's modbus implementation is sparse and tricky to understand.

Full disclosure, I used Claude Sonnet 4.6 to help write this guide and figure out how to implement modbus for a H100 VFD with UCCNC, which was very helpful.

# solution

To implement modbus for a H100 VFD with UCCNC, you will need to configure the VFD to use modbus and then configure UCCNC to communicate with the VFD using modbus. You will also need to ensure that the modbus settings on both the VFD and UCCNC are correct and that the modbus communication is working properly. Let's get started!

# 1. Assumptions

This guide was written using the following version/models/hardware:

VFD Model: [H100-2.2S2-1B](/H100_VFD_Manual.pdf)

UCCNC Version: [1.2115](/UCCNC_usersmanual.pdf)

USB to RS485 Converter: [Waveshare SKU 17286](https://www.waveshare.com/wiki/USB_TO_RS485)

Some wires

a 5V relay that has both NC, NO and a low level trigger

a 1N4007 diode

## 2. Wiring

Wire the components to each other as shown in [this wiring schematic](VFD_Modbus_Wiring_Schematic.pdf).

---

## 3. H100 VFD Parameter Settings

Access the VFD keypad and configure these parameters:

| Parameter | Description          | Required Value                  |
|-----------|----------------------|---------------------------------|
| **F001**  | Command source       | `2` (Communication port)        |
| **F002**  | Frequency source     | `2` (Communication port)        |
| **F044**  | X1 Input Behavior    | `13` (emergency cut-off signal) |
| **F163**  | Modbus slave address | `1` (default, must match UCCNC) |
| **F164**  | Modbus baud rate     | `1` (9600 bps recommended)      |
| **F165**  | Data format          | `3` (8N1 for RTU)               |


---

## 4. UCCNC Modbus Master Plugin Setup

### 4.1 Enable the Plugin

1. Open UCCNC and go to **Settings**
2. Click **"Configure plugins"**
3. Enable the **Modbus master plugin**
4. Close and **restart UCCNC**

### 4.2 Create a Connection

1. Open UCCNC and go to **Settings**
2. Click **"Configure plugins"**
3. Click the **Show** button for the Modbus master plugin
4. In the Modbus master plugin window, click **"Add Connection"**
5. Click `00` in the tree view to select it
6. Click the **"Connection settings"** tab
7. Enter "VFD" in the description field
8. Select **"Serial RTU"** as the connection type
9. Configure the serial parameters:

| Field             | Value                       |
|-------------------|-----------------------------|
| Serial port       | Your COM port (e.g. `COM3`) |
| Baud rate         | `9600`                      |
| Data bits         | `8`                         |
| Parity            | `Non`                       |
| Stop bits         | `1`                         |
| Retry count       | `3`                         |
| Timeout(ms)       | `1000`                      |
| Loop interval(ms) | `100`                       |
| RS485 mode        | ☑ **Checked**               |

The COM port you select must be the COM port that the USB to RS485 adapter is connected to.

> Don't forget to check the RS485 checkbox — it's easy to miss.



### 4.3 Add Functions

> **Important note on register addresses:** The H100 uses **standard Modbus RTU** with its own register map. The `0x2000`/`0x2001` addresses commonly found online are for **older Huanyang GT/YL series drives** and will **not work** with the H100 — using them will cause a `Modbus.SlaveException`.

With your connection selected, click **"Add Function"** for each of the following. Click each function in the tree, then go to the **"Function settings"** tab to configure it.

#### Function 1 — Write Control Word (Run/Stop/Direction)

#### These values can be found in the [H100's manual](/H100_VFD_Manual.pdf) on page 85. The addresses are in hex in the manual, and so must be converted to decimal values when setting up the functions in UCCNC. I don't know how one would be able to understand page 85 without the help of someone who already knows the answer; I used Claude Sonnet 4.6 to help me figure it out.

| Field                 | Value                  |
|-----------------------|------------------------|
| Function name         | `Spindle Control`      |
| Function type         | `Write SingleRegister` |
| Slave address         | `1`                    |
| Modbus start register | `512` (= 0x0200)       |
| Register count        | `1`                    |
| UCCNC start register  | `0`                    |

#### Function 2 — Write Frequency Setpoint

| Field                 | Value                  |
|-----------------------|------------------------|
| Function name         | `Spindle Speed`        |
| Function type         | `Write SingleRegister` |
| Slave address         | `1`                    |
| Modbus start register | `513` (= 0x0201)       |
| Register count        | `1`                    |
| UCCNC start register  | `1`                    |

#### Function 3 — Read Output Frequency (feedback)

| Field                 | Value                                                                  |
|-----------------------|------------------------------------------------------------------------|
| Function name         | `Read Frequency`                                                       |
| Function type         | `Read InputRegisters` *(must be InputRegisters, not HoldingRegisters)* |
| Slave address         | `1`                                                                    |
| Modbus start register | `0` (= 0x0000)                                                         |
| Register count        | `1`                                                                    |
| UCCNC start register  | `2`                                                                    |

### 4.4 Start the Loop

Click **"Start loops"** — the button turns red when active. The loop also starts automatically on UCCNC startup at startup.

### 4.5 Verify with Debug View

- **Debug view tab** — functions show green (working) or red (error)
- **Variables table tab** — shows live values in UCCNC's internal register slots

---

## 5. H100 Modbus Register Reference

### Control Word Register (0x0200 = decimal 512)

Write using Modbus function `06H` (Write SingleRegister).

| Value | Binary      | Command                             |
|-------|-------------|-------------------------------------|
| `3`   | `0000 0011` | Forward run (enable + forward bits) |
| `5`   | `0000 0101` | Reverse run (enable + reverse bits) |
| `0`   | `0000 0000` | Stop (clear all bits)               |

> Note: Using `8` (stop bit only) does **not** reliably stop the H100. Always use `0` to stop.

### Frequency Register (0x0201 = decimal 513)

Write using Modbus function `06H`. The H100 expects frequency in **0.1 Hz units**:

| Target Frequency | Value to Send |
|------------------|---------------|
| 400.0 Hz (max)   | `4000`        |
| 200.0 Hz         | `2000`        |
| 100.0 Hz         | `1000`        |

> **Important:** The unit is 0.1 Hz, so `maxHz = 4000` for a 400 Hz spindle. Using `40000` is incorrect and will result in the wrong frequency being sent.

### Input Register Address Table (read with function `04H`)

| Address (Hex) | Decimal | Purpose            |
|---------------|---------|--------------------|
| `0x0000`      | 0       | Output frequency   |
| `0x0001`      | 1       | Set frequency      |
| `0x0002`      | 2       | Output current     |
| `0x0003`      | 3       | Output speed (RPM) |
| `0x0004`      | 4       | DC bus voltage     |

---

## 6. UCCNC Macros

In order for the S-words and M3, M4, and M5 commands to work properly, new M3, M4, and M5 macros must be used.

In UCCNC, macros for M3, M4, and M5 are plain text `.txt` files placed in:
```
C:\UCCNC\Profiles\[YourProfileName]\Macros\
```

Drag the M3.txt, M4.txt, and M5.txt files into this repo into the above folder and replace the existing ones.

## 7. Troubleshooting

| Problem                                  | Likely Cause                            | Fix                                                                     |
|------------------------------------------|-----------------------------------------|-------------------------------------------------------------------------|
| `Modbus.SlaveException`                  | Wrong register address                  | Use `0x0200`/`0x0201`, not `0x2000`/`0x2001`                            |
| SlaveException on reads                  | Wrong function code                     | Use `Read InputRegisters` (04H) for status, not `Read HoldingRegisters` |
| Spindle doesn't respond to Modbus at all | F001/F002 not set to communication mode | Set F001=2, F002=2                                                      |
| Spindle runs but ignores speed           | Wrong frequency scaling                 | Use `maxHz = 4000` not `40000`                                          |
| Spindle always runs at minimum frequency | S-word variable returning 0             | Use `AS3.Getfielddouble(fieldID)` instead of `exec.Getvar(2000)`        |
| Stop command not working                 | Wrong control word value                | Use `0` to stop, not `8`                                                |
| M3 doesn't work after M5                 | VFD not ready after stop                | Add `Wait(2000)` and clear control word to `0` at start of M3           |
| Communication lost after restart         | USB adapter assigned new COM port       | Check Device Manager and update COM port in plugin settings             |
| Noise/random faults                      | Poor wiring                             | Add 120Ω termination resistor, use shielded cable                       |

---

## 8. Key Lessons Learned

1. **The H100 is NOT the same as older Huanyang drives.** The `0x2000`/`0x2001` register addresses found all over the internet are for GT/YL series drives. The H100 uses standard Modbus RTU with registers starting at `0x0200`.

2. **Function code matters.** Reading status registers on the H100 requires `Read InputRegisters` (function code `04H`). Using `Read HoldingRegisters` (03H) will cause a SlaveException even if the register address is correct.

3. **Frequency unit is 0.1 Hz, not 0.01 Hz.** `maxHz = 4000` for a 400 Hz max spindle, not `40000`.

4. **Stop by clearing to 0.** Writing `0` to the control word is the reliable way to stop the H100. The dedicated stop bit (`8`) does not work reliably.

5. **The Modbus loop must be running** before macros will work. Macros write to UCCNC's internal register table; the plugin loop pushes those values to the VFD.

6. **Each UCCNC profile has its own macros folder.** Launching the wrong profile means none of your macros or plugin configuration will be present.

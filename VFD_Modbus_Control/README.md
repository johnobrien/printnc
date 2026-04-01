# modbus
Instructions on how to implement modbus for a H100 VFD with UCCNC

# disclaimer
⚠️ WARNING & DISCLAIMER
The information, designs, code, and instructions in this repository are provided for educational and informational purposes only. Building a CNC machine involves working with high-voltage electronics, high-speed rotating parts, and powerful motors that can cause serious injury, property damage, or death if handled improperly.
By using any content from this repository, you acknowledge that:

You assume full responsibility for your own safety and the safety of others.
The author(s) of this repository are not liable for any injury, damage, or loss resulting from the use or misuse of this information.
This repository makes no guarantees regarding the accuracy, completeness, or fitness of any design or instruction for any particular purpose.

Always follow local electrical codes, safety regulations, and best practices. If you are unsure about any step, consult a qualified professional before proceeding.

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

a 5 V relay that has both NC, NO and a low-level trigger

a 1N4007 diode

## 2. Wiring

Wire the components to each other as shown in [this wiring schematic](VFD_Modbus_Wiring_Schematic.pdf).

One quirky aspect of the wirint to note is you want the wire the Output 2 (DB 25 Pin 1) to the relay DC- **and** IN pins.
The reason why is because when DB 25 Pin 1 goes high on the UC300ETH, the Output 2 terminal on the back of the G540
closes to G540 PSU Ground. With a low-pass trigger, this opens the relay, which breaks the connection between X1 and 
GND on the VFD, allowing the spindle to run.

If you don't wire Output 2 to both the IN pin and DC - pin, when Output 2 goes to ground when the G540 is not powered, 
the relay will open and the VFD will run even if there is no command from UCCNC.

This is not what you want, so connect both the IN pin and DC - pin to the Output 2 terminal on the back of the G540.

---

## 3. H100 VFD Parameter Settings

Access the VFD keypad and configure these parameters:

| Parameter | Description                    | Required Value                                 |
|-----------|--------------------------------|------------------------------------------------|
| **F001**  | Command source                 | `2` (Communication port)                       |
| **F002**  | Frequency source               | `2` (Communication port)                       |
| **F003**  | Main Frequency                 | Match the value for your spindle               |
| **F004**  | Reference Frequency            | Match the value for your spindle               |
| **F005**  | Reference Limit                | Match the value for your spindle               |
| **F044**  | X1 Input Behavior              | `13` (emergency cut-off signal)                |
| **F143**  | Number of poles for your motor | Match the value for your spindle               |
| **F144**  | Max RPM                        | Max RPM for your motor (see manual for syntax) |
| **F163**  | Modbus slave address           | `1` (default, must match UCCNC)                |
| **F164**  | Modbus baud rate               | `1` (9600 bps recommended)                     |
| **F165**  | Data format                    | `3` (8N1 for RTU)                              |

Thanks to https://www.youtube.com/watch?v=D610-kN4-xc for help with the VFD parameters.

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


## 5. UCCNC Macros

In order for the S-words and M3, M4 and M5 commands to work properly, new M3, M4 and M5 macros must be used.

In UCCNC, macros for M3, M4 and M5 are plain text `.txt` files placed in:
```
C:\UCCNC\Profiles\[YourProfileName]\Macros\
```

Drag the M3.txt, M4.txt and M5.txt files into this repo into the above folder and replace the existing ones.

## 7. Done

You should now be able to use the M3, M4, and M5 commands in UCCNC to control the VFD.
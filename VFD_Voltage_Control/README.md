# modbus
Instructions on how to implement voltage control for a H100 VFD with UCCNC

# disclaimer
⚠️ WARNING & DISCLAIMER
The information, designs, code, and instructions in this repository are provided for educational and informational purposes only. Building a CNC machine involves working with high-voltage electronics, high-speed rotating parts, and powerful motors that can cause serious injury, property damage, or death if handled improperly.
By using any content from this repository, you acknowledge that:

You assume full responsibility for your own safety and the safety of others.
The author(s) of this repository are not liable for any injury, damage, or loss resulting from the use or misuse of this information.
This repository makes no guarantees regarding the accuracy, completeness, or fitness of any design or instruction for any particular purpose.

Always follow local electrical codes, safety regulations, and best practices. If you are unsure about any step, consult a qualified professional before proceeding.

# background


Full disclosure, I used Claude Sonnet 4.6 to help write this guide and figure out how to implement modbus for a H100 VFD with UCCNC, which was very helpful.

# solution

To implement voltage control for a H100 VFD with UCCNC, you will need to configure the VFD to use voltage control and then configure UCCNC to communicate with the VFD by sending a PWM and direction signal to the VFD through the G540. Let's get started!

# 1. Assumptions

This guide was written using the following version/models/hardware:

VFD Model: [H100-2.2S2-1B](/H100_VFD_Manual.pdf)

UCCNC Version: [1.2115](/UCCNC_usersmanual.pdf)

Some wires

a 5 V relay that has both NC, NO and a low-level trigger

a 1N4007 diode

## 2. Wiring

Wire the components to each other as shown in [this wiring schematic](VFD_Voltage_Control_Wiring_Schematic.pdf).

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
| **F001**  | Command source                 | `1` (External terminal)                        |
| **F002**  | Frequency source               | `1` (AI1)                                      |
| **F003**  | Main Frequency                 | Match the value for your spindle               |
| **F004**  | Reference Frequency            | Match the value for your spindle               |
| **F005**  | Reference Limit                | Match the value for your spindle               |
| **F044**  | X1 Input Behavior              | `2` (forward))                                 |
| **F045**  | X2 Input Behavior              | `3` (reverse)                                  |
| **F143**  | Number of poles for your motor | Match the value for your spindle               |
| **F144**  | Max RPM                        | Max RPM for your motor (see manual for syntax) |


Thanks to https://www.youtube.com/watch?v=D610-kN4-xc for help with the VFD parameters.

---

## 4. UCCNC Configuration

1. Launch UCCNC, navigate to Settings->Spindle.
2. Check the checkbox next to "PWM Spindle" if it is not already checked.
3. Set the PWM pin to 14 port 2.
4. Check the checkbox next to "Spindle Relay Output Enabled" if it is not already checked.
5. Set the M3 Relay pin to pin 1 port 2.
6. Set the M4 Relay pin to pin 17 port 2.
7. Click "Save" to save your changes.

## 5. Done

You should now be able to use the M3, M4, and M5 commands in UCCNC to control the VFD.
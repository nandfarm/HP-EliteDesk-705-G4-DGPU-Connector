# HP-EliteDesk-705-G4-DGPU-Connector
Exploring the pinout of the HP EliteDesk 705 G4 DGPU Connector

# Purpose and story:
This is mostly an exploration journal. The purpose of the whole endeavor is to maybe build a riser for this series of USFF PCs and take advantage of that 8 lane PCIe 3 connector. 

I came across a couple of these battered up HP EliteDesk 705 G4s at a recycler near where I lived and after a serious barter talk (I accepted the first price he gave me), I bought them home. Unfortunately none of them worked. All had the dreaded HP Beep Code 3.4 so I went down the rabbit whole and managed to salvage 3 out of 4 of them. The last one bravely donated parts to his companions. He will be remembered...

If you've cracked open an HP EliteDesk (or similar 1-liter mini PC) and found a proprietary graphics card connector staring back at you, you've probably wondered: "Can I run standard PCIe devices off this?"

The answer is yes. But HP doesn't publish the pinouts, and making a mistake here means ending the new life of another PC...

This guide documents the process used to reverse-engineer the proprietary 80-pin dGPU connector on the HP EliteDesk motherboard and to *hopefully* to design a custom standard PCIe / dual M.2 NVMe riser.

# Philosophy:

We are dealing with a mix of raw power, high-speed differential pairs, and proprietary OEM telemetry. To map this without frying the board, the process is split into three phases: Unpowered mapping (Continuity/Diode mode), Powered logic mapping (Voltage mode), and Active Signal mapping (Oscilloscope).

# Phase 1: Unpowered Mapping (Data Lanes & Ground)

Before you even think about plugging the motherboard in, you need to map the high-speed TX/RX data lanes directly to the CPU socket. Some of them are easy, some are not... 
Before relying on any pinout map you found online, you need to verify its orientation.

1. Put your multimeter in continuity mode.

2. Place one probe on a solid motherboard ground (like a USB port shield).

3. Probe the CPU socket for a known VSS (Ground) pin.

*The Mirror Trap:* Most AM4/LGA socket diagrams show the bottom of the CPU, meaning the socket on the motherboard is mirrored horizontally. If your VSS pins aren't beeping to ground, flip your reference image horizontally.

TODO: Add Photo of CPU pin map

4. Once you are oriented, trace the PCIe TX and RX lanes from the CPU socket directly to the proprietary connector.

TODO: Add Photo of the probe in the socket... *wink*

# Phase 2: Power and the "Dummy" Test

Now we need to find the raw power delivery. Plug the motherboard in, but do not use a bare wire to test anything.

1. Switch a multimeter to Voltage mode.
   
2. HP does not send a 12V rail to the connector but the main 19V rail. This needs to be converted down for anything usefull. The 19V/20V rail and the 3.3V logic run power are active immediately when the board turns on. They are easy to map out. 
*Note on WAKE#:* You won't find 3.3V auxiliary standby power (+3.3Vaux) while the board is off. Because a GPU has no reason to wake the PC from sleep, HP completely omitted standby power from this slot.

3. To find the logic pins (like PRSNT# or CLKREQ#), you need to pull pins to ground to see how the BIOS reacts. Never use a bare wire. If you accidentally hit a 3.3V power plane, you will instantly cause a dead short and fry the regulator. Solder a micro-probe to a 1kΩ resistor, and tie the other end to ground. This acts as a safe pull-down. If you hit a logic pin, it safely pulls to 0V. If you hit a power plane, the resistor absorbs the load.
TODO: Add Photo of the probe

# Phase 3: Active Signals & The Oscilloscope

This is where the magic happens. We need to find the 100MHz reference clock and the PCIe sideband signals.

1. Probe the remaining differential pairs with an oscilloscope looking for a continuous 100MHz square-ish wave.

2. PERST# is an active-low reset signal. Set your scope to trigger on a falling edge at ~1.5V. During boot (POST), you will see this pin resting at 3.3V, but it will rapidly dip to 0V multiple times as the BIOS tries to initialize the missing GPU.

3. HP uses the System Management Bus (I2C) to read thermal data and hardware IDs from the proprietary GPU. On the scope, these pins sit at 3.3V but will show distinct bursts of digital "chatter" (rapid square-wave dropouts) during POST as the BIOS blindly queries the empty slot. Zoom in to around 10µs/div: the Clock (SCL) will be a uniform square wave, while Data (SDA) will be erratic.
TODO: Add Photo of the I2C lines

# ⚠️ DANGER: There be crowbar pins...
  
During your probing, you may hit a 3.3V pin that instantly kills the system and triggers a 4-3 BIOS Beep Code (Thermal/Power emergency). This is a proprietary hardware protection pin (likely PROCHOT# or a VRM feedback loop). Even the tiny 1MΩ capacitance of an oscilloscope probe is enough to drag the voltage down, tricking the motherboard into thinking the non-existent GPU is on fire. It instantly crowbars the power supply to save the system. I marked this pin as NC (No Connect) and left it floating.

# Bifurcation (Splitting the x8 to x4x4)

It appears that there is a way to bifurcate the x8 connection int two x4s only from the software/firmware. This is under investigation still. TBD

Stay Tuned...

























# Photon Counter

## What this Section Covers:

- How to implemenent a simple pulse counter with a programmable dead-time and integration window that will output the windowed count as a 14-bit DAC voltage that can be seen on an oscilloscope using the e1 digital pins. 
- How to transalte your 14-bit DAC voltage into number of counts
- Making your own Custom Code/ IP block

## Background

### LVTTL Pulses in Photon Counting
- LVTTL stands for Low-Voltage Transistor-Transistor Logic - a digital logic standard that guarantees an output at around 2.4-3.3 V for the logic high level and 0-0.4 V for logic low level. [^1]
- In LVTTL, an incoming signal is interpreted as logic HIGH (1) if its voltage is between 2.0-3.3 V and logic LOW (0) if it is 0-0.8 V
- An LVTTL pulse is a short transient signal that rises from 0V to approximately 3.3V and back to 0V. Each pulse represents the occurence of a discrete event e.g. the detection of a photon by the single-photon detector.

Because we only care about when a pulse occurs and not its precise voltage, we use a digital input pin like the E1 pin instead of the ADC. The E1 pins directly register these 0/1 transitions, which simplifies the counting logic and avoids unnecessary analogue conversion.

![LVTTL](/images/photon_counter/LVTTL.svg)
**Figure 1:** Illustration of LVTTL pulses representing discrete events *(Note: LVTTL pulses don't have to be periodic)*

### Programmable Dead time and Integration Window.

To measure photon detection events, the counter is configured to count each LVTTL pulse during a defined integration window. However, because the FPGA operates at a fixed clock frequency (125 MHz corresponding to 8 ns per clock cycle), a single pulse that reamins high for several clock periods could be counted multiple times.

- **Integration Window:**
    - Counting is enabled for a fixed duration - the integration window.
    - When the window ends, the total count is output and the counter resets before the next window begins.
    - This allows measurement of the photon count rates over specific time intervals.

- **Problem: Multiple Counts per Pulse:**
    - The FPGA samples inputs on each rising clock edge.
    - If an LVTTL pulse remains high for longer than one clock period, multiple rising edges may detect it as still high, resulting in false multiple counts.
    - This issue is illustrated by the red dashed lines in Figure 2, where a single input pulse spans 2 clock edges.
   
- **Dead time**:
    - To prevent multiple counts from the same pulse, a dead time is introduced immediately after each detected rising edge.
    - During this programmable interval, the counter stops counting even if the input remains high.
    - Once the dead time has elapsed, the counter will register the next valid rising edge.
    - This ensures that each physical photon detection event corresponds to exactly one digital count.  

![dead time](/images/photon_counter/dead_time.svg)
**Figure 2:** The cyan waveform represents the incoming LVTTL pulses, while the white dashed waveform indicates the 125 MHz FPGA clock. The red dashed lines illustrates a case where two rising edges of the clock occur within a single LVTTL pulse. Without applying a dead time, both clock edges would be registered as seperate events, causing the counter to double-count one photon pulse.

## References:

[^1]: Euresys. TTL and LVTTL Logic Levels, Coaxlink 10.3 Documentation. Available at: https://documentation.euresys.com/Products/Coaxlink/Coaxlink_10_3/Content/03_Using_Coaxlink/application-notes/connecting-ttl-to-isolated-ports/TTL_and_LVTTL_levels.htm


# Pulse Counter

## What this Section Covers:

- **Pulse Counting Logic:**
    - Introduction to LVTTL pulse signals and their characteristics
    - Detecting and counting LVTTL pulses (e.g., photon or TTL events) using FPGA logic
    - Understanding integration windows and dead time to prevent multiple counts from a single pulse
    - Interfacing the counter with existing IP blocks such as the DAC, Clocking Wizard, and GPIO. 

- **LED Debug:**
    - Using the on-board LEDs to verify that pulses are being received correctly
    - Understanding the underlying LED circuit and signal path
    - Exploring floating inputs and the role of pull-up and pull-down resistors

- **Counter code:**
    - Line-by-line explanation of the Verilog counter module
    - How the counter output is packaged and transmitted to the DAC over a 32-bit AXI4-Stream bus
    - Understanding bit padding and channel mapping for OUT1 and OUT2
    - Converting digital counts into corresponding output voltages for oscilloscope observation

- **Tutorial:**
    - Step-by-step Vivado implementation and block design walkthrough

- **Testing:**
    - Verifying system functionality using a signal generator and oscilloscope
    - Guidance on impedance matching, configuring a square-wave source to emulate LVTTL pulses, and adjusting duty cycles
    - What to expect when observing output waveforms on the oscilloscope 

- **Further Steps:**
    - Reviewing and validating the entire signal chain
    - Implementing a two-flip-flop synchroniser and rising-edge detector
    - Introducing PID control as a next step toward closed-loop feedback systems

- **I would recommend going over the following projects beforehand:**
    - [LED blink](/projects/led_blink.md)
    - [LED Control (GPIO)](/projects/led_control_gpio.md)
    - [Signal Passthrough (ADC TO DAC)](/projects/signal_passthrough_adc_dac.md)

- **Quick Links:**
    - [Tutorial](/projects/pulse_counter.md#tutorial)
    - [Testing](/projects/pulse_counter.md#testing)
    - [Further Steps](/projects/pulse_counter.md#further-steps)
    - [References](/projects/pulse_counter.md#references)
    
## Background

### LVTTL Pulses in Photon Counting
- LVTTL stands for Low-Voltage Transistor-Transistor Logic - a digital logic standard that guarantees an output at around 2.4-3.3 V for the logic high level and 0-0.4 V for logic low level. [^1]
- In LVTTL, an incoming signal is interpreted as logic HIGH (1) if its voltage is between 2.0-3.3 V and logic LOW (0) if it is 0-0.8 V
- An LVTTL pulse is a short transient signal that rises from 0V to approximately 3.3V and back to 0V. Each pulse represents the occurence of a discrete event e.g. the detection of a photon by the single-photon detector.

Because we only care about when a pulse occurs and not its precise voltage, we use a digital input pin like the E1 pin instead of the ADC. The E1 pins directly register these 0/1 transitions, which simplifies the counting logic and avoids unnecessary analogue conversion.

![LVTTL](/images/photon_counter/LVTTL.svg)
**Figure 1:** Illustration of LVTTL pulses representing discrete events *(Note: LVTTL pulses don't have to be periodic)*

### Programmable Dead time and Integration Window.

To measure photon detection events, the counter is configured to count each LVTTL pulse during a defined integration window. However, because the FPGA operates at a fixed clock frequency (125 MHz corresponding to 8 ns per clock cycle), a single pulse that remains high for several clock periods could be counted multiple times as shown in Figure 2.

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

### LED Debug Connection and Pull Configuration

- **How to know E1 pin is receiving a signal**
    - To verify that the E1 pin is receiving a signal, we will connect it to an LED output in the block design. This allows us to visually confirm when the FPGA detects a signal. When a pulse arrives, the LED will light up; when there is no pulse, it will turn off.
    - If you input a high-frequency square wave (e.g. 1 KHz or 1 MHz) to the E1 pin, the LED will appear continuously lit because it is flashing faster than the human eye can percieve. 
    - If you input a low-frequency square wave (e.g. 1 Hz), you will see the LED visibly blink on and off at each pulse.
    - This set up provides a quick hardware check that your E1 pins and signal path are functioning before integrating the counter logic. 

- **Understanding the LED Circuit**
    - From Figure 3, you can see that current flows from the anode to the cathode of the LED. From Figure 4, we can see in the Red Pitaya, the cathode is tied to the ground  (GND), while its anode connects through a resistor to the FPGA pin.
    - When the FPGA drives the pin high, current flows.
    - When the FPGA drives the pin low (0 V), both ends of the LED are at the same potential, so no current flows and the LED remains off.

![diode](/images/photon_counter/diode.svg)
**Figure 3:** Annotated diode circuit symbol showing anode (+) and cathode (-), with conventional current flowing from the anode to the cathode.

![led schematic](/images/photon_counter/led_schematic.png)

**Figure 4:** LED in Red Pitaya schematic[^2]

- **Floating Inputs and Pull-Up/Pull-Down Resistors**
    - Digital input pins like those on the E1 header can sometimes be floating when no signal source is connected - meaning they are not at a defined voltage level (neither 0 V or 3.3 V). This can lead to issues.
    - To prevent unclear voltage states, pull-up or pull-down resistors are used internally.
    - A pull-up resistor connects the input to read as logic high (1) when not actively driven low.
    - A pull-down resistor connects the input to read as logic low (0) when not actively driven high.

    - From Figure 5, we can see that exp_p_tri_io pins have a pull-up configuration meaning they default to high unless an external signal drives them low. 
    - When I had connected `exp_p_tri_io[7:0]` pin to `led[7:0]`, all the pins except the one connected to the source signal (`exp_p_tri_io[0:0]`) were lit up and this is because of this pull-up configuration. This is shown in Figure 6. Since I wanted the LED to just mirror the digital input, I altered the design to `exp_p_tri_io[7:0]` -> `Slice` -> `led_o[0:0]`

![implementation_e1_properties](/images/photon_counter/implemetation_e1_properties.png)
**Figure 5:** Once you have run implementation, you can go the the Flow Navigator, expand Open Implemented Design, and click on Schematic. In the tool bar, you will see X Cells, Y I/O Ports, and Z Nets. Click on the Y I/O Ports and search for `exp_p_tri_io`.

![pull-up behaviour in led connected to e1 pins](/images/photon_counter/pull_up_making_led_1_to_7_light_up.JPEG)
**Figure 6:** Pull-up behaviour by directly connecting `exp_p_tri_io[7:0]` to `led[7:0]` in the block design but with only `exp_p_tri_io[0:0]` connected to the signal generator on the red pitaya.

### Counter Code

- I have annotated the counter code with some explanations. 
- A discussion on the AXI4-Stream data which I haven't annotated will follow as I feel it requires a deeper exploration.

```verilog
module counter(
    
// Module Ports
input [31:0] window_length,                // A 32-bit value specifying how many clock cycles make up one integration window

input e1,                                  // The input pulse signal you want to count

input clk,                                 // The system clock; the counter increments on the rising edge of this clock

input [31:0] val,                          // A 32-bit value defining the dead-time, i.e. how many clock cycles must pass
                                           // before another pulse on e1 is counted.

// AXI4-Stream data out  
output [31:0] M_AXIS_tdata,
output M_AXIS_tvalid,

// AXI4-Stream data in
input [31:0] S_AXIS_tdata,
input S_AXIS_tvalid,
output S_AXIS_tready


    );
// Internal Registers 
reg e1_count1 = 0;                       // Flag to show whether the module is in dead-time or not. 
                                         // When 1, module ignores further pulses until the dead-time counter expires.              

reg [13:0] sum =0;                       // Stores the number of valid pulses detected within the current integration window.


reg [31:0] store_v =0;                   // Used to implement dead-time. 
                                         // Counts cycles while e1_count1 = 1
                                         // and resets when the configured dead-time (val) expires.

reg [31:0] store_w =0;                   // Tracks the number of clock cycles elapsed in the current integration window. 
                                         // When it exceeds the window_length, 
                                         // accumulated sum is captured and the counter resets.


reg [13:0] sum_hold =0;                  // Holds the pulse count from the most recently completed window. 
                                         // This value is packaged into M_AXIS_tdata on the next clock cycle.

always @(posedge clk) begin

// Window Counter
    if (store_w > window_length) begin
        sum_hold <=sum;                  // Stores sum at the end of each window so that it can be output,
        store_w <= 0;                    // while the counter resets for the next window. 
        sum <=0;                         // Initialise the store_w value to 0 after a complete integration window 
    end else begin                       // Clear sum at end of window
        store_w <= store_w + 1;          // Counts clock cycle within integration window
    end
    
// Dead-time / Edge Counting
        if (e1_count1 == 0) begin        // If not in dead-time (e1_count1 == 0), and if you detect a high level on e1, 
        if (e1 == 1) begin               // increment sum and enter dead-time by setting e1_count1 to 1.
        sum <= sum + 1;                      
        e1_count1 <= 1;
            end 
        end
  
    else begin                           // This is when you are in dead time. 
    if (store_v > val) begin             // Increments store_v until it exceeds the configured dead-time value `val`,
        store_v <= 0;                    // then clears the dead-time flag (e1_count1) and resets store_v.
        e1_count1 <= 0;                  // While store_v <= val, further pulses on e1 are ignored because e1_count1 remains 1.
     
    end
    else begin
    store_v <= store_v + 1;
    end
    end
    
    
    end

// AXI4-Stream data      
assign M_AXIS_tdata = {4'b0, {12{e1_count1}}, {3{sum_hold[13]}}, sum_hold[12:0]}; 
assign M_AXIS_tvalid = S_AXIS_tvalid;     // Counter forwards upstream tvalid signal and doesn't independentely decide 
                                          // when the output data is valid to keep data flow synchronous and continuous
assign S_AXIS_tready = 1;                 // Counter always ready to recieve input

endmodule
```

### Explanation of  AXI4-Stream data

- **Referenced code:**
```verilog
assign M_AXIS_tdata = {4'b0, {12{e1_count1}}, {3{sum_hold[13]}}, sum_hold[12:0]}; 
```
This line constructs a 32-bit data word to be sent over the AX14-Stream interface to the DAC.

- **How the 32-bit data is packed:**
```verilog
assign M_AXIS_tdata =
  { 4'b0,                 // [31:28]
    {12{e1_count1}},      // [27:16]
    {3{sum_hold[13]}},    // [15:13]
    sum_hold[12:0]        // [12:0]
  };
```
`M_AXIS_tdata` is 32-bits wide because the DAC's AXI4-Stream input expects a 32-bit bus as shown below in Figure 7.

![DAC block](/images/photon_counter/DAC_block.png)

**Figure 7:** DAC block showing that the `S_AXIS_tdata` expects a 32-bit data bus. 

- **What the DAC Actually Uses:**
    - Although the AX14-Stream bus is 32 bits wide, the DAC module internally reads only 14 bits per output channel:
        - Bits [13:0] -> Channel A (OUT 1)
        - Bits [29:16] -> Channel B (OUT 2)
        - Bits [14], [15], [30], [31] are ignored.
    - So, while `M_AXIS_tdata` must be 32 bits for compatibility, only 28 bits are actively used by the DAC.

- **Photon Counter Output (`sum_hold`):**

```verilog
{3{sum_hold[13]}}, sum_hold[12:0]
```
- Explanation:
    - The counter produces a 14-bit value `sum_hold[13:0]` but we need to pad/extend the 14-bit signal to fit nearly into the 16-bit half of the stream which is why `sum_hold[13]` is repeated three times. 
    - Bits [15:14] are ignored.

### Converting Counts to Output Voltage:  

This section is relevant if you connect the Red Pitaya DAC to an oscilloscope and want to interpret the voltages you observe. This is how I checked if my counter worked.

**Note:**
- The explanation below reflects my reasoning and observations during my internship. I no longer have access to the hardware to re-verify the behaviour, so please treat this as a working interpretation rather than a confirmed fact.
- If you intend to replicate or extend this work, I strongly recommend double-checking the full signal chain:

    - Inspect the counter output codes (digital counts from your HDL).
    - Confirm what values are sent into and out of the DAC module (e.g., after any bit inversions).
    - Measure or simulate the DAC output to understand how the Red Pitaya converts those codes into voltage.

- It’s best to trace this end-to-end—from the Verilog counter to the SMA output—to confirm the exact polarity and scaling on your setup. Some other resources that might be helpful:

    - [Red Pitaya Schematics v1.0.1](https://downloads.redpitaya.com/doc/Red_Pitaya_Schematics_v1.0.1.pdf)
    - [AD9763/AD9765/AD9767 Datasheet](https://www.analog.com/media/en/technical-documentation/data-sheets/AD9763_9765_9767.pdf)

![count output on oscilloscope](/images/photon_counter/count_oscilloscope.JPEG)
**Figure 8:** This is an example of what to expect when you input a square wave from a signal generator to the red pitaya and how its windowed counts are displayed when connected to the DAC.

- The DAC output range corresponds to −1 V to +1 V (2 V full-scale) which can be seen in Figure 9 where instead of outputting the windowed count, you are able to see the counts increasing with time and then wrapping. The voltages are limited by the DAC output.
- Digital values `0 → 8191` represent positive voltages.
- Digital values `8192 → 16383` are interpreted as negative voltages.

    |Count|Digital Count|Output Voltage (approx.)|binary pattern|
    |----|---|---|---|
    |8191|+8191|+1 V|`0111 1111 1111 11`|
    |4096|+4096|+0.5V|`0100 0000 0000 00`|
    |0|0|0 V|`0000 0000 0000 00`|
    |16383|-1|Slightly below 0 V|`1111 1111 1111 11`|
    |12288|-4096|-0.5V|`1100 0000 0000 00`|
    |8192|-8192|−1 V|`1000 0000 0000 00`|

![counter ramp oscilloscope](/images/photon_counter/count_ramp_oscilloscope.JPEG)
**Figure 9:** Sawtooth Wave showing the live DAC output from the counter ramping as it recieves pulses.

- The relationship between the digital count (unsigned from your FPGA) and the analogue output voltage is:

- N< 8192:

$$
V_{\text{out}} = \frac{N}{2^{13} - 1} \times 1\,\text{V}
$$

  - N > 8191:

$$
V_{\text{out}} = \frac{(N - 2^{14})}{2^{13}} \times 1\,\text{V}
$$

where  
$${\text{N}} = {\text{digital count (0 to 16383)}}$$
$${2^{13} = 8192} {\text{ is the midpoint where the sign bit flips}}$$
$${2^{14} = 16384} {\text{ is the total number of possible digital codes}}$$

- For an N-bit signed number, the representable range is:

$$
-2^{N-1} \text{ to } 2^{N-1} - 1
$$

- The MSB is used as the sign bit:
    - `0` for positive
    - `1` for negative
    - When your unsigned counter output crosses 8192, the DAC wraps from +full-scale to −full-scale which in our case is +1 V to -1 V.
- Each Least Significant Bit (LSB) corresponds to:

$$
\text{1 LSB} = \frac{2\,\text{V}}{2^{14}} \approx 122\,\mu\text{V}
$$

- So increasing the digital count by 1 changes the DAC output by roughly 122 µV.

- **Dead-time Indicator (`e1_count1`)**
```verilog
4'b0, {12{e1_count1}}
``` 
- Explanation: 
    - The `e1_count1` signal is a 1-bit flag that goes high when the counter is in dead-time (i.e. temporarily ignoring new pulses).
    - Here, it is used as a simple debugging signal displayed on the DAC's second channel to verify when the module is in dead-time.
    - `{12{e1_count1}}` simply replicates this bit 12 times, filling most of the upper 16-bit half. The remaining 4 bits (`4'b0`) pad to the total 16 bits.
    - The exact padding scheme is flexible, for instance, `1'b0, 15{15{e1_count1}}` would work equally well.
    - The key point is that the DAC only reads 14 of those bits, so the rest just ensures the concatenation produces a valid 32-bit word.

## Tutorial

### Step 1: Open a Vivado Project 

1. Open up your Vivado project described in the [Setup Guide](/introduction/setup_guide.md#opening-a-new-project)
2. Open up your Block Design. 

### Step 2: Adding a Module

1. Click on Add Sources located in the Flow Navigator under Project Manager
![add sources](/images/photon_counter/add_sources.png)

2. Select Add or create design sources and press Next
![add or create design sources](/images/photon_counter/add_or_create_design_sources.png)

3. Select Create File, type counter for File Name and press okay
![file name](/images/photon_counter/file_name.png)

4. Press Finish
![finish](/images/photon_counter/finish.png)

5. Press ok
![ok](/images/photon_counter/ok.png)

6. Navigate to Sources, double click on counter(counter.v) and it will open up the counter.v tab
![counter tab](/images/photon_counter/open_counter_tab.png)

7. Delete the contents of this tab and replace with the following code:

```verilog
`timescale 1ns / 1ps

module counter(
input [31:0] window_length,
input e1,
input clk,
input [31:0] val, 

// AXI4-Stream data out  
output [31:0] M_AXIS_tdata,
output M_AXIS_tvalid,

// AXI4-Stream data in
input [31:0] S_AXIS_tdata,
input S_AXIS_tvalid,
output S_AXIS_tready


    );
    
reg e1_count1 = 0;
reg [13:0] sum =0; 
reg [31:0] store_v =0;
reg [31:0] store_w =0;
reg [13:0] sum_hold =0; 
always @(posedge clk) begin

// Window Counter
    if (store_w > window_length) begin
        sum_hold <=sum; 
        store_w <= 0;                       //Initialise the store_w value to 0 after a complete integration window 
        sum <=0;                            //Clear sum at end of window 
    end else begin
        store_w <= store_w + 1;
    end
    
// Dead-time / edge counting
        if (e1_count1 == 0) begin
        if (e1 == 1) begin
        sum <= sum + 1;
        e1_count1 <= 1;
            end 
        end
  
    else begin
    if (store_v > val) begin
        store_v <= 0;
        e1_count1 <= 0;
     
    end
    else begin
    store_v <= store_v + 1;
    end
    end
    
    
    end
assign M_AXIS_tdata = {4'b0, {12{e1_count1}}, {3{sum_hold[13]}}, sum_hold[12:0]}; //out1 on all
// wire output to DAC
assign M_AXIS_tvalid = S_AXIS_tvalid;
assign S_AXIS_tready = 1;

endmodule
```
8. Ctrl+S or press the Save icon
![save](/images/photon_counter/save.png)

9. Go back to your block design and right click on the empty space and select Add Module..., select the counter and press ok. This is how you add your custom made code to the block design.
![add_module](/images/photon_counter/add_module.png)

10. We are going to set up pretty much what was done in the [Signal Passthrough (ADC -> DAC)](/projects/signal_passthrough_adc_dac.md) Section but with a few modifications. Add your ADC, DAC and clocking wizard to the block design and configure the clocking wizard by double clicking on it as shown below.

![clocking wizard configuration: clocking options](/images/signal_passthrough/clocking_options.png)

![clocking wizard configuration: output clocks](/images/signal_passthrough/output_clocks.png)

11. Connect the ADC, DAC, counter and clocking wizard as shown below. 
![adc_dac_counter_clocking_wizard_connection](/images/photon_counter/adc_dac_counter_clocking_wizard.png)

12. Add 2 GPIOs to your block design and configure as shown below.
![configure_gpio](/images/photon_counter/gpio_settings.png)

13. You will see a banner with Design Assistance Available. Select Run Connection Automation.
![run_design_connection](/images/photon_counter/design_connection_automation.png)

14. The following steps will be about configuring the E1 pin. I took inspiration from the [1.4.2.2.5. Extension 2 of the simple LED blinker lesson by Anton Potočnik](https://redpitaya-knowledge-base.readthedocs.io/en/latest/learn_fpga/4_lessons/LedBlink.html#extension-2) At this point, your block design will look like the below. Note the highlighted port `exp_p_tri_io[7:0]`.

![exp_p_tri_io](/images/photon_counter/exp_tri_p_io.png)

15. Delete that port. Right-click on the empty space in the block design and select Create Port.

![create_port](/images/photon_counter/create_port.png)

16. Do as below.(*Note: The DAC control signals dac_rst and dac_sel are incorrectly shown in this specific diagram. The correct connections are used in all other diagrams where this note does not appear.*)
![e1_pin](/images/photon_counter/e1_pin.png)

17. Insert a slice and configure as shown. (*Note: The DAC control signals dac_rst and dac_sel are incorrectly shown in this specific diagram. The correct connections are used in all other diagrams where this note does not appear.*)

|Field|Meaning|What to enter for our case|
|---|---|---|
|Din Width|The total width of the input signal. This is the size of the bus you are slicing from|`8` because `exp_p_tri_io[7:0]` is an 8-bit bus|
|Din From|The upper index of the range you want to slice|`0` because we want to use the `DIO0_P / EXT TRIG` pin out of the 8 pins, if you wanted to use a different pin, you would choose a different number|
|Din Down To| The lower index of the range you want to slice|`0` because we only want `[0]`|
|Dout Width| The number of bits in your outputs signal|`1` but it should automatically fill once the other two are set|

![slice_properties](/images/photon_counter/slice_properties.png)
18. Re-configure LED to accept the 1 bit from the slice:
- In the block design, locate the port labelled `led_o[7:0]`
- Right-click it and choose External Port Properties.
- At the bottom, there will be two tabs: "General" and "Properties." 
- Click on the Properties tab.
- Change the LEFT attribute from 7 to 0, so only a single LED line (led_o[0:0]) is exposed.
19. Connect the `exp_p_tri_io[7:0]` to the slice which is connected to the `e1` port of the counter and the `led_o[0:0]` port. 
![block_design](/images/photon_counter/block_design.png)
20. Generate your bitstream and connect upload it to your red pitaya. More information on how to do that in the [Connecting to your Red Pitaya](/introduction/setup_guide.md#connecting-to-your-red-pitaya) section of the Set-up Guide

21. To set your window length and dead time:
- You will see the AXI GPIO mapped to a base address (e.g. 0x4120_0000). You can omit the underscore and type it as 0x41200000 in the next step.
- On the Red Pitaya terminal, type for example:[^3]
```bash
monitor 0x41200000 1250000
```
Use the monitor tool to set your window length and dead time to what you want. Remember they are in terms of number of cycles so for 0.01 seconds = 1250000 cycles because 8ns corresponds to 1 cycle.
![address_editor](/images/photon_counter/address_editor.png)

## Testing
- Once everything is connected and configured, you can verify your code is working as it should using a signal generator and an oscilloscope.

- Make sure to [impedance-match](/introduction/red_pitaya.md#outputs-out1-out2) the Red Pitaya output to the oscilloscope input (both should be terminated to 50 Ω) to avoid signal reflections and distortion.

- Set your signal generator to output a square wave that mimics the LVTTL pulses expected by the E1 input.
Since the E1 header operates at 3.3 V logic, configure the generator output to swing between 0 V and 3.3 V.[^4]

![signal_generator_input_to_match_e1_lvttl](/images/photon_counter/signal_generator_e1_lvttl.JPEG)

- Keep the duty cycle small so that the high period of the pulse is short. This prevents double-counting when the counter samples at 125 MHz.

- Ensure that:
    - The dead time you set in the FPGA design is longer than the pulse high duration, so that a single pulse is only counted once.(You can fine-tune this by adjusting the signal generator’s duty cycle.)
    - But also that the dead time is shorter than the total pulse period, so that the next pulse still falls outside the dead-time window.


$$\text{Duty Cycle \%} = \frac{\text{t}_{high}}{\text{T}_{period}} \times 100$$

$$\text{Duty Cycle (\\%)} = \frac{\text{t}_{high}}{\text{T}_{period}} \times 100$$

$$
\text{Duty Cycle (\\%)} = \dfrac{t_\mathrm{high}}{T_\mathrm{period}} \times 100
$$

$${t}_{high} = \text{the time the signal stays high in one cycle}$$
$${T}_{period} = \text{the total time of one complete cycle}$$

- Which E1 pins to connect:
    - Connect the wire recieving the pin to DIO_0 and the the ground to GND.

![extension_connectors](/images/red_pitaya/extension_connectors.png)
![e1_connection](/images/photon_counter/e1_connection.JPEG)
![set_up_e1_to_signal_generator](/images/photon_counter/set_up_e1_to_signal_generator.JPEG)

- When observing the outputs:
    - One DAC output will display the integrated count of pulses over your programmed window length such as in the image below with the measured voltage corresponding to a certain number of pulses as covered in the background section.
    - The other will toggle between high and low, indicating when the counter is in dead-time (ignoring new pulses).

![counted pulses over a window length on oscilloscope](/images/photon_counter/count_oscilloscope.JPEG)
## Further Steps

### Double-Check
- If you intend to replicate or extend this work, I strongly recommend double-checking the full signal chain:

    - Inspect the counter output codes (digital counts from your HDL).
    - Confirm what values are sent into and out of the DAC module (e.g., after any bit inversions).
    - Measure or simulate the DAC output to understand how the Red Pitaya converts those codes into voltage.

- It’s best to trace this end-to-end—from the Verilog counter to the SMA output—to confirm the exact polarity and scaling on your setup. Some other resources that might be helpful:

    - [Red Pitaya Schematics v1.0.1](https://downloads.redpitaya.com/doc/Red_Pitaya_Schematics_v1.0.1.pdf)
    - [AD9763/AD9765/AD9767 Datasheet](https://www.analog.com/media/en/technical-documentation/data-sheets/AD9763_9765_9767.pdf)

### Improving the Counter Design

*Disclaimer: I haven't yet tested them. These are just ideas for improving the counter.*

- Two-flip-flop synchroniser and rising edge detector:
    - The E1 pin carries asynchronous LVTTL pulses which means they are not aligned with the FPGA's internal 125 MHz clock.
    - The logic inside the FPGA only updates on the edge of that internal clock as shown in Figure 2.
    - However, if the external signal changes right around the clock edge of that internal clock, the FPGA flip-flop that samples it can enter a state called metastability where it is not cleanly a `0` or `1` for a short time and can lead to random behaviour downstream.
    - To solve this:

    This code I adapted from the one in [the Crossing Clock Domain lesson on nandland.com](https://nandland.com/lesson-14-crossing-clock-domains/)[^5]
    ```verilog
    reg syn_ff1;
    reg syn_ff2;
    reg syn_ff3;

    always @(posedge clk) begin
        // Two-flip-flop synchroniser
        syn_ff1 <= e1;                               // may go metastable
        syn_ff2 <= syn_ff1;                          // metastability resolves by next cycle
        // rising edge detector
        syn_ff3 <= syn_ff2;

        if (syn_ff3 == 1'b0 && syn_ff2 == 1'b1) begin
        // Dead-time and other logic here
        end
     end
    ```
    OR

    This code, I tweaked to be a bit cleaner.
    ```verilog
    reg syn_ff1;
    reg syn_ff2;
    reg rising_edge;

    always @(posedge clk) begin
        // Two-flip-flop synchroniser
        syn_ff1 <= e1;                               // may go metastable
        syn_ff2 <= syn_ff1;                          // metastability resolves by next cycle
    
        // Rising-edge detector
        rising_edge <= (syn_ff2 & ~syn_ff1);        // ~ is for inverting syn_ff1 so that when syn_ff2 and syn_ff1 are different, the overall statement is true

        if (rising_edge) begin                      // if rising_edge = 1 i.e. true, this code branch will be executed
            // Dead-time and other logic here
        end
    end
    ```

    - What these code do is use a 2-ff synchroniser to re-sample over two cycles. `syn_ff1` might go metastable if the E1 input changes too close to the clock edge. But this is usually resolved withing a few nanoseconds to a definite `0` or `1`. `syn_ff2` samples this one clock cycle later and sees the clean and stable value. This method is to ensure that the metastability dies out before it can affect the rest of the design.
    - Your incoming pulse might be high for multiple cycles and by detecting the `0` to `1` transition, you ensure only one pulse per event is counted.
    - A nice youtube video accompanies the website I referenced if you want to learn what approaches to take if you are going from a slow to fast domain, fast to slow domain and handling multiple clock domains: [Crossing Clock Domains in an FPGA](https://www.youtube.com/watch?v=eyNU6mn_-7g&t=960s)

- Xilinx Parameterized Macros:
    - Instead of manually writing Verilog for a two-flip-flop synchroniser, Xilinx provides a set of [Xilinx Parameterized Macros (XPMs)](https://docs.amd.com/r/en-US/ug953-vivado-7series-libraries/Xilinx-Parameterized-Macros)[^6] which are pre-verified HDL blocks optimised for safe clock domain crossing and other common design patterns. These are directly available within Vivado's IP catalog. 
    - I haven't tested it but perhaps start with:
        - Clicking on the blank space in the block design, select Add IP, search `xpm_cdc` and select that. 
        - Double click the `xpm_cdc` block and configure as shown below.
        ![customise_xpm_cdc](/images/photon_counter/customise_xpm_cdc.png)
        - Add a constant block, set it to 1'b0 and tie to the unused port `src_clk`.
        ![customise_slice](/images/photon_counter/customise_slice.png)
        - Connect blocks together as shown below.
        ![block_design_with_xpm_cdc](/images/photon_counter/block_design_with_xpm_cdc.png)
    - My thinking: 
        - I used the `xpm_cdc_single` to synchronise the E1 input signal to the 125 MHz ADC clock.
        - In the attributes, I set the destination sync flip-flops to 2 as in the Two-flip-flop synchroniser and rising edge detector example above. I think increasing it could lower the metastability risk even more but might create delays in data transfer (increase latency). I think 2 FFs is okay for 125 MHz.
        - The E1 pin generates pulses asynchronously (not driven by another clock) so I wanted to set `src_input_reg` to 0 and tie the unused `src_clk` port to 1'b0 since there is no source clock domain.
        - For reference, look at [Vivado Design Suite 7 Series FPGA and Zynq 7000 SoC Libraries Guide (UG953)](https://docs.amd.com/r/en-US/ug953-vivado-7series-libraries/XPM_CDC_SINGLE), which also includes Verilog and VHDL instantiation templates that you can adapt and embed directly inside your counter module if preferred.

### PID
- During the internship, I began wiring the counter output into a PID controller but didn't have time to finish testing: [PID by Ben Millward](https://github.com/Bentwin2002/Group_IV_RP/blob/1916aeb4dfb7fd2a1658bcffc84c78c3ad14fbd9/Complete_setup/tmp/wip_9/wip_9.srcs/sources_1/new/PID_FINAL_b.v)

## References:

[^1]: Euresys. TTL and LVTTL Logic Levels, Coaxlink 10.3 Documentation. Available at: https://documentation.euresys.com/Products/Coaxlink/Coaxlink_10_3/Content/03_Using_Coaxlink/application-notes/connecting-ttl-to-isolated-ports/TTL_and_LVTTL_levels.htm

[^2]: Red Pitaya d.o.o. *Red Pitaya Schematics v1.0.1*. Available at: https://downloads.redpitaya.com/doc/Red_Pitaya_Schematics_v1.0.1.pdf 

[^3]: Red Pitaya d.o.o. 3.6.8. Monitor utility Available at: https://redpitaya.readthedocs.io/en/latest/appsFeatures/command_line_tools/utils/monitor_util.html

[^4]: Red Pitaya d.o.o. Extension connectors (Original boards) Available at: https://redpitaya.readthedocs.io/en/latest/developerGuide/hardware/ORIG_GEN/hw_specs/extent.html#extension-connector-e1

[^5]: Russell Merrick, Nandland.com. *Crossing Clock Domains in an FPGA* Available at: https://nandland.com/lesson-14-crossing-clock-domains/

[^6]: AMD. *Vivado Design Suite 7 Series FPGA and Zynq 7000 SoC Libraries Guide (UG953)*. Available at: https://docs.amd.com/r/en-US/ug953-vivado-7series-libraries/XPM_CDC_SINGLE
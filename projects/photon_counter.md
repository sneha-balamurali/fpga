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
**Figure 3:** Diode

![led schematic](/images/photon_counter/led_schematic.png)

**Figure 4:** LED in Red Pitaya schematic[^2]

- **Floating Inputs and Pull-Up/Pull-Down Resistors**
    - Digital input pins like those on the E1 header can sometimes be floating when no signal source is connected - meaning they are not at a defined voltage level (neither 0 V or 3.3 V). This can lead to issues.
    - To prevent unclear voltage states, pull-up or pull-down resistors are used internally.
    - A pull-up resistor connects the input to read as logic high (1) when not actively driven low.
    - A pull-down resistor connects the input to read as logic low (0) when not actively driven high.

    - From Figure 5, we can see that exp_p_tri_io pins have a pull-up configuration meaning they default to high unless an external signal drives them low. 
    - When I had connected `exp_p_tri_io[7:0]` pin to `led[7:0]`, all the pins except the one connected to a source signa (`exp_p_tri_io[0:0]`) were lit up and this is because of this pull-up configuration. This is shown in Figure 6. Since I wanted the LED to just mirror the digital input, I altered the design to `exp_p_tri_io[7:0]` -> `Slice` -> `led_o[0:0]`

![implementation_e1_properties](/images/photon_counter/implemetation_e1_properties.png)
**Figure 5:** Once you have run implementation, you can go the the Flow Navigator, expand Open Implemented Design, and click on Schematic. In the tool bar, you will see X Cells, Y I/O Ports, and Z Nets. Click on the Y I/O Ports and search for `exp_p_tri_io`.

![pull-up behaviour in led connected to e1 pins](/images/photon_counter/pull_up_making_led_1_to_7_light_up.JPEG)
**Figure 6:** Pull-up behaviour by directly connecting `exp_p_tri_io[7:0]` pin to `led[7:0]`.

### Counter Code

- I have annotated the counter code with some explanations. 
- A discussion on the AXI4-Stream data which I haven't annotated will follow as I feel it requires a deeper exploration.

```verilog
module counter(
// Module Ports

input [31:0] window_length,                // a 32-bit value specifying how many clock cycles make up one integration window

input e1,                                  // the input pulse signal you want to count

input clk,                                 // the system clock; the counter increments on the rising edge of this clock

input [31:0] val,                          // a 32-bit value defining the dead-time, i.e. how many clock cycles must pass before another pulse on e1 is counted

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
                                         // When 1, module ignores further pulses until the dead-time counter expires              

reg [13:0] sum =0;                       // Stores the number of valid pulses detected eithin the current integration window


reg [31:0] store_v =0;                   // Used to implement dead-time. 
                                         // Counts cycles while e1_count1 = 1 and resets when the configured dead-time (val) expires

reg [31:0] store_w =0;                   // Tracks the number of clock cycles elapsed in the current integration window. 
                                         // When it exceeds the window_length, accumulated sum is captured and the counter resets

reg [13:0] sum_hold =0;                  // Holds the pulse count from the most recently completed window. 
                                         // This value is packaged into M_AXIS_tdata on the next clock cycle.

always @(posedge clk) begin

// Window Counter
    if (store_w > window_length) begin
        sum_hold <=sum;                  // Stores sum at the end of each window so that it can be output
                                         // while the counter resets for the next window

        store_w <= 0;                    // Initialise the store_w value to 0 after a complete integration window 
        sum <=0;                         // Clear sum at end of window 
    end else begin
        store_w <= store_w + 1;          // Counts clock cycle within integration window
    end
    
// Dead-time / Edge Counting
        if (e1_count1 == 0) begin        // If not in dead-time (e1_count1 == 0), and if you detect a high level on e1, 
        if (e1 == 1) begin               // increment sum and enter dead-time by setting e1_count1 to 1
        sum <= sum + 1;                      
        e1_count1 <= 1;
            end 
        end
  
    else begin                           // This is when you are in dead time. Increments store_v until it exceeds the configured 
    if (store_v > val) begin             // dead-time value `val`, then clears the dead-time flag (e1_count1) and resets store_v.
        store_v <= 0;                    // While store_v <= val, further pulses on e1 are ignored because e1_count1 remains 1.
        e1_count1 <= 0;
     
    end
    else begin
    store_v <= store_v + 1;
    end
    end
    
    
    end
assign M_AXIS_tdata = {4'b0, {12{e1_count1}}, {3{sum_hold[13]}}, sum_hold[12:0]}; 
assign M_AXIS_tvalid = S_AXIS_tvalid;
assign S_AXIS_tready = 1;

endmodule
```

## Tutorial

### Step 1: Open a Vivado Project 

1. Open up your Vivado project described in the [Setup Guide](/introduction/setup_guide.md#opening-a-new-project)
2. Open up your Block Design. 

### Step 2:

## References:

[^1]: Euresys. TTL and LVTTL Logic Levels, Coaxlink 10.3 Documentation. Available at: https://documentation.euresys.com/Products/Coaxlink/Coaxlink_10_3/Content/03_Using_Coaxlink/application-notes/connecting-ttl-to-isolated-ports/TTL_and_LVTTL_levels.htm

[^2]: Red Pitaya d.o.o. *Red Pitaya Schematics v1.0.1*. Available at: https://downloads.redpitaya.com/doc/Red_Pitaya_Schematics_v1.0.1.pdf 

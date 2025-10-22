# Signal Passthrough (ADC -> DAC)

## What this Section Covers

This section introduces real I/O — sending and receiving analogue signals through the Red Pitaya’s ADC and DAC.

It covers:

- **ADC and DAC:**
    - How the Red Pitaya's 14-bit ADC and DAC channels operate at 125 MS/s
    - How the DAC uses a Double Data Rate (DDR) interface and a 250 MHz clock
    - How two DAC channels are multiplexed onto one 14-bit bus using ODDR *(not covered in detail, but you can explore it further starting with the referenced page and the DAC core HDL)*
- **Clocking Wizard**
    - Using the Clocking Wizard to derive the 250 MHz DAC clock from the 125 MHz ADC reference
- **AXI4-Stream Dataflow**
    - How the ADC and DAC cores exchange data through continuous AXI4-Stream interfaces
- **Design Integration:**
    - Building a simple signal passthrough system (ADC → DAC) in Vivado 
    - Connecting and Configuring the IP blocks and Clocking Wizard
- **Quick Links:**
    - [Tutorial](/projects/signal_passthrough_adc_dac.md#tutorial)
    - [Testing](/projects/signal_passthrough_adc_dac.md#testing)
    - [References](/projects/signal_passthrough_adc_dac.md#references)
    
By the end of this section, you’ll understand how to route real analogue signals through the FPGA, manage multiple clock domains, and prepare a foundation for inserting your own processing logic (e.g., filters, PID controllers) between the ADC and DAC.

## Background:

This project shows how to take an input signal from the Red Pitaya ADC (Analog-to-Digital Converter) and directly send it to the DAC (Digital-to-Analog Converter). At first glance this looks like a simple “wire” connection, but in practice the ADC and DAC operate on different timing requirements, so we need to manage the clocks carefully.

In this design we use Pavel Demin’s ADC and DAC cores[^1], which are already included in the cores folder. These cores expose the Red Pitaya converters as AXI4-Stream interfaces, making it straightforward to connect them together.

### ADC and DAC on the Red Pitaya

- The Red Pitaya has two ADC channels, each 14-bit, sampling at 125 MS/s.[^2]
- It also has two DAC channels, also 14-bit, which reconstruct analogue signals.
- The ADC data comes in on two buses:
    - `adc_dat_a_i[13:0]`
    - `adc_dat_b_i[13:0]`

- The DAC outputs on one bus:
    - `dac_dat_o[13:0]`

#### Why the DAC requires a faster clock

The DAC is a DDR (Double Data Rate) device using ODDR[^3]. Instead of latching data only on the rising edge of its clock, it captures one channel on the rising edge and the other on the falling edge.

- Rising edge → output channel 1
- Falling edge → output channel 2

Because of this, the DAC needs a 250 MHz clock to achieve an effective 125 MS/s per channel. This explains why the DAC interface looks like a single bus, even though it drives two outputs: the FPGA multiplexes data for the two channels onto one 14-bit bus and uses the faster clock to deliver it.

### Clocking Wizard

The Clocking Wizard is an mixed-mode clock manager (MMCM)/phase-locked loop(PLL)-based clock generator that will let us[^4]: 

- Take the ADC's 125 MHz sampling clock as its input (`clk_in1`)
- Generate 250 MHz output clock (`clk_out1`) for the DAC

This setup ensures that:

- All FPGA-side logic (e.g. AXI-Stream data registers) runs in the 125 MHz domain (`adc_clk`).
- The DAC receives a phase-related 250 MHz clock, so the two domains remain aligned and no drift occurs.

### AXI4-Stream Wrappers

In Vivado, the Red Pitaya ADC and DAC IP blocks are wrapped as AXI4-Stream interfaces:

- ADC block (`axis_red_pitaya_adc`) → provides data on an AXI-Stream interface, 32-bit AXI-Stream data containing both 14-bit channels (with padding).
- DAC block (`axis_red_pitaya_dac`) → consumes data on an AXI-Stream interface.

AXI4-Stream is ideal for this use case: it is a lightweight protocol designed for continuous, high-throughput dataflow. By connecting the ADC’s AXI-Stream output directly to the DAC’s AXI-Stream input, we create the simplest possible passthrough system.

Later, custom processing logic can be inserted between the ADC and DAC blocks (e.g. filters, PID controllers, FFTs) without changing the surrounding clocking scheme.

## Tutorial

### Step 1: Add a Clocking Wizard to your block design and configure it as shown in Figure 1 and Figure 2
- Double click on the Clocking Wizard to re-customise it.

![clocking wizard configuration: clocking options](/images/signal_passthrough/clocking_options.png)
**Figure 1:** Clocking Options

![clocking wizard configuration: output clocks](/images/signal_passthrough/output_clocks.png)
**Figure 2:** Output Clocks

### Step 2: Create the block design shown in Figure 3

![signal_passthrough_block_design](/images/signal_passthrough/block_design.png)
**Figure 3:** Complete block design for signal passthrough

### Step 3: Generate bitsream and upload to Red Pitaya

- [Go to Section Connecting to your Red Pitaya of the Set Up Guide](/introduction/setup_guide.md#connecting-to-your-red-pitaya)

## Testing

### Step 1: Connect signal generator and oscilloscope to Red Pitaya

- Connect a signal generator to one of the Red Pitaya’s inputs and an oscilloscope to one of its outputs.
- Make sure to [impedance-match](/introduction/red_pitaya.md#outputs-out1-out2) the Red Pitaya output to the oscilloscope input. Both should be terminated at 50 Ω to prevent signal reflections and distortion.

![input_output_connections](/images/signal_passthrough/input_output_connections.JPEG)
**Figure 4:** Connections on the Red Pitaya for signal passthrough verfication.

![set_up](/images/signal_passthrough/set_up.JPEG)
**Figure 5:** Complete experimental setup.

### Step 2: Configure your signal generator

- Make sure that the signal generator's peak-to-peak voltage (Vpp) is within the Red Pitaya's supported input range which is [±1V or ±20V range depending on configuration](/introduction/red_pitaya.md#inputs-in1-in2). 

![signal_generator](/images/signal_passthrough/signal_generator.JPEG)
**Figure 6:** Example configuration on signal generator.

### Step 3: Observe Waveform on oscilloscope
- Verify that the waveform observed on the oscilloscope matches the signal generated.

![oscilloscope](/images/signal_passthrough/oscilloscope.JPEG)
**Figure 7:** Output waveform observed on the oscilloscope.

## References

[^1]: Pavel Demin, Available at : https://github.com/pavel-demin/red-pitaya-notes/tree/master/cores

[^2]: Red Pitaya d.o.o. *Red Pitaya Schematics v1.0.1*. Available at: https://downloads.redpitaya.com/doc/Red_Pitaya_Schematics_v1.0.1.pdf 

[^3]: AMD. Vivado Design Suite 7 Series FPGA and Zynq 7000 SoC Libraries Guide (UG953), Section: ODDR. Available at: https://docs.amd.com/r/en-US/ug953-vivado-7series-libraries/ODDR

[^4]: AMD. Clocking Wizard LogiCORE IP Product Guide (PG065). Available at: https://docs.amd.com/r/en-US/pg065-clk-wiz/Features


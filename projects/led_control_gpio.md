# LED Control (GPIO)

## What this Section Covers

When we made the LED blink earlier, the logic ran entirely inside the FPGA: a counter divided down the internal 125 MHz clock, and the result was sent straight to the LED. But what if we want the CPU to decide what the LED should do? This project introduces the idea of the PS talking to the PL over the AXI bus.

It covers:

- **AXI Basics:**
  - What AMBA AXI is and why it matters on the Zynq-7000
  - The manager/subordinate model and the five AXI channels
  - The VALID/READY handshake that every channel uses

- **AXI Variants:**
  - AXI4, AXI4-Lite, and AXI4-Stream, and what “memory-mapped” means

- **AXI GPIO IP:**
  - How GPIO is exposed as memory-mapped registers
  - How to connect AXI GPIO to the LED outputs in Vivado
  - Adding and customising the AXI GPIO IP core
  - Writing values into the GPIO register from the CPU to turn LEDs on/off

By the end of this section, you’ll understand how the PS and PL exchange information over AXI, and you’ll control the LEDs directly from the processor using a memory-mapped GPIO block.

## Background

This section introduces a lot of new concepts. Don't worry if it feels abstract - you don't need to understand it all to do the project. It will make sense once you follow the steps.

#### More in-depth resources:
1. [Arm Technical Docs: AMBA AXI Protocol Specification](https://developer.arm.com/documentation/ihi0022/latest)
  Note that Red Pitaya uses AXI4 family protocols and the newer docs talk about AXI5. The older doc I found with AXI4 being covered was the [AMBA AXI and ACE Protocol Specification (ARM IHI 0022 H.c)](https://www.bing.com/ck/a?!&&p=64bf90b55ca057d4388f91e53e86b3daccaa698d1b12a3a7e5f80018022845ebJmltdHM9MTc1OTQ0OTYwMA&ptn=3&ver=2&hsh=4&fclid=12bb9a37-2ef6-6c4c-075e-8c1c2f166ddb&psq=AMBA%c2%aeAXIandACEProtocol+Specification.+(ARM+IHI+0022+H.c)&u=a1aHR0cHM6Ly9kZXZlbG9wZXIuYXJtLmNvbS8tL21lZGlhL0FybSUyMERldmVsb3BlciUyMENvbW11bml0eS9QREYvSUhJMDAyMkhfYW1iYV9heGlfcHJvdG9jb2xfc3BlYy5wZGY_cmV2aXNpb249NzFiZDdjNTctMmVkNy00ODdiLWJjM2UtNjhjNGFiNTZmYTVmJmxhPWVuJmhhc2g9NjMyNTMxMTAxMkREQURGMjM4QzM1QTZDMEZENzM0RTUyMDc1NEY4Mg)
2. [Master's Thesis on  Real-Time Processing of Laser-Ultrasonic Signals by Thomas Paireder](https://epub.jku.at/obvulihs/download/pdf/2406394?originalFilename=true)
3. [Vivado Design Suite: AXI Reference Guide (UG1037)](https://docs.amd.com/v/u/en-US/ug1037-vivado-axi-reference-guide)

### AMBA AXI Protocol Overview

- AMBA = Advanced Microcontroller Bus Architecture (a set of on-chip communication standards created by Arm).
- AXI = Advanced eXtensible Interface, one of the most widely used AMBA protocols.
- It defines how different parts of a system-on-chip (SoC) talk to each other, like processors, memory controllers, and peripherals.
- (Based on the AMBA AXI protocol specification [^1].)

In our case, AXI is the bus protocol used on the PS–PL AXI ports (GP/HP/ACP) of the Zynq-7000 shown below. These ports are the standard path for memory-mapped communication between the ARM cores in the Processing System (PS) and custom logic in the Programmable Logic (PL). [^2] [^3]

Therefore: if you design a custom FPGA block (e.g. a PID controller, sine generator, FIR filter), AXI is the standard way you connect it so the ARM processor can read/write to it. 

![GP_HP_ACP](/images/led_control_gpio/AMBA/GP_HP_ACP.png)
**Figure 1:** The PS–PL interfaces provide standard AXI pathways between the ARM cores in the Processing System (PS) and custom logic in the Programmable Logic (PL). The screenshot shows the GP (General Purpose), HP (High Performance), and ACP (Accelerator Coherency Port) AXI ports available in the Vivado block design.

### Master and Slave Components

AXI specifies how intellectual property (IP) blocks communicate with each other. It standardises the interfaces at the block boundaries. Within AXI, there are two roles for interfaces: a Manager (Master) and a Subordinate (Slave) [^1].

- A Manager/Master = the one who initiates a transaction
  - Example: the ARM processor (PS) writes data to a custom filter in the FPGA
- A Subordinate/Slave = the one who responds to the transaction
  - Example: your custom Verilog FIR filter block that receives the data

If multiple managers and subordinates are present in a system, an AXI interconnect is used, which has manager and subordinate interfaces of its own. It makes it easier to integrate different IP blocks. Vivado automatically inserts this as AXI SmartConnect, as shown in [Step 3: Use Design Assistance to Run Connection Automation](/projects/led_control_gpio.md#step-3-use-design-assistance-to-run-connection-automation) labelled AXI SmartConnect.

![AXI SmartConnect](/images/led_control_gpio/led_control_block_design.png)
**Figure 2:** Completed block design showing the AXI SmartConnect inserted by Connection Automation.

Note: In older documents you will see "Master" and "Slave" being used. Obviously, it is quite problematic terminology and so in newer docs you will see "Manager" and "Subordinate" instead.

### The Five AXI Channels

The AXI protocol defines 5 channels that are divided into two groups:[^1]

- channels for write transactions
- channels for read transactions

By separating them, AXI allows both to occur in parallel because they can happen at the same time.

### Write path:
- The manager provides a target address on the Write Address (**AW**) channel which is used by the subordinate to decide where the data is received.[^1]
- It then sends the corresponding data over the Write Data (**W**) channel.
- Once the subordinate has stored the data, it sends a confirmation or error back through the Write Response (**B**) channel.

![write transaction](/images/led_control_gpio/axi_protocol/write_transaction.png)
**Figure 3:** Adapted from Figure A1.1 of the AMBA AXI Protocol Specification [^4]

### Read path:
- The manager specifies the location it wants to access using the Read Address (**AR**) channel.[^1]
- The subordinate returns the requested data through the Read Data (**R**) channel from the requested address.
- If the access fails (invalid address, corrupted data, or insufficient permissions), the error is also flagged on the R channel.
![read transaction](/images/led_control_gpio/axi_protocol/read_transaction.png)
**Figure 4:** Adapted from Figure A1.2 of the AMBA AXI Protocol Specification [^4]

### Handshake signals (VALID/READY)

In AXI, every channel uses a common handshake procedure built on two signals: VALID and READY. [^1]

- VALID: driven by the source to show that data or control information is available. 
- READY: driven by the destination to show it is able to accept that information.

Which side acts as the source or destination depends on the channel. For instance, the manager drives the Write Address (**AW**) channel, but it is the destination on the Write Response channel (**B**) channel.

When a source asserts VALID, it must keep the signal high until the destination asserts READY and the transfer is accepted.

The transfer happens on the rising clock edge when both VALID and READY are high.

#### VALID and READY combinations:

![VALID and READY together](/images/led_control_gpio/handshake/valid_ready_together/valid_ready_together.svg)

**Figure 5:** VALID and READY asserted in the same cycle. Illustration of a transfer when VALID and READY are asserted together (based on AXI protocol description).

![VALID before READY](/images/led_control_gpio/handshake/valid_before_ready/valid_before_ready.svg)

**Figure 6:** VALID asserted before READY. Illustration of a transfer when the source asserts VALID before the destination asserts READY (based on AXI protocol description).

![VALID after READY](/images/led_control_gpio/handshake/ready_before_valid/ready_before_valid.svg)

**Figure 7:** READY asserted before VALID. Illustration of a transfer when the destination asserts READY before the source asserts VALID (based on AXI protocol description).

|Abbreviation|Channel|
|---|---|
|`ar`|Read Address|
|`aw`|Write Address|
|`b`|Write Response|
|`r`|Read Data|
|`w`|Write Data|

**Table 1:** The 5 AMBA AXI channels

![GPIO S AXI](/images/led_control_gpio/handshake/axi_gpio_channels.png)

**Figure 8:** The AXI GPIO IP from the tutorial below shows how the concepts described above - AXI channels, manager/subordinate roles, and VALID/READY handshakes - come together. By using Table 1, you can identify the type of AXI channel each block pin corresponds to. 

### AXI4 vs AXI4-Lite vs AXI-Stream and Memory Mapped

- In a memory-mapped protocol (such as AXI3, AXI4, and AXI4-Lite), each transaction specifies a location in the system’s address space along with the data to be transferred. In other words, peripherals appear as if they were regions of memory, and the processor interacts with them using read and write operations to those addresses.

AXI comes in several variants:

- AXI4 supports burst-based transfers, where multiple data words can be sent in sequence after a single address. This makes it efficient for high-throughput memory operations. 
- AXI4-Lite is a simplified version with no bursts — every transfer is a single address/data pair. It is typically used for lightweight peripherals that are accessed through memory-mapped registers, such as GPIO blocks[^5]. In this project, the LED controller is exposed to the processor through an AXI4-Lite interface.
- AXI4-Stream has no addresses at all, and is designed for continuous dataflow (for example, ADC/DAC passthrough).[^8]

For more details:

[Read Section 4.2.1. Basics of the AXI protocol and 4.2.2. Memory Map of the Master's Thesis on  Real-Time Processing of
Laser-Ultrasonic Signals by Thomas Paireder](https://epub.jku.at/obvulihs/download/pdf/2406394?originalFilename=true)

[Vivado Design Suite: AXI Reference Guide (UG1037)](https://docs.amd.com/v/u/en-US/ug1037-vivado-axi-reference-guide)

## Tutorial

In this tutorial, we won’t be writing any new HDL code ourselves. Instead, we’ll use IP cores — pre-built modules that Vivado provides.

You’ll build a design where the Zynq Processing System (PS) can write binary values into a memory-mapped register, and those values will appear on the Red Pitaya LEDs e.g. writing 3 lights LED 0 and LED 1.

1. Open up your Vivado project described in the [Setup Guide](/introduction/setup_guide.md#opening-a-new-project)
2. Open up your Block Design. 
3. Right-click in the blank workspace and select Add IP.

![add_ip](/images/led_blink/add_IP.png)
**Figure 9:** Add IP
## Step 1: Add an AXI GPIO

- Insert an AXI GPIO into your block design.[^5]
- Double click on the block to re-customise the IP and adjust the settings as shown below.

![change AXI GPIO settings](/images/led_control_gpio/axi_gpio_re_customise.png)
**Figure 10:** Change AXI GPIO settings
## Step 2: Connect AXI GPIO output to LED


- Expand the GPIO output by clicking the **+** sign next to `GPIO`.  
  `gpio_io_o[7:0]` should appear.  
- Connect this to the `led_o[7:0]` port, as shown below.
- Tip: Use Zoom Fit and Regenerate Layout (toolbar icons at the top) for a cleaner view.  

![connect AXI GPIO output to LED](/images/led_control_gpio/connect_axi_gpio_to_led.png)
**Figure 11:** Connect AXI GPIO to LED
## Step 3: Use Design Assistance to Run Connection Automation

- You will have noticed a green banner appearing after you added your AXI GPIO. 
- Run Connection Automation, you can leave it as Auto. By default, it maps the GPIO's AXI interface clock to `FCLK_CLK0`. To avoid any confusion, you can explicitly select `FCLK_CLK0` in the options.

![run connection automation](/images/led_control_gpio/run_connection_automation.png)
**Figure 12:** Run Connection Automation
![explicitly select FCLK_CLK0 in properties](/images/led_control_gpio/explicitly_select_options_in_run_automation.png)
**Figure 13:** Explicitly selecting FCLK_CLK0 (125 MHz) in properties
- After automation, your block design should look like below. 
- Note: In older Vivado versions, you may see an `AXI Interconnect` block (labelled `ps7_0_axi_periph`). In newer releases, Vivado uses `AXI SmartConnect (axi_smc)` instead. Both serve the same purpose: they bridge the PS master AXI interface peripheral slaves. [^6]

![block design](/images/led_control_gpio/led_control_block_design.png)
**Figure 14:** Complete block design
## Step 4: Generate Bitstream

- Click Generate Bitstream
- [Once complete, connect and upload to your Red Pitaya](/introduction/setup_guide.md#connecting-to-your-red-pitaya)

## Step 5: Write from the CPU

- In the figure below, you will see three tabs above your block design: **Diagram**, **Address Editor** and **Address Map**.
![three tabs: diagram, address editor, address map](/images/led_control_gpio/three_tabs.png)
**Figure 15:** Diagram, address editor and address map tabs
- Go to the Address Editor tab
![address_editor](/images/led_control_gpio/address_editor.png)
**Figure 16:** Address Editor
- You will see the AXI GPIO mapped to a base address (e.g. 0x4120_0000 but yours may differ). You can omit the underscore and type it as 0x41200000 in the next step.
- On the Red Pitaya terminal, type:[^7]
```bash
monitor 0x41200000 3
```
- You will see LED 0 and LED 1 light up. Play around with different values using the `monitor` command and watch the binary representation of the numbers reflected in the LED lights.

## Learner's Tip:

- Instead of changing the GPIO width from 32 to 8, how else can we cut it down to 8? *Hint: What did we do in the Led Blink tutorial?*

## References

[^1]: Arm Ltd., Learn the architecture - An introduction to AMBA AXI Version 3.0 Available: https://developer.arm.com/documentation/102202/0300?lang=en
 
[^2]: AMD. Zynq 7000 SoC Technical Reference Manual (UG585), Available at: https://docs.amd.com/r/en-US/ug585-zynq-7000-SoC-TRM/Zynq-7000-SoC-Technical-Reference-Manual 

[^3]: Xilinx Inc. (2018). PS/PL Interfaces. In: PYNQ Documentation, v2.5.1. Revision 9a0a0568. Available at: https://pynq.readthedocs.io/en/v2.5.1/overlay_design_methodology/pspl_interface.html?utm_source=chatgpt.com 

[^4]: Arm Ltd. (2025). AMBA AXI Protocol Specification. Document number: ARM IHI 0022, Issue L, Released, Non-confidential, 27 August 2025. Arm Limited, Cambridge, UK.

[^5]: AMD. AXI GPIO v2.0 Product Guide (PG144) Available at: https://docs.amd.com/v/u/en-US/pg144-axi-gpio

[^6]: AMD. Vivado Design Suite User Guide: Designing IP Subsystems Using IP Integrator (UG994) Available at: https://docs.amd.com/r/en-US/ug994-vivado-ip-subsystems/InterConnect-vs.-SmartConnect

[^7]: Red Pitaya d.o.o. 3.6.8. Monitor utility Available at: https://redpitaya.readthedocs.io/en/latest/appsFeatures/command_line_tools/utils/monitor_util.html

[^8]: AMD. Vivado Design Suite: AXI Reference Guide (UG1037), Available at: https://docs.amd.com/v/u/en-US/ug1037-vivado-axi-reference-guide




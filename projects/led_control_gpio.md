# LED Control (GPIO)

## Background

- In this tutorial, we won’t be writing any new HDL code ourselves. Instead, we’ll use IP cores — pre-built modules that Vivado provides.

## Tutorial

You’ll build a design where the Zynq Processing System (PS) can write binary values into a memory-mapped register, and those values will appear on the Red Pitaya LEDs e.g. if you write 3 LED 0 and LED 1 will light up.

1. Open up your Vivado project described in the [Setup Guide](/introduction/setup_guide.md#opening-a-new-project)
2. Open up your Block Design. 
3. Right-click in the blank workspace and select Add IP.

![add_ip](/images/led_blink/add_IP.png)

## Step 1: Add an AXI GPIO

- Insert an AXI GPIO into your block design.[^1]
- Double click on the block to re-customise the IP and adjust the settings as shown below.

![change AXI GPIO settings](/images/led_control_gpio/axi_gpio_re_customise.png)

## Step 2: Connect AXI GPIO output to LED


- Expand the GPIO output by clicking the **+** sign next to `GPIO`.  
  `gpio_io_o[7:0]` should appear.  
- Connect this to the `led_o[7:0]` port, as shown below.
- Tip: Use Zoom Fit and Regenerate Layout (toolbar icons at the top) for a cleaner view.  

![connect AXI GPIO output to LED](/images/led_control_gpio/connect_axi_gpio_to_led.png)

## Step 3: Use Design Assistance to Run Connection Automation

- You will have noticed a green banner appearing after you added your AXI GPIO. 
- Run Connection Automation, you can leave it as Auto. By default, it maps the GPIO's AXI interface clock to `FCLK_CLK0`. To avoid any confusion, you can explicitly select `FCLK_CLK0` in the properties.

![run connection automation](/images/led_control_gpio/run_connection_automation.png)

![explicitly select FCLK_CLK0 in properties](/images/led_control_gpio/explicitly_select_options_in_run_automation.png)

- After automation, your block design should look like below. 
- Note: In older Vivado versions, you may see an `AXI Interconnect` block (labelled `ps7_0_axi_periph`). In newer releases, Vivado uses `AXI SmartConnect (axi_smc)` instead. Both serve the same purpose: they bridge the PS master AXI interface peripheral slaves. [^2]

![block design](/images/led_control_gpio/led_control_block_design.png)

## Step 4: Generate Bitsream

- Click Generate Bitstream
- [Once complete, connect and upload to your Red Pitaya](/introduction/setup_guide.md#connecting-to-your-red-pitaya)

## Step 5: Write to your CPU

- In the figure below, you will see three tabs above your block design: **Diagram**, **Address Editor** and **Address Map**.
![three tabs: diagram, address editor, address map](/images/led_control_gpio/three_tabs.png)
- Go to the Address Editor tab
![address_editor](/images/led_control_gpio/address_editor.png)
- You will see the AXI GPIO mapped to a base address (e.g. 0x4120_0000). You can omit the underscore and type it as 0x41200000 in the next step.
- On the Red Pitaya terminal, type:[^3]
```bash
monitor 0x41200000 3
```
- You will see LED 0 and LED 1 light up. Play around with different numbers by writing to the CPU with the `monitor` command and observe the LEDs change. The binary representation of the numbers are reflected in the LED lights.

## References

[^1]: AMD. AXI GPIO v2.0 Product Guide (PG144) Available at: https://docs.amd.com/v/u/en-US/pg144-axi-gpio

[^2]: AMD. Vivado Design Suite User Guide: Designing IP Subsystems Using IP Integrator (UG994) Available at: https://docs.amd.com/r/en-US/ug994-vivado-ip-subsystems/InterConnect-vs.-SmartConnect

[^3]: Red Pitaya d.o.o. 3.6.8. Monitor utility Available at: https://redpitaya.readthedocs.io/en/latest/appsFeatures/command_line_tools/utils/monitor_util.html






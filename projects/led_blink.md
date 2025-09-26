# LED Blink

## Background:

Welcome to the first project! You can think of this as the "Hello World" equivalent of Python.

We will be using the Red Pitaya's internal clock to drive a counter. However, that internal clock is 125 MHz, which would mean that the LED would blink every 8ns...far too fast for our eyes to notice. The LED would just look like its on and not flashing on and off. In order to deal with that, we will use one bit of that counter to blink an LED at a visible rate. 

We will begin by adding two IP blocks in Vivado: a Binary Counter and a Slice.

- The Binary counter is clocked using `FCLK_CLK0` from the Zynq Processing System, which runs at 125 MHz. On each rising edge of this clock, the counter incremets its value by one. 

- Since the counter runs very fast, we need to pick out a single bit from its output that toggles slowly enough to be visible to the human eye. In this case, the 27th bit is used. Because the counter increments every 8ns (1/125 MHz), bit 27 flips approximately once per second:

$$
  \frac{2^{27}}{125 \times 10^6 \,\text{s}^{-1}} \approx 1.07 \,\text{s}
$$  

- To access that bit, we insert a Slice block. The Slice is configured with an input width of 28 and both Din From and Din To set to 27. This means only the 27th bit of the counter is passed to the output. 

Finally, the single-bit output is connected to the LED port. If the port shows as `led_o[7:0]`, it means it expects 8 bits (one for each of the 8 LEDs on the Red Pitaya). To use only one LED, the width can be changed by adjusting the LEFT attribute in the port properties to `0`, leaving you with `led_0[0:0]`.

With this setup, the chosen LED will now blink roughly once per second.

1. Open up your Vivado project described in the [Set-up Guide](/introduction/setup_guide.md)
2. Open up your Block Design. 
3. Right-click in the blank workspace and select Add IP.

![add_ip](/images/led_blink/add_IP.png)

## Step 1: Add a Binary Counter

- Insert the Binary Counter IP into the design.
- Double-click the block to open its configuration menu.
- Change the Output Width to 28.

![output_width_to_28](/images/led_blink/change_output_width_of_binary_counter.png)

## Step 2: Add a Slice

- Add a Slice IP (Vivado may label it as “Slice (Discontinued)” — this is fine, it still works).
- (Tip: for new projects, Xilinx recommends slicing signals directly in HDL instead of using this block. But for learning, it’s fine to keep the Slice IP.)
- Double-click and set the properties as shown below:

![slice properties](/images/led_blink/slice_properties.png)

## Step 3: Connect to the LED port

- In the block design, locate the port labelled `led_o[7:0]`
-Right-click it and choose External Port Properties.
- At the bottom, there will be two tabs: "General" and "Properties." Click on the Properties tab. 
- Change the LEFT attribute from `7` to `0`, so only a single LED line (`led_o[0:0]`) is exposed.

![led port properties](/images/led_blink/led_port_properties.png)

## Step 4: Wire the Blocks Together

- Hover over a port (e.g. the `CLK` pin of the Binary Counter). A pen icon will appear.
- Click and drag it to connect it to `FCLK_CLK0` in the ZYNQ7 Processing System.
- Connect the rest as shown in the block design below.
- To disconnect something, right-click the port and select Disconnect Pin.

![led blink block design](/images/led_blink/led_blink_bd.png)

## Step 5: Generate the Bitstream

- In the Flow Navigator on the left, click Generate Bitstream and pop up will open asking to save the block design, select to save. Then the below will open up asking to run implementation and synthesis. Click yes. 
![implementation and synthesis request](/images/led_blink/implementation_synthesis_run_request.png)
- You will then get a pop up called Launch Runs, as shown below. Click ok and wait for the bitstream to generate. 

![launch runs](/images/led_blink/launch_runs.png)


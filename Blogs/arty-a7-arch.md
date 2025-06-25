# The Arty A7 FPGA

The FPGA model we are using is [Digilent’s Arty A7](https://digilent.com/reference/programmable-logic/arty-a7/start) . The board's layout is illustrated in the diagram below and described in the table of contents.

![[Pasted image 20250624140716.png]]


The components that matter most to us are:

- DONE LED (1): lights up when the program has finished execution
- UART port (2): enables communication through MicroUSB, over UART. We’ll use this in the next post.
- Ethernet connector (3): supports incoming/outgoing Ethernet connections.
- MAC address sticker (4): is a unique ID for the Ethernet connector.
- Power jack (5): allow for an optional external power supply.
- Power good LED (6): shows whether the power supply (whether through USB or the dedicated jack) is acceptable.
- LEDs (7) and RGB LEDs (8): the design running on our FPGA can turn these on/off, allowing them to act as a basic “output” for our design. We’ll use these in this post to demonstrate that our CPU can run programs correctly.
- User Push Buttons (9) and Slide Switches (10): send their current state (pressed/unpressed, on/off) to the FPGA as input. This can be used by some designs to provide user input.
- Artix FPGA (19): is the actual programmable FPGA; everything else on this board is just peripherals and pins to help interface with / provide power to this FPGA.
- Micron DDR3 memory (20): a block of DDR3 memory that can be used by the FPGA.

## Running Hardcaml on an FPGA

The work we’ve done so far has allowed us to code in OCaml/Hardcaml, and compile that to equivalent Verilog. However, there are a few issues that remained unanswered:

1. How do we load this design onto a physical FPGA?
2. How do we map the inputs/outputs of our design to the FPGA’s actual hardware?

One option is copying our generated Verilog into the Vivado GUI, and running a [synthesis, implementation, and bitstream generation](https://www.realdigital.org/doc/bd6a53089056fc9e2888deabdfcb2a66) there. That’s not ideal, as it would add a manual component to our automated workflow. Instead, we’ll use the [Hardcaml Arty](https://github.com/fyquah/hardcaml_arty) library, which was created by [Fu Yong Quah](https://github.com/fyquah), a member of the Hardcaml team, and provides:

- A high-level API for interacting with the Arty-A7’s hardware.
- Scripts and configuration files that programmatically generate a bitstream, which can be loaded onto our FPGA. These still use Xilinx’s Vivado under the surface, but are done automatically.

Note that the `hardcaml_arty` library isn’t currently installable; for now, you’ll need to run the following command to install it:

```bash
opam pin hardcaml_arty https://github.com/askvortsov1/hardcaml_arty.git#as/make-installable
```


We decided to place all Hardcaml Arty-related files into the `arty` subfolder of our project. This consists of config files/scripts, which we copied from Hardcaml Arty [as per instructions](https://github.com/fyquah/hardcaml_arty), and code that wraps our design in the interface required by Hardcaml Arty, with relevant tests. 

We can now run the following scripts to generate the bitstream and program our FPGA. Note that we ran these from the `arty` subdirectory of our project:

```bash
source path/to/xilinx/installation/Vivado/2018.2/settings64.sh
make outputs/hardcaml_arty_top.bit
djtgcfg prog --file outputs/hardcaml_arty_top.bit -d Arty -i 0
```


## Linking In Our Design

Now that we have been able to run simple Hardcaml code on our FPGA, we want to link in our datapath. This itself is pretty easy, we can just include it into the Arty wrapper as a [hierarchical subcircuit](https://ceramichacker.com/blog/11-5x-multi-module-circuits-in-hardcaml). The challenge is testing whether the CPU actually worked on real hardware. We’ll do this by calculating a simple program (10 + 4), and validating the answer (14). But this begs the question: how do we display this value on our board?

Practically speaking, we have 3 “output” options: the LEDs (simplest but most limited), UART (relatively simple but flexible), and Ethernet (complex, but powerful). To keep things simple for now, we’ll stick to the LEDs. We’ll also treat the RGB LEDs as binary to give us 8 bits of output. We could have used the full range of r/g/b combinations, but this would be more complicated to visually decipher without major benefits. 8 bits is enough to display a trivial program’s output; in our case, 14. In a future post, we’ll restructure our design to use UART for input and output.



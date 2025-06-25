# OCaml All The Way Down

**abstract**

Did you know that Jane Street uses OCaml for, like, everything? 

Did you also know that Jane Street builds FPGA designs? 

A problem? 

Come and find out how we design and test our FPGAs. We'll have some fun (or terrible disasters) with some demos on the Arty A7 hobbyist FPGA board, with the design expressed using HardCaml, an OCaml library for creating hardware designs, and driven by an embedded software stack written in OCaml and using ports of your favorite Jane Street libraries. 

I'll round up with some thoughts on the pros and cons of writing hardware in OCaml, and talk about some ideas we would like to explore to make the process more productive in the future.

 
Presented by: Andy Ray
Andy has been designing IP cores for nearly 20 years mainly in the areas of networking and video coding. Frustration with standard RTL development processes led him to develop the HardCaml suite of hardware design tools in OCaml. Then one day while down at the pub he got an email from Jane Street wondering about some sort of collaboration, and the rest is history.

---

Can anyone who watched the video summarize? I'm interested in the subject, but don't have the time/concentration for long videos when an article often suffices.

my notes:

* HardCaml - DSP for FPGA/ASIC, * embedded SW in OCaml (running on Xilinx MicroBlaze), * HardCaml example: AXI interface, GPIO, SPI, cordic, * FPGA: LUTs and regs, BRAM, DSP,

skip to 9:40 if you are familiar with FPGA, skip to 17:50 if you know HDLs, go to 37:10 for a demo

* HardCaml datatypes: registers, wires (can be used before they are assigned), memory, ...

* API: arithmetic, logic, `pipeline`, `reg`, bit selects, `width` --> useful for HW generation, `inst` - use VHDL and Verilog in HardCaml

* signed and unsigned types: checks at compilation, * `bits` = non-synthesizable, `signal`= synthesizable, * `Always` ~ Verilog `always` - used mainly for FSM

* built-in cycle-accurate simulator: supports multiple process, MicroBlaze simulator (instruction set simulator)

* terminal-based waveform viewer and VCD export

* formal verification: SAT solvers - boolean function --> SAT/UNSAT

* software: Ocaml -> bytecode -> Ocaml interpreter for MB, C-stubs in Ocaml library, no signals, no threads, no Unix

* code for HardCaml available on Jane Street GitHub


---

# Slide 1


Design FPGAs in OCaml, Embedded software in OCaml [O]
Soft CPU - vendor IP,  Back end toolchain [X]

The heart of this talk is OCamlhalf the way down. More specifically, what we can do is design hardware, FPGAs, ASICs in OCaml using a DSL called Hardcaml. 

 What they don’t have, at least for this demo, is a soft CPU in OCaml and designed with Hardcaml, so they’re using some vendor IP for that instead. Then there’s the backend toolchain, so that’s like the Xilinx FPGA place and route tools, and quite frankly, life’s too short.

The Demo starts with a little state machine for how this can go. It goes all right on the right, but you will note there are more failure cases.

![[eventstatemachine.excalidraw]]


What happens When I pressed that button ? How did I run a hardcaml program on a FPGA?

FPGA has this vendor CPU, the Microblaze, that’s attached to 256 meg of DDR3 memory but **I also wanted to try to run it on a  softcore RISC-V CPU** to run a hello world program.  There’s a little timer core in there which generates interrupts up to the CPU. Down on the right is the Hardcaml design. That’s attached to the CPU via this avimux thing. That’s got the registry interface in it, as well.



**Blinker program**
Then that attaches to these three IP cores.

*TL;DR: IP cores let you outsource parts of your design (onboard CPUs, video processors, mp3 decoders etc) to other people. They design and make available (for purchase, or for free) RTL descriptions of these modules that you can then use in your design. So, for instance, you can simply download an ARM microprocessor, rather than having to code your own from scratch. This encourages companies like ARM to make useful, general-purpose IP cores so that people will buy them.*



There’s a digi-LED driver, that’s for the strip across the front here. There’s a SPI core, which is the standard communications interface between low-bandwidth hardware, and that’s attached to a light sensor, which is connected to the board here. 



*SPI stands for Serial Peripheral Interface. It is a synchronous serial communication interface, primarily used in embedded systems for short-distance communication between integrated circuits. It's a widely used protocol for connecting microcontrollers to peripherals like sensors, memory chips, and displays*



Finally, I stuck a little CORDIC core. CORDIC is a clever math algorithm for hardware that calculates things like sine, cosine, hyperbolic functions, and so on, but it just does it with adds and shifts.

![[statemachineexpanded.excalidraw]]


Where all of that OCaml came from is the software that we’ll be running. 

Once the FPGA loaded, it started with this custom bootloader, which then loads a program from SPI flash and copies it into DDR memory.  I should say, those LEDs you saw flashing were actually driven by the bootloader, so as it loads every something like 64K, it increments a counter and then outputs it on the LEDs.

---

# Slide 2


**Quick Intro to FPGAs**

![[FPGAarch.excalidraw]]


Quick intro to FPGAs. They’re basically a grid of these things. The black boxes are LUTs in a register … This is an abstraction, it’s not really like this, but it’s kind of like this. They’re connected in a grid pattern, and they have local routing between each other. This local routing is very fast, because it’s dedicated routing. 

The blue boxes are switch boxes. They take the outputs of LUTs and then route them up into what’s called global routing, which are resources that allow you to move signals all the way across the chip. At this sort of scale, I would say this grid goes about three miles that way and three miles that way, so they’re big things.


![[4inputLUT.excalidraw]]

---

A **LUT**, which stands for **LookUp Table**, in general terms is basically a table that determines what the output is for any given input(s). In the context of combinational logic, it is the **truth table**. This truth table effectively defines how your combinatorial logic behaves.

In other words, whatever behavior you get by interconnecting any number of gates (like AND, NOR, etc.), without feedback paths (to ensure it is state-less), can be implemented by a LUT.

The way FPGAs typically implement combinatorial logic is with LUTs, and when the FPGA gets configured, it just fills in the table output values, which are called the "LUT-Mask", and is physically composed of SRAM bits. So the same physical LUT can implement Y=AB and Y=AB', but the LUT-Mask is different, since the truth table is different.

You can also create your own lookup tables. For example, you could build a table for a complex mathematical function, which would work much faster than actually calculating the value by following an algorithm. This table would be stored in RAM or ROM.

This brings us to viewing the LUTs simply as memory, where the inputs are the address, and the corresponding outputs are the data stored in the given address.

Here's a snapshot from [FPGA Architecture](https://www.altera.com/content/dam/altera-www/global/en_US/pdfs/literature/wp/wp-01003.pdf "FPGA Architecture") by Altera:

![enter image description here](https://i.sstatic.net/0htLQ.png)

A two input LUT (lookup table) is can be represented generically like this:

![enter image description here](https://i.sstatic.net/brFOV.jpg)

A LUT consists of a block of SRAM that is indexed by the LUT's inputs. The output of the LUT is whatever value is in the indexed location in it's SRAM.

Although we think of RAM normally being organized into 8, 16, 32 or 64-bit words, SRAM in FPGA's is 1 bit in depth. So for example a 3 input LUT uses an 8x1 SRAM (2³=8)

Because RAM is volatile, the contents have to be initialized when the chip is powered up. This is done by transferring the contents of the configuration memory into the SRAM.

The output of a LUT is whatever you want it to be. For a two-input AND gate,

```
Address In ([1:0])      Output

     0  0                 0
     0  1                 0
     1  0                 0
     1  1                 1
```

For your second example, only the truth table changes:

```
Address In ([1:0])      Output

     0  0                 0
     0  1                 1
     1  0                 0
     1  1                 0
```

and finally, A xor B:

```
Address In ([1:0])      Output

     0  0                 0
     0  1                 1
     1  0                 1
     1  1                 0
```

So it is not the same LUT in each case, since the LUT defines the output. Obviously, the number of inputs to an LUT can be far more than two.

The LUT is actually implemented using a combination of the SRAM bits and a MUX:

Here the bits across the top 0 1 0 0 0 1 1 1 represents the _output_ of the truth table for this LUT. The three inputs to the MUX on the left a, b, and c select the appropriate output value.

![enter image description here](https://i.sstatic.net/5DgI8.jpg)


---


Within each of these LUTs, they’re basically formed of this red thing here, which are 16 static RAM cells. The values in those cells are set up at FPGA load time, and they can be basically anything. The LUT inputs then drive the select of this multiplexer. Basically, the four LUT inputs can address any one of these 16 cells. With that architecture, you can basically implement any four-input Boolean function you like. I should mention that modern FPGAs tend to use six-input LUTs these days, but that was a lot easier to draw. Coming out of that, you have an optional register. Every LUT has a register. The value that’s computed can be registered. Registers are kind of free in FPGA fabric for that reason, but you can also bypass the register, and that allows you to build bigger combinational functions.


Just to demonstrate how you set functions up here, if you wanted to build a four-input AND gate, what you would do is set everything to zero except the last entry, which we’d set to one. That means, when these address 15, so they’re all high, it would output this cell, which is a one, and that would be an AND. Similarly, an OR has a zero here and all ones here.

---

# Slide 3

**What else are there in FPGAs?**

- BRAMs

	I think the most important resource is probably block RAM. They’re RAMs of around 10 kilobits up to … The latest ones have like 288 kilobit RAM blocks in them. You have in the order of hundreds up to tens of thousands of these RAMs in modern FPGAs. What’s interesting about that is if you try and add up the bandwidth to all these RAMs together, it’s ridiculous, it’s absolutely enormous, and using that appropriately is why FPGAs can be pretty fast.

- DSP Blocks
	Another resource are DSP blocks. They’re basically multipliers made out of actual silicon rather than the fabric. These days, they’re a lot more complicated than that. They’ve got pre and post-adders, subtractors, and so on for various specific applications. They’re also set up to implement floating-point adders and multipliers efficiently.

- Clocking resources
	Then you’ve got your clocking resources. These are like PLLs and they used to be called DCMs, I think they’re MCM-somethings these days. Basically, they can take a clock signal in and then produce like six different clock signals out, which are some multiplication and division factor of that input clock, and also phase shift it, and also, they do some magic, I’m not quite sure how this works, where they can align the clock across the entirety of the silicon, because there are propagation delays there, and they matter. Basically, when you clock logic, you have to do it in a specific way in an FPGA using these clocking resources. There’s a whole big manual about configuration of them. This design, for example, is loading out of a little flash. Commonly, we’ll do that. Another thing that’s worth noting is most FPGA boards these days come with a USB interface which allows you to program the FPGA from your PC. That’s really useful.

- Configuration

- PINs
	Then there are the pins on the FPGA. These are complicated beasts with an extremely large manual. I was talking to a Xilinx guy once, and he was saying that in top-end FPGAs with these huge serializers, like 50 gigabit serializers, half the die is now set aside for just the pins. It’s crazy. 

- Hard blocks PCIe, MACs, ARMs

	Over the last 10 years, the FPGA vendors have started putting lots of what they call hard blocks, so it’s actual hard silicon for specific functions like PCI-E and gigabit Ethernet, and more interesting, perhaps, there are a bunch of FPGAs which have ARM cores on them, pretty high-end stuff, similar to your mobile phone, so quad ARM running at 1.2 gig or something.


---


# Slide 4

**Quick Intro to RTL**

A quick intro to RTL. There was a cool lecture I saw a while ago, I thought I’d point this out, which shows how the abstractions build on each other. I think it’s pretty useful to understand where this stuff fits in the general scheme of things. We’re living down towards the bottom. Really, the combinational logic and the clock logic is what FPGA design is mainly about. Combinational logic is things like adders, one-hot decoders, one-hot encoders, a bunch of standard RTL design blocks. You stick them together with clock logic, which are basically registers, but also these RAMs. These days, with the advent, especially, of soft CPUs, you’ll tend to get into more embedded programming as well to get these things running, so you start thinking about things like instruction sets and operating systems.

### Level of Abstraction?

```markdown
**MIT 6-002**

• Maxwells equations
• Lumped circuit abstraction
• Analogue amplifier
• Boolean gates
• Combinational logic ✓
• Clocked logic ✓
• Instruction set✓
• Operating system✓
• OCaml ✓ ✓ ✓
```


Instruction set’s an interesting one. You might think that’s just for a CPU, but you’ll also build programmable state machines, which have a kind of instruction set, so it kind of overlaps there, as well. In the operating system world, you may not be running Windows or Linux on this. Well, you can, but there are RTOSs that you’ll play with. You almost certainly have to get familiar with things like the C library that’s provided for the compilers. For us, we do OCaml on this, as well. Why not?


### Industry Standard languages

- VHDL
- Verilog / System Verilog
- [System] C[++]


The industry standard for developing RTL is VHDL and Verilog, mainly, actually probably System Verilog these days. C is also getting a big push. There’s a few variants there. There’s SystemC, which is more a modeling language and is made to look like VHDL and Verilog, although there are some crazy companies who’ve tried synthesizing that, too, and there are also a bunch of tools which can take untimed C descriptions and turn them into hardware with fairly varying results, I would say. One thing that I always found interesting is these languages have taken over hardware design, they really have, but they were actually conceived as documentation languages. They were never meant to do anything except document designs. That’s why they’re awful.

What are they?


### Event Driven languages

```c

always (a or b) x <= a ❘ b;

always (posedge clock) y <= x;

initial begin
	a <1: #5:
	b <= 2; #15:
end

```


They’re like nothing you’ve ever come across before. They’re totally weird. The idea is, you’ve got these always blocks. On the right-hand side, you have things like assignments, if statements, and so on. They feel like a little imperative language, but this stuff is always triggered on what’s called a sensitivity list. This first one will update x. If you can’t see that, it says, “x = a or b.” It will only update x when there is a change on the signal a or the signal b. That’s how you describe combinational logic. The next one down describes a register. Here, the sensitivity list is, “Change on the positive edge of the clock,” which means on the transition of the clock from low to high. That will cause x to be loaded into y, and it will hold its value for one clock cycle. That’s how you describe registers.

An RTL design is a passive thing. It reacts to signals coming into it. In order to actually do anything with this within Verilog, you need another language, which is for generating events, generating transitions. You’ve got things like this. In particular, it’s this #5 … This says, “Set a to 1, then wait for five time steps, then set b to 2, wait for 15 time steps, then do something else.” This is how you get events into the system and get it processing stuff. When you do this, this will trigger, and you might have a clock generator causing that to trigger, but none of this can be synthesized. This is the other half of Verilog. Half of it can be turned into hardware, half of it can only be used for test benches.


### Commercial simulators

• ModelSim
• VCS
• Riviera Pro

### Opensource simulators
• Icarus Verilog
• GHDL
• Verilator


The commercial simulators generally in use, mainly, I think, ModelSim and VCS these days. Riviera Pro’s still around. In the open-source world, we’ve got Icarus Verilog, which is a pretty decent Verilog simulator, GHDL, which is a VHDL simulator that’s improved massively over the last few years, and then Verilator, which is a kind of interesting tool that takes Verilog and converts it to a C++ blob which has been pretty heavily optimized, but with a number of assumptions. You can only put synthesizable RTL into it, and under those assumptions, it can create a very fast model. That tends to actually get used a lot commercially, as well. One thing that is lacking in the open-source world, and this is becoming more a problem, is any sort of System Verilog support. All the FPGA vendors have converted their IP to System Verilog, but there really isn’t anything out there in the open-source world to simulate it.


![[Pasted image 20250623190952.png]]

 *Simulating with ModelSim - MIT 6.111, this is how they designed your phone !*


This is what it looks like when you’re an RTL designer. I’ve spent far too long watching those things. First of all, look at all those buttons. That’s a lot of buttons. This thing here, this window (the signal and grey one), that’s how you debug RTL. That’s showing all the transitions going through time and all your signals. This was a tiny little simulation I did. These things get absolutely gigantic, and yet, that’s how they designed your iPhone. It’s amazing stuff.


### Verification
There are a bunch of slightly newer technologies that came around, around about with System Verilog.

• Constrained random stimulus generation and Coverage driven
	 There’s things like constrained random stimulus generation. The idea there is, you’re going to generate random data into your simulation, but you want to constrain it so that it ticks interesting paths in the simulation. For example, say you’ve got a network packet processing core, and it only works with some specific protocols. You can tell the tool that the packet should look a bit like this, and it will generate a bunch of random stuff. Alongside the coverage-driven frameworks that the tools provide, that allows you to dig into interesting parts of the design kind of automatically.
 
• Temporal assertion based property checking 
	I think more interesting are the temporal assertion-based property checkers. These things allow you to … Say an event occurs, I need some other event to occur within five clock cycles. Various temporal events like that can be described, and then, when you run the simulation, the simulator will check that these properties are always true. 
	
• Formal verification
	The really interesting stuff is formal verification. That takes the same sort of temporal assertions but will prove that the properties hold true for any trace, using extremely black magic, but it’s very cool stuff. Highly impressive tools in this area.


---


# Slide 5

**HardCaml**

Okay, so getting to Hardcaml. At its heart, it’s something like this. 


```ocaml
type t =
| Register of register_stuff
| Memory of Demory_stuff
| Mux of select * t list
| Op of t  t
| Wire of t ref
| other_stuff
```

• Hardware is represented as an OCaml datatype (Signal.t).
• Wires allow cycles
• Hardware designs are cyclic graphs with hardware operations at the nodes


You’ve got variant type with registers, memories, multiplexers, and Op hides some details, your adders, XOR, and so on in there. At this point, I think that looks a lot like a standard OCaml AST for a compiler.

Then we come to wires, where things get a little different. Wires can be created, then used, before they’re actually assigned. What this allows you to do is create cycles. I think at this point, it’s better to think of this thing as a graph of hardware nodes with connections which may be cyclic.



**API overview**

|**Class**|**Functions**|
|---|---|
|**arithmetic**|`+:` `-:` `*:` `**:`|
|**logical**|`&:` `^:` `^:` `!:`|
|**comparison**|`==:` `<>:` `<!` `>:` `<=:` `>=:`|
|**constants**|`const` `zero` `one` `ones`|
|**mux**|`mux` `mux2`|
|**bits**|`bit` `select` `concat` `@:` `msb` `lsb`|
|**misc**|`width` `wire` `--` `<=:` `inst`|
|**sequential**|`reg` `pipeline` `memory`|

In terms of the API, there are an awful lot more functions than that, but this is really the core. Everything gets built on this basic core. You’ve got general arithmetic, logical operations, comparisons, constants, muxes. Bits are kind of interesting. What we’re dealing with are not fixed-size integers or anything like that. We’re dealing with fixed-size, but variable-sized vectors, vectors of any size, be it one bit or 1,000 bits. These functions work over all these different sizes with some rules.

Down towards the bottom, we have width. Width’s an interesting one, actually. It’s worth noting that any signal you’ve got access to in a Hardcaml design, you can query for its width, and that allows you to create self-adaptive designs, just look at a width and construct the appropriate logic for that. It’s very easy to do that with this, whereas in Verilog, it’s a real nightmare. The other one is instantiation, or inst. That allows you to kind of put a hole in the Hardcaml design that you can fill in with VHDL or Verilog later. That allows you to work with third-party IP, or with vendor primitives, macros, and so on.



**API Rules**

• Unsigned / non-signed operators end in:  
• Signed operators end in +  
• Lots of checks occur at runtime

The general rules, there. If the operation is unsigned or doesn’t care about the sign, it ends in a colon. In a few cases, the operator ends in a plus. That’s where you have to care whether it’s signed or unsigned. Multipliers are an example of that. An unsigned multiplier has a different architecture to a signed multiplier, similarly with comparisons. That tends to mean there are quite a few checks that occur at runtime, although in hardware parlance, it’s not simulation time or anything like that, it’s at elaboration time. It’s when the graph that describes the hardware is fully constructed. 

**Signal widths are strict**  

```ocaml
utop # *const '11' +: const '101';;   
Exception: ("Operands to +: must be the same width'  
(11 101)).
```


We’re quite strict about things like signal widths in the API. Here, we’re adding a two-bit constant to a three-bit constant, and it says, “No, can’t do that.”


 **Bits**
 Computes values.

```ocaml
utop $ const "10" +: const "01";;  
- : Bits.t = 11  
```

 
 **Signals**
 Records computations.

```ocaml
utop $ const "10" :: const "01";;  
- : t = Op {{s_id = 41; s.name = []; s_width = 2; s_deps = []}}  
[Const {{s_id = 40; s.name = []; s_width = 2; s_deps = []]}  
[Const {{s_id = 39; s.name = []; s_width = 2; s_deps = []]}  
[Signal.add]
```


```bash
utop # Hardcaml.Signal.(const "10" +: const "01");;
Line 1, characters 17-22:
Alert deprecated: Hardcaml.Signal.const
[since 2019-11] const
Line 1, characters 31-36:
Alert deprecated: Hardcaml.Signal.const
[since 2019-11] const
Line 1, characters 17-22:
Alert deprecated: Hardcaml.Signal.const
[since 2019-11] const
Line 1, characters 31-36:
Alert deprecated: Hardcaml.Signal.const
[since 2019-11] const
- : t =
Const
 {Hardcaml.Signal.signal_id =
   {s_id = 10L; s_names = []; s_width = 2; s_attributes = []; s_deps = [];
    caller_id = Base.Option.Some <abstr>};
  constant = <abstr>}
```



There’s a kind of technical detail in the library which is not well-loved here, but it still exists. We have this idea of the API over two different types. Bits compute values, so here we see const add 2 to 1, we see we get 3 out. With signals, you’re recording the computation. The place where this is still used is, we design hardware using signals, so it describes the computation or the graph that you want, and then, when we go to simulation, we use bits to drive values into the ports.


**Sequential logic**  

• Clocks  
• Hardcaml primitives; reg, memory  
• Build up RAMs, state machines

Sequential logic in Hardcaml, there are actually only two primitives. You can create registers, or you can create a memory primitive. It’s technically an asynchronous read memory, and we use that to build up proper memory structures that can be instantiated in FPGAs, ASICs, and so on, and also build stuff to build state machines.


**Always DSL**

- A DSL on top of Hardcaml primitives that models Verilog Always blocks  

```ocaml
Always compile {  
    statements.switch {  
	    Start, [  
		    when..start {  
			    statements.set.next Run;  
    ]  
];
```

for example, the always DSL. This models Verilog always blocks as I showed you before. It’s a little bit different, but it basically provides a way of assigning to registers or even combinational values under guarded conditions, so you’ve got ifs, switches and so on. You can decide when to assign values to registers. We tend to use this mainly for state machines, which are like a switch statement with the case listed, and on the condition, you can check the next state of the state machine.


**RAMS**  

• Modelled with Hardcaml memories  
• Inferred by the Vendor synthesis tool
• Need primitives for more complex blocks


As I was saying, Hardcaml just provides a simple memory primitive, and then it builds proper memories by applying a register to the address port or the output port of the memory. That’ll create a read-before-write or a write-before-read memory. That should then be inferred by the vendor synthesis tools. They say it does. It doesn’t always. As is actually the case with Verilog, as well, there are more complicated primitives. A standard true dual-port RAM can’t actually be described properly in Verilog and automatically synthesized, so you have to use macros, same with Hardcaml, you have to use macros for that.

---

### Slide 6

![[FIRfilter.excalidraw]]

A quick example. This is kind of what I consider the Hello, World of hardware design, a little FIR filter. Across the top, we’ve got four registers, the input enters at the left, and then on each clock cycle, it steps down the pipeline, so effectively, you have a window of the last four samples. These are then multiplied by a constant coefficient and added in a tree of adders. 

In Hardcaml, that would look something like this:

```ocaml
let rec shift_registers n d_in =
    if n=0 then [i]
    else
    let r = Reg.reg ~e:vdd d_in in
    r :: shift_registers (n-1) r:: 
    val shift_registers : int -> signal -> signal list

let fir_filter coefs d_in =
    shift_registers (list.length coefs) d_in
    |> List.map2-cnn coefs -f:{ *+ }
    |> tree 2 [readecs {-:}}]:; 
    val fir_filter : signal list -> signal -> signal
```

This is constructing those registers across the top, returning the window of registers as a list, and then we just instantiate the registers, map across the coefficients and the registers, performing a signed multiplication, and then we’ve got this tree function, which will take a list in and create a balanced tree down to one value. That can be any binary operator.

**Testbenches**
* Built in cycle accurate simulation

For testing, Hardcaml has built-in cycle-accurate simulation. That’s the same as Verilator, which I mentioned before. It doesn’t support event-driven simulation, at least not at the moment. It looks something like this.

```ocaml
module Simulator = Cyclosim.With.interface
    {Inputs} {Outputs}
let sim = Simulator.create Design.create in
let inputs = Cyclosim.inputs sim in
let outputs = Cyclosim.outputs sim in
```

```ocaml
inputs.a := Bits.conat '11010110';
inputs.b := Bits.conat '1101';
Cyclosim.cycle sim:
Printf.printf '%i\n' {Bits.to.int {{outputs.c}}
```

You build the simulator, and then the thing at the bottom is where the real stuff happens. You set input ports to the circuit, you call cycle, which steps the simulation on one cycle … It evaluates one clock cycle, and then you can read the outputs of your circuit, and you do that again and again, stepping through the states you want to test.

**Hardcaml_step_testbench**  

• Problem  
	• Multiple independent inputs/output ports 
	• Only one control flow hardcaml_step_testbench  

• `hardcaml_step_testbench`
	• `async` for hardware testbenches  

• Integration with Microblaze simulator

There is a problem with that, though. That’s fine for simple hardware, but most hardware isn’t that simple and will have multiple independent input and output blocks of signals. You might have, for example, three FIFOs going in, a register interface, a memory interface, a bunch of other stuff, and in that model where there’s just one flow control, you effectively have to turn your whole testbench into a state machine to interact with all these different things. We’ve written this thing called Hardcaml_step_testbench, which you can pretty much think of async for hardware testbenches. It allows you to have multiple independent flow controls which can deal with different interfaces and ports on the hardware design, but still interact with each other.

A very cool feature is we have this Microblaze simulator, written, again, by good old Mark, and we integrated that with Hardcaml_step_testbench. That means we could load a Microblaze ELF file, so that’s a Microblaze program, simulate it with this instruction set simulator, which is not hardware, it’s not trying to simulate hardware, it’s simulating instruction sets, so it’s fast … I don’t know, maybe a few megahertz fast? That can actually interact with a hardware design attached over the standard bus interfaces. In this case, it’s like AXI bus for the Microblaze. That’s really very powerful. 


```ocaml
let set.a.and.b =  
    let%bind o = Step.cycle  
    { inputs with a = Bits.conat '1010101'  
    : b = Bits.conat '1110' } in  
    return o.c  
in  
let set.x.and.y =  
    let%bind o = Step.cycle  
    { inputs with x = Bits.conat '1010101'  
    : y = Bits.conat '1110' } in  
    return o.s  
in  
let%bind result.c = spawn set.a.and.b in  
let%bind result.a = spawn set.x.and.y in  
...:spawn, cycle...  
let%bind o = wait_for result.c in  
let%bind a = wait_for result.a in  
return o + s
```

This thing looks … Well, it doesn’t really matter. I think the things to note, you’ve got multiple tasks, they get spawned, you can wait for results, you can return results, and it’s all in a monad, of course it is, it’s always in a monad.

```markdown
┌Signals────────┐┌Waves──────────────────────────────────────────────┐
│clock          ││┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌──│
│               ││    └───┘   └───┘   └───┘   └───┘   └───┘   └───┘  │
│clear          ││                        ┌───────┐                  │
│               ││────────────────────────┘       └───────────────   │
│incr           ││        ┌───────────────┐                          │
│               ││────────┘               └───────────────────────   │
│               ││────────────────┬───────┬───────┬───────────────   │
│dout           ││ 00             │01     │02     │00                │
│               ││────────────────┴───────┴───────┴───────────────   │
└───────────────┘└───────────────────────────────────────────────────┘
```

For debugging, I don’t like GUIs, so I wrote a text-based waveform viewer. 

**Hardcaml Waveform support**   
  
• Interactive terminal based waveform viewer  
• VCD file output  
• GTKWave integration  
• Expect tests!

The support we have in Hardcaml … There’s an interactive terminal-based waveform viewer with scrollbars and the whole shebang, that I actually do use a lot for debugging stuff, but if you want to, it can output VCD files, which are standard Verilog waveform files, and they work with any waveform viewer, in particular GTKWave, which is an open-source waveform viewer. It’s actually pretty good. There’s also a bit tighter integration with Hardcaml simulation in that it can start a simulation, fire up GTKWave, and pump data across to GTKWave for it to be viewed while the simulation is actually happening.

A new feature is we now have waveform Expect tests, which are super cool. You may have heard we quite like Expect tests around here. They look like that. 

```markdown
let%expect_test "counter" =
  let waves = test ()
  Waveform.print waves
  [%expect {|
┌Signals────────┐┌Waves──────────────────────────────────────────────┐
│clock          ││┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌──│
│               ││    └───┘   └───┘   └───┘   └───┘   └───┘   └───┘  │
│clear          ││                        ┌───────┐                  │
│               ││────────────────────────┘       └───────────────   │
│incr           ││        ┌───────────────┐                          │
│               ││────────┘               └───────────────────────   │
│               ││────────────────┬───────┬───────┬───────────────   │
│dout           ││ 00             │01     │02     │00                │
│               ││────────────────┴───────┴───────┴───────────────   │
│               ││                                                   │
└───────────────┘└───────────────────────────────────────────────────┘
  |}]
```

What does this mean? This means that our continuous integration system is now checking Hardcaml simulations through the data dumps in the actual waveform, so when something goes wrong, you get a diff, and you can go and fix it. There are cons to this. This stuff is good for maybe 30 clock cycles before it just gets too far off the other side. That being said, it’s still incredibly useful for looking at little protocol details and having them actually written down in code. “There, this is how it should work, it should not change. If it does, you need to fix something.”


**Formal verification**  

• You can prove 20 bits by exhaustive enumeration. 
• What about 200 bits?

I thought we’d have a quick detour into the world of formal verification. Let’s say you have a 10-bit adder, and you want to prove it’s right. A 10-bit adder, why would you do that? It turns out, a lot of papers get written about adders, a lot of papers. You’ve got a 10-bit adder, 20 bits in total, so that’s a million test cases, no problem. What if you want to test 200 bits? It just ain’t going to happen that way.


**SAT Solving**

• Given a boolean function £
• Of variables xy.
• Find an assignment to the variables that makes f = 1
	• SAT returns the satisfying assignment
	• UNSAT there is no satisfying assignment


Enter SAT solvers. They’re cool things. The idea behind this is you’ve got a Boolean function, so it produces true or false of some number of input variables. You can go in and ask the SAT solver to find an assignment to the input variables that makes the Boolean function equal 1. If it can do that, it returns SAT, and it tells you what the values of the input variables were. It’s not if it can’t find one, it’s if there is no assignment, it will return UNSAT. 

**The "trick"**

• Ask the SAT solver about not f
• **UNSAT**
	•No assignment can make not f = 1
	• Implies nothing can make £= 0 ie £ is always 1. QED!
• **SAT**
	• Solution is a bug!

There’s a trick we need to play here. What we usually want is to prove that a property is true always. The trick is, we actually ask it about the negation of the function. What happens is, if the SAT solver returns UNSAT, it’s saying it can never make not f = 1, which implies that f is always 1, always 1. That’s proven. If you get SAT, the cool thing about it is it will tell you where it went wrong. It will actually tell you, “Here’s your bug, and this is what caused it.”


**Hardcaml formal verification support**

•Convert combinational circuits to SAT problem instances
• Sequential equivalency checking
• Bounded model checking with LTL property specifications

In Hardcaml, we’ve got reasonable verification support now. We can convert any combinational circuit into … Well, when I say any, any combinational circuit with one-bit outputs, so a property over a combinational circuit can be converted into a SAT problem. We’ve got a little framework for sequential equivalence checking. The way we use that is we were porting some Verilog to Hardcaml, and so I had this Hardcaml, and I was like, “How do I show it’s still the same?” I used this open-source tool called Yosys, which converted the Verilog into this low-level netlist which I could then load into Hardcaml and form a SAT problem comparing the outputs. Are all the outputs always equal to each other?


The trick here is that both circuits have to have registers in exactly the same places. To lift that restriction, we have bounded model checking. I’ve got a bounded model checker. I don’t necessarily trust it yet, but it kind of works on small cases. It’s also got LTL property specification. With these things, you could compare two circuits, they could have different state, and you can ask it, “Over a trace of 100 cycles, are these things the same?” and it can prove that. It can’t prove anything about the 101st cycle, that’s why it’s called bounded model checking, but nonetheless, it’s a very powerful technique.

---

# Slide 7

**Software Stack**

* Running on what?
* Built How?



**Microblaze**
• Xilinx Soft CPU
• Standard RISC architecture
• GCC cross compiler


This software stack. As I mentioned, it’s running on the Microblaze. That’s a Xilinx soft CPU built out of the fabric of the FPGA. That is a standard RISC thing. The most important thing is it’s got a GCC-based cross-compiler. 



**OCaml byte code**
• Interpreted by ZINC abstract machine
• Cruntime
• Cross compile the C runtime


What are we running? It’s the OCaml bytecode. In the OCaml world, there are two ways to compile code. You can either compile it to native code if the compiler supports your architecture, which it doesn’t, or you can compile it to bytecode, which is generic and runs on a C-based interpreter called the ZINC abstract machine.


**OCaml libraries**
• cma files built as normal on x86
• Compile a top level executable with -output-obj
• Gives a C-code file with the byte code embedded as an array


The first step was for me to cross-compile the OCaml C runtime with this interpreter in it. Then we need to get all the libraries. As it turns out, you can just build bytecode CMA libraries for OCaml, just like normal on x86. In fact, they’ve been built by a 64-bit compiler, even though they’re being run on 32-bit. You then construct a toplevel executable, bytecode executable, and pass this magic flag called -output-obj. What that makes OCaml do is write a C file which has an int array with all the bytecode for your program embedded in it, and then you can compile that with your cross-compiler.



**OCaml library C-stubs**
• Need to link with all OCaml C-stubs
• Just need to cross compile with microblaze gcc compiler
• Not yet sure about a portable way to do this with opam • Surprisingly few of them

The tricky bit is you need to link all the C stubs that were referred to in those OCaml libraries into your final application to run on the processor. You just need to compile them, but you’ve got to go and find them within the source code tree. It’s not that easy. Anyway, there actually aren’t that many of them. I think I found maybe 40 C files in total.



**What doesn't work**
• No signals
• No threads
• No Unix
• 80% because of the FPGA C-runtime, 20% lazyiness.

What isn’t working? Signals don’t work. That’s actually quite annoying. I think I could probably make that work. There are no threads. You could get around that if we ran this thing on top of an RTOS, but I bet the threads that an RTOS provides are not quite what OCaml expects. I’m not sure. I’m not sure if it’s worth it. No Unix … Yeah, there’s no files, so how do you expect Unix? 


**What does work**
• Base
• Core_kernel
• Async_kernel
• The OCaml top level

What does work? Base, Core_Kernel, Async_Kernel, they’re all up and running in this thing. Rather neatly, the OCaml toplevel runs in that. That is the OCaml compiler, also running on the FPGA.



**Caveat emptor**
• Min executable size about 400 KB 
• With Janestreet libraries - 1-4 MB 
• With top level-2-25 MB
	•  Need to link everything referenced 
	• Need `cmi` files for runtime compilation

Why wouldn’t you want to do this? Well, the smallest executable you can get out of this is about 400K, which, in this world, is enormous. When you start layering on the Jane Street libraries … Actually, it’s not quite true, one meg, it’s actually nearer about 700K. When stuff starts linking in, you’re using more features, it quickly gets up to the four, five meg range. Then when you put the toplevel in with all this stuff, well, it added up at 23 megabytes. Ouch. It’s understandable in a way. When you’re doing the toplevel, it can’t optimize out stuff you’re not using, because you could potentially use anything. That accounts for about seven meg of it. It turns out these CMI files which are used by OCaml for type checking the code you write are enormous. They took up about 16-odd meg … Just details.

---

# Demo Time

• Pulse width modulation - Turn LEDs on and off really fast to give impression of different intensities.

• Cylon or Kit? - Strobe each LED on from left to right and back again, while reducing the intesity by half to create a tail.

•  Light sensor - Read the light sensor, produce an average and display result on the LED.

• Cordic - Use the cordic core to calculate **cosine** and display the function on the LEDs.

• Trippy Async threads - Use various Jane Street libraries (Base, Core_kernel, Async_kernel) to run multiple threads each controlling a part of the LED strip.


Let’s see if we can get some demos working, then. Okay, so what do I need to do … 33857 

```bash
rlwrap nc localhost 33857
| let a = 1;;
```

What just happened there? The processor is redirecting its standard in and standard out through this USB cable up to a driver on my laptop, which is then bridging that across onto a TCP socket, which I’m then attaching to with netcat. Anyway, stdin, stdout there goes down to there. When I type that … I typed, “let a = 1.” That was compiled by the FPGA, or at least by OCaml on the FPGA, and it showed the result.

```ocaml
Open Regmap

(* Do nothing for some number of cycles te 10,000. *)  
let delay_loop count =  
    for i=0 to count do () done  
;;  

(* Pulse width modulation *)  

let pwm_demo -iterations =  
    let pwm loops on_for =  
    for i=0 to loops do  
    for j=0 to on_for do  
    Leds.set_rgb 0x02020202;  
    done;  
    for j=on_for to 100 do  
    Leds.set_rgb 0x0;  
    done;  
    done  
in  
for _ = 0 to iterations-1 do  
    for on_for=0 to 10 do  
    pwm 100 (on_for * 10)  
    done;  
    for on_for=9 downto 1 do  
    pwm 100 (on_for * 10)  
    done  
done  
;;  
```

```bash
pwm_demo ~iterations:10;;
```


Hopefully you can see that. This is a pulse-width modulation demo. Let me run that again. The basic idea is, these LEDs can only ever be on or off. Pulse-width modulation is a fancy way of saying we turn it on for a little bit and then turn it off, but do it really fast so it looks like it’s half intensity instead. We do that over and over again with intensity going up and down so the LEDs breathe, as Brian likes to call it. In terms of code, it’s just a bunch of for loops setting the LEDs to these values, either on or off.

```ocaml
(* Cylon (or kit) *)

let cylon ~iterations ~speed =  
    let dtrn = ref 1 in  
    let index = ref 0 in  

Led_strip.enable ();  
Led_strip.set_count 30;  

let next_led () =  
    if !index = 0  
    then dtrn := 1  
    else if !index = 29  
    then dtrn := -1;  
    index := !index + !dtrn;  
    !index  
in  

for i=0 to (iterations * 58) do  
    delay_loop (100000 / speed);  
    for index=0 to 29 do  
    Led_strip.set_int ~index  
    ((Led_strip.get_int ~index land 0xfeefe) lsr 1)  
    done;  
    let index = next_led () in  
    Led_strip.set_rgb ~index ~r:255 ~g:0 ~b:0  
    done ;  
    Led_strip.disable ()  
```

```bash
cyclon ~iterations:10 ~speed:3;;
```


Is it working? Yay! I was so excited when I got the strip. This is all I wanted to do with it. That’s basically all I’ve been doing for weeks. It’s great. That’s the little Cylon demo. All that’s happening there is we’ve basically set the next LED to the left until we get to the end, and the next LED to the right, and there’s a little busy loop after setting each LED so it doesn’t go so fast you can’t see it. Every time, just before we set the LED, we run through the entire strip and halve the current intensity of all the LEDs. That gives you the tail running behind it.

```ocaml
(* Light sensor *)

let read_sensor () =  
    Spi.start();  
    while not (Spi.done_ ()) do () done;  
    (Spi.get_data () lsr 4) land 255  

};

let avg_sensor_readings n =  
    let rec read sum m =  
    if m=0 then sum / n  
    else read (sum + read_sensor ()) (m-1) in  
    read 0 n  

};

let light_sensor -iterations =  
    Led_strip.enable();  
    Led_strip.set_count 30;  
    Spi.set_data -bits:16 ~data:0;  
    for i=0 to iterations do  
    let intensity = avg_sensor_readings 500 in  
    let max_Led = min (intensity lsr 3) 29 in  
    for index=0 to max_Led do  
    Led_strip.set_rgb ~index ~r:0 ~g:255 ~b:0;  
    done;  
    for index=max_Led to 29 do  
    Led_strip.set_rgb ~index ~r:0 ~g:0 ~b:255;  
    done;  
    done  
```

Okay, so that’s what’s called a Pmod that’s been attached to this board. If you go to the Digilent website, there’s like 50 or 60 different Pmods you can buy and attach to this and have a play with. This particular one is, obviously, a light sensor. That’s read via the SPI protocol I was talking about before. What happens here is, this is doing the SPI transfer, so you start the SPI hardware, wait for it to be done, and then you read out the data. The way the data is formatted, it’s basically the bottom four bits are nonsense, and then eight bits are the current intensity value. I’m not entirely sure if it’s picking up the flickering of the lights or it’s just not always very accurate, so you have to average it for quite a while to get a nice, sensible value. I think it’s doing it about 500 times, taking an average, and then, as you can see, it would show the intensity of light in green on the LED strip. That’s what’s done here. You basically take the eight-bit value and shift it down to 32, because this has got 30 LEDs on it.


```ocaml
Open Base

let cordic_scale = Float.of_int (1 lsl 16)
let gain    = 1.647
let igain    = Float.(1. / gain)
let pi    = Float.pi

let float_to_fixed f =
   let f = Float.(f * cordic_scale) in
   if Float.(f < 0.)
   then Float.(to_int (f - 0.5))
   else Float.(to_int (f +. 0.5))

};

let fixed_to_float f =
   let f = Float.of_int f in
   Float.(f / cordic_scale)

};

let cordic_cos_sin () =
   Cordic.config Cordic.Mode.Circular Cordic.System.Rotation;
   Cordic.set_x (float_to_fixed igain);
   Cordic.set_y 0;
   Cordic.set_c 0;
   Cordic.set_y 0;
   Cordic.set_c 0;
   (fun angle ->
       Cordic.set_z angle;
       Cordic.start();
       while not (Cordic.done_()) do () done;
       Cordic.x(), Cordic.y())
};

let test_cos_sin arg =
   let cos_sin = cordic_cos_sin () in
   let cos,sin = cos_sin (float_to_fixed arg) in
   let cos,sin = fixed_to_float cos, fixed_to_float sin in
   (cos, float.cos arg),
   (sin, float.sin arg)
};

let p_1_2, p_2_2, p_3_2, p_4_2 =
   float_to_fixed float.(pi / 2.),
   float_to_fixed pi,
   float_to_fixed float.(3. * pi / 2.),
   float_to_fixed float.(2. * pi)

let calc_cos () =
   let cos_sin = cordic_cos_sin () in
   let f arg = fst (cos_sin arg) in
   fun arg ->
   (* cos x = cos -x*)
   let arg = if arg < 0 then - arg else arg in
   (* reduce to [0 .. 2pi] *)
   let arg =
   let rec f arg = if arg > p_4_2 then f (arg - p_4_2) else arg t
   f arg
   in
   (* check quadrant *)
   if arg < p_1_2
   then f arg
   else if arg < p_2_2
   else if arg < p_2_2
   then - (f (p_2_2 - arg))
   else if arg < p_3_2
   then - (f (arg - p_2_2))
   else f (p_4_2 - arg)

};

let display_cos ~frequency ~phase ~iterations =
   led_strip.enable();
   led_strip.set_count 30;

   let cos = calc_cos () in
   let incr_phase = float_to_fixed phase in
   let incr_per_led = float_to_fixed Float.(2. * pi / (30. / frequent)

   let start, pos = ref 0, ref 0 in
   for i=0 to iterations - 1 do
   pos := !start;
   for index=0 to 29 do
   let cos = (cos !pos) asr 10 in
   if cos < 0
   then Led_strip.set_rgb ~index ~r:0 ~g:(~cos) ~b:0
   else Led_strip.set_rgb ~index ~r:cos ~g:0 ~b:0;
   pos := !pos + incr_per_led;
   done;
   start := !start + incr_phase;
   delay_loop 40000;
done
;;
```

```bash
display_cos ~frequency:1.0 ~phase:0.2 ~iterations:5;;
```

This one’s a bit more complicated and not as much fun, I’d say, but anyway … Here, we’re using the CORDIC core that I mentioned that’s in Hardcaml, as well. The CORDIC core is actually pretty general. It’ll do all sorts of functions from atan to hyperbolic functions to coordinate rotations. Here, I’m using it the simplest way, which is just to calculate cosine. This is how you configure it. You set the four inputs, you set the x, y and c inputs to these fixed values, and then, each time you want to compute cosine for an angle, you set z, start it, and then just wait for the core to be finished. It actually computes cos and sin at the same time, but we’re only interested in the cos one.

It then takes that value … Oh, yeah, there’s this horrible bit of logic. The algorithm will only converge in … Actually, it’s like minus pi over 2 through to pi over 2. What I have to do here is take the argument … Basically, I reduce it down to 2 pi, and then for each quadrant, you can use symmetry to work out how to project it into the quadrant that converges. Then what we’re doing is just displaying the cosine function on the LEDs, so use the intensity of the LED to say the value, except if it’s negative … I think, was it green and positive, it’s red? Then you run through various situations where you just increment the phase by a little bit, and that makes it walk across the LED.


Now we’ve got my favorite one … Okay, this takes a little while to load. 

```ocaml
open Async_kernel
let span_ms = Core_kernel.Time_ns.Span.of_int_ms;
let reduce_intensity ~ms =
    let rec f () =
    for index=0 to 29 do
    Led_strip.set_int ~index
        ((Led_strip.get_int ~index land 0xfeefe) lsr 1)
    done;
    after (span_ms ms) >>= fun () -> f () in
	f ()
;

let strobe ~min ~max ~ms ~colour =
let rec loop index direction =
    Led_strip.set_int ~index (Led_strip.get_int ~index lor colour);
    after (span_ms ms)
    >>= fun () ->
    if max = index
    then loop (index-1) (-1)
    else if min = index
    then loop (index+1) 1
    else loop (index+direction) direction in
	loop min 1
;

let go () =
    let t = Async_kernel.Async_kernel_scheduler.t () in
    while true do
    Async_kernel.Async_kernel_scheduler.run_cycle t
	done
};

let trippy_async_demo () =
    Led_strip.enable ();
    Led_strip.set_count 30;
    Deferred.all
    [ strobe ~min:0 ~max:14 ~ms:30 ~colour:0x00ff00
    ; strobe ~min:15 ~max:29 ~ms:60 ~colour:0x0000ff
    ; strobe ~min:0 ~max:29 ~ms:90 ~colour:0xff0000
    ; reduce_intensity ~ms:45 ] |> ignore
```

```bash
trippy_async_demo ();;
```


This is async running on an FPGA, which is just crazy stuff. What we’re going to do is something similar to the Cylon demo that I did at the start with the LED bouncing across the strip, except what we’re going to do is fire up three separate strobes that run from a min to a max LED and just bounce in a range with a specific color at a specific speed. I’m going to fire up three threads. One is going to go on the left half of the LED strip, one’s going to go on the right half, and then another one’s going to go across the full strip. Then we run a fourth task, which is doing this halving of the intensity to display the trail. I’m so happy. Yeah, to do this demo took about 200,000 lines of OCaml.

---

# Slide 8



**Opensource release**
• New version of Hardcaml released any day now-will live under Jane Street repos.
• 2 further releases cleaning up various old libraries and adding some new ones planned in coming 3 months.

**Demo code**
• Code is a bit of hack
• But I've tidied up most of it now
• Need to set up some special OPAM packages which cross compile the required C-code.
• Will try to get something out in the next couple of weeks.


Getting this stuff released, as ever, is all last-minute, but we are, I think, maybe a day, maybe two, from getting the latest version of Hardcaml released under the Jane Street repos. It’s a pretty major upgrade over the last open-source release, especially in terms of testing. It’s got so much more testing now. It’s in much better shape. We’re planning on a couple of further releases, some to tidy up and release some of the older libraries that I’ve got, like the wavterm stuff, and then there’s some new stuff, like Hardcaml_step_testbench, the Microblaze simulator, which we’ll get out, as well. I say three months … We’ll see, but hopefully. This demo code I’d like to release, as well, but oh, man, it’s a hack, it’s awful, but I’ve started tidying it up.

The really tricky stuff if we want to put this out on OPAM is figuring out how to compile or cross-compile the C code. I’m thinking of setting up opam-remote or a remote repository with packages which just take the standard Jane Street async, download it, compile it, and then do a patch which will compile the Microblaze library, as well, and stick it somewhere that it can be used. I think it’s actually doable. I will try and get something out in the next week or two on that front so you can have a play with this stuff.

---

**END**

Q: The OCaml toplevel that you had running on there, that was running on an IP core that was flashed onto the FPGA, was that right, or is that not right?

Andy: No, that’s right. It’s running on a soft CPU. It’s this thing called a Microblaze, which is basically a five-stage RISC CPU that we’ve included in our design to run C code.

Q: I guess I’m not sure: why would you want to do that, rather than run on the ARM processor, which is probably a lot faster and probably uses a lot less power?

Andy: There isn’t one.

Q: There isn’t one?

Andy: There’s no processor on this board. The FPGA we’re using here doesn’t have a processor. There are FPGAs that do have processors, but we don’t have one.

Q: Why wouldn’t you get one of those boards if you wanted to run an OCaml toplevel?

Andy: You could, I would say. If you had an FPGA which had one, you would use one of them, for sure. If you don’t, then you’re down this rabbit hole. 

A: Surprisingly, the really high-end ones we often use in the industry, in finance, don’t have a system-on-chip design.

Q: Wouldn’t it be better if you really wanted to run bytecode or just instructions you would want a network interface communicating with an external CPU?

A: If you want to know what’s going on in the circuit itself, it’s kind of hard to do anything else.

Q: You’re going through a bit of trouble in order to execute the OCaml bytecode on top of the Microblaze. Have you considered, it might be a bunch of work, but executing the bytecode directly in a soft design?

Andy: Yeah. Oh, yeah. I started looking at that, actually. It was a while ago, now, but I did write a little state machine that could execute about three-quarters of the OCaml bytecode VM directly in hardware.  I would love to get back and finish that. It’s on GitHub, hardcaml-zinc, there’s a project for that. The place I got a little bit stuck is: yes, you can do that, but actually, there’s still an awful lot of OCaml that’s implemented in C, so you need a RISC CPU anyway. I started doing that, it was like, “Oh, I need a RISC CPU,” so I started writing that, and then I got a job. 

Q: There was one slide where you said, “There’s no Unix here.” Could you just elaborate a bit on what you mean by that?

Andy: The Unix module in OCaml deals with things like filesystems and networking. The soft CPU here, it’s not connected to a filesystem at all, so none of that stuff … You can’t fopen anything, because there’s nothing there. At the moment, we don’t have any network … Well, that’s not true, actually. In reality, that design does have a network interface to this little RJ45 connector. I just haven’t got it working yet. Potentially, we could consider doing networking stuff, but we’d then have to mint up a sockets emulation layer for it to work with Unix, and chances are, we’ll just use low-level APIs instead.


Q: I’m slightly confused. Is the Hardcaml DSL being compiled down or synthesized down into lookup tables and routes, or is it just running It’s just always running as bytecode?

Andy: No. The Hardcaml DSL is used to describe a hardware design as an in-memory data structure on your PC. That is then processed and turned into Verilog, and that’s where it’s handed off to the backend tools. I did say I’d speak about pros and cons of this stuff. That is one of the big cons of using Hardcaml. The Verilog that’s generated is entirely unreadable. Nobody wants to go near that. That’s my job. That’s what I have to do. Because the backend tools are designed for Verilog and VHDL, there’s a tension between getting the right names from OCaml all the way through to the netlist so you can do time enclosion, see the right names on your nets, and so on. It is doable, but you have to be careful. 

Q: Any comparisons between some of the stuff you’ve done here and Bluespec, because that’s like the Haskell. In terms of design

Andy: Yeah, sure. Maybe $100,000? Yeah, it’s completely different. Hardcaml, in terms of abstraction, is pretty much the same as Verilog and VHDL. It treats designs the same way. It doesn’t really try and infer where stuff should happen on what clock edge or anything like that, which Bluespec does do. I like Bluespec, I like the idea of it. I’ve never actually used it, I’ve just read papers about it, but I like the idea. We’re definitely thinking about looking into a DSL on top of Hardcaml which models the atomic actions of Bluespec and seeing where that takes us. Yeah?

Q: Some of this stuff [inaudible] when I was using it, it generates Verilog code, and actually, sometimes I ran into timing problems [inaudible] how does Hardcaml [inaudible] what do you do if you run into that?

Andy: Yeah. If there’s a timing problem, there’s something wrong with your architecture. Bluespec, I think because it’s a compiler which will change your design or change your intent … not really your intent, but you know what I mean, it creates a microarchitecture for you, it’s kind of pushing you down a route where I guess it’s quite difficult to get out of that. Hardcaml is not like that. The thing you write in Hardcaml is exactly the Verilog you will generate, and therefore, the architecture you will pass to the tool. If it’s failing timing, it’s your fault. That being said, because it has to go through this translation to Verilog, the thing you must do is basically apply in your Hardcaml design a name to every single register. That means when you get timing reports, you actually get a timing report from a name to a name. As long as you do that, it’s okay. It’s workable.


Q: Why is the Verilog that’s generated so unreadable? It seems like it maps pretty closely to Verilog, then, in the DSL.


Andy: I don’t know if I have a good answer for exactly why it happens. I think most of the generation systems I’ve seen that are similar to Hardcaml have the same problem. I’m not entirely sure, to be honest. It just is awful. You don’t want to go near it.

Q; Have you thought about using RISC-V?

Andy: Absolutely. That is the processor I want to replace the Microblaze with. I will eventually convince Ron that I can do this and that it’s a good idea … No, there’s a serious reason for this, actually. We’re trying to get these tools into our workflow, and getting these huge vendor toolchains as part of this is a real problem, especially when we’re upgrading all the time. It seems to me that having control over that bit of actually pretty core technology to us would mean that we get to decide when we upgrade the stuff, and we get to decide when we upgrade the compiler. I think it would just be a lot easier for us to manage. I think there’s really good technical reasons for doing it, and I kind of want to do it, as well.

Q: I’d love to hear about exactly why the bytecode is running on the chip, on the card. Is it just for demo, or in a production environment, could you do the same?

Andy: Oh, it’s just because I can. I say there’s no practical reason, there is practical reason, actually. We are thinking about using … Well, we are using these soft CPUs in our design. At the moment, we’re having to write C code, and that’s kind of anathema, and actually for good reason. Some of this code will be fairly critical. It might be kill switches for designs that go and do real stuff. It’s got to be reviewed, and the way we know how to review code is OCaml. If there’s a whole pile of C code getting injected in there, it’s just not good- [crosstalk] No, we wouldn’t have … The toplevel’s just a bit of fun, just because I could. [crosstalk] Yeah. I don’t know-

Q: It would be a great debugging tool if all of our FPGAs, you could log in and use the OCaml toplevel- It would be insane, but-

A: [crosstalk] and check your email, and play Doom.

Andy: Yeah.

A: Everything should host Doom.

Not emails.

[inaudible]

Andy: I can actually see using the toplevel more generally just for early design tasks where you just want to go in and poke some registers in a design and see how the thing is working. If you’ve got it there, why not use it? 

Q: For soft CPU, I’m still wondering why it’s there, is it just for administrative or debug stuff? Would that ever be in the hot path? Never in the hot path?

Andy: No & No.

Q: You were mentioning that you were using SMT software for [inaudible] verification stuff. What are you using?

Andy: I’m actually using SAT stuff at the moment. I haven’t figured out how to do this weird SMT-LIB language, although, that being said, recently, into our repo, Z3 went in, and it’s done with the OCaml bindings, so that’s a much nicer interface into it. When I have a bit of time, I really fancy going and having a play with that, because I think it’d just be a quick change. Instead of using SAT, MiniSat is the main one I use, I could switch to Z3.

Q: Yeah, Z3 is beautiful.

Andy: Yeah.


















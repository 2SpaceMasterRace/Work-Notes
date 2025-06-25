While working on my class projects, it felt like most of my frustration / issues originated from the Verilog language / tooling. I'll talk more about this later, but to give a brief example, a friend and I once spent over an hour debugging an issue, only to find out that I mispelled a wire name. Most modern languages wouldn't even let that compile/run. This made the concept of Hardcaml _really_ interesting because OCaml is functional, compiled, and statically typed. Also (and somewhat more importantly), Hardcaml [supports testing](https://blog.janestreet.com/using-ascii-waveforms-to-test-hardware-designs/), meaning that all the superpowers of software testing could now also be applied to hardware.


# Basics

### What are computers, really?

At the core, computers are just general-purpose input-output machines. They take some input (e.g. data, user inputs, etc) and a set of instructions (code), and produce output (e.g. files, calculations, graphics, updated GUIs, etc) by running the input through the instructions. Along the way, they read and update a "state" (e.g. reading/writing to the filesystem, database, external APIs, etc),.

The lowest computational level of a computer that's worth thinking about is the [logic gate](https://www.tutorialspoint.com/computer_logical_organization/logic_gates.htm). These are small functions that let us perform boolean algebra on electric signals. 

![[Pasted image 20250624154230.png]]


At first glance, these seem too simple to be useful. Luckily for us, we can combine a bunch of them together and make all sorts of cool things, like [add (or subtract) 2 numbers](https://www.circuitstoday.com/ripple-carry-adder), and even [multiply and divide](https://www.nutsvolts.com/magazine/article/small-logic-gates-spawn-big-dreams-part-2).  Problem is, it's not enough to calculate: if we want to write non-trivial interesting programs, we need to be able to store various intermediate data. For example, let's say we want to sum a list of N numbers. We can either design circuits for every possible N (an impossible feat), or a single circuit that adds the numbers one at a time to some variable, and finally outputs the variable.

![[Pasted image 20250624154325.png]]

Before we go into storing state/variables, we need to understand how hardware works over time. There's this thing called a clock. Essentially, it's just a wire that alternates between being on (binary `1`) and off (binary `0`) on a regular interval. The time it takes for the clock to go from `1` to `0` and back to `1` is called a "clock cycle" or "period", and represents a unit of hardware time.

Alright, back to state. Turns out if we make a circuit that uses its own output as one of its inputs, we can use logic gates to make something called a [flip-flop](https://dept-info.labri.fr/~strandh/Teaching/AMP/Common/Strandh-Tutorial/flip-flops.html).  A flip flop stores a single bit, which it constantly emits as an output. It also takes a single bit as input. Once every clock cycle (usually at the moment when the clock changes from `0` to `1`), the flip-flop replaces its stored value with the input. If we use multiple flip-flops, we can store an arbitrary variable for a clock cycle. Unsurprisingly, we can wrap a flip flop in a simple logic gate circuit to create a [register](http://cpuville.com/Educational/Registers.html), which is the hardware equivalent of a variable.

![[Pasted image 20250624154404.png]]

It turns out that we now have everything we need to design hardware: the ability to store and make calculations on arbitrary values. From now on we'll be working at a higher level of abstraction, but before we move on, I want to emphasize 2 types of circuits:

- **Combinational** circuits are [pure functions](https://www.sitepoint.com/functional-programming-pure-functions/) that produce an output by _combining_ their inputs via some calculation/computation. This includes adding 2 numbers, truncating a value, concatenating 2 values (e.g. `01` <> `10` = `0110`), etc.
- **Sequential** circuits also have a state which they can read/right. For example, if you make a circuit for a counter that increments by `1` every clock cycle, you'll need to store and read its current value.


Before we proceed, I want to clarify my use of "stored value" and "memory". Computers don't remember things like we do, so if they calculate a value and want to use it later, they need to store it somewhere. There are 2 main options:

- **Registers** (what I referred to previously as "stored values") are storage right on the CPU used for storing intermediate values of calculations. For example, if we're adding a list of numbers, we'd probably put the running sum (also the address of the list in memory and the index of the list element we're currently on) in registers. These are usually implemented as flip-flops and are very fast, but there are very few of them (in MIPS, we can use 32 32-bit registers for instructions to read/write).
- **Memory** is longer term storage located off the CPU, often as RAM. That's where data used by your program (and the [execution stack](https://en.wikipedia.org/wiki/Call_stack)) is stored. It's a lot cheaper than registers, and there's a lot of it (gigabytes!).

---


# Hardware Design 101


Historically, hardware designers had to manually arrange logic gates and wires. Luckily, [tools were developed](https://en.wikipedia.org/wiki/Electronic_design_automation) that allow us to describe our hardware in a RTL (Register Transfer Layer) language, which represents how data signals flow through circuits. Then, software [synthesises](https://en.wikipedia.org/wiki/Logic_synthesis) that RTL into a netlist, which represents the actual layout of gates and wires.

The most used RTL languages are Verilog and VHDL. For an example, let's design the PC Register  in Verilog:

```verilog
module pc_reg (
    input clk,
    input [31:0] pc_next,
    output reg [31:0] pc
);

initial pc = 0;

always @(posedge clk) begin
    pc <= pc_next;
end
```


Simple enough: on every positive clock edge (i.e. once per cycle), we replace the stored address with a new one. We constantly emit the current address as output, and our inputs are the next address and the clock. It's concise, clear, and simple: almost as if Verilog was originally designed to serve as documentation and describe simulations. [Oh wait...](https://en.wikipedia.org/wiki/Verilog#Beginning)

> Originally, Verilog was only intended to describe and allow simulation; the automated synthesis of subsets of the language to physically realizable structures (gates etc.) was developed after the language had achieved widespread usage.


And **that's** the issue. Verilog is great at what it's supposed to do: describing circuits and simulating events. But when we start trying to write anything larger than a simple circuit, we start running into issues (at least in Vivado, the dominant IDE for hardware design):

- The synthesis process doesn't throw error in cases where you would expect it to:
    - You can have multiple modules / circuits outputting to the same wire. Understandably, when this happens the simulation gets really confused and doesn't show anything meaningful. It'll still synthesize though.
    - You can pass wires into a module by name, but it won't error if a wire is missing, misnamed, etc. As I mentioned in [the intro post for this series](https://ceramichacker.com/blog/1-hardcaml-mips-project-intro-what-and-why), [@DoperBeats](https://ceramichacker.com/u/DoperBeats) and I once spent an hour debugging only to find that I mistyped a wire name.
    - You can pass a 1-bit wire to a 5-bit port on a module (or any other mismatched combination). I imagine there might be genuine use cases for this, but often this means you made a typo. Explicitly extending/truncating signals is already possible by making a module to transform your input/output, so I'm not sure why this implicit conversion should be allowed.
    - On the rare occasions you do get errors/warnings, they're generally cryptic and require relatively deep understanding of Verilog. Not great for beginners.
- Tooling sucks
    - All the synthesis errors I complained about above could be identified through static analysis and surfaced as warnings/errors in the editor. Emphasis on **could be.**
    - The rare errors you do get are hard for beginners to understand.
    - Want to write code in a different editor like VSCode? Alright, but you'll need to manually register every new file you make in Vivado. Oh, and there's no syntax error highlighting in VSCode's Verilog extensions either.
    - Don't want to use Vivado (or can't because of export controls)? The alternatives are even worse.
    - And the worst by far: Vivado doesn't really have a dark mode 
- Testing is clunky
    - Tests also have to be Verilog modules, whereas usually in software it's preferable to use functions/methods for test cases.
    - Partially as a consequence of the previous point, testing requires a lot of boilerplate.
    - There isn't really an ecosystem for things like continuous integration. For that matter, many testing innovations from the software world (like mocks/stubs) don't seem to carry over well to hardware.
    - A lot of automated testing seems to be based around printing assertions and generating waveforms in the Vivado console/waveform viewer. That's not very automatable.
- Documentation is scarce
    - A significant portion of support answers on the Xilinx (makers of Vivado) forums are links to 100-page manuals, and are almost always jargon-heavy. Not a deal breaker by any means, but quite challenging for beginners.


Maybe (to be honest, probably) some of these are due to me improperly using/configuring Vivado. But I maintain that the majority of these things (especially the synthesis issues) would be baked into a proper language by default. First impressions matter, and my semester working with Verilog has left me very frustrated at the language and its ecosystem.

One thing I want to note is that I do not consider the (relatively) low level of abstraction in Verilog to be a shortcoming. Designing good hardware requires understanding of how your design comes together in terms of gates, wires, and transistors. Generally, you'll want to draw out the modules and circuits before writing up code to describe it.

### Why OCaml?

The Verilog syntax is decent for documenting functionality, but in my opinion, a more powerful representation of most simple circuits would be a function that takes inputs and produces outputs. For sequential circuits that need state, we could use a function that takes inputs and the current state, and produces outputs and the new state (or encapsulates state in something that makes it clean to work with).

As someone who's been looking for an excuse to learn a proper functional programming language for a while, this realization really excited me. And when I [learned about Hardcaml](https://ceramichacker.com/blog/1-hardcaml-mips-project-intro-what-and-why), it seemed perfect. OCaml is a functional programming language with an expressive static type system. Maybe that will help eliminate some of the structural errors I mentioned earlier. Hardcaml also has libraries for [automated testing](https://blog.janestreet.com/using-ascii-waveforms-to-test-hardware-designs/), where you put in the waveforms (i.e. the outputs you'd expect for various inputs and clock stages) drawn out in ASCII, and Hardcaml will simulate your circuit and compare the actual generated waveform.

There are libraries that do similar things in other languages, like [Chisel for Scala](https://www.chisel-lang.org/) and [HHDL for Haskell](https://hackage.haskell.org/package/HHDL), but I found out about Hardcaml first. Also, OCaml is more "functional" than Scala, and a bit more practical than Haskell (although eventually, I'd like to learn Haskell too).

---


### FPGYay!

In this final, somewhat tangential section, I want to briefly explain what FPGAs are, and why they are awesome. If you're not familiar, an FPGA (Field Programmable Gate Array) is essentially reprogrammable hardware: you can configure it to represent various circuits and hardware designs as needed.

Historically, if you wanted to design hardware, you'd come up with a design and send it to a foundry. Four months and tens of thousands of dollars later, you'd get back a huge pile of chips. Let's hope you didn't make any mistakes whatsoever in your design, or that your problem hasn't suddenly changed!

The way an FPGA works is in the name: **Field Programmable Gate Array.** It's a huge grid of circuits called [LUTs](https://electronics.stackexchange.com/questions/169532/what-is-an-lut-in-fpga) (short for lookup tables) that can represent arbitrary logic gates. The LUTs can then connected as needed to form pretty much any circuit. This means that you could purchase several FPGAs, and almost instantly reprogram them into various circuits/hardware to fix bugs, tweak your algorithm, or do something completely different. And although there is of course a cost for this abstraction, FPGA's aren't **that** much slower than custom-fabricated chips.

This opens up the possibility to use custom hardware (and its various performance/cost/parallelism advantages) for all sorts of problems relatively affordably.

---

**Glossary**

**logic synthesis** is a process by which an abstract specification of desired [circuit](https://en.wikipedia.org/wiki/Circuit_\(electronics\) "Circuit (electronics)") behavior, typically at [register transfer level](https://en.wikipedia.org/wiki/Register_transfer_level "Register transfer level") (RTL), is turned into a design implementation in terms of [logic gates](https://en.wikipedia.org/wiki/Logic_gates "Logic gates"), typically by a [computer program](https://en.wikipedia.org/wiki/Computer_program "Computer program") called a _synthesis tool_. Common examples of this process include synthesis of designs specified in [hardware description languages](https://en.wikipedia.org/wiki/Hardware_description_language "Hardware description language"), including [VHDL](https://en.wikipedia.org/wiki/VHDL "VHDL") and [Verilog](https://en.wikipedia.org/wiki/Verilog "Verilog").[[1]](https://en.wikipedia.org/wiki/Logic_synthesis#cite_note-Verilog_2005-1) Some synthesis tools generate [bitstreams](https://en.wikipedia.org/wiki/Bitstream "Bitstream") for [programmable logic devices](https://en.wikipedia.org/wiki/Programmable_logic_device "Programmable logic device") such as [PALs](https://en.wikipedia.org/wiki/Programmable_array_logic "Programmable array logic") or [FPGAs](https://en.wikipedia.org/wiki/FPGA "FPGA"), while others target the creation of [ASICs](https://en.wikipedia.org/wiki/ASIC "ASIC").


A **bitstream** (or **bit stream**), also known as **binary sequence**, is a [sequence](https://en.wikipedia.org/wiki/Sequence "Sequence") of [bits](https://en.wikipedia.org/wiki/Bit "Bit"). A **bytestream** is a sequence of [bytes](https://en.wikipedia.org/wiki/Byte "Byte"). Typically, each byte is an [8-bit quantity](https://en.wikipedia.org/wiki/Octet_\(computing\) "Octet (computing)"), and so the term **octet stream** is sometimes used interchangeably. An octet may be encoded as a sequence of 8 bits in multiple different ways (see [bit numbering](https://en.wikipedia.org/wiki/Bit_numbering "Bit numbering")) so there is no unique and direct translation between bytestreams and bitstreams.
# High-Performance Hardware Design with Hardcaml 

**Abstract**

Hardcaml is an embedded DSL in OCaml designed for high-performance FPGA designs. This talk will go over the design of Hardcaml, providing general principles for designing new embedded DSLs for hardware design, and showcase how Hardcaml enables both high-level abstraction and low-level optimizations.


**About the author**

My goal is to build systems that democratize the _design_ and _use_ of specialized hardware. I am excited to work with folks who are dissatisfied with the current state of tools and techniques for hardware design. I am interested in radically new approaches that combine ideas from programming languages, computer architecture, VLSI, and computer-aided design to address the design, verification, and usability challenges of specialized hardware. In doing so, I want to build real systems, evaluate them rigorously, and get other people to use them.

My PhD research has produced three systems:

- [Calyx](https://calyxir.org): A compiler infrastructure for compiling high-level languages to efficient circuits. Calyx has been adopted by the [LLVM CIRCT](https://circt.llvm.org) project and is the basis for several academic and research tools.
- [Filament](https://filamenthdl.com): a new hardware description language that uses a novel type system to guarantee correctness of pipeline composition. Filament’s ideas have influenced the design of Google’s [XLS](https://google.github.io/xls/) system and Jane Street’s [HardCaml](https://github.com/janestreet/hardcaml) language.
- [Dahlia](https://capra.cs.cornell.edu/dahlia): A high-level language for predictable accelerator generation. Dahlia demonstrated how type systems can connect high-level abstractions with circuit-level constraints.


The goal of the FLAME lab is to design principled abstractions across the programming systems stack (languages, compilers, architectures) to enable the efficient design and use of specialized hardware.

> If you are interested in joining the lab, please review the [specific instructions](https://flame.csail.mit.edu/lab/prospective/) before contacting us.

## Directions

### Computer Architecture

A staggering number of new hardware accelerators have been developed to combat the end of scaling. However, each accelerator is a unique snowflake with its own design principles, implementation methodology, and software stack. _Can we bring order to this heterogeneity by extracting flexible and composable ideas for architectural design?_

### Programming Languages

Riding on the wave of physical and architectural scaling, programming languages have focused on gluttonous abstraction to enable large-scale design. With the end of scaling, languages need to work in tandem with compilers and architectures to carefully expose low-level control. _Can we design new programming languages that enable low-level control without compromising on modular abstraction?_

### Compilers

The clear abstraction boundary between hardware and software provided by traditional processors is lost in the realm of accelerators. While this boundary has enabled the principled study and design of software compilers, its absence means that accelerator compilation only has point solutions. _How can we design scalable and automated compilers for accelerator design and use?_


---

## Slide 1

Let's build an eHDL


start with a fundamental question, what is an adder?

start with basic simple building blocks then proceed to combinational ones

An adder : take two signals and return a new signal 

![[adder.excalidraw]]

```python
def add (l: signal, r: signal) -> signal {
// don't know what the implementation is
...
}
```

even if you leave it abstract, you can still use it with the definition of the adder

for example take this function that represents the circuit adder and build a new function call _const_mul_

```python
def const_mul (k: int, s: signal) -> signal {
	current_signal = s
	while (k>0):
		current_signal = add(current_signal, k)
		k -= 1
	return current_signal
}
```

it's multiplying with some compile time constant, so there is something that represents the signal and the signal  is a circuit time value, it's a value when you're executing the signal it's a value show up only when the circuit is executing

but value K is compile time and actually exists in python, so the way the code works is when you run _const_mul_, it's going to use the value K and build a circuit for you by repeatedly calling the _add_ operator on it. 

s is guaranteed to be a signal, *curr_sig* at the start or its add which returns a signal


this is the magic of embedded DSLs, we have something inside the language called add that build those circuits for you and you just get to use the host language -- the one you're working in  to build that circuit for you -- you can define high order  functions or libraries

you can do with python w pymetal, scala with chisel and obv with ocaml -- gods greatest language // primeagen ocaml shilling 


![[adder_annotated.excalidraw]]


Takeaway: EHDls have some recipes going on in building one of these 

![[circuit.excalidraw]]



The eHDL recipe:

* Define **small** set of primitives
* Use **Host language** to build useful libraries 

you define small set of primitives like add, subtract or even registers if you are feeling a little adventurous and use host the features of the host language to build the libraries.

if you look into API of chisel and hardcaml or pymetal, it's a massive API and you question how to implement something this size.

Takeaway is you don't have too -- you can build it in the software language itself as long as you define the small set of primitives

![[blocks.excalidraw]]

the add, mul are fundamental blocks and higher level functions are built on top of it

**Notes**

Try to give a intro to fpga's and their use-case and try to reverse engineer why Jane street use fpga's and show them and yourself on how eHDLs simplify life

---

## Slide 2

What's in a signal?


```python
def add (l: signal, r: signal) -> signal 
```

![[adder.excalidraw]]


L can be:
- **constant**
- **wire** or **registers**
- another **primitive** circuit
- ... and that's it ! 

How do you implement the signal type ? it's not too hard you just need those 3 things to be represented.

![[signalstruct.excalidraw]]

kind of type here is constant what value it has, if it's a wire or register who's driving it, if it's a circuit what are it's own dependent

some examples of struct types are:

```python
zero  = signal ("zero", constant(0))
clock = signal ("clock", wire)
```

now a function like add, we can define as:

```python
def add (l: signal, r: signal) -> signal {
	return signal("add", primitive_add(l,r))
}
```


why do we need _name_?  The way we build embedding, the HDL is everything on the right side of the = , clock is just the name in the host langauge

In HardCaml, we have pre-processor pass and just add clock = signal(wire) and it will copy that name for you but by default you can have clock 0, clock 1, clock 2, and give it the same variable name

There's a separation going on between the host language like python and object/circuit language that we're constructing on the right side

What we are doing is constructing a computational graph like pytorch

![[addergraph.excalidraw]]


**We are building the structure of our circuit and capturing it by connecting things together by signal representation**


This is how pytorch and tensorflow works -- they capture the computational graph that you write down that represents your machine learning kernel and compile it and executing it (they do a little more but functionally it's the same)

---

## Slide 3


Final step is compiling programs to RTL

```python

"Syntax-Directed" compiler

def compile(s: signal) -> RTL:
	match s with
	| const(c) -> f "b' {c}"
	| add (l,r) -> l_c = compile(l) 
	|              r_c = compile(r)
	|              f" {l_c} + {r_c}" 
```


This is a sort of a baby's way of building a compiler and the way you do it is by using some made up syntax

take a signal and produce RTL, RTL is just strings -- If the type of the signal is some constant, return b' "the constant"

if you take the interesting case of the adder, you compile the left side and compile the right side and embed a string that represent the rest of the circuit and recursively compile each dependent 

![[addergraph.excalidraw]]


because we have collected this bring computation graph, recursively go down, compile each component, combine everything and get the output.

is this readable? nope

when you dump the verilog no one can read it so it's important for us to add those identifiers


**Lifetime of a eHDL Program**

```python
const_mul(10, a) --------                                  High-level 
						|                                  abstractions
						v
				add(add(add(..., a))) ---------            primitives
											  |
											  v
										a + a + q ...      RTL	  

```

A HDL program starts with some high-level operations on it with things like *const_mul* and other API's that are defined in terms of basic building blocks like *add* that are then compiled to RTL. 


Now you know how to build a eHDL. But the problem is HDL alone isn't enough, we need things like :
		- Simulator
		- Verification
		- Formal ...?

One of the big reasons people like chisel is they give you nice new abstractions that are not available in verilog. You want to build a simulator  that can take your high level language and give you error messages and simulation reports, you want verification tools and formal tools.

Everything you've learnt upto now actually makes it quite trivial to build these tools and we can do this by building a simulator with the same representation.

**Notes**

Explore HardCaml codebase, use the techniques from hashimoto and all to see how to break down the codebase and understand it's internals to write an article on it


---

## Slide 4

**What's in a signal? (Redux)**

- Circuits are "computation graphs"
- But we get to define what the **base primitives** mean

There is nothing inherent in the way we defined add that lends itself to being a circuit. It can be defined or interpret it to anything else like a little function when we call it totals the result of our computation. The basic insight is as once you defined the base primitives, we can interpret them in different ways.

Simplest interpretation is building circuits but another interpretation could be define the set of functions that are going to represent the result of simulation like combinational sequences. 

**Let's build a eHDL simulator**

```rust
struct signal {
	compute : function*, // signal tracks an update function that represents it's simulation behaviour
	typ     : type
}
```

You need a bit more trickery for registers but this will work for combinational components. 
You define a different signal types and instead of having field id we have a field compute which has a function pointer, and it captures the computation that node is going to do is.

```python
def add(l:signal, r:signal) -> signal {
	return {
		compute : fn() {
			l.compute() + r.compute()
		}
	}
}
```

For example, with the implementation of add -- add computes what the left and right nodes and adds the results. You can build again in the very nice syntax directed way the whole simulator. It won't be fast or great but you can do that with building different interpretations of the signal type. You'll need more functionality for registers and hopefully you get a flavor of what it takes to build simulators of the same API. 

Takeaway: If you do this correctly, you can define a set of base primitives to compile to RTL and that'll give you a circuit generator where you can define what they mean computationally and it'll give you a  simulator or formally and it'll give  a formal validator. You can build high level libraries and not worry about the step, because the base primitives are sort of the interposing step that allow you to do this.

![[ehdlrecipe.excalidraw]]

Larger **Base Primitives** library enable more control of the **Generated RTL** on how it's generated or simulated but it **harder to implement** new tools.

---

## Slide 5

Hardcaml is the realization of this idea. Here's the base type of hardcaml:

```rust
Module type Comb_intf.Primitives
// Type required to generate the full combinational API

val mux : t -> t Hardcaml__.Import.list -> t

    multiplexer

val (+:) : t -> t -> t

    addition

val (-:) : t -> t -> t

    subtraction

val (*:) : t -> t -> t

    unsigned multiplication

val (*+) : t -> t -> t

    signed multiplication

val (==:) : t -> t -> t

    equality

val (<:) : t -> t -> t

    less than
```

This is all you need to build the combinational component of the HDL. You can get an API of this size:

![[apiderivied.excalidraw]]

There's also a simulator implemented in Hardcaml using the kind of idea I described to you, and have several simulators -- for quick simulations for unit testing, you use *CycleSim* which can sort of run in-line but is written in OCaml and is slow for big designs. But we also have *CycleSim_verilator* which will compile your designs (those 7 primitives ) to get the verilator  verilated components you can simulate. 

![[cyclesimapi.excalidraw]]

HardCaml have many things implemented :

![[completehardcaml.excalidraw]]

more here: https://ocaml.janestreet.com/ocaml-core/v0.12/doc/hardcaml/Hardcaml__/Comb_intf/module-type-S/index.html

Hopefully the style of building DSLs is complete and useful for you to think about.

---

**Why people build their own embedded Hardware Description Languages?**

The capability of building reusable generators popularized by Chisel language thanks to Berkeley  made it evident that you can build really big designs by building small blocks and composing them.

The other thing that drives people to build their own HDL is it might fit neatly with the existing infrastructure - JS has their own mature OCaml ecosystem testing, verification and validation stack and so it makes sense to write OCaml and take engineers from a different team, have them contribute to some like function that they might are about and we write the shell to accelerate their computation.

Circuits have state so they're not technically computational graphs. HardCaml has this other type called *Signal.t* which has combinational components, registers and memory. These 3 things make a full circuit/HDL. Registers are also defined with *clock*, *clear*, *reset_to*, etc. 

You can annotate *Signal* type to track more information. One of the things you can build is clock domain tracking on the signal type and notionally the only change you gotta do is redefine the six primitives to track clock domains and give error messages when they mismatch and all of the existing code can benefit from clock domain checking. 

---

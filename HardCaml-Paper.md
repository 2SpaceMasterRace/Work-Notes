**TLDR**:

- **Hardcaml** is an embedded hardware DSL built in **OCaml** that combines **low-level control** with abstraction over traditional HDLs like **Verilog** and **VHDL**.
    
- Leverages OCaml’s **strong type system** and **fast elaboration checks** to reduce bugs and ensure correctness.
    
- Offers integrated **cycle-accurate and event-driven simulators** plus embedded **unit tests** with ASCII waveform outputs for rapid feedback.
    
- Supports **formal verification** techniques including **SAT proving**.
    
- **Industry-proven** and actively used at **Jane Street** for large FPGA designs.

---
# Background:

**SystemVerilog** and **VHDL** are currently the most widely used hardware description languages (HDLs) in the industry. While they are expressive enough to define hardware circuits, they differ significantly from traditional software programming languages. As a result, they have not fully benefited from modern advances in programming language design—advances that enhance productivity and flexibility. These improvements are increasingly important in hardware design, as systems continue to grow in size and complexity. Today, a significant portion of a hardware designer’s time is spent on modeling, simulation, and testing.

A large number of existing domain-specific languages (DSLs) have been developed to address various shortcomings of Verilog and VHDL. These DSLs offer different levels of abstraction and control over hardware primitives, provide safety guarantees, and often include additional features such as meta-programming. The benefits of using such languages have been demonstrated empirically. 


**Diff in verilog and DSLs**:

For instance, a RISC-V core design that required 1,000 lines of handwritten Verilog was implemented in only 586 lines using Chisel while a similar design required just 637 lines using Hardcaml (AjayMT/risk5: RISC-V CPU in Hardcaml/ -- https://github.com/AjayMT/risk5). 

*Reducing the amount of code that designers must write not only boosts productivity but also reduces the likelihood of bugs. Furthermore, there is growing interest in translating traditional RTL designs into the domain of these DSLs (# Verilog to Chisel Translation)*


**DSL types**:

**Existing domain-specific languages (DSLs)** can be broadly categorized into two main types based on their abstraction level from the underlying RTL: **compiler-style DSLs** and **full-control DSLs**.

**Compiler-style DSLs** allow designers to describe circuits using dataflow models or higher-level programming languages such as C++.  - The DSL framework compiles these descriptions down to RTL.

- This approach enhances productivity by enabling rapid iteration over potentially large design spaces with many parameters. However, it limits fine-grained, clock cycle-level control and optimization.
    
- Compiler-style DSLs are often associated with high-level synthesis (HLS) tools. They may be implemented as standalone solutions or as wrappers around existing toolchains and general-purpose compiler infrastructures.
    
- Some DSLs allow hardware designs to be expressed using declarative rules that fire automatically with handshaking mechanisms to control data flow.



**Full-control DSLs** enable designers to write hardware that directly instantiates hardware primitives such as registers, memories, multiplexors, and more.
    
- These DSLs allow optimization of circuits down to clock-cycle latencies. They offer enhancements over traditional HDL languages through software language parameterization and meta-programming.
    
- Hardcaml is an example of a full-control DSL.
    
- Other similar DSLs provide comparable capabilities but each offers a slightly different set of tools and features. These DSLs enable safe module reuse, pipeline scheduling, and parametrization. Such features help designers optimize and customize their hardware designs more effectively.




---

# Overview


Hardcaml is a domain-specific language (DSL) embedded in OCaml that enhances reliability when writing hardware description languages (HDL) while remaining lightweight and easy to use. OCaml is a multi-paradigm (functional, imperative, object-oriented) high-level programming language featuring an expressive type system and type inference. Hardcaml is a collection of OCaml libraries that allows designers to express circuits with the same level of control as Verilog or VHDL, but with the advantages of a higher-level programming language. It eliminates much of the mundane, error-prone work involved in connecting modules and parameterizing circuits, as will be demonstrated. A strong built-in linter helps prevent bugs such as mismatched signal widths, floating signals, and multiple drivers.

The typical workflow for designing with Hardcaml is as follows:

1. Write the circuit implementation using Hardcaml libraries.
    
2. Compose testbenches, which are run using the built-in Hardcaml simulator.
    
3. Debug using the built-in terminal-based waveform viewer and expect-tests.
    
4. Generate Verilog from OCaml using the built-in Hardcaml RTL generator.
    
5. Run the generated Verilog through standard vendor tooling (e.g., Vivado).
    

Steps 1 through 3 can leverage existing OCaml tooling, such as the Dune build system, for rapid development.

**Ocaml's advantages**:

OCaml modules and functors provide flexible circuit parameterization in Hardcaml. Both the strong type system and an extensive testing and verification toolkit ensure safety and correctness. Circuit elaboration is fast and occurs during both RTL generation and simulation, entirely within OCaml. During elaboration, Hardcaml enforces that there are no floating wires, no multiple drivers, and that connected signals have matching widths. This combination helps avoid common hardware design bugs, allowing engineers to focus more on the enjoyable and valuable aspects of hardware design.

Hardcaml also features both a built-in cycle-accurate and event-driven simulator, enabling unit-level tests alongside the source code. These tests can optionally print digital ASCII waveforms, providing fast feedback on designs and helping catch future bugs early.

Additionally, Hardcaml supports hardware generation from abstraction models written in OCaml, simulation expect-tests with waveforms, interactive waveform viewers, and synthesis reports. Ultimately, it allows designers to produce production-ready Verilog and/or VHDL code.


**DevFlow**:

The development flow for hardware design in Hardcaml closely resembles software development. The availability of Hardcaml simulators and various tooling, combined with the OCaml Dune build system and a wide range of OCaml libraries, integrates testing and simulation into the standard software development lifecycle. This comprehensive testing environment, all within a single programming language, enables rapid hardware development with a fast feedback loop during simulation.


**What makes hardcaml different from others**:

While existing DSLs address various aspects of safety and productivity, Hardcaml provides hardware engineers with a more contained and complete solution. Unlike other DSLs, Hardcaml includes comprehensive testing capabilities such as unit tests with waveforms and built-in simulators. Each DSL is embedded within a higher-level software language, which ultimately defines its capabilities and limitations. OCaml, a strongly typed and modern functional programming language, serves as the foundation for Hardcaml, which is fully embedded within OCaml without any secondary abstraction layer. This design makes Hardcaml lightweight and simplifies the development of improvements or debugging directly in the source code. Hardcaml is an actively maintained open-source project developed at Jane Street, where it serves as the primary system for creating production FPGA designs.

Hardcaml also provides the hardcaml_fixed_point library, with support for hardware-synthesizable arithmetic/operations such as +, -, and *. It provides a rich set of rounding and clamping modes, and seperate types for performing signed or unsigned operations. This library is useful for many signal processing applications, where explicit control over integer and fractional bit-widths is required. Functors can be used with this library to implement and experiment different bit-widths and the resulting change of output precision.

**Drawbacks of Hardcaml**

- Mapping between Hardcaml function calls and generated RTL can be unclear unless explicitly labeled, complicating debugging of timing violations in tools like Vivado.
    
- Interfacing with external IP cores requires blackbox wrappers that cannot be simulated directly in Hardcaml without extra steps such as importing RTL.
    
- Users may face a learning curve due to unfamiliarity with functional programming or OCaml.


**Future Work**

We are continuously enhancing Hardcaml and releasing improvements as open source. Planned new features aim to further distinguish Hardcaml from other DSLs:

- **Encoding Signal Clock-Domains within the Type System:**  
    Currently, Hardcaml uses a single `Signal.t` type for the entire design. We plan to constrain `Signal.t` to specific clock domains using _phantom types_, a functional programming technique. This will leverage the type system to prevent accidental mixing of signals across different clock domains. When cross-clock-domain crossing is necessary, signals can be explicitly marked to relax these constraints. This approach balances clock-domain safety with flexibility in practical designs.
    
- **Floating Point Support:**  
    We are developing a suite of floating-point cores, including adders, multipliers, fixed-point converters, and more, which will be included in the open-source Hardcaml library once ready.
    
- **Integration with Higher-Level HDLs:**  
    Some HDLs like Filament and Bluespec prioritize higher-level abstraction over low-level control. We plan to create seamless integration—potentially as a Hardcaml library—between Hardcaml’s low-level control circuits and these higher-level HDLs. This integration aims to improve programmer productivity by offering more tools within the Hardcaml environment, including:
    
    - A user-friendly API for constructing circuits in these HDLs
        
    - Rapid circuit elaboration and validation
        
    - Integration with Hardcaml’s existing simulation frameworks and RTL generation


---

# Basics

**Datatypes**

The core type in a Hardcaml circuit is the Signal.t type. Various functions and operators that take Signal.t as parameters represent hardware primitives:

| Symbol(s)         | Description                             |
| ----------------- | --------------------------------------- |
| `+:`, `+:.`       | Addition operator                       |
| `-:`, `-:.`       | Subtraction operator                    |
| `&:`              | Bitwise AND operator                    |
| `\|:`             | Bitwise OR operator                     |
| `==:`, `==:.`     | Equality operator                       |
| `reg`, `reg_spec` | Registers and clock/clear specification |
| `mux2`, `mux`     | Multiplexors                            |

All operators contain a : to distinguish them between regular OCaml operators on integers, as OCaml does not support operator overloading. There are multiple variants of arithmetic operators (eg: +: and +:.). The +: and +:. operators both represent addition. +: operates on two Signal.ts that must match bit-width, while +:. operates on a Signal.t and integer that has its bit-width inferred. The programmer can create a hardware circuit by calling functions on a value of type Signal.t, and then storing
the returned value (which is also a Signal.t). For


reg_specs specify clock and clear signals for sequential logic such as regs. 

mux is a general multiplexor that takes a list of arguments

mux2 is a convenient helper function to write a simple if-then-else multiplexor with two cases.


|**Verilog Counter**|
|---|
```verilog
module counter #(
  parameter COUNT_TO = 100,
  parameter CTR_WIDTH = $clog2(COUNT_TO + 1)
) (
  input clock,
  output reg [CTR_WIDTH-1:0] x
);
  always @ (posedge clock) begin
    x <= x == COUNT_TO ? 0 : x + 1;
  end
endmodule
```

|**Hardcaml (OCaml) Counter**|
|---|
```ocaml
let counter ~clock ~count_to =
  let spec = Reg_spec.create ~clock () in
  let width = Int.ceil_log2 (count_to + 1) in
  let x = wire width in
  x <== reg spec (mux2 (x ==:. count_to) (zero width) (x +:. 1));
  x
```

**The advantage of using Hardcaml here is it will prevent the counter output connecting to something that isn’t wide enough, which if happened in Verilog would silently drop the unused bits.**



**Defining New Types**

Hardcaml can represent each logical idea in its own type, defined by the programmer. OCaml’s type system then statically guarantees that values of a given type cannot be used in ways other than those defined valid for that type. This enables both abstraction and eliminates entire classes of potential errors. parameterized type argument allows us to use Rectangle.t in different contexts, like tests, debugging, and HDL generation.

> Note that these types are parameterized with 'a, later to be replaced with Signal.ts. 

the following Hardcaml snippet defines a type called Rectangle.t, which has a 10-bit length and a 6-bit width:

```ocaml
module Rectangle = struct
	type 'a t =
	{ length : 'a [@bits 10]
	; width : 'a [@bits 6]
	}
	[@@deriving hardcaml]
end
```

You can then write a function that takes a Rectangle.t as an argument. This snippet would generate a circuit that outputs the area of a given rectangle.

```ocaml
let calculate_area
	~clock
	(rectangle : Signal.t Rectangle.t) =
let spec = Reg_spec.create ~clock () in
reg spec (rectangle.length *: rectangle.width)
```

[@@deriving hardcaml] instructs the Hardcaml preprocessor to generate utility functions working with the Rectangle.t type. The derived Rectangle.map function applies a function to every field. For example, this function returns a rectangle where both the length and width are increased by a certain size:

```ocaml
let extend_length_and_width ~by rectangle =
	Rectangle.map rectangle ~f:(fun x -> x +:. by)
```

Custom types can be nested arbitrarily to create multiple layers of logical separation. For example:

```ocaml
module Shape_collection = struct
	type 'a t =
	{ rectangle1 : 'a Rectangle.t
	; rectangle2 : 'a Rectangle.t
	}
	[@@deriving hardcaml]
end
```

Another abstract type that one would define is a scalar-type. These are bit-vectors of a particular width that have logical meaning.

For example, we can define a 32-bit price in US dollars. Then can define some custom functions that operate on this type, namely is_gte_zero, min, and max. Including the module Types.Scalar creates a type Price_in_usd.t and common functions on that type.

```ocaml
module Price_in_usd : sig
	include Types.Scalar.S
	
	val is_gte_zero : Signal.t t -> Signal.t
	val min : Signal.t t -> Signal.t t -> Signal.t
	val max : Signal.t t -> Signal.t t -> Signal.t
	
	end = struct
		include Types.Scalar.Make(struct
			let port_name = "price_in_usd"
			let port_width = 32
		end)
	
	let is_gte_zero t = t >=:. 0
	let min a b = mux2 (a <: b) a b
	let max a b = mux2 (a >: b) a b
end
```

Functions can be defined to explicitly expect a value of type Price_in_usd.t, rather than an arbitrary value of type Signal.t

The OCaml compiler will permit the following.

```ocaml
cap_price_at_zero (Price_in_usd.Of_signal.of_int 42)
```

On the other hand, the OCaml compiler will emit a compile-time error due to a type mismatch in the following code snippet.
```ocaml
cap_price_at_zero (Signal.of_int ~width:32 42)
```

This is an important safety feature of Hardcaml as it prevents the programmer from passing values of unexpected types into functions. These scalar types can also be nested within interfaces like:


```ocaml
module Min_max = struct
	type 'a t =
	{ min : 'a Price_in_usd.t
	; max : 'a Price_in_usd.t
	}
	[@@deriving hardcaml]
end
```




**Hierarchical Modules**

A Hardcaml circuit module can be structured similar to a Verilog circuit module. Using the hierarchy library inside Hardcaml, a global circuit hierarchy can be constructed. This produces simulation waveforms with hierarchical signals which increases debugging productivity, and generates RTL with hierarchical modules, which works better with existing Verilog and VHDL-based vendor tooling.


To define a Hardcaml circuit module, the programmer defines I and O modules, corresponding to the input and output interfaces.

```ocaml
module I = struct
	type 'a t =
	{ clock : 'a
	; x : 'a [@bits 10]
	}
	[@@deriving hardcaml]
end

module O = struct
	type 'a t =
	{ y : 'a [@bits 20]
	}
	[@@deriving hardcaml]
end
```


Then, the programmer defines a create function that when given Signal.t I.t, returns Signal.t O.t.

```ocaml
let create (i : Signal.t I.t) =
	let spec = Reg_spec.create ~clock:i.clock () in
	{ O.y = reg spec (i.x *: i.x) }
```




**Bit-width Inference and Post Processing Validation**

Hardcaml handles bit-width strictly. The circuit construction step validates bit-widths for all operations before running simulations or generating Verilog. This prevents many errors at the compilation stage, increasing the speed of the designer’s feedback loop. For example, the following program will fail due to a width mismatch:

```ocaml
let not_allowed a b = uresize a 32 +: uresize b 16
```

Hardcaml can also infer the bit-width for most operations, so users don’t need to specify the width for most internal wires. In the case above we specifically resized our wires to 32-bits and 16-bits.

For example, Hardcaml will infer the width of mult_result as width a + width b and the width of eq_result as 1:

```ocaml
let foo a b =
	let mult_result = a *: b in
	let eq_result = a ==: b in
	...
```

During circuit elaboration, Hardcaml also performs other validation, ensuring that:

	• Module instantiations have appropriate port widths.
	• All wires have exactly one driver.
	• Operation bit-widths match.
	
These errors are caught early on before simulation is run or RTL is generated.




**Module-Level Parameterization with Functors**

In OCaml a functor is a module that takes one or more modules as a parameter, and returns a new module. These allow us to parameterize Hardcaml circuits, similar to Verilog’s parameters. The programmer first defines a config argument for the parameterized module. For example:

```ocaml
module type Config = sig
	val metadata_bits : int
	
	module Data : Hardcaml.Interface.S
end
```

Then, they can define the corresponding circuit and module using fields (metadata_bits) or datatypes (Data.t) from the functor argument. The module Make below is an OCaml functor:

```ocaml
module Make(X : Config) = struct
	module I = struct
		type 'a t =
		{ clock : 'a
		; metadata : 'a [@bits X.metadata_bits]
		; data : 'a X.Data.t
		}
		[@@deriving hardcaml]
	end
end
```

The functor can then be instantiated with an argument of the appropriate type signature. For example:

```ocaml
module Foo = struct
	type 'a t =
	{ bar : 'a
	; baz : 'a
	}
	[@@deriving hardcaml]
end


module A = Make(struct
	let metadata_bits = 42
	module Data = Foo
end)
```

This is similar to parameterized circuits in Verilog. The key advantage of this **functor-style** is that the parameters themselves can be arbitrary arguments—such as lists, integers, arrays, or even custom types (e.g., `Data.t` in the example above). This allows for **high expressiveness** in describing parameters.

When instantiating the `Make` functor, the **OCaml compiler** ensures that the provided argument matches the expected type, adding type safety at compile time.

Functors, as demonstrated above, are a powerful feature in OCaml that enable **modularity and parametrization**. This highlights a core advantage of Hardcaml being embedded in a full-featured programming language: designers can leverage **existing software-level features** to write more expressive, modular, and robust hardware.

###  Testing and Simulation

One of the aspects of Hardcaml that sets it apart from other DSLs is its emphasis and support for circuit testing, simulation, and verification.

#### Expect-test Simulations and Waveforms

The ability to simulate circuits, view resulting waveforms, and fix any bugs are important parts of the workflow. While many other DSLs in this space require transformation through an intermediate form and the use of external simulators and waveform display tools, Hardcaml supports direct simulation of the circuit graph inside OCaml. The cycle-based simulator has been optimized for runtime performance, and common primitive bit-level manipulation is done via fast OCaml Foreign Function Interface (FFI) calls into C.

OCaml tests are written in a ”snapshot” style, where statements can be immediately followed by an [%expect]  block that prints that statement’s output. Any changes in the code that affect this frozen output are detected as test failures. Hardcaml integrates into this flow seamlessly, and circuit inputs and outputs can be printed at any cycle in the simulation. 

Waveterm is one of Hardcaml’s many supporting libraries. It can print ASCII waveforms over a given number of cycles, and it can also operate as a stand-alone tool for interactive use. It can also be integrated into project-specific testing binaries. 

The advantage of ASCII waveforms is that they can be included in expect-test output. This helps users understand how Hardcaml functions work. It also makes clear how changes in source code change test outputs, which makes breaking changes easier to detect. This is also helpful when reviewing changes submitted to a version-control repository, since the reviewer can directly see how the code changes affect the output. This will also allow for quick detection of any unwanted changes to source code that cause existing circuits tests to fail.



### Integration into Other Simulation Backends

In addition to the built-in simulator Cyclesim provided, Hardcaml also allows for seamless simulation into two other backends - hardcaml_c and hardcaml_verilator. These provide better simulation performance at the expense of a longer compilation time. 


hardcaml_c uses even more bindings to C for the circuit evaluation operations, and hardcaml_verilator compiles the circuit to a shared object and calls into a Verilator binary.

When the simulation is complete, the backend sends the results back to the Hardcaml framework, which displays them in the same waveform viewers and debugging tools used with the built-in simulator. The user doesn’t need to change their workflow, other than calling a different simulator function. By allowing seamless integration into these backends, simulation runtime can be optimized depending on the size and number of input stimuli.

### Hardcaml Verify

When designing circuits where correctness is paramount, more detailed verification is required. The library hardcaml_verify provides many tools for the user:
	• SAT solvers can check the equivalence of combinational circuits and identify error cases.
	• The Bounded Model Checker can check the equivalence of two sequential circuits, up to a given number of clock cycles.
	• NuSMV [Cimatti et al.(2002)] integrates with Hardcaml to prove LTL assertions. While this is slow and works better for small circuits, Hardcaml makes it possible to parameterize the size of the circuit, so NuSMV can prove assertions for small versions of the circuit.


## Built-in Synthesis Reports

Hardcaml also has a built-in synthesis report utility. This allows visualizing hierarchical synthesis results of Hardcaml circuits by running them through backend tools. For example, the programmer can setup a synthesis executable for the design below, which instantiates multiple Hardcaml modules. The user can then run the executable to obtain a table with the timing summary and resource utilization of the synthesis results, as show in an example in figure 2. The ease of running this tool lowers the barriers to running synthesis on small circuits. There are also more options available to control the output artifacts, such as running place and route on top of synthesis, or generating intermediate design checkpoint files.


---

# Example Circuit Descriptions

**Linear Feedback Shift Register**

A Linear Feedback Shift Register (LFSR) is a shift register whose input bit is the XOR of 2 of more bits (taps) of it’s previous state. This can be expressed in Hardcaml as follows:


```ocaml
let xor_lfsr ~width ~clock taps =
	let spec = Reg_spec.create ~clock () in
	reg_fb ~width spec ~f:(fun x ->
		let input_bit =
		List.map taps ~f:(fun i -> x.:(i))
		|> reduce ~f:(^:)
		in
		lsbs x @: input_bit)
```

The constructed LFSR x behaves identically to the following Verilog code snippet:

```verilog
reg[6:0] x;
	wire input_bit = x[1] ^ x[2] ^ x[3];
	always @ (posedge clock) begin
		x <= { x[5:0], input_bit };
end
```


**Constructing a RAM**

Hardcaml has built-in support for constructing memories, which are inferrable by backend tools like Vivado. Models are provided for both simulation and synthesis, which have been verified against vendor-provided RTL using the formal proving libraries provided with Hardcaml.


---




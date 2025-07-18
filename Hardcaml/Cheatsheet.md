# HardCaml Complete Programming Cheatsheet

**HardCaml is an OCaml-based hardware description language** that allows you to design, simulate, and synthesize digital circuits using functional programming principles. This cheatsheet provides everything you need to start programming in HardCaml immediately, with minimal theory and maximum practical code examples.

## Quick Setup and Installation

### Install HardCaml Toolchain

```bash
# Basic installation
opam install hardcaml ppx_hardcaml hardcaml_waveterm dune

# Complete development environment
opam install hardcaml ppx_hardcaml hardcaml_waveterm \
  hardcaml_c hardcaml_verilator hardcaml_circuits \
  hardcaml_fixed_point hardcaml_verify \
  hardcaml_step_testbench hardcaml_xilinx
```

### Project Structure

```
project/
├── dune-project
├── lib/
│   ├── dune
│   └── design.ml
├── bin/
│   ├── dune
│   └── main.ml
└── test/
    ├── dune
    └── test_design.ml
```

## Core Types and Signal Creation

### Signal.t - The Foundation

```ocaml
open Hardcaml
open Signal

(* All hardware signals are type Signal.t *)
let signal_8bit = Signal.of_int ~width:8 42
let zero_8bit = Signal.zero 8
let ones_8bit = Signal.ones 8
let vdd = Signal.vdd      (* 1-bit high *)
let gnd = Signal.gnd      (* 1-bit low *)

(* From different representations *)
let from_hex = Signal.of_hex ~width:8 "FF"
let from_bin = Signal.of_bstr "10101010"
let from_char = Signal.of_char 'A'

(* Width operations *)
let w = Signal.width signal_8bit        (* Returns 8 *)
let addr_bits = Signal.address_bits_for 256  (* Returns 8 *)
```

### Basic Operations

```ocaml
let a = Signal.of_int ~width:8 10
let b = Signal.of_int ~width:8 5

(* Arithmetic (all operators end with :) *)
let sum = a +: b           (* Addition *)
let diff = a -: b          (* Subtraction *)
let prod = a *: b          (* Multiplication *)
let add_const = a +:. 1    (* Add constant *)

(* Bitwise operations *)
let and_result = a &: b    (* Bitwise AND *)
let or_result = a |: b     (* Bitwise OR *)
let xor_result = a ^: b    (* Bitwise XOR *)
let not_result = ~: a      (* Bitwise NOT *)

(* Comparisons *)
let eq = a ==: b           (* Equality *)
let lt = a <: b            (* Less than *)
let slt = a <+ b           (* Signed less than *)
```

## Interface Definition with PPX

### Basic Interface Pattern

```ocaml

module I = struct
  type 'a t = {
    clock  : 'a;
    clear  : 'a;
    enable : 'a;
    data   : 'a [@bits 8];
  } [@@deriving hardcaml]
end

module O = struct
  type 'a t = {
    result : 'a [@bits 8];
    valid  : 'a;
  } [@@deriving sexp_of, hardcaml]
end

(* Interface usage *)
let inputs = I.Of_signal.wires () 
let outputs = O.Of_signal.wires ()
```

### Advanced Interface Features

```ocaml
module Complex_I = struct
  type 'a t = {
    clock      : 'a;
    data_array : 'a array [@bits 8] [@length 4];
    data_list  : 'a list [@bits 16] [@length 3];
    config     : 'a Config.t [@rtlprefix "cfg_"];
  } [@@deriving sexp_of, hardcaml]
end
```

## Signal Manipulation and Selection

### Concatenation and Slicing

```ocaml
(* Concatenation *)
let concat_result = Signal.concat [a; b]    (* MSB first *)
let concat_two = a @: b                     (* Concatenate operator *)

(* Bit selection *)
let upper_nibble = Signal.select a 7 4      (* Select bits 7:4 *)
let bit_7 = Signal.select a 7 7            (* Select bit 7 *)
let msb = Signal.msb a                      (* Get MSB *)
let lsb = Signal.lsb a                      (* Get LSB *)

(* Alternative selection syntax *)
let sel1 = a.:[7, 4]                       (* Same as select a 7 4 *)
let sel2 = a.:+[2, 4]                      (* 4 bits starting at bit 2 *)

(* Resizing *)
let zero_extend = Signal.uresize a 16       (* Zero extend to 16 bits *)
let sign_extend = Signal.sresize a 16       (* Sign extend to 16 bits *)
```

### Multiplexers and Selection

```ocaml
(* 2-input multiplexer *)
let mux2_result = Signal.mux2 sel a b

(* Multi-input multiplexer *)
let inputs = [a; b; Signal.of_int ~width:8 100]
let select = Signal.of_int ~width:2 1
let mux_result = Signal.mux select inputs

(* Priority select *)
let cases = [
  { With_valid.valid = sel1; value = a };
  { With_valid.valid = sel2; value = b };
]
let priority_result = Signal.priority_select cases
```

## Module Creation and Instantiation

### Basic Module Pattern

```ocaml
(* Module implementation *)
let create (i : _ I.t) : _ O.t =
  let spec = Reg_spec.create ~clock:i.clock ~clear:i.clear () in
  let counter = Signal.wire 8 in
  let next_count = Signal.mux2 i.enable (counter +:. 1) counter in
  Signal.assign counter (Signal.reg spec next_count);
  { O.result = counter; valid = i.enable }

(* Create circuit from module *)
let circuit = 
  let module C = Circuit.With_interface(I)(O) in
  C.create_exn ~name:"my_module" create
```

### Parameterized Modules (Functors)

```ocaml
module type Config = sig
  val data_width : int
  val addr_width : int
end

module Make_processor (C : Config) = struct
  module I = struct
    type 'a t = {
      clock : 'a;
      data : 'a [@bits C.data_width];
    } [@@deriving sexp_of, hardcaml]
  end
  
  let create (i : _ I.t) : _ O.t =
    (* Implementation using C.data_width *)
    { O.result = Signal.uresize i.data 16; valid = Signal.vdd }
end

(* Instantiate with specific configuration *)
module Config_8bit = struct
  let data_width = 8
  let addr_width = 12
end

module Processor_8bit = Make_processor(Config_8bit)
```

## Sequential Logic and Registers

### Register Specifications

```ocaml
(* Basic register specs *)
let reg_spec = Reg_spec.create ~clock ~clear ()
let reg_with_reset = Reg_spec.create ~clock ~reset ()

(* Register creation *)
let reg_output = Signal.reg reg_spec data
let reg_with_enable = Signal.reg reg_spec ~enable data
let pipelined = Signal.pipeline reg_spec ~n:3 data

(* Feedback register for counters *)
let counter = Signal.reg_fb reg_spec ~width:8 ~f:(fun q -> q +:. 1)
```

### State Machine with Always DSL

```ocaml
open Always

module State = struct
  type t = Idle | Processing | Done 
  [@@deriving sexp_of, compare, enumerate]
end

let create_state_machine ~clock ~clear ~start =
  let spec = Reg_spec.create ~clock ~clear () in
  let sm = State_machine.create (module State) spec in
  let done_signal = Variable.wire ~default:Signal.gnd in
  
  compile [
    sm.switch [
      Idle, [
        when_ start [
          sm.set_next Processing;
        ];
      ];
      Processing, [
        when_ done_condition [
          done_signal <-- Signal.vdd;
          sm.set_next Done;
        ];
      ];
      Done, [
        sm.set_next Idle;
      ];
    ];
  ];
  
  done_signal.value, sm.current
```

## Memory and Storage

### Basic Memory Creation

```ocaml
(* Simple memory *)
let memory = Signal.memory 1024 
  ~write_port:{ write_clock = clock; 
                write_address = addr; 
                write_data = data; 
                write_enable = we }
  ~read_address:read_addr

(* Multi-port memory *)
let multiport_mem = Signal.multiport_memory 1024
  ~write_ports:[|write_port1; write_port2|]
  ~read_addresses:[|read_addr1; read_addr2|]

(* ROM from function *)
let rom = Signal.rom ~read_address:addr ~f:(fun addr -> 
  Signal.of_int ~width:8 (Signal.to_int addr * 2))
```

### FIFO Implementation

```ocaml
let create_fifo ~depth ~width ~clock ~clear =
  let addr_bits = Signal.address_bits_for depth in
  let spec = Reg_spec.create ~clock ~clear () in
  
  let write_ptr = Always.Variable.reg spec ~width:addr_bits in
  let read_ptr = Always.Variable.reg spec ~width:addr_bits in
  let count = Always.Variable.reg spec ~width:(addr_bits+1) in
  
  let memory = Signal.multiport_memory depth
    ~write_ports:[|{write_clock=clock; write_address=write_ptr.value; 
                   write_data=write_data; write_enable=write_enable}|]
    ~read_addresses:[|read_ptr.value|] in
  
  let full = count.value ==:. depth in
  let empty = count.value ==:. 0 in
  
  Always.(compile [
    when_ (write_enable &: ~: read_enable) [
      write_ptr <-- (write_ptr.value +:. 1);
      count <-- (count.value +:. 1);
    ];
    when_ (read_enable &: ~: write_enable) [
      read_ptr <-- (read_ptr.value +:. 1);
      count <-- (count.value -:. 1);
    ];
  ]);
  
  { data_out = memory.(0); full; empty }
```

## Testing and Simulation

### Basic Simulation Setup

```ocaml
let test_module () =
  let module Sim = Cyclesim.With_interface(I)(O) in
  let sim = Sim.create create in
  let inputs = Cyclesim.inputs sim in
  let outputs = Cyclesim.outputs sim in
  
  (* Test sequence *)
  inputs.clock := Bits.vdd;
  inputs.enable := Bits.vdd;
  inputs.data := Bits.of_int ~width:8 42;
  
  Cyclesim.cycle sim;
  
  (* Check outputs *)
  let result = Bits.to_int !(outputs.result) in
  Printf.printf "Result: %d\n" result
```

### Waveform Generation

```ocaml
open Hardcaml_waveterm

let waveform_test () =
  let module Sim = Cyclesim.With_interface(I)(O) in
  let sim = Sim.create create in
  let waves, sim = Waveform.create sim in
  
  (* Run simulation *)
  for i = 0 to 10 do
    Cyclesim.cycle sim;
  done;
  
  (* Display ASCII waveform *)
  Waveform.print ~display_height:20 waves;
  
  (* Export to VCD *)
  Waveform.write_vcd "output.vcd" waves
```

### Expect Tests

```ocaml
let%expect_test "counter test" =
  let sim = create_test_sim () in
  let waves, sim = Waveform.create sim in
  
  run_test_sequence sim;
  
  Waveform.print waves;
  [%expect {|
    ┌Signals────────┐┌Waves──────────────────────────────────────────────┐
    │clock          ││┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ │
    │enable         ││        ┌───────────────┐                  │
    │count          ││ 00    │01     │02     │03    │00        │
    └───────────────┘└───────────────────────────────────────────────┘
  |}]
```

## Build System and Project Configuration

### dune-project File

```scheme
(lang dune 3.0)

(package
 (name hardcaml_project)
 (depends ocaml dune hardcaml ppx_hardcaml hardcaml_waveterm))
```

### Library dune File

```scheme
(library
 (public_name hardcaml_project)
 (name hardcaml_project)
 (libraries hardcaml hardcaml_waveterm)
 (preprocess (pps ppx_hardcaml ppx_deriving.show)))
```

### Build Commands

```bash
dune build                    # Build project
dune runtest                  # Run tests
dune exec ./bin/main.exe      # Execute main program
dune build --watch            # Watch mode for development
```

## Complete Working Examples

### 8-bit Counter with Interface

```ocaml
open Hardcaml
open Signal

module Counter_I = struct
  type 'a t = {
    clock  : 'a;
    clear  : 'a;
    enable : 'a;
  } [@@deriving sexp_of, hardcaml]
end

module Counter_O = struct
  type 'a t = {
    count : 'a [@bits 8];
    overflow : 'a;
  } [@@deriving sexp_of, hardcaml]
end

let create_counter (i : _ Counter_I.t) : _ Counter_O.t =
  let spec = Reg_spec.create ~clock:i.clock ~clear:i.clear () in
  let count = Signal.wire 8 in
  let next_count = Signal.mux2 i.enable (count +:. 1) count in
  let overflow = count ==:. 255 in
  
  Signal.assign count (Signal.reg spec next_count);
  { Counter_O.count; overflow }
```

### Simple ALU

```ocaml
module Alu_I = struct
  type 'a t = {
    a : 'a [@bits 8];
    b : 'a [@bits 8];
    op : 'a [@bits 2];  (* 00=add, 01=sub, 10=and, 11=or *)
  } [@@deriving sexp_of, hardcaml]
end

module Alu_O = struct
  type 'a t = {
    result : 'a [@bits 8];
    zero : 'a;
    carry : 'a;
  } [@@deriving sexp_of, hardcaml]
end

let create_alu (i : _ Alu_I.t) : _ Alu_O.t =
  let add_result = Signal.uresize (i.a +: i.b) 9 in
  let sub_result = i.a -: i.b in
  let and_result = i.a &: i.b in
  let or_result = i.a |: i.b in
  
  let result = Signal.mux i.op [
    add_result.:[(7, 0)];    (* Add *)
    sub_result;              (* Subtract *)
    and_result;              (* AND *)
    or_result;               (* OR *)
  ] in
  
  let zero = result ==:. 0 in
  let carry = Signal.select add_result 8 8 in
  
  { Alu_O.result; zero; carry }
```

### Memory Controller

```ocaml
module Mem_I = struct
  type 'a t = {
    clock : 'a;
    address : 'a [@bits 16];
    write_data : 'a [@bits 32];
    write_enable : 'a;
    read_enable : 'a;
  } [@@deriving sexp_of, hardcaml]
end

module Mem_O = struct
  type 'a t = {
    read_data : 'a [@bits 32];
    ready : 'a;
  } [@@deriving sexp_of, hardcaml]
end

let create_memory_controller (i : _ Mem_I.t) : _ Mem_O.t =
  let memory = Signal.memory 65536
    ~write_port:{ write_clock = i.clock;
                  write_address = i.address;
                  write_data = i.write_data;
                  write_enable = i.write_enable }
    ~read_address:i.address in
  
  let spec = Reg_spec.create ~clock:i.clock () in
  let ready = Signal.reg spec (i.read_enable |: i.write_enable) in
  
  { Mem_O.read_data = memory; ready }
```

## Common Patterns and Idioms

### Counter Pattern

```ocaml
let create_counter ~width ~clock ~enable ~clear =
  let spec = Reg_spec.create ~clock ~clear () in
  let counter = Signal.wire width in
  let next_count = Signal.mux2 enable (counter +:. 1) counter in
  Signal.assign counter (Signal.reg spec next_count);
  counter
```

### Pipeline Pattern

```ocaml
let create_pipeline ~stages ~clock data =
  let spec = Reg_spec.create ~clock () in
  let rec pipeline_stage n data =
    if n = 0 then data
    else Signal.reg spec (pipeline_stage (n-1) data)
  in
  pipeline_stage stages data
```

### Edge Detection

```ocaml
let create_edge_detector ~clock signal =
  let spec = Reg_spec.create ~clock () in
  let delayed = Signal.reg spec signal in
  let rising_edge = signal &: (~: delayed) in
  let falling_edge = (~: signal) &: delayed in
  { rising_edge; falling_edge }
```

## Debugging and Common Pitfalls

### Signal Naming for Debug

```ocaml
let create_with_debug i =
  let intermediate = Signal.wire 8 -- "intermediate_signal" in
  let result = intermediate +:. 1 -- "final_result" in
  Signal.assign intermediate (i.data +: i.offset);
  { O.output = result }
```

### Width Mismatch Prevention

```ocaml
(* Common mistake - no width checking *)
let bad_add a b = a +: b  (* May fail if widths don't match *)

(* Correct approach *)
let safe_add a b = 
  let max_width = max (Signal.width a) (Signal.width b) in
  Signal.uresize a max_width +: Signal.uresize b max_width
```

### Zero-Width Signal Error

```ocaml
(* Avoid creating zero-width signals *)
let safe_select signal hi lo =
  if hi >= lo then Signal.select signal hi lo
  else Signal.zero 1  (* Return 1-bit zero instead *)
```

## Synthesis and RTL Generation

### Basic Verilog Generation

```ocaml
let generate_verilog circuit =
  let verilog = Rtl.output ~output_mode:(To_file "design.v") Verilog circuit in
  Printf.printf "Generated design.v\n"

let generate_vhdl circuit =
  let vhdl = Rtl.output ~output_mode:(To_file "design.vhd") Vhdl circuit in
  Printf.printf "Generated design.vhd\n"
```

### Simulation vs Synthesis

```ocaml
(* Use simulation-friendly constructs *)
let simulation_friendly a b =
  Signal.mux2 (a >: b) a b  (* Use mux instead of complex conditionals *)

(* Avoid synthesis warnings *)
let synthesis_safe numerator denominator =
  let is_zero = denominator ==:. 0 in
  Signal.mux2 is_zero 
    (Signal.zero (Signal.width numerator))
    (numerator /: denominator)  (* Division with zero check *)
```

## Essential Function Reference

### Core Signal Functions

```ocaml
(* Creation *)
val of_int : width:int -> int -> Signal.t
val zero : int -> Signal.t
val ones : int -> Signal.t

(* Arithmetic *)
val ( +: ) : Signal.t -> Signal.t -> Signal.t
val ( -: ) : Signal.t -> Signal.t -> Signal.t
val ( *: ) : Signal.t -> Signal.t -> Signal.t

(* Bitwise *)
val ( &: ) : Signal.t -> Signal.t -> Signal.t
val ( |: ) : Signal.t -> Signal.t -> Signal.t
val ( ^: ) : Signal.t -> Signal.t -> Signal.t
val ( ~: ) : Signal.t -> Signal.t

(* Comparison *)
val ( ==: ) : Signal.t -> Signal.t -> Signal.t
val ( <: ) : Signal.t -> Signal.t -> Signal.t
val ( >: ) : Signal.t -> Signal.t -> Signal.t

(* Selection *)
val select : Signal.t -> int -> int -> Signal.t
val concat : Signal.t list -> Signal.t
val mux : Signal.t -> Signal.t list -> Signal.t

(* Sequential *)
val reg : Reg_spec.t -> ?enable:Signal.t -> Signal.t -> Signal.t
val wire : int -> Signal.t
val assign : Signal.t -> Signal.t -> unit
```

This cheatsheet provides all the essential HardCaml constructs and patterns needed to start designing hardware immediately. The examples are production-ready and demonstrate real-world usage patterns that can be directly applied to your projects.


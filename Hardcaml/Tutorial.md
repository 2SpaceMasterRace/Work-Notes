# Hardcaml Tutorial: Learn by Example

## Introduction: What is Hardcaml?

Hardcaml is a Hardware Construction Language (HCL) built as an OCaml library. Unlike traditional Hardware Description Languages (HDLs) like Verilog or VHDL, Hardcaml allows you to describe hardware using the full power of a programming language.

**Key Points:**
- Hardcaml programs build data structures representing hardware designs
- It can convert designs to Verilog/VHDL for synthesis
- You get all the benefits of OCaml (type system, tooling, libraries)
- It's a meta-programming approach: writing programs that generate hardware

## Chapter 1: Basic Signals and Operations

### Concept
In Hardcaml, everything revolves around `Signal.t` - the basic building block representing wires in your circuit. Signals have a width (number of bits) and can be constants, inputs, or derived from operations.

### Example
```ocaml
open Hardcaml
open Signal

(* Create a 4-bit constant signal with value 5 *)
let const_signal = of_int ~width:4 5

(* Create input signals *)
let a = input "a" 8  (* 8-bit input named "a" *)
let b = input "b" 8  (* 8-bit input named "b" *)

(* Basic operations *)
let sum = a +: b     (* Addition *)
let diff = a -: b    (* Subtraction *)
let and_result = a &: b  (* Bitwise AND *)
let or_result = a |: b   (* Bitwise OR *)
let xor_result = a ^: b  (* Bitwise XOR *)
let not_result = ~:(a)   (* Bitwise NOT *)
```

### Exercise 1: Full Adder
Create a full adder circuit that takes three 1-bit inputs (a, b, carry_in) and produces two outputs (sum, carry_out).

```ocaml
open Hardcaml
open Signal

let full_adder a b carry_in =
  (* Your implementation here *)
  (* Hint: sum = a XOR b XOR carry_in *)
  (* Hint: carry_out = (a AND b) OR (carry_in AND (a XOR b)) *)
  let sum = (* TODO *) in
  let carry_out = (* TODO *) in
  sum, carry_out
```

**Solution:**
```ocaml
let full_adder a b carry_in =
  let sum = a ^: b ^: carry_in in
  let carry_out = (a &: b) |: (carry_in &: (a ^: b)) in
  sum, carry_out
```

## Chapter 2: Multi-bit Operations and Concatenation

### Concept
Hardcaml provides powerful operations for working with multi-bit signals, including concatenation, bit selection, and zero/sign extension.

### Example
```ocaml
(* Concatenation *)
let upper = of_int ~width:4 0xA
let lower = of_int ~width:4 0xB
let combined = concat_msb [upper; lower]  (* Results in 8'hAB *)

(* Bit selection *)
let byte_signal = of_int ~width:8 0xFF
let upper_nibble = select byte_signal 7 4  (* Bits 7 down to 4 *)
let bit_3 = bit byte_signal 3              (* Just bit 3 *)

(* Extension *)
let small = of_int ~width:4 (-2)
let zero_extended = uresize small 8   (* Zero extend to 8 bits *)
let sign_extended = sresize small 8   (* Sign extend to 8 bits *)
```

### Exercise 2: Barrel Shifter
Create a 4-bit barrel shifter that can rotate its input by a variable amount.

```ocaml
let barrel_shifter data shift_amount =
  (* Your implementation here *)
  (* Hint: Use mux and multiple shifted versions *)
  (* shift_amount is 2 bits (0-3) *)
  (* TODO *)
```

**Solution:**
```ocaml
let barrel_shifter data shift_amount =
  let shift0 = data in
  let shift1 = concat_msb [select data 2 0; select data 3 3] in
  let shift2 = concat_msb [select data 1 0; select data 3 2] in
  let shift3 = concat_msb [bit data 0; select data 3 1] in
  mux shift_amount [shift0; shift1; shift2; shift3]
```

## Chapter 3: Multiplexers and Conditional Logic

### Concept
Multiplexers (mux) are fundamental building blocks for conditional logic in hardware. Hardcaml provides several mux variants.

### Example
```ocaml
(* 2-to-1 mux *)
let result = mux2 sel on_false on_true

(* Multi-way mux *)
let sel = of_int ~width:2 2
let options = [sig_a; sig_b; sig_c; sig_d]
let result = mux sel options  (* Selects sig_c when sel=2 *)

(* Conditional assignment *)
let result = 
  sel_list
    ~default:zero_value
    [ condition1, value1
    ; condition2, value2
    ; condition3, value3 ]
```

### Exercise 3: Priority Encoder
Create a 4-bit priority encoder that outputs the position of the highest set bit.

```ocaml
let priority_encoder input_vector =
  (* Your implementation here *)
  (* Input: 4-bit vector *)
  (* Output: 2-bit position (0-3) and valid bit *)
  (* Example: 4'b1010 -> position=3, valid=1 *)
  (* TODO *)
```

**Solution:**
```ocaml
let priority_encoder input_vector =
  let bit3 = bit input_vector 3 in
  let bit2 = bit input_vector 2 in
  let bit1 = bit input_vector 1 in
  let bit0 = bit input_vector 0 in
  
  let valid = reduce ~f:(|:) (bits_lsb input_vector) in
  let position = 
    sel_list
      ~default:(of_int ~width:2 0)
      [ bit3, of_int ~width:2 3
      ; bit2 &: ~:(bit3), of_int ~width:2 2
      ; bit1 &: ~:(bit3) &: ~:(bit2), of_int ~width:2 1
      ; bit0 &: ~:(bit3) &: ~:(bit2) &: ~:(bit1), of_int ~width:2 0 ]
  in
  position, valid
```

## Chapter 4: Sequential Logic and Registers

### Concept
Sequential logic requires registers (flip-flops) that store state across clock cycles. Hardcaml provides the `reg` function and its variants.

### Example
```ocaml
module I = struct
  type 'a t = {
    clock : 'a;
    clear : 'a;
    enable : 'a;
    d : 'a[@bits 8];
  } [@@deriving hardcaml]
end

module O = struct
  type 'a t = {
    q : 'a[@bits 8];
  } [@@deriving hardcaml]
end

let create (i : _ I.t) =
  let open Signal in
  let spec = Reg_spec.create ~clock:i.clock ~clear:i.clear () in
  { O.q = reg spec ~enable:i.enable i.d }
```

### Exercise 4: 4-bit Counter
Create a 4-bit up counter with enable and synchronous reset.

```ocaml
module Counter_I = struct
  type 'a t = {
    clock : 'a;
    reset : 'a;
    enable : 'a;
  } [@@deriving hardcaml]
end

module Counter_O = struct
  type 'a t = {
    count : 'a[@bits 4];
  } [@@deriving hardcaml]
end

let counter (i : _ Counter_I.t) =
  (* Your implementation here *)
  (* TODO *)
```

**Solution:**
```ocaml
let counter (i : _ Counter_I.t) =
  let open Signal in
  let spec = Reg_spec.create ~clock:i.clock () in
  let count = wire 4 in
  let next_count = 
    mux2 i.reset 
      (of_int ~width:4 0)
      (mux2 i.enable (count +: of_int ~width:4 1) count)
  in
  count <== reg spec next_count;
  { Counter_O.count }
```

## Chapter 5: Finite State Machines (FSMs)

### Concept
FSMs are fundamental to digital design. Hardcaml provides elegant ways to express state machines using variant types.

### Example
```ocaml
module States = struct
  type t = Idle | Active | Done [@@deriving sexp, compare, enumerate]
end

let fsm ~clock ~reset ~start ~done_signal =
  let open Signal in
  let state = State_machine.create (module States) ~enable:vdd in
  let { State_machine.current; next; set_next } = state in
  
  let () = 
    match current with
    | Idle -> when_ start (set_next Active)
    | Active -> when_ done_signal (set_next Done)
    | Done -> set_next Idle
  in
  
  State_machine.output ~default:gnd state ~f:(function
    | Idle -> gnd
    | Active -> vdd
    | Done -> gnd)
```

### Exercise 5: Traffic Light Controller
Create a simple traffic light controller with Red (3 cycles), Yellow (1 cycle), and Green (3 cycles) states.

```ocaml
module Traffic_States = struct
  type t = Red | Yellow | Green [@@deriving sexp, compare, enumerate]
end

let traffic_light ~clock ~reset =
  (* Your implementation here *)
  (* Output: 3-bit signal for R, Y, G lights *)
  (* TODO *)
```

**Solution:**
```ocaml
let traffic_light ~clock ~reset =
  let open Signal in
  let spec = Reg_spec.create ~clock ~reset () in
  
  let state = State_machine.create (module Traffic_States) 
    ~spec ~enable:vdd in
  let { State_machine.current; set_next; _ } = state in
  
  let counter = reg_fb spec ~enable:vdd ~width:2 ~f:(fun counter ->
    mux2 (counter ==: of_int ~width:2 2) 
      (of_int ~width:2 0) 
      (counter +: of_int ~width:2 1)
  ) in
  
  let () =
    match current with
    | Red -> when_ (counter ==: of_int ~width:2 2) (set_next Green)
    | Green -> when_ (counter ==: of_int ~width:2 2) (set_next Yellow)
    | Yellow -> set_next Red
  in
  
  let r = State_machine.is state Red in
  let y = State_machine.is state Yellow in
  let g = State_machine.is state Green in
  concat_lsb [r; y; g]
```

## Chapter 6: Memories and Arrays

### Concept
Hardcaml supports various memory primitives including RAMs, ROMs, and FIFOs.

### Example
```ocaml
(* Simple RAM *)
let ram ~clock ~write_enable ~address ~data_in =
  let open Signal in
  Ram.create ~collision_mode:Read_before_write
    ~size:256 ~write_port:{write_clock=clock; 
                          write_enable; 
                          write_address=address; 
                          write_data=data_in}
    ~read_port:{read_clock=clock; 
                read_enable=vdd; 
                read_address=address}

(* ROM from array *)
let rom_data = [| 0x12; 0x34; 0x56; 0x78 |]
let rom ~address =
  mux address (List.map ~f:(of_int ~width:8) (Array.to_list rom_data))
```

### Exercise 6: FIFO Buffer
Create a simple 4-element FIFO with write/read interfaces.

```ocaml
module Fifo_I = struct
  type 'a t = {
    clock : 'a;
    reset : 'a;
    write_enable : 'a;
    write_data : 'a[@bits 8];
    read_enable : 'a;
  } [@@deriving hardcaml]
end

module Fifo_O = struct
  type 'a t = {
    read_data : 'a[@bits 8];
    empty : 'a;
    full : 'a;
  } [@@deriving hardcaml]
end

let fifo (i : _ Fifo_I.t) =
  (* Your implementation here *)
  (* TODO *)
```

**Solution:**
```ocaml
let fifo (i : _ Fifo_I.t) =
  let open Signal in
  let spec = Reg_spec.create ~clock:i.clock ~reset:i.reset () in
  
  (* Pointers *)
  let write_ptr = reg_fb spec ~enable:vdd ~width:2 ~f:(fun ptr ->
    mux2 (i.write_enable &: ~:(i.Fifo_I.full))
      (ptr +: of_int ~width:2 1) ptr
  ) in
  
  let read_ptr = reg_fb spec ~enable:vdd ~width:2 ~f:(fun ptr ->
    mux2 (i.read_enable &: ~:(i.Fifo_I.empty))
      (ptr +: of_int ~width:2 1) ptr
  ) in
  
  (* Memory *)
  let memory = Ram.create ~collision_mode:Read_before_write
    ~size:4
    ~write_port:{
      write_clock=i.clock;
      write_enable=i.write_enable &: ~:(full);
      write_address=write_ptr;
      write_data=i.write_data;
    }
    ~read_port:{
      read_clock=i.clock;
      read_enable=vdd;
      read_address=read_ptr;
    } in
  
  (* Status *)
  let count = reg_fb spec ~enable:vdd ~width:3 ~f:(fun count ->
    let inc = i.write_enable &: ~:(full) &: ~:(i.read_enable |: empty) in
    let dec = i.read_enable &: ~:(empty) &: ~:(i.write_enable |: full) in
    mux2 inc (count +: of_int ~width:3 1)
      (mux2 dec (count -: of_int ~width:3 1) count)
  ) in
  
  let empty = count ==: of_int ~width:3 0 in
  let full = count ==: of_int ~width:3 4 in
  
  { Fifo_O.read_data = memory; empty; full }
```

## Chapter 7: Pipelining and Timing

### Concept
Pipelining is crucial for high-performance hardware. Hardcaml provides utilities to help manage pipeline stages.

### Example
```ocaml
(* Manual pipeline *)
let pipelined_multiply ~clock a b =
  let open Signal in
  let spec = Reg_spec.create ~clock () in
  
  (* Stage 1: Compute partial products *)
  let partial1 = a *: (b &: of_int ~width:(width b) 1) in
  let partial2 = (a *: (b &: of_int ~width:(width b) 2)) <<: 1 in
  
  (* Register between stages *)
  let partial1_reg = reg spec partial1 in
  let partial2_reg = reg spec partial2 in
  
  (* Stage 2: Add partial products *)
  let result = partial1_reg +: partial2_reg in
  reg spec result

(* Using pipeline function *)
let pipelined_adder_tree ~clock values =
  let rec tree = function
    | [] -> []
    | [x] -> [x]
    | values ->
      let pairs = List.chunks_of ~length:2 values in
      let sums = List.map pairs ~f:(function
        | [a; b] -> a +: b
        | [a] -> a
        | _ -> assert false) in
      tree (List.map ~f:(reg (Reg_spec.create ~clock ())) sums)
  in
  List.hd_exn (tree values)
```

### Exercise 7: Pipelined Multiplier
Create a 2-stage pipelined 4x4 bit multiplier.

```ocaml
let pipelined_multiplier ~clock ~a ~b =
  (* Your implementation here *)
  (* Stage 1: Generate partial products *)
  (* Stage 2: Add partial products *)
  (* TODO *)
```

**Solution:**
```ocaml
let pipelined_multiplier ~clock ~a ~b =
  let open Signal in
  let spec = Reg_spec.create ~clock () in
  
  (* Stage 1: Generate all partial products *)
  let pp0 = mux2 (bit b 0) (zero 4) a in
  let pp1 = mux2 (bit b 1) (zero 5) (uresize a 5 <<: 1) in
  let pp2 = mux2 (bit b 2) (zero 6) (uresize a 6 <<: 2) in
  let pp3 = mux2 (bit b 3) (zero 7) (uresize a 7 <<: 3) in
  
  (* Pipeline registers *)
  let pp0_reg = reg spec pp0 in
  let pp1_reg = reg spec pp1 in
  let pp2_reg = reg spec pp2 in
  let pp3_reg = reg spec pp3 in
  
  (* Stage 2: Add all partial products *)
  let sum01 = uresize pp0_reg 8 +: uresize pp1_reg 8 in
  let sum23 = uresize pp2_reg 8 +: uresize pp3_reg 8 in
  sum01 +: sum23
```

## Chapter 8: Interfaces and Hierarchical Design

### Concept
Hardcaml's interface system using PPX derivers makes it easy to create modular, hierarchical designs.

### Example
```ocaml
(* Define interfaces *)
module Alu_input = struct
  type 'a t = {
    a : 'a[@bits 8];
    b : 'a[@bits 8];
    op : 'a[@bits 2];  (* 00=add, 01=sub, 10=and, 11=or *)
  } [@@deriving hardcaml]
end

module Alu_output = struct
  type 'a t = {
    result : 'a[@bits 8];
    zero : 'a;
    overflow : 'a;
  } [@@deriving hardcaml]
end

(* Hierarchical module *)
let alu (i : _ Alu_input.t) =
  let open Signal in
  let add_result = i.a +: i.b in
  let sub_result = i.a -: i.b in
  let and_result = i.a &: i.b in
  let or_result = i.a |: i.b in
  
  let result = mux i.op 
    [add_result; sub_result; and_result; or_result] in
  
  { Alu_output.
    result;
    zero = result ==: zero 8;
    overflow = gnd; (* Simplified *)
  }

(* Using hierarchy *)
let cpu_datapath ~clock ~instruction =
  let alu1 = alu { Alu_input.
    a = reg_a;
    b = reg_b;
    op = select instruction 7 6;
  } in
  (* Use alu1.result, etc. *)
```

### Exercise 8: Datapath with ALU
Create a simple datapath with two registers and an ALU.

```ocaml
module Datapath_I = struct
  type 'a t = {
    clock : 'a;
    reset : 'a;
    load_a : 'a;
    load_b : 'a;
    data_in : 'a[@bits 8];
    alu_op : 'a[@bits 2];
  } [@@deriving hardcaml]
end

module Datapath_O = struct
  type 'a t = {
    result : 'a[@bits 8];
    zero_flag : 'a;
  } [@@deriving hardcaml]
end

let datapath (i : _ Datapath_I.t) =
  (* Your implementation here *)
  (* TODO *)
```

**Solution:**
```ocaml
let datapath (i : _ Datapath_I.t) =
  let open Signal in
  let spec = Reg_spec.create ~clock:i.clock ~reset:i.reset () in
  
  (* Registers *)
  let reg_a = reg spec ~enable:i.load_a i.data_in in
  let reg_b = reg spec ~enable:i.load_b i.data_in in
  
  (* Instantiate ALU *)
  let alu_out = alu { Alu_input.
    a = reg_a;
    b = reg_b;
    op = i.alu_op;
  } in
  
  { Datapath_O.
    result = alu_out.result;
    zero_flag = alu_out.zero;
  }
```

## Chapter 9: Testing and Simulation

### Concept
Hardcaml provides powerful simulation capabilities for testing your designs.

### Example
```ocaml
(* Create a testbench *)
let testbench () =
  let module Sim = Cyclesim.With_interface(I)(O) in
  let sim = Sim.create my_circuit in
  let inputs = Cyclesim.inputs sim in
  let outputs = Cyclesim.outputs sim in
  
  (* Set inputs *)
  inputs.a := Bits.of_int ~width:8 42;
  inputs.b := Bits.of_int ~width:8 17;
  
  (* Run simulation *)
  Cyclesim.cycle sim;
  
  (* Check outputs *)
  printf "Result: %d\n" (Bits.to_int !(outputs.result));
  
  (* Waveform capture *)
  let waves, sim = Waveform.create sim in
  (* Run simulation with waveform capture *)
  Waveform.print ~display_height:20 ~display_width:80 waves
```

### Exercise 9: Testbench for Counter
Write a comprehensive testbench for the counter from Exercise 4.

```ocaml
let test_counter () =
  (* Your implementation here *)
  (* Test reset, enable, and counting behavior *)
  (* TODO *)
```

**Solution:**
```ocaml
let test_counter () =
  let module Sim = Cyclesim.With_interface(Counter_I)(Counter_O) in
  let sim = Sim.create counter in
  let inputs = Cyclesim.inputs sim in
  let outputs = Cyclesim.outputs sim in
  let waves, sim = Waveform.create sim in
  
  (* Initialize *)
  inputs.clock := Bits.gnd;
  inputs.reset := Bits.vdd;
  inputs.enable := Bits.gnd;
  
  let cycle () =
    inputs.clock := Bits.vdd;
    Cyclesim.cycle sim;
    inputs.clock := Bits.gnd;
    Cyclesim.cycle sim
  in
  
  (* Test reset *)
  cycle ();
  assert (Bits.to_int !(outputs.count) = 0);
  
  (* Release reset and enable *)
  inputs.reset := Bits.gnd;
  inputs.enable := Bits.vdd;
  
  (* Test counting *)
  for i = 0 to 15 do
    assert (Bits.to_int !(outputs.count) = i mod 16);
    cycle ()
  done;
  
  (* Test enable *)
  inputs.enable := Bits.gnd;
  let current = Bits.to_int !(outputs.count) in
  cycle ();
  assert (Bits.to_int !(outputs.count) = current);
  
  (* Display waveform *)
  Waveform.print ~display_height:20 ~display_width:80 waves;
  print_endline "Counter test passed!"
```

## Chapter 10: Advanced Topics

### Concept
Advanced Hardcaml features include:
- Custom primitives and black boxes
- Generate statements and elaboration-time computation
- Integration with external tools
- Optimization techniques

### Example
```ocaml
(* Instantiating external IP *)
let multiplier_ip a b =
  let open Signal in
  let inst = 
    Instantiation.create ()
      ~name:"vendor_multiplier"
      ~inputs:["A", a; "B", b]
      ~outputs:["P", width a + width b]
  in
  Map.find_exn inst "P"

(* Generate-like functionality *)
let generate_adder_tree inputs =
  let rec build_tree = function
    | [] -> failwith "Empty input list"
    | [x] -> x
    | inputs ->
      let pairs = List.chunks_of ~length:2 inputs in
      let sums = List.map pairs ~f:(function
        | [a; b] -> a +: b
        | [a] -> a
        | _ -> assert false)
      in
      build_tree sums
  in
  build_tree inputs

(* Parameterized module generation *)
let make_shift_register ~length ~width ~clock ~enable ~data_in =
  let open Signal in
  let spec = Reg_spec.create ~clock () in
  let stages = 
    List.init length ~f:(fun i ->
      if i = 0 then
        reg spec ~enable data_in
      else
        reg spec ~enable (List.nth_exn stages (i-1))
    )
  in
  List.last_exn stages
```

### Final Exercise: Parameterized FIR Filter
Create a parameterized FIR filter with configurable number of taps.

```ocaml
let fir_filter ~coefficients ~clock ~data_in =
  (* Your implementation here *)
  (* coefficients is a list of integers *)
  (* Create a FIR filter with len(coefficients) taps *)
  (* TODO *)
```

**Solution:**
```ocaml
let fir_filter ~coefficients ~clock ~data_in =
  let open Signal in
  let spec = Reg_spec.create ~clock () in
  let width = width data_in in
  
  (* Create delay line *)
  let delays = 
    List.init (List.length coefficients) ~f:(fun i ->
      if i = 0 then data_in
      else reg spec (List.nth_exn delays (i-1))
    )
  in
  
  (* Multiply and accumulate *)
  let products = 
    List.map2_exn delays coefficients ~f:(fun delayed coeff ->
      delayed *: (of_int ~width coeff)
    )
  in
  
  (* Sum all products *)
  List.fold products ~init:(zero (width + 16)) ~f:(+:)
```

## Summary and Best Practices

### Key Takeaways
1. **Think in Hardware**: Remember that you're describing circuits, not writing sequential software
2. **Use Types**: Leverage OCaml's type system to catch errors at compile time
3. **Modular Design**: Use interfaces and hierarchy for maintainable designs
4. **Test Thoroughly**: Use Hardcaml's simulation features extensively
5. **Pipeline When Needed**: Balance throughput and latency requirements

### Common Pitfalls to Avoid
- Creating combinational loops (use registers to break them)
- Forgetting clock and reset in sequential circuits
- Not considering bit widths in operations
- Over-pipelining simple circuits
- Under-testing edge cases

### Next Steps
- Explore Hardcaml's standard library for more components
- Learn about clock domain crossing techniques
- Study real-world Hardcaml projects
- Practice converting existing Verilog designs to Hardcaml
- Contribute to the Hardcaml ecosystem

## Resources
- [Official Hardcaml Documentation](https://github.com/janestreet/hardcaml)
- [Jane Street Tech Blog posts on Hardcaml](https://blog.janestreet.com/tags/hardcaml/)
- [Hardcaml Examples Repository](https://github.com/janestreet/hardcaml_examples)
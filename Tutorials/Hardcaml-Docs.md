
**HARDCAML TUTORIAL COMING SOON ALONG WITH HOW TO SETUP VIVADO AND ALL**

# 1.1 Why Hardcaml

Most commonly, hardware is designed using (System)Verilog or VHDL. Both are called hardware description languages (HDLs) and provide features to both write and test a hardware design. Each language exposes a subset which is called 'synthesizable,' i.e., able to be turned into hardware. The full language can only be used for testing.

A more recent alternative is High Level Synthesis (HLS). Here, hardware designs are expressed in C or C++ (there may be other hosting languages available, but these are by far the most common), and a sophisticated compiler generates an equivalent hardware design. Notably, the exact cycle-by-cycle timing of the circuit is not specified by the designer and is left up to the compiler. The designer can add various pragmas to guide the compiler's choices in order to come up with a suitable hardware design that meets performance, area, or frequency requirements.

Finally, we come to Hardware Construction Languages (HCLs) - the class to which Hardcaml belongs. Chisel, Clash, MyHDL, PyMtl, and SpinalHDL are other examples of HCLs. Here, hardware designs are described at a structural level within a standard software programming language. Various tooling is provided to test your designs or convert them to an HDL.


# Why use Hardcaml?

[](http://127.0.0.1:8000/hardcaml-docs-site/introduction/why#why-use-hardcaml)

Hardcaml is an OCaml library. Hardcaml designs are programs written in OCaml. When a Hardcaml program executes, it builds a data structure representing the hardware design the user describes. From there, Hardcaml can convert it to a standard HDL, build a simulation model for testing, among various other capabilities we will discuss in this documentation.

In terms of expressing our hardware designs, Hardcaml (and other HCLs) are able to describe the same circuits as normal HDLs like Verilog. Similarly, if you can express it in Hardcaml, then you can also do so in Verilog. Hardware architecture and design knowledge is transferable between these worlds.

When we describe hardware using Hardcaml, we are performing a task called meta-programming. This means we are writing a program to generate something - in this case, a special representation of hardware that Hardcaml understands.

There are a number of advantages to this approach - here are a few:

- Reuse standard OCaml tooling (i.e., editor integration, continuous integration).
    
- Easily create highly parameterized designs. This means we can often write a design once but instantiate it multiple different ways to suit different requirements.
    
- Leverage the powerful OCaml type system to enforce invariants on our types.
    
- Use standard software libraries across our hardware designs and testing frameworks. Quickcheck for testing is a hugely useful example.
    

There are some downsides, of course:

- Generated HDL code (for use with vendor tooling like Vivado or Quartus) is computer generated and nothing like what a human would write.
    
- Hardcaml makes up names in generated code that look like `_8277`. This isn't very helpful when reading the logs from vendor tools. We have some tricks we can play here to make things a bit easier.
    

Of course the above is all a matter of opinion and you should make up your own!

---
# 1.2 Installing the Opensource Release

Hardcaml can be installed with opam. I recommend the following packages to get started

1.3 Quick Overview

Mon May 19 2025

[View Source](https://github.com/janestreet/hardcaml/introduction/quick_overview.md)

Hardcaml is a library for designing and simulating Register Transfer Level (RTL) hardware designs. Hardcaml designs can be converted to Verilog or VHDL for use with vendor synthesis and place and route tools.```bash
opam install hardcaml hardcaml_waveterm ppx_hardcaml
```

This will install the core Hardcaml library, a waveform viewer which is helpful for building tests, and the hardcaml ppx.

To access the latest packages or the most up-to-date version you may want to try the Jane Street bleeding edge opam repository.

```bash
opam repo add janestreet-bleeding https://ocaml.janestreet.com/opam-repository
```


---

# Basic Usage

Hardcaml can be used with utop (or another OCaml top level)

```bash
#require "hardcaml"
open Hardcaml
```

This provides a great environment to learn the `Bits` API. The following utop command makes things nicer to play with

```bash
# install_printer Bits.pp
```

# Dune

The following is an example Dune file for linking to Hardcaml

```bash
(library
  (name my_hardcaml_lib)  (libraries base hardcaml)  (preprocess (pps ppx_jane ppx_hardcaml)))
```

Hardcaml is a library for designing and simulating Register Transfer Level (RTL) hardware designs. Hardcaml designs can be converted to Verilog or VHDL for use with vendor synthesis and place and route tools.

---






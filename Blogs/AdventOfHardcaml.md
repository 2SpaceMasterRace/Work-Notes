For 2024, I spent a few weekends trying to solve some of the puzzles entirely on an FPGA. It’s an interesting challenge adapting classic algorithms like graph traversal, sorting, dynamic programming, and recursion to the unique capabilities of the FPGA. Choosing to target a relatively small FPGA part (84K LUTs, 4 Mbits of RAM) also requires some algorithmic optimizations which would not be a consideration if running on a regular computer.

---
## Why use Hardcaml for this project?

I decided to implement this project in [Hardcaml](https://github.com/janestreet/hardcaml), which is an open-source Hardware Description Language embedded inside of OCaml that we maintain at Jane Street. Hardcaml includes a simulation backend as well as support for compiling down to RTL, so the entire project, from designing the hardware to validating the design to synthesizing it for an FPGA, is done in OCaml.

To get the advantages of implementing software algorithms in hardware, you usually have to “unroll” iterative or sequential operations into a hardware pipeline that can compute many things in parallel. Traditional HDLs offer only very primitive generation features, so this unrolling becomes unwieldy, and tends to result in code that’s hard to read and to reuse across projects.

Hardcaml’s strong type system and expressive metaprogramming capabilities allow us to implement highly flexible circuits concisely using OCaml. This also eliminates the risk of type-confusion bugs (in hardware, everything’s just a wire, but we can assign semantic meanings to those wires using OCaml’s type system), and makes the resultant code easier to understand and adapt. The ability to parametrize components using functors and higher-order functions lets us easily reuse them across designs and projects.

All of this also makes the development experience a lot more efficient and enjoyable, which are particularly imporant traits when working on a side-project.
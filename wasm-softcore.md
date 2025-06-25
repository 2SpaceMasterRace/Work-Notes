*Whatever I understand, put them into a proper meeting ready doc and update logs directly* 

---


NOTE:: Look into WASM 1.0 only

1. What is a softcore? How can I build a WASM softcore? Are there any existing WASM softcores that are available? How are they built and how do they compile? Are there documentation such as blogs for it?

2. Have proof for complexity, understand why it is complex and actually visualize and be able to write a prototype for this

3. What is a WASM engine? How do I prove and validate if HardCaml can write such a thing?

4. How does softcore work on an FPGA?


---


### problems encountered in building wasm softcore

Bad bet in view of x86 bugward compat , wasm evolution, history rhymes.

Architectures decoupled from transport formats with JS (Java bytecode? not on Web, JS won), recoupling costs more than xlation overhead.

Modern Intel h/w parses the ugly old mess that is x86 into wider internal code. If WebAssembly evolves similarly, any wasm CPU designed now might have to do same. Wasm as transfer format w/ fast typecheck & safe syntax (no or simple verifier) more important than burning in SiO2.

**Happy to be shown wrong, and an FPGA wasm CPU would be interesting no matter how things play out longer term. Could lead to full custom SoC in best case. Wanted to avoid coming off as negative. Glass is .x full, find x!**

The main problem I've found with WasMachine (my FPGA implementation) is that I'm not an expert, and probably performance will not be as good as using llvm to compile to a modern CPU, but if this is not a problem...

Another problem I found is that WebAssemble ISA is NOT focused on being directly executable, but in being very compact, but with a previous step to "decompress" it in memory in a more optimized form, it's totally feasable.

I concept it as a hardware accelerator for web pages, but JIT compiling, it make pointless. Also, being stack-based is not as performance as register based. What use case would be useful for something like this, then?

Well, it's a pipe dream. For general purpose computing, I want to get rid of all virtual machines, and reduce all microarchitectures to one. Get rid of ARM and x86, and just use Wasm. We live in a world of compilers, with one type of target, Wasm. Wasm runs everywhere

It seems that such hardware should also be able to decode LEB128 somehow to run instructions after that
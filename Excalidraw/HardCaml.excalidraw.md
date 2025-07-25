---

excalidraw-plugin: parsed
tags: [excalidraw]

---
==⚠  Switch to EXCALIDRAW VIEW in the MORE OPTIONS menu of this document. ⚠== You can decompress Drawing data with the command palette: 'Decompress current Excalidraw file'. For more info check in plugin settings under 'Saving'


# Excalidraw Data

## Text Elements
Questions to Answer ^VBljEG1l

Goals  ^xQD5j8GC

the goal is to design a pipelined core targeting wasm as ISA, instead of RISC-V or X86 ^kMbcfPSu

A bare metal physical implementation of WebAssembly. That's right, a WebAssembly CPU ^jvcbtDtP

Understand the WebAssembly ISA ^myCzTx8T

Implement the mircoarchitecture in HardCaml ^mL3bOCKx

Simulate the design and verify how the 
microarchitecture should work ^7WMRA04X

Test the implementation on an FPGA ^0JKqypPz

Things to do ^LVqytlru

The C compiler compiles your C code to ASM  ^dChsMxwg

(This process occurs for each C source file) ^wv7iLyHk

Then the linker combines all of these into your executable, usually in the embedded world an ELF file, which an OS can interpret and load - think RTOS ^5Jj9KTlw

You can use the objcopy to convert the elf into a .bin, which is just the raw binary code, without any special headers. ^eKVgzBPE

then the assembler turns that into object files. ^v1szDdzc

explain how a C program get compiled and executed on a RISC-V soft-core processor on the ARTY A7 FPGA. Visualize the entire process and go as deep as you can and break down every step of the way. ^6P2nKbEj

Loading Methods: ^kA4zYqts

Direct Programming: ^1vBXTSr1

The ELF file’s contents can be embedded in the FPGA bitstream (e.g., initializing block RAM during synthesis). ^XYk96rcJ

Bootloader: ^1nmXfBt8

A small program in the processor’s memory loads the ELF file from an external source (e.g., SD card or serial port). ^gGzOrEZ5

Debugger: ^YbUOAEUS

Tools like OpenOCD use JTAG to upload the ELF file into memory and set the program counter. ^cRjCJihv

%%
## Drawing
```compressed-json
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
```
%%
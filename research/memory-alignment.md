# 64-bit processor between caches and RAM.
A 64-bit processor (specifically x86-64 or modern ARM) does not divide the cache based on the 64-bit word size, but rather into fixed-size blocks called cache 
lines, which are typically 64 bytes (512 bits) long. The processor divides memory into contiguous, 64-byte aligned chunks. When data is requested, the CPU 
fetches the entire 64-byte block containing that data from main memory (RAM) and loads it into the cache.

How Cache Lines are Structured Block/Line Size (64 Bytes): 
The smallest unit of data transferred between RAM and cache is a cache line, almost universally 64 bytes on modern desktop and server processors.
Addressing (Offset): Since a cache line is 64 bytes (2^6), the lowest 6 bits of a memory address are used as an offset to identify a specific byte within 
that 64-byte line. 
Alignment: Cache lines are aligned on 64-byte boundaries. This means that the starting address of any cache line in memory always ends with six zeros in binary 
(i.e., address divisible by 64).

A memory misalignment occurs when data is stored at an address that is not evenly divisible by the size of the data type or when a single data structure 
straddles the boundary between two cache lines.
While modern 64-bit processors (like x86-64) can technically handle most misaligned accesses, they do so with a performance penalty because the hardware must 
perform extra work to "stitch" the data together.

Common Causes of Misalignment
- Crossing Cache Line Boundaries: The most impactful misalignment happens when an 8-byte value (like a double or long) starts at the 
end of one 64-byte cache line and finishes in the next. The CPU must fetch two different cache lines to retrieve a single value, doubling the memory latency.
- Improper Pointer Casting: Manually casting a character pointer (char*) to a larger type (like int* or double*) often creates unaligned addresses. 
For example, if you have a byte array and try to read a 4-byte integer starting at index 1, the address (e.g., 0x1001) is not divisible by 4.
- Packed Structures: Developers sometimes use "packing" (e.g., #pragma pack(1)) to save space by removing the "padding" bytes that compilers usually insert. 
While this reduces memory usage, it forces subsequent fields to start at unaligned addresses.
- Network Buffers and File I/O: Data received from a network or read from a binary file is often a continuous stream of bytes. 
If a 4-byte header is followed by an 8-byte value, that 8-byte value will naturally start at an offset of 4, which is not 8-byte aligned.

Why Hardware Cares
- Multiple Memory Cycles: Instead of one "slurp" of data, the CPU must perform two separate reads and use internal logic (shifters and maskers) to combine the 
pieces.
- Loss of Atomicity: An aligned 8-byte write is "atomic" (happens all at once). A misaligned write that spans two cache lines can be interrupted in the middle, 
potentially leading to data corruption in multi-threaded programs.
- Hardware Faults: On some strict architectures (like older SPARC or specific ARM modes), misaligned access will trigger a Bus Error or Alignment Trap, 
causing the program to crash immediately.

If a 4-byte integer starts at 0x1001 (4097) which is indivisible by 4, I need padding to make it starts at 0x1004 (4100)
Same an 8-byte integer starts after 4-byte integer which means 0x1004 indivisible by 8. I need to pad by 4 so it starts at 0x1008

Compilers follow a rule called Natural Alignment: a data type should start at an address that is a multiple of its own size.
The 4-byte Integer ExampleThe Problem: Your integer starts at 0x1001. Since 1001 / 4 has a remainder, it is misaligned.
The Fix: The compiler inserts 3 bytes of padding (at 0x1001, 0x1002, and 0x1003).The Result: The integer now starts at 0x1004. 
Because 1004 is divisible by 4, the CPU can grab it in a single, clean operation.

If your 4-byte integer had been at 0x1000 (ending at 0x1003), the next address would be 0x1004. Since 0x1004 is not divisible by 8, the compiler would add 4 
bytes of padding to push the 8-byte integer to 0x1008.

Summary of the "Rules"Char (1 byte): Can start anywhere.
Short (2 bytes): Address must end in 0, 2, 4, 6, 8, A, C, E.
Int (4 bytes): Address must end in 0, 4, 8, C.
Long/Double (8 bytes): Address must end in 0 or 8.
(16,32,64 bytes) (SIMD)


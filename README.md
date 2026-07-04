# python-rust-lab
A repository dedicated to low-level optimizations, memory safety, and high-performance.

Throughout my journey in the development field, I have come across many topics that required research to truly understand what happens under the hood. 
After going through documentation, I found it valuable to compile and share this knowledge in one place—especially for developers who might find it difficult to gather these 
pieces of information scattered across multiple sources.


## 🧠 Core Research
*   [Memory Alignment & CPU Cycles](./research/memory-alignment.md): Why `#[repr(C)]` and padding matter for L1/L2 cache efficiency.
*   [FFI Safety & Bytemuck](./research/ffi-safety-bytemuck.md): Handling uninitialized data and sound casting between Python/Rust.
*   [Python Internals](./research/python-internals.md): `__slots__` vs `msgspec` vs `Numpy` and the impact on memory footprint.
*   [Networking Internals](./research/async-ssl-internals.md): `async` vs `TCP` vs `SSL` and the impact on network sockets
*   [Async_Programming](./research/python-rust-async.md): `async await`, `runtime`, `scheduling` mapping between asyncio and Tokio
*   [Async_Semaphore](./research/semaphore.md): `acquire`, `release`, mapping between Python/Rust

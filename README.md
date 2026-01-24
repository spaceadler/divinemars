# UNIT 99: Divine Mars

> **"A Temple on Mars."**

**UNIT 99** is a **Ring-0 Nock Interpreter** for **ZealOS**.

It runs the Urbit Virtual Machine directly on bare metal x86_64 hardware, bypassing Linux, Unix, and C runtimes entirely.

## 1\. Introduction

**Modern computing is a stack of lies.**

We run virtual machines inside containers inside managed runtimes on top of kernels we didn't compile. We have lost touch with the machine.

**UNIT 99** strips away the abstraction.

- **Host:** ZealOS (The Divine Frontend). A Ring-0, identity-mapped, JIT-compiled operating system.
- **Guest:** Urbit (The Martian Backend). A strict, functional, immutable state machine.

There is no `libc`. There is no `libuv`. There is no virtual memory protection.

There is only **The Loom** (Urbit's memory) mapped directly to **God's RAM** (ZealOS high memory).

## 2\. Architecture

### The Fusion

The project runs as a single System Task (Daemon).

```
[ Hardware (x86_64) ]
|
 <--- UNIT 99 Hooks (Interrupts/Memory)
|
[ HolyC Nock Interpreter ] <--- The Bridge
|
 <--- The Universe
```

### The Memory Model

We utilize the **Identity Map**.

The Urbit Loom is a contiguous 2GB arena allocated in High Memory (above 4GB) to preserve the low 2GB Code Heap for the ZealOS JIT compiler.

- **Loom Base:** `0x400000000` (configurable).
- **Addressing:** 32-bit offsets relative to Loom Base.
- **Allocation:** Custom Bump Allocator (Road Model) implemented in HolyC.

## 3\. Installation & Usage

**Prerequisites:**

- A physical machine or VM (QEMU/VirtualBox) running **ZealOS** (latest fork).
- At least 4GB RAM.

**Build Instructions:**

1.  Clone this repo into `::/Home/Unit99`.
2.  Include the manifest:C
    
    ```
    #include "::/Home/Unit99/Load.CC"
    ```
    
3.  Compile the Nock Kernel:C
    
    ```
    Comp("Nock.CC");
    ```
    
4.  Boot the Ship (Fake Zod):C
    
    ```
    U99Boot("zod");
    ```
    

## 4\. Technical Hurdles (Status)

|     |     |     |
| --- | --- | --- |
| **Component** | **Status** | **Challenge** |
| **Nock Interpreter** | 🚧 WIP | Implementing non-recursive stack to prevent Ring-0 overflow. |
| **Loom Allocator** | 🚧 WIP | Mapping Urbit's "Road" model to a bare metal bump allocator. |
| **Networking** | 🛑 TODO | Intercepting raw UDP packets from `E1000` driver interrupt. |
| **Noun Packing** | ⚠️ RISK | Efficiently packing 32-bit atoms into 64-bit HolyC registers. |

## 5\. Contribution Guidelines

We are operating in **Ring-0**.

- **Pointer Arithmetic:** Must be verified. A bad pointer doesn't segfault; it reboots the machine.
- **Style:** Strict HolyC conventions. Hungarian notation is encouraged.
- **Philosophy:** Minimal code. If you can do it in 10 lines, don't use 20.

**"The Temple is built one stone at a time."**

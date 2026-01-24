# Research

# UNIT 99: Divine Mars – A High-Performance Nock Interpreter on ZealOS Ring-0

## Comprehensive Research Report & Technical Specification

### 1\. Executive Summary

**1.1 The Convergence of Sovereign Architectures**

In the current epoch of computing, the prevailing paradigm relies heavily on deep abstraction stacks. A typical user application floats atop a precarious tower: a managed runtime (like JVM or V8), standard libraries, a kernel interface (syscalls), the kernel itself (Linux/Windows), hypervisors, and finally the hardware. While this architecture provides convenience and safety, it severs the user's connection to the machine and introduces massive complexity overhead—often measured in tens of millions of lines of code.

UNIT 99, codenamed "Divine Mars," is a radical rejection of this status quo. It proposes a fusion of two "First Principles" systems that lie at opposite ends of the philosophical spectrum yet share a disdain for modern bloat: **ZealOS** (the "Divine" frontend) and **Urbit** (the "Martian" backend).

ZealOS, a modernized fork of the Temple Operating System, provides a 64-bit, Ring-0, identity-mapped environment where the programmer has absolute control over every register and memory address. It is a single-address-space operating system that treats the computer as a single, coherent instrument rather than a collection of secure, isolated silos.

Urbit, conversely, is a deterministic, functional operating function. It defines a rigid virtual machine (Nock) and an immutable state progression. It provides what ZealOS lacks: a robust, cryptographically secure identity system and a peer-to-peer network protocol (Ames) designed for permanence.

**1.2 The Strategic Objective**

The objective of UNIT 99 is to execute the Urbit Nock Virtual Machine directly inside the Ring-0 kernel space of ZealOS. By doing so, we aim to construct a "Sovereign Computer"—a machine where the hardware interface is transparent (ZealOS) and the network state is immutable (Urbit).

This fusion eliminates the traditional "User Mode" entirely. The Nock interpreter runs as a high-priority system task (a "Seth" daemon), interacting directly with the network interface controller (NIC) and raw physical memory. This architecture bypasses the context-switching overhead, virtual memory paging latency, and massive driver stacks of Linux/Unix systems, theoretically offering lower latency and higher throughput for the specific workload of processing Ames packets.

**1.3 Scope of Analysis**

This report provides an exhaustive technical specification for UNIT 99. It details the memory topology required to map Urbit’s 2GB "Loom" into ZealOS’s identity-mapped RAM. It analyzes the critical engineering hurdles of porting a 32-bit noun architecture to a 64-bit host. It specifies the interrupt-driven networking model required to bypass the OS stack. Finally, it outlines the "Road" allocation strategy necessary to manage memory without a global garbage collector. This document serves as the primary source material for engineering the system’s initial proof-of-concept.

---

### 2\. Architectural Specifications: The Host Environment (ZealOS)

To successfully implant the Martian kernel (Urbit) into the Divine body (ZealOS), one must possess a nuanced understanding of the host’s anatomy. ZealOS is not merely a "Linux-lite"; it is a distinct genus of operating system with unique constraints and capabilities.

#### 2.1 The ZealOS Ring-0 Execution Model

Standard operating systems rely on x86 protection rings to isolate the kernel (Ring 0) from user applications (Ring 3). This isolation protects the system from crashes but imposes a performance penalty due to context switching (saving/restoring registers, flushing TLBs) whenever an application needs a system resource.

ZealOS operates exclusively in **Ring-0**. Every task, from the window manager to the compiler to the user’s game, runs with full kernel privileges.

- **Implication for UNIT 99:** The Nock interpreter has "God Mode" access. It can execute privileged instructions like `CLI` (Clear Interrupts), modify Control Registers (CR0, CR3), and access any physical memory address directly.
- **Risk Profile:** A bug in the Nock interpreter (e.g., a null pointer dereference) will not cause a segmentation fault; it will likely corrupt system-critical structures or cause a triple fault, instantly rebooting the physical machine. The engineering standard for UNIT 99 must therefore be higher than for a standard userspace `vere` process.

#### 2.2 Identity Mapping and Memory Topology

Memory management in ZealOS is radically simple compared to the paging madness of Linux/Windows. ZealOS uses **Identity Mapping**, meaning Virtual Address $V$ is numerically identical to Physical Address $P$ ($V = P$).

**The Memory Map:**

|     |     |     |
| --- | --- | --- |
| **Address Range** | **Description** | **Usage in UNIT 99** |
| `0x00000000` - `0x0009FFFF` | Real Mode / Legacy | BIOS Data (Reserved) |
| `0x00100000` - `0x7FFFFFFF` | **Code Heap (Low 2GB)** | **Executable Code.** The HolyC Nock Interpreter logic resides here. |
| `0x80000000` - `0xFFFFFFFF` | 3GB-4GB Hole | Memory Mapped I/O (PCI Devices, LAPIC, IOAPIC). |
| `0x100000000` + | **High Memory (>4GB)** | **Data Heap.** The Urbit "Loom" (State) resides here. |

**The Code Heap Constraint:** The HolyC JIT compiler utilizes the x86_64 `CALL` instruction with 32-bit relative offsets (`REL32`). This optimization reduces code size but physically restricts all executable machine code to the lowest 2GB of system memory.

- **Constraint:** The Nock Interpreter functions _must_ reside in the low 2GB.
- **Solution:** The Urbit State (the Loom), which is typically 2GB in size, cannot fit in the Code Heap alongside the OS. It must be allocated in **High Memory** (above 4GB). This necessitates a bifurcated architecture where the _logic_ is low and the _data_ is high.

#### 2.3 The HolyC Language and Runtime

UNIT 99 is implemented in **HolyC** (formerly C+), a dialect of C optimized for system tasks.

- **Just-In-Time (JIT):** HolyC is compiled on demand. There are no object files or linkers in the traditional sense. The `Load("Nock.CC")` command parses, optimizes, and compiles the code directly into memory.
- **64-Bit Type System:** The primitive integer type is `I64` (signed 64-bit int). While `I8`, `I16`, and `I32` exist for storage, all calculations are promoted to 64-bit registers (`RAX`, `RBX`, etc.). This creates a friction point with Urbit's 32-bit atom architecture, requiring careful bitwise masking.
- **No** `libc`**:** There is no standard library. Functions like `malloc`, `printf`, and `socket` do not exist. Their ZealOS equivalents are `MAlloc`, `Print`, and raw NIC driver hooks.

#### 2.4 Task Scheduling: Adam, Seth, and the User

ZealOS uses a Round-Robin scheduler with simple priority levels.

- **Adam Task:** The "init" process. It spawns at boot and owns the persistent system heap.
- **Seth Tasks:** These are worker daemons or system tasks.
- **User Tasks:** Interactive terminals.

UNIT 99 is designed to run as a **System Task** (Seth). It essentially monopolizes a CPU core (e.g., Core 1 on a multicore system), running a continuous event loop that checks for network packets or timer events. Because ZealOS is non-preemptive (cooperative multitasking), the Nock interpreter must explicitly `Yield()` to the scheduler occasionally to allow the system to handle other interrupts, though on a dedicated core, this is less critical.

---

### 3\. Architectural Specifications: The Guest Environment (Urbit)

To implement the guest, we must deconstruct Urbit into its atomic components. Urbit is not just software; it is a formal definition of a computer.

#### 3.1 Nock: The Axiomatic Combinator

Nock is a stateless functional interpreter. It does not "run software"; it reduces nouns. A **Noun** is the fundamental data type:

1.  **Atom:** An unsigned integer of arbitrary size.
2.  **Cell:** An ordered pair of two nouns `[head tail]`.

The interpreter function is defined as `*(subject formula)`. The "Subject" is the data context (variables), and the "Formula" is the code (logic).

- **Axioms:** Nock is defined by roughly 12 axioms (0-11). For example, `*[a 0 b]` means "retrieve the noun at tree address `b` from subject `a`."
- **Simplicity:** The specification is small enough to be printed on a t-shirt.
- **Complexity:** While the rules are simple, the structures they build (like the Hoon compiler) are deeply nested binary trees. The interpreter must handle deep recursion efficiently.

#### 3.2 The Loom: Memory as a Deterministic Arena

Urbit’s state is stored in a contiguous memory arena called **The Loom**.

- **Size:** Standard Urbit ships use a 2GB Loom.
- **Determinism:** The Loom is a persistent image. The "Event Log" records all inputs (events). Replaying the event log from the genesis block against the initial state produces the exact same Loom state, bit-for-bit.
- **Addressing:** Internal pointers in Urbit are usually 32-bit offsets relative to the start of the Loom. This allows the Loom to be "portable"—it can be saved to disk, moved to a different machine, and loaded at a different base address without needing to rewrite every pointer in memory.

**Integration Strategy:** UNIT 99 must emulate this Loom. We cannot scatter Nock nouns across the ZealOS heap using standard `MAlloc`. We must allocate a single 2GB block and write a custom allocator that manages strictly this block.

#### 3.3 Ames: The Martian Network

Ames is Urbit’s peer-to-peer network protocol.

- **Transport:** UDP.
- **Addressing:** 128-bit addresses (Planets/Moons) or shorter (Galaxies/Stars).
- **Security:** Packets are encrypted with Elliptic Curve Cryptography (implementation of which is a significant sub-project for UNIT 99).
- **Semantics:** Ames provides exactly-once delivery semantics. It handles fragmentation, retransmission, and flow control at the application layer, treating UDP merely as a dumb pipe.

**ZealOS Fit:** ZealOS has a very thin network stack. This is an advantage. Ames _wants_ a dumb pipe. By capturing raw UDP packets in the Ring-0 interrupt handler, we bypass the overhead of a complex TCP/IP stack that Ames doesn't need anyway.

---

### 4\. Critical Engineering Challenges

The theoretical fit is good, but the engineering reality involves several "Hard Parts" where the two architectures clash.

#### ⚠️ 4.1 Noun Representation: The 32/64-Bit Mismatch

**The Problem:**

Urbit is optimized for 32-bit architectures (originally). A "Direct Atom" is a value less than $2^{31}$. If the high bit is set, the value is interpreted as a pointer (offset).

ZealOS is strictly 64-bit. Its registers (`RAX`, `RCX`) are 64-bit. Its pointers are 64-bit absolute addresses.

**The Friction:**

If we represent a Nock noun as a standard HolyC class:

C

```
class CNoun {
  I64 is_cell;
  CNoun *head;
  CNoun *tail;
};
```

This structure consumes 24 bytes (3 words) per noun. In a system with millions of tiny integers (atoms), this is a massive waste of memory and cache bandwidth.

**The Solution: Tagged Pointer Packing**

We must adopt a **Tagged Pointer** strategy similar to dynamic language runtimes (like V8 or LuaJIT), but adapted for the ZealOS Identity Map.

- **The Register:** We use the full 64-bit register (`U64`) to hold a noun.
- **Tagging:** We use the low 2 bits or high bits to distinguish types.
    - **Direct Atom:** Value shifted left by 1 bit, low bit `1`. (Allows 63-bit integers).
    - **Cell Pointer:** A 32-bit offset into the Loom, low bits `00`.
    - **Indirect Atom Pointer:** A 32-bit offset, low bits `10`.

**Implementation Detail:**

ZealOS pointers are absolute. To dereference a "Cell Pointer" (which is a 32-bit offset $O$), we must perform:

$$P_{physical} = B_{loom} + O$$

Where $B_{loom}$ is the 64-bit base address of our high-memory reservation (e.g., `0x400000000`).

The interpreter must aggressively use HolyC macros to handle this translation without function call overhead.

C

```
#define LOOM_BASE 0x400000000
#define TO_PTR(noun) (LOOM_BASE + (noun & 0xFFFFFFFC))
```

#### ⚠️ 4.2 Allocation Strategy: The Road Model

**The Problem:** Urbit relies on "Roads" for memory management. A Road is a temporary heap.

- **Outer Road:** Long-term state (The Ship).
- **Inner Road:** Ephemeral state (Processing one packet).
- **Leaping:** Results from an inner road must be copied to the outer road before the inner road is destroyed.

ZealOS’s `MAlloc` is a global, thread-safe allocator. It is too slow and creates too much fragmentation for the rapid-fire allocation/deallocation cycle of a Nock interpreter.

**The Solution: A Ring-0 Bump Allocator**

We will implement the Road model manually within our 2GB Loom.

- **State:** We maintain a `U32 cap` (start of free space) and `U32 mat` (end of free space) for the current road.
- **Allocation:** To allocate $N$ words:C
    
    ```
    U32 Alloc(I64 size) {
       U32 ptr = current_road->cap;
       current_road->cap += size;
       return ptr;
    }
    ```
    
    This instruction sequence compiles to just 3-4 assembly instructions.
- **Garbage Collection:** Is effectively instant. When an event finishes processing, we simply reset the `cap` pointer to its original value. All "garbage" generated during the event is conceptually erased.

#### ⚠️ 4.3 The Opcode Dispatcher: Recursion vs. Stack Depth

**The Problem:**

Nock is recursive. The instruction `*[a [b c]]` evaluates `*[a b]` then `*[a c]` then conses them.

A naive C implementation uses the C stack:

C

```
Noun Nock(Noun sub, Noun fol) {
  //...
  return Cell(Nock(sub, head), Nock(sub, tail));
}
```

**The Killer Constraint:** The ZealOS Kernel Stack is fixed and small (typically 128KB per task). Nock recursion depths can easily exceed 10,000 frames. A stack overflow in Ring-0 is a **Double Fault**. The CPU cannot push the error handler onto the stack because the stack is full, leading to a Triple Fault and a hard reboot.

**The Solution: The Virtual Stack**

We must implement a **Trampoline Interpreter**.

Instead of using the CPU’s `RSP` (Stack Pointer), we allocate a large "Virtual Stack" in the high-memory Loom (e.g., 50MB).

- **Frame Structure:**C
    
    ```
    class CFrame {
       U32 subject;
       U32 formula;
       U8  state; // 0=Eval Head, 1=Eval Tail, 2=Cons
       U32 *dest; // Where to write the result
    };
    ```
    
- **The Loop:** The interpreter becomes a single `while` loop that peeks at the top of the Virtual Stack.
    - If `state == 0`: Push a new frame to evaluate the Head.
    - If `state == 1`: Push a new frame to evaluate the Tail.
    - If `state == 2`: Cons the results and Pop.

This allows recursion depth limited only by available RAM (gigabytes), not the kernel stack.

---

### 5\. Detailed System Design: The UNIT 99 Stack

The implementation of UNIT 99 is divided into four distinct layers, moving from the metal up to the user interface.

#### 5.1 Layer 0: The Hardware Hook (Networking)

The first requirement is getting data in and out. ZealOS supports several NICs, with the AMD PCNet32 and Intel E1000 being the most common in virtualization. We focus on the **E1000**.

**Interrupt Vectoring:**

When an Ethernet packet arrives, the E1000 asserts an interrupt (IRQ). The CPU pauses execution and jumps to the Interrupt Descriptor Table (IDT) handler.

In a standard ZealOS network stack, this handler copies the packet to a queue.

**The UNIT 99 Hook:**

We will inject a "Snoop" function at the very top of the Interrupt Service Routine (ISR).

C

```
interrupt U0 E1000_ISR_Hook() {
   // 1. Read Interrupt Cause Register (ICR)
   // 2. If it's a Receive Timer Interrupt (RXT0):
   //    a. Access the RX Ring Buffer via Memory Mapped I/O
   //    b. Inspect the packet header (Offset 14 for IP, Offset 23 for Protocol)
   //    c. If UDP (17) AND Port == 13337:
   //       i.   MemCpy packet to Loom Event Queue
   //       ii.  Acknowledge Interrupt
   //       iii. RETURN (Skip OS processing)
   // 3. Else: Call original OS Handler
}
```

This effectively hides Urbit traffic from the OS. The OS sees a quiet network; the Nock VM sees a stream of Ames packets.

#### 5.2 Layer 1: The Memory Manager (Loom Control)

Before the interpreter runs, we must establish the physical substrate.

**Initialization Routine:**

1.  **Reserve High Memory:** Using `SysMAlloc`, we request 2GB of _uncached_ or _write-combining_ memory if possible, though standard cached RAM is fine for the Loom.
2.  **Zeroing:** The Loom must be zeroed. `MemSet(LOOM_BASE, 0, LOOM_SIZE)`.
3.  **Boot Image:** We need to load an initial Urbit "Pill" (bootstrap image). This can be read from the ZealOS disk (`::/Home/Unit99/urbit.pill`) and unpacked into the Loom. The unpacking process involves deserializing the "Jam" format, which is a run-length encoded serial format.

#### 5.3 Layer 2: The Interpreter (The Engine)

The core logic is the `Work()` function of the Seth task.

**The Event Loop:**

C

```
U0 Unit99Task() {
  while (TRUE) {
    // 1. Check for pending network packets in the Ring Buffer
    if (PacketPending()) {
       // 2. Inject packet as an "Event" into the Nock Subject
       Noun event = Cue(packet_buffer);
       
       // 3. Execute the Urbit Lifecycle Function (Arvo)
       // Formula: The Arvo Core
       // Subject:
       Noun result = Nock(State, Cell(event, ArvoCore));
       
       // 4. Process Effects (New State, Outgoing Packets)
       ProcessEffects(result);
    }
    
    // 5. Yield to prevent system lockup (Cooperative Multitasking)
    Yield; 
  }
}
```

**Instruction Set Implementation:**

The 11 axioms must be implemented as efficiently as possible.

- **Increment (4):** Simple integer addition.
- **Cell Test (3):** Checking the tag bit of the noun.
- **Tree Addressing (0):** This is the "Axis" operator. To find axis `3` of `[a b]`, we take `b`. To find axis `2`, we take `a`. This requires a tree traversal algorithm `Axis(noun, axis)`. Since `axis` can be large, this traversal corresponds to the binary representation of the axis number.

#### 5.4 Layer 3: The Interface (DolDoc)

The user needs to see the Urbit console (Dojo). ZealOS uses a document format called **DolDoc** for all text output. DolDoc supports rich text, macros, and blinking cursors.

**The Console Bridge:**

The Nock interpreter will emit "Effects" (output instructions). One of these effects is `%blit` (terminal output).

When UNIT 99 receives a `%blit` effect:

1.  Parse the text/string.
2.  Call ZealOS `DocPrint(text)`.
3.  This renders the text directly to the active window's framebuffer.

Because we are in Ring-0, we can also implement direct video memory writing (`0xB8000` or VBE buffer) if we want to bypass the window manager entirely for a "FullScreen Matrix" mode.

---

### 6\. Implementation Roadmap & Milestones

This project is divided into three distinct phases of escalating complexity.

#### Phase 1: The Bedrock (Memory & Math)

- **Goal:** Boot a "Fake" Urbit state (a comet) and run simple Nock formulas.
- **Tasks:**
    - Implement the High Memory Loom allocator.
    - Implement `+jam` and `+cue` serialization in HolyC.
    - Implement the Iterative Nock Interpreter (Opcodes 0-5).
    - Verify correct noun packing (32-bit logical / 64-bit physical).

#### Phase 2: The Logic (Flow & State)

- **Goal:** Run the full Arvo lifecycle function.
- **Tasks:**
    - Implement the Virtual Stack to handle Opcode 2 (Eval) and 6 (Cond).
    - Implement "Jet" dashboard. (Jets are C/HolyC accelerators for common Nock functions like math or hashing. Without Jets, Nock is too slow for crypto).
    - Implement SHA-256 (Mug) and Ed25519 crypto in HolyC (or port a C library).

#### Phase 3: The Connection (Ames & Driver)

- **Goal:** Connect to the live Urbit network.
- **Tasks:**
    - Reverse engineer the E1000/PCNet32 driver in ZealOS.
    - Implement the Interrupt Hook.
    - Implement the Ames packet logic (CRC, encryption, fragment reassembly).
    - **Victory Condition:** A successful "Hi" message sent from the ZealOS Dojo to a node on the planetary network.

---

### 7\. Philosophical Objectives: Why Build This?

**7.1 Sovereignty via Simplicity**

Modern "Sovereign Computing" (like standard Urbit) often compromises on the "Computing" part. It claims sovereignty while relying on AWS, Linux, and Intel ME. UNIT 99 argues that true sovereignty requires **Full Stack Comprehension**. A user cannot own what they cannot understand.

- **Linux Kernel:** ~30,000,000 lines of code. (Impossible for one person to understand).
- **ZealOS Kernel:** ~50,000 lines of code. (Readable in a few weeks).
- **Nock Spec:** ~200 words. (Readable in 5 minutes).

By combining ZealOS and Nock, we create a system where a single motivated individual can theoretically audit every line of code that drives their digital life.

**7.2 The Rejection of Protection**

Safety features (Memory Protection, User Mode, Rust borrow checkers) are designed to protect the machine from the user, and users from each other.

On a personal, sovereign computer, **I am the only user**. I do not need protection from myself. I need **Power**.

UNIT 99 trades safety for raw, unadulterated access to the machine's potential. It is a computer for the fearless.

---

### 8\. Conclusion

UNIT 99 is not a practical replacement for a MacBook. It is an exploration of the possible. It demonstrates that the layers of abstraction we accept as "necessary" are, in fact, optional. By fusing the Divine Temple of ZealOS with the Martian Logic of Urbit, we create a chimera that is faster, simpler, and more dangerous than anything produced by Silicon Valley.

**The Temple is Open. The Ships are Landing.**

---

### 9\. Supporting Data & Citations

**Table 1: Architecture Comparison**

|     |     |     |     |
| --- | --- | --- | --- |
| **Feature** | **Standard Urbit (Vere)** | **UNIT 99 (Divine Mars)** | **Implications** |
| **Host OS** | Linux / macOS / Windows | ZealOS (Ring-0) | Zero system call overhead; total hardware control. |
| **Privilege** | User Space (Ring 3) | Kernel Space (Ring 0) | Crash = Reboot; No memory protection. |
| **Memory** | Virtual (Paged) | Identity (Physical) | No TLB miss penalties; fragmentation risks. |
| **Language** | C (with libc) | HolyC (JIT) | Runtime compilation; no standard library bloat. |
| **Network** | Kernel Socket API | Raw Interrupt Hook | Lower latency; requires custom packet parsing. |

**Key Research Sources:**

- **ZealOS Memory Model:** (Identity Mapping, Code Heap limitations).
- **Urbit Internals:** (Loom, Road Model).
- **Nock Specification:** (Axioms, combinator logic).
- **Networking:** (Ames Protocol, Bare-metal interrupts).
- **Interpreter Theory:** (Stack overflow mitigation in interpreters).

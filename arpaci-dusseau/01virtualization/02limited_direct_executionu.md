# Limited Direct Execution

## 6.1 Basic Technique: Limited Direct Execution (LDE)

**Limited Direct Execution (LDE)** is a technique that lets programs run very fast while still allowing the **operating system (OS)** to stay in control.

- **Direct execution** = the OS lets the program run directly on the CPU.
- To start a program, the OS:
  1. Creates a **process entry** in its process list,
  2. Allocates **memory**,
  3. Loads the **program code** from disk,
  4. Finds the **entry point** (e.g., `main()`),
  5. Jumps to the entry point and starts running the program.

This is fast, but it causes two main problems:

1. How does the OS stop a program from doing forbidden things?
2. How does the OS stop one process and switch to another (for time sharing)?

Without extra limits, the OS would not be in control—hence the need for **limited** direct execution.

---

## 6.2 Problem #1: Restricted Operations

Running directly on the CPU is efficient, but dangerous.  
A program could try to:
- access the disk directly,
- steal CPU time,
- read or write memory it should not.

To avoid this, modern CPUs have two modes:

### **User mode**
- Normal programs run here.
- **Restricted**: they cannot perform privileged operations.
- If they try, the CPU raises an **exception**, and the OS usually kills the program.

### **Kernel mode**
- The OS runs here.
- Has **full privileges**:
  - can issue I/O requests,
  - can access all hardware,
  - can manage memory and processes.

---

## System Calls

A program in user mode cannot perform privileged actions directly.  
Instead, it must request them using a **system call**.

A **system call** is a safe way for a program to ask the OS to do something (like read a file or create a process).

### How a system call works

The program uses a **trap instruction**:

- A **trap**:
  - switches the CPU from user mode → kernel mode,
  - jumps into OS code that handles system calls.

After the OS finishes, it uses **return-from-trap** to:
- return to the user program,
- switch the CPU back to user mode.

The CPU automatically:
- saves registers on a **kernel stack** when entering the OS,
- restores them when returning.

---

## Trap Table

When a trap happens, the OS must control which code is run.  
A user program is not allowed to choose a kernel address (too dangerous).

So the OS sets up a **trap table** at boot time.

- The trap table maps events (system calls, interrupts) to specific OS functions.
- The OS installs it using a **privileged instruction**.
- User programs cannot modify the trap table.

---

## System Call Numbers

Each system call has a **system-call number**.

- The program places this number in a register,
- The OS reads the number inside the trap handler,
- If valid, the OS runs the correct system call implementation.

This prevents programs from jumping directly into kernel code.

---

## Summary of the Limited Direct Execution Protocol

There are two phases:

### **Phase 1: Boot time**
- OS installs:
  - the **trap table**
  - privileged operations

### **Phase 2: Running a process**
1. OS creates process entry and allocates memory.
2. OS starts the program using **return-from-trap** → CPU enters **user mode**.
3. Program runs directly on the CPU.
4. When the program needs a privileged operation:
   - it issues a **system call**,
   - CPU traps into the OS (kernel mode),
   - OS performs the operation,
   - OS returns to the program in user mode.
5. When the program finishes, it calls the `exit()` system call.
6. OS cleans up the process.

**Result:** The program runs fast, while the OS maintains full control.


# Simple Explanation of the Text (with all concepts preserved)

## 6.3 Problem #2: Switching Between Processes

Switching between processes sounds simple: the operating system (OS) should just stop one process and start another. But there is a challenge: 

- When a **process** is running on the CPU, the **OS is not running**.
- If the OS is not running, it **cannot take any action**.
- Therefore, the OS needs a safe way to regain control of the CPU.

This leads to two main approaches.

---

## Cooperative Approach: Wait for System Calls

In the *cooperative approach*, used in some old systems (like early Macintosh OS and Xerox Alto), the OS **trusts processes** to behave well.

Processes voluntarily give up the CPU by:

1. Making **system calls**  
   Examples: opening files, reading data, sending messages, creating processes.  
   A system call causes a **trap** into the OS, giving it control again.

2. Calling an explicit **yield** system call  
   This does nothing except return control to the OS.

3. Causing an **illegal operation**  
   For example: division by zero or accessing illegal memory.  
   This triggers an exception (trap) into the OS.

However, this approach has a problem:

- If a process enters an **infinite loop** and never makes a system call, the OS never regains control.
- The only solution in such systems is to **reboot the machine**.

Because of this weak behavior, modern systems do not rely on cooperation alone.

---

## Non-Cooperative Approach: The OS Takes Control

Modern systems use hardware support to forcibly regain control.

The key mechanism is a **timer interrupt**.

### Timer interrupt
- A hardware timer is programmed to interrupt the CPU **every few milliseconds**.
- When the interrupt triggers:
  - The currently running process is paused.
  - Control jumps to a **pre-configured interrupt handler** in the OS.
- Now the OS can decide what to do next.

To make this work:

1. At **boot time**, the OS tells the hardware which code to run when a timer interrupt occurs.  
   (This requires **privileged instructions**.)

2. The OS starts the **timer device**, which begins generating interrupts.  
   (Also a privileged operation.)

3. When an interrupt occurs:
   - The hardware saves enough of the process’s CPU state (registers) on that process’s **kernel stack**.
   - The OS interrupt handler runs in **kernel mode**.

This is similar to how system calls work.

---

## Saving and Restoring Context (Context Switch)

Once the OS regains control (via a system call or timer interrupt), it must decide:

- Should the current process keep running, or
- Should the OS switch to another process?

This decision is made by the **scheduler**.

If a switch is needed, the OS performs a **context switch**.

### Context switch
A context switch saves the state of the current process and restores the state of the next process.

This includes:
- general-purpose registers,
- program counter (PC),
- kernel stack pointer.

Steps in a context switch:

1. Save the register values of the current process  
   (onto its kernel stack or process structure).

2. Load the register values of the next process  
   (from its process structure).

3. Switch to the next process’s **kernel stack**.

4. Execute **return-from-trap**, which restores registers and begins running the next process in user mode.

---

## Two Types of Saving/Restoring

There are two kinds of register saving:

1. **Hardware saves registers**  
   - Happens automatically when a timer interrupt or system call occurs.  
   - Saved onto the process’s kernel stack.

2. **OS saves registers**  
   - Happens during the context switch.  
   - Saves kernel registers into the process structure.  
   - Makes it look as if the next process had just trapped into the OS.

Together, these allow the system to safely switch from process A to process B.

---

## Summary

- A process switch requires the OS to regain control of the CPU.
- Cooperative systems rely on processes making system calls, but this is unreliable.
- Modern systems use **timer interrupts** to forcibly return control to the OS.
- The OS uses **context switching** to save the current process's state and restore another’s.
- This enables multiple processes to share the CPU safely and efficiently.

# Simple Explanation of the Text (with all concepts preserved)

## 6.4 Worried About Concurrency?

The text raises an important question: What happens if a **timer interrupt** occurs while the OS is already handling a **system call** or another **interrupt**?  
This is a classic issue in **concurrency** — dealing with multiple things happening at the same time.

The OS must make sure that overlapping interrupts or traps do not corrupt data or cause unpredictable behavior.

A full discussion is left for the next part of the book (on concurrency), but the text gives a brief overview.

---

## Preventing Interrupt Problems

### Disabling interrupts
A simple technique the OS might use is to **disable interrupts** while it is handling another interrupt.  
This prevents new interrupts from arriving at a bad time.

However, interrupts cannot be disabled for long, because:

- Interrupts could be **lost**  
- Important events (like hardware input) would be missed

Therefore, the OS must disable interrupts carefully and briefly.

---

## Locking inside the OS

Operating systems also use **locking schemes** (synchronization methods) to protect shared data structures inside the kernel.

- Locks ensure that **only one piece of code** accesses a shared data structure at a time.
- This is especially important on **multiprocessor systems**, where the kernel may be running on several CPUs at once.

However, locking is complex and can cause hard-to-find bugs if used incorrectly. The second part of the book focuses on these concurrency issues.

---

## Context Switch Example (xv6)

The text includes assembly code for the `swtch` function in **xv6**, which performs a **context switch**.  
A context switch saves the register state of the current process and restores the state of the next process.

The function does two main things:

1. **Save old registers**  
   - Saves instruction pointer (IP), stack pointer (SP), and general-purpose registers into the `old` context structure.

2. **Load new registers**  
   - Restores registers and stack pointer from the `new` context structure.

Finally, it performs a `ret` instruction, which returns into the newly-loaded context, effectively switching to another process.

---

## Summary of Limited Direct Execution (LDE)

The text summarizes how **CPU virtualization** is implemented using a technique called **limited direct execution**.

### Key idea:
- Run processes directly on the CPU for speed,
- But **restrict** what they can do so they cannot harm the system.

### Baby-proofing analogy
Just like you “baby-proof” a room by covering dangers, the OS “baby-proofs” the CPU:

1. Sets up **trap handlers** at boot time  
2. Starts a **timer interrupt**  
3. Runs applications only in **user mode**

This way, user programs run efficiently but cannot perform dangerous actions without OS oversight.

### When a process:
- Needs a privileged operation → it makes a **system call**
- Runs too long → the **timer interrupt** forces the OS to take back control

Thus, the OS stays in control while letting programs run quickly.

---

## What Comes Next?

Virtualization mechanisms are in place, but a key question remains:

**Which process should run next?**

The **scheduler** makes this decision. Scheduling is the next topic covered after CPU virtualization.

---

# ASIDE: Key CPU Virtualization Terms (Mechanisms)

- CPUs must support at least **two modes**:  
  - **User mode** (restricted)  
  - **Kernel mode** (privileged)

- User applications run in **user mode** and use **system calls** to request operating system services.

- The **trap instruction**:
  - Saves registers,
  - Switches CPU to kernel mode,
  - Jumps to the OS using the **trap table**.

- The **return-from-trap** instruction returns to user mode after the system call.

- The **trap table** is set up at boot time and cannot be modified by user programs.

- The OS uses a **timer interrupt** to ensure programs do not run forever (non-cooperative scheduling).

- The OS may perform a **context switch** during an interrupt or system call to switch from one process to another.

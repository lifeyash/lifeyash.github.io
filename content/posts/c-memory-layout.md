+++
date = '2026-07-03T18:30:00+05:30'
draft = false
title = 'C Memory Layout'
+++


# C Program Memory Layout

Every C program occupies memory while it is running. However, not all data in a program has the same purpose or lifetime. Some data must remain unchanged throughout execution, while other data is created, modified, or destroyed as the program runs.

For example:

- **Machine instructions** must remain unchanged throughout the program's execution.
- **Global and static variables** need to exist for the entire lifetime of the program.
- **Local variables** are created when a function is called and automatically destroyed when it returns.
- **Dynamically allocated memory** exists only until it is explicitly released.

Since each type of data has different requirements, the **compiler** and **linker** organize a program into **separate memory sections** instead of placing everything into one large block of memory.

In a typical embedded system, these sections are mapped into two primary types of memory:

- **Flash (ROM)** – Stores the program instructions and other **read-only data**. Its contents are **non-volatile**, meaning they are retained even after power is removed.

- **RAM** – Stores data that can be **modified during program execution**, including initialized variables, uninitialized variables, the **heap**, and the **stack**.

A simplified view of a typical embedded system looks like this:
```
            Flash (Non-Volatile Memory)
0x0000                              
┌────────────────────────────────────────────┐
│ .text      (Program Instructions)          │
├────────────────────────────────────────────┤
│ .rodata    (Read-Only Data)                │
├────────────────────────────────────────────┤
│ Initial values for .data                   │
└────────────────────────────────────────────┘
0x3FFF
            RAM (Volatile Memory)
0x2000              
┌────────────────────────────────────────────┐
│ .data   (Initialized Variables)            │
├────────────────────────────────────────────┤
│ .bss    (Zero-Initialized Variables)       │
├────────────────────────────────────────────┤
│                        grows towards    │  │
│ Heap                   higher addresses │  │
│                                         ▼  │
│------------ Free Memory -------------------│
│                                         ▲  │
│                        grows toward     |  │
│ Stack                  lower addresses  |  │
└────────────────────────────────────────────┘
0x2FFF
```

Throughout this article, we'll explore what each of these sections contains, why it exists, and how the compiler, linker, and startup code work together to prepare them before your program reaches `main()`.

---

## 1. `.text` Section — Program Instructions

* The **`.text` section** contains the executable machine instructions generated from your C source code. Every function that is compiled, including `main()`, is typically placed in this section.
* In embedded systems, the `.text` section is usually stored in **Flash (or ROM)** because program instructions must remain available even after power is removed.
* The `.text` section is generally **read-only** during program execution. Preventing accidental modification of program instructions helps improve system reliability and protects the integrity of the firmware.

Consider the following example:

```c
#include <stdio.h>

void greet(void)
{
    printf("Hello World\n");
}

int main(void)
{
    greet();
    return 0;
}
```

Both `main()` and `greet()` are compiled into machine instructions and placed inside the `.text` section.
The `.text` section contains the executable instructions for your program. During linking, the linker combines the `.text` sections from all object files into a single program code section.

---

## 2. `.rodata` Section — Read-Only Data

* The **`.rodata` (Read-Only Data)** section stores data that should **never be modified** while the program is running.
* Unlike the `.text` section, which contains executable instructions, `.rodata` contains **constant values** that are accessed by the program but never written to.
* In embedded systems, the `.rodata` section is typically stored in **Flash (or ROM)** because its contents never change during execution.

Common examples of data stored in `.rodata` include:

- String literals
- Variables declared with the `const` qualifier
- Constant lookup tables

Consider the following example:

```c
#include <stdint.h>

const uint32_t max_users = 100;

int main(void)
{
    const char *msg = "Access Denied!";

    return 0;
}
```

In this example:

* `max_users` is typically stored in the **`.rodata`** section.
* The string literal `"Access Denied!"` is also typically stored in **`.rodata`**.
* The pointer variable `msg` is **not** stored in `.rodata`. Since it is a local variable, it resides on the **stack**.

---

## 3. `.data` Section — Initialized Data

* The **`.data` section** stores **global** and **static** variables that have been explicitly initialized to a non-zero value.
* Unlike the `.text` and `.rodata` sections, variables in `.data` are **modifiable** during program execution. Since their values can change, they must reside in **RAM**.

Consider the following example:

```c
#include <stdint.h>

uint32_t system_state = 1;

static uint16_t boot_count = 5;

int main(void)
{
    static uint32_t retry_count = 3;

    return 0;
}
```

In this example:

- `system_state` is stored in the `.data` section.
- `boot_count` is also stored in the `.data` section.
- `retry_count` is a static local variable, but because it has static storage duration and is initialized to a non-zero value, it is also placed in the `.data` section.

### Why is `.data` stored in RAM?

Variables in the `.data` section are expected to change while the program is running.

For example:

```c
system_state = 2;
boot_count++;
```

Since Flash memory is not intended for frequent runtime modification, these variables are placed in RAM, where they can be read from and written to efficiently.

### Initial Values

Although the variables themselves reside in **RAM** during execution, their **initial values** are stored in **Flash** as part of the program image.

Before `main()` is called, the startup code copies these initial values from Flash into the `.data` section in RAM.

> **Note:** We'll explore how this copy operation works in the **Startup Code**.
Although variables in the .data section execute from RAM, their initial values are stored in Flash as part of the program image. 
> This separation between where data is stored and where it executes is described using the concepts of **Load Memory Address (LMA)** and **Virtual Memory Address (VMA)**. We'll explore these concepts in detail when we discuss linker scripts.

---

## 4. `.bss` Section — Uninitialized Data

* The **`.bss` (Block Started by Symbol)** section stores **global** and **static** variables that are either:
    * Declared without an explicit initializer.
    * Explicitly initialized to `0`.
* Unlike the `.data` section, these variables **do not need their initial values to be stored in Flash** because they all begin with the value zero.

Consider the following example:

```c
#include <stdint.h>

uint32_t system_status;

static uint32_t sensor_count;

int32_t error_counter = 0;

int main(void)
{
    static uint32_t retry_count;

    return 0;
}
```

In this example:

- `system_status` is placed in the `.bss` section.
- `sensor_count` is also placed in the `.bss` section.
- `error_counter` is initialized to zero, so it is typically placed in the `.bss` section.
- `retry_count` is a static local variable with no initializer and is also stored in `.bss`.

### Why have a separate `.bss` section?

Suppose you declare:

```c
uint8_t buffer[1024];
```

If this array were stored in the `.data` section, the executable would need to contain **1024 bytes of zeros**.

Instead, the linker simply records:

> "Reserve 1024 bytes for this variable."

When the microcontroller starts, the startup code clears the entire `.bss` section by filling it with zeros before `main()` is executed.

This approach significantly reduces the size of the executable while still guaranteeing that all variables in `.bss` begin with the value zero.

---

## 5. Heap — Dynamic Memory

* The **heap** is a region of RAM used for **dynamic memory allocation**. Unlike global or local variables, memory from the heap is allocated **at runtime** only when the program requests it.
* Memory on the heap is typically obtained using functions such as `malloc()`, `calloc()`, and `realloc()`, and must be released using `free()` when it is no longer needed.

Consider the following example:

```c
#include <stdlib.h>

int main(void)
{
    int *buffer = malloc(10 * sizeof(int));

    if (buffer == NULL)
    {
        return -1;
    }

    /* Use the allocated memory */

    free(buffer);

    return 0;
}
```

In this example:

- `buffer` (the pointer variable) is a local variable and resides on the **stack**.
- The memory allocated by `malloc()` resides on the **heap**.

### Heap Growth

* The heap typically begins immediately after the `.bss` section and grows **toward higher memory addresses** as memory is allocated.
* Unlike the stack, memory allocated on the heap remains valid until it is explicitly released using `free()` or until the program terminates.

### Dynamic Memory in Embedded Systems

Although dynamic memory allocation is common in desktop applications, many embedded systems avoid using the heap because it can introduce:

- Memory fragmentation
- Unpredictable allocation time
- Memory leaks
- Increased RAM usage

For these reasons, many real-time and safety-critical embedded applications rely primarily on **static** or **stack** allocation instead of dynamic allocation.

> **Note:** Whether dynamic memory allocation is available depends on the embedded system. Some projects disable `malloc()` entirely, while others use custom memory allocators designed for deterministic behavior.

---

## 6. Stack — Automatic Storage

The **stack** is a region of RAM used to manage **function calls** and **automatic (local) variables**. Unlike the heap, memory on the stack is allocated and released automatically by the compiler as functions are entered and exited.

Because stack allocation is automatic, the programmer does not need to manually allocate or free memory.

Consider the following example:

```c
#include <stdint.h>

void process_data(void)
{
    int32_t sensor_value = 100;
    char status = 'A';

    /* Use local variables */
}

int main(void)
{
    process_data();
    return 0;
}
```

In this example:

- `sensor_value` is stored on the stack.
- `status` is also stored on the stack.
- When `process_data()` returns, both variables are automatically destroyed.

Unlike global variables, local variables exist only for the duration of the function call.

### Stack Frames

Whenever a function is called, the processor creates a **stack frame**.

A stack frame typically contains:
- Local variables
- Function parameters (depending on the calling convention)
- Return address
- Saved CPU registers
- Temporary values used by the compiler

A simplified stack frame looks like this:

```text
Higher Memory Address

            Previous Stack Frames
┌──────────────────────────────────────────┐
│                                          │
└──────────────────────────────────────────┘

            Current Stack Frame
┌──────────────────────────────────────────┐
│ Local Variables                          │
├──────────────────────────────────────────┤
│ Temporary Data                           │
├──────────────────────────────────────────┤
│ Saved CPU Registers                      │
├──────────────────────────────────────────┤
│ Return Address                           │
└──────────────────────────────────────────┘
                     │
                     ▼  New function call
            New Stack Frame
┌──────────────────────────────────────────┐
│ Local Variables                          │
├──────────────────────────────────────────┤
│ Temporary Data                           │
├──────────────────────────────────────────┤
│ Saved CPU Registers                      │
├──────────────────────────────────────────┤
│ Return Address                           │
└──────────────────────────────────────────┘

Lower Memory Address
```

### Stack Growth

On most processor architectures, including ARM Cortex-M, the stack grows **toward lower memory addresses**.

```text
           Higher RAM Address (0x2FFF)

┌──────────────────────────────────────┐
│                Stack                 │
└──────────────────────────────────────┘
                  │
                  │ grows toward
                  ▼ lower addresses

            Available RAM

                  ▲ grows toward
                  │ higher addresses
┌──────────────────────────────────────┐
│                 Heap                 │
└──────────────────────────────────────┘
│                 .bss                 │
└──────────────────────────────────────┘
│                .data                 │
└──────────────────────────────────────┘
           Lower RAM Address (0x2000)
```
Every function call decreases the stack pointer, reserving space for a new stack frame. When the function returns, the stack pointer is restored, automatically releasing the memory occupied by that frame.

### Stack Overflow

The stack has a finite size.

If too many nested function calls occur, or if very large local variables are declared, the stack may grow into other memory regions, causing a **stack overflow**.

For example:

```c
void process(void)
{
    uint8_t buffer[10000];
}
```

On many embedded systems with only a few kilobytes of RAM, allocating such a large local array may exhaust the available stack space.

For this reason, embedded developers generally avoid placing large buffers on the stack.

### Key Takeaways

- The stack stores local variables and manages function calls.
- Memory is allocated and released automatically.
- Every function call creates a new stack frame.
- The stack typically grows toward lower memory addresses.
- Large local variables or deep recursion can cause a stack overflow.

_**Author's Note:** Technical content by the author. Edited and reworded with AI assistance._

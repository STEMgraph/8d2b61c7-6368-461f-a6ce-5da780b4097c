<!---
{
  "id": "8d2b61c7-6368-461f-a6ce-5da780b4097c",
  "teaches": "Debugging: Watchpoints on a Global Variable with GDB",
  "depends_on": [
    "ec9e6408-a2c2-4ef3-9546-0212928a9abb"
  ],
  "author": "Stephan Bökelmann",
  "first_used": "2026-10-05",
  "keywords": [
    "gdb",
    "watchpoint",
    "watch",
    "rwatch",
    "hardware watchpoint",
    "global variable",
    "pointer aliasing"
  ]
}
--->

# Debugging: Watchpoints on a Global Variable with GDB

> In this exercise a global variable has the wrong value at the end of the program, and no line in sight assigns it. You will use a GDB watchpoint to stop the program at the exact instruction that changes the variable, wherever that is, and read the call chain that led there. No `sudo` is required.

## Introduction

A breakpoint stops at a *place*. A **watchpoint** stops at an *event*: whenever a given variable or memory address is written (or, with `rwatch`, read). You do not need to know which function, which file, or which pointer does the writing. On x86 the processor supports a handful of hardware watchpoints, so the program runs at full speed until the write happens.

This is the question `DBG` and logging cannot answer at all: "who touched this?" With `printf` you would have to guess every place that *might* write the variable, including code in libraries you do not have the source for.

### 1.1) Further Readings and Other Sources

* [GDB Manual: Setting Watchpoints](https://sourceware.org/gdb/current/onlinedocs/gdb/Set-Watchpoints.html)
* [GDB Manual: Breakpoints, Watchpoints, and Catchpoints](https://sourceware.org/gdb/current/onlinedocs/gdb/Breakpoints.html)

## Tasks

### Task 1: The Program

```bash
cd ~/dbg-gdb
```

```c
// file: watch.c
#include <stdio.h>

int max_items = 10;

void clamp(int *x, int lo, int hi) {
    if (*x < lo) *x = lo;
    if (*x > hi) *x = hi;
}

void init_config(void) {
    int retries = 99;
    clamp(&retries, 0, 5);
    clamp(&max_items, 0, 5);
}

int main(void) {
    init_config();
    printf("max_items = %d\n", max_items);
    return 0;
}
```

```bash
gcc -Wall -Wextra -O0 -g watch.c -o watch && ./watch
```

Expected 10, printed 5. In this toy the culprit is three lines away. Pretend `init_config` lives in a 4000-line file you did not write.

### Task 2: Set a Watchpoint

```bash
gdb -q ./watch
```

```
(gdb) start
(gdb) watch max_items
(gdb) continue
(gdb) backtrace
```

`start` is needed first: a watchpoint on a global can be set before the program runs, but GDB then has to use a slower software watchpoint. After `start` it uses a hardware watchpoint. GDB reports `Old value = 10`, `New value = 5` and stops *after* the instruction that wrote it, inside `clamp`. `backtrace` shows that `init_config` called it with `&max_items`.

### Task 3: Watch an Expression and a Local

```
(gdb) delete
(gdb) break init_config
(gdb) run
(gdb) watch retries
(gdb) continue
(gdb) continue
```

A watchpoint on a local variable is deleted automatically when the function returns; GDB tells you so. Then try `watch max_items > 7`: GDB stops when the *expression* changes its truth value.

### Task 4: Read Watchpoints

```
(gdb) delete
(gdb) rwatch max_items
(gdb) run
(gdb) backtrace
```

`rwatch` stops when the variable is *read*. Which line reads it first? `awatch` stops on both.

### Task 5: Watch a Struct Member Through a Pointer

Add to the program:

```c
struct config { int limit; int verbose; };
struct config cfg = { 100, 0 };
void apply(struct config *c) { c->limit /= 2; }
/* in main, before init_config(): */
    apply(&cfg);
```

```
(gdb) start
(gdb) watch cfg.limit
(gdb) continue
```

The write goes through a pointer `c` inside `apply`; the watchpoint still fires, because it watches the *memory*, not the name.

### Task 6: Count the Hardware

```
(gdb) watch max_items
(gdb) watch cfg.limit
(gdb) watch cfg.verbose
(gdb) watch retries
(gdb) info watchpoints
```

Keep adding until GDB says it has to fall back to software watchpoints, then `continue` and feel the difference in speed. x86 processors have four debug registers.

### Task 7: Find It Without the Source

```bash
gcc -O0 -g -c watch.c -o watch.o && gcc -O0 -c -fPIC -shared -x c - -o libcfg.so <<'EOF'
extern int max_items;
void mystery(void) { max_items = 7; }
EOF
```

Add `void mystery(void); mystery();` at the end of `main`, link with `-L. -lcfg -Wl,-rpath,'$ORIGIN'`, run under GDB with `watch max_items` and `continue` twice. The second stop is inside a library you have no source for, and `backtrace` still names the function.

## Questions

- Why does GDB prefer a hardware watchpoint, and how many does your processor have?
- Why must the program be running (`start`) before `watch` on a global uses the hardware?
- What happens to a watchpoint on a local variable when the function returns, and why?
- `watch`, `rwatch`, `awatch`: give one debugging question for each.
- Task 5: the variable was written through a pointer in another function. How would you have found this with `DBG`?
- Task 7 stopped inside a library without source. What information did `backtrace` still have, and where did it come from?

## Advice

"Who changes this value?" is one of the most common questions in debugging and one of the hardest to answer by reading. A watchpoint answers it in seconds, across files, pointers and libraries. Reach for it whenever a variable is wrong and you cannot see an assignment.

Remember the limit of four hardware watchpoints and the fact that watchpoints on locals die with their frame. For everything else, `watch` plus `backtrace` is the whole technique.

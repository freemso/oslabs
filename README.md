# OS Lab 1 Report

*Author: Chengzu Ou (14302010037)*

## Description
**Write a description about the procedure of the booting up. What difficulties did you meet during this lab and how you conquered them.**

The first thing that a PC does when booting up is to execute instructions in BIOS. The BIOS is responsible for performing basic system initialization such as activating the video card and checking the amount of memory installed. After performing this initialization, the BIOS tries to load the operating system from some appropriate location. When the BIOS makes sure which disk the OS is in, it will load the first sector of the disk, a.k.a. *boot sector* into memory. Boot sector has a very important program called *boot loader* which is responsible for the loading of the entire operating system and some configuration work. After that, the operating system starts to run.

Thus, the procedure of booting up is basically BIOS -\> boot loader -\> operating system kernel.

At first I was trying to config the lab environment on my Ubuntu 16.04 Virtual Machine. But I keep getting errors when compiling `jos`. From the error message, I guess it must has something to do with the `gcc`. Since I did something changes to the `gcc` on this operating system to do the lab of another course, Computer Architecture, I figure that why bother trying to change the configuration back and forth when I can just simply use another machine to do the job. After all, it’s all *virtual*. So I did this lab on my Ubuntu 18.04 Virtual Machine.

In exercise 3, I had some trouble understanding the relationship between `BIOS`, `boot/boot.s`, and `boot/main.c` at first. I can see that in `boot.s`, it switches from real mode to protected mode and in `main.c`, it loads the operating system kernel to the memory. Then I realize that `BIOS` is the first program to run when booting up the computer. After `BIOS` finished its job to set some hardware stuff up and load the first sector of the disk which stores the boot loader program, it calls `boot.s` to run boot loader. Boot loader comprise of `boot.s` and `main.c`.

In exercise 8, I first don’t understand how `cprintf` could accept variable length of arguments. After searching online, I get that it is `va_start`, `va_arg`, and `va_end` that make all these things possible. But unfortunately, I could not see the exact operation of those functions(or macro to be precise). I took a bold guess and came up with the following answers.

In the last two exercises, it took me a lot of time figuring out the format of the output. I even went to see the grading script to get the desired output format.



## Answers
#### Q1: At what point does the processor start executing 32-bit code? What exactly causes the switch from 16- to 32-bit mode?
In file `boot.s`, cpu first works in real mode, which is 16-bit mode. After the instruction `ljmp    $PROT_MODE_CSEG, $protcseg` in line 55, the processor switches to 32-bit mode.

The switch from 16- to 32-bit mode takes a lot of efforts. The part in `boot.s` that tries to enable line A20, including `seta20.1` and `seta20.2` is the preparation work of the switch.

The cause of the switch is also related to the instruction `lgdt    gdtdesc` in line 48. This instruction load the value in `gdtdesc` into GDTR which is the register that stores some important information about GDT including the start address of GDT and the size of it.

After that, it sets the lowest bit of CR0, which is a control register, to 1. This indicates the start of protected mode.

#### Q2: What is the last instruction of the boot loader executed, and what is the first instruction of the kernel it just loaded?
The last instruction that boot loader executed is the last statement of `bootmain()` in `boot.c` file, which is `((void (*)(void)) (ELFHDR->e_entry))();`. According to `boot.asm` file, this statement calls the function in address `*0x10018`, which is the start point of kernel program.

The first instruction of the kernel is in line 44 of `entry.s` file, `movw	$0x1234,0x472`.

#### Q3: Where is the first instruction of the kernel?
According to `gdb`, we find that the first instruction of the kernel, which is `movw	$0x1234,0x472` in `entry.s` is at address `0x10000c`.

#### Q4: How does the boot loader decide how many sectors it must read in order to fetch the entire kernel from disk? Where does it find this information?
The information of how many segments the OS has and how many sectors that each segment has is store in *program header table*, which in `boot.c` is the data structure `struct Proghdr`. In this table, every entry is correlated to a segment in OS and it contains the size of the segment, the start address of the sector, and the offset.

Program header table is stored in the header of ELF file, which starts at address `ELFHDR`.

#### Q5: Explain the interface between `printf.c` and `console.c`. Specifically, what function does `console.c` export? How is this function used by `printf.c`?
`console.c` is used exports function on how to show a character on console and some operation on I/O ports. Function `cputchar` is a high level console I/O used by `readline` and `cprintf`. It calls `cons_put`, which output a character to the console. There are 3 sub programs in `cons_putc`: `serial_putc`, `lpt_putc`, and `cga_putc`. `serial_putc` is used to output a character to serial port. `lpt_putc` is used to output a character to parallel port. And `cga_putc` is to show character on console. 

`printf.c` defines some functions used for format printing such as `printf` and `sprintf`. Functions in this file calls functions exported by `printfmt.c` and eventually `console.c` to put characters on console, such as `cputchar` function.

#### Q6: Explain the following from `console.c`:
```c
if (crt_pos >= CRT_SIZE) {
	int i;
	memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));
	for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
		crt_buf[i] = 0x0700 | ' ';
	crt_pos -= CRT_COLS;
}
```

`crt_buf` is a character buffer, in which stores characters to be put on console. `crt_pos` is the position of the last character on console.

When `crt_pos` is larger or equal to `CRT_SIZE`(which is `80*25`), for that we’ve already know that the range of `crt_pos` is from `0` to `80*25-1`, it means the content shown on console has exceeded on page. Thus we need to scroll down a line.

We use `memcpy` to copy characters from line `1` to line `79` in `crt_buf` to line `0` to line `78`. And use a `for` loop to set the last line, line `79`, to empty characters. At last, we update `crt_pos`.

#### Q7: Trace the execution of the following code step-by-step:
```c
int x = 1, y = 3, z = 4;
cprintf("x %d, y %x, z %d\n", x, y, z);
```
**In the call to `cprintf()`, to what does `fmt` point? To what does `ap` point?**

In the call to `cprintf()`, `fmt` points to the format string to be shown. In this case, it’s `"x %d, y %x, z %d\n"`.

The type of `ap` is `va_list`. It is used to handle variable number of arguments. It points to the set of all input arguments.

**List (in order of execution) each call to `cons_putc`, `va_arg`, and `vcprintf`. For `cons_putc`, list its argument as well. For `va_arg`, list what `ap` points to before and after the call. For `vcprintf` list the values of its two arguments.**

The first function to be called is `cons_putc`. It is called by function `cputchar`, which is called by `putch`, which is called by `printfmt` after the calling of `getint`. The argument of `cons_putc` is just a single `c`, which in this case is `"x"` because it’s the first character to be printed on console.

The second function to be called is `vcprint`. It is called by `cprint`. The first argument, `fmt`, is the formatted string `"x %d, y %x, z %d\n"`, and the second argument, `ap` is the list of other arguments passed to `cprint`, in this case, `x, y, z`.

The last function to be called is `va_arg`. It is called by function `getint` in `vprintfmt` which is called by `vcprint`. The first time that function `va_arg` is called, it’s called to handle `"x %d"`. Thus, before calling, `ap` points to `x, y, z`, which is `1, 3, 4`. After calling, `x` is removed from the list, leaving only `y, z`, which is `3, 4`, in the list that `ap` points to.

#### Q8: Run the following code.
```c
unsigned int i = 0x00646c72;
cprintf("H%x Wo%s", 57616, &i);
```
**What is the output? Explain how this output is arrived at in the step-by-step manner of the previous exercise.**

The output is `"He110, World"`.

When calling `cprint` to print the string, it first read through the string and print every “normal” characters, by which I mean the character that is not start with `%`. Thus it will first print `H` just as usual.

Then, it comes to `%x`. This mean print the first argument after the string as hex. The first argument is `57616`. The corresponding hex is `e110`. Thus it prints `He110`.

The next “abnormal” character is `%s`, which asks the second argument to be printed as a string. Unfortunately, the second argument is `&i`, which is the address of the variable `i`, which is an unsigned integer. So we should just print `i` as a string. For that x86 is little-endian, so the `int` type variable `i` will be stored as `0x72, 0x6c, 0x64, 0x00` within memory, which is `'r', 'l', 'd', '\0'` according to ASCII table.

**The output depends on that fact that the x86 is little-endian. If the x86 were instead big-endian what would you set `i` to in order to yield the same output? Would you need to change `57616` to a different value?**

If x86 is big-endian, `i` should be `0x726c6400`. We do not need to modify `57616` because it has nothing to do with little-endian or big-endian.

#### Q9: In the following code, what is going to be printed after `'y='`? (note: the answer is not a specific value.) Why does this happen?
```c
cprintf("x=%d y=%d", 3);
```
The output is `x=3 y=-267380580`.

It is because we does not specify the argument of `y`, so the output value will be uncertain.

#### Q10: Let's say that GCC changed its calling convention so that it pushed arguments on the stack in declaration order, so that the last argument is pushed last. How would you have to change `cprintf` or its interface so that it would still be possible to pass it a variable number of arguments?
As we can see, `cprintf` use `va_start` to calculate the start address of `ap` and `va_arg` to get the current argument in `ap` and move the point to the next one.

If GCC changed its calling convention so that it pushed arguments on the stack in declaration order, then we should change the behavior of `va_start` and `va_arg` so that they would **subtract** the address rather than **add** the address to get the next argument.

# OS Lab2 Answers

*Author: Chengzu Ou (14302010037)*

### Q1: Assuming that the following JOS kernel code is correct, what type should variable `x` have, `uintptr_t` or`physaddr_t`?

```c
mystery_t x;
char* value = return_a_pointer();
*value = 10;
x = (mystery_t) value;
```

The type of variable `x` should be `uintptr_r` because it uses `*` operator on it.

### Q2: What entries (rows) in the page directory have been filled in at this point? What addresses do they map and where do they point? In other words, fill out this table as much as possible:

| Entry | Base Virtual Address | Points to (logically):                                       |
| ----- | -------------------- | ------------------------------------------------------------ |
| 1023  | 0xffc00000           | Page table for top 4MB of phys memory                        |
| 1022  | 0xff800000           | 4MB kernel space                                             |
| ...   | ...                  | ...                                                          |
| 960   | 0xf0000000           | All physical memory mapped at this address. (KERNBASE)       |
| 959   | 0xefc00000           | Kernel stack. (KSTACKTOP)                                    |
| 958   | 0xef800000           | The limit of user programming. (ULIM)                        |
| 957   | 0xef400000           | Intented for user to access VPT. (UVPT)                      |
| 956   | 0xef000000           | Mapping of struct Page in memory. (UPAGES)                   |
| 955   | 0xeec00000           | The top of user exception stack, and the limit of user to write. (UTOP, UENVS, UXSTACKTOP) |
| .     | 0xeebfe000           | The top of user stack area. (USTACKTOP)                      |
| ...   | ...                  | ...                                                          |
| 2     | 0x00800000           | User program space. (UTEXT)                                  |
| 1     | 0x00400000           | User temporary space to copy temporary data. (UTEMP)         |
| 0     | 0x00000000           | Page table for bottom 4MB of physical memory(empty memory of user space). |

### Q3: We have placed the kernel and user environment in the same address space. Why will user programs not be able to read or write the kernel's memory? What specific mechanisms protect the kernel memory?

We can not let user program to access (read or write) kernel space because that may disrupt kernel and cause system crashes.

We use paging translation to protect kernel space. By setting the permission of page entry in kernel space to kernel only, user will not be able to access this page.

### Q4: What is the maximum amount of physical memory that this operating system can support? Why?

The operating system uses one page directory entry, `UPAGES`, with size 4MB to store all `PageInfo` structures. Every structure takes 8B space. Thus there can be in total 512K `PageInfo` structures, which means 512K physical pages. Every physical page is 4KB. So the maximum amount of physical memory that this OS can support is 2GB.

### Q5: How much space overhead is there for managing memory, if we actually had the maximum amount of physical memory? How is this overhead broken down?

To manage the mapping between virtual memory and physical memory. First we need to store all `PageInfo` structures on memory, which costs 4MB. Then we need a 4KB `kern_pgdir`. And we also need to store current page table which is 2MB. Thus the total cost is 6MB + 4KB.

### Q6: Revisit the page table setup in `kern/entry.S` and`kern/entrypgdir.c`. Immediately after we turn on paging, EIP is still a low number (a little over 1MB). At what point do we transition to running at an EIP above KERNBASE? What makes it possible for us to continue executing at a low EIP between when we enable paging and when we begin running at an EIP above KERNBASE? Why is this transition necessary?

In `entry.s`, there is an instruction `jmp *%eax`. This will set `eip` to the value stored in `eax` which is larger than KERNBASE. This completes the transition from low number to above KERNBASE.

The reason why we could continue executing at a low `eip` after enabling paging and before runing at an `eip` above KERNBASE is that we also map the lowest address(from 0 to 4MB) in virtual memory to the lowest address in physical memory in `entry_pgdir` table. Thus when access virtual address within [0, 4MB], it will mapping to the same address in physical memory.

# OS Lab3 Report

*Author: Chengzu Ou (14302010037)*

## Description

There are two global variable in `kern/env.c`.

```c
struct Env *envs = NULL; // All environments
struct Env *curenv = NULL; // The current env
static struct Env *env_free_list; // Free environment list
```

After JOS bootup, `envs` points to a list of `Env` which stores all environments. It uses `env_free_list` to manage free environments and `curenv` to represent current running environment.

Note that when implementing `env_init()`, we should add items in `envs` to `env_free_list` in reverse order so that the first we call `env_alloc()`, we will get `envs[0]`.

In `env_setup_vm`, we can use `kern_pgdir` as a template to set `env_pgdir`. But to better understand the memory layout, I choose to manually set it up.

In `region_alloc`, we should round down the start address and round up the end address of the region to allocate.

In `load_icode`, we use `lcr3()` to load the physical address of the starting point of environment directory table. And set `e_entry` to `e->env_tf.tf_eip` so that the environment start executing here.

When we are at kernel mode and we need a system call, we just have to directly call the function. But when we are at user mode, we need a system interrupt to do a system call.

To handle system call interrupt, we need to add a switch case in `trap_dispatch()`. Note that we need to call `syscall()` function in `kern/syscall.c`.

`syscall()` in `kern/syscall.c` serve as a dispatch to call other functions in the same file based on the `syscallno` argument passed by.

## Answers

#### Q1: What is the purpose of having an individual handler function for each exception/interrupt? (i.e., if all exceptions/interrupts were delivered to the same handler, what feature that exists in the current implementation could not be provided?)

Different exceptions/interrupts need different handler function because we handle them in different ways.

An *interrupt* is a protected control transfer that is caused by an asynchronous event usually external to the processor, such as notification of external device I/O activity. An *exception*, in contrast, is a protected control transfer caused synchronously by the currently running code, for example due to a divide by zero or an invalid memory access.

If all exceptions/interrupts were delivered to the same handler, we will not be able to differenciate between those exceptions that are caused by errors and those that are cause by interruptions and need to go back to the program after.

#### Q2: Did you have to do anything to make the `user/softint` program behave correctly? The grade script expects it to produce a general protection fault (trap 13), but `softint`'s code says `int $14`. *Why* should this produce interrupt vector 13? What happens if the kernel actually allows `softint`'s `int $14` instruction to invoke the kernel's page fault handler (which is interrupt vector 14)?

This is because current program is running under user mode and at  privilege level 3. Instruction INT is a system instruction and at privilege level 0. Program with privilege level 3 can't directly call program with privilege level 0. Otherwise, it will cause a general protection exception, a.k.a. trap 13.

#### Q3: The break point test case will either generate a break point exception or a general protection fault depending on how you initialized the break point entry in the IDT (i.e., your call to `SETGATE` from `trap_init`). Why? How do you need to set it up in order to get the breakpoint exception to work as specified above and what incorrect setup would cause it to trigger a general protection fault?

If we set DPL of breakpoint exception entry in IDT to 3, it will generate a break point exception. It set it to 0, it will generate a general protection exception. 

This is because when executing handler, we need current privileged level less than DPL. Otherwise, there will be a general protection exception. So we have to set it to 3 so that DPL is equal to CPL.

#### Q4: What do you think is the point of these mechanisms, particularly in light of what the `user/softint` test program does?

These machanisms is used to make sure that user programs do not directly call kernel programs to prevent unsafe calling.

# OS Lab4 Report

*Author: Chengzu Ou (14302010037)*

## Description
In the first exercise, we are asked to implement `mmio_map_region` in `kern/pmap.c` to allocate space from MMIO region and map device memory to it. Note that we should check if the memory exceed `MMIOLIM`.

In the second exercise, `boot_aps()` first copies code in `mpentry.S` to memory at `MPENTRY_PADDR`. Then, it start a process for every `cpu` which is `APS`. At last, it jumps to execute `mpentry.S`. The function of `mpentry.S` is similar to boot loader. It will jump to `mp_main` and do some initialization work.

In the third exercise, we are asked to allocate memory for each CPU. And in exercise four, we should init every CPU.

To prevent race condition when multiple CPUs run kernel code simultaneously, we should implement a kernel lock. In this lab, environments in user mode can run concurrently on any available CPUs, but no more than one environment can run in kernel mode; any other environments that try to enter kernel mode are forced to wait.

## Answers
#### Q1: Compare `kern/mpentry.S` side by side with `boot/boot.S`. Bearing in mind that `kern/mpentry.S` is compiled and linked to run above `KERNBASE` just like everything else in the kernel, what is the purpose of macro `MPBOOTPHYS`? Why is it necessary in `kern/mpentry.S` but not in `boot/boot.S`? In other words, what could go wrong if it were omitted in kern/`mpentry.S`? 

The function of `MPBOOTPHYS` is to map high address to low address. When executing this code, we are still in real mode but the address in the code is already in protected mode. So we have to translate it from high address to low address.

#### Q2: It seems that using the big kernel lock guarantees that only one CPU can run the kernel code at a time. Why do we still need separate kernel stacks for each CPU? Describe a scenario in which using a shared kernel stack will go wrong, even with the protection of the big kernel lock.

When the interrupt happens, it will push to the stack and we have not got the lock yet. When multiple CPUs have interrupt at the same time, shared kernel stack would cause error.

#### Q3: In your implementation of `env_run()` you should have called `lcr3()`. Before and after the call to `lcr3()`, your code makes references (at least it should) to the variable `e`, the argument to `env_run`. Upon loading the `%cr3` register, the addressing context used by the MMU is instantly changed. But a virtual address (namely `e`) has meaning relative to a given address context--the address context specifies the physical address to which the virtual address maps. Why can the pointer `e` be dereferenced both before and after the addressing switch?

This is because we did static mapping in `envs` and `mem_init()` uses `kern_pgdir` as a template so that `pgdir` in every `env` is copied from `kern_pgdir` and also has mapping.


#### Q4: Whenever the kernel switches from one environment to another, it must ensure the old environment's registers are saved so they can be restored properly later. Why? Where does this happen?

It is absolutely necessary to save old registers when doing context switching otherwise when switching back, the CPU would have no idea what state it is in and where is the next instruction.

The saving of context is done when interrupts happens. We copied a `Trapframe` in `trap()`.
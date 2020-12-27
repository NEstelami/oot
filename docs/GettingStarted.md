# Setup

Please refer to the readme for detailed steps on setting up the repo.

# Workflow
Undecompiled code is located in the asm/non_matchings folder. There are two folders, `code` and `boot`, which represent the two main executables (more on that later). Each subfolder represents one of the original C files, and contains a `.S` assembly file for each function within that C file (Note: Not all C files have been split into these folders yet).

Before working on a C file, check the Projects tab to make sure no one else is working on it. Be sure to reserve the file so that no one else takes it.

In the src/ folder, you'll find a set of .c files. Assuming no decompilation work has been done on a given file, you'll find a bunch of statements like so:

```
#pragma GLOBAL_ASM("asm/non_matchings/code/z_lights/func_80079D30.s")

#pragma GLOBAL_ASM("asm/non_matchings/code/z_lights/func_80079D8C.s")

#pragma GLOBAL_ASM("asm/non_matchings/code/z_lights/func_80079E58.s")
```

`#pragma GLOBAL_ASM` is a command which tells the assembly processor to include the assembly function from the specified file path. As you work on a file, you will replace these commands with proper C functions, until none of these remain.

# Decompiling your first function

Let's take the following function from z_actor and convert it to C.

```
glabel func_8002CCF0
/* AA3E90 8002CCF0 8C8E1D3C */  lw    $t6, 0x1d3c($a0)
/* AA3E94 8002CCF4 240F0001 */  li    $t7, 1
/* AA3E98 8002CCF8 00AFC004 */  sllv  $t8, $t7, $a1
/* AA3E9C 8002CCFC 01D8C825 */  or    $t9, $t6, $t8
/* AA3EA0 8002CD00 03E00008 */  jr    $ra
/* AA3EA4 8002CD04 AC991D3C */   sw    $t9, 0x1d3c($a0)
```

Let's take a look at the firt line.

`/* AA3E90 8002CCF0 8C8E1D3C */  lw    $t6, 0x1d3c($a0)`

There are two things that we can take away from this line.

First, `$a0` is in use. `$a0` contains the first argument to the function, and thus we know that this function has at least one argument.

Second, we know that `$a0` is a pointer because it is loading something relative to the address it points to and storing it in `$t6`. Since the `lw` or "load word" instruction is being used, we know its loading a 32-bit integer.

```
void func_8002CCF0(u32 a0)
{
    u32 t6 = *(u32*)(a0 + 0x13DC);

}
```

Let's take a look at the next few lines:

```
/* AA3E94 8002CCF4 240F0001 */  li    $t7, 1
/* AA3E98 8002CCF8 00AFC004 */  sllv  $t8, $t7, $a1
/* AA3E9C 8002CCFC 01D8C825 */  or    $t9, $t6, $t8
```

An integer value `1` is being loaded into register `$t7`. This value is then shifted to the left using `$a1` to determine how many bits it should be shifted, with the resullt being saved in `$t8`. Finally, this result is Binary OR'd with the contents of `$t6`. This roughly translates to:

```
void func_8002CCF0(u32 a0, u32 a1)
{
    u32 t6 = *(u32*)(a0 + 0x13DC);
    u32 t7 = 1;
    u32 t8 = t7 << a1;
    u32 t9 = t6 | t8;

}
```

Let's take a look at the final two lines:

```
/* AA3EA0 8002CD00 03E00008 */  jr    $ra
/* AA3EA4 8002CD04 AC991D3C */   sw    $t9, 0x1d3c($a0)
```

The result of these calculations are saved back into `$a0 + 0x1D3C`, then the function returns. Although the return instrument, `jr $ra` comes first, the following instruction gets executed first due to the use of a delay slot. 

This translates to:

```
void func_8002CCF0(u32 a0, u32 a1)
{
    u32 t6 = *(u32*)(a0 + 0x13DC);
    u32 t7 = 1;
    u32 t8 = t7 << a1;
    u32 t9 = t6 | t8;

    *(u32*)(a0 + 0x13DC) = t9;
}
```

This can be simplified down to:

```
void func_8002CCF0(u32 a0, u32 a1)
{
    *(u32*)(a0 + 0x13DC) |= (1 << a1);
}
```

Be sure to comment out the `#pragma GLOBAL_ASM` line with your function. Otherwise, you'll get an error about duplicate functions.

# Compiling

Open up a terminal and type `make`

If after compilation you see a message which says, `zelda_ocarina_mq_dbg.z64: OK `, you've successfully matched that function. If you've instead received a messasge which states, `zelda_ocarina_mq_dbg.z64: FAILED`, then your function does not match the original. You can find your compiled function in build/src/code/[name of your c file]. From there you can compare the two functions side by side to see what is different.
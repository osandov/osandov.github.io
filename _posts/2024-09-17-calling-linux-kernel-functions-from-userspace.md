---
layout: post
title: Calling Linux Kernel Functions From Userspace (!)
tags: kernel compilers debugging
---

I just landed a really exciting feature for [drgn](https://github.com/osandov/drgn): the ability to call arbitrary functions and write to memory in the Linux kernel. I think the technical details of the implementation are very interesting, and it's probably the funniest thing I've ever done, so I wanted to write about how it works.

## Background

[drgn](https://github.com/osandov/drgn) is a programmable debugger primarily targeted at the Linux kernel. It lets you read kernel debugging symbols and data structures from Python:

```python
>>> from drgn.helpers.linux.list import list_for_each_entry
>>> for mod in list_for_each_entry("struct module",
...                                prog["modules"].address_of_(),
...                                "list"):
...    if mod.refcnt.counter > 10:
...        print(mod.name)
...
(char [56])"snd"
(char [56])"evdev"
(char [56])"i915"
```

Until now, drgn has been purely read-only: it can inspect anything in kernel memory, but it can't change it or otherwise interfere. It uses safe kernel interfaces (namely, [`/proc/kcore`](https://man7.org/linux/man-pages/man5/proc_kcore.5.html)) that can't crash the kernel (barring kernel bugs).

So, when one of my teammates asked me whether drgn could call arbitrary functions in the kernel, my initial impulse was to say "of course not". But, after thinking about it for a few more moments, I had an idea.

## Generating Kernel Module Source Code

A crucial design decision in drgn is that it emphasizes programmatic interfaces for everything: not only can you print a variable or a type, but you can also use them through [Python APIs](https://drgn.readthedocs.io/en/latest/api_reference.html). These APIs enable many advanced use cases. In this case, they enabled my crazy idea: I could use drgn's APIs to generate the source code for a loadable kernel module that would make the desired function call.

So, the first version of this feature translated this:

```python
call_function("_printk", "Hello, world %d\n", 1234)
```

into this source code:

```c
#include <linux/module.h>
#include <linux/uaccess.h>

static int __init kmodify_init(void)
{
        struct {
                unsigned char arg0[17];
                typeof(int) ret;
        } out = {
                .arg0 = { 0x48, 0x65, 0x6c, 0x6c, 0x6f, 0x2c,
                          0x20, 0x77, 0x6f, 0x72, 0x6c, 0x64,
                          0x20, 0x25, 0x64, 0x0a, 0x00 },
        };

        out.ret = ((int (*)(const char *fmt, ...))0xffffffffb34bafaaUL)(
                (const char *)&out.arg0,
                (int)1234
        );

        if (copy_to_user((void __user *)0x7f7ee60694a0UL, &out, sizeof(out)))
                return -EFAULT;

        return -EINPROGRESS;
}

module_init(kmodify_init);
MODULE_LICENSE("GPL");
```

Then, it invoked the kernel build system to build the module before loading it with [`finit_module(2)`](https://man7.org/linux/man-pages/man2/finit_module.2.html).

There are a few important details here: the function is called via its address, casted to the proper type. This makes it possible to call static functions. The passed arguments are embedded directly into the source code. Additionally, the return value is copied back to drgn so that drgn can then return it to the user.

This can easily be extended to support writing to kernel memory by generating a call to `memcpy()`.

This approach was a great proof of concept, but it has a couple of major limitations:

* It requires that the kernel module build system (``kernel-devel``) is installed. This is not the case in many environments. A suitable toolchain must also be installed. If the compiler used is not exactly the same as the one used to build the kernel, you get [annoying warnings](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Makefile?h=v6.11#n1799). If the kernel was built with Clang/LLVM, then the module will fail to build with GCC/binutils.
* Compiling a kernel module is slow: about 3 seconds in my tests. It feels really clunky for a single function call to take that long.

Unhappy with these limitations, I was ready to toss my script into [`drgn/contrib`](https://github.com/osandov/drgn/tree/main/contrib) as a gimmick and move on. But, I had a nagging feeling that there was a better way.

## What is a Kernel Module, Really?

I was curious what it would take to craft a kernel module manually without the kernel build system. To do that, we need to understand the kernel module file format. A kernel module is a relocatable object file, similar to a `.o` file:

```console
$ file kmodify.ko
kmodify.ko: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), BuildID[sha1]=21a6fa6683ab50d6a9231eb336c18766a99ba579, with debug_info, not stripped
```

Like any relocatable object file, it consists of sections containing code, data, and metadata. The file may reference symbols (functions, global variables) whose addresses are not known until the file is loaded. So, the file metadata includes a list of "relocations": records indicating where the loader should store a resolved symbol address. In addition to the standard `.text`, `.data`, etc. sections, kernel modules have a couple of special sections. The `.modinfo` section comprises a list of key-value pairs with information about the module:

```console
$ readelf -p .modinfo kmodify.ko

String dump of section '.modinfo':
  [     0]  license=GPL
  [     c]  depends=
  [    15]  retpoline=Y
  [    21]  name=kmodify
  [    2e]  vermagic=6.11.0-vmtest30.1default SMP preempt mod_unload
```

The `.gnu.linkonce.this_module` section contains the `struct module` that will be used to track the module in the kernel. In the object file, it is zeroed out other than the `name` field.

Using some code from drgn's unit tests for generating custom ELF files, I figured out the minimum requirements for a valid kernel module file:

* It must have a symbol table and the corresponding string table (both of which may be empty).
* It must have a `.gnu.linkonce.this_module` section with `sh_size == sizeof(struct module)` and a non-empty `name`.
* It must have a `.modinfo` section containing a `vermagic` field that matches the kernel's `vermagic`. (It's a good idea to include some of the other fields.)

In order to execute code when the module is loaded, there are a few more requirements:

* It must have an `.init.text` section containing the executable code.
* It must have a function symbol named `init_module` referring to the code in the `.init.text` section.
* It must have a relocation that writes the address of the `init_module` symbol to the `init` function pointer in `.gnu.linkonce.this_module`.

drgn is able to get all of the necessary information using the kernel debugging symbols, so I could now craft a kernel module to run whatever machine code I wanted without any additional dependencies.

## Writing a Tiny Compiler

Now I needed a way to translate my high-level `call_function()` Python function into machine code. In other words, I needed a compiler. So, I wrote one.

### Front End

Like other compilers, this compiler has a "front end" that verifies the syntax and semantics of the input and transforms the input into an intermediate representation (IR). The input to this compiler is the `call_function()` call. First, it converts all of the arguments to [`drgn.Object`](https://drgn.readthedocs.io/en/latest/api_reference.html#drgn.Object)s. Then, it type checks the arguments just like a C compiler would. (In fact, I had to take a detour to implement a type checking helper function, [`drgn.implicit_convert()`](https://drgn.readthedocs.io/en/latest/api_reference.html#drgn.implicit_convert).) Finally, it generates the IR, which for this compiler is a tree of Python objects. For the example from earlier:

```python
call_function("_printk", "Hello, world %d\n", 1234)
```

the IR looks like this:

```python
_Function(
    body=[
        _Call(
            func=_Symbol(name="func"),
            args=[
                _Symbol(name=".data", section=True),
                _Integer(size=4, value=1234),
            ],
        ),
        _StoreReturnValue(
            size=4,
            dst=_Symbol(name=".data", offset=20, section=True),
        ),
        _Call(
            func=_Symbol(name="copy_to_user_nofault"),
            args=[
                _Integer(size=8, value=0x7f7ee60694a0),
                _Symbol(name=".data", section=True),
                _Integer(size=8, value=24),
            ],
        ),
        _ReturnIfLastReturnValueNonZero(
            value=_Integer(size=4, value=-errno.EFAULT),
        ),
        _Return(
            value=_Integer(size=4, value=-errno.EINPROGRESS),
        ),
    ]
)
```

Notice the resemblance to the generated C source code from earlier. Also notice that references to functions and data are represented by symbols.

### Back End

The compiler "back end" is responsible for generating the executable code for an IR. This can be a huge task. Luckily, our IR is very simple. In fact, the above example demonstrated the extent of its capabilities.

Our compiler contains code generation rules for each of these IR nodes. Function calls are the most complex since we need to implement the architecture's calling convention (i.e., which registers and stack locations to use for arguments and return values). Referencing the [x86-64 psABI specification](https://gitlab.com/x86-psABIs/x86-64-ABI) was a necessity. Symbol references also require us to generate relocations.

Since the whole point of this exercise was to avoid dependencies, we generate machine code directly instead of going through an assembler. Again, this is feasible because the needed operations are limited. I made heavy use of the [OSDev wiki](https://wiki.osdev.org/X86-64_Instruction_Encoding) and [`objdump(1)`](https://man7.org/linux/man-pages/man1/objdump.1.html) to understand the details of x86-64 instruction encoding.

The final compilation step is wrapping up the generated code, data, relocations, symbols, and kernel module metadata into a file which can then be loaded.

## Applications

There are two main ways I envision this feature being used. The first use case is debugging. Calling internal functions can be very helpful for understanding the state of the system. This is especially useful during development, where much less caution is needed.

The second use case is mitigating bugs in production. If a critical system gets in a bad state, it may be possible to fix that state by making a function call or overwriting some memory. For example, lost wake-ups are a common class of bug where, due to a race condition, a thread that is waiting on a condition misses its signal to wake up and waits forever.[^1] A lost wake-up can be cleared by manually waking up the thread with [`wake_up_process()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/core.c?h=v6.11#n4286). (This use case is complementary to [live patching](https://docs.kernel.org/livepatch/livepatch.html), which can prevent a bug from being hit in the future but usually doesn't resolve the bug if it was already hit.)

A brief note on security: this feature of course requires root (specifically, [``CAP_SYS_MODULE``](https://man7.org/linux/man-pages/man7/capabilities.7.html)). It doesn't allow a user to do anything they couldn't do before, although it certainly makes it easier. The proper way to disallow this is to require module signatures with [`CONFIG_MODULE_SIG_FORCE`](https://docs.kernel.org/admin-guide/module-signing.html) or [`kernel_lockdown(7)`](https://man7.org/linux/man-pages/man7/kernel_lockdown.7.html).

Regardless of security concerns, calling arbitrary functions and overwriting memory are very dangerous, so do it with care.

## Conclusion

I'm really excited about this feature, and I had a blast implementing it. There's still more work to do: supporting architectures other than x86-64; supporting the full calling convention (specifically, structure arguments and return values); making use of kprobes and ftrace to implement something akin to breakpoints; and integrating with module signing so that only very trusted users can use it.

This feature is available in the [`drgn.helpers.experimental.kmodify`](https://drgn.readthedocs.io/en/latest/helpers.html#kmodify) package in drgn's main branch, and it will ship in drgn 0.0.28. Try it out!

---

[^1]: I often [encountered](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6c0ca7ae292adea09b8bdd33a524bb9326c3e989) (and [created](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=fcf38cdf332a81b20a59e3ebaea81f6b316bbe0c)) this class of bug when I worked on Linux's multiqueue block layer.

# Barebones Rust OS
## Overview
This is (quick) guide to build a barebones kernel in Rust, in parallel to the barebones kernel we wrote in C.  Here, we write a kernel for the *x86 architecture*.

This is mostly based on [Phil Opp's tutorial](https://os.phil-opp.com/freestanding-rust-binary/).
## First Setup
We set up a new project with Cargo:
``` Shell
$ cargo new rust_os --bin
```
Remember `--bin` indicates we are setting up a package to produce a binary executable crate.

Cargo automatically creates a main file `src/main.rs`.
``` Rust
fn main() {
	println!("Hello, world!");
}
```
## No Standard Library
Our kernel must be free-standing, meaning it can't make use of the *Standard Library*: no threads, standard output, heap.  However, the core features of Rust are provided in a pre-compiled library aplty called `Core`, which is part of the compiler.  We'll need to recompile it for our target platform later, but we'll begin by cutting ourselves off from the Standard Library.

First off, we need to get rid of `println!` since it is a macro from the Standard Library.

Next we add a directive to tell the Rust compiler not to use the Standard Library when compiling this file:
``` Rust
// in main.rs
#![no_std]

fn main() { }
```

We'll delete the `main()` function later too.

## Runtime and Compile Panics
The Rust Compiler by default internally uses part of the Standard Library to deal with runtime panics and compile-time panics.

First, we re-implement the **Panic Handler**: this is function the compiler invokes when a runtime panic occurs.
``` Rust
// in main.rs
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
	loop{}
}

fn main() { }
```
Some notes:
- The parameter `PanicInfo` object holds the filename and line where the panic occurred (and panic message).
- This is a *diverging function* (never returns) as indicated by the return type `!`.
- For now, the function just loops forever; we'll fill it in later.

Now for compile-time panics.  By default, when the compiler panics it tries to call some functions from the Standard Library.  Instead, we tell compiler to simply abort by adding to the *manifest* (`cargo.toml`):
``` toml
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

> [! Note]- Compile-Time Panics
> The default behavior of compiler panics is `panic = "unwind"`.  By `unwind`, the compiler will try to free (or "unwind") the stack using functions in the Standard Library labeled as `lang_items`.  To learn more, read the [Unstable Rust Book](https://doc.rust-lang.org/unstable-book/language-features/lang-items.html).
## Entry Point
Compiled languages, when first executed by an operating system, start a **runtime system** that sets up the garbage collector or threads before invoking the entry point function `main()`.

In Rust, this runtime system is provided by the Standard Library via `crt0`, a C runtime library.

We are not going to use the runtime system because we are going to invoke our binary from a bootloader.

First, we get rid of `main()` because without `crt0` it is not recognized by the compiler as our entry point.  We also add the directive `#![no_main]` to tell the compiler we are not using the default entry point.  Next, we write our own entry point:
``` Rust
// in main.rs

// .. panic handler ..

#![no_main]
#[no_mangle]
pub extern "C" fn _start() -> ! {
	loop{}
}
```

A function named `_start()` in the C *calling convention* is recognized as the entry point of a program.  Therefore:
- The directive `#![no_mangle]` tells the Rust compiler to disable *name_mangling*.  In the output binary, the function `_start()` will be named `_start()` instead of something else.
- The prefix `pub extern "C"` says to use the C calling convention.

Again, this is a diverging function.  For now, we'll have this function loop endlessly.
## Building our Kernel
We are going to build our kernel for x86 systems.  

### Bootloader
Writing a bootloader from scratch is a pain.  Fortunately, there is a Rust crate that can prepend a bootloader to the beginning of our kernel binary.  We can install it with:

``` Shell
$ cargo install bootimage
```

And add the dependency in our `Cargo.toml` manifest:
``` toml
[dependencies]
bootimage = "0.9.8"
```

Note that his bootloader is *not Multiboot compliant*.

### Toolchain
The Rust installation manager *Rustup* provides cross-compilation capabilities to the Rust compiler, so setting up the toolchain should be easier than bootstrapping *gcc* cross-compiler.

#### Compiler
We're going to compile our package to a custom target platform; this requires that our compiler have some experimental features provided in the *nightly channel* of Rust.  We can install the nightly version of the Rust toolchain with:
``` Shell
$ rustup toolchain install nightly
```

We also need to install the nightly version of the Rust source code (because we need to recompile the Core library in the next section):
``` Shell
$ rustup component add rust-src --toolchains nightly-x86_64-unknown-linux-gnu
```

We want to make this the default in our project.  From the project root directiory, we can override the global default with:
``` Shell
$ rustup override set nightly
```

It may also be a good idea to indicate this in a `rust-toolchain.toml` file:
``` toml 
\\ rust-toolchain.toml

[toolchain]
channel = "nightly"
```

#### Cross-Compilation Target
In general, we can specify a compilation target by passing a *target-triple* to the Rust compiler or Cargo with the `--target` parameter.  

We can also specify a custom target by providing a `.JSON` file with custom target parameters.

Our's will be `x86-64-rust_os.JSON`:
``` json
{
    "llvm-target": "x86_64-unknown-none",
    "data-layout": "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-i128:128-f80:128-n8:16:32:64-S128",
    "arch": "x86_64",
    "target-endian": "little",
    "target-pointer-width": "64",
    "target-c-int-width": "32",
    "os": "none",
    "executables": true,
    "linker-flavor": "ld.lld",
    "linker": "rust-lld",
    "panic-strategy": "abort",
    "disable-redzone": true,
    "features": "-mmx,-sse,+soft-float"
}
```

Some notes:
- Our `"os"` is `"none"`, indicating our binary will run on "bare-metal".
- The options for `"linker-flavor"` and `"linker"` indicates to use LLVM's cross-platform linker, instead of the standard Rust linker.
- `"panic-strategy" : "abort"` does the same as `panic = "abort"` in the manifest `Cargo.toml`, but also applies to any recompiled `Core` features.  Technically the option in the manifest is redundant.
- `"disable-redzone"` removes a buffer at the bottom of stack.  This is where we need to put our interrupts table will go.  More details [here](https://os.phil-opp.com/red-zone/).
- `"features"` are arguments to LLVM.  Basically we disable *SIMD* and enable *soft-float*, which deal with floating-point numbers in x86-64. 

#### Recompile Core Library
The core features of Rust are part of the compiler as a pre-compiled library called `Core`.  We need to recompile them to use.  In the `.cargo/config.toml` configuration file (you'll have to make this yourself):

``` toml
[unstable]
build-std = ["core", "compiler_builtins"]
```

One last tweak - by default the *memory management* part of the core library is disabled.  Normally memory management is provided through the operating system via a C library (on Unix-likes).  However, we are running on bare metal, so we need to provide our own memory management tools.  Thankfully we can use memory management as provided by the core library, since writing our own would be dangerous.

In the same `config.toml` file:
``` toml
[unstable]
build-std-features = ["compiler-builtins-mem"]
build-std = ["core", "compiler_builtins"]
```
### Initial Build
To now build:

``` Shell
$ cargo build --target x86-64-rust_os.json
```

If it all compiles, great!  Notice that the core library is being recompiled.

Instead of passing our `.json` file every time we build, we can add this argument to the `config.toml` file:
``` toml
[build]
target = "x86-64-rust_os.json"

[unstable]
build-std-features = ["compiler-builtins-mem"]
build-std = ["core", "compiler_builtins"]
```

Now we can simply run:
``` Shell
$ cargo build
```

## Barebones Kernel
Like the Barebones OS in C, we can test our build by making our kernel simply print something to the screen.

Since we are in x86, we can make use of the **VGA text buffer**, located at physical memory `0xb8000`, which display the ASCII characters here onto the screen.  Each cell of the buffer consists of a character byte and a color byte.

We modify our `main.rs` file:

``` Rust
// in main.rs

static HELLO: &[u8] = b"Hello World!";

#[no_mangle]
pub extern "C" fn _start() -> ! {
    let vga_buffer = 0xb8000 as *mut u8;

    for (i, &byte) in HELLO.iter().enumerate() {
        unsafe {
            *vga_buffer.offset(i as isize * 2) = byte;
            *vga_buffer.offset(i as isize * 2 + 1) = 0xb;
        }
    }

    loop {}
}
```

Some notes:
- Our message is `Hello World`, which is here as a *byte string* (because it is prepended by `b`).
- Note we are placing the bytes from our message in offsets of 2 in the buffer (`.offset(i as isize * 2)`) since the first byte of a buffer cell is the character byte.
- We're setting the text color to white: `0xb`
- The loop body is an *unsafe* block, because we are writing directly to memory via *raw pointers*.

## Bootloader

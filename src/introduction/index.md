# The Rust FFI Guide


Welcome to the Rust FFI Guide, i.e. **using unsafe for fun and profit**.

> **Note:** This guide assumes you're already familiar with the [Rust][rust]
> language and have a relatively recent version of the compiler installed. If 
> you're a little rusty, you might want to skim through [The Book][book] to 
refresh your memory.
>
> It is recommended to know some basic C/C++ or Python, as it will be largely used 
> in the examples.


The main goal of this guide is to show how to interoperate between `Rust` and
other languages without reintroducing problems that Rust is trying to avoid: segfaults and undefined behaviour.

Some things this book will cover:

* [Compiling and linking from the command line](./introduction/index.html#Hello-World)
* [Using arrays](./arrays/index.html)
* [Sharing basic structs between languages](./structs/index.html)
* Proper error handling
* Calling Rust from other (i.e. not C) languages
* [How to use strings without leaking memory or segfaults](./strings/index.html) 
  (it's harder than you'd think)
* Asynchronous operations, threading and callbacks
* Bindgen
* [Emulating methods and OO](./pythonic/index.html), and
* [Other miscellaneous bits and pieces or best practices I've picked up along
  the way](./best_practices.html)


## Some Useful Links:

* [This Guide](https://michael-f-bryan.github.io/rust-ffi-guide/)
* [The GitHub Repo](https://github.com/Michael-F-Bryan/rust-ffi-guide)
* [FFI Page from *The Book*](https://doc.rust-lang.org/book/ffi.html)
* [The Rust FFI Omnibus](http://jakegoulding.com/rust-ffi-omnibus/)
* [Complex Types With Rust's FFI](https://medium.com/jim-fleming/complex-types-with-rust-s-ffi-315d14619479)
* [FFI: Foreign Function Interfaces for Fun & Industry](https://spin.atomicobject.com/2013/02/15/ffi-foreign-function-interfaces/)


## Hello World

What would any programming guide be without the obligatory "hello world" example?

> **Note:** For most of these examples I'll be using `C` to interoperate with 
> my `Rust` code. It's pretty much the *lingua franca* of the programming world, so we won't go much into every detail here.
> If you don't know `C`, that's not a problem, the examples will be pretty simple and understandable even then, as we need to explain differences from `Rust`.
 
To start off with, we'll try to call a `C` program from `Rust`. Here's the 
contents of my [hello.c](./introduction/hello.c):

```c
#include <stdio.h>

void say_hello(char *name) {
    printf("Hello %s!\n", name);
}
```

And here's the `Rust` code which will be using it ([main.rs](./introduction/main.rs)):

```rust
use std::ffi::CString;
use std::os::raw::c_char;

extern "C" {
    fn say_hello(name: *const c_char);
}

fn main() {
    let me = CString::new("World").unwrap();

    unsafe {
        say_hello(me.as_ptr());
    }
}
```

First, we tell the compiler that we'll be using an external function called 
`say_hello()`. The "C" bit indicates that it should use the "C" calling 
convention. A calling convention specifies low level details like how 
parameters are passed to a function, which registers the callee must preserve 
for the caller, and other nitty gritty details you only really need to know if 
you're a compiler writer or assembly programmer. For our use, it is enough to know that we must tell the compiler which approach it must use to correctly call the function.

Also, notice that we import `std::os::raw::c_char`, which we use in the signature we give to the external function. `Rust` and `C` both have a type called `char`, but they are different. `Rust`s `char` is a [UTF-8][utf-8] character, which, in memory, could take different lenghts. `C`s `char`s always have the same size, usually a byte. But `Rust` knows about these types, they hide in the `std::os::raw` module. 

The differences don't end there. `C` encodes strings as a pointer to some characters, where the end of the string is the [NUL][NUL] byte.
`Rust` encodes strings differently then `C`. Internally, they are a pointer to the same characters, but they also carry the _length_ along with it. This means that `Rust` doesn't use the NUL character.

Our `say_hello()` function is expecting a pointer to a null-terminated string, though, 
and `Rust` provides an easy way to create one of those is with [CString][cstring].

Almost all of what we're doing here purposefully sidesteps Rust's memory guarantees, so 
expect to see a lot more `unsafe` blocks. In this case, the reason is that the C function could do
whatever it wants with our string. But we know it doesn't do anything dangerous, and we indicate that to the compiler by wrapping the function call in 
`unsafe` and taking the responsibility.

Next you'll need to compile the `C` code into a library which can be called by
`Rust`. In this example I'm going to compile it into a `shared library`.

> **Note:** If you aren't familiar with the difference between a `static` and 
> `dynamic` library, or how you use them you might want to read 
> [this Stack Overflow question][static-vs-dynamic].

```bash
$ clang -shared -fPIC -o libhello.so hello.c
```

TODO: also give a gcc example.

The `-shared` flag tells clang to compile as a dynamically linked library 
(typically "\*.so" on Linux or "\*.dll" on Windows). You'll also need the `-fPIC`
flag to tell the compiler to generate Position Independent Code so that the 
generated machine code is not dependent on being located at a specific address 
in order to work. This basically means when invoking a function it'll use 
relative jumps rather than absolute.

Next up is compiling `main.rs`:

```bash
$ rustc -l hello -L . main.rs
```

The `-l` flag tells `rustc` which library it'll need to link against so it can
resolve symbols, and the `-L` flag adds the current directory to the list of 
places to search when finding the `hello` library.

Finally, we can actually run the program

```bash
$ LD_LIBRARY_PATH=. ./main
```

> **Note:** When you try to run a program, what actually happens is the 
> [loader][loader] loads the binary into memory, then tries to find any symbols
> belonging to shared libraries. Because `libhello.so` isn't in any of the 
> standard directories the loader usually searches, we have to explicitly tell
> the loader where it is by overriding `LD_LIBRARY_PATH`.

If we didn't override the `LD_LIBRARY_PATH` then you'd see an error something
like this:

```bash 
$ ./main
./main: error while loading shared libraries: libhello.so: cannot open shared object file: No such file or directory
```

There are much more elegant solutions than this, but it'll suffice for now.


[rust]:  https://www.rust-lang.org/
[book]: https://doc.rust-lang.org/stable/book/
[loader]: https://en.wikipedia.org/wiki/Loader_(computing)
[static-vs-dynamic]: http://stackoverflow.com/questions/2649334/difference-between-static-and-shared-libraries
[cstring]: https://doc.rust-lang.org/nightly/std/ffi/struct.CString.html

+++
title = "What I wished I knew when learning C"
+++

# What I wish I knew when learning C

This article is a response/take on both [Daniel Bachler's article on learning F#](https://danielbachler.de/2020/12/23/what-i-wish-i-knew-when-learning-fsharp.html) and [Hillel Wayne's on the hard part of learning a new language](https://www.hillelwayne.com/post/learning-a-language/). I thought it would be educational and a fun excersize to read this article in response to C.

Often teaching tutorials don't focus as much on the surrounding ecosystem of how to do development.

## Why would I want to use C?

C is many ways a foundational language to other ecosystems -- it is not unusual for other programming languages to depend partly or completely on a C compiler to bootstrap it's compiler or runtime, many languages use C as the defacto Foreign Function Interface, and for several systems system utilities are exposed as C libraries you call into.

- When you want to add extensions to multiple programming languages
- When you need to interface with a system or user library that only exposes a C API
- When you're writing something that requires not having a heavy runtime, like operating system or firmware on a resource constrained system.
- When using a more managed higher level language is too slow or has too much overhead
- When you are writing Driver code.
- When you're doing CTFs

## Why would I _Not_ want to use C

- C is a memory unsafe language, and you have to manage memory management yourself.
- When correctness is more important than performance (or where performance isn't constraining)
- When you need to do complex string manipulation
- when you're doing something that probably should just have GC

## The C compilation model

C has a somewhat peculiar way of compiling things if you're coming from other languages: Normally, you compile each `*.c` file to an object file; then you end up linking these together into a final executable using a linker, and then use some sort of archiving tool to create dynamic or static libraries.

Thus if you have a project with the files:

- foo.h
- foo.c
- main.c
- util.h

Compiling it using the traditional unix style toolchain would look like

```sh
> cc foo.c -o foo.o
> cc main.c -o foo.o
> ld main.o foo.o -o main
```

This model exists for almost all compilers, although you often do not execute it explicitly but rely on the build tool, or convenience utilities of your compiler to handle the intermediate stuff.

One of the advantages of this is partial builds -- if you keep around the object files after compiling something, you only need to recompile the object files for the c files you update.

## Building C

Like other older languages, C predates the design trend where language compilers come with an extended set of tools bundled together (see Go, Rust), and thus you have a large ecosystem set of possible build tools; some are meant to be generic, some are effectively tied to one system, some are explicitly tied to one system.

### Make (GMake, BMake)

Make is probably the oldest build system out there. While POSIX defines a very limited subset of rules and utilities, the major Make implementations (GNU Make and BSD Make) both have a much larger set of features.

Both also tend to rely heavily on the presence of other programs on your machine via a shell script.

### Ninja

Ninja is like a much more constrained Make, but was instead developed for meta-build systems like CMake below. It was designed to be fast to execute and build, and to not be written by hand.

### CMake

CMake bills itself as a "meta-build system" -- you write a project's build scripts in the CMake scripting language, and these are then compiled into a build target suitable for the system (such as Makefiles, Ninja, Visual Studio Code).

As CMake has grown into popularity, it's not unusual for libraries to define there self in terms of CMake so that other projects can pull them in (either as a git submodule or via a top level cmake-config script installed with the `install` target).

CMake also tends to change the layout of projects -- while Make-style targets usually put header files the same file as implementation files, CMake projects tend to put them in a distinct location and then add that to the header include path via commands, which means your IDE will probably have to have some introspection of the CMakeLists.txt command or it's output to allow intellisense to handle things correctly.

### Visual Studio

Visual Studio C has it's own format and build tool; but really would rather you edit it via the.

### Autotools

Autotools is also like CMake in that it is a meta-build system; unlike CMake, it has much more tenacious feature detection (to figure out what platform you are on) and the build files it creates are designed to be easily shipped, rather than the intermediate build artifact's CMake creates.

I haven't personally used autotools, so I really can't speak to it's quality.

### Meson

Another meta-build system, this time using a python-like syntax for it's scripting languages.

I haven't personally used this either, but AFAIK it's still constrained to unix like targets.


### Everything else under the sun

There several other build system I've heard of, but haven't used:

- A few of Google's large open source projects (Android, Chrome) use a homebrew build system.
- SCons, Premake and a few other attempts at trying to make a nicer make-like language
- Build tools originally meant for other languages that have utilities added for building C.

## Debugging C

Debugging C is something a lot of newcomers don't think about at first. 

### `printf()`

It's kind of dumb to say this, but sometimes it's useful to rely on printing out information to have some idea of what's going on in your system. More advanced projects will develop some sort of logging library, which is just a fancier printf that lets you add some extra information about where a call was made and controlling what sort of depth of information gets printed.

### Debuggers (gdb, lldb, windbg)

When `printf()` doesn't cover it, or you're getting a SEGFAULT, then you should bring in debuggers. These take advantage of system hooks to allow you to execute a program line by line and inspect the values of variables and parameters, and to show you were segfaults or other signals are triggered.

GDB is a perennial one, and provides both a command line interface and a remote protocol interface so that you can debug on remote machines.

Windbg is a windows-specific debugger that has two frontends, and can also be used for debugging kernel level information.

To use debuggers, you should probably put your code in debug mode/low optimization and add debug symbols, which are necessary for debuggers to introspect a binary.

### Valgrind

Valgrind is a much more heavy weight debugger and effectively acts as a native code interpreter you feed a binary program in, allowing you to inspect for errors that are not obviously until runtime, including more subtle ones like misuse of file descriptors.

It has a pretty major speed overhead, which may make it difficult to use for some projects or if they have memory usage. But you get a pretty wide suite of detection for errors.

### Sanitizers

Sanitizers perform similar tasks to Valgrind (finding higher level errors at runtime), but unlike Valgrind does not require loading the binary into an interpreter; instead all of the runtime checks and tooling are installed into the binary during compile time, allowing you to simply run the binary like normal and only seeing errors if one occurs.

Unfortunately, sanitizers are partially incompatible with each other, and thus the permutation of sanitizers you can have enable at once is constrained; in particular UBSan and ASAN are not compatible together.

### Static Analyzers

## Standards, system libraries, undefined behavior, and the ideal of "Portable Assembly"

A common idea that you might hear if you just go around searching the internet is that C is a sort of "portable assembler".

Dear Reader, I do not believe in being a hater, but this is Extremely False.

It _may_ have been true after a fashion with the earliest instantiations of C, B and BCPL in the 70s, but it is simply not true today.

C has a fairly constrained set of rules a compiler expects you to follow, although you can change the particular quirks of that dialect through compiler flags; in exchange, these extremely advanced compilers can do wildly destructive transformations of what you think your code "should" look into something that will run faster (or hopefully so).

### Standards

C has a set of language standards, handled through the standardization organization of the ISO; these standards are somewhat ironically not released publicly and require paying for them. Regardless, the draft versions of the standards are free and widely available, and what basically everyone who handles C language tooling will end up reading these instead.

The C standards define the core language, the abstract C machine, the primary library (libc), and a number of extension elements to libc.

- C89  
    Generally used nowadays as an extremely conservative target to hit, this was the first standardization of C.

- C99  
    This was the second major version and probably the other semi-conservative target people try to hit for general compatibility. In addition to clarifying the language, it also added several features including fixed sized arithmetic types, variable sized structs (_flexible array members_), and runtime sized arrays.

- C11  
    Major version that added support for threads

### The Abstract Machine

C, as it is defined in the various C standards, are defined around _the C abstract machine_. This is sort of idealized interpreter for the language that defines what would happen if a program executes this behavior. In practice, while C interpreters do exist, most C code is written with the intent of being designed.

### Other standards

The other major set of language standards C relies on is POSIX; which was created as a sort of unified set of common denominators between various Unix and Unix-like systems. These include the rest of the sort of libraries you tend to rely on in Linux for things like parsing arguments, manipulating file descriptors, network programming, and so on, threading support, spawning new processes, and so on.

A lot of stuff you think should be in libc is in fact inside of the POSIX set of libraries; libc does not actually support anything like the ability to iterate over the contents of a directory. To get around this Lua (which heavily relies on just libc) instead uses pattern matching globs to handle it when it occurs.

### Extensions and other things

Most major compilers (GCC, Clang mostly following GCC, and MSVC) often add their own custom extensions to the C language to make it easier to use. The Linux kernel actually relies on a large number of these, and is essentially written in a weird bespoke version of GNU-C; getting it to compile under Clang was an open challenge and I believe still relies on a custom define file.

Additionally, GNU's libc (glibc) adds a number of additional functions not defined anywhere else that software relies on.

MSVC, on the other hand, only supports a some of C99 (but has kept very up to date with C++ standards), and almost all of the operations programs will use instead rely on the Win32 library, which has an entirely distinct set of naming conventions, types, functions and other utilities.

It's also not unusual for very constrained resource devices (embedded) to define their own abstraction layer called a HAL, which often does not look like libc.

### You don't get portability for free

If that seemed like a bit of a long rant about various systems, it's because it was. But it has a point.

The point is -- you do not get portability for free.

The ideal of C you can sell people on is that it's this hyper portable language with a very constrained low-level platform, and as long as you carefully handle code sitting in that space, you can have code that can compile anywhere, from quirked up IBM mainframes running exotic architectures you haven't heard of, to you phone, to your laptop, to the tiny chip running your laundry machine.

And you know what? If you really are careful, C really is a whole lot more portable than other languages! You can write software that can maybe probably work in a whole lot of environments.

But it's not perfect, and almost every platform has a variety of weird quirks, and junk, and functions that don't work how you think you should, and odd defines, or distinct function prototypes. And often times, important things -- like working with threads, or IPC, or spawning new processes, or interacting with hardware, or anything to do with graphics -- are just not defined in a consistent way.

Almost any platform that tries to be portable either accepts its relying on something larger, or creates a set of portability shims/polyfill that implement behavior that most software actually wants, but whose details vary across systems. And this is a lot of work! It often can show up in strange places, and each additional platform you want to support will rely effort and testing on them.

It's also very easy to accidentally rely on a piece of implementation specific or even undefined behavior that Just Works on your machine, and it turns out you have created a sea of problems when you want to port your program over to a new platform.

This isn't a pure hypothetical, or for dealing with weird out of the way platforms: this happens to people porting a game from one major popular platform (the PC) to another (The Nintendo Switch)

> With that out of the way, the next step was multiplayer determinism. One big goal was that I didn't want to cut multiplayer from the game. Furthermore, I wanted players on PC to play with players on Nintendo Switch. This is the first time we had to make sure the game is deterministic between ARM and x86. We should be fine, C++ is portable, right? Just don't use undefined behavior. Turns out we use quite a lot of undefined behavior, both in our main code and in the libraries. For example, when casting a double to an integer, if the value does not fit in the integer, it is considered undefined behavior and the resulted value is different on ARM and x86 CPUs.

<https://factorio.com/blog/post/fff-370>

I'm saying this because the ideal that you can get this hyper portable language that you're software can run with on almost any platform can be intoxicating, but it's just no true. You should embrace this fact, and prioritize testing and development on the platforms you realistically want to support.

If it turns out an enterprising hobbyist can get your game running on the PSP, that's awesome! But you shouldn't create endless work for yourself to make this possible.

## Weird things you'll see in C

### The do/while macro block

I forget where I learned this trick, but a useful thing to do when you want to make a function-like macro that does not have a return value is to expand it out to a do/while block;

```c
#define DO_THING(x) do{\
        foo(x);\
    } while(0)
```

which allows you to call it like a function and still put following semi-colons.

```c
if (x > 5) DO_THING(x);
```

### `sizeof` operations on values

`sizeof()` is a compile timer operation that lets you calculate the size in bytes of either a type of an expression; because it is compile time, you can actually perform arbitrary expressions on it without the parenthesis. This is slightly mystify the first time you see it, because it _feels_ like you should be getting a segfault doing this.

```c
{
    char buffer[256];
    memset(buffer, 0, sizeof buffer);
    // or
    struct Foo* foo = NULL; 
    foo = malloc(sizeof *foo);
}
```

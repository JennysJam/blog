+++
title = "What I wished I knew when learning C"
+++

# What I wish I knew when learning C

This article is a response/take on both [Daniel Bachler's article on learning F#](https://danielbachler.de/2020/12/23/what-i-wish-i-knew-when-learning-fsharp.html) and [Hillel Wayne's on the hard part of learning a new language](https://www.hillelwayne.com/post/learning-a-language/). I thought it would be educational and a fun excersize to read this article in response to C.

Often teaching tutorials don't focus as much on the surrounding ecosystem of how to do development, what tooling to use, or some deeper lore.

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

## Installing a development enviroment: Toolchains and tools

The set of tools needed to build C are generally refered to as a _toolchain_, which traditionally includes the compiler, assembler, linker, pre-proprocessor and a set of other associated tools. The specifics are toolchain specific but inside of the \*nix world, they mostly follow a similar pattern. You don't need to generally worry about installing a particular part of a compiler since they are packaged together

### GCC

GCC (standing for the _Gnu Compiler Collection_) is one of the most common and well known of the major compilers avaliable to devs. It is avaliable for free (and is one of the canonical examples of Free/Open Source software), and generally can be installed via your system package manager. One of the advantages of GCC is it's age: it's a long running complex project, and it also has a number of compiler backend targets you can utilize, as well as a pretty large space of added extensions and utilities. Generally, GCC is the compiler I default to using unless I have a specific reason to choose another, although Clang is equally as worthwhile and useful. 

```shell
# install on Ubuntu
> sudo apt install gcc
```

### Clang

_Clang_ is the other major open source \*nix toolchains, and was developed in part to accept most of the extensions and compiler settings that GCC provides, meaning most software that uses GCC can probably be compiled with Clang. Clang and GCC are mostly compatible in terms of performance, so either are a fine choice unless you're relying on a particular backend or feature. It is also helpful to try and build software with both because it might pick up a number of errors the other might not. One thing Clang provides is that it is (by default, anyway) a cross compiler with the supported backends built in, whereas with GCC you will need to install a build of the compiler for that specific backend.

```shell
# install on Ubuntu
> sudo apt install clang
```

### MSVC

_MSVC_ is Microsofts in house C and C++ Compiler (they treat the barrier between the two languages a bit fuzzier than clang and gcc), and while it is not open source it is freely avaliable. To install it you have to install visual studio (which is _not_ visual studio code, the naming trips me up sometimes too), and versions of the compiler have until recently been strongly bundled with the broader visual studio environment, which you can download [here](https://visualstudio.microsoft.com/downloads/).

While you dont need to actually use the Visual Studio IDE if you don't want to and can instead rely on the standalone CLI tools, it's not unusual that this will cause some problems because those are not natively added to the envirnoment but are instead added to the environment through special shortcuts that open `cmd.exe` with all of the useful commandline files added, see [documentation for details](https://learn.microsoft.com/en-us/cpp/build/building-on-the-command-line?view=msvc-170).

One of the smaller benifets of CMake is that, as long as your MSVC is installed in a standard location, CMake can actually pick up on it's location and natively call it's tools for both the configure and build steps.

### Mingw

Mingw (and it's more up to date port Mingw-64) is a port of GCC designed to compile code for Windows, either as cross-compilation or hosted natively.
Generally speaking, MSVC is _the_ way to build native C/++ code for Windows; Mingw works and can handle compiling a large number of projects but the toolchain has some tradeoffs that using MSVC doesn't, like utilizing an old and undocumented systems library. Unless you have a strong reason to, I would recommend utilizing MSVC if you are building compiler native.

It can be installed on most repos via their package managers, and can also be downloaded [here](https://www.mingw-w64.org/downloads/).

## How should I be writing this: Text editors and IDEs

Since C is so old and utilized on so many platforms, most major text editors have some form of C/++ support, either builtin or through user provided packages.

### Visual Studio Code

Visual Studio Code is a cross-platform general purpose text editor and "light IDE", with a pretty huge repository of official and community supplied plugins. It is responsible for the creation of the [Langauge Service Protocol](https://en.wikipedia.org/wiki/Language_Server_Protocol), which has been adopted in other editors, and is overall a very handy and mature editor for a large variety of langauges. It also has a very useful remote editing mode. This is my go to default.

For setting up C/++ development specifically, these are wonderful tools provided by Microsoft themselves.

- [C/C++ Tools](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools)
- [CMake Tools](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cmake-tools)
- [Makefile Tools](https://marketplace.visualstudio.com/items?itemName=ms-vscode.makefile-tools)

### Visual Studio

Visual Studio, in addition to hosting the C/++ toolchain and other tools, is a fully featured and fairly rich IDE. It has a number of officially supported plugins for all the major langauges Microsoft supports, and has been working on the C editor experience for years. I don't generally use it, but it's a solid and mature bit of tech, and if you are targetting or developing on Windows it's a solid choice.

### CLion

CLion was developed by Jetbrains, who have built a number of langauge specific editors, and CLion is there C/C++ specific tooling. It generally relies on CMake as its build system, although it can import projects utilizing Makefiles or other constructs. It's a paid IDE and costs about $100/year, or less if you buy the full bundle of Intellisense tools. I quite like CLion, although a few choices (like some of the default formatting) are a bit fiddly and I have to fight against or edit the configs, but is a rich editing environment and I'm happy to have paid for it. Unfortunately it's third party plugin ecosystem is much more sparse than the other code editors on this list.

CLion can be installed [here](https://www.jetbrains.com/clion/).

### Vim

Vim (and it's quasi fork Neovim) is a pretty old and advanced command line based editing environment. By default it only has syntax highlighting for code, but has a pretty huge developer code base so you can absolutely fill up with tons of utility. Generally some of the bigger advantages of Vim is that it's likely to be present most computers, and you can use it if you are developing code remotely via SSH.

## Who should I ask for help: Documentation

### Manpages

The historical way of finding development for C langauge functionality have been the _Manpages_, a database of documentation and a command line client that is pre-installed on almost all *nix style computers. Generally, utilities, the system libraries and many libraries will put documentation in a manpage, which are normally written in a simple type setting system called Roff.

Manpages are ordered into sections which act as namespaces ([See this handy manpage for details](https://man7.org/linux/man-pages/man7/man-pages.7.html)). Generally, the sections you'll want are in section 1 (command line tools), section 2 (syscalls and their wrapper functions), section 3 (library functions), and occasionally section 7 (overview or general topics).

Online versions of the manpages are hosted at several locations ([The linux kernel project mains one](https://man7.org/linux/man-pages/index.html)), but these might have differences from the versions installed on your computer. I would instead recommend the extremely useful [DWWW](https://packages.debian.org/sid/dwww), a local webserver that transforms the manpages into HTML and indexes them to be searchable.

Unfortunately, manpags are slightly out of vogue, so it's not unusual for projects to no longer provide manpages or provide fairly spartan manpages with the expection you'll look elsewhere.

### Gnu Info

Gnu Info was designed to act as an upgrade of sorts to the manpages -- manpages were small and standalone, while Gnu Info was meant to have more of a hyptext-style set of linked pages.

Gnu Info isn't used as much (not even as much as manpages, in my experience), but many Gnu projects utilize it (oftentimes provided a very spartan manpage telling you to look up the information in Info), but otherwise I think it's mostly passed. Even the Gnu projects often provide online HTML or PDF formats for their documentation.

### Locally built documentation: Doxygen

Doxygen is both a tool and general format for providing documentation inline in a C++ file -- while it's not unusual for developors to use doxygen style comments to add documentation to their code but not actually use Doxygen proper, it's still pretty common for older projects.

Notably, the doxygen format allows for some more complicated type settings and for including in other files.

If a project has doxygen documentation, you can build it locally and thus have an up to date version. Active project often host their documentation online, but usually only for the latest or long term release.

### Microsoft Development Network (MSDN)

Microsoft provides documentation for their compilers, language extensions, and libraries on [MSDN](https://learn.microsoft.com/en-us/cpp/?view=msvc-170), and it acts like the manpages for Linux do, often as primary form of documentation.

## The C build process

C has a somewhat peculiar way of compiling things if you're coming from other languages: Normally, you _compile_ each `*.c` file to an _object file_; then you end up linking these together into a final executable using a _linker_.

If you have a project with the files:

- foo.h
- foo.c
- main.c
- util.h

Compiling it using the traditional unix style toolchain would look like

```sh
> cc -C foo.c -o foo.o
> cc -C main.c -o foo.o
> ld main.o foo.o -o main
```

While I called this model "peculilar", the only element that is unusual is the use of textual inclusion for header files; most other aspects are pretty similar to what most compiled langauges do, the traditional C development environment only makes this pretty explicit.

One of the advantages of this is partial builds -- if you keep around the object files after compiling something, you only need to recompile the object files for the c files you update.

Depending on your build system, the intermediate ouputs of compilation might be stored either in the same directories as the source code (generally refered to as _in tree builds_), or stored in a seperate build directory (_out of tree builds_). Makefile-based projects lean heavily towards in tree builds, and CMake leans heavily towards out of tree.

### Includes and the include path

C has two flavor of ways of including header files

```c
#include <stdint.h>
#include "somefile.h>
```

_Technically_ the definition is implementation defined, but universally the meaning ascribed to these is that `<>` searches through some global set of paths, whereas the `""` include style first searches the local directories for this.

In both cases, there's a set of paths pre-determined by your development environmnet and toolchain, and you can add files to the include path as a command line argument, normally `-I /include/path`. This is relied on heavily by project that move their header files to a seperate directory, although usually that is handled by the build system in some flavor.

### Macros and the preprocessor

The C preprocessor is also, technically, a standalone tool you can run standalone on other files, although the only other major example I can think of is Fortran. It is still quite strongly tied up with the langauge definition, and C macros are required to follow the rules of valid identifiers in C code (although as they are textual macros, you can do _wild_ stuff with them like passing `*` as an argument to a function).

If you ever need to debug macro code, you can call `gcc -E code.c -o code_preprocessed.c` in \*nix land, [MSVC uses distinct arguments](https://stackoverflow.com/questions/3917316/gcc-preprocessor).

### Assembly and the assembler

Technically, the C code to object file pipeline has a 2nd intermediate step where you compile code to a format that is (in some ways) a textual representation of what native code for your target architecture looks like, traditionally called _assembly_, which is transformed into native code using an _assembler_. This format is normally not output unless you specifically ask for it. On \*nix machines it traditionally has the file extension `.s` or `.S` (if you use the preprocessor), while on Windows it is `.asm`.

In addition to it being useful to look at a textual output of your final code, you can also write files in Assembly and manually invoke the assembler like `as foo.s -o foo.o`. This is usually something you don't need but is sometimes used if you need extreme performance and need to handtune the output of your code, or for operating systems that need to provide the low leve implemtations of system calls that cannot otherwise be done in C.

Note that assembly syntax is _extremely_ target and vendor specific -- each machine architecture probably has it's own syntax and quirks, and the X86 family of architectures makes sure to go the extra mile by having _two_ syntaxes, and even two competing assemblers might differ in details beyond the base syntax.

### Linkage and the linker

The final step of the traditional build pipeline is _linking_, where you take all of the intermediate code in the object files and fuse them together into a final executable or library. This step involves laying out sections of data in the executable, and making sure that function calls to functions defined outside of an object file are correctly linked up.

There is a custom langauge to control and manipulate how the linker behaves called _linker script_. Unless you are writing a compiler this a pretty rare format to interact with. [McYoung has a fantastic guide to the details of what interacting with linker script is like](https://mcyoung.xyz/2021/06/01/linker-script/).

### Bundling libraries

In \*nix land you normally build a static library via utilizing `ar` (originally an archive manager) to bundle all of your files together, and shared libraries via a special flag to the compiler. While these create file that toolchain (and for dynamic files, the loader) to use, there normally is a more complex set of steps for actually bundling code to distribute to other libraries.

## The C Environment

If you're writing C, you're probably doing it in what the standard calls a _hosted_ environment: You are relying on the presence of an operating system to help manage things for you. There is also so-called _free-standing_, where you can not rely on the host operating system or libraries; this is the case for very resource constrained devices.

### Basic environment

If you've used many other programming languages, some or all of these will be familiar to you. This is because a lot of programming languages borrowed from C or even simply delegate their logic to the equivalent C.

### Entry point

C traditionally uses a function called `main()` as the entrypoint for an executable -- it is the first piece of user code that is run (Although it's likely other functions are called before this in order to set up pieces of an environment.)

```c
int main(int argc, char* argv[]) {
    //...
    return 0;
}
```

When you invoke a program on the command line like `./foo 1 2 3`, those values are placed by the hosted environment in `argv`, and the numbers of arguments + 1 are recoreded in argc.

`argv[0]` actually contains the string describing how a function was called: here it would be `./foo`. Multiple-application binaries like [Busybox](https://busybox.net/) utilize this to know what application to pretend to act like.

Some platforms expose the environmental variables as a third paramter to `main()`.

### Environmental variables

Another aspect of the C runtime is that of "environmental variables": this is basically a Key-Value store of strings that are initialized when a program is run. It is often used to edit information about library loading, or to customize the behavior of a function without explitly passing arguments or recompiling the binary.

```c
#include <stdlib.h>

int main(int argc, char* argv[]) {
    char* envvar = getenv("FOO_LOG");
    if (strcmp(envvar, "FOO_LOG") != 0 ) {
        printf("Enabling logging!\n");
    }
    else {
        printf("Disabled logging!\n");
    }
}
```

### File Streams

One of the few abstractions built into the standard library for interacting with the host, _Files_ ( or _Streams_ ) are a way of abstracting storing data in a filesystem. Normally that means reading or saving information to a hard drive or SSD.

However many other filesystems exist, for instance representing a remote storage location, read-only filesystems that are used on low-powered embedded devices to storage behavior, and so on.

When an executable begins, three special files have already been initialized:

- stdin: stream of information that is fed into a process
- stdout: stream of information that is fed out of a process
- stderr: a second stream of information that is used to describe diagnostic problems

Normally, stdout and stderr both are printed to the command line; and stdin will record any keypresses you make while the process is running.

### Signals

C's signals are a simple form of _Interprocess Communication_ (IPC): they can be used to send a request to another Process to perform some logic. Signals are defined as integers with no additional information, and you can associate a function via a function pointer to handle them.

For instance, debugger's like GDB set the Signal Interupt (The signal sent if you hit Ctrl-C to interupt a program) handler to bring up an interactive interface so you stop execution to perform manipulation.

There are numerous signals used by C, but the one that you'll probably become the most familiar with is `SIGSEGV`, also known as a segmentation fault (_segfault_ for short). This occurs if you perform a memory violation, normally dereferencing a null or invalid pointer, or attemping to write to memory that is mapped readonly.  

### Locales

C has attempted to formalize the concept of internationalization (editing behavior to fit in different languages and cultures) with _locales_, which are used for adding logic to display text or parse input text.

By default, C will be loaded with the `"C"` locale, which is mostly equivalent to English.

### Errno

`errno` is a bit of an odd oddball: the C standard library has several functions that do not have an obvious way of indicating that they failed (say, when a function to parse text to numbers fails). For most of these, they indicate failure by setting _errno_ with some integer value, which you must then check to see if it suceeded or not. The integer value 0 (called `ERR_OK`) is used to indicate no error.

```c

#include <errno.h>
#include <stdio.h>

int main(int argc, char* argv[]) {
    if (argc != 2) {
        return 1;
    }

    errno = 0;

    long i = strtol(argv[1], NULL, 10);
    if (errno != 0) {
        printf("Invalid number!\n");
        return 0;
    }
    else printf("Sucessfuly parsed!\n");
    return 0;
}
```

After Threads and threading became common, `errno` shifted from being truly global to being a `thread_local` value, which means there is one errno value for each thread.

Errno is something most C projects only interact with to see if a standard library has failed, it is quite rare for code that isn't in libc or the posix extended libraries to change it to indicate error.

### Memory allocation

C requires you to handle memory allocation yourself, and does so through a standard set of a few functions to map new memory in and make it avaliable to the user.

- `malloc(size)` to allocate new storage for values
- `realloc(pointer, new_size)` to adjust or re-allocate a current bit of storage
- `free(pointer)` to release memmory so it can be used later by another allocation

### Nonloacl control flow

The C stdlib exposes two special functions (and commonly more) called `setjmp()` and `longjmp()` 

## Libraries

Whenever you need to rely on behavior that's not built into the langauge, you'll have to rely on a _library_. Libraries are a way of tying together functions for performing actions as well as interacting with the host systems. For instance, _SDL_ is a library with functions that let you draw things to a screen; _libjson_ is a library with functions for parsing and serializing data to the JSON interchange format.

There's two functional varieties of libraries in use: static and dynamic libraries, both utilizing header files to define their interfaces.

### Header Files

_Header files_ are not technically considered a distinct construct by the C standard, but are so defacto their built into the naming convention of files: files meant to be `#include` use a file extension of `.h`.

Header files are often used internally in projects to logically seperate details of a project seperate, and often it's normal for there to be a pair of `.c` and `.h` files for one part of a program, e.g. `parser.h` and `parser.c` for the code in a compiler to parse code.

Header files are also used to describe the interface a library exposes to consumers of a library: normally header files contain prototypes for functions that the library implement, definition of types for them to operate on, and macros for both utility and to describe architecture specific behavior. The only way for a library to expose it's functions and constructs is via a set of header files.

### Static Libraries

These are technically the older of the two concepts. A static library is a grouping of functions in a way that can be _linked_ into a binary when that binary is being constructed. This allows you to fix in behavior and not rely on the presence of a dynamic library on the users system.

Some of the advantages of static linking are that you can set behavior at compile time, and you can prune any of the functions and values that are not used by the binary.

Some larger projects also use static libraries to break up implemetnation and usage -- if you're creating a video game, you could put all of the main logic in a library called `libgame` in a function called `startgame()`, and have one executable called `game` that immediately calls `startgame()`, and another executable that instead is test the rendering system with examples. 

The disadvantages are that you are permanently locking in information, rather than relying on potentially more update dynamic libraries. This could be very bad if the version of your networking library has an error that opens you up to security faults, or if a change in unicode adds a number of new digraphs an older version of the library couldn't handle.

### Dynamic Libraries

Dynamic libraries are the opposite of static libraries: the functions you are using are not loaded in til the executable is run, and then those functions are fetched in so you are always using the implementation the system provides, even if you change those functions out.

Dynamic libraries are also often used for plugins: since you can manually load in libraries and associated values with Unix `dlopen()/dlsym()` and Windows `LoadLibrary()/GetProcAddress()`, you can load in additional logic to be run without baking it in. A video game might do this to load in mods, or a text editor to add support for a new programming language.

The advantage of dynamic libraries are that they are always loaded in fresh, and thus you can fix security faults or add new behavior to older programs, and they can also save on system RAM -- if you don't have to load a copy of the math functions into every running binary, then you save a good deal of extra space.

The problem with dynamic libraries is to take advantage of these benifets, you have to make sure that the library is the right version to match what your executable expects to use, which in the worst possible case involves bundling copies of the dynamic library you are relying on with your code!

### Libc and other special libraries

There is one library that is generally singled out for a special purpose: the standard library, usually called _libc_. Almost all hosted programs utilize libc, which exposes most of the functions in the standard, as well as manipulating some of the core runtime features like files and signal handling. While libc could just be a special library, it is often intimitately tied into the runtime system, and some compilers hard code expectations about what will be in it.

In addition to libc, a few other libraries exist in the sort of special place where they are likely tied to the compiler (or even part of libc)

- libm, which implements several math functions
- libdl, which implements functions for working with dynamic libraries 

### Header only libraries

Header only libraries are more of a concept in the C++ world, but do seem some usage in C. Because of the complexity of build systems and wide range of practices, sometimes developers take advantage of the way `inline` allows for multiple definitions in a project to provide both the prototypes and implementations in a header file, which means that once you have the downloaded somewhere on the system you have no additional work to build it with aside from `#include <headeronly.h>`.

## C Build Systems

Like other older languages, C predates the design trend where language compilers come with an extended set of tools bundled together and suggested structure for what a project should look like. Instead, there's a pretty wide range of different tools to use, and each implicitly suggests a certain structure.

### Why use a build system?

You _could_  just make a buildscript that looks like `gcc *.c -o foo`, but while that might work for very small projects it quickly becomes a headache if your project is very big, or have a more complicated layout.

Build systems are a way of describing what actions to take to compile a set of source files into a final form (or several final forms). They allow to describe this in a more concise, declarative form: working with the command line version itself might get messy if you need to only compile `foo_windows.c` on Windows, but need to compile `foo.c` on both Linux and MacOS.

Additionally, build systems keep track of when the last time you edited a file, and only compile the source code files that have been changed. This doesn't seem like a big deal if your dealing with a small project that compiles in under a second, but it really matters if you're working with a massive codebase that can take hours to compile. 

### Make (GMake, BMake)

Make is probably the oldest build system out there. While POSIX defines a very limited subset of rules and utilities, the major Make implementations (GNU Make and BSD Make) both have a much larger set of features.

Generally, I use Make for for smaller projects and simple examples because it is very easy to spin up with, and to add items to it incrementally. I generally lean away from using it if the project is expected to be quite large, or rely lots of complex conditional logic.

The sample for our little project might look like

```make
.PHONY: all clean

CFLAGS= -Wall -Wextra

all: foo

foo: main.o foo.o 
    @$(CC) $(CFLAGS) main.o foo.o $(LDFLAGS) -o foo

main.o: main.c foo.h
foo.o: foo.c foo.h

clean:
    @rm -rf *.o foo
```

### Ninja

Ninja is like a much more constrained Make, but was instead developed for meta-build systems like CMake below. It was designed to be fast to execute and build, and to not be written by hand.

### CMake

CMake bills itself as a "meta-build system" -- you write a project's build scripts in the CMake scripting language, and these are then compiled into a build target suitable for the system (such as Makefiles, Ninja, Visual Studio Code). It also acts as a sort of configuration system so that libraries using CMake can be easily built and consumd by other projects also using it with a minimal ammount of glue code.

CMake also tends to change the layout of projects -- while Make-style targets usually put header files the same file as implementation files, CMake projects tend to put them in a distinct location and then add that to the header include path via commands, which means your IDE will probably have to have some introspection of the CMakeLists.txt command or it's output to allow intellisense to handle things correctly.

Generally I use CMake if I expect the project to be bigger, involve a lot of fiddly logic, or if I'm using C++. It has a bit of a nasty reputation, but as long as you internalize a good way of structuring things I find it isn't too bad, just verbose. I prefer using most of the style suggestions in [An Introduction to Modern CMake](https://cliutils.gitlab.io/modern-cmake/).

### Visual Studio

Visual Studio C has it's own format (which is a specialized XML file) and build tool; but really would rather you edit it via an exposed gui interface. I've used it a decent ammount, but it's quite hard to easily show GUI interfaces textually. I don't have anything against it either -- if you are making a Windows-primary thing and using Visual Studio, you should probably either use this or CMake.

## Debugging C

Debugging C is something a lot of newcomers don't think about at first, but its necessary as your codebases grow in scope to be able to inspect a program as its running to figure out possible errors.

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

## Standards and versions

C is a _standardized langauge_, meaning there is a set of canonical documents describing it and the runtime, environment and so on. This is a trend that's become less popular in newer languages as compiler development has become more open source, but historically this was valuable to ensure your code didn't accidently rely on vendor specific extensions that would make it difficult to port your code somewhere else.

### The Standards

The major versions of C are laid out in the ISO Standards documents. Unfortunately, all of the actual standards documents themselves are behind a decently pricey paywall, so most of the community relies on utilizing the final drafts, since these are almost surely quite close to the final standard.

These aren't just hypotheticals -- GCC and Clang both take an option specificying which version of the standard you want to adhere too. Companies or individual projects often will strongly specify what standard is allowed, and different standards expose specific functionality.

Developers interested in portability often target older versions of the standard, usually either C89 or incresingly C99. If you utilize the command line arguments to select a particular dialect, you might end up disabling access to some of the functions specified by POSIX or your local environment.

#### C89 (aka ANSI C)

[Draft avaliable here](https://web.archive.org/web/20200909074736if_/https://www.pdf-archive.com/2014/10/02/ansi-iso-9899-1990-1/ansi-iso-9899-1990-1.pdf).

This is the earliest standardized version of C. Often times, this is the baseline people wanting to make their code generally usable across a wide swatches of platforms will default to. The primary difference between it and later versions are that all variables must be declared at the the start of a block, and several standard types and functions were later added to the library.

Fun fact: the reference compiler of C89 was actually written in pre-standardized C++!

#### C99

[Draft avaliable here](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1256.pdf)

C99 is the other major version of C people treat as a generally safe common denominator. The major syntactic additions it added are variably length structs via _flexible array members_ and _variable length arrays_ (later walked back in C11), commplex number types, the C++ `//` single line comment style, variable declarations allowed in the middle of a block, and the addition of several types for fixed size arithmetic.

#### C11

[Draft avaliable here](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1548.pdf)

C11 makes some of the new addition in C99 (like variable length arrays) optional. It also adds a limited form of compile time generics via the `_Generic` directive, as well as support for threads and associated multi-threading primitives.

#### C17

[Draft avalaible here](https://files.lhmouse.com/standards/ISO%20C%20N2176.pdf)

C17 was a pretty small standard that added few things, and mostly depreceated or clarified positions in the C11 draft.

#### C2X

[Current working draft here](https://open-std.org/JTC1/SC22/WG14/www/docs/n3096.pdf)

The current version of the standard being worked on appears to be a major change and thus this section likely to be out of date soon, this has changed a number of technical details (several old macros have now become keyword), introduced C++ style attributes, a preprocessor directive used for embedding files directly as binary arrays.

### POSIX

POSIX refers to a set of specifications made in the 90s and later updated, trying to find the minimal common set of library interfaces, tools and behavior that were in common between them. The libraries defined in POSIX are the "rest" [and a quick listing is avaliable here](https://en.wikipedia.org/wiki/C_POSIX_library).

### Glibc and GCC extensions

GCC and the GNU Projects major libc implementation (`glibc`) provide a huge number of extension functions and utilities, and software has come to rely on it. Like Windows, the glibc developers have focused on backwards compatbility, so it acts as a bit of a de facto standard much like the Win32 environment does.

### The Win32 Environment

While Windows doesn't define any sort of standard describing it's libraries and interfaces, Microsoft has a pretty extreme dedication to reverse compatbility that means you can effectively rely on the Win32 C environment to act as a stable environment to build software against.

## Portability, Undefined behavior, and memory safety

### Defined and undefined behavior

### The dangers of undefined behavior

It's also very easy to accidentally rely on a piece of implementation specific or even undefined behavior that Just Works on your machine, and it turns out you have created a sea of problems when you want to port your program over to a new platform.

This isn't a pure hypothetical, or for dealing with weird out of the way platforms: this happens to people porting a game from one major popular platform (the PC) to another (The Nintendo Switch)

> With that out of the way, the next step was multiplayer determinism. One big goal was that I didn't want to cut multiplayer from the game. Furthermore, I wanted players on PC to play with players on Nintendo Switch. This is the first time we had to make sure the game is deterministic between ARM and x86. We should be fine, C++ is portable, right? Just don't use undefined behavior. Turns out we use quite a lot of undefined behavior, both in our main code and in the libraries. For example, when casting a double to an integer, if the value does not fit in the integer, it is considered undefined behavior and the resulted value is different on ARM and x86 CPUs.

<https://factorio.com/blog/post/fff-370>

I'm saying this because the ideal that you can get this hyper portable language that you're software can run with on almost any platform can be intoxicating, but it's just no true. You should embrace this fact, and prioritize testing and development on the platforms you realistically want to support.

If it turns out an enterprising hobbyist can get your game running on the PSP, that's awesome! But you shouldn't create endless work for yourself to make this possible.


## Dependency management

Or, _where **are** libraries exactly...?_

In what has probably become a theme now, C is old enough that it predates the sorts of environments that necessitate handling depencies and libraries.

### System wide package manager

This is a bit more of a Linux construct, but probably one of the default ways many software engineers rely on is using the system wide package manager and relying on your local distribution managers to have built and packaged up a library, header files, and any other additional configuration details in a well known place.

For instance, if I want to use the cryptology library, then I can pull install `libopenssl` (and on most systems, the other necessary build items in `libopenssl-dev`) via the command line and begin developing my application.

```shell
> sudo apt install openssl openssl-dev
```

This is one of those things where most of the time, it _just works_ no problems asked. It also has some nice advantages: if you are distributing your code as source, then you can build it and link it against the one installed on your system and can rely on things that are baked into the vendored version.

Unfortunately, if you are using a rarer library, an older or newer version than supported, or performing cross or are relying on certain features or properties that the packager for your system has opted against it, then you might need to rely on another method anyway.

### Git submodules and building the library as part of the build step

### Manual vendoring

## Neat tricks and odd things

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

### Typedef of a recursive structure

Sometimes you need to use the name of a structure within itself; one of the most ergnomic ways I know of doing this is using both the tag name inside of a typedef itself.

```c
typedef struct Object {
    int type;
    struct Object* next_object;
    char name[32];
    void (*finalizer)(Object* object, void* data);
} Object;
```
**LLJVM** provides a set of tools and libraries for running comparatively low level
languages (such as C) on the JVM.

The C to JVM bytecode compilation provided by LLJVM involves several steps. Source code is first compiled to [LLVM][llvm] intermediate representation (IR) by a frontend such as `llvm-gcc` or [clang][clang]. LLVM IR is then translated to [Jasmin][jasmin] assembly code, linked against other Java classes, and then assembled to JVM bytecode.

The use of LLVM IR as the intermediate representation allows more information about the source program to be preserved, compared to other methods which use MIPS binary as the intermediate representation (such as [NestedVM][nestedvm] and [Cibyl][cibyl]). For example, functions are mapped to individual JVM methods, and all function calls are made with native JVM invocation instructions. This allows compiled code to be linked against arbitrary Java classes, and Java programs to natively call individual functions in the compiled code. It also allows programs to be split across multiple classes (comparable to dynamic linking), rather than statically linking everything into a single class.

Also note that while C is currently the only supported input language, any language with a compiler targeting LLVM IR could potentially be supported.

For a **quick demonstration** of some pre-compiled classes, download the [runtime library][lljvm-jar] and the [demo package][lljvm-demo-jar] to a directory on your machine, and run `java -jar lljvm-demo-0.2.jar`.

To **quickly start** translating your own programs you might want to use [Docker][docker]. Getting LLJVM to run on modern machines (and keeping it running) can be quite tricky as it depends heavily on an old version of LLVM. As an alternative, Martin Haye has set up a virtual machine image [here][mhaye-lljvm] that is pre-configured for and includes LLJVM. Install Docker (you may need Boot2Docker) and start it this way: `docker run -i -v /your/project/dir:/project -t mhaye/lljvm /bin/bash`

To compile LLJVM from source, follow the instructions below. If you have problems compiling from source and are running Linux on an i386-compatible platform, download the [binary release][lljvm-bin], extract it, and download the [runtime library][lljvm-jar] into the resulting directory.

- [HN comments][hn-lljvm]
- 2009-2015 [David A Roberts](https://davidar.io)


## INSTALLATION
[LLVM 2.7][llvm] must be installed to compile and run LLJVM. [Jasmin][jasmin] is needed for
assembling JVM bytecode.

To compile LLJVM, simply call `make` in the project root directory, and call
`make check` to run the testsuite.

To generate the the documentation for the java packages and the backend, call
`make doc`, which will generate HTML documentation in `doc/java/` and
`doc/backend/` respectively. PDF documentation for the backend can be obtained by
calling `make doc-pdf` (requires PDFLaTeX), which will generate
`doc/backend.pdf`. A zip archive containing all of the documentation can be
created by calling `make doc-zip`.


## C COMPILER
The `lljvm-cc` frontend ties together several components to provide a C source
code to JVM class file compiler.

In order to use the frontend, either `llvm-gcc` or `clang` must be installed.
`llvm-gcc` is recommended, and is used instead of `clang` if both are available.
It is also required that the classpath contain [jasmin.jar][jasmin-jar].

To compile a small number of source files to a class file, the following style
of command can be used:

    lljvm-cc foo.c bar.c -o foo

This will generate `foo.class`, as well as a shell script `foo`, which sets the
classpath appropriately and executes the classfile with the arguments passed to
it. This allows the class to be called in the same way as any other executable.
If calling the class without the shell script, note that the entire list of
arguments must be passed to the class, including argument 0 (the name of the
command).

For a larger number of source files, it is often more efficient to separately
compile each file to an object file as needed:

    lljvm-cc -c foo.c
then link the object files together into a class file:

    lljvm-cc -link foo.o bar.o baz.o -o foo
Note that the -link flag must be used if object files are being supplied
instead of source files.

To link the object files together as a library instead of an executable,
`-link-as-library` can be passed instead of `-link`. In this case the shell script
will not be generated.

To link against a library generated by `lljvm-cc`, the `-l<name>` flag can be used.
This will search for a library in the classpath called `lib<name>.class`.
Additional directories can be added to the classpath with the `-L<path>` flag.
Such additions to the classpath will also be added to the shell script, so
when linking an executable, the `-L` flag must be passed for each directory
containing libraries needed by the executable (except the current directory and
directories in the system classpath).

An exception to the above is the `-lm` flag. This will not search for `libm.class`,
but will rather link against `java.lang.Math` and `lljvm.runtime.Math`. Also, `-lc`
should not be used as `libc` is linked by default. If this is not desired, then
the `-nostdlib` flag can be used.

By default all classes generated are placed in the default package. To assign
the class to a specific package, the `-classname` flag can be used. For example,
the following command:

    lljvm-cc ... -classname=com.example.foo -o bar/baz
will create a class named foo, in the package com.example, placing it in the
directory `bar/com/example/`. The shell script will be output to `bar/baz`.

If the `-g3` flag is used, then Jasmin assembly with full debugging information
will be output to `<output>.j`.

In addition to the above flags, any flag accepted by `gcc` or `ld` can also be
used. However, sometimes these flags may not be passed to the correct
component (this is a bug with `lljvm-cc` and should be reported).

The frontend can also be used to compile `autoconf`-based projects relatively
easily (although sometimes some changes to the project's build system may be
required):

    ./configure CC=lljvm-cc LD='lljvm-cc -link'
    make CCLD='lljvm-cc -link'


## DEMO
To demonstrate the capabilities of LLJVM, several common software packages can
be compiled with `lljvm-cc` by entering the `demo/` subdirectory and calling
`make`. To verify that all of these compiled correctly, call `make check`.

A JAR archive containing all of the demo programs can be created by calling
`make demo` in the project root directory. For usage information, execute the
JAR in the project root directory with no arguments:

    java -jar lljvm-demo.jar


## BACKEND
The LLJVM Backend transforms LLVM IR into Jasmin assembly code.
It can be invoked by:

    lljvm-backend foo.bc > foo.j

There are two flags that are accepted by the backend: `-classname` and `-g`.
Run `lljvm-backend --help` for more information.

The output file should then be linked by the LLJVM Linker (see below), and
assembled into a class file by [Jasmin][jasmin]:

    java -jar jasmin.jar foo.j


## RUNTIME
The LLVJM Runtime has three components, the Core Runtime (`lljvm.runtime`), the
I/O Support Library (`lljvm.io`), and the C Standard Library (`lljvm.lib.c`). See
the JavaDoc documentation for further details on the former two. The latter is
[Newlib][newlib] compiled to JVM bytecode by `lljvm-cc`.


## TOOLS
There are two command-line tools available: the linker, and the info utility.
There are several ways to invoke these tools. The simplest way is to call them
directly from the jar archive:

    java -jar lljvm.jar <cmd> args...

Alternatively, if `lljvm.jar` is already in the classpath, one of the following
can be used:

    java lljvm.tools.Main <cmd> args...
    java lljvm.tools.<cmd>.Main args...


## LINKER
The LLJVM Linker qualifies references to external methods and fields in Jasmin
assembly code. At the top of the code (before any `.method` directives), external
references should be specified through the `.extern` pseudo-directive, such as:

    .extern field foo I
    .extern method bar(I)V

Then the linker can be invoked by:

    java -jar lljvm.jar ld LIBRARY... < INPUT.j > OUTPUT.j

For example, the following assembly code:

    .extern method cos(D)D
    .extern field NULL I
    ...
    invokestatic cos(D)D
    getstatic NULL I
    CLASSFORMETHOD cos(D)D
linked with the command:

    java -jar lljvm.jar ld java.lang.Math lljvm.runtime.Memory
would produce:

    ...
    invokestatic java/lang/Math/cos(D)D
    getstatic lljvm/runtime/Memory/NULL I
    ldc "java/lang/Math"


## INFO
The info utility lists the type signatures of the public static fields and
methods provided by a class. For example, to list those provided by `libc`:

    java -jar lljvm.jar info lljvm.lib.c

By default, any identifiers beginning with an underscore are omitted. To
disable this behaviour, pass the `-v` flag:

    java -jar lljvm.jar info -v lljvm.lib.c


[llvm]: http://llvm.org/
[jasmin]: https://github.com/davidar/jasmin
[jasmin-jar]: https://github.com/davidar/jasmin/raw/master/jasmin.jar
[newlib]: http://sourceware.org/newlib/
[hn-lljvm]: http://news.ycombinator.com/item?id=961834
[lljvm-jar]: https://github.com/davidar/lljvm/releases/download/0.2/lljvm-0.2.jar
[docker]: https://www.docker.com/
[mhaye-lljvm]: https://registry.hub.docker.com/u/mhaye/lljvm/
[lljvm-demo-jar]: https://github.com/davidar/lljvm/releases/download/0.2/lljvm-demo-0.2.jar
[lljvm-bin]: https://github.com/davidar/lljvm/releases/download/0.2/lljvm-bin-linux-i386-0.2.tar.gz
[clang]: http://clang.llvm.org/
[nestedvm]: http://nestedvm.ibex.org/
[cibyl]: https://github.com/SimonKagstrom/cibyl

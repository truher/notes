(an old draft post; I could delete it, or I could publish it, so here you go.)

-----

As part of my job at Recurve, I've been gluing together a piece of **Fortran** code with my Java application recently, which is a fairly uncommon thing to do, so I thought I would write down some of what I've learned, for your benefit. If you're curious what the system actually does, contact me.

## Compiling.

I have several deployment targets, including windows (32 and 64 bit), linux (32 and 64 bit), and, eventually, mac. We use all of these as development platforms too, so I'd like to have a fully connected canadian cross toolchain. Out of the box, gfortran works pretty well for linux host and target, i.e. install gfortran-multilib and the "-m32" and "-m64" options work fine. I tried mingw for linux host, windows target, and it also seems to work great. It was surprisingly easy. I haven't yet tackled the windows host; we use cygwin in development, so i'll probably try that first. I haven't tried to get mac working at all, either as host or target.

Our native build artifact is a library that Java can load dynamically, i.e. a ".so" or ".dll" or ".dylib."

## Deploying.

In production, we deploy to Linux using some simple scripts, and we deploy to windows and mac using Java Web Start (JWS). The JWS idea is to create a jar manifest, which it uses to pull all the required jars, squirrelling them away somewhere that its classloader knows about. It works for java and native code; just jar up the native library and java will find it. So far, so good.

## Loading Dependencies

The thing that took some head scratching is what to do about dependent native libraries. For example, compiling fortran, you produce a library that wants to link libgfortran.so. So how do you get this through JWS? I couldn't find a way to twiddle LD_LIBRARY_PATH, and I think it's impossible and a bad idea if it were possible.

First I tried statically linking the dependencies, which is apparently an unusual thing to do. In this scenario, the build artifact simply includes libgfortran, statically, and so there are no JWS dependencies to resolve. That probably would have worked, except that the libgfortran.a that comes with the default install is not relocatable (i.e. was not compiled using the "-fPIC" flag), so it can't be linked into a relocatable liibrary. So, to fix that, I would have to recompile libgfortran, bleah.

So, incredibly, what worked was to use System.load(). Just load the dependency before the dependent thing, and you're good.

## Interfacing

The Fortran code uses files as its main interface, and I didn't like that, so I set about to replace all the file access with native methods. We use [JNA](https://github.com/twall/jna), which works fine.

The easiest thing to replace is procedures that simply read an entire file into some data structure. These are all common anyway, so we just write an alternate populator procedure and call it prior to running the main procedure.

For output, I use a Callback and a common variable. That is, whenever I want to intercept output, I stuff it into the common variable (i.e. a static outvar) and call the callback. I think I could replace this by giving the callback an argument, but I had trouble with that, and quickly gave up. It would be cleaner, though.

For some kinds of input, the file is scanned piecewise. Read some input, do some work, read some more input. For this case, I keep the state of the scan in java (like how read keeps the file offset), and use another Callback to populate another common variable (a static invar this time). Again, I could probably do this more cleanly with a callback argument.

## Debugging

There is no good debugging method. When the native code crashes, there's no way to catch it. So, in production it will run in a separate process, with a babysitter to restart it if it dies. In development, I've tried gdb on the standalone fortran executable, without any success, but I'm not a gdb expert, so I may have been doing something wrong. So mainly I use the poor man's debugger, "print."
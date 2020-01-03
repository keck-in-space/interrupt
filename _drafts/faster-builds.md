---
title: "My Firmware Build is Slow. Now What?"
description:
  "Strategies for speeding up C/C++ compile times for embedded software projects using ccache, preprocessor optimizations, parallel execution, and other techniques."
tag: [better-firmware]
author: tyler
image: /img/using-conda/using-conda-cover.png
---

You are tasked with starting a new firmware project with an entirely new code base. Finally! You can ditch the old, archaic spaghetti code that builds in a "coffee break" number of minutes. You can start over with a clean slate. New and updated tools, faster build times, faster iteration cycle. You download an example app for FreeRTOS, and it compiles in a mere 10 seconds. You can quickly edit, build, flash, and test. Life is great. 

A few months in, the build becomes more of a slow job rather than an all out sprint. It takes a few minutes to build the project now. A few more months down the road, the project now builds in 10 minutes again, and we are back to where we started. 


<!-- excerpt start -->

Build times don't have to be slow. Many factors play a role, and we dive into which ones slow down the build tremendously, and also go over some easy wins which you can immediately contribute to your project and share with your teammates. The techniques discussed apply to most C/C++ projects, especially those using the GNU GCC compiler.

<!-- excerpt end -->

Hold on tight, as there is a lot to talk about.

_Like Interrupt? [Subscribe](http://eepurl.com/gpRedv) to get our latest posts
straight to your mailbox_

{:.no_toc}

## Table of Contents

<!-- prettier-ignore -->
* auto-gen TOC:
{:toc}

## Why Speed Up Firmware Build?

Waiting for a build to complete is a **waste of time**. It's as if someone decided to tie your hands behind your back for hours each day, ruining your productivity. You wouldn't want to work 

There are a number of "infrastructure" related items that aren't directly in the line of development that can slow a firmware project down. 

- Slow build times (talked about in this post)
- Painstaking and repetitive debugging ([automated debugging]({% post_url 2019-07-02-automate-debugging-with-gdb-python-api %}))
- Manually verifying the build succeeds for all targets ([continuous integration]({% post_url 2019-09-17-continuous-integration-for-firmware %}))
- Manual and rigorous testing ([unit testing]({% post_url 2019-09-17-continuous-integration-for-firmware %}))
- Poor developer environment management ([using an environment manager]({% post_url 2019-09-17-continuous-integration-for-firmware %}))
- And many more

I believe that slow build times is one of the most important pieces in the "infrastructure" realm and it can't be taken lightly. The build system needs constant supervision to keep everyone on the team as productive as possible. There's no use in paying an engineer (or ten) if most of the time they sit around waiting for the builds to complete. 

We write firmware that fits into 1MB of flash with 128KB of RAM. Our build times should be in the same scope, ideally under a minute. 

## Steps to Take

I don't love build systems. They are complex, unintuitive, and require a deep understanding of them to do things *right*. I believe the [GNU Make Manual](https://www.gnu.org/software/make/manual/make.html) is the only manual I've read cover to cover to date, and it was one of the best uses of time. When you throw in wildcard expressions, shelling out to Python, download files from other package managers, and custom binary packaging scripts, it could very easy become mess of scripts and rules. 



### The Basics

These are some up-front rules that one should take before trying to speed up the build using the other, more difficult methods. If one of these isn't satisfied, I'd start there.

- Don't settle for less than a quad-core CPU & solid-state disk on your development machine.
- Don't work on virtualized file systems or network drives. Use the host file system.
- Exclude your working directories from your your anti-virus software.
- Use the newest set of compilers and tools that you can reasonably use.

### Use A Faster Compiler

Some compilers are faster than others. 

### Parallel Compilation

The next thing to ensure about your build system is that you are building with multiple threads. To my knowledge, the only build system that *doesn't* support multiple threads is Keil. In their documentation, they [suggest using Make as an alternative](http://www.keil.com/support/man/docs/armcc/armcc_chr1359124201769.htm). 

When testing on my quad-core MacBook Pro, I got the following results when compiling with one thread and four threads respectively. 

### Optimizing Header Includes and Dependencies

#### Large Files = Problem

Generally, the larger the input file into the compiler is, the longer it will take to compile. If the compiler is fed a file with 10 lines, it might take 1 ms to compile, but with 5000 lines, it might take 100 ms. 

One of the steps that happens before a file is compiled is preprocessing. A preprocessor takes all `#include "file.h"` calls and replaces them with the *entire contents* of the file being included, and does this recursively. This means that a `.c` file as small as the following could wind up being 5000+ lines of code for the compiler!

```
#include "stm32f4xx.h"

int myfunc(void) {
  return 1;
}
```

We can check the contents of the preprocessed (post-processed) file by calling `gcc` with the `-E` flag.

```
$ arm-none-eabi-gcc -E -g -Os ... -c src/extra1.c > preprocessed.txt
```

It turns out that the header file `stm32f4xx.h` and all of *its* included header files amass into a file of 5398 lines. To put some build numbers to this result, let's test compiling this small file with and without the `stm32f4xx.h` header.

```
# Header included
$ time arm-none-eabi-gcc ... -c -o obj/extra1.o ./src/extra1.c
0.17s user 0.03s system 96% cpu 0.206 total

# Header excluded
$ time arm-none-eabi-gcc ... -c -o obj/extra1.o ./src/extra1.c
0.03s user 0.02s system 90% cpu 0.053 total
```

This shows that, just by including a single header to our otherwise 3 line `.c` file, the file went from being built in 50 ms to 200 ms.

#### Verifying Header Dependencies is *hard*

It's *incredibly* easy to accidentally add a nasty header dependency chain into your code base. Take for example the file `FreeRTOSConfig.h` in the AWS nRF52840 port, [linked here](https://github.com/aws/amazon-freertos/blob/master/vendors/nordic/boards/nrf52840-dk/aws_demos/config_files/FreeRTOSConfig.h). 

That file should have *no external dependencies*. It's also a file that you should be free to import anywhere and everywhere, as it's small and self contained. However, in this nRF52840 port, the `FreeRTOSConfig.h` file has a few extra imports. Here's the problem. 

- `FreeRTOSConfig.h` includes `app_util_platform.h`
- `app_util_platform.h` includes `mdk/compiler_abstraction.h`
- The chain eventually includes `cmsis/include/core_cm4.h`

and by this point, all is lost. The ideally tiny and self contained `FreeRTOS/list.c` file now includes 60 extra header files it didn't need, and compiling this file takes roughly 500 ms on my machine. 

So, before adding that extra header file (especially *to another header file*), think twice. 


#### Exploring Header Dependency Graphs




IWYU (Include What You Use)[^4]

Both minimizing header includes

### ccache - Compiler Cache

> NOTE: This is trivial to set up when using GCC + Make. It might be more difficult or impossible with other build systems.

ccache[^1] is a compiler cache that speeds up recompilation. It compares the input files and compilation flags to any previous inputs it has previously compiled, and if their is a match, it will pull the compiled object file from it's cache and provide that instead of recompiling. It is useful when switching branches often and in CI systems where subsequent builds are very similar. It's not uncommon to see 10x speedups. 

It can be used to compile individual files by prefixing the compiler.

```
$ ccache arm-none-eabi-gcc <args>
```

ccache can be installed using in the following ways for each operating system

- macOS - [Brew](https://formulae.brew.sh/formula/ccache) or [Conda](https://anaconda.org/conda-forge/ccache)
- Linux - [Apt](https://launchpad.net/ubuntu/bionic/+package/ccache) or [Conda](https://anaconda.org/conda-forge/ccache)
- Windows - [Github nagayasu-shinya/ccache-win64](https://github.com/nagayasu-shinya/ccache-win64)

I suggest **against** the commonly suggested way of symlinking your compilers to the `ccache` binary and instead to set up your build system in the following way

```
ARM_CC    = arm-none-eabi-gcc
CC_PREFIX = ccache
CC = $(CC_PREFIX) $(ARM_CC)

%.o : %.c
  $(CC) $(CFLAGS) -c -o $@ $<
```

This way, your environment remains simple and developers only need to install ccache with no further setup.

You can easily check whether ccache is set up properly using the following steps and verifying there are "cache hits".

```
$ make
...
$ ccache --zero-stats
$ make
... 
$ ccache --show-stats
cache directory                     /Users/tyler/.ccache
primary config                      /Users/tyler/.ccache/ccache.conf
secondary config      (readonly)    /usr/local/Cellar/ccache/3.7.3/etc/ccache.conf
stats updated                       Fri Jan  3 14:10:39 2020
cache hit (direct)                   540
cache hit (preprocessed)             140
cache miss                           303
cache hit rate                     69.18 %
called for link                       31
cleanups performed                     0
files in cache                       853
cache size                          76.1 MB
max cache size                       5.0 GB
```

> If you have a *very* large project, I've heard good things about icecc[^2] and sccache[^3], but these are generally for projects with thousands of large files and many developers.

### Pre-compiled headers

> NOTE: This is a GCC only feature.


## How to determine what is slow

### Run each command individually and profile

### Profiling Make

http://alangrow.com/blog/profiling-every-command-in-a-makefile

Use `time`

Don't use Make

`--output-sync=recurse` with GNU Make




{:.no_toc}

## Closing



{:.no_toc}

## References

[^1]: [ccache](https://ccache.dev/)
[^2]: [icecc - IceCream](https://github.com/icecc/icecream)
[^3]: [Mozilla sccache](https://github.com/mozilla/sccache)

---
published: true
title: 'Fluidsynth hack continued: Oboe driver'
---
## Updates since the last effort

[It was two weeks ago](https://dev.to/atsushieno/fluidsynth-20x-for-android-4j6b) that I finally got a practically working fluidsynth port with OpenSLES driver support. It was after big restructuring on the build system (building glib is a complicated task, and my build setup was messed due to Android NDK evolution). And by the time I got back successful builds there is already a new "better" audio driver Oboe (which is modern and ready for low latency than before).

And here you are, the Oboe driver is done: https://github.com/FluidSynth/fluidsynth/pull/464

This post is a handful of followups on the new Oboe driver implementation. There were many traps that drugged me on the ground...


## Mixing C++ and C

If you are not familiar with CMake, like me, you'd find it problematic on finding the right way to not let CMake ignore c++ sources. You need `CXX` in the `add_library` function call explicitly, which takes a list of languages to use for compilation. CMake silently ignores your `*.cxx` files, even if you clearly specify them in the source list.


## (historical) Oboe needs Android NDK r17, while Cerbero specified r16

It is now only a past issue, but Cerbero `android.config` specified Android NDK r16 explicitly. It was upgraded to r18 about 5 days ago.

NDK r16 lacks some required constant definition in AAudio API, which results in Oboe build failure. Therefore it is mandatory to use NDK r17 or later anyways.

I ended up to hack around Cerbero sources (again!) - this time it was trivial and that particular AAudio issue was resolved. But then another problem showed up...

## gnustl and libc++

[Since Android NDK r17](https://github.com/android-ndk/ndk/wiki/Changelog-r17), it's moving away from gnustl and started using libc++ in clang. And with r18 gcc and co. are totally gone, meaning that there is no gnustl anymore.

I have been using Cerbero build system to get a working glib build with all the dependencies. Unfortunately, their build script still specifies android-16 (JellyBean) in `config/cross-android-*.cbc`, and that triggered some problem. Let's see NDK r17 release notes...

> libc++ is now the default STL for CMake and standalone toolchains. If you manually selected a different STL, we strongly encourage you to move to libc++. Note that ndk-build still defaults to no STL. For more details, see this blog post.

also...

> The platform static libraries (libc.a, libm.a, etc.) have been updated.
>
> - All NDK platforms now contain a modern version of these static libraries. Previously they were all Gingerbread (perhaps even older) or Lollipop.
> - Prior NDKs could not use the static libraries with a modern NDK API level because of symbol collisions between libc.a and libandroid_support. This has been solved by removing libandroid_support for modern API levels. A side effect of this is that you must now target at least android-21 to use the static libraries, but these binaries will still work on older devices."

What makes things complicated is Cerbero android build which in general (namely `android/external/cerbero/packages/base-system-1.0.package`) specifies 'gnustl' as part of the dependencies. gnustl is the standard C++ STL implementation library from GCC. See the first quote from NDK r17 release notes. They now have switched to libc++ and "strongly encourage" to move to libc++...

What happens if we mix uses of both? Some "unresolved reference" errors due to lack of either at linking.

Unfortunately, gnustl is widely used among many recipes and that should be handled by whoever is involved in Cerbero well (which is hopefully not very complicated).

## solution: two shared libraries

Compared to the problem above, it is much easier to deal with, but another minor problem with Oboe was that it requires C++ build support. And that slightly messed up existing `CMakeLists.txt`.

To sort out those issues, I ended up to build a C-based oboe shared library `liboboe-c.so` and `libfluidsynth.so`.


- liboboe-c.so: based on C++, links libc++
- libfluidsynth.so: based on C, links gnustl

While I had been trying to get this working, there were some new commits in Cerbero and it had moved to NDK r18. But since gnustl is still in use, the symbol expectation mismatch still happens. I decided to keep this `liboboe-c.so` until it can go totally unnecessary...

(And in the future when Android NDK r19 lands, it will also migrate from gold linker to lld, which could bring in other kinds of problem.)

(I might add more lines later, but so far I'd publish this first to make it visible on what I've been doing regarding them.)

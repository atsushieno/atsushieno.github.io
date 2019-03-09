---
title: Fluidsynth for Android
date: 2018-03-10 00:00:00 Z
tags:
- android,
- audio,
- midi,
- xamarin
description: 
---

## Motivation

One of my unknown projects is Fluidsynth for Android. Since Android 6.0, it supports its own [MIDI API](https://developer.android.com/reference/android/media/midi/package-summary.html) as well as MIDI connection over BLE, so it became possible to connect MIDI devices to Android (if you would like to read my Japanese article, it's part of the book ["Android Masters!"](https://booth.pm/ja/items/126263)). A technically-looking-cool feature is that it supports virtual MIDI device services so that anyone can implement a MIDI device service that can provide either MIDI input devices or MIDI output devices (or both) that other apps can connect as clients and play them just like other operating systems (Windows, Mac, Linux, iOS...).

It was designed to make it capable to run a software MIDI synthesizer through the service, so why not porting any of the existing bits? I thought so, and ended up to bring [Fluidsynth](http://www.fluidsynth.org/) into Android land.

## Build system

It was not simple; first of all, it does not make sense if there is no sound output. Fluidsynth is a software synthesizer that supports various audio APIs but not for Android. For Android, AudioTrack and [OpenSL ES](https://developer.android.com/ndk/guides/audio/opensl/index.html) are the available choices (when I was implementing it; there was no AAudio nor Oboe). Fluidsynth has its audio abstraction layer in their [`drivers`](https://github.com/FluidSynth/fluidsynth/tree/master/src/drivers) source directory, so the only thing I had to do would be just to add another one there, right?

It was not that simple.

First, fluidsynth needed to be built with Android NDK toolchains. The original fluidsynth does not support Android builds. It would be particularly because of glib dependency; there is no intuitive way to build it for Android. glib is autotools-based and NDK did not play with it nicely.

... Wait. There are handful of glib-dependent apps and libraries that are known to work on Android. There is [Gimp](https://play.google.com/store/apps/details?id=org.dandroidmobile.xgimp), right? ... but it runs on top of some weird X server. Next, how about GStreamer? It is [known to support Android](https://gstreamer.freedesktop.org/modules/gst-android.html). How is it built? ... that's how I found [Cerbero](https://gstreamer.freedesktop.org/documentation/installing/building-from-source-using-cerbero.html) build system. It builds everything, including glib, in autotools manner, for Android as well as other targets.

All those dependency libraries can be built with "recipe", which customize each library build. cerbero's ultimate purpose would be to build GStreamer, but it can be anything if a recipe is added to the tree. And it was quite easy to add fluidsynth to the catalog.

I had to make some changes to support custom Android NDK setup, so my cerbero tree ended up to become an incompatible fork with the original tree, but I could build libfluidsynth.so for Android in the end.

Everything ended up to become this repo: https://github.com/atsushieno/android-fluidsynth/

## Android/OpenSLES implementation

Second, I had to add opensles implementation. The source structure was nice enough and I could easily add `fluid_opensles.c` to the tree. A minor problem was that there was almost only one [reference sample](https://bitbucket.org/victorlazzarini/android-audiotest/src) by Victor Lazzarini ([his blog](https://audioprograming.wordpress.com/category/android/) used to be publicly visible but now it's private...). Even samples from an NDK book published in Japanese were almost the same as these samples.

One another thing I had to implement was to support custom stream loader for SoundFont files. Fluidsynth only offered filename-based loader which simply used local file I/O. So I had to extend fluidsynth itself to accept custom SF loader - the abstraction API and Android assets-based implementation.

Since my application is written in C#, I added those extensions to my P/Invoke wrappers (that's only what I had to do - if I were using Java, I'd also have to add JNI entrypoints too...).

## NFluidsynth and FluidsynthMidiService

Fluidsynth is cross-platform and works fine on Linux, which makes it easier for me to develop C# binding. Fluidsynth itself works as a virtual MIDI synthesizer, but to bridge from the system's MIDI service entrypoints, we have to make fluidsynth functions callable and map from those entrypoints.

Therefore I made binding for  fluidsynth API using P/Invoke, released as https://github.com/atsushieno/nfluidsynth . And making it as a Xamarin.Android library is almost at zero cost. I only had to care about Android-specific extensions built only for Android.

At last, I created an  Android MIDI device service on top of this NFluidsynth.Android. To build such a service, we have to implement android.media.midi.MidiDeviceService. The entire API is super weird regarding inputs and outputs directions, but is not very difficult to implement.

During this attempts, I found that supporting Android MIDI API is almost no benefits. My managed-midi API is cross-platform, inspired by Web MIDI API, and it makes more sense to rather implement Android backend for it. Therefore I came up with two implementations; one for MidiDeviceService and another for my API.

They end up to become this repo: https://github.com/atsushieno/fluidsynth-midi-service

## CMake switches and android-fluidsynth port

All those works were done in earlier years, so it is kind of weird that I write this post in 2018. I thought there would have been more software MIDI synthesizers for Android being released, but it did not seem to happen. What I got instead was, a handful of inquiries about my android-fluidsynth port. Since the entire build system is quite tricky, most of those who attempted to build failed hard (I feel sorry for that...).

One of the reasons it is kept undocumented was that the current state is (to me) very temporary - when I started this project, it was [hosted at sourceforge](https://sourceforge.net/projects/fluidsynth/) and it was based on autotools. Now it is [hosted at Github](https://github.com/Fluidsynth/fluidsynth/)  and the build system moved to CMake.

CMake is problematic right now - cerbero technically supports CMake, but it never seemed to support Android. What cerbero does there is to specify custom toolchains (CC, LD, etc.). And when you override some toolchains specified by cmake toolchains config (there is one in Android NDK) ... it will detect inconsistent build specification and restarts the build again, without specified cmake options(!). That ends up to ignore my build options and the build fails. I still find no answer to solve this problem.

Graduating Cerbero is an option, and the option comes with a handful of choices - cerbero is in our build system because it resolves glib dependency problem. To eliminate cerbero, we have to find ways to resolve glib build issue.

[VolcanoMobile/fluidsynth-android](https://github.com/VolcanoMobile/fluidsynth-android) is an option; it removed all glib dependencies. This tree however became incompatible with the original tree, and I don't feel comfortable with that. However, [there is an effort](https://github.com/degill/fluidsynth-android-opensles) to incorporate my OpenSLES implementation into that diverged version. I'm thinking to switch the basis for binary build artifacts to the ones from this tree.

Other choices are [Google cdep](https://github.com/google/cdep/) (I don't like their basic idea of forking sources to make their own changes, which makes it merging origin unnecessarily messy), or incorporating [mono eglib](https://github.com/mono/eglib) instead of depending on glib. But so far I'm okay with the glib-less port above.

UPDATE (2018-03/12): eglib doesn't provide gthreads, so it's not an option.

## What's different from user's point of view

Fluidsynth automatically chooses the available options to build, for each platform. And opensles will be the default driver in my Android port.

There are some changes in SoundFont loader API and Android developers would like to use it (explained a bit, above). But other than that, there is no difference from the original source tree.

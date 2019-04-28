---
title: 'Fluidsynth for Android take 4: AMidi, SF3, Kotlin app, MidiReceivers'
date: 2019-04-28 15:03:00 Z
tags:
- midi
- audio
- fluidsynth
- android
- kotlin
---

## merged to master

I haven't worked on Fluidsynth for Android stuff until early March (because of that [fantasy music fes](https://atsushieno.github.io/2019/03/08/_.html)), but after that I became so free, so I resumed the task. And it is now [merged](https://github.com/FluidSynth/fluidsynth/pull/464) to master. I assume it will be part of the next major updates (2.1).

The most annoying part was that [I use cerbero](https://atsushieno.github.io/2018/03/10/fluidsynth-for-android.html) as the build system, and it is certainly annoying for others (moreover, my build script is `Makefile.android` which brings more annoyance for those who hate Makefiles!). In the end we provide the build script as a "document", stored at `doc/android` subdirectory.

In any case, building fluidsynth for Android is an annoying task, so I created [a FAQ issue](https://github.com/atsushieno/fluidsynth-midi-service-j/issues/12) and listed all what people want - prebuild binaries, link to `Dockerfile`, and link to design docs (which contains `bitrise.yml` - but be careful, a build can take like 40+ minutes).

There are some notable changes since [the last description](https://atsushieno.github.io/2018/11/15/_.html):

- Oboe driver is now built at C++ driver (fluidsynth itself takes C++ code for this), so that I don't have to provide liboboe-c anymore. (I still have it for [oboe-sharp](https://github.com/atsushieno/oboe-sharp/) project for Xamarin.Android, but I have never written any app with it.)
- Android asset font loader is now built as a separate library `libfuildsynth-assetloader.so` and it is built only via a build script. It is automatically built with my make script.


## exploring AMidi

While I was working on various build changes for the merging, Google announced Android Q preview, and it came with Native MIDI API. That sounded fascinating. What if I can avoid all those JNI bits for my Fluidsynth Android apps?

I thought I could make use of it within Fluidsynth and created [an enhancement issue](https://github.com/FluidSynth/fluidsynth/issues/520). But a few weeks later when I resumed work on it, I noticed that there was no entrypoint for MidiDeviceServices to connect to this Native MIDI API. Therefore the issue is closed now.

I shoot a question to [android-midi](https://groups.google.com/forum/#!forum/android-midi) Google groups asking if there is chance to use this Native MIDI stuff at service side, but there is so far no feedback.

An interesting discovery was that for platform-specific libraries like AMidi (Android Q or later), it is not appropriate to directly link the library and use the features. It is due to nature of Linux shared libraries. When you open a shared library using `dlopen()`, the dependencies are also loaded. It may be delayed with `RTLD_LAZY` flag, but if your code tries to call `dlsym()` that will result in non-existent shared library failure, then it will happen anyways (think about strong binding system like JNA, for example).

The same kind of problem occurs with Oboe. Oboe supports both AAudio and OpenSLES, but if liboboe tries to load symbols in AAudio it will immediately fail on earlier Android versions than Android P (regardless of whether the Java code checks version code). Actually, Oboe has a solution to it - it simply uses `dlopen()` and `dlsym()` to dynamically load without strong reference to `libaaudio.so`. It is a reasonable trick to avoid real world issue. I like it, and followed the way [in my AMidi experiment](https://github.com/atsushieno/fluidsynth-fork/blob/d92cbc5e8d644595d6ecad809330725f14ec8b05/src/drivers/fluid_android_amidi.c#L40).

## Building libsndfile for SoundFont3 support

There was an additional build annoyance that I found recently. Fluidsynth supports SoundFont v3 (compressed audio), but it requires on optional dependencies - libsndfile. And for libsndfile, support for compressed audio formats is optional - it requires libogg, libvorbis and libflac. FLAC is totally unnecessary, but due to build configuration complexity, libsndfile requires all of them as mandatory dependencies.

So - I had to build all of them. Fortunately, all of libogg, libvorbis and flac are already supported in Cerbero. The only missing piece was libsndfile. After some attempts to build it from sources, I just ended up to add a new Cerbero recipe for libsndfile... it was only a few minutes task (compared to hours spent on how to manually build and integrate it in my build scripts...).

Now my cerbero builds are based on my own fork again, but so far I am still alive with it - at least there is a Dockerfile now. It wouldn't need significant updates, at least for a while (months)...

There is a caution regarding SoundFont3 in Android apps - SF3 files are better for reducing your apk size, but you should specify `android { aaptOptions { noCompress 'sf3' } }` in your `build.gradle`.


## fluidsynth-midi-device-service-j project

Now that I got a working libfluidsynth with appropriate asset SF loader, I could start working on actual Android apps that makes use of it. Now [fluidsynth-midi-service-j](https://github.com/atsushieno/fluidsynth-midi-service-j) app project works as a functional MidiDeviceService. It comes with some configuration UI for creating fluidsynth drivers.

The new app is based on Kotlin (and Java, if JNA binding is counted so), with my partial port of [managed-midi](https://github.com/atsushieno/managed-midi) to Kotlin. I used to have Xamarin.Android app [fluidsynth-midi-service](https://github.com/atsushieno/fluidsynth-midi-service), but it became non-maintainable due to long-term [xamarin-android](https://github.com/xamarin/xamarin-android/) Linux build breakage over months. It may come back if xamarin-android recovers its build state, but I expected that for months and unfortunately it never happened.

I found it interesting that audio sampling rate is better set to the best native sampling rate that AudioManager API suggests, than setting lower sample rate so that the fluidsynth could perform the synthesis work at less cost. It seems that sample rate conversion could impact more, at least on modern emulator (P/Q) and device (Pixel 2).


## MidiDeviceService.getInputPortReceivers()

One annoyance I found was that there is no distinct step for MidiDeviceService to provide information on ports, to return the actual port implementations, and to open the ports. Once you return MidiReceiver for fluidsynth output ports (note that as [the official Android NDK docs admits](https://developer.android.com/preview/features/midi), their way to denote "input" and "output" are weird - therefore "receiver" for "output" ports).

Whenever `FluidsynthMidiDeviceService` returns `FluidsynthMidiReceiver` instances, it has to return a list of "already opened" MidiReceivers. Otherwise the next code that we can acknowledge is `onSend()` method, which is not supposed to take long long time (note that initializing fluidsynth involves loading sound fonts, which takes a few *seconds*). Let's see what typical MIDI APIs do instead:

- return list of output ports (e.g. `navigator.requestMIDIAccess()`, `MIDIAccess.outputs`)
- open a specific port (e.g. `MIDIPort.open()`)
- send MIDI messages (e.g. `MIDIPort.send()`)

If there were `MidiReceiver.open()` then it could have been non-issue.

In the end, FluidsynthMidiReceiver initializes itself as almost opened state. fluidsynth-midi-service-j app comes with `FluidR3Mono_GM.sf3` as the default soundfont, and it would take like 100+ MB RAM (uncompressed) at load time. It happens at any time when any MIDI client queries MIDI ports via the MidiDeviceService. I once thought of opening 4 ports at a time (just like timidity++ does on my Ubuntu desktop), but reminded of that because of this problem. One is enough.
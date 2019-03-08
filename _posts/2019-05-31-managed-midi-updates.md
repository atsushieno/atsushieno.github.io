---
published: true
title: managed-midi updates
description: null
tags: MIDI mono xamarin
---

Back in January, I wrote an introduction [blog post](https://dev.to/atsushieno/managed-midi-the-truly-cross-platform-net-midi-api-56hk) about [managed-midi](https://github.com/atsushieno/managed-midi). It's been 4 months since then, and there are couple of updates, so it's nice to sum them up.

### MIDI input supported

I have a few MIDI input devices now, especially Seaboard Block is cool (it's quite useless for my actual use though :p).

Maybe you think I'm just showing off my cool toy, but it's slightly more than that. Seaboard Block is based on BLE MIDI, which used to be supported only on iOS. Android 6.0 supported BLE MIDI as part of its Android MIDI API too. But Linux had no chance to use it until fairly recent [Bluez and Linux kernel got support for that](https://blog.felipetonello.com/2017/01/13/midi-over-bluetooth-low-energy-on-linux-finally-accepted/). I'm on Ubuntu 17.10, so I also needed to switch to [some patched build](https://bugs.launchpad.net/ubuntu/+source/bluez/+bug/1713017).

Anyhow, I got some reason to work on MIDI input support in managed-midi and it's done there. I also have M-AUDIO KeyStation Mini 32, but those two devices give totally different input messages... what Seaboard brings in is [MPE](http://expressiveness.org/2015/04/24/midi-specifications-for-multidimensional-polyphonic-expression-mpe) - for each keypress it sends pitchbend, PAf etc. accordingly to the specification, which is a lot of information (like accelerometer multi-axes inputs). It will be fun once I can think of apps and support nicer handling of those MPE messages.

### netstandard2 based NuGet packaging

I don't care much about nuget packaging, but it is of some goodness to spread use of this library (which, well, again, I don't care...). As explained in the [previous post](https://dev.to/atsushieno/managed-midi-the-truly-cross-platform-net-midi-api-56hk), Xamarin.Mac Full project is blocked and it's unchanged. Probably few people cares, but I do care - Xamarin.Mac full is the only profile that I can get Xwt working, and therefore my xwt-based [xmdsp](https://github.com/atsushieno/xmdsp) working) too.

Anyhow Xamarin, which brings the primary reason for me to "package" managed-midi, has migrated to .NET Standard 2.0 world from PCL-based world. So I moved forward too.

There should be .NET Core version of this library too (unlike the netstandard2.0 library, it will contain ALSA support etc.), hopefully in the near future.

### Precise time controller

managed-midi used to be a dumb, simple MIDI player that never dealt with latency. It was somehow significant and not ignorable, and it was [reported a while ago](https://github.com/atsushieno/managed-midi/issues/14). It's been a big concern for me too.

managed-midi comes with a timer class called MidiTimeManager. Usually we use the default timer, which waits for the specified time span (simple as Task.Delay()), while we can alternatively use virtual time manager. It is just like TestScheduler in Rx.NET. You don't want to wait forever when running unit tests.

So, technically timers can do more than simple and stupid wait. And any MIDI libraries would be dealing with that. I was lazy to actually implement it until very recently.

While it is still impossible to test this improvements (unless I actually play some  songs and measure the actual play time within the unit tests...), I use my xmdsp MIDI player to examine the latest accuracy. I used to author quite complicated song files (either copies or original songs) that I use for dogfooding.

### What's next

I have no idea TBH. My target is shifting to either raw audio based stuff (such as android-fluidsynth) and some MIDI composition environment. But since managed-midi is foundation for all my existing apps, it will keep going.
---
published: true
---
# managed-midi: the truly cross platform .NET MIDI API

I have a (surprisingly) long term project called ["managed-midi"](https://github.com/atsushieno/managed-midi) which is to offer cross platform C#/.NET MIDI access API. My "cross platform" here is not a marketing term; it targets Linux (ALSA), macOS (CoreMIDI) and Windows (WinMM), along with Android, iOS and UWP.

Actually, my managed-midi project is not about cross-platform goodness. It's more of a structured music composition as well as messaging foundation. This is my primary motivation to develop this library.

But I'm going to discuss cross-platform part.

Wasn't there any existing effort to provide Midi access API? The only project I can think of is [NAudio](https://github.com/naudio/NAudio), which only cares about Windows. My primary desktop is GNOME on Linux, so it's no-go. This is a typical huge problem in C# developer ecosystem that they only care about Windows.

Audio and music libraries are always like that, and few people have interest in cross-platform AND platform specific development. This development situation is similar to developing "bait-and-switch" PCLs, but like I generally don't care about Mac and UWP, people don't care about platforms they don't use.

It is kind of ironic that MIDI features are categorized within very platform specific API groups, whereas MIDI itself is designed to be device independent.

The situation in C++ world is however changing. VST is now available on Linux too (I mean, the original library from Steinberg). We are seeing new-generation DAWs working on Linux (Renoise, Bitwig Studio, Tracktion WaveForm etc.).

## wrapping around cross-platform MIDI API

Back in 2009 when I started launching this project among other small projects I had in mind (when this project didn't even have a name and repo), I found [PortMIDI](http://portmedia.sourceforge.net/), which is a cross-platform MIDI library that supports Windows, Mac and Linux, and I found it cool. I didn't want to deal with platform-specific MIDI APIs, especially whatever I was not familiar with. Instead, I wrote a P/Invoke wrapper and OO-style API around it. I think my idea was good (I still kind of) - it is important to quickly get what we need, right? Those smart native developers dealt with the platform specifics, and I made use of the outcome.

That was the beginning of this library. I just wanted an SMF parser and a MIDI player that plays MIDI songs that I can compose using my music macro language compiler [mugene](https://github.com/atsushieno/mugene). There wasn't even appropriate abstraction.

It was however somewhat messy to get apps that are based on this library working. My primary .NET runtime has been always Mono, and I was primarily on Windows at that time. I couldn't care much about Linux. Since portmidi is a pure third-pary library it always required binary shared libraries. portmidi was painful to me on Windows since it required JNI header files to build. And it has to be offered in 32bit for Mono (which supported only 32-bit x86 on Windows at that time) and 32bit binary didn't work on .NET Framework for some reason. I was not familiar with them. I almost dumped the idea of offering binaries by myself.

A few years later, I found [RtMidi](https://github.com/thestk/rtmidi) and it was simpler. Unfortunately it was C++ only, so I wrote C wrapper for that (which I [ended up to contribute](https://github.com/thestk/rtmidi/commit/a5c375c7) to the project). It had the same issue as portmidi too but didn't require JNI stuff, so building binaries was not so painful. Moreover, the development was active (at that moment).

There was still no proper abstraction, but it was implicitly done. The abstraction was done only as `MidiPlayer` class (then I had `PortMidiPlayer` and `RtMidiPlayer`). I didn't care much about that.

## MIDI API abstraction

The primary reason I was developing managed-midi was to write usable MIDI application. So it does not have to be C#. There has been another effort to bring MIDI to cross-platform world - Web MIDI API. It is expected to be working on Web browsers, but right now it is only Chrome which supports it.

Web MIDI support in early days was not actually great for Linux software synthesizers (Chrome supported only Mac at first, and software synthesizers need special permission to be enabled), but its API structure was informative. It gave me the basic ideas on what we can/should provide in any platform and what not.

Therefore, I started refactoring the entire API structure. What's good with Web MIDI API was that it is kind of OO-style and closer to C#, compared to former C libraries.

Thus, current API structure is like:

- `IMidiAccess`
- `IMidiPortDetails`
- `IMidiPort`
  - `IMidiInput`
  - `IMidiOutput`

The MIDI messages are represented just as byte arrays. It is same as Web MIDI and RtMidi (portmidi was different, had its own message type).

## Implementing direct access to native API

Even with RtMidi, I was reluctant to build those libraries for Windows (I already switched my primary desktop to Linux) and Mac (I hate Apple's attitude against Adobe Flash, dumped my iPhone and switched to Android because of that), especially since I had to provide my custom library that included C API.

After abstracting the API, I started to think that I could implement platform-specific implementation and switch the default implementation per platform so that I didn't have to bother to build native binaries anymore.

Though the first problem I was faced was no P/Invoke availability in PCLs. I quickly thought that it's not suitable for PCLs. But there is another tech trend in .NET: PCL with "bait and switch" technology that makes it possible to provide a common API with platform-specific implementation. Technically this problem seemed resolved.

The first native implementation target was Windows. Windows at this state means: Windows desktop and UWP (I don't care about WinRT, there is no official MIDI API and those users are used to be limited their ability). As of that time, what I wanted was rather getting my visual MIDI player [xmdsp](https://github.com/atsushieno/xmdsp) seamlessly working on Windows and Mac as well as Linux, and it was on top of [xwt](https://github.com/mono/xwt) which is based on WPF and thus desktop .NET Framework. Therefore WinMM matters. WinMM was easy. UWP is more organized API and wrapping around it was easy, but since I don't have any managed-midi apps it is totally untested.

CoreMidi was different. It did not even start with desktop. What I actually thought was to get managed-midi working on Android (and it was not even for Android MIDI API - it was for my [fluidsynth-midi-service](https://github.com/atsushieno/fluidsynth-midi-service/) project, but it's going to be too long to discuss that here). Anyhow mobiles got in sight at that time. Android MIDI API is shitty with the class names, but wrapping around it was easy.

While I wrote CoreMidi implementation for iOS, I didn't have any managed-midi apps that work on iOS, so I didn't even try it. Even on Mac, I tried only once when Xamarin.Mac started working fine with Xwt with CoreMidi (there was complicated library resolution bug that I had to wait for fixes in 2017). API wise, CoreMidi was a bit more complicated than others due to their concept of endpoints.

And... finally ALSA. I didn't want to work on ALSA because rtmidi just works, building rtmidi was always easy, and ALSA is complicated. But copying librtmidi.so everywhere is messy (like [this issue](https://github.com/atsushieno/managed-midi/issues/8)) and as long as I am dependent on rtmidi I could not resolve some weird [issues like this](https://github.com/atsushieno/managed-midi/issues/1), so I ended up to learn about ALSA and implemented it as [alsa-sharp](https://github.com/atsushieno/alsa-sharp/).

## Fundamental problem with the "bait and switch" trick in .NET Standard

At this state, managed-midi lacks a lot of features like input support everywhere (surprisingly? I didn't need it yet) and device connection state detector (which is part of Web MIDI API so it should be implemented too). I only cared about output (my primary goal is to get a MIDI player). Device detection is complicated on ALSA as it has to be done outside ALSA, and probably WinMM which doesn't provide the feature too.

NuGet packaging is another bit problem - while I intended to build a cross-platform library, it does not take shape of "bait-and-switch" PCL or .NET Standard library. Here is why: Xamarin.Mac full profile requires its own `ProjectTypeGuids`, which means it is basically different from .NET Framework desktop profile. They have to be different. And I cannot depend on Xamarin.Mac when I'm working on this library because I'm not on Mac (even if I had Mac, no one should be required to work on Mac anyways) and the entire desktop library cannot be Xamarin.Mac specific (see [this post](https://medium.com/@donblas/xamarin-mac-and-netstandard2-708a06890302) for the PCL/NuGet target moniker).

Therefore managed-midi has two different assemblies for the same netstandard moniker (net461). That makes it impossible to package one of either. Since there is no reason to prefer Xamarin.Mac specific assembly for net461, it is basically ignored.

This is something .NET Standard designers should resolve.

----

This is an import from https://dev.to/atsushieno/managed-midi-the-truly-cross-platform-net-midi-api-56hk

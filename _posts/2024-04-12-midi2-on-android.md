
---
published: true
title: "Building MIDI 2.0 Ecosystems on Android"
date: 2024-04-12 15:30:00 +09:00
tags:
- Android
- MIDI2
- MIDI-CI
- MIDI
- JUCE
---

# Building MIDI 2.0 Ecosystems on Android

These days I have been exploring the new MIDI 2.0 opportunity on Android. There has been a lot of potential spaces to make things happen, and I would break various aspects of audio and MIDI 2.0 application features down to small pieces.

## New in Android 15: MidiUmpDeviceService

For Android music app developers like me, Android 15 is a big update! It brought in MIDI 2.0 to "everyone" as [`MidiUmpDeviceService`](https://developer.android.com/reference/android/media/midi/MidiUmpDeviceService), i.e. support for non-USB MIDI 2.0 devices.

Back in Android 13, we were able to use MIDI 2.0 USB devices on Android, and [`MidiManager` class](https://developer.android.com/reference/android/media/midi/MidiManager) connected our MIDI client apps to USB MIDI 2.0 devices. USB MIDI 2.0 is a tailored USB class that supports "UMP" which is the new MIDI 2.0 transport which is not the traditional MIDI 1.0 bytestream transport. Android 13 came only with USB MIDI 2.0 support, and it left non-USB MIDI devices behind MIDI 1.0 legacy.

Around that time, I have created [an issue at Google issue tracker](https://issuetracker.google.com/issues/227690391), asking for some way to create software MIDI 2.0 devices. In the end we needed a new platform feature to achieve that.

There has been confused years on how MIDI 2.0 devices would be supported at platform layer. Only macOS was a platform player that supported MIDI 2.0 devices natively. The game has changed in June, 2023 though, when MMA/AMEI published the totally revamped MIDI 2.0 specification documents. Protocol Negotiation is gone and platforms are getting clearer that we would need dedicated ports for MIDI 2.0 UMP ports alongside traditional MIDI 1.0 ports.

`MidiUmpDeviceService` is the only class for the latest MIDI 2.0 additions in Android 15, and it is basically [`MidiDeviceService`](https://developer.android.com/reference/android/media/midi/MidiDeviceService) for UMP. There are surprisingly small difference between `MidiDeviceService` and `MidiUmpDeviceService` - both accepts `ByteArray` (in Kotlin) as the input and output messages. Their input receivers and output senders are created almost the same way. The differences are trivial:

- UMP ports are bidirectional so that you only describe `<port>` in MIDI device descriptor XML
- for `<service>` element in `AndroidManifest.xml`, we need `<property>` element instead of `<meta-data>` element.

API wise, UMP ports being bidirectional does not affect how we deal with MIDI input and output streams in terms of `MidiSender` and `MidiReceiver`.

I do not create any example project to demonstrate the feature - instead I have various use cases on my existing projects that already supported `MidiDeviceService`. In other words, they are already *real world* usages:

- [Audio Plugins for Android](https://github.com/atsushieno/aap-core/) already [integrates](https://github.com/atsushieno/aap-core/blob/0dc07da5aefbd1e3999fb8221735f185cb341822/androidaudioplugin-midi-device-service/src/main/java/org/androidaudioplugin/midideviceservice/StandaloneAudioPluginMidiDeviceService.kt#L14) MidiUmpDeviceService so that any instrument plugin could behave as a UMP MIDI device, just like it used to for MIDI 1.0 device. [aap-lv2-mda](https://github.com/atsushieno/aap-lv2-mda), [aap-lv2-sfizz](https://github.com/atsushieno/aap-lv2-sfizz), and [aap-juce-hera](https://github.com/atsushieno/aap-juce-hera) already have these changes applied.
- [Resident MIDI Keyboard](https://github.com/atsushieno/resident-midi-keyboard) works as a virtual MIDI keyboard that can send MIDI 1.0 events to other MIDI apps, and now it could also work as a MIDI 2.0 keyboard that is recognized by Android platform and usable with *any* UMP device.
- ktmidi-ci-tool in [ktmidi](https://github.com/atsushieno/ktmidi) handles MIDI-CI messages, and provides virtual MIDI I/O ports so that it could work as a virtual MIDI-CI service too. It now works as a virtual MIDI 2.0 device and can handle groups too.

Note that they are available only as sources. Android 15 is still at Developer Preview state whose API is not stable yet. We cannot publish any app that uses the preview API to Google Play Store yet. They are available via [AAP APK Installer](https://github.com/atsushieno/android-ci-package-installer) instead (you need GitHub token to get those non-release GitHub Actions builds) - if it works (unconfirmed).

## Dealing with UMP

`MidiUmpDeviceService` is just an aspect of MIDI 2.0 apps. Like `MidiDeviceService` for MIDI 1.0 bytestream, it only provides byte stream of UMPs. It was alright for MIDI 1.0 as it is popular enough and everyone could handle the bytestream inputs and outputs. It is probably not the case for UMP - people have little idea about UMP and would not know how to deal with them. There is nothing to be afraid though - they are still just arrays of integers that you can always parse and/or generate. We just need to understand the new format.

MIDI 2.0 UMP (Universal MIDI Packet) is a totally different representation of MIDI events that is extended to 32-bit, 64-bit, and 128-bit integers (compared to 7-bit MIDI 1.0 bytes). For example, MIDI 1.0 Note On message looked like this (in hexadecimal):

> | 90 | 30 | 78 | 

MIDI 2.0 UMP Note On message looks like:

> | 40 | 90 | 30 | 00 | F8 | 00 | 00 | 00 |

If you want to write code in Kotlin to deal with UMPs, [ktmidi](https://github.com/atsushieno/ktmidi/) has been always there to support MIDI 2.0 UMP development. I also have C library called [cmidi2](https://github.com/atsushieno/cmidi2) in case you prefer coding in C. If you write C++ code, there have been more options these days:

- [libremidi](https://github.com/jcelerier/libremidi) is a cross-platform MIDI 1.0/2.0 API that can handle platform MIDI accesses.
- JUCE has been providing its preliminary [UMP support API](https://github.com/juce-framework/JUCE/tree/master/modules/juce_audio_basics/midi/ump) in `juce_audio_basics` module.
- [ni-midi2](https://github.com/midi2-dev/ni-midi2) offers some API to handle UMP and MIDI-CI messages.

## Dealing with "SMF2"

As of writing this in April 2024, there is no corresponding format to SMF for MIDI 2.0 UMPs - there is a MIDI2 specification for "MIDI Clip File" which is basically a single track of a UMP sequence. The thing is that, the concept of a song file has changed through the decades and we have some intermediate concepts between a MIDI event and a "song" - SMF1 consists of a sequence of tracks, and a track consists of a sequence of MIDI events. But most modern DAW tracks have "clips" i.e. a group of MIDI events, and multiple clips could be reused, and/or overlapped each other. So, MMA has been taking longer time to finalize the "song" file format which is going to be a bundled collection of those clips and miscellaneous resources.

So far, SMF2 "Clip File" is already a finalized specification so that we could expect to be usable. SMF2 is basically a UMP sequence with a bit of non-UMP header chunk, followed by UMP-based header chunk (i.e. you can use standard UMP parser to parse the rest), with a prefixed "delta clockstamp" (like SMF1 did with 7-bit variant length), which is also a UMP packet. ktmidi supports it too. ktmidi has been supporting a feature parity of SMF (song file) for UMP in its own format too, so that you can store them for playback.

## Dealing with MIDI-CI and UMP Endpoints

MIDI-CI is loudly advertised as part of MIDI 2.0. To support MIDI 2.0, however, **it is not required at all**. You can leverage full power of MIDI 2.0 UMP without needing any MIDI-CI stuff at all, and it's totally fine. MIDI-CI brings in slight benefits to your MIDI apps, and most of other MIDI apps do not really support MIDI-CI features.

So, what is MIDI-CI? MIDI-CI makes use of bidirectional messaging channels between two MIDI devices so that they can "understand each other" and then provide advanced features based on the established session. With MIDI 1.0, everything was uni-directional like UDP. MIDI 2.0 transport made it like HTTP/3, by combining two uni-directional message channels together as one. Note that bidirectional messaging itself is NOT what MIDI-CI achieves. MIDI-CI is built on top of such messengers.

If you are still interested in MIDI-CI, then you can use ktmidi-ci if you prefer writing code in Kotlin, or [`juce_midi_ci` module](https://docs.juce.com/master/namespacemidi__ci.html) in JUCE 7.0.9 (or later), or [ni-midi2](https://github.com/midi2-dev/ni-midi2) to some extent, if you prefer C++ and NDK. I [wrote about ktmidi-ci here](https://atsushieno.github.io/2024/01/26/ktmidi-ci-tool-released.html) a few months ago. The API has greatly improved in the latest v0.8.0 release and became much nicer to deal with as a library.

MIDI-CI brings in three notable features: **Profile Configuration**, **Property Exchange**, and **Process Inquiry**. There used to be another component called Protocol Negotiation, which used to be essential to establish MIDI 2.0 connection and protocol promotion, but it's totally gone in June 2023 updates. MIDI-CI itself is on top of MIDI 1.0-compatible SysEx messages and no need to use other MIDI 2.0 bits.

Profile Configuration matters if (and only if) your application provides additional features to certain MIDI devices, based on "profiles". A profile indicates that a supporting MIDI device provides whatever feature it expects by its profile specification. For example, there is "GM2 single channel profile" that is to declare that the MIDI device channel supports the controllers along with what GM2 specifies. Profiles are not strongly typed and less bound to programmable APIs yet. Neither ktmidi nor `juce_midi_ci` offers Profile related features other than on-off switches.

Property Exchange matters if your application wants to control MIDI devices that provides something that goes beyond standard MIDI controllers such as CCs or NRPNs (Assignable Controllers in MIDI 2.0 wording), and still not too specific as device-specific SysEx. Properties are typically represented as JSON, so, it is typically used for something that works in non-realtime way. Since there is no MIDI 2.0 apps that offer MIDI-CI Properties on Android yet, it would be either a controller app to control USB MIDI 2.0 devices (if any), or you build a `MidiUmpDeviceService` which you intercept its incoming MIDI messages and handle MIDI-CI property messages. ktmidi-ci-tool is a reference implementation that does intercept those MIDI-CI messages.

There are slightly better ideas for proper use of Property Exchange, documented as MMA/AMEI defined properties, but I would skip it for now.

Process Inquiry matters if you want to have your virtual MIDI device expose current state of controllers, or you want to get your connected MIDI devices dump all of its status (note status/controllers/pitch bend etc.), IF it supports Process Inquiry (MIDI Message Report).

They are all supported in ktmidi. If you ignore Process Inquiry (you most likely at this state), they are available with `juce_midi_ci` too.

Lastly, in case you want to build somewhat complex UMP device and have to deal with UMP endpoint configuration and messaging, the latest ktmidi v0.8.0 added a new [`UmpEndpoint` class](https://atsushieno.github.io/ktmidi/ktmidi/dev.atsushieno.ktmidi.umpdevice/-ump-endpoint/index.html) that can manage Function Blocks and can handle incoming UMP stream message requests and send replies. It can also be used as a UMP stream client so that you can easily handle the connected UMP endpoint details (endpoint info, device identity, function blocks, etc.).

## Bridging MIDI 2.0 devices over MIDI 1.0 ecosystem

Now we have our MIDI devices ready for MIDI 2.0 clients using `MidiUmpDeviceService`, and we can connect to MIDI 2.0 services using `MidiManager` since Android 13. UMP ports have their own ecosystem along with MIDI 1.0 ports.

But most of those DAWs are still based on MIDI 1.0. Can we use our UMP devices with those MIDI 1.0 DAWs? The simplest answer is "no", but if you provide some way e.g. `MidiDeviceService` that are supposed to receive MIDI 1.0 inputs and then your device code translates its 1.0 inputs to UMP inputs and forward to another UMP based device, as well as receive its UMP outputs and down-translate to MIDI1.

MIDI 2.0 "protocol" is designed to be easily translatable to MIDI 1.0 "protocol", and we could use this UMP translator trick to generate MIDI 2.0 inputs from MIDI 1.0 inputs, and output MIDI 1.0 bytes back when the UMP device is done with UMP outputs. ktmidi offers `UmpTranslator` class that implements normative conversion UMP-to-and-from-MIDI1 streams.

## Access to MIDI 2.0 devices in Kotlin Multiplatform

If you build Kotlin Multiplatform apps or planning to migrate your Android app to KMP,  and want to have MIDI 2.0 features, the latest ktmidi v0.8.0 added support for UMP ports in `MidiAccess`. The latest `MidiUmpDeviceService` was the great motivation for me to work on the stack. On Android it just uses the not-that-new API added in Android 13.

ktmidi v0.8.0 also added `CoreMidiAccess` (which is actually either `TraditionalCoreMidiAccess` or `UmpMidiAccess`) that you can use in your Kotlin Native apps. So it would be realistic to use KMP for cross-platform MIDI 2.0 apps. It still lacks ALSA support (and Windows is out of course until [Windows MIDI Services](https://github.com/microsoft/MIDI/) became ready for general consumption [around the end of year](https://www.youtube.com/watch?v=4dS0hWQxyFA)), but hopefully soon.


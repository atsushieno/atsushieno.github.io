---
published: true
title: "AAP 2023 Year in Review"
date: 2024-01-28 01:00:00 +09:00
tags:
- AAP
- Android
- Audio
- MIDI
- JUCE
---

# AAP 2023 Year in Review

Usually I write this "AAP Year in Review" blog post around the end of the year. I have been privately busy around the end of last year, and also been concentrating on ktmidi-ci-tool I just blogged a few days ago. Now that ktmidi-ci-tool is "released" to some shape, it's time to come back to AAP. This is the first thing to do in AAP world.

## Project management work

I began 2023 with rebranding the project: it is **renamed to Audio Plugin For Android**, renamed from Android Audio Plugin Framework. It was to avoid possible trademark confusion.

I have published [**FAQ**](https://github.com/atsushieno/aap-core/wiki/FAQ) and [**Participating**](https://github.com/atsushieno/aap-core/wiki/Participating) wiki pages. The former one worked whenever I want to forward any question to.

Now we have [**AAP APK Installer**](https://github.com/atsushieno/android-ci-package-installer) application that lets you install the AAP hosts and plugins that are registered in the app, so that those plugin project APKs can be easily installed by this installer. The behavior is still unstable (especially access to GitHub Actions artifacts without PAT seems quite lame), but our overall installation experience became better than nothing.


![AAP APK Installer](https://i.imgur.com/Ajz5dav.png)


## Remote plugin UI support

The biggest challenge in AAP would be support for **in-process plugin UI**. Before 2023, we planned to primarily use Web UI that can be instantiated by host "remotely". It was then supposed to communicate with the plugin itself through Binder. In 2023, we began with experimental in-process GUI using System Alert Window, and it worked to some extent. During the development process, I noticed that Android 31 has added a new remote UI foundation called **SurfaceControlViewHost** and found it useful enough to achieve in-process plugin UI. Now AAP's in-process UI can be instantiated "remotely on a DAW" (I will mention aap-juce-helio later) with some glitch (like it cannot be moved; UI adjustment needed on app code in juce_gui_basic):

![aap-juce-helio show UI before](https://i.imgur.com/a5KmvSU.png)
![aap-juce-helio show UI after](https://i.imgur.com/nuOQTef.png)

This default plugin UI is based on Jetpack Compose, which is apparently not part of this JUCE-based DAW. We still haven't figured out how we can integrate JUCE plugin UI yet, but it won't be hard (at least not harder than managing it as a cross-process plugin UI)

## New MIDI extension and MIDI message mappings

The GUI feature above is backed by a handful of improvements in the AAP framework foundation. We used to have `aap-midi2` extension that only managed MIDI protocols. What I figured this last year was that there is not a lot of JUCE audio plugins that treats MIDI Program Change message as a preset changer, while we could easily achieve. This led me to implement default mappings for program changes and parameter changes (which was already sent over MIDI2 channel [in 2022](https://atsushieno.github.io/2022/12/24/aap-2022-year-in-review.html)) to CCs, Assignable Controllers (NRPNs), and per-note controllers, in the name of (simply) `midi` extension. After this change, an arbitrary MIDI client can connect to AAP MidiDeviceService and send a program change message, and that results in the preset change.

![AAP MIDI mappings](https://i.imgur.com/fasCf6o.png)

## New plugin manager app, resident-midi-keyboard, and aap-juce-simple-host

One of the annoyance in AAP until last year was that the default main launcher activity was not great and it demonstrated the audio effector feature only by pre-defined wav file in fixed sample rate, without appropriate WAV loader. Also the MIDI phrase it previewed was fixed. It needed complete overhaul.

The first simplification I could achieve was removal of any MidiDeviceService testing feature on the UI. They could be done totally outside the activity. While exploring the GUI support, I figured that System Alert Window can be useful for a "remote MIDI keyboard" app, and the idea came to realize as a standalone MIDI keyboard app: [**resident-midi-keyboard**](https://github.com/atsushieno/resident-midi-keyboard). It ended up to become the first Android app on Google Play Store "by the AAP Project". I [blogged more details about the app](https://atsushieno.github.io/2023/08/07/rmk.html) last uear.

![resident-midi-keyboard over FL Mobile](https://github.com/atsushieno/resident-midi-keyboard/blob/main/docs/images/kvr/rmk-on-flmobile.jpg?raw=true)

The next one was the biggest work: On the plugin default UI (`PluginDetails` Composable screen) I rewrote all the backend part (was written in Kotlin), in C++. Now we make use of [Tracktion/choc](https://github.com/Tracktion/choc/) to process sampled audio files, as well as designed and implemented super-simple `AudioGraph` processor (after evaluating some existing solution...) in brand-new `androidaudioplugin-manager` module. We can play MIDI notes, send parameter changes in more compact UI, and choose presets now. Web UI and Native UI can also send those events as long as the controllers are implemented and on the UI.

![the new plugin manager UI](https://i.imgur.com/OXc6J7R.png)

On another aspect, I have been using aap-juce-plugin-host, which is an Android port of JUCE AudioPluginHost. Since it involves a lot of extra code and is designed for desktop, it is always cumbersome and not appropriate for mobile uses. I ended up creating another simplified plugin hosting app, [`aap-juce-simple-host`](https://github.com/atsushieno/aap-juce-simple-host), which I can use with much less hassle.


![aap-juce-simple-host](https://i.imgur.com/N1WUGbv.png)


## Serious realtime processing for plugin extensions

2023 was a year that I started working seriously on realtime processing at various levels. One of the technical field that needs realtime processing is extension (AAPXS) messaging. The work, **Realtime AAPXS**, is also [detailed in another blog post](https://atsushieno.github.io/2023/12/24/realtime-aapxs.html), but the key thing is that extensions cannot be messaging while audio processing is active, because the audio thread is the only messaging channel. Thus, every extension requests have to be translated to MIDI 2.0 SysEx8 messages. And we made it to work.

This was a big change and involved breaking changes, so I wanted to make a lot more changes at this chance (I avoid breaking changes unless it is really needed). Thus it took longer time, but then now we have "URID" extension (similar to what LV2 URID does), host extension foundation, and so on.

Similarly, we made another breaking API changes around audio processing buffers: `aap_buffer_t`. It looks more like CLAP audio processing model. There we could hide some implementation details and eliminate unnecessary API exposure.

## Build simplification

The AAP plugins have become a lot over the years, and their build scripts had been diverse, which was just hard to maintain, especially at GitHub Actions which we cannot really run the tasks locally. Now we have **simplified and commonized GitHub Actions build settings** so that we do not have to worry about incompatibility. What we do mess instead is the Make-based build system, which is another annoyance but way better than dealing with GitHub Actions `*.yml`.

Another big restructuring in AAP is **manifest simplification**. that we do not need to generate static `aap_metadata.xml` in most case anymore. We can dynamically populate ports and parameters from the plugin code and extensions now, which makes it possible to have `aap_metadata.xml` almost manually editable.

Building and running AAP metadata importer tool has been always problematic, especially on aap-juce plugins, because we have to make changes to the desktop builds to add AAP classes resolved. But we kind of had to do it because it was not practical to manually write `aap_metadata.xml` without tooling aid. But we do not have to do it anymore. This opened the door to AAP world for some plugin projects. Namely we have [`aap-juce-surge` ](https://github.com/atsushieno/aap-juce-surge) project as an outcome.

## Quality improvements

At a bit of background, I had been running online audio dev. meetup in Japan, and for one of the meetup we were discussing realtime audio processing (like, talking about the famous [Real-time 101](https://www.youtube.com/watch?v=Q0vrQFyAdWI) ADC19 session). Around that time, aap-juce plugin ports were quite in low quality and needed vast improvements. I have spent many hours to identify what caused the problems from finding any audio glitch causes to realtime safety defects. Namely -

- applied strict thread switching using [ALooper API](https://developer.android.com/ndk/reference/group/looper) to dispatch any non-RT task to non-RT thread in MidiDeviceService implementation
- disabled and prohibited auto MIDI device opening in aap-juce ports. JUCE on Android, by default, tries to open MIDI port that can be a software synthesizer which can perform realtime audio processing, and caused unnecessary audio glitches imposed on our code for no reason. It is an evil behavior by JUCE and strictly prohibited.
- tracing support. We can examine how much time an AAP plugin `process()` takes, on both host and plugin sides.
- Big aap-juce hosting performance improvements by identifying a JUCE Oboe issue: JUCE until 7.0.5 was allocating memory on Oboe realtime audio thread.

These changes have vastly improved `aap-juce-helio`, an AAP port of [Helio Workstation](https://github.com/helio-fm/helio-workstation/) DAW. It is known to run on Android, so I could just add some build tweaks to get it working. But the quality was not great until I fixed the issues I mentioned above.

## New AAP plugin ports

I did not spend a lot of time to create new audio plugin ports (I cannot manage too many ports anyway), but there are still some. Here is the list of newly -or- emerged plugin ports that started working last year:

- aap-juce-os251: OS251 port means react-juce works
- aap-juce-byod: BYOD port means RTNeural works
- aap-juce-peak-eater
- aap-juce-seele
- aap-juce-adlplug-ae: OPL/OPM/OPN emulator as JUCE CMake-based porting project. (I cannot move forward without OPN...)
- aap-juce-ddsp: DDSP-VST port means tensorflow works (not TFLite though)
- aap-juce-surge
- aap-lv2-aida-x

## Review

There had been a lot of achievements in 2023!  I spent a lot of hours on the underlying work in 2022, and 2023 was of a bunch of leaps. We have GUI integration. We have new AudioGraph based plugin manager. We have realtime AAPXS foundation. We also have an installer foundation. We have nicer build automation. And we have a kind-of-working DAW that can apply AAP. It is finally getting the initial blueprint of the entire project.

There are, however, still a lot of incomplete works and TODOs. I will come up with the updated "Roadmap for 2024" to replace [the 2023 version](https://github.com/atsushieno/aap-core/issues/128) soon, but already seeing a lot of milestones.



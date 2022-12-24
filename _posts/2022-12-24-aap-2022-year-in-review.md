---
published: true
title: "AAP 2022 Year in Review"
date: 2022-12-24 21:00:00 +09:00
tags:
- AAP
---

It is almost end of 2022. It is still far from set in stone, but there are handful of updates to list up here. Let's continue the iterative year in review cycle, from that [of 2021](https://atsushieno.github.io/2021/12/28/aap-2021-year-in-review.html).

BTW I also had an unpublished draft of YiR for 2020. I [salvaged it](https://github.com/atsushieno/android-audio-plugin-framework/wiki/%5BOldNotes%5D-AAP-2020-Year-in-Review)  onto our newly started Wiki on GitHub.

## Problems to resolve in 2022

In 2021, I ported a bunch of audio plugins to Android, which were about 20-ish in total. We still didn't have solid foundation for the plugin hosting at the beginning of 2022, so it was important to stabilize things. MidiDeviceService was pretty fragile in particular. There was only a few instruments that could generate some sound outputs. It is still far from satisfying, but there was a bunch of improvements.

Part of the issue was due to hacky hosting bits. The host is not really "live", only calling audio `process()` function in static manner. In the beginning of the year I was planning to completely rewrite hosting to work more dynamic (processing audio with continuous inputs like part of an arbitrary audio file or flexible MIDI inputs or even an SMF).

We had [AAP roadmap 2022 edition](https://github.com/atsushieno/android-audio-plugin-framework/issues/96), but as anticipated there were more fundamentals to sort out.

## AAP evolution

These are the accomplishments I can recall for this year (not in chronological order).

### AAPXS: extension service foundation

In early AAP days, extensibility was already a key concept i.e. only essential features consist of core. State support  was implemented in native plugin examples as an extension, but at AIDL level it was not an extension. It was a hard coded feature. And when I tried to implement it, I noticed that we need more generalized API implementation by extension developer to connect a plugin and a host.

Therefore we came up with [AAPXS](https://github.com/atsushieno/android-audio-plugin-framework/blob/main/docs/EXTENSIONS.md) this year. During its development we also decoupled extension support from hosting API and implementation, so that technically AAPXSes should work with any host.

Once AAPXS had established, the state extension was rewritten to use this foundation, and the specific AIDL methods were removed. Then the same extensibility was used for presets API.

### designing new ports and parameters

Until 2021 I had recorded a lot of issues left open. Actually, I would say, they are not a lot by number. They are left open forever mostly because each task is kind of big. In any case, among all those issues, The biggest one across multiple issue threads was about port and parameters. When I started AAP, I was in general following what LV2 does. At the same time, I was fairly newbie and did not really understand a lot of its details. LV2 Ports and parameters looked in the same category in general (they indeed are, as ControlPort is used to control parameters). But after dealing with various plugins I figured that typical float parameters should remain as parameters (i.e. keep it simple and stupid) while there could be audio and MIDI ports. In the new design, parameters are defined as parameter, and we control them by some different way than parameter/control ports.

And even for audio and MIDI ports I was already seeing some problems. JUCE AudioPluginHost exposed a common Android platform issue that input is mono in general. Apps are responsible to clone the device input into two inputs, and feed them to effect plugins (unless the input is totally MIDI and the instrument plugin emits stereo outputs). I was also thinking to use MIDI ports to transmit general plugin control messages and messages like active sensing, at that time (well, I still am, kind of).

I ended up to write [a new design document](https://github.com/atsushieno/android-audio-plugin-framework/blob/main/docs/design/NEW_PORTS.md) to sort out every known issues and tasks, and (as explained later) the major parts are resolved.

### rewriting the hosting APIs

Writing a design document is one thing, and rewriting existing bits to it was another big task of course. Until this goal, we didn't really have to mess around the hosting API (it was just irrelevant to most of the work done in 2021). For example, in 2020 it was like that an instance of a plugin is created at once i.e. there is no concept of multiple instances for a plugin. It is fine for desktop plugin hosts, but for Android Service it is impossible for an application to create multiple instance of the same service. It binds only once as a service, and we should manage multiple instances per app. It is also possible for an AAP Service to serve instances for multiple apps. It is of course not practical (for example, you have two tracks for SFZ samplers using aap-sfizz then it's impossible). [A comprehensive rewrites](https://github.com/atsushieno/android-audio-plugin-framework/issues/101) of the entire design was required.

In 2021, there were two breaking changes in AIDL. Old AAPs and old hosts became incompatible with new AAPs and new hosts. The first breakage was introduced for this task, along with AAPXS changes above.

Another mess was Kotlin plugin hosting. Until this year, the Kotlin hosting API directly dealt with the AIDL. It was going problematic as the communication between native plugin and native hosting implementation goes complicated. We shouldn't be implementing the same complicated protocol in both C++ and Kotlin.

When it comes to aap-juce, things were even more fragile like it randomly crashed. It took a while for me to figure out [the threading model issue and fixes in JUCE on Android](https://atsushieno.github.io/2022/03/16/juce-android-threads.html). Once I got it fixed (by creating a patch for JUCE) I could go forward and fix all the remaining known issues on instancing threads.

### automatic port settings

One cosmetic-looking but significant part of the redesigned ports is that I don't want to let plugin fix audio and MIDI ports. MIDI ports are going to be necessary for any plugin (explained above a bit), and audio port configs really depend on host. Having them dynamic improves usability.

Audio bus configuration is a typical extension point, and we also came up with such an extension API. But there was not a single use case yet. So far, I could make changes to the framework so that `<ports>` become totally optional. Still, some plugins don't seem to work with it yet. A handful of small issues like this remain.

### parameters extension and MIDI 2.0 (only) ports

In summer I had been learning a lot about CLAP plugin format. The overall design is nice and I like it. CLAP is not something that can be placed on Android, but I took some good parts. One big design principle I introduced, inspired by CLAP, was the MIDI port unification. There will be only one MIDI input port and one MIDI output port in the latest "V2" protocol. Multiple input sequences and/or output sequences (from multiple ports) will confuse event processing by undefined order of the event messages.

CLAP defines its own event format, but I took another approach to entirely use MIDI 2.0 UMPs. Parameter changes were first designed to be sent as Assignable Controllers (NRPN). They could be Per-Note ACC as well. Though a few weeks later I noticed that it is going to run into the same pitfalls as [that of VST3 CC](https://forums.steinberg.net/t/vst3-and-midi-cc-pitfall/201879/). So now they are sent as a sysex8 UMP packet.

Before that I was thinking to support both MIDI1 and MIDI2 for a MIDI port, switching between them by MIDI-CI Set New Protocol messages. But as the parameter changes are always sent as UMPs and not it is not substitutable on MIDI 1.0 in practice (32-bit data vs. 7-bit data), I came to the conclusion that supporting only MIDI2 is the way to go. The imported plugins of course remain working. aap-lv2 and aap-juce (modules) internally take MIDI1 inputs, which were translated from UMP inputs from AAP host, and translate the output MIDI1 messages etc. back to UMP for its outputs. Well, I wrote a lie; LV2 Atom outputs are not supported yet. but you'd got the idea.

I had implemented MIDI 2.0 UMP translators to and from MIDI1 and MIDI2, and the translator implementations in both [cmidi2](https://github.com/atsushieno/cmidi2) and [ktmidi](https://github.com/atsushieno/ktmidi) were useful to achieve this task.

### aap-core v0.7.4, aap-lv2 v0.2.4, aap-juce v0.4.4

After all those bunch of changes for the new V2 protocol, we created a new release for aap-core, aap-lv2 and aap-juce, then updated a bunch of plugins. It is the most stable release in 2022.

I decided to give up updating "everything" - there are many plugins that cost upgrades.

## Recap and vision for 2023

These changes in 2022 were not like a few updates. There was not a big new feature like GUI integration, but you might find that GUI integration without decent parameter change protocol does not make sense. Solid plugin protocol without mature extensibility foundation would not last. Inflexible port configuration will limit plugin usability. I believe all those changes I made this year were towards better foundation.

Having said that, It does not mean everything is getting stable. We still don't have those plugin apps on Play Store yet (which I hoped to achieve this year). There are code quality issues that I really want to address sooner e.g. instrumented tests for connected plugins (I have been working on it these days). 2023 will be still busy. I haven't made it yet but will come up with the new and revised roadmap, early next year.

A big objective I still missed this year is that I did not make it more "open" to others. Sure, all the source code is open, but that is not enough. Several people asked me about the AAP status and usability, and I always reply like "it's still fragile and not quite usable" and/or "I should ask people help the project". I still didn't make it happen. I have listed a bunch of [issues](https://github.com/atsushieno/android-audio-plugin-framework/issues) on GitHub, but they are not likely welcoming "enough". I would need something better.

I'm trying to fix it. As I mentioned earlier a bit, I have set up wiki and have these pages now:

- [Are We Plugin Format Yet?](https://github.com/atsushieno/android-audio-plugin-framework/wiki/Are-We-Audio-Plugin-Format-Yet%3F) describes what AAP has kinda achieved
- [List of AAP Plugins and Hosts](https://github.com/atsushieno/android-audio-plugin-framework/wiki/List-of-AAP-plugins-and-hosts) not with some tasty treat screenshots, but there's already a lot of plugins

Last but not least, I should make it sustainable. It is currently a hobby project done by a jobless developer. I can stay so for a while, but not forever (even if I personally find it okay, I will have less social choices and that will make me unhappy). I believe AAP should be less bound to a single company than VST3 (Steinberg) or CLAP (Bitwig).

OK, that's all so far. Happy holidays and (a little bit eary) happy new year! Thank you for reading this all.



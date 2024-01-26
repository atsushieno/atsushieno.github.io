---
published: true
title: "ktmidi-ci-tool: MIDI-CI controller tool for desktop, Android, and Web"
date: 2024-01-26 20:00:00 +09:00
tags:
- ktmidi
- MIDI-CI
- MIDI-2.0
- Kotlin
- Compose-Multiplatform
- Jetpack-Compose
---

# ktmidi-ci-tool: MIDI-CI controller tool for desktop, Android, and Web

Today I'm announcing a new pet project **ktmidi-ci-tool**, a new experimental MIDI-CI controller tool. It is part of my [ktmidi project](https://github.com/atsushieno/ktmidi/) which just released the latest version 0.7.0, written in Kotlin, and runs on desktop (Linux / Mac / Windows), Android, and Web browsers (thanks to the latest Compose Multiplatform on Kotlin/Wasm), also potentially on iOS (no MIDI device access yet).

For Android, I published an open testing version on Google Play: https://play.google.com/store/apps/details?id=dev.atsushieno.ktmidi.citool

For Web, you can find the actual running version here: https://androidaudioplugin.web.app/misc/ktmidi-ci-tool-wasm-first-preview/

(I can build and run the app on iOS Simulator too, but currently we have no working Kotlin/Native MIDI device access API implementation on iOS yet.)

![ktmidi-ci-tool screenshot](https://github.com/atsushieno/ktmidi/blob/c1cec650bdd969e80e9e5c900f55c0f27a6525a8/docs/images/ktmidi-0.7.png?raw=true)
Since MIDI-CI feature is not very well known and the entire UI is not very intuitive per se, I have written another dedicated article on what the tools feature, interoperating with similar MIDI-CI tool in JUCE: CapabilityInquiryDemo: https://atsushieno.github.io/2024/01/26/midi-ci-tools.html


## MIDI 2.0 in 2023, observed

MIDI 2.0 was released back in 2020, but its adoption was, well, Largo. It seemed like chicken and egg situation: no DAWs would support MIDI 2.0 inputs if there is no MIDI2-ready keyboards, and now hardware vendors would have built ones if there is no DAW that supports MIDI2 inputs. A typical interoperability situation.

The situation has somewhat changed in 2023 when MMA released updated specifications (here we call it June 2023 Updates), and there are some development activities emerged afterwards:

- Roland finally supported MIDI 2.0 on their A-88MkII (which had long stated that "it will support MIDI 2.0" since the specification release in 2020). By supporting MIDI2, it receives and sends finer velocity than 128 levels, natively.
- KORG Keystage supports MIDI 2.0 with Property Exchange, along with some plugin that supports those messages
- ALSA project supports MIDI 2.0 UMP ports since Linux Kernel 6.5 and alsa-lib 1.2.10
- [libremidi](https://github.com/jcelerier/libremidi), a modern C++ cross-platform MIDI library, started to support MIDI 2.0 since v4.0
- JUCE since 7.0.9 (released in November 2023) supports MIDI-CI in the new `juce_midi_ci` module

And in prior to those, I should mention that, Apple CoreMIDI and AudioUnit supported MIDI2, Google supported USB MIDI 2.0 on Android 13, [CLAP](https://github.com/free-audio/clap) supported MIDI2 as one of event types since 1.0 (although it discourages use of MIDI2 in favor of its own event type...), etc.

### ktmidi, AAP, and MIDI 2.0 interoperability until 2023

My take on MIDI 2.0 was different, in the first place. MIDI 2.0 UMPs should be quite useful as better interoperable music sequence structure. There was no SMF 2.0 (there still isn't as an SMF replacement), but the basic data model is there. We made sure that there are no Control Changes beyond 127 even in MIDI 2.0 (compared to, for example, [SFZ format expanded them](https://sfzformat.com/extensions/midi_ccs/)). I support the idea behind UMP that respects backward compatible translators as much as possible.

My ktmidi library started as a new cross-platform MIDI library for Kotlin (as a successor of C# version of it, managed-midi), which covered only MIDI 1.0 but soon expanded its potential to MIDI 2.0 UMPs. It has been useful for me to generate some music information abstractions  (namely I had been authoring my own [Music Macro Languages translated to modern Tracktion Engine sequencer](https://atsushieno.github.io/2021/12/15/augene-ng-4.html)).

Also since I wanted to incorporate MIDI 2.0 support in Android audio plugins (AAP), I have created another C library called [cmidi2](https://github.com/atsushieno/cmidi2) back in 2020 and added MIDI 2.0 extension feature in AAP. It was done apart from ktmidi (as audio plugins need realtime safety).


## Determine whether MIDI-CI is useful and makes sense or not

As MIDI 2.0 related development became active among many manufacturers and developers, connectivity is getting a real chance as well as a concern. As an audio plugin format developer, I see it as an opportunity that we may be able to have MIDI 2.0 devices connected and controlling our plugins directly through the MIDI protocols. To achieve that, UMP support is not enough. We will have to support MIDI-CI.

(Note that we do NOT really need MIDI-CI if we just want to transmit UMPs without letting the sender or the receiver understand what your MIDI device is like. There is no protocol negotiation anymore. Your user simply connects your devices just as UMP transport based MIDI devices.)

I don't know what is the driving motivation, but JUCE has added `juce_midi_ci` module a few months ago. It would similarly give more enhancements to JUCE plugins. Audio plugins that are made with JUCE would expose some configuration points via Profile Configuration and Parameters via Parameter Exchange, in the standard MIDI-CI manner. Apple CoreMIDI supports MIDI-CI Device Discovery and Profile Configuration, so AudioUnit might benefit more from the platform integration later and JUCE plugins, when building AUv3 plugins, might fit there well.

Note that "parameter" in MIDI-CI Parameter Exchange does not really map to audio plugin parameters. MIDI-CI is NOT suitable for realtime processing, as it involves a lot of JSON parsing and generation, at least under the Common Rules for Property Exchange specification. You want to make use of something else like MIDI 2.0 channel voice messages like Assignable Controllers (or Per-Note version of them) in UMP, with some MIDI mappings. AAP supports parameters based on that approach, with some mapping configuration options (to avoid [some kind of pitfalls](https://forums.steinberg.net/t/vst3-and-midi-cc-pitfall/201879/1)).

### What MIDI-CI offers

If its "parameters" are not audio plugin parameters, is that even useful? MIDI-CI, as of v1.2 specification, gives opportunity for your MIDI devices to offer control points over:

- Profiles: you can either define *your* profiles (or third-party profiles), or declare that your device conforms to "MMA/AMEI defined profiles"
  - Each of them is an *unnamed* feature set that your MIDI controller can assume your device conforms to a common behavior e.g. "its channel x, y, and z are usable", "the channel accepts CC number x, y, and z", etc.
  - they can be enabled and disabled, per channel, per group, or all (the entire "function block")
- Property Exchange: you can define what kind of "parameters" your MIDI device provides
  - The parameter getter and setter comes with a header and a body parts
  - There is "Common Rules for Parameter Exchange" specification apart from MIDI-CI specification itself, and MIDI-CI implementations would mostly conform to that, to achieve basic interoperability.
  - The parameter body part is BLOBs i.e. untyped, unless you give some JSON Schema along with the Common Rules.
  - In the Common Rules, identifying the target parameter is done by name string, which means, it is not suitable for realtime processing (avoid string comparison; or in other words, why [LV2 URID](https://lv2plug.in/ns/ext/urid) matters).
  - value can be partially updated (i.e. parts of the entire JSON value tree)
- Process Inquiry and namely MIDI Message Report: offers the standard way for "bulk dump", kind of.
  - The interchanged messages involve non-SysEx MIDI messages
  - The dump is most likely bulky so it is very unlikely useful for realtime processing
  - It is one way; no setters

Property Exchange at this state is most likely useful to offer named configuration points which can be formed in JSON tree structure.

Does that make sense to your MIDI product? If no, MIDI-CI is not for you and you can skip that.


## Supplemental

### MIDI protocols and MIDI transport protocols

The MIDI 2.0 support development around ktmidi was, therefore, without caring interoperability with others much. It was mostly because it was unclear for me how MIDI 2.0 things should work, up to the underlying platforms. MIDI-CI specification used to have Protocol Negotiation as one of the three core concepts (they call it three-Ps along with Profile configration and Propety exchange) and it looked like we could switch MIDI 1.0 and 2.0 protocols, *different* transports (MIDI 1.0 bytestream and UMPs), over the *same* underlying (platform-level) transport.

This was not always doable. For example, Web MIDI API implementation inspects the bytestream to see if it contains System Exclusives and rejects them if the web session is not approved the SysEx messaging permission. UMPs are totally alien to them. Google Chrome (as of v120) reports a failure if it starts with <`80h` byte (treating it as running status, and reports that it is unexpected), regardless of whether we permit SysEx on the webpage.

The modern consensus on MIDI transports after June 2023 Updates seems that we define a transport with a fixed transport protocol (either MIDI 1.0 bytestream or UMP) in prior, and it is not assumed to be promotable. (Note that the MIDI protocol *within UMP transport protocol* can be still promoted, by the new UMP Stream Configuration Request message.)

I have some more words on that matter, but it was mostly described on [another post](https://atsushieno.github.io/2023/08/07/rmk.html).

### Upgrading MIDI-CI implementation for June 2023 Updates

Late last November, I was writing a blog post about `juce_midi_ci` (among Japanese "advent calendar" culture), and to investigate what it really is, I ended up rewriting my existing MIDI-CI implementation in ktmidi for experiment. Surprisingly(?), I had already implemented basic agent model as `MidiCIInitiator` and `MidiCIResponder` that gets connected each other, back in 2022 (in addition to MIDI-CI SysEx packet factory class written in 2021). But it was written when there was no June 2023 Updates.

It was a total rewrite. MIDI-CI specification has changed a lot at v1.2, the June 2023 Updates. The core of the specification was "three Ps" a.k.a Protocol Negotiation, Profile Configuration, and Property Exchange, but the entire Protocol Negotiation got deprecated(!). My implementation was *only* about connections (because its protocol negotiation was my only interest), so only the MIDI-CI SysEx factory classes remain useful.

If you have ever learned anything about MIDI-CI protocol establishment, forget about them all. MIDI-CI is based on MIDI 1.0-compatible SysEx, and (based on the modern consensus I mentioned above) there is no chance for MIDI 1.0 devices to get Protocol Negotiation working. The application-level MIDI protocol promotions can occur only in UMP land, and thus MMA moved the protocol negotiation part to the UMP specification (UMP Stream Configuration message).



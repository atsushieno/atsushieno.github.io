---
published: true
title: AAP: 2021 year in review
date: 2021-12-28 22:30:00 +09:00
tags:
- AAP
---

This is kind of secret post on this blog that I don't advertise broadly. You reached here only via search engines or link to the top of this blog, or whatever indirect.

AAP has been still quietly under development. I have spent a lot of time on non-AAP stuff (ktmidi and co.) this year, so things were not as productive as past years, but there are steady improvements.

## A bunch of plugin port repos and CMake support

What has happened on [android-audio-plugin-framework](https://github.com/atsushieno/android-audio-plugin-framework/tree/bf37e21dedd867d42919e097cb415f4aa107a61d), [aap-lv2](https://github.com/atsushieno/aap-lv2/tree/2d5825a1bac5260e57de10a0ff5509288d7c1dcb), and [aap-juce](https://github.com/atsushieno/aap-juce/tree/37c5d7558876c71f50ca138e0a7e39c04c001157) since last year? (They link to the old revs, not to the HEADs.)
 
Around the end of last year, I started splitting aap-lv2 plugin ports to individual repos and slimmed down aap-lv2 itself. aap-juce was still a big mess at that time. You can find how our `samples` looked like [last year](https://github.com/atsushieno/aap-juce/tree/37c5d7558876c71f50ca138e0a7e39c04c001157/samples).

I made a big split in aap-juce this year, and there are now >10 individual repositories. I mean, I kept adding more ports. They are important for me as proof of concept, dogfooding, and testbed. When some plugins failed to generate valid audio, crashed, or had different build setup, those ports from various sources helped me diagnosing the problems.

The big repository split was to ensure that they build, especially on CI. It was simply impossible before, as it could take a few hours to finish builds. Now I can easily see how new build changes go and promptly fix the issues if any. It comes with certain cost that keeping things up to date costs my development time though (thus I don't really update everything at a time, it's just too much). Probably I should leave those tasks to others when it becomes like a community project.

(There are good parts and bad parts by NOT building everything all at once and I indeed tried to create [a "world" repo](https://github.com/atsushieno/aap-juce-world) that contains "everything". But since it is too big, I cannot build it on GitHub Actions anymore - for the record, the last successful build took 4h 53m 46s.)

Another big challenge was CMake support. JUCE does not support CMake on Android, but CMake based projects are growing popular. [I ended up with my own porting template](https://atsushieno.github.io/2021/01/16/juce-cmake-android-now-works.html) with minimal impact on existing projects, and it is quite successful.

With all those improvements, I could feel easy to add new audio plugin ports, even if I'm not sure if they work as expected (because I have much limited controls over them at this state). These are some remarkable ports I made this year.

![Hera](https://i.imgur.com/EIhaXdb.png)
![Odin2](https://i.imgur.com/AXoKMrT.png)

![Vitaloid](https://user-images.githubusercontent.com/53929/147199726-c0b49d89-e1d0-494a-aa2c-895d63b5d11e.png)
![Frequalizer](https://user-images.githubusercontent.com/53929/147383092-6e918558-821c-4bec-9c8d-a2d18b3204d0.png)
![ChowPhaser](https://i.imgur.com/DxysqDj.png)

## AAP as MidiDeviceService, exploring MIDI 2.0 support

AAP started with an ambitious milestone - to realize audio plugin ecosystem on Android platform just like desktop or even, like AudioUnit on iOS. While it should be doable in theory, there are bunch of issues to resolve.

I kind of decided to start with smaller milestone. One thing I want is portable sound. Setting up an audio plugin hosts costs (development resource wise). I had [Fluidsynth working as a virtual MIDI device on Android](https://github.com/Rebloom/fluidsynth-midi-service-j), so why not sfizz, Dexed, ADLplug, and Vital? Fully-featured audio plugins would be hard, but virtual MIDI devices would not be. And this is where we are at now:

![AAPs as MIDI devices](https://i.imgur.com/IaeYiO8.png)

The app, [kmmk](https://github.com/atsushieno/kmmk/), is from my another development direction i.e. Kotlin MIDI 1.0/2.0 ecosystem. As an aspect, it is nothing but a virtual MIDI keyboard app, which is a client that can open arbitrary MIDI output devices.

Since these AAP MidiDeviceServices are conformant to Android MIDI API, they can be listed on other apps:

[AAP MIDI devices on Volcano Mobile MIDI sequencer](https://i.imgur.com/eceqJyv.png)

It is however not at very usable state right now; the audio outputs are glitchy for some reason yet. I had spent much time on another adventurous bits - MIDI 2.0. It was along with MIDI 2.0 support in that kmmk app, and I was exploring and wandering around some different approaches to support it, and things did not get fixed until I reached the current state. MIDI 2.0 support was once working in AAP MIDI devices, and currently it is not. I am on revisiting the entire stack, including the fundamental redesigning on ports and parameters (which also keeps me away from [GUI integration](https://github.com/atsushieno/android-audio-plugin-framework/issues/34)).

## JUCE Audio effect plugin ports

I made a lot of JUCE audio plugin ports for reasons; they are either Projucer-based or CMake-based, and they are either instruments or effects. They were good to find condition on any problems, and those effect plugin ports indeed didn't work.

It was due to issues in JUCE on Android, It is gone now and those effect plugins now work, kind of. 

## Still too early to advertise much

While these features look pretty cool (it is awkward that I say so by myself, but I would think so if someone else did them), but they are still not in good quality or often even not at usable state. MIDI device services don't always respond to note on/off messages, and crashes after multiple-time uses. No intuitive way to shut down the plugin service and potentially consume battery behind. I would go for bugfixes first.

## Revising tasks priority in the roadmap

In February I set up a roadmap for AAP for this year.

https://github.com/atsushieno/android-audio-plugin-framework/issues/71

Soon I switched to my primary hacking targets to Kotlin MIDI ecosystem and AAP was affected by it as shown in MIDI 2.0 support bits. I thought "ports should be of MIDI 2.0, not 1.0" (which kind of messed existing bits up) and I started working on it, putting aside those listed tasks. It is not bad, but those tasks were left unaccomplished.

In fact those tasks, for example, UI integration, would (or *should*) not be accomplished until  some other dependent tasks gets resolved. There are handful of issues around port design, namely:

- https://github.com/atsushieno/android-audio-plugin-framework/issues/73
- https://github.com/atsushieno/android-audio-plugin-framework/issues/79
- https://github.com/atsushieno/android-audio-plugin-framework/issues/80
- https://github.com/atsushieno/android-audio-plugin-framework/issues/44

When I designed AAP in 2019 I was simply following the LV2 approach because it looked flexible. It is indeed, but it seems that the users faced problem on how to deal with parameters in good manner. Most of them are single float value and there are hundreds of parameters that we don't want to potentially deal with buffers and thus having one controller port with lv2:patch support is taking place. We should do it simpler in unified form.

## Closing 2021

I'm not sure if I should write about my vision for the next year at earlier stage as I had that not-accomplished roadmap for 2021, but probably it's worth recording it. Though not right now, probably early next year.

At the same time though, I should also clearly state that I will not concentrate on this particular project; I am doing this for [the Generative music era](https://atsushieno.github.io/2021/12/15/augene-ng-5.html) and I will spend time on the actual creative works. It is not my advantage at all and it's better if other people do it, but I'm on these projects to prove that what I want to see is actually possible. Android audio plugin is for audio plugin ubiquity, and it is a mean, not the ultimate purpose.


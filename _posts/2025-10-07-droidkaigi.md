---
published: true
title: "On 'Android Audio: Beyond Winning on It' at DroidKaigi 2025"
date: 2025-10-07 21:00:00 +09:00
tags:
- Android
- Audio
- Plugin
- AAP
- MIDI
---

It is a compound translation from [my Japanese blog post](https://zenn.dev/atsushieno/articles/3c32802a377d53), Google Translate then a bunch of edits by myself. Some contents that do not make sense in English are altered.

----

At [DroidKaigi 2025](https://2025.droidkaigi.jp/en/), I gave a talk "Android Audio: Beyond Winning On It" (in English),  where I discussed the latest audio features missing on the platform at the Android audio application development frontline. The session recordings were already published [the very next day :](https://bsky.app/profile/atsushieno.bsky.social/post/3lymnhpidz22u)

https://2025.droidkaigi.jp/en/timetable/941476/

The session slides are available at speakerdeck:

https://speakerdeck.com/atsushieno/android-audio-beyond-winning-on-it

My discussions there were that, how Android has much less music apps compared to iOS, and how we should address our problems. The title implies that "you can achieve things fairly well on Android" is not enough. However, I wrote too much in the talk proposal. I intended to mention everything I wrote, but I regret that I ended up being too tied down by it. There were also some things I had intended to talk about at the time that I didn't mention at all (e.g. how MIDI2 can take place). There were things I couldn't possibly cover in 40 minutes, so I'd like to discuss them in writing, adding various supplementary information.

## What are music apps?

Since DroidKaigi is not an event where audio app developers gather (while such occasions are rare, they do exist e.g. audiodevcon), I gave some bits of explanation on how audio features are provided on operating systems and platforms such as Android.

Except for small apps like command-line tools that run on the desktop, most applications use some kind of sound-related functionality. The most obvious way to trigger sound is through notifications. Platform notifications, by default, will play some kind of sound unless the user has muted it. For example, a warning sound might be emitted when the user sends an invalid command in a terminal, or an error dialog might be displayed if the application notifies you of an error, or if the application itself encounters an error. App developers rarely call audio functionality directly for this purpose; at most, they might play a single audio file, like `System.Media.SystemSounds.Beep.Play()` on Windows.

What about audio applications in terms of music? Typical users listen to music and watch videos, so in this context it is still generally tied to features that typical users use. While this falls within the scope of what's known as a media API, playing media like audio and video with a proper user experience requires some skills, and if the platform's media API itself is lame, then mere playback feature could become awkward.

Of course, Android doesn't any concern at such level, and as of 2025, these features can be mostly achieved by using the Jetpack Media3 (formerly ExoPlayer) API. The Google Play Store offers a wide range of media players, including YouTube, TikTok, Spotify, and even Apple Music. Since Android isn't far behind Apple in this area, there's little point in discussing app development in this area. There were handful of sponsor companies that offer media streaming services with their booths at DroidKaigi, and some of them told me that they are using modified versions of Media3.

What Android lacks is not applications for playback, but applications for the creative class. Music production apps such as DAWs used in trackmaking (in Japangrish we call this kind of work "DTM", desktop music - regardless of that this term is appropriate in a mobile environment) require more strict real-time audio processing (we'll explain why later).

## Android Platform Audio state of union (2025)

Audio developers are always reluctant to develop for mobile, and it's worse on Android development compared to iOS development, for various reasons. Here we discuss how so.

### "Android has a lot of audio latency"

A common complaint about Android audio in the 2010s was that Android lacked or had insufficient real-time audio capabilities. Although it is only historical nowaday, there used to be some detailed analysis of Android audio latency published by Superpowered around Android 5.x:

https://superpowered.com/androidaudiopathlatency

It used to be true, but their points are outdated in the 2020s, especially in 2025. On Android, starting with Android 2.3 (API Level 7), audio apps could implement audio processing in C/C++ instead of Java, and Android 8.1 (API Level 27) made it possible to use AAudio, which supports MMAP-ed streams (strictly speaking, AAudio itself was usable in Android 8.0, but since it did not support MMAP-ed streams, it was only about their new API without low latency support).

Another thing to keep in mind is [the Audio Latency (5.6)](https://source.android.com/docs/compatibility/16/android-16-cdd#5_6_audio_latency) and [Professional Audio (5.10](https://source.android.com/docs/compatibility/16/android-16-cdd#510_professional_audio) ) sections of the CDD (Compatibility Definition Document), which Google requires device vendors to comply with in order to call their devices Android. These sections define audio latency standards; for example, a device must have a round-trip latency of 20 ms or less to claim that they "support Pro Audio". Since the Android platform will remain open at the source code level, as represented by AOSP, until 2025 and in theory any low-quality device can be built, arguments like "Android audio latency is..." does not make sense. When people discuss hard low latency on Android, it should be primarily about Pro Audio compliant devices.

The time-consuming processes that cause delays are found in ALSA and HAL (`android.hardware.audio.service` process or bus and DAC/ADC), or in the AudioFlinger implementation, including AAudio. Google's implementation of AudioFlinger does sufficient job for low-latency audio, and since Google has clarified quality standards, there is probably nothing that Google could work on anymore. (BTW you might sometimes see some misleading technical explanations such as "Audio Flinger was the audio foundation before AAudio". But if you carefully trace the source code, you would see that AAudio also processes audio via AudioFlinger.)

Since DroidKaigi is not an conference for audio developers, so I took the time to explain what problems arise when audio delays occur, what real-time audio processing is, etc. I didn't mention this in the session, if you're interested, I recommend reading [Ross Bencina's Real-Time audio programming 101: time waits for nothing](http://www.rossbencina.com/code/real-time-audio-programming-101-time-waits-for-nothing) or watching [the ADC19 Real-Time 101 session video](https://www.youtube.com/watch?v=Q0vrQFyAdWI) .

Also, although not mentioned in the session, there were some statistics on actual latency per device.

-   [https://superpowered.com/latency](https://superpowered.com/latency) : As mentioned above, this was around Android 5.x. The information is outdated, only for historical material.
-   [https://juce.com/maq](https://juce.com/maq) : Statistics collected by JUCE around 2018. The page has gone.
-   [https://ntrack.com/android-latency/devices](https://ntrack.com/android-latency/devices) : These figures are published by n-track studio, a company that makes DAWs. But I don't find them reliable - even if we only look at Pixel devices, the figures show that the Pixel 4a has the lowest latency and the Pixel 10 has the slowest.

…And so, the only documents I was able to provide were these two:

-   [https://android-developers.googleblog.com/2021/03/an-update-on-androids-audio-latency.html](https://android-developers.googleblog.com/2021/03/an-update-on-androids-audio-latency.html) : A blog post written by Don Turner, who was an audio developer advocate on the Android audio team at that moment. However, this blog post focuses on the latency of the average Android device as of 2021, so it's not for those who are interested in low-latency audio. Even numerically, an average latency of 30 ms is still quite poor for that.
-  [https://chromium.googlesource.com/external/github.com/google/oboe/+/devicelist/docs/Devices.md](https://chromium.googlesource.com/external/github.com/google/oboe/+/devicelist/docs/Devices.md) : This document summarizes the audio latency of Android Pro Audio compliant devices, showing that latency of less than 20ms is possible. This document mentions not only Pixel but even Nexus, and mainly contains information on devices that are even older than the above blog.

These are old, but while "audio delay is too large" is no longer useful as it is old information, "audio delay has been improved enough" is in general useful even if it is outdated. (Technically there can be some regressions, such as "MIDI latency has increased" in iOS 14, so I do say "in general".)

### "Android is Java, so..."

When people say "Android is Java," they're really talking about two separate things:

(1) Java cannot be used because it is not a language suitable for real-time audio processing. Most languages, including C#, JavaScript, Python, and Ruby, are not RT-safe. Therefore, the argument goes, Android is not suitable for audio applications. It is simply misunderstanding. No proper audio app developer would say this. Audio processing is required on a real-time audio thread, and Android's audio processing is of course done in C/C++. Developers up to Android 2.2 could not use OpenSL ES, so this argument was correct until then, but in 2025, almost no one uses Gingerbread.

(2) Having to code everything except the audio processing in Java is a hassle. This is true to some extent, but there are two points to note:

-   (a) Multi-language app development isn't only about Android. If you're developing an audio application for Windows using WPF, you'll likely handle audio processing in C++ and GUI in C#. Even on Apple platforms, where UI and audio code can run on the same native code runtime, building Swift for SwiftUI and C++ for audio together requires some annoyance to interoperate, at least not as easily as with Objective-C++.
-   (b) Audio app developers tend to favor cross-platform development, including support for multiple plugin formats, more than app developers in other fields. Therefore, their GUI part is often implemented using cross-platform C++ APIs. APIs like [JUCE](https://juce.com/) , OpenGL/Vulkan, and Qt are also available for Android, so if Android were to be excluded in this sense, it would only be that the library doesn't support Android. While DPF and iPlug2 don't support Android, the DPF plug-in seems to work on Android except for the GUI (some have been ported as aap-lv2 applications), and iPlug2 is rarely used in OSS (I'm not aware of non-OSS implementations), so this doesn't matter much.

Also, I didn't have time to mention this in the session, but you should design a separate GUI for the mobile platform. I've ported several JUCE plugins to Android, and even though JUCE supports Android, I've never found the GUIs of individual plugins made for desktop to be usable.

### "iOS has great audio features, but Android doesn't."

It's generally true that Apple is a company that's committed to audio features, and Google is not, but the Android platform does indeed have various audio features, including:

- MIDI 2.0 support: Android 13 added support for the USB-MIDI 2.0 protocol, and Android 15 added support for virtual MIDI 2.0 devices. Since Apple has not developed BLE-MIDI 2.0, MIDI 2.0 support in Android is complete. (In recent years, real-time performance has become a strict requirement for MIDI device input and output, and BLE cannot achieve real-time performance and is not RT-safe in principle.) USB-MIDI 2.0 support was implemented before ALSA, so unlike MIDI 1.0, it was implemented in Java without ALSA rawmidi, so it was ahead of MIDI 1.0 to the extent that it was implemented in Java. Windows still does not support MIDI 2.0.
  - By the way, the MIDI 2.0 specification had revamped in 2023, and support for MIDI-CI, as implemented by Apple, is not required in the platform API (Apple also states that the MIDI-CI API has been deprecated). The specification process was kind of in haste before quality.
-   The term "Android XR" arose last year, but Android 12 added support for Spatializer, which allows acceleration by hardware (mainly head trackers).
-   BLE Audio codec implementation. Since Pixel Buds are in the market, Google is actively committed to this field at the platform level.
-   Android 14 adds support for car audio plugins, which allow you to adjust DSP functions required for Android Auto.

Should Google still support any further audio features on the platform? Well, actually yes... there is **audio plugin support**. Android doesn't have an audio plugin feature equivalent to iOS AudioUnit V3, DAW features as well as plugin products in general are built on the premise of audio plugin formats, and therefore they don't exist on Android. This needs to be fixed somehow.

### "Mobile platforms are cumbersome"

Last but not least, but for developers who primarily work on desktops, mobile platforms are simply a pain. They have small memory and storage capacities, weak CPUs, and can only display one application per screen. Even if you could feature the fully functional DAW on a mobile device, it's too small to make tracks. Both iOS and Android have strict boundaries for app installation and execution, making it hard for apps to communicate with each other.

These problems cannot be solved by app developers alone, so the most reasonable approach is to release a product tailored to the mobile platform. It would be great to be able to carry out complementary tasks such as editing synth presets on the mobile device while lying down, and then transfer the resulting sound to a PC via the cloud.

Of course, mobile is troublesome not only for Android but also for iOS, so in the context of what makes it inferior to iOS, there is no further points digging in this topic.


## What's the point of an audio plugin?

So far, the only remaining issue Android has to overcome, especially compared to iOS, is the lack of audio plugins. Is this an issue that can't be resolved?

Since the audience at DroidKaigi is not audio developers, I had to explain what an audio plugin is in the first place. If you are similar, take a look at the slides. There were hacky explanations like "DAW is like an IDE," "audio plugin formats are like programming language runtimes," and "plugin formats are like LSPs", because the attendees were mostly programmers or software engineers.

When we release a DAW product on the platforms like Android where audio plugin formats don't exist, the DAW must provide all of the instruments that can be used within that DAW. However, just like there are various piano brands like Steinway, Yamaha, and Kawai, and various guitar brands like Gibson and Ibanez, software synth developers also have their own unique sounds. It is not realistic to expect that a DAW vendor (rather than instrument vendors) provides all the available plugins. Even if a DAW vendor were able to design an online marketplace, limiting the catalog of instruments available there harms users' freedom on trackmaking.

In this slide, I gave an example of a [Cubasis 3](https://play.google.com/store/apps/details?id=com.steinberg.cubasis3)  review, where a user left kind of negative feedback like "we can't use plugins like in the iOS version", and the developer responded like "because there's no plugin system on the platform" (I can't link to the individual reviews, so please search for "VST".), but there are many more similar reviews for [FL Studio Mobile](https://play.google.com/store/apps/details?id=com.imageline.FLM) . The lack of a plugin format affects reviews as users are dissatisfied with it.

Effects can also be used in situations other than music software. For example, if you want to process audio in a live streaming app like OBS Studio, you can apply some VST (although whether you can apply a  VST "3" plugin or not is [another matter](https://github.com/obsproject/obs-studio/discussions/5074) ).


## Barriers to Audio Plugin Mechanisms on Android

If there isn't a plugin format that can be used on Android, why not create one? For platforms other than Apple, it was Steinberg, not Microsoft, that developed VST2 and VST3 for Windows (Microsoft even created an audio plugin system called DirectX Plugin in the past, and also has a system called APO, though it's not generally used here). LV2, which is primarily used on Linux but can be used on both Windows and Mac, was started by a member of the Linux audio community. Until AudioUnit V3 was released, people used AudioBus on iOS, created by app developers in the Apple ecosystem. Should Google create a plugin format for Android by their own?

Based on all those kind of thoughts, I manage a project called [AAP: Audio Plugins For Android](https://github.com/atsushieno/aap-core) (I only mentioned the link briefly at the end of the slides, didn't even mention the name of the project in the session at all). Based on my experience, I would like to explain some of the features of plugin formats that are hard to achieve on platforms like Android.

(I wrote a similar article about AAP at shibuya.apk (occasional Android dev meetups) two years ago (in Japanese), with a more about AAP internals. The content is a bit outdated: https://speakerdeck.com/atsushieno/building-audio-plugin-ecosystem-on-android (written in Japanese)

### DAWs are unable to dlopen() plugins

On the desktop, a DAW can load a plug-in executable program `dlopen()`(or some `NSBundle` API on macOS,  `LoadLibrary()`on Windows), and run it in the same process as the DAW, but Android runs isolated apps i.e. a separate Unix user is created for each application, and the DAW and plugin run in separate processes, so communication between the DAW and plugin must be achieved using IPC (inter-process communication).

To go a bit further, while it is technically possible for an Android app to dynamically load native libraries passed to it by other apps, such apps are prohibited by Google Play Store policy. Such apps would have to be distributed outside the Google Play Store. I only mentioned during the session talk though, at [the Linux Audio Conference 2025](https://jimlac25.inria.fr/lac/) , which I attended in June, there was a session on a project called [LDSP](https://github.com/victorzappi/LDSP) , which aims to freely access ALSA and use audio functions on rooted Android devices .

At DroidKaigi we only talked about this level of granularity, but here is a more detailed discussion:

On desktops, DAW audio engines and audio plugins generally run in a single process. In contrast, the plugin mechanisms of typical web browsers are equipped with a sandbox mechanism, which maintains safety by preventing the browser runtime crashes even if a plugin crashes. IDEs are not as clear as that. VSCode has an Extension Host Process mechanism, but in IntelliJ IDE, plugins run in the IDE process. Generally, a process-separated model increases robustness, but the inability to share memory space between the host and plugins increases communication costs and complicates plugin system development.

One of the fundamental principles of realtime programming is "no system calls." System call implementations are system-dependent, and depending on the implementation, they may contain code that breaks realtime safety, so they are generally not marked as safe. In reality, some syscalls, such as `memcpy()`are commonly used and not considered as problematic (though this is not considered good for effective memory caching). Syscalls used by IPC features are usually non-RT-safe.

I described "DAW audio engines" (not like "the DAW app" entirely) because I'm having such a design in mind that the audio processing part and the other part (including the DAW GUI) reside in respective process spaces. It's better to load plugins in the audio engine process, and even if they crash the process, it would not affect the main part. Loading individual plugins in their own processes reduces the scope of the impact, but communication across process boundaries cost a lot, and the disadvantage of going beyond process boundaries between the audio engine and all plug-ins within the strict constraints of real-time processing likely outweighs the cost of more frequent audio engine crashes. Bitwig Studio has an option for such a mode, but should be considered as "for debugging plugins only".

### Binder IPC is not RT-safe

Since its early days, Android has had a mechanism called Binder that enabled low-latency IPC on the Linux kernel. In Android 8.0, this mechanism has been further improved to support [realtime priority inheritance](https://source.android.com/docs/core/architecture/hidl/binder-ipc#rt-priority) . Priority inheritance allows two threads communicating via IPC to raise the priority of the thread when handing over control to the other application. Without this mechanism, for example, if an AAudio thread running at real-time priority passes control to another application's process, that thread would run at normal priority and have to wait for the other thread to finish processing before being handed over control. This results in the original AAudio thread having to wait endlessly for this. This is known as [priority inversion](https://en.wikipedia.org/wiki/Priority_inversion) .

Binder's priority inheritance has a fundamental defect: it doesn't work in the framework domain. It can only be used by developers of the Android platform itself or by vendor device drivers. Binder, which ordinary users use in their own services, runs in the framework domain, so priority inheritance is not possible. If we, as Android app developers, design an audio plugin format and use Binder internally, audio processing will not be performed in real time.

This is one of the limitations that makes it impossible to create an audio plugin mechanism in Android at the moment.

### Plug-in GUI and the principle of one app, one screen

Android applications generally run one app per screen. If the DAW and plug-in are separate applications, their GUIs cannot be displayed at the same time. Therefore, some kind of workaround is required.

GUI is not mandatory for audio plugins, and it is possible to retrieve and manipulate lists of parameters and presets. In fact, AAP uses Jetpack Compose to create such a default GUI, but it is generally cumbersome for trackmakers to use. It is similar to the idea that "you can do anything by objectifying complex business logic and manipulating it via a PropertyGrid". It is better if we can simply support GUI instead.

When I talked about AAP at shibuya.apk, I introduced two approaches to implementing it:

- Web UI: If it is not possible to load the plugin's execution code in the host process, then the approach is to display a WebView on the host and load the Web UI resources provided by the plugin as the plugin's GUI within it. Since plugin parameter operations can be performed via IPC, the maximum that can be performed from the GUI is "all operations that can be performed from the host via IPC" (this is only the maximum, and it is not necessary to support everything).
- SystemAlertWindow: By obtaining special permission from the user, you can display an overlay on top of the host UI in response to a special request from the host, while keeping the GUI in the plugin process.

However, I later realized the following approach is possible, and now I believe this is the most optimal solution:

- `SurfaceControlViewHost`: Android 11 added a feature that allows the host `SurfaceView` to reflect any android.view.View on another (plugin) app side, transferred via IPC , so this approach uses this. UI events are handled by the plugin's `View` implementation. The host can use surface control APIs as well as control `SurfaceView`itself e.g. show and hide it. It is closer to audio plugins in desktop DAWs. Since the host itself displays the UI of other applications by the host's own will, there is no security concern like UI hijacking (because of "one app per screen" system).

The problem with the Web UI approach is that the DSP and GUI of a plugin cannot share memory space. This model is well suited to plugins developed for the LV2 format, where the DSP and GUI are separate dynamic libraries and do not share memory space. However, only some DPF plugins use the Web UI in LV2, and it is unclear whether they can be universally reused for Android porting.

The SurfaceControlViewHost approach can handle any view and share memory space in the plugin process, so if your desktop GUI code works on Android, it's in theory possible to port it directly. Having said that, depending on the GUI framework, transferring it to a SurfaceView may not work as expected. It's actually quite uncertain. For example, while I was able to display the GUI of JUCE (described below), I have never been able to get its inputs to work properly in the plugin process. JUCE's Android support is not well-developed with such a use case in mind.

[I explained how to use SurfaceControlViewHost in detail in TechBooster's INSIDE [Technology Secrets]](https://techbooster.booth.pm/items/5010062) two years ago (but only in Japanese...). In English, the only information I have is [a blog post by CommonsWare](https://commonsware.com/blog/2020/03/27/peek-surfacecontrolviewhost-android-r.html) published around the time of the release of Android 11. There are some Chinese blog posts, but they are not very insightful either (I can read some Chinese). I also briefly talked about this API at a [Mobile Dev Japan meetup](https://mobiledev-japan.connpass.com/event/350200/) this year (just the basics).

Regarding GUI, I think it's safe to say that the technical hurdle of mastering `SurfaceControlViewHost` is "difficult" but no longer "impossible."

### Appendix: Plugin format based on Binder IPC and maintaining forward and backward compatibility

This is something I always skip due to lack of time, whether at DroidKaigi or shibuya.apk, but audio plugin formats require APIs designed with forward and backward compatibility in mind. If a situation were to arise where a plugin with a new API couldn't be loaded on an older DAW, or vice versa, the plugin ecosystem shrinks. Maintaining compatibility is a key requirement. Both LV2 and CLAP avoid changing the core by defining extensions as individual APIs, and VST3 ensures this with a COM-like query interface called [VST3-MA](https://steinbergmedia.github.io/vst3_dev_portal/pages/Technical+Documentation/VST+Module+Architecture/Index.html). AudioUnit takes a similar approach to VST2, but explaining it would be tedious, so I'll skip it this time (it is not important this time).

Existing desktop audio plugin APIs are all built on the assumption that dynamic libraries can be loaded, so they cannot be applied as is to mobile platforms, and simply communicating through a proxy client/server cannot solve this problem of forward and backward compatibility.

If a plugin format is built using IPC, forward and backward compatibility can only be achieved by adding new methods. If the data indexes assigned once could be fixed, as in the case of Protocol Buffers, forward and backward compatibility might be possible, but deleting a method in the AIDL used by Binder would simply change the index and result in an incompatible interface. This would require supporting multiple versions of the protocol to bridge the gap, necessitating a cumbersome mechanism for filling the version gaps, like how Android Audio HAL works. (The difference between HIDL and AIDL in Android's lower audio layers makes things even more complicated, but I'll skip it for now.)

For this reason, I believe it is best to structure the API of a plug-in format so that it does not depend on AIDL as much as possible. In AAP, all operations based on the plug-in API's extended functions are realized using MIDI 2.0 SysEx to avoid hard binary dependencies on the API. Plugins or hosts that do not respond to the extended function's SysEx will ignore those commands.

## How Apple Overcame Technical Challenges with iOS and AudioUnit V3

Apple originally had its own desktop plugin format called AudioUnit V2, and simply built an API to make it compatible with the iOS architecture. That being said, it is relatively easier than designing a plugin format from scratch, but designing a plugin format to fit iOS's architecture, especially its separated process architecture, is not easy (I hope you understand why I've been writing so much up to this point).

How has Apple overcome this technological challenge?

First, a mechanism called App Extension was built for the iOS platform. This allows the plug-in mechanism to be used in any application running on the iOS or macOS platform, and AudioUnit V3 is designed to work as an App Extension for DAWs. Communication between the App Extension host and the plugin is carried out via IPC, which is similar to what is done with Android Services.

The GUI display will be developed as an AudioUnit GUI Extension. Like `SurfaceControlViewHost`, it is explicitly loaded by the DAW as an extension, so there is no concern about UI hijacking.

While App Extension IPC itself did not have realtime IPC support like Binder, iOS 12 introduced a realtime IPC priority inheritance mechanism called Turnstiles to the XNU kernel. The design idea for Turnstiles is not originated by Apple; it seems to have been implemented by Sun Microsystems (then) in Solaris in the last century, 199x. [A SunWorld article from 1999](https://news.ycombinator.com/item?id=21751269) provides more details ( [via hacker news](https://news.ycombinator.com/item?id=21751269) ). This feature is now available in AudioUnit V3.

XNU turnstiles allows AUv3 DSP processing at realtime priority when the DSP is running on a single thread, which already made it possible to achieve RT-safety ahead of Android. However, iOS 14 further added a feature called Audio Workgroups. This "groups" not only the audio thread run by the AUv3 host and the DSP thread running the audio processing function in the AUv3 plugin, but also other threads running at real-time priority, maintaining the real-time nature of cooperative operation. It is not generally necessary unless such real-time threads are created in the plugin. See the [WWDC 2020 session video](https://developer.apple.com/videos/play/wwdc2020/10224/) for more details.

Looking at it this way, some readers might get the impression that Apple is ahead of Android in audio features, but in fact, iOS 12, which implemented XNU turnstiles, was released in 2018, which is quite recent. Five years have passed since then, until Apple releases Logic Pro for iPad in 2023.

[As I explained in a blog post](https://zenn.dev/atsushieno/articles/aa07869fd9d96c) (note: in Japanese), I wrote following the release of Logic Pro for iPad , AudioUnit V3 recently added a feature called Compact View. This allows you to create a UI apart from the main plugin UI. It displays just a few knobs for key parameters, allowing you to see and adjust multiple plugins installed on each track on a relatively small screen like an iPad. It's these kinds of UX optimizations that make the UX of DAWs that support AUv3 possible. Of course, this is merely a matter of plug-in format conventions, so it should be easy to implement this for plug-in formats that work on Android.

## How should the Android platform respond?

### Challenges for the Android platform

The only feature missing from the Android platform seems to be audio plugins, but the platform is still evolving in terms of optimizing its internal design. One interesting feature added in Android 16 is in-process media codecs. This means that hardware vendors can implement media codecs in a "safe language" that can be called and executed within the application process. While it says "safe language," this is currently believed to refer to Rust. Conventional media codecs that do not use IPC will continue to run in a sandbox, so IPC is required to communicate with the application (although it should be at the level of exchanging control commands rather than passing data via IPC). For more details, see this [AndroidAuthority article](https://www.androidauthority.com/android-16-in-process-software-audio-codecs-3541408/) .

(PS: AOSP has a loader implementation [here](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/av/media/codec2/hal/client/client.cpp;l=1199;drc=61197364367c9e404c7da6900658f1b16c42d0da).)

This is an interesting move by the Android audio team, as one possible solution would be to allow safe plugin DSP code to be loaded into the host process.

However, the problem is not that simple. DSP code running in the audio thread needs to cooperate with other code (not necessarily GUI). For example, a sampler plugin would have to implement the code that loads and prepares a file stream and passes it to the audio thread in an RT-safe way in a non-RT-safe thread. If this thread were to run in a separate process space from the audio processing thread, then IPC would be required within the plugin implementation. This would be a significant difference from the existing plugin development paradigm, and thus would go against code sharing.

How about this, Google allow DSPs implemented in "safe" code to inherit realtime priory in Binder? This seems possible. (Dangerous) operations such as file I/O are usually performed in non-DSP code, so we can say they should not be called in DSP threads. However, it is unclear whether safety verification can be performed on a function call graphs, and the details of the safety verification are currently too unclear. Relevant sources may be in AOSP, but I did not dig in depth.

Another challenge is the lack of official support for Rust in the Android NDK. This issue [has been logged in GitHub issues](https://github.com/android/ndk/issues/1742) since 2021 , but progress has been slow. User space is currently seeking code in a language that isn't officially supported. While audio developers' continued use of C++ is a drawback, plug-in developers, like those working with real instruments, have a wide variety of brands, so developers likely won't be as enthusiastic about switching to Rust. Furthermore, plugins will likely continue to be developed using cross-platform technology, so migration to Rust just for Android is unlikely to happen. In this sense, I assume that an approach like safe C++ or alt-C++, where developers reuse and build C++ code, may still be viable. In that sense, Google's [Carbon](https://docs.carbon-lang.dev/) , which from the outside seems like it's unclear what it's doing, could be one solution (although it's not particularly popular in the alt-C++ community).

### Challenges for the Audio Developer Community

If we wait for the platform to have all the necessary features, the audio application ecosystem will fall behind even worse, so we had better act on whatever we can as user developer community.

It would be tough to get audio developers to directly use frameworks and plugin SDKs that can be used on Android alone. The same is true for the AudioUnitSDK and VST3SDK. When searching for open source plugin projects on web sites like GitHub, the vast majority of them are JUCE projects. JUCE is a cross-platform, multi-plugin format framework for audio app development, and is widely used in this field. Projects that directly use the VST3SDK are rare. To put it in terms of mobile platforms, it's like everyone is writing apps in Flutter or React Native instead of Kotlin (ignoring KMP here).

JUCE is designed to make it relatively easy to extend the platform and plugin format, so for my AAP, I can use my JUCE module called [aap-juce](https://github.com/atsushieno/aap-juce) (this is also a unique world, but it's not the topic for today so I'll skip it) to create and host AAP. If you're starting your own plugin format project like AAP, it's a good idea to create a JUCE module.

A pain point here is that JUCE is (effectively) a commercial product, dual-licensed under both a commercial license and AGPLv3, and is a closed project that ignores contributions that are unrelated to the company's current direction (though at least they're not completely refusing contributions, which is much better than before). Android support isn't as powerful as other platforms, but compared to most other plugin SDKs that don't support Android in the first place, it's still _relatively_ robust. So, for now, it seems like the most effective approach is for JUCE to utilize what it has and support Android support for other plugin SDKs like [DPF](https://github.com/DISTRHO/DPF/).

Furthermore, since MIDI 2.0 has already standardized the functionalities that used to be fragmented among various audio plugin formats, I believe that actively adopting some MIDI2-oriented approach, where "whatever can be done with MIDI 2.0 should be done with MIDI 2.0, and whatever can be done only with audio plugins should be done with audio plugins", will lead to a breakthrough. It is a matter of development on the host side rather than the plugin side, but if plugins can accept MIDI 2.0 input, they will no longer need to process plugin-format-specific events, which will increase portability. JUCE is [expected to](https://forum.juce.com/t/midi-2-0-preview-branch/66453) support MIDI 2.0 input in its next major version update , so the future looks bright in this area.

Additionally, in terms of plugin GUI support, active use of Web UIs may make it easier to support GUI-equipped plugins on Android. However, how a plugin development framework supports Web UIs is still a unique solution for each framework, so some research would be required to determine what is needed to enable seamless use of them on mobile platforms. Another issue is how to enable the exchange of audio buffers and complex data types when hosting a GUI in a WebView (currently, audio streams are linked from the WebView via resource resolution using the URL resolver).

## summary

The Android platform is comparable to other platforms when it comes to developing music applications such as DAWs, except for the lack of a plugin mechanism, but without a plugin format the musical instrument ecosystem cannot develop, so something should be done. However, to do this, the complicated issue of real-time priority inheritance of IPC needs to be somehow resolved at the platform level, and for the time being, the app developer community should focus on expanding Android support.



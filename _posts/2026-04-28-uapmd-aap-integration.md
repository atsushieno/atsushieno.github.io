---
published: true
title: "UAPMD v0.4 x AAP v0.10"
date: 2026-04-28 14:45:00 +09:00
tags:
- UAPMD
- Android
- Plugin
- AAP
- JUCE
---

Almost 4 months ago I [wrote](https://atsushieno.github.io/2026/01/01/uapmd.html) about my latest development, [UAPMD](https://github.com/atsushieno/uapmd), which WAS about realizing virtual MIDI 2.0 devices on desktop using arbitrary audio plugins in VST3/AU/LV2/CLAP formats. Many changes happened after that, until now. Yesterday I released the latest UAPMD, which is version 0.4. This is the latest screenshot.

![UAPMD v0.4 sshot](https://github.com/atsushieno/uapmd/blob/main/docs/images/uapmd-app-v0.4-sshot.png?raw=true)

Is it looking as complicated as v0.1?

![UAPMD v0.1 sshot](https://github.com/atsushieno/uapmd/blob/main/docs/images/uapmd-app-v0.1-sshot.png?raw=true)

But note that the screenshot was mostly filled with various plugins. They were just to show off the variety of supported formats. The v0.4 sshot above only contains one plugin, and it is taken on Android. What's going on there?

I will explain the progress I had been making these months step by step.

## UAPMD v0.2: from audio plugin host to multi-track sequencer

The initial rationale behind UAPMD - well, it still *is* kind of - is to achieve MIDI 2.0 sequencer i.e. standardized ways to play music in the modern and flexible set of instruments, not just [traditional hardware MIDI module lunch boxes](https://www.reddit.com/r/retrocomputing/comments/1lymhke/midi_modules_for_classic_pc_gaming/).

But that means, we need way more complicated software. When I released v0.1 milestone and wrote the blog post last time, I was like:

> Support for decent multi-track engine is one of the post-0.1 milestones, and that would come up with simple-ish imaginary MIDI 2.0 Container file player (imaginary as the spec. document is still not published yet) which would look more like a DAW.

Therefore, UAPMD went that path. It became more like a multi-track sequencer, as this v0.2 sshot tells:
![UAPMD v0.2 sshot](https://github.com/atsushieno/uapmd/blob/main/docs/images/uapmd-app-v0.2-sshot.png?raw=true)

At this state, it started looking much more like a DAW. It was still MIDI oriented sequencer, but the clips can be MIDI2 or audio. It featured multi-track MIDI 2.0 clip player. The MIDI 2.0 events are actually translated to plugin notes, parameter changes, and direct MIDI messages, mostly down-converted to MIDI 1.0. Each SMF2 Clip is saved and loaded as a file, and then bundled within a zip project archive (`.uapmdz` file).

In UAPMD, an audio track is like an "instrument" - we can apply audio plugins and they can be controlled by those MIDI2 clips. There is actually no concept of "track type" - both audio clips and MIDI clips are just timed events.

The project archive is a set of project files, that looked (well, it still is for now) like [what SMF2 Container Format is supposed to be](https://midi.org/the-state-of-midi-2-0-high-resolution-performance-and-the-rise-of-profiles-update-feb-2026) (the link may die sooner or later, The MIDI Association has no interest in preserving their blog post URLs). As of now, SMF2 Container is not going to be our project file format anymore though.

There was of course a lot more of bugfixes and quality improvements. As of v0.1, the plugins I could retrieve MIDI-CI properties was quite limited. They tend to stall in the middle. There is no such a problem, nowaday. AUv2 and AUv3 have their hsoting implementation respectively to get full features to not miss whatever AUv2 bridge lacks support. Wayland detection and support was awkward in v0.1 (it may still not work, as I haven't set up Wayland-only environment). UMP Function Blocks have more solid mappings.

There were handful of additional features. We started supporting QuickJS scripting.

In the meantime, UAPMD v0.2 had been polishing the UMP device mapping support (which still continued until 0.3). If you remembered the previous blog post, it also mentioned "midicci" MIDI 2.0 keyboard app. Now it looks like this:

![midicci-app v0.2](https://github.com/atsushieno/midicci/blob/main/docs/images/midicci-app-v0.2.0-sshot.png?raw=true)

## UAPMD v0.3: ubiquitous sequencer (Android, WebAssembly, iOS, and Windows)

UAPMD v0.2 achieved another important milestone; I got a nice music project file format for a DAW-alike sequencer engine, all based on open technology. That is what I kind of wanted to have for my Android audio plugin ecosystem. Thus, support for Android slipped into my post-v0.2 milestones.

### UAPMD on Android

As [Audio Plugins For Android](https://github.com/atsushieno/aap-core) is one of my primary projects, Android should be a first citizen target.

Supporting Android as a target was not hard - API wise. UAPMD API is designed to be common abstraction with format-specific extensions and platform-specific extensions. The biggest hurdle is ImGui-based UI - I made it work, based on SDL3 on Android, which I think is fairly straightforward. SDL3 on Android however does not work like other native platform API - especially [its "main" thread is NOT the platform main thread](https://github.com/libsdl-org/sdl/issues/15010). It is the same kind of problem like [Flutter < 3.29 had before](https://www.reddit.com/r/FlutterDev/comments/1io6wo4/whats_new_in_flutter_329/). But so far, it is working. When I deal with AAP, it is still crashy compared to the desktop version - I need to figure out what is causing them.

![UAPMD v0.3 sshot](https://github.com/atsushieno/uapmd/blob/main/docs/images/uapmd-app-v0.3-sshot.png?raw=true)

The greatest thing about this is that, now we have most solid AAP hosting application ever. Formerly, we had AAP hosts as:

- AAPHostSample: it is a PoC host app written in Compose. It does not involve complicated process flow like a valid DAW has e.g. parameter refresh, state loading and saving, bus connectors, etc.
- aap-juce-plugin-host(-cmake): JUCE audio plugin host. Since JUCE itself does not support plugins on Android, everything is not quite stable.
- aap-juce-simple-host: it was created to isolate various issues I experienced at aap-juce-plugin-host. It has both problems that JUCE has and AAPHostSample has (not a real-world use).
- aap-juce-helio: it was the only JUCE-based DAW that I could bring to Android. It was already great I coudl *actually* do that. But I need in-depth debugging to resolve problems as I'm not sure if they are caused by juce_audio_plugin_client or the DAW itself.

So UAPMD is going to be a savior for me. Having said that, the JUCE API is still going to be useful as it provides the common way to deal with plugin APIs, and its behavior should be still informative. However, it is also crashy so that we have to patch around various issues. One big reason I dumped the idea of investigating it was that every time I close the plugin UI the container window caused crashes, and it was not limited to the plugin UI window. Making changes to all those plugin UI code is beyond one human work.

### UAPMD on iOS

After I dealt with Android, I thought that it would also be possible to run it on iOS, although I don't have any iOS device. But since we already have first-class AUv3 support, it should be possible in a bit - yes, it was:

![UAPMD on iOS Simulator](https://i.imgur.com/C46Iej2.png)

### UAPMD on Windows

As time went by, it was becoming more likely that Microsoft was finally going to release the stable version of Windows MIDI Services. Supportting Windows was going to be a big win to the MIDI 2.0 ecosystem, which was kind of my interest as of that time, so enhancing Windows support became an objective in the next step. The MIDI 2.0 support is still [stuck in the middle](https://github.com/celtera/libremidi/issues/194), but I made some effort to stabilize plugin hosting side and I reached the status that there is no known VST3 hosting issue. (We have some CLAP hosting issues, but that's not limited to Windows.)


![UAPMD around v0.3 on Windiws](https://i.imgur.com/ya31puK.png)

### UAPMD on the Web

After I have successfully got those platforms covered, I became optimistic like "why not support WebCLAP as well?" because I was sure that ImGui works well on the browser (built as WebAssembly), and I don't have hard native platform dependency. I don't see a lot of information on WAM2 and there was no development activity in these months, so my interest is only on WebCLAP so far.

I still don't have a lot of experience on Web Audio, not just WebCLAP, so I ended up letting Codex inspect the existing WebCLAP headers and samples to get them working within UAPMD, with some help by signalsmith (the WebCLAP developer).

![UAPMD on the Web (actually v0.4](https://i.imgur.com/T9F7KgM.png)

There was no synth plugin ported to WebCLAP at the time of building it (and as of when I am writing now), so only effect plugins are tested.

You can try the latest wasm build from here: https://atsushieno.github.io/uapmd/playground/latest/

Support for WebCLAP was not actually straightforward. It requires such an architecture that splits audio processing and anything else into respective threads. It seemed like audio engine separation that I wanted to implement at some stage for isolating the entire plugin hosting process, but that wasn't. Separate audio engine only requires data sharing across processes, but separating the entire plugin host requires a lot more IPC work (still not implemented).

I also had to make a change in our State API. CLAP requires the state API functions invoked on the *main* thread, but UAPMD cannot block the wasm main thread. To reconcile the situation, I had to make changes to our plugin client API (not the plugin API abstraction layer itself) to become asynchronous. The WebCLAP official sample host does not provide state features yet, so it is all my own experimental stuff, but it works for now.

Another related improvements is that the audio graph API is now part of the realtime-safe part of the entire system. Until this version, UAPMD only provided simple linear connection of nodes so there was not much pressure on RT-safety requirement. But what happens when someone started implementing complicated audio graph - can we really require that? This question led me to the conclusion - I have to prove that anyway. That became part of the next release.

## UAPMD v0.4

Up until now, UAPMD is getting more features such as piano roll editor, audio graph, and audio warp editor. However, UAPMD is primarily a plugin host so far, and while I keep bringing in DAW-like features, they exist for providing some way to implement common plugin format features such as latency reporting (and thus compensation).

In the meantime, I had to work on a complicated audio graph. I had to provide some PoC implementation anyway, to really provide latency reporting API and tail samples API in the plugin hosting API abstraction (`remidy`). Fortunately the theory here is very traditional. We need DAG, and editing the tree has to be RT-safe by the new requirement we just got as mentioned earlier.

Adding support for DAG-based audio graph was done with some API redesigning. What I aim to provide with UAPMD is a flexible API that brings in developers choices. I don't want to force specific graph design and implementation. I have been designing the entire sequencer engine as split in a few chunks and making them independent of each other. This means, the overall audio graph is still independent of multi-track sequencer / editor. as well as plugin instancing API abstraction. To achieve that, I ended up defining the abstraction layer for minimum requirement (like "add this plugin instance to this graph - how to actually connect nodes is implementation dependent though"). It was especially not easy when I was dealing with UMP Function Blocks vs. tracks, but it kind of makes sense by now. If I didn't work on this kind of adjustment, it could quickly bring in leaky abstraction and made API changes harder.

Another notable feature is support for external API endpoints such as HTTP server and Android Provider for embedded scripting and MCP endpoints. The MCP endpoint helps development, not just trackmaking, when telling coding AI agents perform some investigating and verification based on my human language instructions.

All those new features were already enough to tag v0.4, but unlike the existing versions, v0.4 was not tagged for achieving general milestone. It was tagged in along with the new release of AAP, because UAPMD can become the first real-world use case for AAP. While UAPMD it still primarily a plugin host, it can load existing music projects that were created on desktops, then assign AAPs as audio plugins, and play it on the device. And there was a big achievement in AAP land: I could finally get JUCE plugin UI working. This is a copy-pasted capture of the latest [androidaudioplugin.org](https://androidaudioplugin.org/) landing page:

![androidaudioplugin.org capture](https://i.imgur.com/BDskYyM.png)

I had to make various fixes to get the latest plugin UIs working with UAPMD, but now it's almost good. That's how AAP got released as v0.10.0, and UAPMD v0.4.0.

It's still NOT usable yet though; we still have some issues to get AAP working fine with project saving in UAPMD.

Those plugin UIs are not designed for mobiles, so there are many pieces that are not working great on Android. Having said that, it's still better to have the plugin UIs so that we can pick some presets from their UI. An inconvenient truth about JUCE plugins is that most of those plugins do not expose their presets through the plugin format API (via JUCE API). On such a plugin, we can only pick their presets on the UI.

On many plugins, presets are not designed for platforms like Android where people usually do not have home directory. I sometimes added a first-time-run content extractor on those plugins that enumerates presets from home or user data directories.

Another Android-specific issue (or maybe mobile-specific issue, depending on how AUv3 works) is that many plugins use `addtoDesktop()` directly or indirectly, which does not work on remote plugin UI controller (Android `SurfaceControlViewHost`). This includes JUCE `PopupMenu` kind of API. As soon as the plugin UI attempts to invoke that, the UI goes unresponsive because it renders the component to the global desktop, which does not exist on the audio plugin `Service` contexts. There are many plugin app patches I added there. These crazy amount of works are not very human friendly. They are labor, not fun part of coding. This is certainly what coding AI agents take place.

But all those efforts are worth. With usable presets, dealing with AAP is a lot more of fun, even though we still cannot save the manipulation results.

In the next development cycle, I will have to bring in some ABI incompatibility in AAP and that's going to be a big deal. But once I get all those features working on UAPMD, hopefully I will be able to move forward to the first 1.0 release cycle. There are many parts that I want to "modernize" from the API to the samples, but things now look realistic than ever.



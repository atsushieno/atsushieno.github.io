---
title: 'juce_emscripten: the latest JUCE on WebAssembly'
date: 2020-01-01 21:30:00 +09:00
tags:
- juce
- wasm
- vocal
- synthesizer
- synthesizer-v
---

It is kind of English translation from [my Japanese blog post](http://atsushieno.hatenablog.com/entry/2020/01/01/121050).

New year beginning, with juicy news to the audio developers community!

Today I'm introducing [**Dreamtonics/juce_emscripten**](https://github.com/Dreamtonics/juce_emscripten/), the latest JUCE fork development for the Web. It is developed for a virtual singer product called [Synthesizer V](synthesizerv.com/) by Dreamtonics, a small company in Tokyo. They have been developing their next generation of the product and today they have published a Web version of the preview release. (I am not the developer, just helping the effort a little bit there.)

You can try it from here: [https://web.synthesizerv.com/](https://web.synthesizerv.com/)

The most impressive thing is that they have been using [JUCE framework](juce.com/) and brought it up onto the Web using [Emscripten](https://emscripten.org/). It is actually based on the [former work by beschulz](https://github.com/beschulz/juce_emscripten) about 5 years ago, and Dreamtonics brought it to so featureful state that they can run their product which is entirely based on JUCE.

(For reference, there had been similar outcome from [WebAudioModules (WAMs)](https://www.webaudiomodules.org/) team. WAM is actually a small bit of C API definition with minimum JS bridge. JUCE could be integrated with their C API. They have released Web version of [Dexed](https://webaudiomodules.org/demos/wasm/dexed.html), [OBXD](https://github.com/jariseon/webOBXD) and so on. WebOBXD repo comes with JUCE based sources.)

## Experience juce_emscripten

You can try out juce_emscripten locally too. Let's see how you can do it.

In the Dreamtonics/juce_emscripten repo, [examples/DemoRunner](https://github.com/Dreamtonics/juce_emscripten/tree/master/examples/DemoRunner) is configured for Emscripten to build. Since there is no Projucer support (we will come back here later), there is an already configured Makefile within `Builds/Emscripten` directory there. As far as verified, it builds on Linux (Ubuntu) and Mac. WSL on Windows should be good too.

We are going to use emsdk (Emscripten toolchains). If you don't have Emscripten emsdk ready, you have to set it up. The README.md in juce_emscripten [also explains](https://github.com/Dreamtonics/juce_emscripten#build-instructions) but let's take quote it here:

```
$ git clone https://github.com/emscripten-core/emsdk.git
$ cd emsdk
$ ./emsdk install latest
$ ./emsdk activate latest
```

Once emsdk is set up, then you need environment settings to make emscripten toolchains usable. Install once, set environment every time before you start hacking juce_emscripten:

```
$ source ./emsdk_env.sh # これはemmake
```

Now you can build it with `emmake` at `examples/DemoRunner/Builds/Emscripten` directory. (Note that there is already `Makefile` which is configured, and if you run `Projucer --resave DemoRunner.jucer` it will overwrite this working file with broken output!)

```
$ cd /path/to/juce_emscripten/examples/DemoRunner/Builds/Emscripten
$ emmake make
```

Once you are done, there will be Emscripten build output web resources. You can run the app on Chrome/Chromium. The README says `python -m SimpleHttpServer` in `build` directory. I prefer `npx http-server build`. Any HTTP server is fine.

![DemoRunner sshot](/images/posts/upload_86387bbf02462e4de265173f8e2b4af0.png)

## JUCE platform backend basics

How come can JUCE the desktop application framework became Web browser ready? It is obvious to those who have been having eyes on cross-platform libraries, but usually those libraries extract platform-specific parts out and provide only common parts as their stable API (in general).

JUCE already has support for Windows, Mac, Linux, iOS and Android, and those platform-specific parts are implemented in their "modules". Unlike React Native, Xamarin, or Flutter, JUCE primarily targets audio apps, therefore it has to provide audio and MIDI I/O parts in cross-plaform manner. The Audio/MIDI part is implemented in `juce_audio_devices` module, and the GUI part is primarily in `juce_gui_basics` module.

For the GUI part, every OS/platform has a message loop for each GUI app, usually on the main thread, to manage the entire set of GUI events. In JUCE it is covered by `juce_events` module, which is used by both `juce_gui_basics` and `juce_audio_devices` (as audio I/O is also usually achieved by native audio callback framework). `juce_events` and `juce_core` combined together resembles GNOME glib (which is apart from the toolkit i.e. gtk).

These modules have platform-specific implementation in their source code respectively. For `juce_audio_devices` they are at [`juce_audio_devices/native`](https://github.com/WeAreROLI/JUCE/tree/master/modules/juce_audio_devices/native), and for `juce_gui_basics` it is at [`juce_gui_basics/native`](https://github.com/WeAreROLI/JUCE/tree/master/modules/juce_gui_basics/native). You can see that there are `android_...` or `ios_...` or `win32_...` files.

And you can add you own platform-specific implementation there too (in your fork). It is what juce_emscripten does there.

## Notes on build your apps with juce_emscripten

You can use juce_emscripten with your JUCE project, if it works. It has a lot of potential to unleash the awesomeness you made with JUCE to the Web, not just the new Synthesizer V editor and JUCE DemoRunner.

Since JUCE has wide range of feature coverage, juce_emscripten however has various limitation on the feature coverage. Also, since it is currently not backed by Projucer, you will have to make some changes to the build script Makefile. I am going to details them one by one.

### missing modules

First of all, check out the [Status section on README.md](https://github.com/Dreamtonics/juce_emscripten/#status). It has a list of all the standard JUCE modules with coverage status. (I'm not going to copy the list here since it may be actively changing.)

Especially, right now there is no support for `juce_audio_plugin_client`, which is most likely used by almost all JUCE-based audio plugin projects. Even standalone app fails to build so far.

If your application uses those features that are not supported by juce_emscripten, either rewrite code and disable them, or try to implement the missing bits in juce_emscripten itself. Since we already have the working foundation, implementing missing bits would not be a huge task in general (depending on the features, of course). Comparing the sources with the original JUCE would also help understanding what is under the hood.

### missing Projucer builder type

Projucer (the tool that generates the actual platform-specific project files from `.jucer`) comes with various types of project builders, but there is no Emscripten builder (yet). You most likely have to use `LinuxMakefile` builder output, then modify the output.`DemoRunner.jucer` has some [helper build output section for Emscripten](https://github.com/Dreamtonics/juce_emscripten/blob/9e209d7d65d15aa8573f1dc8ac564609f0abec3a/examples/DemoRunner/DemoRunner.jucer#L80) that brings in various build options, which would help porting other apps too.

Note that Projucer output `Makefile`, even with that helper, still needs manual modification. Here is the excerpt from the jucer project comment:

```
After resaving, do the following by hand:
Remove $(shell pkg-config ...) from the Makefile
Change JUCE library path to juce_emscripten
Add .html extension to target app
Optionally, add -s SAFE_HEAP=1 -s ASSERTIONS=1 to JUCE_CFLAGS for Debug build
Optionally, add -s WASM_OBJECT_FILES=0 to JUCE_CFLAGS and
 --llvm-lto 1 to JUCE_LDFLAGS for Release build.
```

You will have to make these changes every time you save your project on Projucer. If you find them too annoying, you might want to hack Projucer and add support for this format. Contributions are welcomed.

At this state, even the default "GUI Application" from Projucer template would not build, because it enables `juce_opengl` module which is not supported by juce_emscripten. You will have to remove the reference to it.

### missing resources

Another point to note is that you will have to add `--preload-file` option to Emscripten emcc as `LDFLAGS`. In fact, even standard ASCII characters won't be rendered if you don't embed the font files. The ported `DemoRunner` project contains references to those font files that are copied from X11 Linux desktop environment.

If you are reusing the  builder section in `DemoRunner.jucer`, your app will need those font files too. Other (non-fonts) files are not necessary.


## You can fill the missing bits

That's all. I hope you could find it possible and realistic to bring your JUCE apps to the Web.

Portting sounds difficult? It wouldn't be. I was very new to Emscripten, have never installed emsdk before, but it didn't take long time to understand them and could make some changes to juce_emscripten. It's fairly straightforward.

However, JUCE has various features and while Synthesizer V is a fully-featured app that makes use of various features, there is still a lot of missing bits to implement and rooms for improvements. But since the most significant parts, audio and GUI, are already ported by the original author and Dreamtonics, it would be possible to port your apps if you fill the missing tiny bits and hopefully contribute them back to the project, then it will become more solid JUCE foundation for everyone!

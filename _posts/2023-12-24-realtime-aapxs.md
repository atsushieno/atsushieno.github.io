---
published: true
title: "Achieving realtime safety on inter-process audio plugin extension foundation"
date: 2023-12-24 23:45:00 +09:00
tags:
- AAP
- AAPXS
- audio
- realtime
- MIDI-2.0
---

# Achieving realtime safety on inter-process audio plugin extension foundation

I have been exploring audio plugins opportunity and possibilities on Android which ends up my [AAP project](https://github.com/atsushieno/aap-core). My primary goal these months is realtime safety in extension foundation, which was finalized in the upcoming aap-core v0.8.0 release.

There should be some explanation on what that means.

## Process isolation and realtime safety

AAP (Android Audio Plugin) is unique in that audio plugins cannot be loaded as an in-process dynamic library within the host DAW. When an AAP DAW "loads" a plugin, the AAP lives in its own process, so as the DAW in its own process. Your app cannot load native executable code that was provided by another app (it seems technically you can, but Google Play Store will kick you out).

DAW and plugin needs to process audio and MIDI-like event stream together i.e. the DAW passes the plugin audio buffers and event buffers. As AAP plugins and hosts reside in their respective process spaces, we use IPC (inter-process communication) between the processes, with shared memory buffers (instead of embedding audio data buffers within the message, as they are not small and processed very frequently). It is usually a big disadvantage for audio processing; audio needs to be processed in realtime manner, but IPC foundation usually involves mutex locks that violate realtime safety.

AAP uses Android Binder IPC framework in C++ (NdkBinder) which is `minSdk = 29` since the beginning in 2019. Binder is designed to be realtime ready, implemented in the underlying Linux kernel (in its fork in the beginning of Android OS, and merged to the mainline kernel later). One thing to note is that while Binder itself is realtime ready, the realtime binder capability is not openly available to us Android app developers. But that is another story, we ignore that so far. What is important here is it is *technically* capable.

Android Binder is usually used with some interface descriptions called AIDL, and we do have one for AAP. It defines its interface like [this](https://github.com/atsushieno/aap-core/blob/2c546bfb52ce0c4130e5e26c2f322430ea8e63c5/androidaudioplugin/src/main/aidl/org/androidaudioplugin/AudioPluginInterface.aidl) (excerpt):

```
interface AudioPluginInterface {
	int beginCreate(String pluginId, int sampleRate);
	void endCreate(int instanceID);
	void beginPrepare(int instanceID);
	void prepareMemory(int instanceID, int shmFDIndex, in ParcelFileDescriptor sharedMemoryFD);
	void endPrepare(int instanceID, int frameCount);
	void activate(int instanceID);
	void process(int instanceID, int frameCount, int timeoutInNanoseconds);
	void deactivate(int instanceID);
	void destroy(int instanceID);
}
```

If you are familiar with any of audio plugin APIs (VST, AU, LV2, CLAP, etc.), you would find it quite similar to existing ones.

## Plugin extension foundation and extensibility service

AAP is designed to be extensible. An extension in an audio plugin format usually means an optional feature that plugins and hosts can use or ignore, so that the core plugin API itself does not have to introduce breaking changes. Audio plugin features are often designed to be extensible such as VST-MA (module architecture), LV2 extension APIs. AAP follows the same pattern. AAP has parameters, state, presets, MIDI, and GUI extensions, to name some.

In general audio plugin formats that support extensibility, plugin extensions usually come up with functions, and they are supposed to be invoked directly in the DAW code. However, as I explained earlier, Android DAW process and plugin process are separate, so we use Binder IPC here too. `AudioPluginInterface` interface above actually has some additional methods:

```
	void addExtension(int instanceID, String uri, in ParcelFileDescriptor sharedMemoryFD, int size);
  void extension(int instanceID, String uri, int opcode);
```

Unlike typical plugin format APIs, AAP extensions cannot be simply achieved by defining their own AIDL. Technically we can ask DAW and plugin developers to instantiate their own Binder objects for every AIDL for respective extension API, but dealing with such multiple Binder connections is quite cumbersome and also wasteful on device resources (memory, threads, shared memory handles, etc.).  Also, it is not very encouraging for DAW and plugin developers to write their code if their plugins or DAWs have to deal with this IPC foundation directly.

To sort out the situation, AAP has the extension foundation in different way: each extension API designer / developer also offers "extensibility service" implementation that handles binary serialization and deserialization of their IPC requests and replies, and DAW developers and plugin developers just deal with the extension "proxies" that AAP framework (or hosting reference implementation I would say) offers. It's similar to GRPC on top of Protocol Buffers, but runs on top of Binder. It is how `extension()` function works. To make every extension invocation typeless, their arguments and return values are serialized in the shared memory.

This way, DAW and plugin developers have to deal only with the plugin API, without needing to deal with Binder IPC by themselves. The extension API developer needs to deal with the IPC parts, but AAP v0.8.0 provides some public API facade that they can gain access to implement their requests and replies. We call it AAPXS (AAP eXtensibility Service).

## Plugin extension foundation and realtime safety

Until AAP v0.7.7, it was the only way how we invoke extension functions over the IPC. However, there was a problem.

Many of those extension functions do not have to be processed in realtime manner, but some extension functions do. For example, if your plugin format handles parameter changes, you would most likely need the changes applied within certain audio processing cycle rather than any random point of time.

Binder itself is realtime safe in theory, but (even if we have access to realtime Binder) the Binder needs to be occupied only by RT-safe audio loop. If we use the one and only Binder connection for audio processing purpose i.e. `process()` function in the AIDL above, we should not try to use the same Binder for potentially non-RT safe uses.

If we only support extension functions via Binder, they could be always invoked via `extension()` and those non-RT-safe functions will block. There will be only one invocation on one Binder connection, so simultaneous `process()` call won't happen. And considering that there should not be multiple RT-ready Binder threads running IF it were possible, no Binder method invocation other than `process()` should happen anyways.

But is it possible to pass extension invocations at `process()` ? Certainly - it is how AAP v0.7.8 introduced extension controller MIDI 2.0 SysEx8 messages. As a quick background, AAP makes full use of MIDI 2.0 UMPs as its event messages. Those note-on, note-off, control changes, program changes, etc. are directly passed to the plugin. The extension requests and replies are serialized to mere binary chunks, so it is fairly straightforward to reuse the binary data into UMP packets.

Since AAP v0.7.8, we don't invoke `extension()` method when the plugin is in "active" == realtime mode, and send AAPXS SysEx8 messages on the MIDI2 port instead.

## Async-ify plugin API or not

With AAPXS SysEx8, it becames possible to invoke extension functions within realtime processing pipeline. It is good for RT-safe extension functions. But how about non-RT-safe functions? We also need to ensure that those non-RT-safe functions do not block audio processing.

Here we need some asynchronous dispatcher that forwards the non-RT request invocation to non-RT thread. This also means, the extension function does not return within the audio processing in the `process()` cycle and thus the extension reply needs to be sent back to the host in another audio `process()` function. Every AAPXS request is assigned a request ID and the reply to that message has the corresponding request ID.

Data passing wise, the MIDI2 messaging works perfectly. AAP plugin reads through the MIDI2 input messages, detects and interprets AAPXS SysEx8 messages, and dispatches the extension request to the AAPXS "request handler" implementation that the AAPXS developer provides.

Function signature wise, it is not easy to determine what to do. In typical audio plugin format APIs, extension functions are designed to be synchronous e.g. "getState()", "setState()", "getPresetName()", "setParameter()" etc. If those extension functions should work asynchronously to work along with the AAPXS, they should become different, like a pair of "beginGetState()" and "endGetState()", or those functions with "callback" argument.

(To preserve ABI stability, we have to define extension API in C, not C++. Therefore there won't be types like `std::function` or `std::future` in the API.)

Alternatively, we could differentiate plugin extension API and AAPXS API, but that means we give up simple proxying of extension API to AAPXS. We could do that, but it is not what we have so far.



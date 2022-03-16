---
published: true
title: "JUCE on Android: fixing ClassNotFoundException on non-main thread"
date: 2022-03-16 17:30:00 +09:00
tags:
- Android
- JUCE
- JNI
---

(It was originally [written in Japanese](https://atsushieno.hatenablog.com/entry/2022/03/13/173518) but I have made some additional notes here and hence it is not a line-by-line translation.)

Through my hacks on JUCE on Android, I had been suffered from this issue https://forum.juce.com/t/env-findclass-does-not-work-inside-juce-app-very-strange/46886 ...

```
2022-03-03 18:30:48.014 6086-6086/org.androidaudioplugin.samples.pluginhost A/ples.pluginhos: java_vm_ext.cc:577] JNI DETECTED ERROR IN APPLICATION: JNI GetStaticMethodID called with pending exception java.lang.ClassNotFoundException: Didn't find class "org.androidaudioplugin.hosting.AudioPluginHostHelper" on path: DexPathList[[dex file "/data/data/org.androidaudioplugin.samples.pluginhost/code_cache/92b2cc10381543894b9e6bf969a1314aa739b976.dex"],nativeLibraryDirectories=[/system/lib, /system_ext/lib]]
```

...and finally fixed the issue after various attempts.

The problem is that JUCE cannot handle JNI invocation on Java (Kotlin, Dalvik) classes that are not part of the Android-system API or the (registered) JUCE API from non-main thread, or often even on the main thread. The class I use from my library looks for audio plugins using Android framework ("queryIntentService") on a background thread that JUCE AudioPluginHost spawns, for example. Therefore having a working thread functionality is important for me.

I created a set of fixes, and I could just explain them to those who understands the JUCE on Android under the hood. But how many there would be... the relevant JUCE code has not been updated since 2019. It would be probably more beneficial to spend more words and explain the overall technology so that more people can easily get involved in the future.

Therefore, I'm going to explain what I have discovered in JUCE on Android JNI interop foundation through this post. I am not a JUCE framework developer, so it is an external observation.


## "I think there is a bug in the framework... oh, wait..."

There are certainly JUCE issues that cause `ClassNotFoundException` on Android. But, it is often more likely that it is caused by your own code. I myself was stuck a lot because of my own wrong code.

So to not reproduce unhappy situation, I created a checklist that contains a couple of possible causes. Do these every time (or nearly every time) you make code changes and it does not seem working:

- Check if the native library is indeed built from the latest code. It might be a stale build. 
- Check if the app apk contains the class.
  - Build > Analyze APK > check all `classes*.dex`
  - If the code is Kotlin, make sure to add `apply plugin: 'kotlin-android` to `build.gradle(.kts)`
- Check if the latest apk and contents are deployed as expected.
  - Android Studio is often buggy and fails to deploy native code. "Configure" app build > "Installation Options" > "Always install with package manager (disables deploy optimizations on Android 11 and later)"
  - Best to uninstall and reinstall the apk.
- Check if `jclass` and `jmethodID` handles are correct, at least non-null.
  - name strings might be wrong.
  - jclass might be acquired from another JNIEnv. Then `Get*MethodID()` could treat the argument jclass as nonexistent.


## Analysis Pt.1: Android/Dalvik/ART

Before diving into JUCE sources, let's learn about general Android/Dalvik/ART threads.

### ClassLoader needs java.lang.Thread

The core cause of the problem is that [`juce::Thread`](https://docs.juce.com/master/classThread.html) does not follow the guidance on Android NDK documentation on JNI and threads:

https://developer.android.com/training/articles/perf-jni#threads

> It's usually best to use `Thread.start()` to create any thread that needs to call in to [Java](http://d.hatena.ne.jp/keyword/Java) code. Doing so will ensure that you have sufficient stack space, that you're in the correct `ThreadGroup`, and that you're using the same `ClassLoader` as your [Java](http://d.hatena.ne.jp/keyword/Java) code. It's also easier to set the thread's name for debugging in [Java](http://d.hatena.ne.jp/keyword/Java) than from native code (see `pthread_setname_np()` if you have a `pthread_t` or `thread_t`, and `std::thread::native_handle()` if you have a `std::thread` and want a `pthread_t`).

Once we started the thread, we cannot set an appropriate [java.lang.ClassLoader](https://developer.android.com/reference/java/lang/ClassLoader) using [`java.lang.Thread.setContextClassLoader()`](https://developer.android.com/reference/java/lang/Thread#setContextClassLoader(java.lang.ClassLoader)). Thus it is important to set up a `java.lang.Thread` for the `juce::Thread` before `startThread()` (and thus `pthread_create()`) is called.

### JNI invocation fails on Oboe audio threads too.

The missing`java.lang.Thread` issue is not specific to JUCE. The same problem occurs with [google/Oboe](https://github.com/Google/oboe/) for example. [Oboe audio processing](https://github.com/google/oboe/blob/main/docs/GettingStarted.md#creating-an-audio-stream) is done through `AudioStream`'s `onAudioReady()` callback function on its own thread, on top of either AAudio or OpenSL ES. The callback thread is created with `pthread_create()`. Since there is no JNI or JVM involved, when we later acquire `JNIEnv*` via `JavaVM::AttachCurrentThread()`, the corresponding `java.lang.Thread` is missing the sufficient ClassLoader that is active on the main thread.

(The callbacks on AAudio are created using `pthread_create()`. You can look for uses of `pthread_create()` at Android Code Search. The "Call Hierarchy" part is useful in particular.

![`pthread_create` on Android Code Search](https://i.imgur.com/YZRFNcQ.png)

I made an experiment to prove my hypothesis. I modified the "hello-oboe" example sources to acquire `JNIEnv*` and call `FindClass("androidx/constraintlayout/widget/ConstraintLayout")` at two locations: one is within some JNI function, another within `onAudioReady()` implementation. ConstraintLayout is used in the example app, and it is most likely NOT included in the main `classes.dex` (depends on r8 dex sharding). The changes I made are on this gist: https://gist.github.com/atsushieno/8e6b0bc82005cdeef65254f9e8bcec1b

The result was along with my assumption. `FindClass()` returned the expected class on the JNI invocation, and the other returned `nullptr`. It seems impossible to use JNI from within the audio callback thread.

It is actually the expected behavior. There is a GitHub Discussions thread regarding this: https://github.com/google/oboe/discussions/1418#discussioncomment-1510284

In practice, audio callbacks are expected to complete in realtime manner. Manipulating ART VM via JNI is not realtime-safe, so if you want to manipulate ART then it should be done in another thread and maybe interact with the thread using some non-thread-local memory and atomic operations.

This justification does not apply to ordinary non-realtime `juce::Thread`s.

### solution: use java.lang.Thread

As described on the Android NDK documentation that I quoted at the very beginning of this section, we should set up `java.lang.Thread`  as the right solution and implement `juce::Thread` members on top of the Java API, to not bug JNI calls on non-main threads. `java.lang.Thread` takes `java.lang.Runnable` object, whose `run()` method can dispatch to native callback and the actual C++ code implemented as `juce::Thread::run()` can be part of the native callback, after it acquired `pthread_t` ID via `pthread_self()`.

Since the actual `java.lang.Thread` implementation is a thin wrapper over pthreads in Android bionic, just acquiring `pthread_t` ID in general suffices, but some methods involve some local fields in `java.lang.Thread` internals (for example, `getPriority()` returns a Java field that is set by `setPriority()` without querying native thread priority).

Although, when it comes to aborting a thread, Android does not implement `pthread_cancel()` and therefore `java.lang.Thread.stop()` is not really implemented (it just throws `UnsupportedOperationException`) so we don't have to worry about it (!). Everyone should implement threaded processing to check cancellation flags and complete to the end.

There is no way to first acquire `pthread_t` ID via `pthread_create()` and THEN assign it to a `java.lang.Thread` instance.

### How does java.lang.Thread acquire the "right" ClassLoader?

Either JUCE or Oboe/AAudio misses ClassLoader by its "pthreads first" nature, but then how can `java.lang.Thread` be set the right ClassLoader?

Through my code reading [at cs.android.com](https://cs.android.com/android/platform/superproject/+/master:libcore/ojluni/src/main/java/java/lang/Thread.java;bpv=1;bpt=1), I figured that `Thread.setContextClassLoader()` does not really check its thread state. It is likely that the ClassLoader is retrieved (only) at `start()` and then the value would be simply ignored.

There are couple of miscellaneous information either I should share or I found through the investigation:

- On Android, we load `*.dex` instead of `*.class`. Class bytecode is not supported at run-time. Unlike `*.class`, a `*.dex` contains multiple classes so it would look more like a library file (e.g. `*.dll`) but not necessarily compiled per library units.
- If the right ClassLoader is not set, then `classes*.dex` (classes.dex, classes1.dex, classes2.dex, ...) in the apk are not loaded as expected. It seems true to ART either. (I would mention `getSystemClassLoader()` later.)
- At API Level 24, `java.*` API implementation has moved from Apache Harmony to openjdk libcore, and is mixed with ART-specific implementation a.k.a. "ojluni".
- In JNI world, ClassLoader resides in the thread local storage, so it has to remain in each thread. Various JNI invocations are done via `JNIEnv` and it has to be acquired per thread, via either `JavaVM::GetEnv()` or `JavaVM::AttachCurrentContext()`.
- If a `java.lang.Thread` does not exist when `AttachCurrentContext() was called, then a new one is created. But it is bound to the system ClassLoader whose classes coverage is quite limited.
- While ART is designed to make it possible to load multiple dex-es natively, the system ClassLoader seems incapable of retrieving them.

Here is another experiment to examine how the system ClassLoader and the fully working ClassLoader work differently. In an Android project generated by Projucer, there is a source for `com.rmsl.juce.Java.initialiseJUCE()` Java class. Try loading `com.rmsl.juce.JuceApp` class using those ClassLoaders respectively:

```
				this.getClass().getClassLoader().loadClass("com.rmsl.juce.Java");
				ClassLoader.getSystemClassLoader().loadClass("com.rmsl.juce.Java");
				com.rmsl.juce.Java.initialiseJUCE(context.getApplicationContext());
```

If you run this, you will see the second line throws `ClassNotFoundException`. This shows that the system ClassLoader is insufficient.


## Analysis Pt.2: JUCE on Android

JUCE Android support is complicated. The most recent and detailed explanation on the low-level internals can be found at:

- https://forum.juce.com/t/android-re-structuring-of-juce-s-low-level-android-code/30340
- https://forum.juce.com/t/breaking-changes-in-juce-s-low-level-android-code/30338 (linked from above)
- https://forum.juce.com/t/android-mix-match-native-and-juce-code-gui-elements/30339 (linked from above)

### A brief intro to JNIClassBase and need for Java interop

JUCE on Android often has to interoperate with Android framework. For example,  `juce_audio_devces/midi_io` 
needs access to `android.media.midi` API (there is NDK native MIDI API, but its minSdkVersion is 29). Also, `juce_events` which is used by `juce_gui_basics` provides application loop and it needs access to `android.app.Application.ActivityLifecycleCallbacks` and `android.app.Activity`.

If we use JNI on the main thread with valid ClassLoader, it is doable with no problem. But it involves a lot of boilerplate code. To avoid that JUCE has some foundation as internal C++ macros to define and use C++ JNI interop classes that correspond to those Java classes in minimalistic code.

It is actually possible to use those macros outside JUCE, but since it is not part of the public API (at least on the API reference) we should be aware that use of those macros may cause future build breakage.

Another interesting bits in JUCE JNI integration is that JUCE often has to implement abstract methods in Java interfaces (I noticed that I'm not actually sure about virtual or abstract functions in abstract classes, so I would skip mention on them so far). It requires somewhat advanced understanding on native-and-Java interoperability, similar to Java bindings in Xamarin.Android, bridges in React Native, or platform channels in Flutter. JUCE offers a minimalistic integration without binding automation, so you still have to write a lot of actual JNI interop stuff.

You can find the actual definitions in `juce_core/native/juce_android_JNIHelpers.h` and `.cpp`.

(It would be possible to implement a code generator over this foundation, but since JUCE is designed so that building a JUCE app needs to compile everything unlike other frameworks, generating code for mostly unused framework APIs would bring in massive garbages to app compilcation steps.)

### Defining a Java class (and, proxy to it) using JNIClassBase

Java objects that are based on this binding foundation and manipulated in JUCE C++ code, are defined as derived from `JNIClassBase`, using macros named `DECLARE_JNI_CLASS`, `DECLARE_JNI_CLASS_WITH_MIN_SDK`, or `DECLARE_JNI_CLASS_WITH_BYTECODE`. They all fall back to the last one.

Let's see how it is used for example, from `juce_android_JNIHelpers.h`:

```
#define JNI_CLASS_MEMBERS(METHOD, STATICMETHOD, FIELD, STATICFIELD, CALLBACK) \  
 METHOD       (findClass, "findClass", "(Ljava/lang/String;)Ljava/lang/Class;") \  
 STATICMETHOD (getSystemClassLoader, "getSystemClassLoader", "()Ljava/lang/ClassLoader;")  
  
  DECLARE_JNI_CLASS (JavaClassLoader, "java/lang/ClassLoader")  
#undef JNI_CLASS_MEMBERS
```

- The first argument to `DECLARE_JNI_CLASS` determines the class name as well as the static global variable of the declared class.
- To define the members, `DECLARE_JNI_CLASS` uses a macro named `JNI_CLASS_MEMBERS`. This macro defines the set of the members for the class, and the definition is explicitly undeclared once `DECLARE_JNI_CLASS*` is placed.
- The members definition includes `FIELD`, `METHOD`, `STATICFIELD`, `STATICMETHOD`, and `CALLBACK`. Everything is optional. The static/non-static distinction is important to determine which JNIEnv member to use (`GetMethodID()` vs. `GetStaticMethodID()` for example). The actual definitions of those macros are given at `DECLARE_JNI_CLASS_WITH_BYTECODE` and we don't have to care about them.
- This `JavaClassLoader` class is useful when we use `JavaClassLoader.getSystemClassLoader` method ID to call `JNIEnv::CallStaticObjectMethod()`.

### BLOBs in JNIClassBase

JNIClassBase often holds bytecode data as `uint8[]` constant:

```
static const uint8 invocationHandleByteCode[] =  
{31,139,8,8,215,115,161,94,...,12,5,0,0,0,0};  
  
(..)

DECLARE_JNI_CLASS_WITH_BYTECODE (JuceInvocationHandler,
   "com/roli/juce/JuceInvocationHandler", 10, invocationHandleByteCode, sizeof (invocationHandleByteCode))
```

It is in fact a gzip-ed dex bytecode. If it is specified, the `JNIClassBase` loads the class using either `InMemoryDexClassLoader` or `DexClassLoader` (depending on the SDK version in use). Those BLOBs in JUCE sources come up with the corresponding Java sources so it is not to hide source code (as far as I know).

Those dex BLOBs take place where otherwise app builds would have to involve Java compilation in Android Studio / Gradle but only resort to C++ compilation. I don't find technical advantage beyond minor build setup complication, but it is how JUCE works...

As part of this investigation, I created a JUCE patch that strips out all those BLOBs but expects `build.gradle(.kts)` to include those Java sources into its Java compilation step: https://gist.github.com/atsushieno/1ab9f9d4a1f9119db1d2ac88b4257bcb

If we choose my approach to compile Java sources over existing one, `build.gradle(.kts)` needs explicit source directory specitication to those directories. It is actually [already done by Projucer](https://github.com/juce-framework/JUCE/blob/2f980209cc4091a4490bb1bafc5d530f16834e58/examples/DemoRunner/Builds/Android/app/build.gradle#L83) as part of current `build.gradle` generation *for core stuff*. It has to be extended to other source directories. [My AudioPluginHost build](https://github.com/atsushieno/aap-juce-plugin-host/) for Android with the patch above comes with:

```
main.java.srcDirs +=  
 ["../../../juce-modules/juce_core/native/javacore/init",  
     "../../../juce-modules/juce_core/native/javacore/app",  
     "../../../juce-modules/juce_gui_basics/native/javaopt/app",  
  
     "../../../juce-modules/juce_audio_devices/native/java/app",  
     "../../../juce-modules/juce_core/native/java/app/",  
     "../../../juce-modules/juce_gui_basics/native/java/app/",  
     //"../../../juce-modules/juce_gui_extra/native/javaopt/app",
     //"../../../juce-modules/juce_product_unlocking/native/javaopt/app",
     "../../../juce-modules/juce_opengl/native/java/app/",  
     //"../../../juce-modules/juce_video/native/java/app/"  
  ]
```

Determining which directory to include is kind of annoying so that Projucer should automatically handle. Compiling all those Java code is another option, but some of those modules come up with extraneous dependencies such as Firebase (for juce_product_unlocking) which is annoying. It is also why some lines in the above snippet are commented out.

If we choose to [use CMake to build Android app](https://atsushieno.hatenablog.com/entry/2021/01/16/020602), then the problem becomes simpler as only the app developers could care about their own `build.gradle(.kts)`.

Note that use of those BLOBs does not improve build performance. Regardless of those extra Java sources, Java compilation step exists (you have `JuceApp.java`), and in most cases the actual compilation does not happen as those sources don't change.

### Use JNIClassBase at run-time

`JNIClassBase` is initialized when `Thread::initialiseJUCE()` is invoked (which is usually done by `com.rmsl.juce.JUCE.initialiseJUCE()`). All `JNIClassBase` implementations are registered to the static local storage returned by `JNIClassBase::getClasses()`, by `DECLARE_JNI_CLASS*` macros. So, after `Thread::initialiseJUCE()`, any `JNIClassBase` instance created later are not initialized (they might still work if `initialise()` is somehow done). If BLOB is specified, then they are all loaded at that time and stored as classes in memory (or temporary file depending on the sdk version in use).

### Declaring classes with JNIClassBase that are not declared in JUCE

If you add `#include <juce_core/juce_android_JNIHelpers.h>` in your source code, the macros should be available. If your class declaration is wrong and does not match the actual Dalvik class, it will cause run-time error as `JNIClassBase` actually tries to find the members using `JNIEnv` functions.

As the result of this investigation, I created a patch that makes use of `java.lang.Thread` to implement `juce::Thread` internals, and I declared `JavaLangThread` class using `JNIClassBase` there. It ends up to stay within `juce_android_JNIHelpers.h` after all, but it should work if I put it in my own source file, as long as the macro and classes are available within the scope of the compilation unit.

### AndroidInterfaceImplementer

Okay, it's been a long section, but now that we are done with the first half of the issues. The latter issue was about implementing abstract functions in Java interface, in C++. For example, `java.lang.Thread` constructor takes a `java.lang.Runnable` object as an argument, and we have to come up with an instance of this interface and implements its `run()` method to dispatch to our C++ function (or lambda).

There are couple of ways to provide such missing Java interface implementations. JUCE in particular makes use of `java.lang.reflect.Proxy` and `java.lang.reflect.InvocationHandler` and involves no build-time code generation.

A Proxy acts like a Java object of the associated interfaces, but when a method of them is invoked, it makes use of `invoke()` method of the `InvocationHandler` object that was passed at its constructor:

```
abstract Object invoke(Object proxy, Method method, Array<Object> args);
```

The `InvocationHandler` is passed to `java.lang.reflect.Proxy.newProxyInstance()`:

```
public static Object newProxyInstance (
    ClassLoader loader, Class[]<?> interfaces, InvocationHandler h);
```

JUCE has `com.rmsl.juce.JUCEInvocationHandler` as an implementation of this interface, which dispatches `invoke()` to the corresponding C++ function (this Java class is embedded in `juce_android_JNIHelpers.cpp` as a gzip-ed dex bytecode as explained earlier). It is an implementation details that we JNI users have to care about (it will show up on your app stacktraces when your JNI call fails, but now that I explained it here you don't have to wonder what it is).

In this JUCE source file there are functions named `CreateJavaInterface()`. It invokes `Proxy.newProxyInterface()` via JNI and returns the Proxy object. Then we can simply use the object just as an interface implementation. There are handful of the function overloads but let's have a loot at one of those:

```
LocalRef<jobject> CreateJavaInterface (AndroidInterfaceImplementer* implementer,  
  const StringArray& interfaceNames,  
  LocalRef<jobject> subclass)
```

The argument `AndroidInterfaceImplementer` class, a new face here, is used as a C++ base to implement the relevant Java interfaces. When we implement the Java methods over this proxying feature, we override `AndroidInterfaceImplementer::invoke()`. For example, `MediaScannerConnectionClient` class in `juce_android_Files.cpp` implements this like:

```
class MediaScannerConnectionClient : public AndroidInterfaceImplementer
{
    (...)
    jobject invoke (jobject proxy, jobject method, jobjectArray args) override
    {
        auto* env = getEnv();

        auto methodName = juceString ((jstring) env->CallObjectMethod (method, JavaMethod.getName));

        if (methodName == "onMediaScannerConnected")
        {
            onMediaScannerConnected();
            return nullptr;
        }
        else if (methodName == "onScanCompleted")
        {
            onScanCompleted();
            return nullptr;
        }

        return AndroidInterfaceImplementer::invoke (proxy, method, args);
    }
};
```

This `invoke()` works just like `java.lang.reflect.Proxy.invoke()`, so you dispatch to individual C++ implementation for each interface method by name and args. (You may have multiple interfaces to implement and your `invoke()` may have to resolve method conflicts by yourself.)

## Fix: rewrite juce::Thread implementation using java.lang.Thread

The lengthy analysis and explanation is done. Let focus on the present issue. What we want to achieve is working `juce::Thread` implementation that does not break JNI invocation.

As explained at Analysis Pt.1, if we use `pthread_create()` in C++ region, it seems impossible to associate it with `java.lang.Thread` and acquire and set appropriate ClassLoader. Then, instead of using `pthread_create()`, we could instantiate `java.lang.Thread`, pass a `Runnable` implementation with our own proxy (`AndroidInterfaceImplementer` implementation), and in the `invoke()` implementation we retrieve `pthread_t` ID using `pthread_self()` and continue with the user-defined `run()` implementation on the `juce::Thread` instance.

Here is the patch: https://gist.github.com/atsushieno/8e176beca9d6fd4ea91a6953838195b6

BUT, it was not sufficient. There is another issue in the existing Proxy that its ClassLoader is not fully aware of multiple dex-es, which was explained around the end of Pt.1. We have to somehow acquire the "right" ClassLoader. So here is another patch: https://gist.github.com/atsushieno/eff946b6daf8897eb01ec76068154c17

It retrieves the ClassLoader from `com.rmsl.juce.Java` instance, which seems sufficient.  Though, this class is not available at `JNI_OnLoad()` whereas it calls `JNIClassBase::initialiseAllClasses()` (which needs the working ClassLoader), I had to move the call to `initialiseAllClasses()` to `Thread::initialiseJUCE()`. I believe it makes more sense.

The second patch is unnecessary if you go further and apply my forementioned BLOB removal patch, but since my patch does not come up with Projucer change, the resulting Android gradle projects won't work without user tweaks.


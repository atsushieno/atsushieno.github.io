---
published: false
title: Instant music CD release experience for an indie music festival
---
I had been quiet after attending to Audio Developer Conference (in English). I had been enjoying Europe after 10+ years of absence there. Then I changed something in life... I had been busy to become a songwriter. It's going to be somewhat long story.

## I'm not interested in long tale, just show me the outcome

These are the trailers on soundcloud and youtube. They are my first video and song uploads...!

<iframe width="100%" height="300" scrolling="no" frameborder="no" allow="autoplay" src="https://w.soundcloud.com/player/?url=https%3A//api.soundcloud.com/tracks/579425697&color=%23ff5500&auto_play=false&hide_related=false&show_comments=true&show_user=true&show_reposts=false&show_teaser=true&visual=true"></iframe>

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/_-QcC_td3lM" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

All those song files at MIDI level are published to my [github repo](https://github.com/atsushieno/mugene/tree/master/samples/mugene-fantasy-suite).

## So WTH did you become a composer?

It is surprising maybe to those who know me somewhat well. When I attended to ADC2018, I talked to some random people and then I shot question around, like "do you compose music?" And the answers were _mostly_ "yes".

I felt incompetent. They were mostly songwriters and they were mostly developers at the same time. I wasn't. I have been developing MML (music macro language) compiler which is all about songwriting in text, but I didn't produce music with it... at least, a lot. I did some, but it was like in 199x and then I only "ported" some old compositions to my own language. That was kind of dogfooding, but I should have done more of those. I wasn't indeed much confident with my own principles "I can compose songs with my own toolchains".

I thought that I should begin something. The first thing I tried was to attend to something like [M3](http://www.m3net.jp/), one of the biggest indie music festival which inherits in-house Comic-Market-like culture, with 10,000 visitors and 1000-ish exhibitors. I had some experience with that kind of "festival", so I thought it would be good for me. It was in late November and the next M3 was too far away. I wanted to begin with "easier" one for entry level composer.

That's how I found http://gensouongaku.info/fes/ - it was a festival specific to "Fantasy" kind of music. It comes with "you define what 'fantasy' means" statement. It practically meant something like soundtracks (like "the Lord of the Rings"), VGMs (Final Fantasy series), even storytelling (in Japangrish we call them "voice drama CDs"), can be literally anything...

That was kind of familiar to me. I applied for it with little-to-no thoughts. After a few weeks, the organizers stated that they expanded the space and all the applicants were accepted. There were about 90 exhibitor groups in the end, including mine.

I was under pressure. I hadn't really compose anything for like 15+ years. And now I have to compose something only by myself. I used to write some songs with a MIDI device (Roland SC-8820) when I was a student, but that was ages ago. In later life I often tried to learn modern DAWs like FL Studio, as well as Komplete, and tried to learn them using Windows, but I didn't make it with Windows after all. I thought I could dump and run away... (then the organizers will mark me as "should be dropped for future festivals")

Fortunately I had certain amount of time. I could spend all my time for learning how to compose songs using modern software, as well as composition basics. It was post-ADC time and I had a lot of interest in the new technology, but I decided to concentrate on songwriting. To dive into this kind of software industry, I have to learn those technologies anyways.

## Determining basic directions

At that state, everything was left in blank. I could take any approach to produce a "music disc" (that was what I wrote in my application submission: "I will come up with a music disc, sources for them, and how-to book about it)"... I had my MML compiler, but I was not really confident enough that it's usable for "practical" use for daily songwriting. I played with [Tracktion Waveform](https://www.tracktion.com/products/waveform) a bit (because it was one of those emerging DAWs that support Linux desktop) but not much.

In any case, I began with "playing with sounds" i.e. listen to sound and familiarize myself, either with my old MIDI toy SC-8820, or VST plugins like Collective on Waveform. It was like, I ended a day just with only four bars with one simple string chord as outcome. Then I started playing with Waveform more, by importing various MIDI song files that I used to create (they were mostly copy of some metal bands like Dream Theater, or similar Japanese prog-rock bands), altering MIDI program changes to VSTi to get familiarized with modern sound.

At that stage, I only knew General MIDI programs, but it was actually more than what Collective has. I was like "why does it not come with piccolo...", and I often fell back to [juicysfplugin](https://github.com/Birch-san/juicysfplugin) + FluidR3_GM.sf3. I seemed a dead-end to me after all...

At that time, I planned to play with [JUCE](https://github.com/WeAreROLI/JUCE/) and [tracktion_engine](https://github.com/Tracktion/tracktion_engine/issues), but I found it nearly impossible because JUCE on Linux is quite featureless (no VST3, no Lv2, poor UI implementation etc.). Therefore I ended up to set up a new Mac environment. So I decided to 1) basically compose songs using MML on my daily Linux desktop, and 2) bring MIDI songs to Waveform and continue on Mac with Kontakt etc. After all, it was a quite successful decision. It was already past new year days.


## Using MML for daily composition

I haven't really explained what MML is. Music Macro Language is a text program-like syntax that is used to describe songs by operations, which are compiled to some song files. It was popular in 20th century, and then modern DAWs took place in DTM world. There was no dominant song data format, so as no dominant MML syntax - just like programming languages, there were various syntaxes. It was because, the output can be either FM/PSG chip (mostly ones from YAMAHA), MIDI, or even BEEP sound. My compiler is to generate Standard MIDI file.

The examples below are typical MML operations:

- `c`, `d`, `e`, `f`, `g`, `a`, `b` : they are "notes" in sol-fa (do, re, mi...).
- `r` : rest
- `o4` : set octave to 4
- `>` : shift octave (either up or down, depending on the syntax definition)
- `V100` : set channel volume to 100.
- `l8` : set "default length" to 8 (eighth note)
- `[` ... `]4` : repeat from `[` to `]`, 4 times.

There is usually a lot more operations (I defined like 250+ in my compiler syntax) and it's up to the output module (FM sound drivers and MIDI are totally different beast).

Using this classic technology has its own advantage - it works very well with flexible text editor operations, version controls like git, and often features like [Language Server Protocol](https://langserver.org/). But apparently it's not for everyone - its learning curve is quite steep...

In any case, when I decided to use my own MML compiler (to both ease composition as well as dogfooding), I had a handful of tools to make it possible:

- [mugene](https://github.com/atsushieno/mugene) MML compiler: the compiler toolchain. I actually (ab)use it in my various projects e.g. [fluidsynth-midi-service](https://github.com/atsushieno/fluidsynth-midi-service) to ensure that it is also usable to play arbitrary MIDI songs.
  - It comes with LSP (which somehow does not seem to be working right now...) that implements basic operations like "go to definition".

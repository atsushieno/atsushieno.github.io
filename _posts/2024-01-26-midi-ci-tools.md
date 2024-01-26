---
published: true
title: "Understanding MIDI-CI tools"
date: 2024-01-26 20:00:00 +09:00
tags:
- ktmidi
- MIDI-CI
- MIDI-2.0
---

# Understanding MIDI-CI tools

These months I had been working on my latest project, [**ktmidi-ci-tool**](https://github.com/atsushieno/ktmidi/tree/main/ktmidi-ci-tool). I have written [another blog post](https://atsushieno.github.io/2024/01/26/ktmidi-ci-tool-released.html) on the development itself, but since MIDI-CI itself is quite unknown at this state, I thought we need some explanation on what MIDI-CI tools are for, what we can achieve with MIDI-CI tools, and how to use them. Hence, I will be explaining a lot of the specification and the tools with lots of words this time.

## evaluating MIDI-CI implementation interoperability

MIDI-CI is about interoperability, and there are some MIDI-CI implementations. So, why not try to connect each other? We should make it clear *what* and *how* they implement though. Here are the list of implementations I know of (listing from most featureful ones):

- JUCE [**CapabilityInquiryDemo**](https://github.com/juce-framework/JUCE/blob/develop/examples/Audio/CapabilityInquiryDemo.h): based on juce_midi_ci, it exposes almost all MIDI-CI  features it supports. It can define its own profiles and properties to test its receiver functionality. No Process Inquiry.
- [**ktmidi-ci-tool**](https://github.com/atsushieno/ktmidi/tree/main/ktmidi-ci-tool): it is build upon similar mindset to JUCE CapabilityInquiryDemo provides, with additional features like in-place value text editor, Process Inquiry support, runs on mobiles and web browsers.
- [**MIDI 2.0 Workbench**](https://github.com/midi2-dev/MIDI2.0Workbench): has various "checklist" items to verify if the connected device correctly implements those MIDI-CI features.
- Apple [**CoreMIDI**](https://developer.apple.com/documentation/coremidi/midi_capability_inquiry) provides MIDI-CI features. No Property Exchange, no Process Inquiry.

MIDI 2.0 Workbench and ktmidi-ci-tool provide virtual MIDI ports so that if the other device (such as CapabilityInquiryDemo) does not come up with any MIDI ports it can still connect to those tools. (All those tools should not be tied to the actual MIDI connections, but everything - including ktmidi-ci-tool - so far expects MIDI connections.)

Among those implementations, ktmidi-ci-tool and JUCE CapabilityInquiryDemo are fully featured MIDI-CI tools that is enough to check almost all MIDI-CI implementation behaviors on both Initiator and Responder sides of a MIDI-CI device. Since JUCE CapabilityInquiryDemo is "just" a demo app, it does not come up with any users manual, sort of. Since MIDI-CI is not something every musicians or even music app developers understand well, the UI would look totally alien. Let's try to understand what they do so far.

These tools by themselves would be primarily demonstrating what you can achieve when you integrate those MIDI-CI features into your real-or-virtual MIDI devices, like synthesizers, effectors, controllers, etc. Such an actual product would not expose the same kind of UI as these tools. Instead, those tools would be useful to connect to your MIDI devices (or anyone else's).

## Connect on JUCE CapabilityInquiryDemo

Here is the main UI. I shrunk it to small size as it's almost blank.

![CapabilityInquiryDemo main window](https://i.imgur.com/FsgZXKR.png)
MIDI-CI assumes there is a pair of MIDI input and MIDI output.

Conceptually MIDI-CI is merely based on a binary I/O agents i.e. an **Initiator** and a **Responder**. They do not necessarily have to be MIDI devices on the platform, but CapabilityInquiryDemo expects them so far. It is similar to client and server, but since MIDI-CI is bidirectional, sometimes a device that acted as an Initiator might also work as a Responder when the other party sends a request to it. These labels are used per request (**Inquiry** and **Reply** in typical MIDI-CI names) basis.

When you use this tool, it is mostly for manipulating another MIDI-CI device (either really or virtually, like ktmidi-ci-tool), so you would need to choose the right pair of the MIDI in and out ports of the same device. Some MIDI devices would provide multiple in and out ports, then you need to pick up the right I/O ports. In the future this tool might present different form of ports selector for MIDI 2.0 support, but let's skip that so far.

Some other tools, such as MIDI 2.0 Workbench and ktmidi-ci-tool, also offer "virtual MIDI ports" to let other MIDI-CI devices connect from it without hassle, making connection tests easy. You can do it with JUCE, but CapabilityInquiryDemo doesn't, so far. (It is doable only on modern platforms such as Linux, MacOS, and iOS though. Windows developers will have to wait for [Windows MIDI Services](https://github.com/microsoft/MIDI/).).

The sshot below is where ktmidi-ci-tool configures MIDI port connections, but you don't have to choose anything if you are connecting it to CapabilityInquiryDemo and that app chooses "KtMidi-CI-Tool Virtual In/Out ports":

![ktmidi-ci-tool MIDI device seletion](https://i.imgur.com/2zbSOCd.png)
### Save and Load the state on ktmidi-ci-tool

JUCE CapabilityInquiryDemo saves the state when it quits the application (or it might be doing that at any state change). Our ktmidi-ci-tool does not always do that - you need to explicitly perform "Save configuration" to save any local configuration. At this state there is no way to load arbitrary configuration file yet (cross-platform support messes it).

## Discover our device(s) on JUCE CapabilityInquiryDemo

The next step would be to discover your device from CapabilityInquiryDemo. Go to "Discovery" and click "Discover Devices".

![CapabilityInquiryDemo Discovery tab](https://i.imgur.com/2mi6svd.png)
This tab shows information about and controllers for the "remote" "connected" devices. Clicking "Discover Devices" button publishes "Discovery Message" MIDI-CI message to the connection.

It would usually result in only one "Reply to Discovery Message" message, but there can be many replies from multiple **Endpoints**. I will explain what an Endpoint is later, but for now take it as an operation target MIDI-CI entity. This tab shows the details of only one Endpoint, so if you need to control another Endpoint, choose the device from top-right combo box that shows current device's MUID and device name (if available).

It should also be noted that there can be multiple "MIDI-CI devices" that replies to the Discovery Message. For example, your connection target may be a MIDI-CI "hub" kind of device that connects to multiple MIDI-CI device, or a virtual MIDI port that shoots Discovery Inquiry on different transport protocol such as [Bonjour](https://en.wikipedia.org/wiki/Bonjour_(software)).

At the first sight the "Discovery" tab looks like it shows only "Basics" and "Profiles", but it also supports "Properties" if you scroll down.

## The initial messaging interactions

The "Discover Devices" click event itself only triggers MIDI-CI initiator messaging of "Discovery Message". But when the client received a "Reply to Discovery Message", a lot of subsequent messages are sent to the responder:

- **Endpoint Message**: it asks for the details about the Endpoint. We will discuss it soon.
- **Profile Inquiry**: if the Responder supports Profile Configuration, it asks for currently available profiles on the MIDI-CI device, either enabled or disabled. We will discuss Profile Configuration later.
- ** Get Property Exchange Capabilities Inquiry**: if the Responder supports Property Exchange, it will first start asking about its Property Exchange Capabilities, namely the max number of concurrent property requests. We will discuss Property Exchange later.

And if the responder is CapabilityInquiryDemo or probably any juce_midi_ci based MIDI-CI device, this Initiator will receive the same set of these messages above from the Responder as well(!). It is because the Responder kind of need to know what this connecting MIDI-CI device can do. If you go to the "Logging" tab, you can see how it is interacting with the other device:

![CapabilityInquiryDemo logging for the first sessions](https://i.imgur.com/2ZrUehs.png)
Let's focus on the replies to each of those further requests. The replies are unordered.

(1) It receives a **Reply to Endpoint Message**. It is supposed to contain *various* information about the endpoint, but as of current MIDI-CI v1.2 specification, it only contains the "product instance ID" of the MIDI-CI device.

This ID is usually a serial ID of the device, if applicable. Sometimes more than one devices of the same product are connected, and we often need to identify those different instances of them. Their MUIDs are reset every time the device restarts, so they cannot be used as their identifiers when we need to identify them across the MIDI-CI session.

(2) It receives **Reply to Profile Inquiry** messages, per address and group. There can be many MIDI-CI Profiles defined on the MIDI-CI device, and they all have to be returned. It is due to a glitch between how MIDI-CI profiles would be defined and how one Reply to Profile Inquiry message contains multiple profiles:

- When it is being defined, a profile ID would be first defined and the target channels are specified as its details
- When they are represented in the messages, they are grouped, each reply message is bound to at most one "address", by common MIDI-CI message definition

![CapabilityInquiryDemo multiple profile replies on the logs](https://i.imgur.com/tRrtEfw.png)
The profiles replies are visible on the other side. ktmidi-ci-tool logging screen looks more verbose on the log record height - it is because the tool also targets mobile screens (Android and iOS) where I want to avoid expanding horizontally:

![ktmidi-ci-tool profile replies logs](https://i.imgur.com/Ggd22qA.png)
(3) It receives a **Reply to Property Exchange Capabilities** message. It contains (a) the number of the number of simultaneous Property Exchange requests supported, and (b) the Property Exchange version (major and minor). (b) needs to be clarified to determine if the Initiator can actually send further PE requests that could be understood by the Responder. (a) also needs to be clarified to limit the number of parallel Get Property Data Inquiries.

## Profile Configuration

**Profile Configuration** is a feature that a MIDI-CI device declares that it supports a set of features that can be enabled or disabled by the connected Initiator.

Profiles are defined either by MMA/AMEI, as a **Standard Defined Profile**, or by each MIDI device manufacturers. A MIDI-CI Profile ID is a 5-bytes array, and is used to identify them.

According to Common Rules for MIDI-CI Profiles (M2-102-U_v1-1) specification section 2.2, if the first byte is `7Eh` then it is a Standard Defined Prpfile. There is only one Standard Defined Profile as of 2023 though: Default Control Change Mapping Profile specification (M2-113-UM_1-0). For MIDI-CI tools, it is not very important specification (take it just an example profile). It should also be noted that the last byte is used to identify the "Profile Level" which has the fixed set of values, according to the specification, section 3.2.

Defining a Profile and defining the set of Profiles on an Endpoint are different work, but defining a Profile in terms of Profile Configuration is not very different from defining the Profile set. You specify a Profile ID and its target (available) channels. That's all. You cannot even name a profile. Here is how you define the set of Profiles on JUCE CapabilityInquiryDemo, on "Local Configuration" tab:

![JUCE CapabilityInquiryDemo: defining Profiles](https://i.imgur.com/9BLeEyZ.png)
A Profile is defined and applied per channel basis, often on all channels in a **Group**, or even all channels in all Groups in a **Function Block**. The term Group is a MIDI 2.0 concepts that expands total number of channels from 16 to 256 - there can be 16 groups on a MIDI 2.0 device, and each one groups 16 channels. A Function Block consists of one or more Groups that acta as an **Endpoint**.

This explains about 50% of the Channel/Group diagram in the screenshot above - each Profile needs to be enabled on particular channels. Each row represents a Group, and the right-end one represents the Group. Those "active" boxes, including the large "Block" bar at the bottom, indicate where they are registered. If you want to register a Profile target channel but want to have it disabled by default, switch to "Show Active Channels" mode and click "Toggle Member Channels". This corresponds to "Set Profile Off" operation from the Initiator, in case you are familiar with messaging.

![JUCE CapabiityInquiryDemo Set Profile Off](https://i.imgur.com/dDDfBiy.png)

Every MIDI-CI message comes with the common message header part (juce_midi_ci calls it "message header", and ktmidi-ci calls it "common" to avoid confusion with Property header) and it contains an **address** field, which is to indicate the channel being addressed. For Profile related messages, the channel part I explained above is set to this "address" field. The address field is one-byte, and unlike channel part in a MIDI channel voice message, `00h`-`0Fh` indicates the respective channel, or `7Eh` indicates the entire Group, or `7Fh` indicates the whole Function Block.

(Personally I have been playing around it and have some extraneous Profile entries on non-zero Groups, but I do not recommend that over MIDI 1.0 connections, as MIDI 1.0 cannot handle Groups and JUCE CapabilityInquiryDemo sends messages related to those Groups without specifing it anyway, causing various weird-looking problems.)

ktmidi-ci-tool has quite different UI for defining the Profile set:

![ktmidi-ci-tool Local Profiles](https://i.imgur.com/hNHcWrn.png)
It has the list of Profiles on the left side, with add/edit controls, but it does not show that nicely drawn set of channel/Group boxes that JUCE has. Instead, it has boring per-line Profile addressing definitions. The pseudo "Ch.Grp." column has a dropdown list that can choose 0-15 channel, or "Group" or "Function Block". It's easy to enable and disable the active Profiles there. There is no "delete" button on the Profiles list - if all those profile address lines are gone then it vanishes too. It follows the way how Profile Configuration messaging works.

(The "+" button the Profile addressing list is disabled. Clicking the button adds a new entry whose target address defaults to "Function Block". If there is already a definition for Function Block like on this screen shot, then it avoid duplicates.)

When you connect to a MIDI-CI device that supports Prrofile Configuration on JUCE CapabilityInquiryDemo, the client side looks like this:

![JUCE CapabilityInquiryDemo: controlling remote Profiles](https://i.imgur.com/ZwYnvsW.png)
On the client side, the channel/Group diagram is looking simpler, as there is no need to toggle "definitions". It only shows the defined Profiles that are sent back from the Responder as a set of Reply To Profile Inquiry messages, rearranged to fit with the diagram. From there, you can toggle "enablement" (this term is also used in `juce_midi_ci` module code) on each address (channel/Group).

ktmidi-ci-tool UI is similar to how we define local Profiles:

![ktmidi-ci-tool controlling remote Profiles](https://i.imgur.com/ouzDfnk.png)
It has an additional feature "Details Inquiry", which you can send a **Profile Details Inquiry** over the specified channel, with the "target byte". A Profile Details Inquiry is a message to the Endpoint to give back more details about the Profile, identified by a byte key (0-127). The reply is not shown on the app UI itself, but the **Reply To Profile Details Inquiry** message will be on the Logging screen.

The Profile Details target is defined as either a **Registered Target** (`00h`-`3Fh`) or **Profile Specific Target** (`40h`-`7Fh`). The Registered Targets are to be defined by MMA/AMEI specifications. Having it aside "Profile Specific" means, they are common to any Profile. There is already one definition in Common Rules for MIDI-CI Profile specification - `00h` is defined as "Number of MIDI Channels". It is kind of related to the concept of "multi-channel Profiles" defined in Common Rules for MIDI-CI Profiles specification, but this concept is not very well tied to the messages that MIDI-CI tools deal with for now, so I'm skipping it. If you are curious, read that specification, namely section 2.3.


## Property Exchange

**Property Exchange** is a feature that a MIDI-CI device declares a set of "resources" (take it as "properties") in MIDI-CI manner, and let clients get and set the resource data, from primitive bytes up to JSON based object manipulation, in either the "full" content, or "partial" portions of it.

Property Exchange in MIDI-CI specification v1.2 itself is type-system agnostic and does not mention JSON, nor even metadata i.e. it does not indicate how properties are defined. The specification states that every Property Exchange messages consists of a **header** and **body**, and properties can be "get", "set", "subscribed", and notified.

### Common Rules for PE

But apparently, that's not enough. For a property and/or type system that makes sense, metadata system is an essential part. Apart from MIDI-CI specification itself, there is another specification that defines it: **Common Rules for MIDI-CI Property Exchange** (I will mention it as Common Rules for PE). According to the Common Rules, properties can be retrieved in its own JSON format, and optionally be formalized using JSON Schema (not mandatory). If you request the Responder for its **ResourceList** resource (property) using **Get Property Data** message, the Responder will return a **Reply To Get Property Data** message that contains a JSON array of those defined resources (properties).

Why is there a separate specification? There must be some reasons, but it is worth mentioning that unlike MIDI-CI itself Common Rules for PE is not something everyone would agree. MIDI and MIDI-CI themselves are realtime ready (as MIDI-CI is just on top of SysEx), but JSON based metadata system isn't. JSON is string based data format, unlike binary based message packets such as Protocol Buffer which can statically resolve all types and members. You can still build your own formats apart from the Common Rules, but since it is not "common", other parties would not understand your format.

It should be also noted that Property Exchange itself is actually type-system agnostic, so MIDI-CI app developers can still Get and Set property data without knowing which properties are being manipulated if the some predefined byte chunks are provided as the property headers. No Common Rules are applied in that case, and it's still interoperable.

Having separate specification has another advantage: MIDI-CI specification itself has updated in June 2023, but this Common Rules for PE has unchanged since December 2020. There could be future specifications that could replace this Common Rules i.e. that works with MIDI-CI Property Exchange but provides better and potentially realtime safe metadata system(!).

### Defining Properties

To try Property Exchange messaging, you would first need to define a set of properties on the Responder. If the responder is JUCE CapabilityInquiryDemo, then switch to "Local Configuration" tab *and scroll down*.

![CapabilityInquiryDemo Properties](https://i.imgur.com/xWO3bEL.png)
There is this "Properties" section down next to the Profiles. Enable it by the checkbox, and go to the bottom of the tab page to find "+" and "-" buttons. Then click "+" to add a new property.

![CapabilityInquiryDemo add a new property](https://i.imgur.com/3Hm11dA.png)
It is at an *invalid* state. There is a blank-looking line on the left-bottom box, but it actually works as the property list. It looks empty because the "Name" field is left blank. Select the textbox and give a name.

At this state, I would mention that Common Rules for PE states that an end-user-defined resource name should begin with "X-" as non-X- resources are reserved by MMA/AMEI. That may sound quite arrogant and you might want to ignore, but some tools such as [MIDI 2.0 Workbench](https://github.com/midi2-dev/MIDI2.0Workbench) reports an error if such a non-X- resource is "settable" (`canSet` != `none`).

In any case, once you give a property name, it becomes valid state. You can continue to customize the rest of the properties, but if you have a look at the property details, you would not understand what they are:

![JUCE CapabilityInquiryDemo local property details](https://i.imgur.com/r3YWRGP.png)
Is it understandable on ktmidi-ci-tool? Not really either:

![ktmidi-ci-tool local property details](https://i.imgur.com/6BPTbhd.png)
Looks like everything is just counter-intuitive. It is. So far, Skip these details and only make these changes:

- On "canSet", select "partial" (there are "none", "full" and "partial")
- Check "canSubscribe"
- If you are using ktmidi-ci-tool, click "Update Metadata" to reflect the changes

Your Initiators can then make changes to the property value (`canSet`) and observe the property value changes (`canSubscribe`).

The next thing is to set the property "value" locally. On JUCE CapabilityInquiryDemo, there is a "Value" tab on the property details:

![CapabilityInquiryDemo local property value](https://i.imgur.com/El1EXfb.png)

You can set the property value here, but not on the text edit field directly(!). You can update the value only by uploading a file. Click "Set Full...". It shows an open file dialog, and if you pick a file, then its content is set as the property value. At this state you want to use only JSON file. Maybe in the real world there would be no chance to edit the text content directly and this UI makes sense, but you might find it absurd especially when you are working on your own product and want intuitive manipulation experience...

Property value editor on ktmidi-ci-tool looks different. If you click the "edit" checkbox, it shows some additional buttons etc.

![ktmidi-ci-tool local property value](https://i.imgur.com/ws74k7S.png)

There is "Set value by file (choose)" button which just works like JUCE CapabilityInquiryDemo. In addition, it supports in-place text editor IF the property media type is `application/json`. Ideally there should be some JSON validator there...

### Manipulating Remote Properties

OK, now that we have defined local properties, it's time to try connecting from another tool. This is what happens when ktmidi-ci-tool connects to CapabilityInquiryDemo and sends Discovery:

![ktmidi-ci-tool sends Discovery over CapabilityInquiryDemo](https://i.imgur.com/ffbFoLn.png)

These tools turn your local property definitions into the "ResourceList" meta property that MIDI-CI Property Exchange implementations (based on Common Rules) will query in the first place. In this case, after the first series of messaging that began with Discovery is done, the Initiator ktmidi-ci-tool sends Get Property Data message to the Responder CapabilityInquiryDemo, for "ResourceList" resource.

You can check how things work on the "Logging" tab on CapabilityInquiryDemo (from Responder's point of view), or "Logs" tab on ktmidi-ci-tool (from Initiator's point of view).

On the other hand, this is what happens if CapabilityInquiryDemo acts as the Initiator that connects to ktmidi-ci-tool.

![CapabilityInquiryDemo sends Discovery over ktmidi-ci-tool](https://i.imgur.com/gKKKXJP.png)

From this CapabilityInquiry Discovery tab, we can update the property value as a client. The UI looks similar to what we did as a local property.

![CapabilityInquiryDemo property value update as a client](https://i.imgur.com/oXK9Cuy.png)

We have different set of controls though. First, there are "Get" and "Subscribe" buttons. "Get" can be used to retrieve the value, and "Subscribe" registers this client as a subscriber. When we register a client to a Responder as a Subscriber, it will receive property change notification messages until the subscription ends. You can experience it by following these steps below:

- On CapabilityInquiryDemo, click "Subscribe" button
- On ktmidi-ci-tool, select the property, click "edit" checkbox, enter some number or any string that works as a valid JSON (e.g. `"a quoted string"`, or `{}`), and click "Commit changes"
- Go back to CapabilityInquiryDemo and confirm that the property value text box now has the same value you entered above.

### Partial property updates

The "Set Full..." button and "Set Partial..." buttons do the same job as the ones at "Local Configuration" tab. I haven't explained "Partial" updates yet. In MIDI-CI Property Exchange there are "full" and "partial" updates. On full updates, Initiator sends the whole property value to the Responder.

On partial updates, Initiator sends a set of diffs to the value what Responder currently holds. The diffs consist of a JSON object key-value pairs, whose key represents a [JSON Pointer](https://www.rfc-editor.org/rfc/rfc6901) and the value is the replacement value. The target property value should be, therefore, a JSON object (otherwise those JSON Pointers won't simply match).

Here is an example by screenshot of partial updates, from ktmidi-ci-tool to CapabilityInquiryDemo:

![Partial property update: before commit](https://i.imgur.com/G9PwwAG.png)

If ktmidi-ci-tool detects that the remote property supports "partial" updates by its `canSet` metadata field, it shows an extra textbox for RFC 6901 Pointer and value in a JSON object. When we perform "Commit changes", the Initiator sends only the text in that textbox. And the Responder performs "patching" to the existing JSON value. The resulting JSON looks like this:

![Partial property update after commit](https://i.imgur.com/VxAosZW.png)

The value looks totally different because all the JSON formatting is gone, but we can see the value specified by the JSON Pointer is now replaced with `1`.

Partial updates are useful when the resource is huge (like, a huge set of parameters we have as an `AudioProcessorValueTreeState`, though PE is not realtime safe) and we don't want to send the entire data all the time.

When it comes to JUCE CapabilityInquiryDemo as the Initiator, there is "Set Partial..." button, and it works just like "Set Full..." button, except that  the file content is supposed to be the JSON object that contains the diffs based on JSON Pointer. It is not a great UX, in case we care this minority...

### Miscellaneous Property Exchange features

Apart from those features I explained above, there are still couple of unmentioned features:

(1) **Content encoding**: the message body parts can be "encoded" other than **ASCII** string. A `mutualEncoding` header field can be used to indicate which encoding the body uses:

- **ASCII** string. It is the default. Note that unlike typical environment, it is NOT UTF-8. UTF-8 strings cannot be simply put in MIDI 1.0 compatible SysEx 7-bit buffer, so any non-ASCII characters and control characters must be converted to `\uXXXX` form.
- **Mcoded7**: for non-string binaries, they have to be converted from 8-bit BLOBs to 7-bit BLOBs by applying Mcoded7 encoding. The concept of Mcoded7 had existed since MIDI 1.0 age and it is documented in the Common Rules for PE specification so I do not spend my words here.
- **zlib+Mcoded7**: after Mcoded7 conversion, it tends to become bloated, so we can optionally apply zlib Deflate compression.

The last zlib compression is something we (I) could not verify interoperability. ktmidi-ci-tool runs into somewhat problematic situation here. There is no built-in zlib / Deflate implementation in Kotlin common API until ktor-io 3.0 lands. As a workaround, we have `expect` functions that are implemented only on Kotlin/JVM and Kotlin/Android (using `java.util.zip` API). It is known that JUCE CapabilityInquiryDemo does not work this API though. And MIDI 2.0 Workbench has [a blocking issue](https://github.com/midi2-dev/MIDI2.0Workbench/issues/5) that prevents Property Exchange testing so far (neither of JUCE CapabilityInquiryDemo nor ktmidi-ci-tool can be examined).

Acceptable encodings can be defined for each property metadata and the definition is a string array. ktmidi-ci-tool UI is designed based on that. However Current Common Rules for PE specification only accepts these 3 encodings. On JUCE CapabilityInquiryDemo, there are 3 checkboxes to choose whether each of those encodings can be used or not. It probably makes good sense (at least it is easier to choose them just by clicks).

When sending and receiving Property Exchange messages with body encoded with one of the above, we use an optional `mutualEncoding` header field to specify it. Then the message recipient should be able to decode it accordingly. On ktmidi-ci-tool, there is an extra dropdown menu that you can indicate which `mutualEncoding` to use on the Inquiry explicitly. CapabilityInquiryDemo seems to pick one of those supported encodings by its own. I prefer my choice of making it explicit (it is optional), as what we want is a development testing helper that lets us flexibly diagnose any problem.

(2) **Pagination**: if a resource only returns a JSON array, its items can be "paginated" i.e. queried partially. A Get Property Data Inquiry message may have `offset` (from which item) and `limit` (max number of the returned items) fields in the header, and the Reply comes with `totalCount` field in the header. ktmidi-ci-tool could tell these fields on the UI:

![ktmidi-ci-tool pagination client](https://i.imgur.com/mkVS86S.png)

To examine pagination feature, there are some extra tasks: you have to ensure that the property is set `canPaginate` metadata value as `true`. If your Responder is JUCE CapabilityInquiryDemo, you can enable it only if the property is *named* ending with `List` (therefore, this example screenshot says the property is `X-bazList`).  Then you can check `canPaginate` option there.

ktmidi-ci-tool shows these text input fields on the "Paginate?" row if and only if the property has `canPaginate` value as true. The query result is shown on the UI as if the entire result were the value on the text field. (We could show `offset`, `limit` and `totalCount` somewhere, but it is not implemented yet...)


## Process Inquiry
 
**Process Inquiry** is a feature that a MIDI-CI device can provide current state of the device. In MIDI-CI v1.2 specification, the only feature it covers is MIDI Message Report. It can be used to provide the state of MIDI device, such as note on state, pitchbend state, controller state, program/bank state, etc., like MIDI bulk dump on a device.

If I were the specification author, I would name it "Device State Inquiry". Why "Process" Inquiry? MIDI 2.0 used to feature "three Ps" which were Protocol Negotiation, Profile Configuration, and Property Exchange. As of version 1.2, MMA had to discard one P: Protocol Negotiation. So they would like another P to complement the lost one. That's all my guess.

In any case, juce_midi_ci does not support Process Inquiry yet. So if we want to test MIDI Message Report interoperability, there are only two I am aware of: ktmidi-ci-tool and MIDI 2.0 Workbench. It does not work as a Responder, but it is not a big deal because when we implement this feature, we mostly care about the receiver part.

An interesting defect by design on MIDI Message Report from my point of view is that it seems impossible for a MIDI-CI Initiator to concurrently wait for MIDI Message Reports from more than one MIDI-CI Responders. It is because this bulky operation comes with no "session" MUIDs except for the first Reply To MIDI Message Report message and the last End Of MIDI Message Report message. Everything inbetween is mere MIDI message such as Note On, Control Change, Program Change, etc. are just MIDI channel voice messages or system common messages that does not convey MIDI-CI MUID. If MIDI Message Report packets from Endpoint #1 and Endpoint #2 are mixed, then we cannot identify which Endpoint it sends the channel voice messages inbetween. Thus, a MIDI-CI Initiator needs to block incoming other MIDI channel voice messages and system common messages until the Responder finishes its MIDI Message Report session.

In any case, we can see how MIDI Message Report works, by using MIDI 2.0 Workbench as the Initiator. Connect to ktmidi-ci-tool and open up the device window (due to some issue, I needed to open, close and open the window again to get "Process Inquiry" part correctly shown - probably the tabs don't get refreshed when the app received Reply To Process Inquiry Capabilities message), and go to "MIDI Message Report" submenu. Then click "Get MIDI Report" button:

![MIDI 2.0 Workbench sends MIDI Message Report](https://i.imgur.com/CzsyXuM.png)

The ktmidi-ci-tool logs would show you what kind of messages were received and sent through the session.

![MIDI Message Report logs](https://i.imgur.com/4PEcdIu.png)

Those "[sent MIDI Message Report..." log lines look too simple, but if you scroll down each of those text fields you can see the actual MIDI bytes in the message. Since they can be thousands of log lines if I do not assemble them into a few chunks, those MIDI Message Report bytes are collected per a channel, per request block. It was only for Channel 1 so there were only 3 chunks, but if I choose "Port" at "Channel" on MIDI 2.0 Workbench, it can be a lot more.

ktmidi-ci-tool implements this feature by leveraging ktmidi `Midi1Machine` class (and it will enhance it with `Midi2Machine` when we add UMP support to ktmidi-ci) - this class existed to preserve any internal state of a virtual "MIDI device" that holds these information items MIDI Message Report are supposed to contain. I implemented this class when I was creating a visual MIDI player which existed only on C# version of the library... The class was ported to Kotlin too, but there was no actual use until this time.

Although it is not a perfect match - MIDI Message Report feature offers an option to indicate whether it should send literally all the state or only those what were set to non-default. `Midi1Machine` does not really preserve the state whether it has changed or not, so the internal structure will have to be changed.

## Summary

We have run through all the features currently available MIDI-CI tools offer. They would be helpful when you are building your MIDI 2.0 synths, effectors, or whatever your code that integrates MIDI-CI features.



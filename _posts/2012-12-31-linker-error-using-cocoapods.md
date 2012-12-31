---
layout: post
title: "Linker error using CocoaPods"
description: ""
category: 
tags: []
---
{% include JB/setup %}

### Linker error using CocoaPods

Recently I worked on an iOS project where I used another framework (OpenCV) that didn't have all symbols compiled into it
which caused numerous linker errors.  It took me a while to track down the source of these errors so hopefully
Google will index this post and save someone a few hours the next time around.  

I'm sure this image will be familiar to others.  The "Symbols not found" error occurs commonly in C libraries used by
C++ code.  The compiler may mangle the function names in the C libraries in a C++ project, resulting in a similar
error code.  

<img src="/images/linker_error.png">

Although this doesn't have anything to do with the error I got, here's a fix for it anyway.  The fix for the name mangling problem is pretty easy, you simply wrap your header stuff with something like this:

    #ifdef __cplusplus
    extern "C"
    {
    #endif

    // The rest of your header file goes in here

    #ifdef __cplusplus
    }
    #endif

Essentially the `extern "C"` block will prevent the mangling, and it won't be inserted if you're compiling a
normal C application.  However, the problem I encountered today had nothing to do with name mangling -- I looked
into the binary file to see if the symbols were actually there or not.  To do this, you just run `nm` on the 
library file and grep for instances of the missing symbol.

    U _AVCaptureSessionPresetHigh
    U _AVCaptureSessionPresetLow
    U _AVCaptureSessionPresetMedium
    U _AVCaptureSessionPresetPhoto
    U _AVLayerVideoGravityResizeAspectFill

As we can see, the symbol does not actually exist in the library.  Symbols which do exist will typically have
a number preceding the symbol to show where it is in the file, like this:

    00001c50 S __ZNK2cv7MatExprcvNS_4Mat_IT_EEIfEEv
    00002724 S __ZNK2cv7MatExprcvNS_4Mat_IT_EEIfEEv.eh
    00001e30 S __ZNK2cv9videostab20WobbleSuppressorBase10frameCountEv
    00001e90 S __ZNK2cv9videostab20WobbleSuppressorBase20stabilizationMotionsEv
    00001e50 S __ZNK2cv9videostab20WobbleSuppressorBase7motionsEv
    00001e70 S __ZNK2cv9videostab20WobbleSuppressorBase8motions2Ev
    00001ef0 S __ZNK2cv9videostab38MoreAccurateMotionWobbleSuppressorBase6periodEv

#### The underlying cause

Because CocoaPods compiles a static library and inserts it among all the other frameworks, it requires the `-ObjC` 
flag passed to the linker.  This flag prevents the "selector not recognized" error due to the fact that Objective-C
does not define linker symbols for functions in the categories.  The `-ObjC` flag forces the linker to load every
object file in the library that may define a class or category.  More details about this flag can be found in 
[Apple's Technical QA 1490](http://developer.apple.com/library/mac/#qa/qa1490/_index.html).

In our case, the OpenCV framework contained unimplemented symbols that were loaded with everything else in libPods.a,
which is the underlying cause of the problem.  We still need the `-ObjC` flag or else the we may get the "selector not recognized" error when running our code.  We'd like to load all the symbols in libPods.a, but leave the OpenCV
framework alone.  

#### The Fix

In the [QA 1490](http://developer.apple.com/library/mac/#qa/qa1490/_index.html), Apple describes an additional option,
`-force_load` as a means to selectively choose a library to load all the symbols for.  We'll use this specifically
for the CocoaPods library so that we don't affect anything else in the build configuration.

To add this linker flag, visit your build target's linking configuration and this as one of the linker flags:

    -force_load $(TARGET_BUILD_DIR)/libPods.a

Here's what it should look like in XCode:

<img src="/images/linker_fix.png">

Hopefully CocoaPods will consider using this linker flag in the future for the corner case that someone happens to
be using a combination of pods and home brew frameworks.

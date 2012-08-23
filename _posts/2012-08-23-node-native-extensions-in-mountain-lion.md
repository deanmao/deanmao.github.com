---
layout: post
title: "Node Native Extensions in Mountain Lion"
description: ""
category: 
tags: []
---
{% include JB/setup %}

### Node Native Extensions in Mountain Lion

I'm a maintainer for the [node-sass](https://github.com/andrew/node-sass) library which uses the
highly portable C++ [libsass](https://github.com/hcatlin/libsass) library to convert sass into css.
I ran into an issue today that may affect other extension writers trying to get their stuff working
in Mountain Lion.  The previous precompiled binaries would result in a segfault that doesn't really
provide any useful information on what went wrong.

The fix is pretty simple, you'll just have to add an xcode flag to target 10.7.  The default in node-gyp
is to target 10.5.

Here's what the new binding.gyp file looks like:

    ['OS=="mac"', {
      'xcode_settings': {
        'MACOSX_DEPLOYMENT_TARGET': '10.7'
      }
     }]
     
Adding this flag causes CFLAGS to add "-mmacosx-version-min=10.7" as an additional argument.

Keep in mind this only includes the section for macs.  You can find the binding.gyp file in its entirety
at the main [github repo for node-sass](https://github.com/andrew/node-sass).  Hopefully this doesn't
cause issues for people running older versions of OSX.



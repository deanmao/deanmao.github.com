---
layout: post
title: "Enter chimera"
description: ""
category: 
tags: []
---
{% include JB/setup %}

This is a follow up article to [Enhanced Web Scraping in Node.js](/2012/08/07/enhanced-web-scraping-in-nodejs/)

### Enter Chimera, the other kind of phantom

I was inspired by [PhantomJS](http://phantomjs.org) and wanted something similar, but could be run inside of the nodejs
environment, without calling out to an external process.  PhantomJS is run as an external process that users can run
under any language, however one must create a fancy glue wrapper so that development isn't impaired.  I created
something that does exactly what phantomjs is capable of doing, except in a full js environment, called [Chimera](http://github.com/deanmao/node-chimera).

If you're installing via npm, you can easily install it like this:

    npm install chimera
    
It does take quite a bit of time to download because it includes precompiled binaries for macosx, linux 32bit, and linux 64bit.  Each binary contains
everything needed to run chimera including parts of the Qt toolkit.

An example is the best way to show how easy this is using chimera:  (coffeescript shown below)

    Chimera = require('chimera').Chimera
    c = new Chimera()
    c.perform
      url: "http://digg.com"
      locals:
        username: myUsername
        password: myPassword
      run: (callback) ->
        setTimeout( ->
          if jQuery('a.modal-close-inline').length > 0
            jQuery('a.modal-close-inline').click()
            jQuery('#modal-login').click()
            setTimeout( ->
              jQuery('#ident').val(username)
              jQuery('#password').val(password)
              pos = jQuery('#login-button').offset()
              chimera.sendEvent("click", pos.left + 10, pos.top + 10)
            , 500)
          else
            setTimeout( ->
              callback(null, "success")
            , 1000)
        , 1000)
      callback: (err, result) ->
        console.log('capture screen shot')
        c.capture("screenshot.png")
        cookies = c.cookies()
        c.close()

Note that this example may not work anymore as digg.com has completely redone their site.  This example code
belongs in a single file that you'd run using something like "node example.js".  

### Let's break it down

Chimera includes all parts of the code to run inside and outside the browser.  The "run" property specifies a
function that will run inside the browser, where the first argument is a callback function that should be
called when you want to return back to node.

The callback property is a function that will be executed immediately when we pass the control flow back
to node.  (minimum example shown below)

    Chimera = require('chimera').Chimera
    c = new Chimera()
    c.perform
      url: "http://digg.com"
      locals:
        username: myUsername
        password: myPassword
      run: (callback) ->
        alert('I run inside the browser!')
        callback(null, "success")
      callback: (err, result) ->
        console.log("I'm back inside nodejs")

Since the browser does not run under the same js execution environment as node does, you cannot use variables defined
in your current closure.  The only way to pass variables into the browser is by making use of the "locals" property.
Each property defined in here will appear as a local variable inside the run method.

### Extracting & setting cookies

One of the interesting things about chimera is the ability to extract and assign cookies to the browser.  We can
assign cookies to our chimera instance so that when the browser is run, it starts with a given state.  For example, we might
first extract the cookies in our callback using the chimera.cookies() method in code that looks like this:

    c = new Chimera()
    c.perform
      url: "http://digg.com"
      run: (callback) ->
        // add login code here
        callback(null, "success")
      callback: (err, result) ->
        console.log("here are the cookies")
        console.log(c.cookies())

Then we can assign the cookies in a completely new browser window like this:

    cookies = "session_id=94d4d4bca3da9f7dc916e16c49363a9c5fd026178ead6cb35747471c994da8c3; expires=Wed, 30-May-2012 02:24:27 GMT; domain=.digg.com; path=/"
    c = new Chimera(cookies: cookies)
    c.perform
      url: "http://digg.com"
      run: (callback) ->
        // we should already be logged in!
        callback(null, "success")
      callback: (err, result) ->
        console.log("done")
        


---
layout: post
title: "Modify a site you don't own"
description: ""
category: 
tags: []
---
{% include JB/setup %}

### Modify a site you don't own

I've always thought the web should be malleable.  Readers on HN will often submit their own styling
to the news feed and allow others to hack up their own with a chrome extension or greasemonkey file, but
what if you wanted to distribute your idea to others without having them to install an extension?  
I've also seen people iframe a site and provide some kind of utility around the site.  There are also
bookmarklets that allow people to modify a site via a simple bookmark, but most of these are just too
troublesome for new users to pick up easily.

### Proxy that site

Some of the more clever companies will proxy all the content of the site to give the appearance that 
it runs locally.  For example, Optimizely does this quite well where they show another site where you
can tweak the appearance for their A/B testing system.  

<img src="/images/optimizely.png" style="display:inline-block">

&#x20;

As you can see here, they've iframed maps.google.com, but you can further tweak the site by using their built-in
wysiwyg editor.  They do this by proxying the content, then injecting more js into the page so that
they can bring up their wysiwyg editor and other tools.  It also provides them with the ability to send
information to and from the iframe since both frames are hosted under their domain.

If you wanted to do this yourself, it's quite a lot of work as you'd have to rewrite the page so that links
and js sources are handled properly.  Fortunately if you're using nodejs for your application, I've already
written everything that you'll need to implement this yourself.  I'll be describing the implementation in
future blog posts, but today I'll show how you can create an application using this kind of technology
immediately:

### Xtendme: The website extender

Check this out:

<textarea id="yourcode" style="width: 600px; height: 110px; font-size: 12px; font-family: monospace">
  // Edit me!
  jQuery('td > center > a').click(function(e) {
    e.preventDefault();
    jQuery('body').prepend('<b>You just tried to upvote something!</b><br>');
  });
  jQuery('body').prepend('<h1>Modifications everywhere!</h1>');
</textarea>

&#x20;

<button onclick="myiframe.postMessage(yourcode.value, 'http://news-ycombinator-com.xtendthis.com');">
  Click me to run that code inside HN!
</button>

&#x20;

Click the button above to inject the code into HN.  When you upvote something, it prepends a message at the
top instead.

<iframe id="myiframe" src="http://news-ycombinator-com.xtendthis.com/" style="display:inline-block" width="600" height="600">
</iframe>

&#x20;



NOTE: If the iframe above shows nothing, it might be that my server is dead.  Go easy on it because it's running in
development mode which means every js & html going through the system is being parsed & rewritten on
every request, no caching.  If I were to enable production mode, it would at least cache the js code that
"requests" to be cached.  I figured I wouldn't need it for a blog article, but if this article gets posted
on reddit, I might have to turn on production mode.

Also, don't bother trying to login on HN, I didn't turn on ssl, but the example github code will show how easy this
is -- it's the same way you'd do it on any other [express](http://expressjs.com) app.

### Implement this yourself

For reference, you can find everything discussed here in my [xtend example app on github](http://github.com/deanmao/xtend-example).
I'll show a brief example how you can implement your own site extension, then show you a running example of it.
The xtend-example uses the [xtendme library](http://github.com/deanmao/xtend) which does all the heavy 
lifting.  

First we'll have to create a Guide.  Guides describe how to transform html & js code.  When you load html
from the xtend system, it loads html from the remote site, transforms it into something slightly different,
then pumps it back out into your browser.  

    if typeof(window) != 'undefined'
      Guide = window.__xtnd_guide
    else
      Guide = global.XtndGuide

    class ExampleGuide extends Guide
      htmlVisitor: (location, name, context, url) ->
        value = super(location, name, context, url)
        if name == 'body' && location == 'end'
          return '<script src="'+@INTERNAL_URL_PREFIX+'/inject.js"></script>'
        else
          value

    if typeof(window) != 'undefined'
      options = require('data::options')
      guide = new ExampleGuide(options)
      window.xtnd = guide.xtnd
    else
      module.exports = ExampleGuide
      
The code above is from example_guide.coffee.  The reason you see it doing typeof(window) at the top of the file is
because this file (along with many others) is actually loaded inside the browser window.  This file can
dictate how to transform html code that may be generated dynamically via javascript inside the browser
window page.  This allows you to transform any html or js that could be loaded anywhere.  In future
examples, we'll show how you can transform js code to piggyback on existing functions like "addListener".  I briefly
covered javascript rewriting in a previous blog article that makes use of this library.

    if name == 'body' && location == 'end'
      return '<script src="'+@INTERNAL_URL_PREFIX+'/inject.js"></script>'
    else
      value

This transformation simply looks for the body tag and inserts a script tag at the end of body element.  The
@INTERNAL_URL_PREFIX is an internal url that will bypass xtendme's url filters so that you can load javascript
locally.

This next snippet of code is taken from app.coffee, the meat of the system:

    injectjs = fs.readFileSync('inject.js').toString()
    guide = new ExampleGuide(host: 'xtendthis.com', fs: fs)

    xtendme.generateScripts __dirname + '/example_guide.coffee', {host: host}, (scripts) ->
      server = express.createServer()
      server.configure 'development', ->
        server.use(express.errorHandler(dumpExceptions: true, showStack: true))
        server.use(express.cookieParser())
        server.use(xtendme.filter(guide: guide, protocol: 'http', scripts: scripts))
        server.use(express.methodOverride())
        server.use(server.router)
      server.get "#{guide.INTERNAL_URL_PREFIX}/:name", (req, res) ->
        name = req.params.name
        if name == 'inject.js'
          res.setHeader('Content-Type', 'text/javascript; charset=UTF-8')
          res.send(injectjs)
      server.listen(8080)

The code above was slightly reorganized from the xtend-example app.coffee code since we're illustrating
an example without SSL.  It's slightly simpler than the example code in the github project, but they
perform the same.  There's essentially 2 parts to configure: instantiating the Guide that we
talked about earlier, and configuring the express server.  

The xtendme library is implemented as a connect module.  You can read more about the 
[connect middleware at senchalabs](http://www.senchalabs.org/connect/) 
You utilize it in an express app like any other connect module:

    server.use(xtendme.filter(guide: guide, protocol: protocol, scripts: scripts))

If you're familiar with express, you can see all the other connect modules that we're using, like the cookieParser
and errorHandler.  Order is important here, the cookieParser must run before the xtendme.filter.  The
remote site may ask to persist cookies under their original domain name, so we will persist these cookies
in our local mongodb.  (Yes, I forgot to mention that you'll need node v0.8, coffeescript, and mongodb to get
all of this stuff running)

I've only added one express action so that I can load inject.js into the user's page:

    server.get "#{guide.INTERNAL_URL_PREFIX}/:name", (req, res) ->
      name = req.params.name
      if name == 'inject.js'
        res.setHeader('Content-Type', 'text/javascript; charset=UTF-8')
        res.send(injectjs)

This is probably extremely familiar to anyone who has written an express app before.  The only addition
is the INTERNAL_URL_PREFIX matcher.  The xtend connect module will intercept all requests except
for the ones that have the INTERNAL_URL_PREFIX as part of the request url.  In this case, we will send
the user the inject.js code if the name matches in the url.

The inject.js file includes the code that we're injecting into remote sites.  In our example, it contains
the following:

    // ... (code for jquery goes here) ...
    window.myjquery = $.noConflict();
    function receiveMessage(event) {
      // this is normally bad, so don't copy me here:
      eval(event.data);
    }
    window.addEventListener("message", receiveMessage, false);

Essentially I'm just eval'ing code sent via postMessage() so I can inject code from any remote domain
that may contain the iframe.  Obviously if you were implementing this yourself, you'd want to compare
event.origin to restrict it, but as a proof of concept you can see what's possible here.

As you can see, it's extremely easy to implement this yourself.  It's only around 30 lines of
coffeescript and we've got a site that proxies and rewrites another site!

### Other implementations

Proxying a site and rewriting all its content is not a new concept.  There have been other implementations
at various degrees of success.  The oldest implementation I can think of is 
[jmarshall's cgiproxy](http://www.jmarshall.com/tools/cgiproxy/).  It's a huge perl script that essentially 
does the same thing as xtendme, but it's written in perl and it's 16,000 lines of code.  I already have a 
headache reading 200 lines of perl so trying to decipher that beast will be a nightmare! (unless you're the
author)  Xtendme is a mere 1500 lines of coffeescript, and it's fairly modular should one decide to tweak the
code.  

After running cgiproxy, you'll soon see that it is not quite capable of dealing with javascript-heavy sites
like gmail or google plus.  Most of the web is moving toward client-heavy code so it's fairly important
that the proxy can handle a site like gmail.  Xtendme actually works quite well on gmail, so I'm fairly 
confident it will handle other javascript sites as well, but it does have one weak point -- it doesn't have
the best html parser so some sites may still cause it to break.  It relies on the popular
[node-htmlparser](https://github.com/tautologistics/node-htmlparser/) project that many other popular projects 
(like jsdom) rely on.  Please consider donating time into improving tautologistics' htmlparser. One small fix 
will improve many node libraries.

I feel that javascript is the best language to write this proxy in because most of the web consists of html and
javascript so it's only natural for a system that transforms the web should be written in the language of
the web.  I haven't seen a good reverse-proxy for node, so I figured this would be a nice little project.

### Security concerns

Yes, this is a man in the middle attack.  You can create a website that serves up gmail running this proxy
and trick the end user into visiting it.  If the user doesn't realize they're not on mail.google.com, they
would be sending all kinds of private data to a server somewhere.  Even sites that implement something like
[CORS](http://en.wikipedia.org/wiki/Cross-origin_resource_sharing) for their login, there are work-arounds
that handle this in Xtendme.  So please, use this for good.  I don't want moms sending me angry emails.


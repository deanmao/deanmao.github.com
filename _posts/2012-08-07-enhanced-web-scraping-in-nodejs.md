---
layout: post
title: "Enhanced web scraping in Node.js"
description: ""
category: 
tags: []
---
{% include JB/setup %}
 
## Everyone does it (web scraping)

Web scraping is something everyone does, only because it's often the only way to get a wealth of
information not possible via a site's API.  Often sites won't even have a usable API and the only
way to get data is by scraping it.  I'm sure if you've ever attempted web scraping, you'll know
there's many different methods for collecting the data.  

### Stuff out there

I started doing web scraping using hpricot, an early user-friendly xml library written by the 
famous rubyist "_why".  The code looked something like this:

    require 'hpricot'
    require 'open-uri'

    doc = Hpricot(open("http://news.ycombinator.com"))
    links = doc/a
    links.each do |link|
      puts link
    end

It's quite easy to use, but it's not very fast as most of the time is spent waiting for the http
response from the server, so if you had to process a large number of links, it would be done
sequentially slowly.  Of course one could use a variety of non-blocking io libraries to make it
faster, but one is still limited by the speed of Hpricot.

These days, you're probably more likely to use nokogiri, or mechanize depending on the complexity
of your scrape.

If you're doing python, there's BeautifulSoup, but in my past experience, BeautifulSoup has 
had difficulties with certain character codes that have caused it to bomb.  Perhaps many of
these bugs have been ironed out, but it hasn't been my choice of weapon lately.

    import urllib2
    from BeautifulSoup import BeautifulSoup

    soup = BeautifulSoup(urllib2.urlopen('http://news.ycombinator.com').read())

    for row in soup('a'):
      print row().string

### Enter node.js

Node is a great tool for doing web scraping as the language is inherently non-blocking so one
can scrape a great number of pages quickly using familiar libraries like jquery.

    var jsdom  = require('jsdom');
    var fs     = require('fs');
    var jquery = fs.readFileSync("./jquery.js").toString();

    jsdom.env({
      html: 'http://news.ycombinator.com/',
      src: [ jquery ],
      done: function(errors, window) {
        var $ = window.$;
        $('a').each(function(){
          console.log( $(this).attr('href') );
        });
      }
    });

In this snippet, we use jsdom which emulates the dom in a nodejs environment.  So if there are
script tags in the page that manipulate the dom, jsdom will emulate what a real web browser 
would do.  However, since jsdom is not actually a real web browser, it doesn't handle all web
pages correctly and may bomb on webpages that run a lot of javascript.  If you're trying to
scrape the facebook wall, you'll need something more robust, like phantomjs.

### Enter phantomjs

PhantomJS is a cool little command line tool that runs QtWebkit in an emulated frame buffer 
environment.  They call their emulated frame buffer branch "lighthouse" which effectively 
abstracts the underlying display calls so that it is easier to write cross platform code.  As 
an additional benefit, one can use Qt's internal frame buffer instead of a real display.  

PhantomJS enables you to run external js code as if you were running a greasemonkey script
inside of any webpage, and output to the console.  You could then parse the content from
the console for your preferred choice of language for manipulation.  Here's a brief example
using pjscrape, a library that helps nodejs interface with phantomjs:

    pjs.addSuite({
        url: 'http://news.ycombinator.com',
        maxDepth: 0,
        scraper: function() {
            return { 
                link: $('a').attr('href')
            }
        }
    });

It's relatively easy to use, considering it actually fires up another process behind the scenes.

### Enter chimera

In a future blog post, I'll be describe what I believe to be the holy grail for web scraping
on nodejs.  
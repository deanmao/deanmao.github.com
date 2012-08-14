---
layout: post
title: "Fancy breadcrumbs via Raphael"
description: ""
category: 
tags: []
---
{% include JB/setup %}

### Breadcrumbs are bubbly

Breadcrumbs are a great UI indication component that shows you where you're been, and potentially where you can go.

Most websites that make use of breadcrumbs, have breadcrumbs dated back from the web 1.0 era where people used 
multi-page navigation, however in the web 2.0 era where many apps are done as single-page apps making use of 
libraries such as emberjs or backbone, we need to go with something a bit fancier.

If you search for "css breadcrumbs", you'll find great css tutorials on how to generate breadcrumbs that look like
these:

<img src="/images/breadcrumb1.png" style="display:inline-block">

&#x20;

<img src="/images/breadcrumb2.png" style="display:inline-block">

&#x20;
 
<img src="/images/breadcrumb3.png" style="display:inline-block">


They look okay, but the amount of difficulty required to create a triangle with gradient in css is actually quite
difficult.  The strategy for creating a triangle via css is by adding a thick border to an element.  It's a cool
trick, but difficult to do anything fancy with.  If your designer wants to create something truly unique, the
best way to go about it is to draw the shapes manually.  The code required to do something like this is actually
easier than it sounds.

### My Fancy Breadcrumbs

Here's my example of a breadcrumb.  You can click on the steps to reveal a different section of the document.  All
the code below can be found in [my breadcrumb repository on github](http://github.com/deanmao/breadcrumb).  It's 
hard to imagine anything slicker than this!
 
&#x20;

<script src="/js/jquery.js">
</script>

&#x20;

<script src="/js/raphael-min.js">
</script>

&#x20;

<script src="/js/raphael-font.js">
</script>

&#x20;

<script src="/js/breadcrumb.js">
</script>

&#x20;

<style type="text/css" media="screen">
  #breadcrumb_canvas {
    height: 55px;
  }
  #mycontent {
    height: 80px;
    width: 400px;
    border: 1px solid #ccc;
    font-size: 18px;
  }
</style>

&#x20;

<div id="breadcrumb_canvas">
</div>

&#x20;

<div id="mycontent">
</div>

&#x20;

<script type="text/javascript" charset="utf-8">
  var bc = new Breadcrumb("breadcrumb_canvas");
  var content = $('#mycontent');
  bc.make("Step 1", function () {
    content.html('Collect underpants.');
  });
  bc.make("Step 2", function () {
    content.html('???');
  });
  bc.make("Step 3", function () {
    content.html('PROFIT!');
  });
  bc.make("Step 4", function () {
    content.html('Should not get here!');
  });
  bc.get("Step 4").makeDisabled();
  bc.get("Step 1").makeActive();
</script>

The code to create something like this is fairly simple by scripting something that builds the "path" commands from
raphael.  The snippet below will build out a triangle.  If it happens to be the first triangle, the left side of
the triangle is flat.  (The code below is written in coffeescript)  The variable "x" starts out as null, and will be
the starting x-coordinate of the next triangle.  If it is null, that means we're at the first triangle.

    @r = Raphael("breadcrumb_canvas")
    arrowWidth = 15
    isStart = true
    padding = 30
    if typeof(x) == 'object'
      isStart = false
      box = x.getBBox()
      x = box.x + box.width - arrowWidth
    x = parseInt(x)
    fontSize = 15
    s = @r.print(0, 0, text, @r.getFont("Museo Sans 500"), fontSize)
    w = parseInt(s.getBBox().width + (padding * 2) + arrowWidth)
    s.remove()
    if isStart
      w = w - arrowWidth
    height = 20
    if active
      height = height + 2
      y = y - 2
      arrowWidth = arrowWidth + 1
      x = x - 2
      fontSize = fontSize + 2
    path = "M "+x+" "+y+" l "+w+" 0 l "+arrowWidth+" "+height+" l -"+arrowWidth+" "+height+" l -"+w+" 0 "
    if isStart
      path += "s -5 0 -5 -5 l 0 -"+((height * 2) - 10)+" s 0 -5 5 -5"
    else
      path += "l "+arrowWidth+" -"+height+" z"
    c = @r.path(path)
    c.path = path
    
Then we apply a gradient to the triangle.  If the triangle is clickable, we indicate it with the cursor as pointer.
    
    if active
      c.attr(
        gradient: '90-#236aa7-#31abd2'
        'stroke-linejoin': 'round'
        'stroke-width': 0
      )
      y = y + 2
    else
      c.attr(
        gradient: '90-#b3b6b5-#ffffff'
        'stroke-width': 1
        cursor: 'pointer'
        stroke: '#a3a6a5'
      )


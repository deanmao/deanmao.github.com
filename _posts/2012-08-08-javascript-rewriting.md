---
layout: post
title: "Javascript code generation"
description: ""
category: 
tags: []
---
{% include JB/setup %}

### Rewriting JS in JS

We're not talking about creating a JS virtual machine, we're just talking about compiling JS
into JS.  Why would you want to do such a thing?  It seems odd that one would want to modify
JS code on the fly -- it would certainly be more efficient to have written the modified code
from the beginning, but there are many non-obvious cases where having a preprocessor trumps
writing the code yourself.  For example, one could perform preprocessing over untrusted JS 
and transform it into "trusted" JS.  I first saw an example of this in the (now defunct) 
OpenSocial project from Google.  Essentially developers could code JS widgets that would run
on other people's webpages.  If the JS code leaked from the sandbox, one could do potentially
do nefarious scripting attacks on the user.  OpenSocial prevented this by running the JS code
through a library called Caja.  Caja turns normal javascript into an AST that is pumped through
a bunch of transformation rules that converts dangerous code into safe code.  The newly
modified AST is converted back into code where is run on client machines.  

Inspired by Caja, I've authored a similar basic library that offers similar capabilities as
the Caja project, only written in JS.  Although my work-in-progress library is not as sophisticated
as Caja, it provides a similar set of capabilities.

### Caja example

In Caja, one would write a rule like this:

    new Rule() {
      @Override
      @RuleDescription(
          matches="@o[@s] = @r",
          substitutes="mymethod(@o, @s, @r)")
      public ParseTreeNode fire(ParseTreeNode node, Scope scope) {
        Map<String, ParseTreeNode> bindings = match(node);
        if (bindings != null) {
          return substV(
              "o", expand(bindings.get("o"), scope),
              "s", expand(bindings.get("s"), scope),
              "r", expand(bindings.get("r"), scope));
        }
        return NONE;
      }
    },

which converts code like this:

    myobject[1+1] = a + b

into a method call looking like this:

    mymethod(myobject, 1+1, a + b)

It's nice how one writes javascript as the matcher and substitution since we're just converting javascript
into javascript.  

### Rewriting JS with pure JS

In my mini-caja implementation, [xtend](http://github.com/deanmao/xtend), one can write a similar rewrite rule, 
except done in javascript (coffeescript example shown here):

    JsRewriter = require('xtend').JsRewriter
    jsr = new JsRewriter({})
    jsr.addRule().find('@obj[@prop] = @rhs', {useExpression: true})
      .replaceWith("mymethod(@obj, @prop, @rhs)")
    
And when you need to convert code, you can write a statement like this:

    jsr.convert('window["location"] = "http://www.google.com"')
    
Which will return a string like this:

    mymethod(window, "location", "http://www.google.com")

As you can see, it's fairly easy to convert existing js code into another form.  One could use this tool
to rewrite javascript running on other people's webpages for the purposes of listening to all events
being fired.  For example, you could write the following snippet of javascript:

    jsr.addRule().find('window.addEventListener(@args+)', {useExpression: true})
      .replaceWith("mySpecialMethod(@args+)")

And implement "mySpecialMethod" to look something like this:

    function mySpecialMethod(args) {
      console.log("addEventListener called with the following arguments:", args)
      window.addEventListener.apply(window, args)
    }

Every time one performs an addEventListener invocation on window, your method will be called instead, but
will perform the existing logic to preserve original functionality.
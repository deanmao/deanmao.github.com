---
layout: post
title: "Converting Lua code into JSON, via JS"
description: ""
category: 
tags: []
---
{% include JB/setup %}

### Lua, the other javascript

It's amazing there's another language older than javascript, with more metaprogramming abilities,
yet employs a relatively simple syntax that is so incredibly identical that one could convert an
existing javascript parser into parsing lua in just a few mere hours.  

[Esprima](http://www.esprima.org) is a javascript parsing infrastructure that converts JS code into
an AST which can be transformed and manipulated.  I was tasked with converting lua code into a 
JSON object and thought this might be an interesting and quick way to get it done.  

### Code conversion

I converted esprima so that one could pass in lua code as a string and would return a json object 
converted from a lua table object.  I was surprised how simple it was to convert esprima to become 
a lua parser.  The author did a great job of maintaining code quality and readability.

Here's a small snippet of my code where given a string of lua code, it will locate table creation
and return the table as a json object.  One could extend this concept to provide a more featureful
lua code editor in-browser.  

    var esprima = require('esprima');

    function jsonExtractor(source) {
      var root = esprima.parse(source);
      var array = []
      function traverse(node, obj, key) {
        var child, visited = false;

        if (node.type === 'TableAssignmentExpression') {
          var name = node.left.name;
          if (node.right.type === 'Literal') {
            obj[name] = node.right.value;
          } else if (node.right.elements) {
            if (name) {
              visited = true;
              obj[name] = traverse(node.right.elements, {})
            }
          }
        }

        if (!visited) {
          for (key in node) {
            if (node.hasOwnProperty(key)) {
              child = node[key];
              if (typeof child === 'object' && child !== null) {
                if (key === 'elements') {
                  var x = traverse(child, {}, key);
                  if (!x.frames) {
                    array.push(x);
                  }
                } else {
                  traverse(child, obj, key);
                }
              }
            }
          }
        }

        return obj;
      }
      traverse(root);
      return array;
    }


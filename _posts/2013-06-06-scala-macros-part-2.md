---
title: Adventures with Scala Macros - Part 2
date: 2013-06-06 00:00:00 Z
categories:
- Tech
tags:
- Scala
- Web Services
- blog
author: jphillpotts
layout: default_post
source: site
summary: In this article we'll look at ways to overcome one of the main restrictions
  of def macros - the ability to only generate functions.
oldlink: http://www.scottlogic.com/blog/2013/06/06/scala-macros-part-2.html
disqus-id: "/2013/06/06/scala-macros-part-2.html"
---

## Where next?

In 
<a href="{{site.baseurl}}{% post_url jphillpotts/2013-06-05-scala-macros-part-1 %}">part 1</a>
we used a macro to generate some regular expressions and a case statement that used them:

<script src="https://gist.github.com/mrpotes/678c1918e1c637da1f7a.js?file=generated-code.scala">
</script>

What you've probably noticed though is that with this code, the regexes are parsed each
time through the block, which isn't going to be very good for performance. What we really
want is to be able to put the regular expressions in an object so that they are 
parsed only once. But we're using Scala 2.10, and all we have available are def (function) 
macros - we can't generate any other types (although type macros do exist in the [macro
paradise](http://docs.scala-lang.org/overviews/macros/typemacros.html)).

Instead we need to create a value which then gets populated by a function that returns an
object, like this:

<script src="https://gist.github.com/mrpotes/1b8b2b2f2d898bd89394.js?file=object-from-function.scala">
</script>

So now we've got two functions for our generated code - and we know how to generate
function bodies. However, we have another problem - each function will be generated by a
separate macro, and we'll need to know what the variables created in one macro are called
when using them in the other macro. Fortunately we can just avoid that problem - each
regular expression is generated from a class name, and each class name (with its package
qualifier) is unique, so we can use the class name to generate the variable names 
(replacing `.` with `$`).

## Code organisation

Our macro code is getting bigger, and I don't want it to end up as a pair of unmanageable
functions that go on for page after page - so I'm going to split the code up into two
classes - the one that has the macro definitions in it and builds the resultant AST for
the generated code, and another class that contains all the logic for analysing the
compilation environment.

As documented on the [def macros page on the Scala site](http://docs.scala-lang.org/overviews/macros/overview.html),
you can create other classes that use your macro context easily enough - you just have
to give the compiler a little helping hand to work out where the context comes from. Of
the two examples for doing this on that page, the second is (to me) much more readable,
so we've got a new class:

<script src="https://gist.github.com/mrpotes/1b8b2b2f2d898bd89394.js?file=helper-class.scala">
</script>

And obviously we can initialise this just like any other class from within our macro
implementations using `val helper = new Helper(c)`. If, for tidiness, you want to
import the functions defined in Helper, you can then do `import helper.{c => cc, _}`,
which renames the `c` value from Helper so that it doesn't collide with the `c`
parameter from our function signature.

Moving the compilation unit processing code into the Helper, and adding some name
processing so that the class name and package name are available to our macro, we 
end up with: 

<script src="https://gist.github.com/mrpotes/1b8b2b2f2d898bd89394.js?file=Helper.scala">
</script>

## Object Creation

When you define an object in normal Scala, you just declare it, `object myObject`,
because the *syntactic sugar* allows you to leave out a default constructor, and
that it extends `scala.AnyRef`. In a macro you don't have that luxury, so to define
a new object, you do the following:

<script src="https://gist.github.com/mrpotes/1b8b2b2f2d898bd89394.js?file=object-declaration.scala">
</script>

We already know how to put a block of code together from part 1, so all we need to
do is merge the two together, and we get:

<script src="https://gist.github.com/mrpotes/1b8b2b2f2d898bd89394.js?file=object-macro.scala">
</script>

This can then be used by declaring `val regexesUsingMacro =
restapi.Generator.macroPaths` in our unit test.

But wait - there's a problem, isn't there? That function is returning an object of
type `Any`, so all our other generated code will know about it is that it's an any
- it won't know anything about the `val`s we've added to it. Well actually, it turns
out this isn't a problem, but is intended that the return type should know more than 
its declaration specifies, if possible, as Eugene Burmako explains 
[here](http://stackoverflow.com/a/13673950).

## Putting it all together

Now that we've got a macro that can be used as a `val` definition, we need to find
that `val` and use it in our match expression. To find the `val` is simple enough -
we just look for a `ValDef` that uses a `Select` for our function as the right hand
side. However, if the user hasn't defined such a `val`, we can't continue - we need
to tell the developer what they've done wrong. The macro `Context` includes 
functions to provide compiler warnings and errors, so we need to emit an error
advising the developer how to fix it. The structure we end up with is as follows:

<script src="https://gist.github.com/mrpotes/1b8b2b2f2d898bd89394.js?file=find-value.scala">
</script>

When integrated with the code we had for the match block from part 1, we end up 
with:

<script src="https://gist.github.com/mrpotes/1b8b2b2f2d898bd89394.js?file=case-macro.scala">
</script>

So now we've got our pattern matching working well, in 
<a href="{{site.baseurl}}{% post_url jphillpotts/2013-06-07-scala-macros-part-3 %}">the next article</a> 
we can start calling an API to produce our endpoints.
 
























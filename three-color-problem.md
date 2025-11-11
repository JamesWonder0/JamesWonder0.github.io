---
layout: page
title: The Third Function Color
permalink: /three-color-problem.html
---

# The Third Function Color
*Async languages already have two function colors. But they really should have three.*

I recently ran across this article about problems using Rust async: [We
Deleted Tokio and Saved \$127K: Our Rust Migration Story \|
Medium](https://medium.com/@the_atomic_architect/we-deleted-tokio-from-our-payment-system-and-cut-cloud-costs-by-127-000-b745a86f973b)

The key part is:

> We had a cascade failure. One database query took longer than
> expected. That blocked an async task. Which held up the
> executor. Which caused other tasks to queue. Within minutes, the whole
> service was timing out.

Many responses pointed out that this is exactly what you are NOT
supposed to do in async code: call functions that may block, and thus
starve the executor. Just avoid blocking and you won't have this
problem!

However...

## Avoiding Blocking Is Hard

Having written and debugged lots of async code myself, this
starvation-due-to-blocking problem is frustratingly common and painful.

The core issue is that it's harder than it seems to avoid blocking in
your async code. Sometimes it's as simple as not realizing the called
function can block -- perhaps it's deep down the call chain, or perhaps
it's library code you don't know well. Maybe it's a blocking mutex that
almost never contends -- until it does contend. Maybe it's a callback
that's intended to never block but is being used incorrectly. Or perhaps
a formerly non-blocking function is changed to be blocking, without
realizing that some async code is calling it.

Often, these calls block for only a short time, so the impact is not
significant in practice -- until something changes, you hit the tipping
point, your system is spiraling into starvation and it's a crisis. And
you have no idea why, or even where to start looking.

## The Third Function Color: Yellow

This got me thinking about the classic article [What Color is Your
Function? --
journal.stuffwithstuff.com](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/).
It explains how async language support results in two "colors" of
functions: red for async functions and blue for sync functions. And
these colors have different rules, enforced by the compiler:

-   Blue (sync) functions can be called anywhere.
-   Red (async) functions can only be called from red functions.

Unfortunately, in practice, these rules are wrong -- or at least,
incomplete.

As we discussed above, you can't call any blue (sync) function from a
red (async) function. You can only safely call a subset of blue
functions -- those that are guaranteed not to block. Otherwise, you risk
starvation and all the pain that comes with it.

Let's give these non-blocking sync functions their own color: yellow.

Then in practice, the actual rules are:

-   Yellow (sync non-blocking) functions can be called anywhere.
-   Blue (sync blocking) functions can only be called from blue
    functions.
-   Red (async) functions can only be called from red functions.

Unfortunately, unlike the original red/blue distinction, the compiler
provides no enforcement of these rules. You're on your own to ensure
that red (async) functions do not call blue (sync blocking) functions.

## Can We Do Better?

Could languages with async programming support (including, but not
limited to, Rust) enforce these rules?

In theory, yes. We already have two colors of functions and we enforce
the calling rules for them. Adding a third color and associated rules is
just extending what's already done.

Unfortunately, in practice, this would be very difficult, because the
"yellow" category simply doesn't exist today. Everything is red or blue.
We would need to audit every blue function and determine if it is
actually yellow. Then we would need to restrict red functions to only
calling yellow functions, not blue ones. Then, we would discover that a
lot of red functions are incorrectly calling blue functions instead of
restricting themselves to yellow functions. These are bugs that should
be fixed, of course -- but fixing them would be a lot of work, too.

But considering how painful starvation is in practice, it might be worth
the cost. (At the very least, new languages should strongly consider
adding this from the start.)

So what might this look like in practice?

## The "noblock" Function Qualifier

Interestingly, there's some prior art in Rust itself we can draw on --
the "const" function qualifier. Just like our yellow functions, const
functions can be called anywhere, but a const function itself can only
call other const functions.

So let's do something similar for yellow functions: qualify them with
the "noblock" keyword, and enforce similar rules.

To ease the transition, the compiler could be made to infer "noblock" in
many cases: any function that only calls "noblock" functions can be
inferred to be "noblock" itself.

Enforcement could start with lints that detect violations of "noblock"
rules. Over time, those lints could evolve into optional compiler
enforcement.

Adopting this across the ecosystem would, of course, be a large
endeavor. But in the long run, the pain of starvation is real enough to
justify this endeavor.

---
title: "Why is shit code everywhere"
date:  2025-03-18
draft: true
description: "TODO html meta su mmary"
summary: "TODO short article description that is rendered"
tags: ["welcome", "new", "about", "first"]
---

## Optimizing costs

There are three costs you can optimize when writting code:

1. How much it costs to write---a measure of *time to market*
2. How much it costs to maintain---this is a measure of robustness, and includes understanding what it does, changing it,
   fixing bugs in it, for everyone that will ever interact with this code, from now until forever
3. How much it costs to run it---a measure of *time it takes to run* x *frequency it is run* x (*HW cost* + *cost of the
   time end user is kept waiting* x *number of end users*)
   
Usually, optimizing one of these comes at the cost of the other two. Dynamic languages obviously optimize 1. at the cost
of 2., while static languages are the reverse.

A key thing to realize is that while the cost of 1. is constant in time, the cost of 2. (and 3.) are at least
linear---they increase, and keep increasing, with time. Therefore, on a long enough timescale, cost of maintainance will
surpass all other costs.

This is in line with the recommendation above---use static languages for "big, long-lived, evolving things"---but at the
same time, it's often not what you actually see in the wild. What you actually see is often pretty much the exact
opposite. Why?

Pretty much all software products started out as an idea of their founders, hacked together as an experiment with no
indication that it would explode into a viable business. Naturally, in those circumstances, using a dynamically typed
language is the way to go---after all, we're just scribbling stuff on pieces of paper, and we've got no idea if what we
scribble down will even exist a week from now. We need to [run experiments on the
market](https://www.youtube.com/watch?v=HSVbD7RhOHU). This is further exacerbated by the fact that founders are
typically geared towards business, not technological infrastructure, and generaly tend to have a hunter, not a gatherer,
mindset. Therefore, optimizing 1. is top priority.

Many such projects die, but some live on and become successfull companies. If they do, the last thing a
business-oriented founder wants to do at that point is slow down, let alone start rewritting things that already work.
Growth is essential, especially if you need to worry about competition or venture capital. Therefore, optimizing 1. is
still top priority.

Finally, if all goes well, the company goes on to transition from being a startup to being an actual enterprise---typically, at
this point, it is no longer existentially dependent on venture capital, and has carved out a stable piece of the market
for itself.

From a business standpoint, I have just described an incredible success story. From a technological standpoint, I've
described a nightmare.

TODO continue

All looks good from the outside, but on the inside, things are getting progressively worse. Optimizing 1.
came at the cost of 2., and technical debt, like any dept, comes with interest.

they typically
experience large growth in a short amount of time, and 

Doesn't apply when "things get removed quickly, because we're still iterating"

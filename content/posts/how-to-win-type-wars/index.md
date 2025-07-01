---
title: "This is how you win type wars"
date:  2025-03-26
description: "Beyond the static vs. dynamic typing flame war: a practical guide to understanding the tradeoffs, when to use each, and how Common Lisp bridges both worlds."
summary: "Beyond the static vs. dynamic typing flame war: a practical guide to understanding the tradeoffs, when to use each, and how Common Lisp bridges both worlds."
tags: ["basics", "types", "lisp"]
---

## Introduction
If you've spent any amount of time in programming circles, you've probably heard some version of the **"static vs.
dynamic typing" debate.** But beneath the tribalism lies a very real and very practical **difference in what these
languages optimize for,** and any good engineer should understand what that difference is, what the tradeoffs are, and
when they should choose which.

I've already made the case for static typing [in a separate
article](/posts/arguments-against-static-typing), so I won't reproduce it here. Instead, I want to dig
into **why the arguments on both sides feel so compelling to the side that puts them forth, touch on an analogy that
drives the whole thing home, and finish by making a short stop in a place not many people get to visit---Common Lisp.**

## There is no war

I want you to think about something that seems completely unrelated---**Microsoft Word vs. pen & paper.**

**Pen & paper is a fantastically flexible tool.** You can use it to write down anything, in any form---words, pictures,
flowcharts, graphs, and anything and everything that has ever been invented or will ever be invented. You can write in
any direction or angle, change the size of what you're writing to fit the space, draw an arrow and just continue
elsewhere if it still doesn't, go back and add a quick note in the space above a sentence you had previously written, or
squeeze something in the margins, or anywhere else. How someone else writes on paper, or how they feel you should, has
absolutely no bearing on how you do it. You don't need to learn anything new. If you need to write something down, you
can write it down in precisely the time it takes to stretch out your arm and grab what you need---it's so fast it
doesn't even register as an activity. **And everything I just described requires constant, and very low, effort to do.**

By contrast, **Microsoft Word is rigid.** You can only really use it to write text, and in a very specific manner---in
Latin languages, it goes line by line, left-to-right. You can tweak the font, size, color, and many other attributes,
but you can ultimately only do so in certain ways, and beyond a certain point it tends to involve significant effort.
You spend a lot of time jumping through hoops, and there are things that you just can't do at all, period. There's a
significant learning curve, so much so that there are dedicated courses that teach you how. That's actually a pretty
absurd thing, when you think about it---**why should you spend weeks learning to do something that you've been able to
do with zero effort and zero cost since you were a child?**

I think the analogy is quite clear. Now ask yourself this: **is there a war between pen & paper and Microsoft Word?** Of
course not! That's completely absurd, and I'm sure that anybody reading this agrees.

Now ask yourself, why? What would you tell two hypothetical people arguing about Microsoft Word vs. pen & paper?

Easy---you'd explain that they're **different tools for different jobs,** ask the people arguing what problem they're
trying to solve, and make a recommendation based on the task at hand. You'd recognize that there's no single correct
answer, and that it depends on context. But you can easily imagine making the case from either side, and ridiculing the
other.

**Would you write a book using pen & paper, much less collaborate on one?** Would you manage project documentation,
which changes every week, using pen & paper? Have you ever tried making sense of someone else's notes, or even tried
deciphering someone else's handwriting for that matter?

From the other side---**would you open up Word just to jot something down quick?** Would you use Word to map out an idea
in your head, or work through something you're not quite clear on? Do you use Word for shopping lists, tic-tac-toe, or
keeping track of scores in a drinking game? Can you imagine doing all three in a single document?

This underscores the key point I'm trying to make: **you can make the argument in both directions, and still be
completely right in the specific scenarios you cherry-picked.** This is the fundamental reason there even is a typing
war in the first place (and probably applies to wars in general)---**both sides are absolutely convinced that they're
correct, because *they really are*, in the scenarios they cherry-picked.**

What's more---they don't realize they're cherry-picking! Their opinions are based on their life experiences and the
problems they've encountered, and all they're doing is optimizing for these problems. **But they've lived different
lives, encountered different challenges, and learned to optimize for different problems!**

**A typical web designer probably doesn't know what it's like to maintain code that was written 10 years ago** by
someone that's not around anymore, in situations where a small mistake can cost hundreds of thousands of dollars. And
**a typical enterprise Java developer probably has no conception of what it's like to have an idea now, hammering
together a working prototype in two hours, polishing it, and tomorrow just being...done.** "Done" is not a word that
exists in enterprise software development.

**How surprising is it that these two groups have different perceptions of what constitutes an everyday problem, and
therefore what people should be optimizing for?** Obviously, these are two very different jobs, and require a very
different set of tools.

How do you win a type war? Simple---by recognizing that **there's no such thing as a type war.**

## "Different tools for different jobs", not "different strokes for different folks"

There is [a group of people](https://world.hey.com/dhh/programming-types-and-mindsets-5b8490bc) who will read the
previous paragraphs and draw the conclusion that, fundamentally, both approaches are always valid, and that the decision
is a purely subjective one---**"different strokes for different folks".**

**They are wrong.**

Yes, there is no choice that's correct *universally*, but that doesn't mean that there isn't **a choice that's correct
*in a given set of circumstances*.** I'm not saying there isn't a twilight zone between night and day where things can
be reasonably done in both ways, but I *am* saying that's not the case in the vast majority of situations.

**All other things being equal, I strongly believe that, in most situations, there are choices that are objectively more
correct than others.** I do want to emphasize the "all other things being equal" part, as in the majority of situations,
all other things are *not* equal. The Chief Architect at Slack [would still choose PHP to build a new app in
2020](https://slack.engineering/taking-php-seriously/), and it's certainly not because of its approach to typing, but in
spite of it.

Statically typed languages **force you to solve a certain problem and explicitly express your solution,** every time,
all the time. **Dynamically typed languages do not force you to do anything** - ever. They are **completely flexible and
arbitrary** and you can do pretty much whatever you want. They don’t prevent you from thinking hard, but **they don’t
force you to, either.** Often, that’s fine, but it **becomes easy to miss subtle problems** that look simple but
actually **require** you to think hard, because there **is no warning sign.** In statically typed languages, the warning
sign is that **it doesn’t compile,** and it's there every time.

As a result, statically typed languages are **better at catching mistakes that you didn’t (or even couldn’t) know you
made.** Above and beyond that, they make code **more regular, more predictable, allow new features to be built, and
allow you to communicate information that is guaranteed to be true.**

**None of that is subjective.**

So, all other things being equal, when should you choose which?

I think the analogy in the previous section provides a very good guiding principle. **Use dynamic languages in
situations where you'd use pen & paper**---when you're doing something small, something that will be finished soon,
finished forever, and something that you're building alone, and will always build alone. **In all other situations,
especially large projects with multiple developers that evolve over time, use a language that provides a strong, static
and sound (looking at you, Java) type system.**

Of course, that's almost never how the choice is actually made, but that's a different story.

## Can you have your cake, and eat it too?

For most of my career, I've operated under the assumption that **this choice is all-or-nothing**---either my language,
and my entire application, is typed, or it's not. Either I **optimize the initial phases of a project and pay for it in
the later stages** (dynamically typed languages), or I **optimize for the later stages and pay for it in the initial
phases** (statically typed languages).

However, there are what you call *gradually-typed* languages, where you can **type parts of your code, and leave others
untyped.** In the vast majority of cases, *gradually-typed* is a fancy name for a **by-product of a dynamic language
that added types in retrospect and needed to maintain backwards compatibility,** which is what happened with type hints
in Python and PHP. Often, the capability is not even part of the language itself, and you need an external tool, e.g.
[PHPStan](https://phpstan.org/writing-php-code/phpdoc-types), [mypy](https://mypy-lang.org/) or
[Sorbet](https://sorbet.org/).

While this would seem to be **a very good compromise**---I can start out flexible, and become more rigid as I need
to---the immediate counter-argument is that **these type systems are usually very low-quality**---obviously, when
anything is bolted on as an afterthought, its quality is going to suffer. Additionally, and perhaps even more
importantly---**if a language started out as dynamic, and only added static types in retrospect, the majority of its
users will not actually use them,** or use them correctly, and neither will the majority of its ecosystem. Both of these
issues significantly impact the actual benefits you can reap from this approach.

But let's [steelman](https://en.wikipedia.org/wiki/Straw_man#Steelmanning) the argument and disregard those
issues---after all, we can imagine a language which was designed from the ground-up to be gradually typed (even though,
interestingly enough, there are practically none). There is still another, much more subtle issue with these languages:
**calling out to untyped code practically never requires any special ceremony.**

Most languages and tools have various knobs and dials that allow you to tweak what happens when you do this, e.g. [PHP
doing runtime checks on function boundaries when configured to do
so](https://inspector.dev/why-use-declarestrict_types1-in-php-fast-tips/), [PHPStan warning when its strictness is
dialed up to the highest levels](https://phpstan.org/user-guide/rule-levels), TypeScript offering [strict
mode](https://www.typescriptlang.org/tsconfig/#strict), and [Sorbet](https://sorbet.org/docs/static) doing the same.
However, in all of these languages/tools, anything beyond "cover your eyes and hope for the best" is opt-in, and none of
them tell you at the call-site that you're calling out to untyped code.

In other words, **in gradually typed languages, [types do not color
functions](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/).** I consider this to be a fatal
flaw. **The boundary between typed and untyped code should be like the Korean demilitarized zone---immediately visible,
and not crossed lightly.**

To my knowledge, **there is only one industrial-grade language that gets this right---[Common
Lisp](https://en.wikipedia.org/wiki/Common_Lisp)[^racket].** CL itself is [dynamically, but strongly,
typed](https://lispcookbook.github.io/cl-cookbook/type.html), with an incredibly rich type system---right out of the
box, you have [an excellent sweet spot between flexibility and safety](https://news.ycombinator.com/item?id=41651507).
Furthermore, its macro system allows you to build whatever language feature you need, and naturally, people have. This
is how we got [Coalton](https://coalton-lang.github.io/), an implementation of a practical subset of the [Haskell type
system](https://softwareengineering.stackexchange.com/a/279362), checked during compile-time---as a library.

This means you can:

* **omit types when you don't want them or they don't make sense,** e.g. when writing macros, prototyping, or
  [interacting with the REPL](https://news.ycombinator.com/item?id=23811382)
* at any time, choose to **switch to one of the strongest and most sound type systems in existence,** with 0 performance
  cost (and, since Coalton generates native Lisp type declarations, potentially [significant performance
  gains](https://lispcookbook.github.io/cl-cookbook/performance.html#type-hints))
* **your type system is just a library**---it's not "another thing", it's right there, next to the rest of your code.
  Want to know what a type declaration actually *does*? Just *go-to-definition* on it. Not sure why your program isn't
  compiling? Debug it as you would any other problem. Can you imagine having the ability, when needed, to **understand
  compiler errors by putting a breakpoint in the typechecker?** In Lisp, that's just a normal Tuesday.
  
Last but not least, [Coalton colors
code](https://github.com/coalton-lang/coalton/blob/main/docs/coalton-lisp-interop.md#lisp-calls-coalton-bridge).

[^racket]: <small>An argument could be made for [Typed Racket](https://docs.racket-lang.org/ts-guide/), however a) I'm
personally not convinced it has a track record that would warrant the title 'industrial-grade', and b) while its
approach to the typed/untyped boundary is sound and robust, it doesn't actually visually distinguish the
call-site</small>

## Summary

The debate between statically and dynamically typed languages is often presented as a war, but **I think it's actually
anything but.**

At its core, **static typing optimizes for maintainability**---explicitly modeling data and relationships, catching
subtle bugs early, and enabling rich tooling, predictable design patterns, and concise communication of information to
the reader. On the flip side, **dynamic typing optimizes for speed of initial development.** It shines when **exploring
new ideas, building throwaway prototypes, or working solo on small-scale tasks.**

Like pen & paper and Microsoft Word, they’re **different tools for different jobs,** rather than opposing forces. Most
flame wars arise because **people tend to optimize for different pain points,** based on the different experiences
they've lived through.

That being said, **I strongly disagree with the idea that the choice between them is simply a subjective one**. The
right tool depends on the task, context, and constraints, and in any given scenario, one is usually more appropriate
than the other. **Knowing when to choose which is a mark of good engineering.**

Ideally, one would want to work in an **ecosystem that put both tools at their disposal,** and while most current
gradually-typed languages haven't fleshed out the concept enough to bring any actual value, **Common Lisp with Coalton
is a rare example where you truly can blend both worlds meaningfully** without giving anything up.

**There is no war. Just tools, tradeoffs, and consequences.**

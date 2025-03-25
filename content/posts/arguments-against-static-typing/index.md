---
title: "Arguments against static typing"
date:  2025-03-25
description: "A practical defense of static typing that addresses common complaints, agrees with them, and shows why they make static typing powerful."
summary: "A practical defense of static typing that addresses common complaints, agrees with them, and shows why they make static typing powerful."
tags: ["basics", "types"]
reddit: "https://www.reddit.com/r/programming/comments/1jkm6ew/arguments_against_static_typing/"
hackernews: "https://news.ycombinator.com/item?id=43487861"
---

## Introduction

The debate between static and dynamic typing has been going on for as long as they have existed, and while I think it's
actually [very easy to settle]({{< relref "how-to-win-type-wars/index.md" >}}), I did wanted to take out some time to
address the most common ones.

## Static typing

The essence of statically typed languages is that they **force you to think about certain problems**---specifically, the
shapes of data---and **force you to be explicit** about how you solve them.

For instance, a (proper) statically typed language **will force you to think, and be explicit,** about how you represent
*absence*---is it `null`, `{}` , `[]`, [`Unit`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-unit/),
`Option.empty()`, or something else? Are any of those even ever permissible, and if so, which one(s)? What do they mean
in the context of the problem I’m solving? You have **no choice but to think about this,** solve it in a way that’s
**consistent with the rest of the code base,** and **precisely and explicitly describe that solution** alongside your
code.

Crucially, **statically typed languages are *mean*,** and force you to think about this even when it might make no
practical difference! For example, you might just be interested in the truthiness of the return value, so any (or some,
depending on the language) of the above will do. Perhaps things are completely self-evidently correct no matter the
choice, or you don’t want to decide right away and instead opt to keep your options open, or you specifically **do**
want it to work for completely different data types. But even in those situations, **you simply have no choice,** and
**must** define a specific, well-defined shape for each and every piece of data in a consistent manner, and adhere to
that shape from then on.

Proponents of dynamically typed languages---“dynamic typers”---will balk at this, with a variation of the following
arguments:

- **types constrain my solutions,** and force me to solve problems in a certain way. [Types take away my ~~guns~~
  freedom](https://www.youtube.com/watch?v=0rR9IaXH1M0)!
- **types prolong the time** it takes to deliver a solution
- **types are a waste of time,** because in 99% of cases, I can write perfectly functioning code without them
- **proving my code is safe to the compiler is a waste of time,** because in 99% of cases, my tests already do that
- **types make code cluttered** and difficult to read

Proponents of static languages...pretty much **agree completely!**

### Types constrain my solutions

*Exactly!*

They force you to write code in a well-defined, well-researched manner that was specifically **designed to prevent
certain kinds of errors from happening, and to do so with mathematical certainty.** It doesn’t matter if you’re tired,
stupid, uninformed, malicious, make a mistake or forgot something. If it compiles, it **provably** doesn’t contain these
mistakes, period.

But there’s more. Since it reduces the amount of different ways solutions can be coded, it actually **makes things
easier to understand**---less variety **reduces the cognitive load for the reader.**

Taken further, since **it forces all of us to think in similar ways,** we tend to express ourselves in similar ways,
which means **what you write tends to be closer to what I would have written,** further shortening the time it takes for
me to understand it. This is pretty much the driving force behind things like [Patterns in Enterprise
Software](https://martinfowler.com/articles/enterprisePatterns.html)---“**let’s solve problems consistently**”.

And there’s even more! Less variety = more patterns. A pattern is equivalent to saying that certain assumptions are
guaranteed to be true. And **when certain assumptions hold, features can be designed around them.** This is why static
languages have reliable IDE hinting, go-to-definition/show usages, refactorings, and so on. This is [**why banning GOTO
opened the doors to try-with & exceptions, and the same principle also lies at the heart of structured
concurrency**](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/).

### Types prolong the time it takes to deliver a solution

*Right on!*

**In a sense, I probably burn more time thinking about types, and naming them, than I do on anything else, apart from
research.**

But while types prolong the time it takes to deliver **a** solution, they also dramatically shorten the time it takes to
deliver a **continuously working** solution---one that works correctly **now, and will continue to do so** when
assumptions change, when the project is 10x bigger, when none of the original authors work here anymore, etc.

**Types expose and make explicit the coupling between distant parts of the codebase,** and cause things to **fail
immediately if these parts are inconsistent.** They also prevent you from **collapsing code** for different data types -
if you want a function that operates on both strings and integers, you need to write two functions (or do some truly
funky-JavaScript-like shit that will make reviewers go [Liam Neeson on your
ass](https://www.youtube.com/watch?v=x7gst82ayzw)).

Given that, it should come as no surprise to anyone that it takes more time to design things in a way so that they *are*
consistent and separated. But that same **extra time you spent will get reimbursed 100x by spending less time fixing
bugs due to inconsistent assumptions in far-away places,** especially as time goes on and things change. In this rare
instance, tight coupling is actually a good thing.

Types also **force you to spend time making sense of things, and give names to those things.** **Things that are named
have *meaning*,** and **expressing a solution through something that has meaning makes the result more understandable**
to the people who maintain it (which includes future-you).

Above and beyond what I just wrote, I actually **don't think that types *themselves* take up any time whatsoever**. What
takes up time is the **problems that they force you to think about.** For example, when you're modelling a business
process, they force you to think hard about *what* is actually transforming into *what* and **what the *what's* should
be called.** *That's* what takes long, but that's a measure of both the complexity of the problem and the fact you don't
understand it fully yet, not of types being time-consuming. Actually writting the types themselves is a matter of
seconds, and if the problem is trivial or you understand it well, i.e. "write a function that returns the first element
of a `List`", you spend no time on types at all.

I would go as far as to say that **types actually *decrease* the amount of time it takes for me to deliver a working
solution**. Why? Because I've been using them for so long that **they've shaped my way of thinking** into immediatelly
focusing on *what* goes in and *what* goes out. I'm so used to this, I do it automatically, and each time I'm doing it,
I train my brain to be a little better at it.

As explained above, understanding what the *what's* are, and naming them, **unlocks a much deeper understanding of
what's going on**. Often, it will even uncover blind spots that you hadn't thought of up until then, and you might even
realize you need to go back to your stakeholders and pick their brains some more. If you hadn't done that, you would've
instead **started writting a bunch of code that would've ended up being wrong.** If you were lucky, you would discover
that at some point down the road before going live, but even then, potentially **a lot of that effort would go down the
drain.**

Types allowed you to save all that time by **finding out in advance, before you wrote a single line of code.**

### Types are a waste of time, because in 99% of cases, I can write perfectly functioning code without them

*Fo’ sho!*

If your understanding of the problem **is** complete and correct, and you know it, [~~clap your
hands~~](https://www.youtube.com/watch?v=sCbOGCY-3Uk) it’s easy to not make a mistake. If your understanding of the
problem **isn’t** complete or correct, and you know it, [~~clap your
hands~~](https://www.youtube.com/watch?v=sCbOGCY-3Uk) it’s also easy to not make a mistake. But **when you *think* your
understanding is complete and correct, but it actually isn’t,** or when **your understanding *is* correct, but you end
up writing something else than what you’re thinking**---that’s when it’s **really difficult not to make a mistake.**
Because **you simply can’t know what you don’t know**---that’s the point.

Statically typed languages ***force* you to think hard about the problem.** And when you think you’ve got it right,
**they check your work and call you out if you’re wrong, regardless of how sure you are of yourself.**

**Dynamic languages do not**---they accept what you write without question, they will not check your work, not even if
you *aren’t* sure of yourself.

Now, a lot of the time, maybe even 99% of the time, that doesn’t matter---naturally, you’re a programmer worth your
weight in bytes, and tend to not make mistakes. But then, one day, you *do* make a mistake, and it *does* matter. And
this is the key difference---**while dynamic languages optimize for the 99%, static languages optimize for that 1%**
(because they know that, more often than not, it's actually a lot more than 1%). And that's before you consider
situations where the **mistakes happen just because you make a change that's incompatible with the implicit assumptions
in some far away place.** You have no way of knowing that. You can be the best programmer in the world, and that's still
going to happen at some point---**it's not if, it's when.**

### Proving my code is safe to the compiler is a waste of time, because in 99% of cases, my tests already do that

*Fo’ shizzle my nizzle!*

If **your perception of the problem truly is correct, and your code is also correct, then it’s a waste of time**---the
job is already done. If **your perception is correct, but your code isn’t correct, tests will likely tell you---you
designed them around your correct understanding of the problem**---so again, a waste of time.

But when **your perception is wrong,** then your code is also wrong, but crucially, **in all likelihood, so are your
tests**! When designed based on flawed understanding of the problem, **tests are likely validating a flawed solution,**
not the correct one. And in those situations, you’ll never know, because **you can’t know what you don’t know.**

And that's even before getting into the fact that tests **rarely cover 100% of the code they're supposed to test,** not to
mention that they're code themselves, which makes them **prone to exactly the same types of mistakes you set out to prove
don't exist in the first place!**

**Types are, literally, automatically generated tests that run at compile time.** They are always generated, always
correct, and always run. They are objective, and **confront your solution with reality, not your (possibly flawed)
perception of it.** And because you’re forced to satisfy them, **they prevent you from making mistakes even in
situations when you don’t realize your reasoning is flawed.**

Again---**dynamic languages optimize for the 99%, static languages optimize for that 1%** (because they know that, more
often than not, it's actually a lot more than 1%).

### Types make code cluttered and difficult to read

Well...yes, but actually, no.

```java
<T> MergeMatchedSetMoreStep<R> set(Field<T> field, Select<? extends Record1<T>> value);
```

Anybody who’s not used to typed languages and generics will look at the above and see a bowl of ASCII soup. But I
don’t---on the contrary, not only do I feel completely comfortable reading it, **I actually feel like I’m flying blind
when reading code that doesn't contain such declarations.**

So what’s the difference between me and dynamic typers? **It’s simple---I’m used to it.** I’ve been looking at types for
so long that **they add almost no cognitive load for me.** Is reading French inherently difficult, or is it only
difficult for those who are not used to it? As with beauty, **difficulty is in the eye of the beholder.**

As every static typer will know, it turns out that, **far from making things more difficult, the contrary is true** -
**types make things much clearer, concisely communicating information** that I would otherwise have to parse out of the
solution. And while I do spend (infinitesimally) more time reading it than I would a simple `function set(field,
value)`, it’s **because I’m busy learning something about the method** that I will then take advantage of when reading
its contents. I have the *option* to ignore the types if I want to, and just read the method and parameter names, but I
*choose* not to, because **the usefulness of the information contained in the types far outweighs the usefulness of the
information contained in parameter names.** Chief among the reasons is the fact that, **unlike parameter names, types
are actually checked for correctness,** so I know they don’t lie.

In fact, a personal ideal I use for good code is **"I shouldn’t have to read anything else than the method signature to
understand what’s going on".** That's not always attainable, but that's why it's an ideal---so I can get as close to it
as is reasonable given the circumstances. It shows me true north.

## Summary

It should be clear by now that I'm a strong proponent of static typing, however that does not mean that I think they
should be used blindly, all the time, and without thought. On the contrary, I think that dynamic and static typing are
complementary tools, and [*both* should be used]({{< relref "how-to-win-type-wars/index.md#there-is-no-war" >}}).

However, that in no way means I think that the choice is a matter of mindset, as [some
do](https://world.hey.com/dhh/programming-types-and-mindsets-5b8490bc)---on the contrary, I think the decision is
[almost always
objective]({{< relref "how-to-win-type-wars/index.md#different-tools-for-different-jobs-not-different-strokes-for-different-folks" >}}).

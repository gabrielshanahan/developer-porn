---
title: "Why I don't use for-cycles"
date:  2025-03-18
draft: true
description: "TODO html meta su mmary"
summary: "TODO short article description that is rendered"
tags: ["welcome", "new", "about", "first"]
---

A few months back, a colleague of mine accidentally made an innocent mistake which unfortunatelly resulted in
not-so-innocent consequences. However, it presents an excellent opportunity to talk about two important
subjects---**strong typing** and **declarative transformations**---and show how they
are, in essence, the **same solution to the same problem in different guises**.

## Introduction

First, the problem. In essence, this was the code that was being used in production:

```java
var labelsPerBox = numLabels / numBoxes;
List<Label> result = new ArrayList<>();
for (var boxIdx = 0; boxIdx < numBoxes; boxIdx++) {
    for (var labelIdx = 0; labelIdx < labelsPerBox; labelIdx++) {
        result.add(
            buildLabel(boxes[boxIdx], labelIdx)
        );
    }
}
```

This is the correct code:

```java
var labelsPerBox = numLabels / numBoxes;
List<Label> result = new ArrayList<>();
for (var boxIdx = 0; boxIdx < numBoxes; boxIdx++) {
    for (var labelIdx = 0; labelIdx < labelsPerBox; labelIdx++) {
        result.add(
            buildLabel(boxes[boxIdx], boxIdx * labelsPerBox + labelIdx)
        );
    }
}
```

In other words, **the label needed to be generated based on its “absolute” position in the sequence of boxes**, not the
position in its box. This is a business requirement that cannot be deduced from the code alone---you have to know that
on your way in---but it was understood by my colleague from the very beginning, and it’s what he set out to write. It
just “got lost between the brain and the fingers”, so to speak.

In reality, the code was quite a bit noisier, so it was even harder to spot, but even in this absolutely reduced form,
one could be forgiven for not noticing the mistake, especially when mixed among other changes.

We later discussed what could have been done to prevent this, and my response of “don’t use for-cycles” predictably
generated rumblings of disagreement. In light of that, I wanted to expand on my thoughts on the subject, why I felt
for-cycles were to blame, and why I believe that they should be avoided like the plague, unless there is no other
reasonable tool for the job.

![A picture of a Slack message from Gabriel Shanahan with the text "honestly, by avoiding for-cycles. I think this falls
  into the "off-by-one" type of programming error - easy to make, difficult to spot. Two people disagree via
  emoticons.](rumblings.png)

However, we’ll be taking the scenic route! So before I talk about for-cycles and declarative transformations in a future
article, I’m going to talk about statically vs. dynamically typed languages in this one.

TODO: Continue

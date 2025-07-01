---
title: "Deprecating Eventual Consistency—Applying Principles of Structured Concurrency to Distributed Systems"
date:  2025-07-06
description: "An introduction to structured concurrency, a new way to design distributed systems, and Scoop, a POC orchestration library."
summary: "An introduction to structured concurrency, a new way to design distributed systems, and Scoop, a POC orchestration library."
tags: ["structured cooperation", "distributed systems", "scoop", "services", "microservices"]
---

If you've ever worked as an enterprise developer in any moderately complex company, you've likely encountered
distributed systems of the kind I want to talk about in this post---two or more systems communicating together via a
message queue (MQ), such as [RabbitMQ](https://www.rabbitmq.com/) or [Apache Kafka](https://kafka.apache.org/).
Distributed, message-based systems are ubiquitous in today's programming landscape, especially due to the (now hopefully
at least somewhat tempered) microservice architecture frenzy that swept over our field during the past decade.

Moving away from a [majestic monolith](https://signalvnoise.com/svn3/the-majestic-monolith/) involves significant
tradeoffs, all of which have been documented extensively over the years. It is well known that dealing with distributed
systems is a [famously painful experience](https://martinfowler.com/bliki/MicroservicePremium.html)---data is only
eventually consistent, errors are difficult to trace and debug, and, perhaps most frustratingly, it gets increasingly
more difficult to reason about the system as a whole. This is compounded by the organic way these systems often
form---rather than being a thought-out and planned architectural decision, many start out as an ad hoc solution to a
particular localized problem and then gradually snowball into a mess.

Nothing I've said so far is news---everybody knows that *distributed systems are a pain*.

**But why?**

**In the following posts, I want to convince you that many of the difficulties traditionally associated with distributed
systems are not actually unique to distributed systems at all. In fact, they're something our industry has encountered,
and solved, not once, but *twice* before---once, around 1968, when [we started thinking about the problems with
`GOTO`](https://homepages.cwi.nl/~storm/teaching/reader/Dijkstra68.pdf), and then again more recently, around 2016, with
[the advent of structured concurrency](https://en.wikipedia.org/wiki/Structured_concurrency).**

**What's more, the solutions to both problems revolve around essentially the same idea, and it turns out that this same
idea---a single, simple rule---is also applicable to how we design distributed systems. Applying that idea not only
prevents many of these difficulties from ever arising, but also opens the door to features that are not readily
available using current approaches, such as (but not limited to) true distributed exceptions---stack traces that span
multiple services, and what amounts to stack unwinding across services. And perhaps most importantly, it makes these
systems significantly easier to reason about.**

**However, those discussions, while interesting and educational, are also rather theoretical, and let's be honest---that's
not everyone's cup of tea. So before I give you a tour of the ivory tower, I want to stay on the ground for a little
while, and show you what you get when you actually apply all that ivory business. To that end, I built
[Scoop](https://github.com/gabrielshanahan/scoop), and in this post, I want to talk about what it is, and what it can
do. Hopefully, that will motivate you to further explore the reasoning that led me to build it in this particular
fashion, and who knows, maybe you'll even learn a thing or two along the way.**

In this post, I'm going to concentrate on what Scoop can do, without going into too much detail about *how*. That will
be the topic of the [subsequent post](posts/implementing-structured-cooperation), which will give you a helicopter
overview of how Scoop and its features are implemented, and concentrate on a few fundamental topics that I purposefully
avoid talking about here. Finally, in the third and final post, I'll frame the core concept Scoop is built around,
something I'm calling **structured cooperation**, in a broader context, and show you how it's the natural continuation
of an idea that has, in one form or another, been shaping our industry for over half a century.

## What is Scoop, and what did I build it with?

Scoop amounts to what you might call an *orchestration library*---it helps you write business operations in distributed
systems. In that sense, it is similar to, e.g., [Temporal](https://temporal.io/), or, to an extent,
[Axon](https://www.axoniq.io/products/axon-framework).[^axon] Scoop is small---it can be read cover-to-cover in a few
hours, and most of the magic happens in ~500 lines of (heavily documented) SQL.

[^axon]: <small>Scoop has nothing to do with event sourcing. I'm including the comparison solely because Axon, by
design, is built for distributed environments, and forces you to model things accordingly.</small>

The primary purpose of Scoop, at least at this point, is to convey an idea. Scoop is a POC, not a production-ready
library. That being said, feature-wise, it packs quite a punch if I may say so myself, especially given how small it is.

The principles upon which Scoop is built, along with the contents of these posts, are language and infrastructure
agnostic, and no specific knowledge is assumed here, other than familiarity with any mainstream programming language,
SQL, and a vague familiarity with MQ's and distributed systems in general (e.g., you know what a *message* or a *topic*
is).

Of course, I did need to write it in something. Scoop is written in Kotlin, on top of [Quarkus](https://quarkus.io/),
and uses [Postgres for everything](https://postgresforeverything.com/). I chose Kotlin primarily because of its syntax
and type system, and Quarkus since it allows using both blocking and reactive database drivers in a single application,
and I wanted to write Scoop in both flavors, for educational and demonstrational purposes. However, I also wanted to
target as wide an audience as possible, and tried to minimize the number of assumptions I made about what my audience
might be familiar with.

Therefore, I:
* don't use any fancy Kotlin features[^coroutine], apart from [extensions](https://kotlinlang.org/docs/extensions.html),
* deliberately chose to implement a simple MQ on top of Postgres instead of using an established MQ,
* try to keep abstraction to a minimum,
* only use Quarkus for dependency injection,
* don't use an ORM,
* write raw SQL everywhere.

[^coroutine]: <small>At times, you might notice the word *coroutine* being used, e.g., in package names. That isn't a
reference to [Kotlin coroutines](https://kotlinlang.org/docs/coroutines-overview.html), but rather to Scoop's own
(distributed) implementation. Kotlin coroutines are not used anywhere in Scoop.</small>

The result certainly won't win any beauty contests, and I know my JVM brothers and sisters have already fainted in
horror at *"keep abstraction to a minimum,"* but hopefully, these choices will make the idea accessible to programmers
from virtually any background, and conveying the idea is all I care about.

There is, unfortunately, some dancing around [Jackson](https://github.com/FasterXML/jackson), the JSON library. There is
always some dancing around Jackson.

## The Saga Begins
One of the most painful consequences of moving to a distributed architecture is the impact it has on transactional
bounderies---you can no longer complete an entire "business operation" within the confines of a single, traditional
database transaction. There are various approaches that work around this issue, such as [two-phased
commits](https://en.wikipedia.org/wiki/Two-phase_commit_protocol), but a common one is the [saga
pattern](https://microservices.io/patterns/data/saga.html). In a saga, you give up
[atomicity](https://en.wikipedia.org/wiki/Atomicity_(database_systems)) by modelling a business operation as "a sequence
of local transactions"---basically you break up the operation into a set of "steps". Each step is wrapped in a regular
transaction, and messages are emitted during their execution, which trigger operations in other services.

This is the approach Scoop takes. Here is an example of a saga in Scoop:

```kotlin
// saga() actually takes a second required parameter,
// but that's one of the things I'll be glossing over
// in this article
val myHandler = saga(name = "my-handler") {
    step { scope, message ->
        println("Hello from step 1 of my-handler!")
        scope.launch("some-topic", JsonObject().put("from", "my-handler"))
    }

    step { scope, message ->
        println("Hello from step 2 of my-handler!")
    }
}
```

*"What's that `scope` thing?"* I hear you exclaim.

Don't worry about it---for now, all you need to understand is that `scope.launch(<topic>, <payload>)` arranges for a
message with `<payload>` to be published on `<topic>` once the (local) transaction of the step commits.[^outbox] Yes, yes, using
a raw `JsonObject` is not at all how this would be done in an actual production implementation, but that's not what
Scoop is.

[^outbox]:<small>Since Scoop uses its own MQ on top of Postgres, publishing messages only when the transaction commits
is easy to do---the messages are part of the transaction. If it were implemented in a realistic context, an external MQ
would likely be involved, which means this would need to use something like the [outbox
pattern](https://microservices.io/patterns/data/transactional-outbox.html).</small>

To have this saga actually do something, you subscribe it to some *topic*:

```kotlin
// MessageQueue is part of Scoop, you just 
// inject it like any other component

messageQueue.subscribe("some-topic", myHandler)

// This is how you would publish a message (with 
// an empty payload) on that same topic
messageQueue.launch("some-topic", JsonObject())
```

After the `subscribe` line, whenever a message is published on `some-topic`, the saga is run, step by step---we'll talk
more about that in a second. You can subscribe multiple sagas to the same topic. You can also scale sagas
horizontally---running 1 instance or 100 *just works*, no configuration needed. Just keep in mind that each step can
potentially be run by a different instance of the service.[^context]

[^context]: <small>You might be wondering how you share data between steps, if you have no guarantee they will all be
run by the same instance of a service. This is what `CooperationContext` is for, and we'll talk about it in the next
article. Basically, it's the equivalent of [reactive
context](https://projectreactor.io/docs/core/release/reference/advancedFeatures/context.html),
[CoroutineContext](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.coroutines/-coroutine-context/), etc.</small>

## Rollbacks & Coordination

There are two rather unique issues that arise with sagas (in general, not just in Scoop).

The first is how the equivalent of a transaction rollback happen---if an error happens somewhere down the road, the
transactions of the previous steps are already committed, so you can't do a traditional rollback. Instead, you basically
need to write code that manually undoes whatever changes you made in each step, usually called a "compensating action",
and when an error happens, arrange for this code to run. I'll show you how that's done in Scoop in a moment.

The second is how you coordinate the steps, since you ideally only want to run a subsequent step when the previous one
has finished. This is actually trickier than it sounds, and there are two fundamental approaches that you can take,
usually called *orchestration* and *choreography*.

### Orchestration

In an orchestrated saga, the saga is usually explicit, which means that there is a specific place in some codebase where
the saga is written out in its entirety, and the "god service" that runs this code *orchestrates* the entire business
operation by calling out to all other services in the proper order to achieve the desired result. This makes it easy to
reason about, but also feels like a step in the opposite direction from the decoupled, decentralized mindset that
SOA/microservices traditionally embody. The god service must have direct knowledge of all the services it calls out to
and how they respond, therefore becoming tightly coupled to them. Essentially, this approach amounts to normal RPC, just
with an MQ sandwiched between the systems for better resilience, and possibly some other bells and whistles.

### Choreography

In a choreographed saga, the saga is often (but not always) implicit---there's no single place in any codebase where the
steps are laid out. Instead, you trigger the saga by emitting a particular message, which is handled by whatever handler
represents the first step. This handler then emits its own messages, which triggers the next step, and so on. Subsequent
steps are triggered in response to intercepting a particular message emitted by the previous step, and messages are
fired and forgotten---handlers don't receive responses to the messages they emit. That's more in line with the
decentralized mindset of microservices, but leads to incredibly messy systems that are difficult to reason about.
There's actually a very deep reason for the inevitable messiness, and we'll talk about it at length in the final post of
this series.

Even if you ignore that, it still gets really tricky really fast. Sometimes, there's no reasonable business message to
emit, because some part of the saga ends up not actually doing anything (e.g., we're handling `CreateCustomer`, but the
customer already exists and already contains that same data, so neither `CustomerCreated` nor `CustomerChanged` make
sense). In that case, you probably want to continue with the next step anyway, but there's no sensible message to emit,
and so nothing to actually react to. This is commonly a problem in event-sourced applications, where the messages
themselves *are* the stored data, so you don't want to be emitting messages that don't mean anything. In other
situations, you actually want to react to a combination of messages arriving (e.g., `CustomerCreated` *and*
`ContractCreated`). Sometimes the order matters to you; sometimes it doesn't. Usually, the payloads matter
(`ContractCreated` needs `customer_id` matching the `id` in `CustomerCreated`). Sometimes you want to wait for a
particular amount of messages of a particular type (e.g., the appropriate number of `LineItemCreated`) before continuing.

Now, to be clear---I'm not saying these problems don't have solutions. They obviously do, otherwise these patterns
couldn't be used. I'm just saying that there's significant baggage that needs to be dealt with, and dealing with it can
feel like a game of whack-a-mole in terms of the tradeoffs one is forced to make, and involve significant effort to
implement and maintain.

But Scoop doesn't quite fit into either of these categories.

## The Rule of Structured Cooperation

Sagas in Scoop are orchestrated in the sense that they are explicit---you will find a specific place in some codebase
where a saga can be seen in its entirety. But they are also choreographed, in the sense that they do not directly call
any services, and simply fire off one or more messages without processing responses to them.

However, they do obey a rule that's somewhere halfway between orchestrated and choreographed:

**When they reach the end of a step in which messages were emitted, sagas suspend their execution and don't continue
with the next step until all handlers of those messages have finished executing.**

This simple rule is at the heart of Scoop, at the heart of structured cooperation, and at the heart of everything these
articles are about.[^heart] And, as you'll see in the rest of the article, having sagas adhere to this rule has some
pretty profound consequences.

[^heart]: <small>If you're getting structured concurrency vibes, you're exactly right! That's where structured
cooperation gets its name from.</small>

Let's take a look at some of them.

## No more eventual consistency

In a system where structured cooperation is obeyed, it's impossible to trip over eventual consistency, unless you
deliberately go out of your way to inflict it upon yourself. That's because all handlers of all messages emitted in any
previous steps are guaranteed to have finished successfully (and all handlers of any messages *they* emitted, and so
on). Crucially, *you don't need to do anything yourself*---you don't need to check for side effects other services might
have in order to determine if they've finished or not.[^how-do-i-know] If a step is executing, the previous steps and
all their side effects have finished executing, period.

[^how-do-i-know]: <small>Wait---how does Scoop find out *which* handlers it was supposed to have been waiting for in the
first place? That's a very important question and not trivial to answer in the context of distributed systems. The way
you decide to answer it---because it *will* be up to you---is one of the key decisions you need to make when using
Scoop, depends on how your system is architected, has implications related to the [CAP
theorem](https://en.wikipedia.org/wiki/CAP_theorem), and is what that second required parameter to `saga` I mentioned
earlier---an instance of `EventLoopStrategy`---is there for. We'll discuss this whole topic at length in the following
article.</small>

Say you're not using structured cooperation, and, while importing some data, fire off a `CustomerCreation` and
`CustomerContractCreation` message. Multiple services are listening to those messages and reacting to them. It could
easily happen that a service starts reacting to `CustomerContractCreation` before it finishes reacting to
`CustomerCreation`, which will lead to a `CustomerNotFound` error. There are ways to deal with that, but it's a can of
worms, e.g., how do you distinguish between a customer that hasn't been created *yet* vs. an actual faulty message?

If your system uses structured cooperation, that can never happen. `CustomerCreation` is fired in a step preceding the
one in which `CustomerContractCreation` is, so you can guarantee that the *entire* system is in a consistent state once
you start executing any subsequent step.

That guarantee is at the heart of what makes structured cooperation so powerful---it allows you to *reason* about the
state of the *entire* system without actually writing any code that would require you to know anything about it. You
don't need to care if there is one, zero, or 500 handlers listening to a message you emitted. You don't need to care if
they themselves need to fire off 1000 messages of their own in order to react to your message, or run some dreadful
local calculation that takes until October to complete. You fire off your messages and suspend, and Scoop will [wake you
up when September ends](https://www.youtube.com/watch?v=NU9JoFKlaZ0).

## Distributed Exceptions

In a system where structured cooperation is obeyed, if I'm inside some saga that's handling some message, and that
message is a "child" message of some other saga---i.e., it was emitted from within another saga---I'm guaranteed that
the "parent" saga is still running, patiently waiting at the end of the step that emitted the message. Therefore, if my
saga fails by throwing an (unhandled) exception, I have the option to propagate that exception to the parent, and
rethrow it there. If the parent doesn't handle it, it bubbles up to *its* parent, and so on---exactly in the same
fashion as they would in regular code.

### Distributed stack traces

The first thing this allows you to do is to build something akin to a distributed stack trace---a description of the
place an error happened, and how the "thread of execution" got there, but *across multiple services*. This overlaps with
the information you get from [distributed
tracing](https://microservices.io/patterns/observability/distributed-tracing.html), except it doesn't require any
separate technology or instrumentation---it's right there, inside your exception, where you need and expect it.

An example is in order (notice we're naming the steps here---optional, but it makes the exceptions more informative):

```kotlin
// Note: Each of the following could, in theory,
//       run in a completely different service,
//       and be written in a completely different
//       language!

// Pretend it listens to "parent-topic"
saga(name = "parent-handler") {
    step("First parent step") { scope, message ->
        logger.log("1")
        scope.launch("child-topic", JsonObject())
    }

    step("Second parent step") { scope, message ->
        logger.log("This will not execute")
    }
}

// Pretend it listens to "child-topic"
saga(name = "child-handler") {
    step("First child step") { scope, message ->
        logger.log("2")
    }

    step("Second child step") { scope, message ->
        logger.log("3")
        scope.launch("child-topic", JsonObject())
    }
}

// Pretend it listens to "grandchild-topic"
saga(name = "grandchild-handler") {
    step("First grandchild step") { scope, message ->
        logger.log("4")
    }

    step("Second grandchild step") { scope, message ->
        logger.log("5")
        throw MyException("My exception message")
    }
}

```

In the above, the log entries would appear in the expected `12345` order (but, naturally, if each saga were running in a
different service, the strings would get logged to different places).

Additionally, in the service running `grandchild-handler`, the following exception would be visible (for brevity, I'm
truncating the stack trace here):

```json
{
  "type": "io.github.gabrielshanahan.scoop.blocking.coroutine.MyException",
  "causes": [],
  "source": "grandchild-handler[0197dfd8-a424-7712-926f-b557d00203c0]",
  "message": "My exception message",
  "stackTrace": [
    {
      "fileName": "DemoTest.kt",
      "className": "io.github.gabrielshanahan.scoop.blocking.coroutine.DemoTest",
      "lineNumber": 48,
      "functionName": "demoTest$lambda$8$lambda$7"
    },
    ...
  ]
}
```

After that, in the service running `child-handler`, the following exception would become visible (again truncating the
stack trace):

```json
{
  "type": "io.github.gabrielshanahan.scoop.shared.coroutine.eventloop.ChildRolledBackException",
  "causes": [
    {
      "type": "io.github.gabrielshanahan.scoop.blocking.coroutine.MyException",
      "causes": [],
      "source": "grandchild-handler[0197dfd8-a424-7712-926f-b557d00203c0]",
      "message": "My exception message",
      "stackTrace": [
        {
          "fileName": "DemoTest.kt",
          "className": "io.github.gabrielshanahan.scoop.blocking.coroutine.DemoTest",
          "lineNumber": 48,
          "functionName": "demoTest$lambda$8$lambda$7"
        },
        ...
      ]
    }
  ],
  "source": "child-handler[0197dfd8-a424-73a9-926e-82e59ae4498a]",
  "message": "Child failure occurred while suspended in step [Second child step]",
  "stackTrace": []
}

```

Finally, in the service running `parent-handler`, the following exception would become visible (truncating the
stack trace again):

```json
{
  "type": "io.github.gabrielshanahan.scoop.shared.coroutine.eventloop.ChildRolledBackException",
  "causes": [
    {
      "type": "io.github.gabrielshanahan.scoop.shared.coroutine.eventloop.ChildRolledBackException",
      "causes": [
        {
          "type": "io.github.gabrielshanahan.scoop.blocking.coroutine.MyException",
          "causes": [],
          "source": "grandchild-handler[0197dfd8-a424-7712-926f-b557d00203c0]",
          "message": "My exception message",
          "stackTrace": [
            {
              "fileName": "DemoTest.kt",
              "className": "io.github.gabrielshanahan.scoop.blocking.coroutine.DemoTest",
              "lineNumber": 48,
              "functionName": "demoTest$lambda$8$lambda$7"
            },
            ...
          ]
        }
      ],
      "source": "child-handler[0197dfd8-a424-73a9-926e-82e59ae4498a]",
      "message": "Child failure occurred while suspended in step [Second child step]",
      "stackTrace": []
    }
  ],
  "source": "parent-handler[0197dfd8-a3ec-7e97-9a99-1ca8a5f1598c]",
  "message": "Child failure occurred while suspended in step [First parent step]",
  "stackTrace": []
}
```

Those UUIDs next to the handler name are there because all sagas in Scoop are horizontally scalable by design, and the
UUID identifies the actual instance of the saga that executed that particular step. We'll talk more about Scoop's
execution model in the following article.

You could even take the above a step further, store the stack trace at the point each message is actually emitted, and
"concatenate" it with the stack trace of the child to get even more precise information. Scoop, being a POC, keeps things
simple and doesn't do that, but it could.

### Distributed stack unwinding

I mentioned earlier that sagas need compensating actions in order to revert the changes they made when an error causes
them to fail. Since parents wait for their children to finish executing, Scoop can execute rollbacks in a very
structured and predictable way, in effect achieving the equivalent of stack unwinding, but *across multiple services*.

Compensating actions are defined as part of the step that they roll back and are executed in the opposite order to the
steps---subsequent steps are rolled back before preceding ones.

Let's look at an example:

```kotlin
saga(name = "parent-handler") {
    step(
        "First parent step",
        invoke = { scope, message ->
            logger.log("1")
            scope.launch("child-topic", JsonObject())
        },
        rollback = { scope, message, throwable ->
            logger.log("9")
        }
    )

    step(
        "Second parent step",
        invoke = { scope, message -> println("This will never print") }
    )
}

saga(name = "child-handler") {
    step(
        "First child step",
        invoke = { scope, message ->
            logger.log("2")

        },
        rollback = { scope, message, throwable ->
            logger.log("8")
        }
    )

    step(
        "Second child step",
        invoke = { scope, message ->
            logger.log("3")
            scope.launch("grandchild-topic", JsonObject())
        },
        rollback = { scope, message, throwable ->
            logger.log("7")
        }
    )
}

saga(name = "grandchild-handler") {
    step(
        "First grandchild step",
        invoke = { scope, message ->
            logger.log("4")
        },
        rollback = { scope, message, throwable ->
            logger.log("6")
        }
    )

    step(
        "Second grandchild step",
        invoke = { scope, message ->
            logger.log("5")
            throw MyException("My exception message")
        },
        rollback = { scope, message, throwable ->
            logger.log(
                """
                This will not execute, because the transaction
                hadn't committed yet when the exception was thrown,
                so a standard transaction rollback happened and there's
                nothing to compensate for.
                """
            )
        }
    )
}

```

Follow the numbers to understand in what order things execute, but it's pretty intuitive---you're basically rolling back
time.

What about if there are multiple handlers listening to one topic, and only one of them fails, while the others succeed?
Glad you asked!

```kotlin
saga(name = "parent-handler") {
    step(
        "First parent step",
        invoke = { scope, message ->
            logger.log("1")
            scope.launch("child-topic", JsonObject())
        },
        rollback = { scope, message, throwable ->
            logger.log("7")
        }
    )

    step(
        "Second parent step",
        invoke = { scope, message -> println("This will never print") }
    )
}

saga(name = "child-handler-1") {
    step(
        "First child-1 step",
        invoke = { scope, message ->
            logger.log("2a")

        },
        rollback = { scope, message, throwable ->
            logger.log("6")
        }
    )

    step(
        "Second child-1 step",
        invoke = { scope, message ->
            logger.log("3a")
            scope.launch("grandchild-topic", JsonObject())
        },
        rollback = { scope, message, throwable ->
            logger.log("5")
        }
    )
}

saga(name = "child-handler-2") {
    step(
        "First child-2 step",
        invoke = { scope, message ->
            logger.log("2b")
        },
        rollback = { scope, message, throwable ->
            logger.log("4b")
        }
    )

    step(
        "Second child-2 step",
        invoke = { scope, message ->
            logger.log("3b")
            throw MyException("My exception message")
        },
        rollback = { scope, message, throwable ->
            logger.log(
                """
                This will not execute, because the transaction
                hadn't committed yet when the exception was thrown,
                so a standard transaction rollback happened and there's
                nothing to compensate for.
                """
            )
        }
    )
}

```

Again, keep in mind that each of those can potentially be running in a completely different service, written in a
completely different language (assuming the same structured cooperation protocol is implemented in that language---more
on that in the next article).

The failing child handler first rolls itself back, after which control is transferred to the parent. The parent sees
that one of its children has failed, so it triggers a rollback of the remaining children, waits for them to complete,
then rolls back itself.

Notice how I logged some of the numbers with a letter---that's to represent that these blocks are running in parallel, so
you can't guarantee their relative order. You could get any of `2a-2b-3a-3b-4b`, `2a-3a-2b-3b-4b`, `2a-2b-3b-4b-3a`, or
any other combination where each `2x` comes before `3x` and `3b` comes before `4b`. The rest of the logs will be ordered
deterministically.

One last example: if any messages were emitted during any step that is being rolled back, compensating actions of those
"child" handlers are run first.

```kotlin
saga(name = "parent-handler") {
    step(
        "First parent step",
        invoke = { scope, message ->
            logger.log("1")
            scope.launch("child-topic", JsonObject())
        },
        rollback = { scope, message, throwable ->
            logger.log("7")
        }
    )

    step(
        "Second parent step",
        invoke = { scope, message -> 
            logger.log("4")
            throw MyException("My exception message")
        },
        rollback = { scope, message, throwable ->
            logger.log(
                """
                This will not execute, because the transaction
                hadn't committed yet when the exception was thrown,
                so a standard transaction rollback happened and there's
                nothing to compensate for.
                """
            )
        }
    )
}

saga(name = "child-handler") {
    step(
        "First child step",
        invoke = { scope, message ->
            logger.log("2")

        },
        rollback = { scope, message, throwable ->
            logger.log("6")
        }
    )

    step(
        "Second child step",
        invoke = { scope, message ->
            logger.log("3")
        },
        rollback = { scope, message, throwable ->
            logger.log("5")
        }
    )
}

```

There are other cases I'm not discussing here---what if there are more than two child handlers? What if more than one
handler fails? What if a rollback step fails? I'll discuss some of these in the next article; others, I'll leave up to
the motivated reader to look up in tests. For now, suffice it to say that in all those scenarios, Scoop is well-behaved,
and you can probably figure out what that behavior is just by thinking about what it *should be*.

As a consequence of this approach to handling failures, you get a lot of non-trivial features for free or very little
work, such as cancellations, timeouts, rollbacks triggered by a user action, and more. Scoop supports all of these, and
we'll talk about some of them in the next article.

## Resource handling

Another key feature recovered by adhering to structured cooperation is *resource handling*. By that, I mean the
distributed analogue to various language constructs that allow you to delimit a block of code within which a resource is
available, while also ensuring that resource is cleaned up regardless of how that block is exited (normally or
exceptionally). This is typically done via `try-finally`.

A resource is anything that's considered expensive. In the context of distributed systems, think less "opening a file"
and more "spinning up a cluster of 100 servers to run a calculation".

In Scoop, because of the way failures are guaranteed to propagate, this is easy to do:

```kotlin

saga("root-handler") {
    tryFinallyStep(
        invoke = { scope, message ->
            k8Service.spinUp(requestId = "123", num = 100)
            scope.launch("do-intensive-calculation", JsonObject())
        },
        finally = { scope, message ->
            k8Service.spinDown(requestId = "123")
        },
    )
}

```

What's important is that `tryFinallyStep` isn't some special primitive---you can build it yourself using what we've
already introduced, plus a single additional thing, `CooperationContext`, which we'll talk about in the next article.

Alternatively, you could wrap the `k8Service` in a saga of its own, and take advantage of the way Scoop works natively.

```kotlin
// Listens on "k8-spinup" topic
saga(name = "k8-spinup") {
    step(
        invoke = { scope, message ->
            k8Service.spinUp(<extract params from message>)
        },
        rollback = { scope, message, throwable ->
            k8Service.spinDown(<extract params from message>)
        }
    )
}

// Listens on "k8-spindown" topic
saga(name = "k8-spindown") {
    step { scope, message ->
        k8Service.spinDown(<extract params from message>)
    }
}


saga(name = "root-handler") {
    step { scope, message ->
        scope.launch("k8-spinup", JsonObject().put("request-id", 123).put("num", 100))
    }
    step { scope, message ->
        scope.launch("do-intensive-calculation", JsonObject())
    }
    step { scope, message ->
        scope.launch(k8-spindown", JsonObject().put("request-id", 123))
    }
}

```

If rolling back whatever `do-intensive-calculation` entailed was itself also intensive, you could even consider having
`k8Service.spinUp` as a compensating action for the `k8-spindown` saga step. The sky's the limit here.

## What if I don't want to cooperate?

The fundamental way structured cooperation works is by *synchronizing* parts of a distributed system---in essence,
structured cooperation is a synchronization primitive, much like structured concurrency is. It allows you to make
explicit things that *depend* on each other, by allowing you to *order* them so that *that which depends* only starts
executing after *that which is depended on* has finished executing. Components of the distributed system *cooperate* to
ensure this is always the case, waiting for each other if needed.

However, while rare, there can be times where you simply *don't* want this behavior---where you want to fire off a
message that's *independent* of the operation you're implementing, for whatever reason.

It's possible to do this in Scoop:


```kotlin
saga(name = "root-handler") {
    step { scope, message ->
        scope.launchOnGlobalScope("some-topic", JsonObject())
    }
}

```

In the above, you're explicitly saying that you're launching a completely independent hierarchy of messages. You're not
waiting for it to complete. If it fails, you won't be notified about it. You *can't*, because you're not
waiting---there's nobody to notify. If your saga rolls back, that message hierarchy will not be notified. It can't be,
because who knows what state it's in---it might not have been completed yet, or it might have already been rolled back,
or it might be in the process of rolling back, or something else.

Doing it like this has two advantages: 

1) you're being explicit---the `launchOnGlobalScope` is immediately visible, can be searched for, etc.,

2) whatever `some-topic` handlers may be launched, *they* can still participate in structured cooperation amongst
themselves.

So in effect, you get the best of both worlds---the ability to synchronize the parts of a distributed system that *need*
to be synchronized, while also not needlessly slowing down the parts that *don't*. Sometimes, the stars align, and you
get to have your cake and eat it too.

Incidentally, basically the same thing happens when only part of the (distributed) system implements structured
cooperation, and another part doesn't. The part that doesn't is just independent of the part that does, and you have no
guarantees about anything that happens there, but it doesn't stop you from using structured cooperation in some subset
of your system. As a consequence, you can switch to using structured cooperation *gradually*, service by service.

## Wrapping up

I hope I've started to convince you that structured cooperation will make your interactions with distributed systems
dramatically simpler, or at least piqued your curiosity. In the [next
article](posts/implementing-structured-cooperation), we'll take a closer look at how structured cooperation is
implemented in Scoop.

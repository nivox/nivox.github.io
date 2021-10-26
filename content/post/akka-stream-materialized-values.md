---
title: "Demystifying Akka Streams' Materialized Values"
date: 2021-10-26T19:07:21Z
draft: true
---

One aspect of Akka Streams newcomers often have difficulty to grok is the one of *materialized values*. This concept comes up even in the simplest of example and is often glossed over without a satisfactory explaination. This creates an aura of mystery which mislead people into deeming it far more complex than what it actually is.

I’ve witnessed developer beaing on their toes regarding *materialized values* even after becoming comfortable with some of Akka Streams’ undeniably more complex features.

In this article I’m gonna try to give an in depth coverage of all there is to know about *materialized values* using as a guide the questions and doubts I had while learning the library:
- why are they needed?
- how do they compose?
- how can we return our own values?

## Why are they needed?

The short answer to this question is that *materialized values* gives us a commincation channel with the various stages of our stream.

In case of an *hello world* example this channel is used to inform us of the completion of the stream.

%[https://gist.github.com/nivox/db1d6bb89df7938eb4a9ae320d49f262]

The same pattern applies when we want to access the value *computed* by a stream.

%[https://gist.github.com/nivox/c3a05772dcbd71444589a357aa3f86ca]

This however is only one (even if prevalent) use of these communication channels.

Another, slighly more complicated, use of *materialization value* could arise when we have an unbounded source (for example producing messages read from a Kafka topic) and we want to perform a graceful shutdown. This requires sending a command to the source instructing it to stop polling for new data and complete.

%[https://gist.github.com/nivox/22a726d1aa0b471e1185846e9db6c691]

Ok, so we need a way for the stream to communicate with the outside world, but is it really necessary to introduce this complexity? Couldn't we achieve the same result by simply using closures to provide the stream feedback channel and the control interface?

Let's try to implement this strategy for the feedback channel.

%[https://gist.github.com/nivox/b4dff9b05ad228239400f847f700eccb]

Everything seem to work as expected! We achieved the same outcome without relying on the value generated as a result of running the stream.

However what would happen if we were to wait for the `streamDone` future to complete and then run the stream a second time? 

Well in this case we would have a failure trying to comple the promise: once a promise has been completed it cannot change its value, thus every attempt at doing so by calling `success` or `fail` on it raises an exception. Even worse, if we were to try and use the promise to get another future to wait for the second run termination, we would get back an already completed future with the result of the first run, effectively making it impossible to receive any signal from the second run.

This experiment allows us to conclude that *materialized values* are necessary to enable stream stages to be reused multiple times. By having stages create their communication channel only when the stream is run Akka Streams ensures that different stream instantiation are independent.

## How do they compose? 
Now that we understand why *materialized values* are needed let's try and shed some light on how they work. 

First of all it is important to note that *materialized values* are not some special properties of sinks and sources. Indeed every stage in Akka Streams **needs** to produce a value during the materialization phase (i.e. when the stream is executed). In case a stage doesn't have anything meaningful to to produce it is convention to use the singleton type `NotUsed`.

When we want to connect 2 independently defined stages we incur in a problem: the result of this composition will be itself a stage that need to declare the type it will produce during the materialization. But given that we are just connecting 2 already define stages, which of the 2 *materialized values* should we?

We might think that a sensible solution is to just collect all of them into a list and let the user decide how to handle it. This approach however has a pretty evident disadvantage: given that each stage can materialize a value of any type, the resulting list would need to be a `List[Any]`. We might be tempted to try and exploit tuples to regain our types (by adding a lot of specialized operators for all the possible arities of the stages, or by rely on a library like Shapeless) however we soon realize that this would become unmanageable as the number of stages in our pipeline increases.

So we reach the conclusion that the better strategy is to deal with *materialized values* while combining stages. This indeed is the same conclusion Akka's authors have come up with.

To this end *Akka Streams* offers us 2 operators `viaMat` and `toMat` which require us to provide a combination function used to produce the *materialization value* of the stages composition. Most of the time what we are interested in is just to select one of the two stages' values or possibly to grab them both. This is so common that the library provides an implementation of these strategies:
- `Keep.left`: select the *materialization value* of the left (upstream) stage
- `Keep.right`: select the *materialization value* of the right (downstream) stage
- `Keep.both`: collet both *materialization values* into a tuple

To further improve developer convenience and code readability *Akka Streams* provides a variation of the above operators which automatically apply the `Keep.left` combination function: `via` and `to`.

## A concrete example
In order to fix into our mind all the things we've said so far let's try and play with a simple example:

```scala
trait ControlInterface {
  def stop: Unit
}

val source: Source[Int, ControlInterface] = ???
val flow1: Flow[Int, Int, NotUsed] = Flow[Int].map(x => x * x)
val flow2: Flow[Int, Int, NotUsed] = Flow[Int].map(x => x + 1)
val sink: Sink[Int, Future[Done]] = Sink.foreach(println)

val (control, doneF): (ControlInterface, Future[Done]) = source
  .viaMat(flow1)(Keep.left) // explicitly specifying Keep.left
  .via(flow2)               // implicitly specifying Keep.left
  .toMat(sink)(Keep.both)   // collecting both values
  .run()

system.scheduler.scheduleOnce(2.seconds) { 
  println("Stopping the source")
  control.stop 
}

doneF.onComplete { result =>
  result match {
    case Success(_) => println("The stream completed successfully")
    case Failure(ex) => println(s"The stream failed with exception: ${ex}")
  }
  system.terminate()
}
```

We want to build a simple stream which given a source of integers, computes their square, increments the result by one and finally prints them to video. The catch is that we want to be able to control when the source should stop emitting new element from outside the stream. To do this we have our source materialize a `ControlInterface`. In a more realistic application this interface might exposed via an http endpoint, but in this example we limit ourself to simply call it after a fixed delay. In order to properly wait for all elements to have been processed before terminating the program, we need to also need to have access to the `Future[Done]` produced by our sink.

Now that we understand how *materialized values* composes we can use the operators we discussed in the previous section to get a tuple containing all the things we are interested in.

The following diagram illustrates how the various combination function are chained in order to obtain our end result.
![Materialized Values composition diagram](/images/post/akka-stream-materialized-values/combination.png)

## How can we return our own values?





---
title: "Demystifying Akka Streams' Materialized Values"
date: 2021-10-26T19:07:21Z
draft: true
tags: ["akka", "stream"]
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

```scala
val done: Future[Done] = Source.repeat("Hello world")
    .take(3)
    .runWith(Sink.foreach(println))
```

The same pattern applies when we want to access the value *computed* by a stream.

```scala
val result: Future[Int] = Source(1 to 10)
    .runWith(Sink.fold(0)(_ + _))
```

This however is only one (even if prevalent) use of these communication channels.

Another, slighly more complicated, use of *materialization value* could arise when we have an unbounded source (for example producing messages read from a Kafka topic) and we want to perform a graceful shutdown. This requires sending a command to the source instructing it to stop polling for new data and complete.

```scala
val stream: RunnableGraph[Control, Future[Done]] = Consumer
    .plainSource(consumerSettings, Subscriptions.topic("my-topic")
    .toMat(Sink.foreach(println))(Keep.both)

val (control, done) = stream.run()

// … when we want to shutdown
control.shutdown()
```

Ok, so we need a way for the stream to communicate with the outside world, but is it really necessary to introduce this complexity? Couldn't we achieve the same result by simply using closures to provide the stream feedback channel and the control interface?

Let's try to implement this strategy for the feedback channel.

```scala
val promise = Promise[Done]()
val stream: RunnableGraph[NotUsed] = Source.repeat("Hello world")
    .take(3)
    .to { 
        Sink.foreach(println).mapMaterializedValue { done =>
            done.onComplete {
                case Success(_) => promise.success(Done)
                case Failure(ex) => promise.fail(ex)
            }
        }
```

Everything seem to work as expected! We achieved the same outcome without relying on the value generated as a result of running the stream.

However what would happen if we were to wait for the `streamDone` future to complete and then run the stream a second time? 

Well in this case we would have a failure trying to comple the promise: once a promise has been completed it cannot change its value, thus every attempt at doing so by calling `success` or `fail` on it raises an exception. Even worse, if we were to try and use the promise to get another future to wait for the second run termination, we would get back an already completed future with the result of the first run, effectively making it impossible to receive any signal from the second run.

This experiment allows us to conclude that *materialized values* are necessary to enable stream stages to be reused multiple times. By having stages create their communication channel only when the stream is run Akka Streams ensures that different stream instantiation are independent.

## How do they compose? 
Now that we understand why *materialized values* are needed let's try and shed some light on how they work. 

First of all it is important to note that *materialized values* are not some special properties of sinks and sources. Indeed every stage in Akka Streams **needs** to produce a value during the materialization phase (i.e. when the stream is executed). In case a stage doesn't have anything meaningful to to produce it is convention to use the singleton type `NotUsed`.

When we want to connect 2 independently defined stages we incur in a problem: the result of this composition will be itself a stage that need to declare the type it will produce during the materialization. But given that we are just connecting 2 already define stages, which of the 2 *materialized values* should we adopt?

We might think that a sensible solution is to just collect all of them into a list and let the user decide how to handle it. This approach however has a pretty evident disadvantage: given that each stage can materialize a value of any type, the resulting list would need to be a `List[Any]`. We might be tempted to try and exploit tuples to regain our types (by adding a lot of specialized operators for all the possible arities of the stages, or by rely on a library like Shapeless) however we soon realize that this would become unmanageable as the number of stages in our pipeline increases.

So we reach the conclusion that the better strategy is to deal with *materialized values* while combining stages. This indeed is the same conclusion Akka's authors have come up with.

To this end *Akka Streams* offers us 2 operators `viaMat` and `toMat` which require us to provide a combination function used to produce the *materialization value* of the stages composition. Most of the time what we are interested in is just to select one of the two stages' values or possibly to grab them both. This is so common that the library provides an implementation of these strategies:
- `Keep.left`: select the *materialization value* of the left (upstream) stage
- `Keep.right`: select the *materialization value* of the right (downstream) stage
- `Keep.both`: collet both *materialization values* into a tuple

To further improve developer convenience and code readability *Akka Streams* provides a variation of the above operators which automatically apply the `Keep.left` combination function: `via` and `to`.

## A concrete example
In order to fix into our mind all the things we've said so far let's try and play with a simple example.

We want to build a simple stream which given a source of integers, computes their square, increments the result by one and finally prints them to video. The catch is that we want to be able to control when the source should stop emitting new element from outside the stream. To do this we have our source materialize a `ControlInterface`. In order to properly wait for all elements to be processed before terminating the program, we need also need to have our sink materialize a `Future[Done]`.

Now that we understand how *materialized values* composes we can use the operators we discussed in the previous section to get a tuple containing both the `ControlInterface` and the `Future[Done]`

```scala
trait ControlInterface {
  def stop(): Unit
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
  control.stop() 
}

doneF.onComplete { result =>
  result match {
    case Success(_) => println("The stream completed successfully")
    case Failure(ex) => println(s"The stream failed with exception: ${ex}")
  }
  system.terminate()
}
```

The following diagram illustrates how the various combination function are chained in order to obtain our end result.
![Materialized Values composition diagram](/images/post/akka-stream-materialized-values/combination.png)

## How can we return our own values?

At this point we feel comfortable working with *materialized values* and we are able to use the various combinator functions to guide the materialization into producing exactly the data whe are interested in.

However there is still something that bother us: what if we wanted to have a stage produce a *materialzed value* of our choosing? The last example featured a source returning a type we defined: `ControlInterface`. This cannot be something that a built-in stage can have generated.

Indded Akka Streams still has some tricks up its sleeve to work on *materialized values*. Up until this point we've only really handled them via the composition functions we specify when combining 2 stages. As we have seen these functions takes 2 values and return a new value as a result. In all the examples we've seen this result was only a projection, however we could have opted to return an entirely different type. In the last example instead of returning a tuple of the `ControlInterface` and the `Future[Done]` we could have opted to create a case class `MyMaterializedValue` containing them.

This intuition should makes us wonder if something similar is possible also when operating on a single stage. That is indeed the case: we can use the method `mapMaterializeValue` to apply a transformation to the *materialized value* of a source, flow or sink. This method take as an argument a function that given the current value needs to produce a new value.

Let's see how we can use this feature to implement the source from the last example:

```scala
class ControlInterfaceImpl(killSwitch: KillSwitch) extends ControlInterface {
  def stop(): Unit = killSwitch.shutdown()
}

val source: Source[Int, ControlInterface] = Source.fromIterator(() => Iterator.from(1))
  .throttle(1, 500.millis)
  .viaMat(KillSwitches.single)(Keep.right)
  .mapMaterializedValue(s => new ControlInterfaceImpl(s))
```

The idea is to leverage a kill switch stage to interrupt the generation of new integers and wrap the materialized `KillSwitch` instance into an implementation of our `ControlInterface`.

This strategy of handling *materialized values* is a good approach when we want to *repackage* the current value into something else. This help in avoiding leaking too many details of how our stages are implemented leaving room to tweak the internal representation without breaking source compatibility.

An important observation we can make on this scheme is that inside the `mapMaterializedValue` we are free to close over whathever value without any chance for Akka Streams to tell us if we are doing something potentially dangerous. As we already discussed in previous sections, stages once defined can be materialized multiple times. Thus we must be extra careful not to close over values which are not intended to be used multiple times (remember the example on promises).

This strategy covers the majority of situation where we need to operate on *materialized values*, so we could stop here. However in the preface I stated that this article would be an in-depth coverage of the topic, so let's go on. In the remainder of this section we will see how to create a custom stage which materialiazes a value of our choosing.

So let's image that for some reason we find ourself unable or unwilling to use kill switches as a mechanism to implement the `ControlInterface`. We need an alternative way to communicate with our source to signal we want it to stop producing new values.

To achieve this we will use the `GraphStage` API: this is the lowest level building block of Akka Streams on top of which all other components are constructed. Explaining this API alone could be the topic of a full article, so we are not going to dwell on the details of how it works. Instead we will limit ourself to discussing the parts which are functional to working with *materialized values*.

Given that our main objective is to produce a *materialized value* we will use a variant of the API called `GraphStageWithMaterializedValue` which allows to define a factory which creates both the logic and the value of our stage.

The idea behind our implementation will be rather simple: we'll define a stage of shape `Source` which will produce integers starting from a specified numebr and provide a callback function which we will use to complete the stage when invoked.

```scala
class AsyncCallbackControlInterface(callback: AsyncCallback[Unit]) extends ControlInterface {
  def stop(): Unit = {
    callback.invoke( () )
  }
}

class StoppableIntSource(from: Int) 
  extends GraphStageWithMaterializedValue[SourceShape[Int], ControlInterface] {
  val out: Outlet[Int] = Outlet("out")
  
  def shape: SourceShape[Int] = SourceShape(out)
  
  class StoppableIntSourceLogic(_shape: Shape) extends GraphStageLogic(shape) {
      private[StoppableIntSource] val stopCallback: AsyncCallback[Unit] = 
        getAsyncCallback[Unit](
          (_) =>
            completeStage()
        )
      
      private var next: Int = from
      
      setHandler(out, new OutHandler {
        def onPull(): Unit = {
          push(out, next)
          next += 1
        }
      })
  }
  
  def createLogicAndMaterializedValue(inheritedAttributes: Attributes)
  : (GraphStageLogic, ControlInterface) = {
    val logic = new StoppableIntSourceLogic(shape)
    
    val controlInterface = new AsyncCallbackControlInterface(logic.stopCallback)
    logic -> controlInterface
  }
}
```

Most of the code is rather simple if a little verbose. The only important bit is the one regarding the handling of the `stopCallback`. For starters we can see that we defined a dedicated class for the stage logic instead of defining it anonymously as it usually done when working with `GraphStage`. This is so that can have access to the callback from the outside of the class. Indeed looking at the `createLogicAndMaterializedValue` method we can see that first we create the logic and then we extract the callback and wrap it inside our `ControlInterface` implementation. 

The other thing to note is that inside the `AsyncCallbackControlInterface` we are not calling the callback directly but instead we are using the `invoke` method. This will schedule the execution of our callback code asynchronously by interleaving it with the data handlers. This strategy guarantees us that once the code is run no other thread has access to the `GraphStageLogic` instance so we are safe to operate on its mutable state or perform management operations on it.

We can now use our `StoppableIntSource` to implement the source:
```scala
val source: Source[Int, ControlInterface] = Source.fromGraph(new StoppableIntSource(1))
  .throttle(1, 500.millis)
```

At this point we should have a good understanding of *materialized values* and how to handle them. We might think that the functinality we have covered are enough to implement whatever program we might think of. However Akka Streams still has an ace up its sleve which comes into our help when we face particularly tricky situation.

## Let's talk pre-materialization
---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
---

# What Is `akka-stream-bladder`?

This Scala-language project provides backpressure-sensitive Akka Stream
components, making it easier to create efficient Akka Stream graphs.

All Akka Stream components have some compensating behavior under backpressure.
Akka's default strategies involve buffering in a strictly ordered queue, based on
arrival order of messages. Each component has an internal buffer, growable to
some fixed max size, which accumulates messages when the component gets backpressured
from downstream. When the buffer fills to its max size the component then must
either ignore new incoming messages or raising an error. The buffer max size may
be zero.

When a componenet's internal buffer fills to its limit, the component employs one of
a few failsafe actions:

* Transmits the backpressure upstream
* Drops messages in proportion to the volume of incoming messages
* Clear the entire buffer
* Raises an error, which typically halts the entire Akka Stream graph

This library provides a few additional behaviors which attempt to compensate for
backpressure before applying one of these drastic actions. All strategies implemented
in this library involve "buffering", but not necessarily in queues, or even fixed-
sized data structures. They each attempt to relieve backpressure by creating
more efficient workloads out of the set of buffered messages.

## Strategies for Efficiency Under Backpressure

* **Prioritizing** - Order incoming messages according to some metric, with the
  intent of offering messages downstream that are "faster" to process before "slower"
  ones. This should relieve backpressure more quickly under higher backpressure
  scenarios. The `PrioritizingFlow` implements this behavior.
* **Reducing** - Combining multiple idempotent or overlapping messages into a
  single message of the same type. Under higher backpressure scenarios where the
  message stream includes messages that can be *reduced* to fewer, this strategy
  relieves pressure. A simple example is reducing redundant idempotent messages.
  The `ReducingFlow` implements this behavior.
* **Bundling** - A.k.a. "boxcarring", involves binding multiple related messages
  into a containerized message that can be more efficiently processed downstream.
  The `BundlingFlow` implements this behavior.
* **Fold-Unfolding** - Accumulating, or "folding", multiple incoming messages into
  an arbitrary internal data structure under backpressure, and "unfolding" individual
  outbound messages from this internal data structure to meet downstream demand.
  In fact, the other strategies (prioritizing, reducing, bundling, even standard
  buffering) are actually special cases of this one. Pretty much any backpressure
  management strategy you can dream up can be implemented with a `(fold, unfold)`
  function pair with. The `MetamorphicFlow` implements this behavior.

  (C.S.-oriented folks may be familiar with the concept of *metamorphism*, the
  dual of a [*hylomorphism*](https://en.wikipedia.org/wiki/Hylomorphism_(computer_science),
  which lends this component its name.)

# Including Dependency in Your Project

I haven't started publishing to any public JAR repositories yet, so you will
have to build and publish locally to include in your project for now. This project
uses the [`mill`](http://www.lihaoyi.com/mill/) build system. After downloading,
checkout the latest stable release, then build and publish locally using:

```
git clone git@github.com:bmaso/akka-stream-bladder.git
git checkout tag/0.0.1-release
mill akka-stream-bladder.publishLocal
```

The project is cross-compiled for Scala 2.12 and 2.13.

Use the `sbt`/`mill` coordinates `bmaso::akka-stream-bladder:0.0.1` in
your local project to declare a dependency.

# Prioritizing

A `PrioritizingFlow[M]` component implements prioritizing buffer
behavior. Use this kind of flow when there is a significant difference in downstream
processing time between different `M` message instances in order to
prioritize "faster" messages over "slower" ones.

Instantiate an instance with three parameters: internal buffer max size (which
may be infinite), overflow strategy, and an `Ordering[M]` which will be used to
sort buffered messages. Buffered messages are yielded downstream in sorted order
instead of arrival-tie order.

# Reducing

A `ReducingFlow[M]` component combines overlapping or idempotent
messages under backpressure, attempting to reduce the accumulated message
volume. Use this type of flow when idempotent or overlapping messages appear in the
input stream frequently. For example, when implementing a periodic task executor as
with an Akka Stream graph, this type of flow will combine multiple copies of identical
tasks. This eliminates duplicates, reducing the downstream load.

## Use Cases

There are a few different anticipated use-cases where a `ReducingFlow[M]` could
relieve buffered message volume during backpressure situations:

* All messages are reducible with each other. For example, imagine a flow comprised
  of timestamped sensor observations for a single sensor. You could use a
  `ReducingFlow` to ensure that, during backpressured time periods, only the
  latest sensor measurement is retained. Reduced (i.e., combined) observations
  could include the number of elided observations and the total time period
  each combined observation covered.
* Messages are partitioned or "grouped" by a key value. Imagine the same sensor
  observation scenario, but in this case the flow contains sensor observations
  from an array of sensors. Using a sensor ID value as a comparable key, a
  `ReducingFlow` would combine observations with matching sensor keys
  during backpressured time periods.
* A flow of heterogenous messages, some reducible with each other while others
  are not reducible. Continuing with the previous examples, imagine a scenario
  where both sensor measurements and sensor status messages ("online/offline"
  indicators, for example) are passed through the same flow. Measurements could be
  combined during backpressure situations, while status messages would pass
  through unchanged.

A `ReducingFlow` instance can be created with three parameter values:
internal buffer max size, overflow strategy, and a single function that compares
two `M` instances to see if they are reducible together, and reduces them if so.
We term this function a *reduce-or-compare* function. More notes on this
function below.

## Notes on *reduce-or-compare* Functions

Two flow components (`ReducingFlow` and `BundlingFlow`) utilize
*reduce-or-compare* functions to either
* Reduce/combine buffered messages
* Or: If two buffered messages are not mutually reducible, then the function
  returns a relative ordering of the message objects.

I choose define the signature for such a function as `(M, M) => Either[Double, M]`.
That is, when invoked with two `M` objects the function *either*

* Returns a `Right[M]` which is a reduction of the original pair of `M` objects
* Or: if the two are irreducible, returns a `Left[Double]` value indicating the relative
  ordering of the two `M` objects. This value should not be zero when comparing
  two irriducible objects: that will lead to ambiguous output message ordering.

*(If you have a good intuition about abstract algebra concepts, you might
  sense a relationship between* order-or-compare *functions and **ordered
  semigroups**. They aren't quite the same. We can't make a* reduce-or-compare
  *function from a partially-ordered semigroup, nor vice-versa.)*

### How `ReducingFlow` Components Utilitize *reduce-or-compare* Functions

If at all possible we want to avoid testing each new `M` message object received
from upstream against each buffered `M` object for irreducibility during backpressure
scenarios. If two `M` objects are irreducible, we want to assign an ordering to them
that is transitive and consistent. Imposing and ordering and avoiding excessive
comparisons ensure the complexity of receiving and buffering an `M` object during
backpressure scenarios is no worse than **O(log(b))**, where **b** is the average
internal buffer size.

Absent a relative ordering, the computational complexity of receiving a message
from upstream becomes **O(b)**. Such a high complexity level could easily negate
any advantage a `ReducingFlow` might provide in relieving backpressure.

The restriction that the ordering value must be non-zero for two irreducible
objects negates the possibility of a trivial comparison function (that is,
one that always returns zero).

Receiving a single message can kick off multiple rounds of reducing and comparing
messages, as a combination of two messages must then be recursively checked for
reducibility and relative ordering against other messages in the `ReducingFlow`'s'
internal buffer. Care must be taken to ensure the comparison function is transitive
and consistent. A pathologic comparison function can lead to infinite recursion.

### Two Alternative Orderings for Output Messages

There are two different logically valid orderings for a `ReducingFlow[M]`
to yield messages to downstream when its internal buffer is non-empty. That is,
if a flow internally has 2 `M` messages buffered (implying they are irreducible
with each other), and the downstream component signals demand, there are two
strategies the component might employ to decide which message to send downstream
first:

* FIFO ordering: the message originally received earlier will be yielded first.
  A message formed from reducing two messages inherits the earliest timestamp of
  the two original messages.
* Use *reduce-or-compare* function ordering (the default).

An optional parameter at creation time indicates which strategy the component
should employ.

# Bundling

In some cases it is more efficient to process a collection of
`M` objects all at once than it is to process the instances separately. An example
from the real world: it is more efficient for my postal carrier to bundle all of my
mail, delivering the bundle of mail once each day, than it would be for him to
deliver each piece of mail individually. (Actually, it is *no slower* for him to
do so, if we are being pedantically logical.)

The `BundlingFlow[M, F[_]]` receives individual `M` message objects from
upstream and yields `F[M]` container-type instances to downstream. `F[_]` can be
any type where the following two functions are provided:

* A binding function `(M) => F[M]` which turns an instance of `M` into an `F[M]`
* And: a *reduce-or-compare* function with parameter type `F[M]`; the
  signature of such a method is `(F[M], F[M]) => Either[Double, F[M]]`

The *reduce-or-compare* function serves in a similar capacity to the *reduce-or-compare*
function of a `ReducingFlow`. This function supports efficient internal
storage and indexing of `F[M]` objects within the `BundlingFlow`. Using
this function, receiving and internally storing an `M` message object from upstream
has a computational complexity of **O(log(b))**, where **b** is the average
number of `F[M]` instances stored internally.

## Use Cases



## Notes on *reduce-or-compare* functions in `BundlingFlow`

A `BundlingFlow` utilizes a *reduce-or-compare* function almost
[the same way]((#notes-on-reduce-or-compare-functions)) that a `ReducingFlow` does.
When an `M` message is received from upstream, the message is:

* First converted to an `F[M]` using the `(M) => F[M]` function provided at creation.
* Second the `F[M]` is (potentially) combined with internally buffered `F[M]` instances
  using the *reduce-or-compare* function (also provided at creation).

### Two Alternative Orderings for Output Messages

The order that `F[M]` bundle objects are yielded from a `BindlingFlow` is
logically ambiguous. Either one of two behaviors are both logically valid and
consistent with Akka's definition of component and message flows:

* Yielded bundle object ordering can be based on the order of receiving the `M`
  messages accumulated in each `F[M]` bundling object -- the bundle that "contains"
  the oldest `M` object gets yielded first.
* Bundle objects can be yielded in the order implied by the ordering of the
  *reduce-and-compare* function. That is, let `c` be such a comparison function,
  then if `val Left(n) = c(f1, f2)`, and *n < 0.0*, then `f1` would always
  be yielded before `f2` for any `BundlingFlow` that uses `c`. This is the default
  behavior for a `BundlingFlow`

# "Fold-Unfolding"

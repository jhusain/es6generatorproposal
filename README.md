ES6 Generator Proposal
==============================

Today the generator proposal in ES6 is flawed in several important ways. The result is that generators, as specified today, can not be effectively used for stream processing.

We aim to convince you of the following:

* Stream processing is an important use case for generators to support.
* Stream processing via generators is broken in ES6.
* __The problem can be easily fixed__ with very few changes to the current specification.
* These fixes will create a platform on which we can build support for async stream processing in ES7.

If we agree that ES6 must be an effective stream programming language, we must first define success. The reason we are where we are today, is that success has been incorrectly defined as...

_An Iterator API that is easy to use._

__We contend this is entirely the wrong goal.__ In this proposal we argue for a much more ambitious definiton of success:

_For the vast majority of use cases, developers should not need to use an Iterator._ In fact, if ES6 is designed correctly, most developers will never know the Iterator type exists.

This goal, ambitious though it may seem, has already been achieved by a popular, widely-used programming language: C#. In fact, this proposal contains absolutely _no new ideas_. It can be summed up in a sentence: __when it comes to stream processing, ES6 should borrow from C#, not Python.__





1. Comprehensions are not future proof. They will not be able to be used to compose either the parallel array or the asynchronous generator objects slated for introduction in ES7. http://smallcultfollowing.com/babysteps/blog/2014/04/24/parallel-pipelines-for-js/
2. The generator comprehension syntax provides functionality not otherwise available through the standard library.
3. The Iterable contract does not dependably create a new iterator object for each iteration. Due to this inconsistency, it is impossible to implement common stream operators over this contract (ex. retry).
4. There is no way to short-circuit a generator. As a result, paging streams of data is prohibitively expensive.

These issues can be resolved if we make the following changes to the ES6 specification. 

Move the comprehension syntax to ES7
------------------------------------------------------

The comprehension syntax's value proposition is that it *enables complex composition without the need to create deeply nested closures.* Many developers get confused when faced with code like this: 

```JavaScript
var sums = [1,2,3].concatMap(x => [4,5,6].map(y => x + y))
```

Comprehensions help by allowing them to replace the code above with this expression...


```JavaScript
var sums = (
  for(let x of [1,2,3])
    for(let y of [4,5,6])
      x + y);
```

The comprehension syntax removes the need to use closures in many cases. So why delay comprehensions until ES7?

__Comprehensions are not future proof__, because they will not work on the forthcoming ES7 parallel array, nor the yet-to-be proposed asynchronous generator. The comprehension syntax should be able to be used to compose any collection type, so we should introduce a more generic syntax (1-2-n rule).

In ES7 we should add a monadic comprehension syntax capable of composing arrays, variables, parallel arrays, and asynchronous generators. Furthermore it will be possible to add comprehension support for future types without new syntax. The monadic comprehension syntax will desugar into method calls, allowing the type itself to define how it is composed. For more information see the ES7 monadic comprehension syntax proposal below.

Although some developers prefer using the comprehension syntax, __comprehensions will never support all operations__.  If developers want to use operations like some(), any(), and contains() they will be forced to use method chaining. In these circumstances, many developers will eschew the comprehension syntax in order to avoid using using multiple composition styles in a single expression. In other words, many developers will prefer this…

```Javascript
var numberExists = list.concatMap(y => otherList.map(x => x + y)).some(x => 42);
```

…to this:

```Javascript
var numberExists = (
  for(let y of list)
    for(let x of otherList)
      x + y).
    some(x => 42);
```

__It is important that we enable the same types of composition operations regardless of whether a developer is using the comprehension syntax or method chaining.__ To ensure this, we must add additional combinators to the prototypes of the various collections.
 
Add concatMap method to the Array prototype
------------------

To facilitate array composition with method chaining we will add concatMap to the Array prototype:
  
```JavaScript
Array.prototype.concatMap = function(projection) {
  let results = [];
  for(let counter = 0; counter < this.length; counter++) {
    results.push.apply(results, projection(this[counter]));
  }
    
  return results;
}
```

The name concatMap (used in Haskell) has been deliberately selected in lieu of flatMap, in order to be explicit about the type of flattening strategy applied after the map operation.

Why specify concatenation explicitly? While concatenation is the only sensible way of flattening a two-dimensional array or generator, the same same is not true for asynchronous generators. Asynchronous generators send information over time, allowing for other flattening strategies to be applied (ex. merge, switch latest).

By being explicit about the kind of flattening strategy being applied, we remove a potential __refactoring hazard__. Let's say a developer decides that they want to change a generator expression to an asynchronous generator expression. By forcing the developer to be explicit about the flattening strategy, the asynchronous generator expression will have the same ordering of elements as the synchronous generator expression by default.

Add a return method to Generators
----------------------------------------------

Generators allow for the lazy evaluation of algorithms. Lazy evaluation is particularly useful in the area of stream processing, allowing chunks of data to be transformed and sent elsewhere as they arrive. The alternative, loading an entire stream of data into memory before processing, is impractical for large data sets.

Today it is possible for generators to abstract over scarce resources like IO streams. However the consumer must iterate the generator to completion to give the generator function the opportunity to free the scarce resources in its finally block.

(Example)

This unnecessary constraint makes common operations like paging impractical because of the overhead involved. Why should a consumer need to continue reading from a stream long after the desired data has been acquired?

This problem can be resolved by adding a return semantic to the generator. Today generator functions give consumers the ability to insert a throw statement at the current yield point by invoking the throw method on the generator. A return method should be added to generator which has similar semantics to throw. Invoking return() should cause the generator function to behave as a though a return statement was added at the current yield point within the generator. This will ensure that finally blocks get run, giving the asynchronous generator the opportunity to free scarce resources. To guarantee the termination of the generator function on return(), yield statements will be prohibited within finally blocks.

The addition of the return method to the iterator will allow commonly used methods like takeWhile to be written.

(Example)

Methods like takeWhile allow developers to conditionally consume sequences without having to build state machines.

However as you can see from the example above, takeUntil cannot be chained with other combinators left-to-right because iterators do not have a shared prototype. Instead it is necessary to compose functions right to left. Thanks to frameworks like jQuery, JavaScript developers have grown accustomed to building expressions via method chaining. Dave Herman recently minted a name for this pattern: _the pipeline adapter pattern_.

How can we enable developers to build complex stream processors using the pipeline pattern?


The Iterable Constructor
--------------------------------

The Iterable contract should be replaced by an explicit constructor. The Iterable constructor will be used to create all instances of Iterables. An Iterable has an iterate() method which can return a Generator or an Iterator. 

```JavaScript
function Iterable(iterateMethodDefinition) {
  this.iterate = iterateMethodDefinition;
}
```

The introduction of the Iterable constructor provides a prototype on which to add pipeline methods. The Iterable prototype should have lazy versions of the following methods:

* map
* concatMap
* filter
* reduce
* concatMap
* some
* any
* find
* forEach
* indexOf
* join
* reduceRight

The Iterable prototype will also include a toArray() which will convert the Iterable to an Array. 

In addition to these prototype methods, the Iterable constructor will also have the following methods found on the Array constructor:

* of
* from

The introduction of the Iterable constructor allows stream processing expressions to be built using the pipeline adapter pattern. 

The style of composition is expressive enough to replace comprehensions until they are introduced in ES7.

The Iterable constructor's iterate() method is expected to return a newly constructed iterator every time it is invoked.

Generator functions should return Iterables, not Iterators
--------------------------

Today ES6 is optimized to make the Iterator easy to use.  __This is not the right goal, .__ Iterators are complex objects. They are stateful and consuming them . In our zeal to reduce the complexity of the Iterator API, we have oversimplified the type and defeated extremely important use cases.

Furthermore after adding the necessary return method to generators, they will have a lifecycle as well.  


This change is motivated by a simple design goal:

__It should be possible to consume the data produced by a generator function without needing to directly use an Iterator..__

Iterators have complex interfaces, and require state machines to consume. With the introduction of the return method, they now have a lifecycle as well.



Iterators have Iterators are mutable data structures and they are complex to use. We should embrace the following design principle:

* developer should be able to 

Iterable Adapteee
 
 any object that  can create an adapter



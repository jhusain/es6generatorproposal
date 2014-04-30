ES6 Generator Proposal
==============================

The current generator proposal in ES6 is deficient in several important ways.

1. Comprehensions are not future proof. They will not be able to be used to compose either the forthcoming parallel array or the asynchronous generator objects that are slated for introduction in ES7. http://smallcultfollowing.com/babysteps/blog/2014/04/24/parallel-pipelines-for-js/
2. The generator comprehension syntax provides functionality not otherwise available through method chaining.
3. The Iterable contract provides no guarantee that the iterator object has been freshly created for the consumer. Without a guarantee of freshness, it is impossible to implement common stream operators over this contract (ex. retry).
4. Generators lack a return() semantic, which forces consumers to fully complete iteration in order to free any scarce resources (ex. IO streams). This makes common stream operations like paging prohibitively expensive.
 
These issues can be resolved if we make the following changes to the ES6 specification. 

Move the comprehension syntax to ES7
------------------------------------------------------

The comprehension syntax's value proposition is that it *enables complex composition without the need to create deeply nested closures.* Many developers get confused when faced with a deeply nested closure. Comprehensions can help by allowing them to replace the following expression...

```JavaScript
var sums = [1,2,3].concatMap(x => [4,5,6].map(y => x + y))
```

…with this:

```JavaScript
var sums = (
  for(let x of [1,2,3])
    for(let y of [4,5,6])
      x + y);
```

The combination of a comprehension syntax and the forthcoming async/await syntax in ES7 will remove the need to use closures entirely in many circumstances. So why delay comprehensions until ES7?

__Comprehensions are not future proof__, because they will not work on the forthcoming ES7 parallel array, nor the yet-to-be proposed asynchronous generator. The comprehension syntax should be able to be used to compose any collection type, so we should introduce a more generic syntax (1-2-n rule).

In ES7 we will add a monadic comprehension syntax capable of composing arrays, variables, parallel arrays, asynchronous generators, and any type introduced in the future. The monadic comprehension syntax will desugar into method calls, allowing the type itself to define how it is composed. For more information, see the ES7 monadic comprehension syntax proposal below.

Although some developers prefer using the comprehension syntax, it will never support all operations.  If developers want to use operations like some(), any(), and contains() they will be forced to use method chaining. In these circumstances, many developers will eschew the comprehension syntax in order to avoid using using multiple composition styles in a single expression. In other words, many developers will prefer this…

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

__It is important that we enable the same types of composition regardless of whether a developer is using the comprehension syntax or method chaining.__ To ensure this, we must add additional combinators to the prototypes of the various collections.
 
Add concatMap method to the Array prototype
------------------

We will add the following method to the Array prototype:
  
```JavaScript
Array.prototype.concatMap = function(projection) {
  let results = [];
  for(let counter = 0; counter < this.length; counter++) {
    results.push.apply(results, projection(this[counter]));
  }
    
  return results;
}
```

In the absence of comprehensions in ES6, this method will allow developers to compose array expressions using function chaining. The name concatMap (used in Haskell) has been deliberately selected in lieu of flatMap, in order to be explicit about the type of flattening strategy applied after the map operation.

Why specify concatenation explicitly? While concatenation is the only sensible way of flattening a two-dimensional array or generator, the same same is not true for asynchronous generators. Asynchronous generators send information over time which allows for the other flattening strategies to be applied. 

Concatenation flattens a two-dimensional collection by ordering elements based on the order in which each inner collection arrives, as well as the order of the elements within each inner collection. Another flattening strategy is a __merge__, which when applied to a two-dimensional asynchronous generator orders items based on the order they arrive in time, irrespective of the order in which their inner collections arrives. Merge acts like a funnel, forwarding items as soon as they arrive in time, and is appropriate when throughput is more important than preserving order.

By being explicit about the kind of flattening strategy being applied, we remove a potential __refactoring hazard__. Let's say a developer decides that they want to change a generator expression to an asynchronous generator expression. By forcing the developer to be explicit about the flattening strategy, the asynchronous generator expression will have the same ordering of elements as the synchronous generator expression by default.

Add a return method to Generators
----------------------------------------------

Generator allow for the lazy evaluation of algorithms. Lazy evaluation is particularly useful in the area of stream processing, allowing chunks of data to be transformed and sent elsewhere as they arrive. The alternative, loading an entire stream of data into memory before processing, is impractical for large data sets.

Today it is possible for generators to abstract over scarce resources like IO streams. However the consumer must iterate the generator to completion to give the generator function the opportunity to free the scarce resources and its finally block.

(Example)

This imposed constraint makes common operations like paging impractical because of the overhead involved. Why should a consumer need to continue reading from a stream long after the desired data has been acquired?

This problem can be resolved by adding a return semantic to the generator. Today generator functions give consumers the ability to insert a throw statement at the current yield point by invoking the throw method on the generator. A return method should be added to generator which has similar semantics. Invoking return() should cause the generator function to behave as a though a return statement was added at the current yield point within the generator. This will ensure that finally blocks get run, giving the asynchronous generator the opportunity to free scarce resources. To guarantee the termination of the generator function on return(), yield statements will be prohibited within finally blocks.

The addition of the return method to the iterator will allow useful methods like takeWhile to be written.

(Example)

Methods like takeWhile allow developers to conditionally consume sequences without having to build state machines.

However as you can see from the example above, takeUntil cannot be chained with other combinators left-to-right because iterators do not have a shared prototype. Thanks to frameworks like jQuery, JavaScript developers have grown accustomed to building expressions via method chaining. Dave Herman recently coined a name for this pattern: _the pipeline adapter pattern_.

How can we enable developers to build complex stream processors using the pipeline pattern?


The Iterable Constructor
--------------------------------

The Iterable contract should be replaced by an explicit constructor. The Iterable constructor will be used to create all instances of Iterables. An Iterable has an iterate() method which can return a Generator or an Iterator. 

```JavaScript
function Iterable(iterateMethodDefinition) {
  this.iterate = iterateMethodDefinition;
}
```
The Iterable prototype will have lazy versions of the following methods found on the Array prototype:

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

This change is motivated by a simple design goal:

__It should be possible to consume the data produced by a generator function without needing to directly use an Iterator.__

Iterators are mutable data structures with a lifecycle, and they are extraordinarily difficult to 



Iterators have Iterators are mutable data structures and they are complex to use. We should embrace the following design principle:

* developer should be able to 

Iterable Adapteee
 
 any object that  can create an adapter



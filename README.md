ES6 Generator Proposal
==============================

The current generator proposal in ES6 is deficient in several an important ways.

1. __Comprehensions are not future proof.__ For example, they can not be used to compose the forthcoming parallel array class in ES7. http://smallcultfollowing.com/babysteps/blog/2014/04/24/parallel-pipelines-for-js/
2. Iterables cannot be composed without the use of the generator comprehension syntax.
3. Iterator currently implements the Iterable contract by returning itself. This undermines the usefulness of the Iterable contract by making it impossible to know whether an iterator has been created for a specific consumer, or is being used by multiple consumers. This ambiguity prevents us from writing common combinators (ex. retry) over this contract.
4. Generators functions return iterators, which do not have a shared prototype. This makes it difficult to express generator composition without using list comprehensions.
5. Generator functions cannot be returned by the data consumer. This means that generators that abstract over scarce resources (ex. IO streams) cannot free these resources without being iterated to completion. 
 
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

*The current comprehension syntax works only on arrays and generators.* This syntax is not future proof, because it will not work on the forthcoming ES7 parallel array or the asynchronous generator. It is clear that we would like comprehensions to work on these additional types, so we should introduce a more generic syntax (1-2-n rule).

In ES7 we will add a monadic comprehension syntax capable of composing arrays, variables, parallel arrays, asynchronous generators, and any type introduced in the future. The monadic comprehension syntax will desugar into method calls, allowing the type itself to define how it is composed. For more information, see the ES7 monadic comprehension syntax proposal below.

Although some developers prefer using the comprehension syntax, it will never support all operations.  If developers want to use operations like some(), any(), and contains() they will be forced to use method chaining. In these circumstances, many developers will eschew the comprehension syntax in order to avoid using different composition styles in a single expression. In other words, many developers will prefer this…

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

The Iterable Constructor
--------------------------------

The Iterable contract will be replaced by an explicit constructor. The Iterable constructor will be used to create all instances of Iterables. An Iterable has an iterator method which can return a Generator or an Iterator. 

```JavaScript
function Iterable(iterateMethodDefinition) {
  this.iterator = iterateMethodDefinition;
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

Add a return method to Generators
----------------------------------------------

Iterators ￼:

1. Enable developers to write asynchronous code that appears to be synchronous. (Task.js)
2. Allow developers to write lazy code that appears to be eager.

One of the more common reasons for lazily processing sequences of data, is to avoid having any large amount of data in memory at any one time.


Generators and Iterators can optionally provide a return method which should be invoked by the data consumer in the event that the function is exited before being run to completion. Semantically calling the return function is equivalent to inserting a return statement at the current position within the generator function.

```JavaScript
var iter = functionThatReturnsGenerator();
try {
   
}
```



Generator functions should return Iterables, not Iterators
--------------------------

This change is motivated by a simple design goal:

__It should be possible to consume the data produced by a generator function without needing to directly use an Iterator.__

Iterators are mutable data structures with a lifecycle, and they are extraordinarily difficult to 



Iterators have Iterators are mutable data structures and they are complex to use. We should embrace the following design principle:

* developer should be able to 

Iterable Adapteee
 
 any object that  can create an adapter



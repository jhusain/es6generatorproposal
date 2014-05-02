ES6 Generator Proposal
==============================

Today the generator proposal in ES6 is flawed in several important ways. The result is that generators, as specified today, can not be used for stream processing. 

I contend that...

* Stream processing is a vitally important use case for generators to support.
* Stream processing can easily be supported with very few changes to the current specification.
* If problems with the specification are not fixed now, they can _not_ be resolved retroactively in ES7.
* Resolving these issues is necessary to support asynchronous generators in ES7.

There are several problems with the existing specification. However before we get into specifics, it is important to understand how we got here. __The reason that generators do not currently support stream processing is that we have been focused on the wrong goal.__ We have been trying to make the generator API easy to use, but in doing so we have oversimplified the type and left out important semantics.

__in the vast majority of use cases, developers do not need to interact directly with an iterator.__ As libraries like Underscore demonstrate, most generator use cases can be accommodated with a small set of well-designed combinator functions. Our goal should be to __add a small but comprehensive set of combinator functions to the standard library, and make it easy to chain them together to create generator expressions.__ If we do this, the (irreducible) complexity of the iterator type will only be exposed in rare set of advanced scenarios.

__In short, don't try and make iterators easy to use, make them invisible.__

In order to be effective as a stream programming language, ES6/7 should have the following features:

* A comprehension syntax that can be used to compose any type, synchronous or asynchronous.
* Generators that can cleanly abstract over scarce resources, allowing comprehensions to be used to build stream processors.
* A small but comprehensive library of stream operators so powerful, most developers never need to use an iterator directly.

We can have all of these features (and more) if we make the following changes to the specification.

Move comprehensions to ES7 and make them monadic
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

Today it is possible for generators to abstract over scarce resources like IO streams.

```JavaScript
function getLines*(fileName) {
  let reader = new SyncReader(fileName);
  try {
    while(!reader.eof) {
      yield reader.readLine();
    }
  }
  finally {
    reader.close();     
  } 
};

However the consumer must iterate the generator to completion to give the generator function the opportunity to free the scarce resources in its finally block. In the example below, breaking out of the iteration early leaks the file handle.

```JavaScript
var prices = [];

var iterator = getLines("./prices.txt"),
  pair;

while(!(pair = iterator.next()).done && prices.length < 20) {
  prices.push(pair.value);
}
```

To ensure scarce resources are properly freed it is necessary to always iterate to completion, even if the desired subset of data has already been acquired. This makes common stream operations like paging impractical because of the overhead involved. 

This problem is easily resolved by adding a return semantic to the generator. Today generator functions give consumers the ability to insert a throw statement at the current yield point by invoking the throw method on the generator. A return method should be added to generator which has similar semantics to throw. Invoking return() should cause the generator function to behave as a though a return statement was added at the current yield point within the generator. This will ensure that finally blocks get run, giving the asynchronous generator the opportunity to free scarce resources. To guarantee the termination of the generator function on return(), yield statements will be prohibited within finally blocks.

The addition of the return method will allow generators to cleanly abstract over scarce resources like IO streams.

```JavaScript
var prices = [];

var iterator = getLines("./prices.txt"),
  pair;

while(!(pair = iterator.next()).done && prices.length < 20) {
  prices.push(pair.value)
}

if (!pair.done) {
  pair.return();
}
```

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

Consume and Emit Iterables, not Iterators
--------------------------

Today the ES6 Iterator API is optimized for ease of use. This goal is not easy to achieve. Iterators are complex types. They are mutable objects, and consuming their data requires building a state machine. Furthermore Iterators must have a lifecycle, and be properly disposed to avoid leaking scarce resources.

In our zeal to make iterator API usable, we oversimplified the type and broke an entirely 

This change is motivated by a simple design goal:

__It should be possible to consume the data produced by a generator function without needing to directly use an Iterator..__

Iterators have complex interfaces, and require state machines to consume. With the introduction of the return method, they now have a lifecycle as well.



Iterators have Iterators are mutable data structures and they are complex to use. We should embrace the following design principle:

* developer should be able to 

Iterable Adapteee
 
 any object that  can create an adapter



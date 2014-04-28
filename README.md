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

In the absence of comprehensions in ES6, this method will allow developers to compose array expressions using function chaining. The name concatMap (used in Haskell) has been deliberately selected in lieu of flatMap. Why specify concatenation explicitly? After all, concatenation is the only reasonable way of flattening a two-dimensional array or generator. The same is not true of asynchronous generators. Asynchronous generators send information over time, allowing for multiple ways of flattening a two-dimensional asynchronous generator. By being specific about the kind of flattening strategy, we make refactoring a synchronous generator expression to an asynchronous generator expression easier. By being explicit about the flattening strategy, the asynchronous generator expression will have the same ordering of elements as the synchronous generator expression by default.

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

Generator functions should return Iterables, not Iterators
---------------------------------------------------------------------------
Iterators have Iterators are mutable data structures and they are complex to use. We should embrace the following design principle:

* developer should be able to 

Iterable Adapteee
 
 any object that  can create an adapter

Add a return method to Generators
----------------------------------------------

Generators can optionally provide a return method that can be invoked by the data consumer to exit the function early. Semantically, this is like inserting a return statement at the current position within the generator function.

```JavaScript
var iter

```

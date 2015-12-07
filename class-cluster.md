## Introduction

Throughout its frameworks, Apple makes use of the “class cluster” pattern as a means to separate the public API out from the (potentially complex) internal representations of the data. Clients of the API simply use the public API, while under the hood a different implementation is chosen to most efficiently represent the provided initialization parameter values.

Unfortunately, because initializers in Swift are not methods like in Objective-C, there is no way to specify what the actual return value should be (short of returning nil for failable initializers). This makes it *impossible* to actually implement the class cluster pattern in Swift.

## Motivation

While developing my own Swift framework, I found myself wanting to provide a similar functionality. For the client of said framework, I wanted them to be able to create an instance of an essentially abstract class, and be returned a private subclass instance suited best for handling whatever input initialization parameters they provided. It didn’t make sense given the circumstances to ask the user to decide which class would be the best representation of the data; it should “just work”.

Additionally, the class cluster pattern can make backwards compatibility significantly easier; instead of littering your code with branches for different versions of an OS, you could instead have one if/switch statement to determine the appropriate subclass for the current OS you’re running on. This allows the developer to trivially keep legacy code for older platforms while taking advantage of new APIs/designs, and also without changing *any* client code. An example of the class cluster pattern being used for this reason can be seen here: http://www.xs-labs.com/en/blog/2013/06/18/ios7-new-ui-strategies/

## Proposed solution

I propose that we allow for implementation of the class cluster pattern by providing a way to (at run time) specify the actual type that should be initialized depending on the provided initialization parameters.

## Detailed design

**Introduce a new class method that can return an appropriate type that should be used for initialization, depending on the provided initialization parameters.**

This is what I believe to be the most clean solution, and with (assumedly) minimal impact on the existing nature of Swift’s initialization process. To ensure this remains safe, the only types allowed to be returned should be subclasses of the parent class (such as returning a __NSArrayI for NSArray). Notably, beyond this method, everything else remains the same; all this does is change what class the initializer is called on initially.

Here is an ideal implementation gist:
https://gist.github.com/rileytestut/0e6e80d3f22b845502e7

## Impact on existing code

There will be zero impact on existing code; if the proposed class method is not implemented, then it will default to simply initializing the “base” class, as it always has.

## Alternatives considered

**Allow for return values in initializers**

This is essentially how most class cluster patterns are implemented in Objective-C. Inside the init method, the class inspects the provided parameters, then assigns self to an instance of the appropriate subclass. Unfortunately, this is wasteful; memory is allocated for the base class, and then subsequently replaced with new memory allocated for the appropriate base class. More importantly though, the whole process can be complicated; it can be very easy to make an infinite recursive loop by calling [super init] in the subclass, which then assigns self to a new instance of the subclass, which then calls [super init]…etc. 

tl;dr; this method would work, but would be somewhat inconvenient to implement.

**Class function to return appropriate instance**

This is probably the simplest approach: simply make a class function that returns an instance of the appropriate class given a few input parameters. This totally works, but it means consumers of the API have to remember to use the class method instead of the initializer. Even if all initializers for the class were marked private, it would be strange to have the dissonance between using initializers and class methods to instantiate types in code. The consumer should not have to know about *any* of the implementation details; everything should “just work”. Forcing them to use alternative means to instantiate objects breaks this philosophy, IMO.

**Derive from Objective-C base class**

Another option is to simply derive from an Objective-C base class, and this is actually what I am doing right now in my framework. Unfortunately, there is one significant drawback: because the initialization is happening in Objective-C, you can only provide Objective-C compatible types for the initialization parameters (so no Swift structs for you!). Additionally, this (obviously) means whatever code is using it is limited to systems with Objective-C support, so it is not as portable as a pure-Swift solution.

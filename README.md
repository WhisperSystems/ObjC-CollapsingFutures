Collapsing Futures
==================

This is a library implementing [futures](https://en.wikipedia.org/wiki/Future_%28programming%29) in Objective-C, featuring:

- **Types**: `TOCFuture` to represent eventual results, `TOCCancelToken` to propagate cancellation notifications, `TOCFutureSource` to produce and control an eventual result, and `TOCCancelTokenSource` to produce and control a cancel token.
- **Automatic Collapsing**: You never have to worry about forgetting to unwrap or flatten a doubly-eventual future. A `[TOCFuture futureWithResult:[TOCFuture futureWithResult:@1]]` is automatically a `[TOCFuture futureWithResult:@1]`.
- **Sticky Main Thread**: Callbacks registered from the main thread will get back on the main thread before executing.  Makes UI work much easier.
- **Cancellable Operations**: All asynchronous operations have variants that can be cancelled by cancelling the `TOCCancelToken` passed to the operation's `until:` or `unless:` parameter.
- **Immortality Detection**: It is impossible to create new space leaks by consuming futures and tokens (but producers still have to be careful). If a reference cycle doesn't involve a token or future's source, it will be broken when the source is deallocated.
- **Documentation**: Useful doc comments on every method and type, that don't just repeat the name, covering corner cases and in some cases basic usage hints. No 'getting started' guides yet, though.

**Recent Changes**

- Bit the breaking change bullet and changed some names so they're not stuck once v1 hits.
- Fixed lack of "toc" prefixes on array category methods like `asyncThenAll` (now `toc_thenAll`).
- Shortened excessively long names, such as `futureWithResultFromAsyncOperationWithResultLastingUntilCancelled` becoming `futureFromUntilOperation`.
- Added `matchLastToCancelBetween:and:` and `matchFirstToCancelBetween:and:` constructors to `TOCCancelToken`, to allow combining lifetimes a bit more flexibly.
- Added check of cancel token *after* returning to main thread, to allow UI code to assume that observing a token as cancelled implies no more returning-to-main-thread on-cancel callbacks will occur.

**Incoming Changes**
- Version 1
- Shortening #import "TwistedOakCollapsingFutures.h" to #import "CollapsingFutures.h"

Installation
============

**Method #1: [CocoaPods](http://cocoapods.org/)**

1. In your [Podfile](http://docs.cocoapods.org/podfile.html), add `pod 'TwistedOakCollapsingFutures'`
2. Consider [versioning](http://docs.cocoapods.org/guides/dependency_versioning.html), like: `pod 'TwistedOakCollapsingFutures', '~> 0.7'`
3. Run `pod install`
4. `#import "TwistedOakCollapsingFutures.h"` wherever you want to access the library's types or methods

**Method #2: Manual**

1. Download one of the [releases](https://github.com/Strilanc/ObjC-CollapsingFutures/releases), or clone the repo
2. Copy the source files from the src/ folder into your project
3. Have ARC enabled
4. `#import "TwistedOakCollapsingFutures.h"` wherever you want to access the library's types or methods


Usage
=====

**External Content**

- [Usage and benefits of collapsing futures](http://twistedoakstudios.com/blog/Post7149_collapsing-futures-in-objective-c)
- [Usage and benefits of cancellation tokens](http://twistedoakstudios.com/blog/Post7391_cancellation-tokens-and-collapsing-futures-for-objective-c)
- [How immortality detection works](http://twistedoakstudios.com/blog/Post7525_using-immortality-to-kill-accidental-callback-cycles)
- [An excellent explanation and motivation for the 'monadic' design of futures (C++)](http://bartoszmilewski.com/2014/02/26/c17-i-see-a-monad-in-your-future/)

**Using a Future**

The following code is an example of how to make a `TOCFuture` *do* something. Use `thenDo` to make things happen when the future succeeds, and `catchDo` to make things happen when it fails (there's also `finallyDo` for cleanup):

```objective-c
#import "TwistedOakCollapsingFutures.h"

// ask for the address book, which is asynchronous because IOS may ask the user to allow it
TOCFuture *futureAddressBook = SomeUtilityClass.asyncGetAddressBook;

// if the user refuses access to the address book (or something else goes wrong), log the problem
[futureAddressBook catchDo:^(id error) {
    NSLog("There was an error (%@) getting the address book.", error);
}];

// if the user allowed access, use the address book
[futureAddressBook thenDo:^(id arcAddressBook) {
    ABAddressBookRef addressBook = (__bridge ABAddressBookRef)arcAddressBook;
    
    ... do stuff with addressBook ...
}];
```

**Creating a Future**

How does the `asyncGetAddressBook` method from the above example control the future it returns?

In the simple case, where the result is already known, you use `TOCFuture futureWithResult:` or `TOCFuture futureWithFailure`.

When the result is not known right away, the class `TOCFutureSource` is used. It has a `future` property that completes after the source's `trySetResult` or `trySetFailure` methods are called.

Here's how `asyncGetAddressBook` is implemented:

```objective-c
#import "TwistedOakCollapsingFutures.h"

+(TOCFuture *) asyncGetAddressBook {
    CFErrorRef creationError = nil;
    ABAddressBookRef addressBookRef = ABAddressBookCreateWithOptions(NULL, &creationError);
    
    // did we fail right away? Then return an already-failed future
    if (creationError != nil) {
        return [TOCFuture futureWithFailure:(__bridge_transfer id)creationError];
    }
    
    // we need to make an asynchronous call, so we'll use a future source
    // that way we can return its future right away and fill it in later
    TOCFutureSource *resultSource = [FutureSource new];
        
    id arcAddressBook = (__bridge_transfer id)addressBookRef; // retain the address book in ARC land
    ABAddressBookRequestAccessWithCompletion(addressBookRef, ^(bool granted, CFErrorRef requestAccessError) {
        // time to fill in the future we returned
        if (granted) {
            [resultSource trySetResult:arcAddressBook];
        } else {
            [resultSource trySetFailure:(__bridge id)requestAccessError];
        }
    });
            
    return resultSource.future;
}
```

**Chaining Futures**

Just creating and using futures is useful, but not what makes them powerful. The true power is in transformative methods like  `then:` and `toc_thenAll` that both consume and produce futures. They make wiring up complicated asynchronous sequences look easy:

```objective-c
#import "TwistedOakCollapsingFutures.h"

+(TOCFuture *) sumOfFutures:(NSArray*)arrayOfFuturesOfNumbers {
    // we want all of the values to be ready before we bother summing
    TOCFuture* futureOfArrayOfNumbers = arrayOfFuturesOfNumbers.toc_thenAll;
    
    // once the array of values is ready, add up its entries to get the sum
    TOCFuture* futureSum = [futureOfArrayOfNumbers then:^(NSArray* numbers) {
        double total = 0;
        for (NSNumber* number in numbers) {
            total += number.doubleValue;
        }
        return @(total);
    }];
    
    // this future will eventually contain the sum of the eventual numbers in the input array
    // if any of the evetnual numbers fails, this future will end up failing as well
    return futureSum;
}
```

The ability to setup transformations to occur once futures are ready allows you to write truly asynchronous code, that doesn't block precious threads, with very little boilerplate and intuitive propagation of failures.

Development
===========

**Building the Project:**

1. **Get Source Code**: [Clone this git repository to your machine](http://rogerdudler.github.io/git-guide/).

2. **Get Dependencies**: [Have cocoa pods installed](http://guides.cocoapods.org/using/getting-started.html). Run "pod install" from the project directory.
	
3. **Open Workspace**: Open 'CollapsingFutures.xworkspace' with XCode (note the project, the workspace). Run tests and confirm that they pass.

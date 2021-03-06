# ReactiveCocoa
ReactiveCocoa (RAC) is an Objective-C framework for [Functional Reactive
Programming][]. It provides APIs for **composing and transforming streams of
values**.

If you're already familiar with functional reactive programming or know the basic
premise of ReactiveCocoa, check out the [Documentation][] folder for a framework
overview and more in-depth information about how it all works in practice.

## Getting Started

To add RAC to your application:

 1. Add the ReactiveCocoa repository as a submodule of your application's
    repository. Make sure to update the submodules within with `git submodule update
    -i --recursive`
 1. Drag and drop `ReactiveCocoaFramework/ReactiveCocoa.xcodeproj` into your
    application's Xcode project or workspace.
 1. On the "Build Phases" tab of your application target, add RAC to the "Link
    Binary With Libraries" phase.
    * **On iOS**, add `libReactiveCocoa-iOS.a`.
    * **On OS X**, add `ReactiveCocoa.framework`. RAC must also be added to any
      "Copy Frameworks" build phase. If you don't already have one, simply add
      a "Copy Files" build phase and target the "Frameworks" destination.
 1. Add `$(BUILD_ROOT)/../IntermediateBuildFilesPath/UninstalledProducts/include
    $(inherited)` to the "Header Search Paths" build setting (this is only
		necessary for archive builds, but it has no negative effect otherwise).
 1. **If you added RAC to a project (not a workspace)**, you will also need to
    add the appropriate RAC target to the "Target Dependencies" of your
    application.

If you would prefer to use [CocoaPods](http://cocoapods.org), there are some
[ReactiveCocoa
podspecs](https://github.com/CocoaPods/Specs/tree/master/ReactiveCocoa) that
have been generously contributed by third parties.

To see a project already set up with RAC, check out the [Mac][GHAPIDemo] or
[iOS][RACiOSDemo] demos.

## Functional Reactive Programming
Functional Reactive Programming (FRP) is a programming paradigm for writing
software that reacts to change.

FRP is built on the abstraction of values over time. Rather than capturing
a value at a particular time, FRP provides signals that capture the past,
present, and future value. These signals can be reasoned about, chained,
composed, and reacted to.

By combining signals, software can be written declaratively, without the need
for code that continually observes and updates values. A text field can be
directly set to always show the current timestamp, for example, instead of using
additional code that watches the clock and updates the text field every second.

Signals can also represent asynchronous operations, much like [futures and
promises][]. This greatly simplifies asynchronous software, including networking
code.

One of the major advantages of FRP is that it provides a single, unified
approach to dealing with different types of reactive, asynchronous behaviors.

Here are some resources for learning more about FRP:

* [What is FRP? - Elm Language](http://elm-lang.org/learn/What-is-FRP.elm)
* [What is Functional Reactive Programming - Stack Overflow](http://stackoverflow.com/questions/1028250/what-is-functional-reactive-programming/1030631#1030631)

## FRP with ReactiveCocoa
Signals in ReactiveCocoa (RAC) are represented using `RACSignal`. Signals are
streams of values that can be observed and transformed.

Applications that are built with RAC use signals to propagate changes. It works
much like KVO, but with blocks instead of overriding
`-observeValueForKeyPath:ofObject:change:context:`.

Here's a simple example:

```objc
// When self.username changes, log the new name to the console.
//
// RACAble(self.username) creates a new RACSignal that sends a new value
// whenever the username changes. -subscribeNext: will execute the block
// whenever the signal sends a value.
[RACAble(self.username) subscribeNext:^(NSString *newName) {
    NSLog(@"%@", newName);
}];
```

But unlike KVO notifications, signals can be chained together and operated on:

```objc
// Only log names that start with "j".
//
// -filter returns a new RACSignal that only sends a new value when its block
// returns YES.
[[RACAble(self.username)
   filter:^(NSString *newName) {
       return [newName hasPrefix:@"j"];
   }]
   subscribeNext:^(NSString *newName) {
       NSLog(@"%@", newName);
   }];
```

Signals can also be used to derive state, which is a key component of FRP.
Instead of observing properties and setting other properties in response to the
new values, RAC makes it possible to express properties in terms of signals and
operations:

```objc
// Create a one-way binding so that self.createEnabled will be
// true whenever self.password and self.passwordConfirmation
// are equal.
//
// RAC() is a macro that makes the binding look nicer.
// 
// +combineLatest:reduce: takes an array of signals, executes the block with the
// latest value from each signal whenever any of them changes, and returns a new
// RACSignal that sends the return value of that block as values.
RAC(self.createEnabled) = [RACSignal 
    combineLatest:@[ RACAble(self.password), RACAble(self.passwordConfirmation) ] 
    reduce:^(NSString *password, NSString *passwordConfirm) {
        return @([passwordConfirm isEqualToString:password]);
    }];
```

Signals can be built on any stream of values over time, not just KVO. For
example, they can also represent button presses:

```objc
// Log a message whenever the button is pressed.
//
// RACCommand is a RACSignal subclass that represents UI actions.
//
// -rac_command is an addition to NSButton. The button will send itself on that
// command whenever it's pressed.
self.button.rac_command = [RACCommand command];
[self.button.rac_command subscribeNext:^(id _) {
    NSLog(@"button was pressed!");
}];
```

Or asynchronous network operations:

```objc
// Hook up a "Log in" button to log in over the network and log a message when
// it was successful.
//
// loginCommand does the actual work of logging in. loginResult sends a value
// whenever the async work is done.
self.loginCommand = [RACAsyncCommand command];

// This block will execute whenever the login command sends a value.
//
// -sequenceMany: will return a signal that represents the combination of all
// the signals returned from this block. In this case, the login result will
// send a value whenever a login attempt sends a value, completes, or errors
// out.
self.loginResult  = [self.loginCommand sequenceMany:^{
    // The hypothetical -logIn method returns a signal that sends a value when
    // the network request finishes.
    //
    // -materialize wraps up values, completions, and errors in RACEvents so errors won't
    // close the login result signal.
    return [[client logIn] materialize];
}];

[[self.loginResult 
    // Filter out everything but successful logins.
    filter:^(RACEvent *x) {
        return x.eventType == RACEventTypeCompleted;
    }]
    subscribeNext:^(id _) {
        NSLog(@"Logged in successfully!");
    }];

// Execute the login command when the button is pressed
self.loginButton.rac_command = self.loginCommand;
```

Signals can also represent timers, other UI events, or anything else that
changes over time.

Using signals for asynchronous operations makes it possible to build up more
complex behavior by chaining and transforming those signals. Work can easily be
trigged after a group of operations completes:

```objc
// Perform 2 network operations and log a message to the console when they are
// both completed.
//
// +merge: takes an array of signals and returns a new RACSignal that passes
// through the values of all of the signals and completes when all of the
// signals complete.
//
// -subscribeCompleted: will execute the block when the signal completes.
[[RACSignal 
    merge:@[ [client fetchUserRepos], [client fetchOrgRepos] ]] 
    subscribeCompleted:^{
        NSLog(@"They're both done!");
    }];
```

Signals can be chained to sequentially execute asynchronous operations, instead
of nesting callbacks with blocks. This is similar to how [futures and promises][]
are usually used:

```objc
// Log in the user, then load any cached messages, then fetch the remaining
// messages from the server. After that's all done, log a message to the
// console.
//
// The hypothetical -logInUser methods returns a signal that completes after
// logging in.
//
// -flattenMap: will execute its block whenever the signal sends a value, and
// return a new RACSignal that merges all of the signals returned from the block
// into a single signal.
[[[[client 
    logInUser] 
    flattenMap:^(User *user) {
        // Return a signal that loads cached messages for the user.
        return [client loadCachedMessagesForUser:user];
    }]
    flattenMap:^(NSArray *messages) {
        // Return a signal that fetches any remaining messages.
        return [client fetchMessagesAfterMessage:messages.lastObject];
    }]
    subscribeCompleted:^{
        NSLog(@"Fetched all messages.");
    }];
```

RAC even makes it easy to bind to the result of an asynchronous operation:

```objc
// Create a one-way binding so that self.imageView.image will be set the user's
// avatar as soon as it's downloaded.
//
// The hypothetical -fetchUserWithUsername: method returns a signal which sends
// the user.
//
// -deliverOn: creates new signals that will do their work on other queues. In
// this example, it's used to move work to a background queue and then back to the main thread.
//
// -map: calls its block with each user that's fetched and returns a new
// RACSignal that sends values returned from the block.
RAC(self.imageView.image) = [[[[client 
    fetchUserWithUsername:@"joshaber"]
    deliverOn:[RACScheduler scheduler]]
    map:^(User *user) {
        // Download the avatar (this is done on a background queue).
        return [[NSImage alloc] initWithContentsOfURL:user.avatarURL];
    }]
    // Now the assignment will be done on the main thread.
    deliverOn:RACScheduler.mainThreadScheduler];
```

That demonstrates some of what RAC can do, but it doesn't demonstrate why RAC is
so powerful. It's hard to appreciate RAC from README-sized examples, but it
makes it possible to write code with less state, less boilerplate, better code
locality, and better expression of intent.

For more sample code, check out the [Mac][GHAPIDemo] or [iOS][RACiOSDemo] demos.
Additional information about RAC can be found in the [Documentation][] folder.

## Foundation Support
There are a number of categories that provide RAC-based bridges to standard
Foundation classes. They're not included as part of the framework proper in
order to keep the framework size down.

You can find them in [RACExtensions][]. To use them, simply add them directly to
your project as needed.

## License
ReactiveCocoa is available under the [MIT License](LICENSE.md).

## More Info
ReactiveCocoa is based on .NET's [Reactive
Extensions](http://msdn.microsoft.com/en-us/data/gg577609) (Rx). Most of the
principles of Rx apply to RAC as well. There are some really good Rx resources
out there:

* [Reactive Extensions MSDN entry](http://msdn.microsoft.com/en-us/library/hh242985.aspx)
* [Reactive Extensions for .NET Introduction](http://leecampbell.blogspot.com/2010/08/reactive-extensions-for-net.html)
* [Rx - Channel 9 videos](http://channel9.msdn.com/tags/Rx/)
* [Reactive Extensions wiki](http://rxwiki.wikidot.com/)
* [101 Rx Samples](http://rxwiki.wikidot.com/101samples)
* [Programming Reactive Extensions and LINQ](http://www.amazon.com/Programming-Reactive-Extensions-Jesse-Liberty/dp/1430237473)

[Documentation]: Documentation
[Functional Reactive Programming]: http://en.wikipedia.org/wiki/Functional_reactive_programming
[futures and promises]: http://en.wikipedia.org/wiki/Futures_and_promises
[GHAPIDemo]:  https://github.com/ReactiveCocoa/GHAPIDemo
[RACiOSDemo]: https://github.com/ReactiveCocoa/RACiOSDemo
[RACExtensions]: RACExtensions

Mock library for Dart inspired by [Mockito](https://github.com/mockito/mockito).

[![Pub](https://img.shields.io/pub/v/mockito.svg)]()
[![Build Status](https://travis-ci.org/dart-lang/mockito.svg?branch=master)](https://travis-ci.org/dart-lang/mockito)

Current mock libraries suffer from specifying method names as strings, which
cause a lot of problems:

* Poor refactoring support: rename method and you need manually search/replace
  it's usage in when/verify clauses.
* Poor support from IDE: no code-completion, no hints on argument types, can't
  jump to definition

Dart's mockito package fixes these issues - stubbing and verifying are
first-class citizens.

## Let's create mocks

```dart
import 'package:mockito/mockito.dart';

// Real class
class Cat {
  String sound() => "Meow";
  bool eatFood(String food, {bool hungry}) => true;
  int walk(List<String> places);
  void sleep() {}
  void hunt(String place, String prey) {}
  int lives = 9;
}

// Mock class
class MockCat extends Mock implements Cat {}

// mock creation
var cat = new MockCat();
```

## Let's verify some behaviour!

```dart
//using mock object
cat.sound();
//verify interaction
verify(cat.sound());
```

Once created, mock will remember all interactions. Then you can selectively
verify whatever interaction you are interested in.

## How about some stubbing?

```dart
// Unstubbed methods return null:
expect(cat.sound(), nullValue);

// Stubbing - before execution:
when(cat.sound()).thenReturn("Purr");
expect(cat.sound(), "Purr");

// You can call it again:
expect(cat.sound(), "Purr");

// Let's change the stub:
when(cat.sound()).thenReturn("Meow");
expect(cat.sound(), "Meow");

// You can stub getters:
when(cat.lives).thenReturn(9);
expect(cat.lives, 9);

// You can stub a method to throw:
when(cat.lives).thenThrow(new RangeError('Boo'));
expect(() => cat.lives, throwsRangeError);

// We can calculate a response at call time:
var responses = ["Purr", "Meow"];
when(cat.sound()).thenAnswer(() => responses.removeAt(0));
expect(cat.sound(), "Purr");
expect(cat.sound(), "Meow");
```

By default, for all methods that return a value, `mock` returns `null`.
Stubbing can be overridden: for example common stubbing can go to fixture setup
but the test methods can override it.  Please note that overridding stubbing is
a potential code smell that points out too much stubbing.  Once stubbed, the
method will always return stubbed value regardless of how many times it is
called.  Last stubbing is more important, when you stubbed the same method with
the same arguments many times. In other words: the order of stubbing matters,
but it is meaningful rarely, e.g. when stubbing exactly the same method calls
or sometimes when argument matchers are used, etc.

### A quick word on async stubbing

**Using `thenReturn` to return a `Future` or `Stream` will throw an
`ArgumentError`.** This is because it can lead to unexpected behaviors. For
example:

* If the method is stubbed in a different zone than the zone that consumes the
  `Future`, unexpected behavior could occur.
* If the method is stubbed to return a failed `Future` or `Stream` and it
  doesn't get consumed in the same run loop, it might get consumed by the
  global exception handler instead of an exception handler the consumer applies.

Instead, use `thenAnswer` to stub methods that return a `Future` or `Stream`.

```
// BAD
when(mock.methodThatReturnsAFuture())
    .thenReturn(new Future.value('Stub'));
when(mock.methodThatReturnsAStream())
    .thenReturn(new Stream.fromIterable(['Stub']));

// GOOD
when(mock.methodThatReturnsAFuture())
    .thenAnswer((_) => new Future.value('Stub'));
when(mock.methodThatReturnsAStream())
    .thenAnswer((_) => new Stream.fromIterable(['Stub']));

````

If, for some reason, you desire the behavior of `thenReturn`, you can return a
pre-defined instance.

```
// Use the above method unless you're sure you want to create the Future ahead
// of time.
final future = new Future.value('Stub');
when(mock.methodThatReturnsAFuture()).thenAnswer((_) => future);
```


## Argument matchers

```dart
// You can use arguments itself:
when(cat.eatFood("fish")).thenReturn(true);

// ... or collections:
when(cat.walk(["roof","tree"])).thenReturn(2);

// ... or matchers:
when(cat.eatFood(argThat(startsWith("dry"))).thenReturn(false);

// ... or mix aguments with matchers:
when(cat.eatFood(argThat(startsWith("dry")), true).thenReturn(true);
expect(cat.eatFood("fish"), isTrue);
expect(cat.walk(["roof","tree"]), equals(2));
expect(cat.eatFood("dry food"), isFalse);
expect(cat.eatFood("dry food", hungry: true), isTrue);

// You can also verify using an argument matcher:
verify(cat.eatFood("fish"));
verify(cat.walk(["roof","tree"]));
verify(cat.eatFood(argThat(contains("food"))));

// You can verify setters:
cat.lives = 9;
verify(cat.lives=9);
```

If an argument other than an ArgMatcher (like `any`, `anyNamed()`, `argThat`,
`captureArg`, etc.) is passed to a mock method, then the `equals` matcher is
used for argument matching. If you need more strict matching consider use
`argThat(identical(arg))`.


## Verifying exact number of invocations / at least x / never

```dart
cat.sound();
cat.sound();

// Exact number of invocations:
verify(cat.sound()).called(2);

// Or using matcher:
verify(cat.sound()).called(greaterThan(1));

// Or never called:
verifyNever(cat.eatFood(any));
```

## Verification in order

```dart
cat.eatFood("Milk");
cat.sound();
cat.eatFood("Fish");
verifyInOrder([
  cat.eatFood("Milk"),
  cat.sound(),
  cat.eatFood("Fish")
]);
```

Verification in order is flexible - you don't have to verify all interactions
one-by-one but only those that you are interested in testing in order.

## Making sure interaction(s) never happened on mock

```dart
  verifyZeroInteractions(cat);
```

## Finding redundant invocations

```dart
cat.sound();
verify(cat.sound());
verifyNoMoreInteractions(cat);
```

## Capturing arguments for further assertions

```dart
// Simple capture:
cat.eatFood("Fish");
expect(verify(cat.eatFood(captureAny)).captured.single, "Fish");

// Capture multiple calls:
cat.eatFood("Milk");
cat.eatFood("Fish");
expect(verify(cat.eatFood(captureAny)).captured, ["Milk", "Fish"]);

// Conditional capture:
cat.eatFood("Milk");
cat.eatFood("Fish");
expect(verify(cat.eatFood(captureThat(startsWith("F")).captured, ["Fish"]);
```

## Waiting for an interaction

```dart
// Waiting for a call:
cat.eatFood("Fish");
await untilCalled(cat.chew()); //completes when cat.chew() is called

// Waiting for a call that has already happened:
cat.eatFood("Fish");
await untilCalled(cat.eatFood(any)); //will complete immediately
```

## Resetting mocks

```dart
// Clearing collected interactions:
cat.eatFood("Fish");
clearInteractions(cat);
cat.eatFood("Fish");
verify(cat.eatFood("Fish")).called(1);

// Resetting stubs and collected interactions:
when(cat.eatFood("Fish")).thenReturn(true);
cat.eatFood("Fish");
reset(cat);
when(cat.eatFood(any)).thenReturn(false);
expect(cat.eatFood("Fish"), false);
```

## Spy

```dart
// Spy creation:
var cat = spy(new MockCat(), new Cat());

// Stubbing - before execution:
when(cat.sound()).thenReturn("Purr");

// Using mocked interaction:
expect(cat.sound(), "Purr");

// Using a real object:
expect(cat.lives, 9);
```

## Debugging

```dart
// Print all collected invocations of any mock methods of a list of mock objects:
logInvocations([catOne, catTwo]);

// Throw every time that a mock method is called without a stub being matched:
throwOnMissingStub(cat);
```

## Strong mode compliance

Unfortunately, the use of the arg matchers in mock method calls (like `cat.eatFood(any)`)
violates the [Strong mode] type system. Specifically, if the method signature of a mocked
method has a parameter with a parameterized type (like `List<int>`), then passing `any` or
`argThat` will result in a Strong mode warning:

> [warning] Unsound implicit cast from dynamic to List&lt;int>

In order to write Strong mode-compliant tests with Mockito, you might need to use `typed`,
annotating it with a type parameter comment. Let's use a slightly different `Cat` class to
show some examples:

```dart
class Cat {
  bool eatFood(List<String> foods, [List<String> mixins]) => true;
  int walk(List<String> places, {Map<String, String> gaits}) => 0;
}

class MockCat extends Mock implements Cat {}

var cat = new MockCat();
```

OK, what if we try to stub using `any`:

```dart
when(cat.eatFood(any)).thenReturn(true);
```

Let's analyze this code:

```
$ dartanalyzer --strong test/cat_test.dart
Analyzing [lib/cat_test.dart]...
[warning] Unsound implicit cast from dynamic to List<String> (test/cat_test.dart, line 12, col 20)
1 warning found.
```

This code is not Strong mode-compliant. Let's change it to use `typed`:

```dart
when(cat.eatFood(typed(any)))
```

```
$ dartanalyzer --strong test/cat_test.dart
Analyzing [lib/cat_test.dart]...
No issues found
```

Great! A little ugly, but it works. Here are some more examples:

```dart
when(cat.eatFood(typed(any), typed(any))).thenReturn(true);
when(cat.eatFood(typed(argThat(contains("fish"))))).thenReturn(true);
```

Named args require one more component: `typed` needs to know what named argument it is
being passed into:

```dart
when(cat.walk(typed(any), gaits: typed(any, named: 'gaits')))
    .thenReturn(true);
```

Note the `named` argument. Mockito should fail gracefully if you forget to name a `typed`
call passed in as a named argument, or name the argument incorrectly.

One more note about the `typed` API: you cannot mix `typed` arguments with `null`
arguments:

```dart
when(cat.eatFood(null, typed(any))).thenReturn(true); // Throws!
when(cat.eatFood(
    argThat(equals(null)),
    typed(any))).thenReturn(true); // Works.
```

[Strong mode]: https://github.com/dart-lang/dev_compiler/blob/master/STRONG_MODE.md

## How it works

The basics of the `Mock` class are nothing special: It uses `noSuchMethod` to catch
all method invocations, and returns the value that you have configured beforehand with
`when()` calls.

The implementation of `when()` is a bit more tricky. Take this example:

```dart
// Unstubbed methods return null:
expect(cat.sound(), nullValue);

// Stubbing - before execution:
when(cat.sound()).thenReturn("Purr");
```

Since `cat.sound()` returns `null`, how can the `when()` call configure it?

It works, because `when` is not a function, but a top level getter that _returns_ a function.
Before returning the function, it sets a flag (`_whenInProgress`), so that all `Mock` objects
know to return a "matcher" (internally `_WhenCall`) instead of the expected value. As soon as
the function has been invoked `_whenInProgress` is set back to `false` and Mock objects behave
as normal.

> **Be careful** never to write `when;` (without the function call) anywhere. This would set
> `_whenInProgress` to `true`, and the next mock invocation will return an unexpected value.

The same goes for "chaining" mock objects in a test call. This will fail:

```dart
var mockUtils = new MockUtils();
var mockStringUtils = new MockStringUtils();

// Setting up mockUtils.stringUtils to return a mock StringUtils implementation
when(mockUtils.stringUtils).thenReturn(mockStringUtils);

// Some tests

// FAILS!
verify(mockUtils.stringUtils.uppercase()).called(1);
// Instead use this:
verify(mockStringUtils.uppercase()).called(1);
```

This fails, because `verify` sets an internal flag, so mock objects don't return their mocked
values anymore but their matchers. So `mockUtils.stringUtils` will *not* return the mocked
`stringUtils` object you put inside.


You can look at the `when` and `Mock.noSuchMethod` implementations to see how it's done.
It's very straightforward.

**NOTE:** This is not an official Google product

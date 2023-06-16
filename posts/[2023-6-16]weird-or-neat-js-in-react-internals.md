<!---
title: Weird or Neat JavaScript in React Internals
description: I really enjoy seeing all the different ways a programming language can be used in unexpected ways, and the JavaScript community has historically been very creative. Although it's almost always an implementation detail for its users, React is no stranger to creative problem solving
socialImage: https://user-images.githubusercontent.com/5233399/246327123-7c72f3b9-2141-42dc-9558-a2aed6343d92.png
slackLabel1: Reading Time
slackLabel1Value: 5 minutes
slackLabel2: Publish Date
slackLabel2Value: June 16, 2023
draft: true
-->

# Weird or Neat JavaScript in React Internals

I really enjoy seeing all the different ways a programming language can be used in unexpected ways, and the JavaScript community has [historically](https://www.sitepen.com/blog/windowname-transport) [been](https://www.alexrothenberg.com/2013/02/11/the-magic-behind-angularjs-dependency-injection.html) _very_ creative. Although it's almost always an implementation detail for its users, React is no stranger to creative problem solving. Here's 5 of the more bizarre uses of JavaScript lurking in React's internals

## Throwing Promises

_Note: The [React Docs](https://react.dev/reference/react/Suspense) are very clear that this API is "unstable and undocumented"_

React's "Suspense" feature enables data fetching libraries to interrupt rendering and signal to a parent component that they need to pause to fetch data. This feature relies on a quirk in JavaScript that allows the `throw` statement to receive _any_ value, not just an Error. 

```js
try {
   throw { an: 'object' };
} catch (err) {
   console.log('Error was', err); // Logs "Error was { an: 'object' }"
}
```

React's (undocumented) APIs for Suspense-enabled data fetching [work by throwing a `Promise` object](https://github.com/facebook/react/issues/17526#issuecomment-769151686) to signal that a request is in flight and rendering should be suspended. 

## Monkey-patching Global Fetch

_Note: Gated by `enableCache` and `enableFetchInstrumentation` feature flags_

The in progress caching APIs (to go along with Suspense) have _some_ form of automatic HTTP request caching. This is done by [overwriting the global copy of `fetch` with a new wrapper](https://github.com/facebook/react/blob/fc929cf4ead35f99c4e9612a95e8a0bb8f5df25d/packages/react/src/ReactFetch.js#L128-L142)

```js
try {
  // eslint-disable-next-line no-native-reassign
  fetch = cachedFetch;
} catch (error1) {
  try {
    // In case assigning it globally fails, try globalThis instead just in case it exists.
    globalThis.fetch = cachedFetch;
  } catch (error2) {
    // Log even in production just to make sure this is seen if only prod is frozen.
    // eslint-disable-next-line react-internal/no-production-logging
    console.warn(
      'React was unable to patch the fetch() function in this environment. ' +
        'Suspensey APIs might not work correctly as a result.',
    );
  }
}
```

The [original PR](https://github.com/facebook/react/pull/25516) explains some of the motivation

> this gives deduping at the network layer to avoid costly mistakes and to make the simple case simple.

## Expando Properties on Promises

In web development[^1], I believe the term "expando" was first introduced via [the `expando` property](https://www.jb51.net/shouce/dhtml/properties/expando.html) in Internet Explorer's [JScript](https://en.wikipedia.org/wiki/JScript). Over the years, I've seen it used to mean "adding a non-standard property to some host object or built-in."

The [new `use()` hook](https://github.com/facebook/react/pull/25084) works by adding a few expando properties (`status`, `value`, and `reason`) to any Promise passed to it. These properties allow React to _synchronously_ inspect the result of a Promise, which is not possible with standard JavaScript Promises.

The RFC [First Class Support for Promises](https://github.com/reactjs/rfcs/blob/9c21ca1a8e39d19338ba750ee3ff6f6c0724a51c/text/0000-first-class-support-for-promises.md#reading-the-result-of-a-promise-that-was-read-previously) makes it clear that the React team would prefer a native JavaScript API

> Keep in mind that React will not add these extra fields to every promise, only those promise objects that are passed to use. It does not require modifying the global environment or the Promise prototype, and will not affect non-React code.
>
> Although this convention is not part of the JavaScript specification, we think it's a reasonable way to track a promise's result. The ideal is that the lifetime of the resolved value corresponds to the lifetime of the promise object. The most straightforward way to implement this is by adding a property directly to the promise.
>
> An alternative would be to use a WeakMap, which offers similar benefits. The advantage of using a property instead of a WeakMap is that other frameworks besides React can access these fields, too. For example, a data framework can set the status and value fields on a promise preemptively, before passing to React, so that React can unwrap it without waiting a microtask.
>
> If JavaScript were to ever adopt a standard API for synchronously inspecting the value of a promise, we would switch to that instead. (Indeed, if an API like Promise.inspect existed, this RFC would be significantly shorter.)

## Custom Directive Pragmas

TODO

[^1]: Other languages (C#, Groovy) have similarish functionality using the term "expando", but the term has taken on a slightly different meaning over the years in JS land
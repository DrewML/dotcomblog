<!---
title: React Internals do some Weird Shit™
description: I really enjoy seeing all the different ways a programming language can be used in unexpected ways, and the JavaScript community has historically been very good at pushing the language to its limits, despite some deficiencies. I wanted to jot down some of the JS tricks I've seen in use within the React source
socialImage: https://user-images.githubusercontent.com/5233399/248584278-256e1376-d48f-439c-ae65-86d344abd50d.png
slackLabel1: Reading Time
slackLabel1Value: 7 minutes
slackLabel2: Publish Date
slackLabel2Value: June 25, 2023
-->

# React Internals do some _Weird Shit™_

I really enjoy seeing all the different ways a programming language can be used in unexpected ways, and the JavaScript community has [historically](https://www.sitepen.com/blog/windowname-transport) [been](https://www.alexrothenberg.com/2013/02/11/the-magic-behind-angularjs-dependency-injection.html) [_very_](https://en.wikipedia.org/wiki/JSONP) good at pushing the language to its limits, despite some deficiencies. I wanted to jot down some of the JS tricks I've seen in use within the React source. Like them or not, they're probably in your bundle, so it's worth knowing the how and why.

## Throwing Promises

_Note: The [React Docs](https://react.dev/reference/react/Suspense) are very clear that this API is "unstable and undocumented"_

React's "Suspense" feature enables data fetching libraries to interrupt ("suspend") rendering and signal to a component that it needs to pause to fetch data. The API exposed to _consumers_ of Suspense-enabled libraries has an interesting feature: it almost looks invisible.

```js
function SomeComponentFetchingData({ id }) {
   const data = readFromSuspenseEnabledDataSource(id);
   return <div>{data.someprop}</div>;
}
```

Although it doesn't stand out reading the code, the `readFromSuspenseEnabledDataSource` function signals to React that `SomeComponentFetchingData` needs to Suspend, and it prevents the next line of code (the access to `data.someprop`) from executing until a Promise resolves. The `readFromSuspenseEnabledDataSource` function does this by [_throwing a Promise_ synchronously](https://github.com/facebook/react/issues/17526#issuecomment-769151686):


```js
// pseudo-code, does not fulfill the full contract needed by React, and doesn't include error handling.
function readFromSuspenseEnabledDataSource(id) {
   if (someCache.has(id)) {
      // synchronously return value if already fetched
      return someCache.get(id);
   }

   if (pendingRequests.has(id)) {
      // if request for this data is in-flight,
      // throw same Promise as we did before
      throw promise;
   }
   
   // trigger data fetch
   const promise = fetchSomeResource(id);
   promise.then(res => {
      // remove promise from pending list
      // so we know it has completed
      pendingRequests.remove(id);
      // record result in our cache so we can
      // return result synchronously later
      someCache.set(id, res);
   });
   // throw promise to signal we're suspending
   throw promise;
}
```

Throwing promises works because of a fun little quirk in JavaScript: the `throw` statement takes _any_ value, not just an Error object.

```js
try {
   throw 'lol';
} catch (err) {
   console.log(typeof err); // "string"
}
```

It _feels_ gross, but I'm really impressed with the general idea of using `throw` to allow any function in a component's callstack to yield some value. It's probably not super useful as a pattern outside of React, though, as there's quite a few constraints placed on React components that made this design feasible (mainly the management of state and effects).

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

I can see the reasoning for _most_ of the tricks used in this post, but the monkey patching of `fetch` feels out of place. Unless I missed something else, it doesn't seem hugely beneficial compared to React exporting a `fetch` wrapper. 

It is worth noting that this function is _mostly_ a proxy to the native `fetch`, as it [checks whether the call to fetch is happening within a React component context](https://github.com/facebook/react/blob/fc929cf4ead35f99c4e9612a95e8a0bb8f5df25d/packages/react/src/ReactFetch.js#L49-L53):

```js
const dispatcher = ReactCurrentCache.current;
if (!dispatcher) {
  // We're outside a cached scope.
  return originalFetch(resource, options);
}
```

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

A "Directive Pragma" is a feature of JavaScript that gives special meaning to a string literal at the top of a Script, Module, or Function body. The 1 well-known Pragma that most JavaScript developers have seen is to enable Strict mode:

```js
'use strict'; // strict module

function abc() {
   'use strict'; // strict function
}
```

With Server Components and Server Actions, React introduced 2 _custom_ Directive Pragmas:

```js
'use client'; // Mark that a file contains "Client Components"

function handleSomeFormSubmit() {
   'use server'; // Mark that a function is a "Server Action"
}
```

I've written more about this in another post, ["use client" and "use server" are standard-ish](https://blog.levineandrew.com/use-client-and-use-server-are-standard-ish).

[^1]: Other languages (C#, Groovy) have similarish functionality using the term "expando", but the term has taken on a slightly different meaning over the years in JS land

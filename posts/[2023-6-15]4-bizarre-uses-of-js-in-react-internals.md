<!---
title: 4 Bizarre Uses of JavaScript in React Internals
description: I really hate that the title of this article sounds like a Buzzfeed Listicle, but I'm so bad at naming things. React also does some bizarre stuff
socialImage: https://null
slackLabel1: Reading Time
slackLabel1Value: 5 minutes
slackLabel2: Publish Date
slackLabel2Value: June 15, 2023
draft: true
-->

# 5 Bizarre Uses of JavaScript in React Internals

I really enjoy seeing all the different ways a programming language can be used in unexpected ways, and the JavaScript community has historically been _very_ creative. Although it's almost always an implementation detail for its users, React is no stranger to creative problem solving. Here's 5 of the more bizarre uses of JavaScript lurking in React's internals

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

## Monkey-patching Fetch

## Custom Directive Pragmas

## Expando Properties on Promises


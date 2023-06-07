<!---
title: Apollo Client Performance
description: A deep-dive into the performance of Apollo Client in a real React application in Production
socialImage: https://user-images.githubusercontent.com/5233399/235226908-024df0ff-3a76-4304-850c-2707e6907ebc.jpg
draft: true
-->


# Apollo Client Real World Performance on a Large Ecommerce Site
I work on a large Ecommerce site for a major grocery chain. The application is built using a few tools that have been popular in the front-end space for the last few years:

- React (w/ SSR via a Node.js app)
- Apollo Client
- React-Apollo Bindings

The purpose of this document is to provide a real life example of the performance cliffs that are present in Apollo Client, and should be considered when deciding to build an application with it.

The goal is _not_ to discredit any of the hard work from the folks at Apollo the company. _However_, marketing a library and encouraging wide adoption does come with the responsibility of ensuring the web stays fast for everyone, and it's important for web developers to highlight when tools fall short of their promises.

## How SSR in React Works

Before digging into the performance challenges with Apollo Client, let's go through a quick summary of how SSR works in a React application.

### History
Until recently, React did not provide a first-class API to perform asynchronous operations _during rendering_ while executing on the server-side[^1]. This posed a challenge for 3rd party library authors that often wrote their APIs thinking of React's client-first model.

Some folks in the open-source world came up with the concept of "two-pass rendering" to try and address this. The general idea is that you render the React app once to initiate async work, and when the async work is complete you render the React app a second time. On the second render, the React app can _synchronously_ read from the cache that was populated in the first render.

This is a cool work-around, but it has real performance implications: you're doing every bit of expensive work twice.

### Simple (Synchronous) SSR Render
The steps to successfully render a _basic_ application with SSR are roughly:

1. Load the app in Node.js, and render the root component of the app using [`renderToString`](https://reactjs.org/docs/react-dom-server.html#rendertostring) or [`renderToPipeableStream`](https://reactjs.org/docs/react-dom-server.html#rendertopipeablestream).
2. Flush the generated HTML down to the client
3. Load React on the client, and call [`hydrateRoot`](https://reactjs.org/docs/react-dom-client.html#hydrateroot), which instructs React to basically rebuild the "Virtual DOM" in memory, and ensure the current markup in the DOM matches with what React expects.

This SSR setup will _not_ work with Apollo Client and React-Apollo because they need to perform work async.

### Apollo Client SSR (Asynchronous)
Apollo Client implements the "two-pass" rendering strategy, but it's a bit of a misnomer: Apollo has no limit on the number of "passes" it will perform[^2].

The way you do SSR, with Apollo Client:

1. Load the app in Node.js, and kick off Apollo Client's data-fetching on the server by passing the root component of the app to Apollo's [`getDataFromTree`](https://www.apollographql.com/docs/react/performance/server-side-rendering/#executing-queries-with-getdatafromtree).
2. `getDataFromTree` calls React's `renderToString`, and waits for the synchronous render to complete. This causes all `useQuery` calls in _some_ components to kick-off their HTTP requests. `getDataFromTree` then discards the results from React.
3. When HTTP calls are complete, Apollo Client populates its cache, and then calls `renderToString` _again_. This time, the React tree will see results _synchronously_ in the Apollo Cache, and React will keep rendering until it finds the next unresolved `useQuery` call. This kicks off the next round of HTTP requests. `getDataFromTree`, again, discards the results from React.
4. Noticed a pattern yet? `getDataFromTree` will continue to render your app from top to bottom, recursively, until it's discovered every `useQuery` in the component hierarchy.
5. Once Apollo Client's cache has been populated with results for _all_ `useQuery` calls, `getDataFromTree` is complete.
6. The eventual result of `getDataFromTree` will be markup that can be hydrated again on the client.
7. Flush the generated HTML down to the client, along with a script tag that includes a serialized copy of the Apollo Client cache
8. Load React on the client, construct an Apollo Client instance with the results from the serialized cache, and call [`hydrateRoot`](https://reactjs.org/docs/react-dom-client.html#hydrateroot), which instructs React to basically rebuild the "Virtual DOM" in memory, and ensure the current markup in the DOM matches with what React expects. During this process, React-Apollo will read query results from the cache instead of going to the network again.

There's a [comment in the implementation of `getDataFromTree`](https://github.com/apollographql/apollo-client/blob/be9786f6d4228c79fad1d49f908e8d19ed4d9cbe/src/react/ssr/getDataFromTree.ts#L38-L42) that acknowledges this is non-ideal:

```js
// Always re-render from the rootElement, even though it might seem
// better to render the children of the component responsible for the
// promise, because it is not possible to reconstruct the full context
// of the original rendering (including all unknown context provider
// elements) for a subtree of the original component tree.
```

## How we use Apollo Client

Our usage of Apollo Client is pretty typical of what you'd see in most applications following Apollo's official documentation. Engineers divide their queries into multiple queries that are distributed amongst the appropriate components. This is the recommendation in Apollo's [Best Practices](https://www.apollographql.com/docs/react/data/operation-best-practices#query-only-the-data-you-need-where-you-need-it) in the official docs.

> One of GraphQL's biggest advantages over a traditional REST API is its support for declarative data fetching. Each component can (and should) query exactly the fields it requires to render, with no superfluous data sent over the network.
>
> If instead your root component executes a single, enormous query to obtain data for all of its children, it might query on behalf of components that aren't even rendered given the current state. This can result in a delayed response, and it drastically reduces the likelihood that the query's result can be reused by a server-side response cache.
>
> In the large majority of cases, a query such as the following should be divided into multiple queries that are distributed among the appropriate components

My experience in Production over the last several years has shown me that this is _not good advice_ if you care about the performance of your app, both in SSR and during client rendering.

### Example

Below is a watered down example of a React component tree you may end up with when following Apollo's recommendation to distribute queries.


<p align="center"><img width="1204" alt="image" src="https://user-images.githubusercontent.com/5233399/201801633-4ef1eb3f-31a0-44a2-a8b7-5eb2ff318d99.png"></p>

It may not be obvious, but this example requires calling React's `renderToString` 4 separate times on the server for a single request.

1. First `renderToString` call reaches `<HomePage />`. This component likely returns a loading state, and rendering stops until that fetch is complete.
2. Second `renderToString` is called. This time it renders the loading state for `<Header />`, `<Footer />`, `<NewProducts />`, `<HomePageHero />`, and `<TrendingProducts />`. Rendering stops until those fetches complete
3. Third `renderToString` is called. This time it renders the loading state for `<MiniCart />`. Rendering stops until that fetch completes
4. Fourth `renderToString` is called. This time every component found its data in the cache, and we can use the resulting markup from React.

### Wait, how many HTTP Requests is that?

Yep, that's 7 separate HTTP requests to fetch data. While it occasionally makes sense to separate out HTTP requests for performance (fast vs slow queries), this is a bit excessive, especially since a selling point of GraphQL is limiting the number of HTTP requests on the client (compared to REST).

Apollo offers one solution to this in the form of the [`BatchHttpLink`](https://www.apollographql.com/docs/react/api/link/apollo-link-batch-http/). It works by operating in 10ms (configurable) windows. Every 10ms, it grabs the last queries its seen, and batches them into 1 HTTP request.

This means that, for many requests, you're introducing an artificial delay on the client! If the main thread is chomping through a large number of tasks when the end of the 10ms window is hit, the delay can be much longer.

## Data!

### Bundle Impact

_Note: These numbers are before minification. You probably shouldn't expect minification to drastically improve the performance of your app, though._

This data was extracted with the SourceMap Explorer in [Bundle Buddy](https://www.bundle-buddy.com/).

**118 KB from the `@apollo/client` package and its dependencies**
<p align="center"><img width="1489" alt="image" src="https://user-images.githubusercontent.com/5233399/201781003-a17b0fe1-5342-4ae7-ac76-e5aac3e6ccca.png"></p>

**55 KB from the `graphql` npm package (required by Apollo Client)**
<p align="center"><img width="1490" alt="image" src="https://user-images.githubusercontent.com/5233399/201781206-d8a3314b-9282-4fa6-aa12-d50958aceb4f.png"></p>


### SSR Performance

All the traces below were captured in Node.js v16.12.0 on a 2019 MacBook Pro i9.

#### Cache Write Costs

Similar to the client-side, writing to the cache is _extremely_ expensive. For a single write to the cache during _one_ of our renders (remember we have to render _many_ times), we block the main thread for over 300ms:

<p align="center"><img width="1022" alt="image" src="https://user-images.githubusercontent.com/5233399/201809896-d7927472-6541-4fc8-961a-4ab545453a77.png"></p>


### Hydration/Client-Side Performance

All the traces below were captured in Chrome 107.0.5304.105 on a newly purchased [2022 Moto G Power](https://www.motorola.com/us/smartphones-moto-g-power-gen-2/p) device running Android 11. According to [@slightlyoff](https://github.com/slightlyoff), this device is [representative of a mid-tier Android device in 2022](https://twitter.com/slightlylate/status/1587729481216987136). Don't worry, though: the Samsung website advertises this device as having `Lag-free performance` 😎.

#### Query Parsing Costs

Although you author queries as strings, Apollo Client requires each query to be parsed so it can be analyzed on the client-side. This is done using the `gql` [tagged template literal](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals#tagged_templates) provided by the [`graphql-tag`](https://www.npmjs.com/package/graphql-tag) library.

**Example**
```js
import { gql, useQuery } from "@apollo/client";

const QUERY = gql`
  query {
    currentUser {
      id
    }
  }
`;

export default function UserIDComponent() {
  const { data } = useQuery(QUERY);
  return (
    <div>
      {data ? data.currentUser.id : "Loading"}
    </div>
  );
}
```

In this example, note that the usage of the `gql` tag is in the "top-level" of this file (module). This means the moment this file is imported on the client-side, the main thread will be blocked running GraphQL's parser over this query.

To render our Home Page, my device spent ~73ms running all the queries in the bundle _before_ React even had a chance to start hydrating the app.

**Total time spent by the `gql` tag prior to React hydration**
<p align="center"><img width="549" alt="image" src="https://user-images.githubusercontent.com/5233399/201786753-0dd80a89-0578-4743-9d1d-2549aa716ea6.png"></p>

**Zooming in shows each file going through GraphQL's recursive descent parser**
<p align="center"><img width="1140" alt="image" src="https://user-images.githubusercontent.com/5233399/201787334-cc5336d4-b67a-4bd1-ab89-e148909da923.png"></p>

#### Query Execution Costs

The cost of executing queries is, by a wide margin, the most expensive thing Apollo does on the client-side. During React hydration, it easily blocked the main thread for > 1s on my device. As far as I can tell, most of this slowness can be attributed to the [`InMemoryCache`](https://master--apollo-client-docs.netlify.app/docs/react/api/cache/InMemoryCache/).

Apollo's React integration with `useQuery` works in 2 phases:

1. `useQuery` calls [`QueryData.prototype.execute`](https://github.com/apollographql/apollo-client/blob/eb9f62a9dfebe3e8418a9578ef66948e41a4fd56/src/react/data/QueryData.ts#L65), which executes immediately during a components' render
2. `useQuery` calls [`QueryData.prototype.afterExecute`](https://github.com/apollographql/apollo-client/blob/eb9f62a9dfebe3e8418a9578ef66948e41a4fd56/src/react/data/QueryData.ts#L101) in a `useEffect`, meaning it runs immediately _after_ React has updated the DOM with any changes from the last render.

For the Home Page in our application, both of these phases can block the main thread for several hundred milliseconds, even though we already have a cached result in memory and don't need to touch the network. For many devices, it would be cheaper if we ignored the cache and fetched a fresh result from the network.

**Execute phase blocking for 556ms**
<p align="center"><img width="451" alt="image" src="https://user-images.githubusercontent.com/5233399/201797287-64b2fd36-08f1-4dba-a386-9f2a68d710aa.png"></p>

**After-Execute phase blocking for 351ms**
<p align="center"><img width="714" alt="image" src="https://user-images.githubusercontent.com/5233399/201797435-a63b5009-dc03-4d9a-8aaa-002993d442f9.png">></p>

These costs are so expensive compared to other libraries that they stand out clearly when looking at the total cost of React's hydration

**Full Hydration**

<p align="center"><img width="815" alt="image" src="https://user-images.githubusercontent.com/5233399/201798286-ecf1ab21-1821-43e7-bba3-35244cb507d9.png"></p>

**Total Hydration Costs of Execute + After Execute (~1.6s)**
<p align="center"><img width="910" alt="image" src="https://user-images.githubusercontent.com/5233399/201800106-0ab6aee6-9ace-4e79-831d-3c4960746843.png"></p>

It's important to note that you pay this cost for every component on the page using the `useQuery` hook. In this example, I'm only showing the costs _for a single component_. If you display 50 products on a page, and each of your `<Product />` components has its own `useQuery` call, you're running both these phases 50 times.

I should note that not every `execute` and `afterExecute` phase is this expensive (although the cumulative costs still aren't great for large component trees). The challenge is that it's not remotely clear what makes these phases so expensive, or what the recommendations would be to remedy.

[^1]: React now supports this directly via [`renderToPipeableStream`](https://reactjs.org/docs/react-dom-server.html#rendertopipeablestream) on the server and [`React.Suspense`](https://reactjs.org/docs/react-api.html#reactsuspense)
[^2]: This can be observed by looking at the [recursive calls to `process` in `getDataFromTree`](https://github.com/apollographql/apollo-client/blob/32691bd2bae6229dfe2860e724e2cdbea5014fd0/src/react/ssr/getDataFromTree.ts#L54)

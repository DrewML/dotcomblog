<!---
title: Quantifying the Impact of Styled Components on Server Response Times
description: Working off of a suspicion, I spent some time at work trying to properly attribute the amount of time spent during SSR to Styled Components
socialImage: https://user-images.githubusercontent.com/5233399/235488415-310fbaad-a1e5-475e-b929-c4e87ef2811d.jpg
draft: true
-->

# Quantifying the Impact of Styled Components on Server Response Times

For the few years I've worked at my current gig, I've operated under the assumption that Styled Components probably wasn't doing us any favors with regard to runtime performance. Conceptually, having to basically run a miniature compiler (parse/transform/serialize) in N components _during rendering_ does not sound fast.

I recently had some time during a hackweek to explore this idea further and see if I could put some real numbers behind it. I'm sharing my work in hopes that it's helpful to other devs evaluating whether to re-platform their approach to CSS.

## Unminified Styled Components

To start my journey, I went through my typical Node.js profiling routine: start the app w/ `--inspect`, execute some code path a bunch to give the JIT a chance to warm-up, then connect via the Debugging Protocol and capture a CPU trace. Unfortunately, this path wasn't super productive initially because Styled Components only ships _minified_ builds with mangled identifier names to `npm`. This can be observed by browsing the [published artifacts for the package](https://unpkg.com/browse/styled-components@5.3.10/).

If you've ever tried to understanding a CPU trace of minified JavaScript, you likely already know this is a dead end path. So I set out to build my own unminified copy of Styled Components. The path to get this done was roughly:

1. Clone the Styled Components repo from Github
2. `git checkout v5.3.10` to match our app's version
3. Open `package.json`, find the `rollup-plugin-flow` entry, and change `github:probablyup/rollup-plugin-flow#breaking-update-flow-remove-types` to `github:quantizor/rollup-plugin-flow#breaking-update-flow-remove-types` (contributor changed their Github username, dep will 404 without change)
4. Run `yarn` at the root to setup all workspaces
5. Open `packages/styled-components/rollup.config.js` and remove all references to `minifierPlugin` in the various build configs within the file
6. Run `yarn build` in `packages/styled-components`.
7. Unminified copies of the library will be written to `packages/styled-components/dist`

At this point, I was able to go into my app's `node_modules/styled-components/dist` dir and manually update to my new unminified copy.

## Unminified ReactDOM

The next challenge to solve was better understanding pieces of the trace outside of Styled Components. Specifically, it is difficult to understand the interplay of React's rendering lifecycle and Styled Components when you can't identify what's happening in React itself.

If you're on a newer version of React, this may not be a problem for you. The React Team recently [stopped mangling identifier names in builds](https://github.com/facebook/react/pull/28881). However, my app is on React `18.2.0` which shipped prior to this PR landing. So, we get to learn how to build React.

1. Clone the React repo from Github
2. `git checkout v18.2.0` to match our app's version
3. Run `yarn` at the root to setup all workspaces
4. Run `node ./scripts/rollup/build.js --pretty`
5. Unminified copies of `ReactDOM` will be written to `build/node_modules`

At this point, I was able to go into my app's `node_modules/react-dom/cjs` dir and manually update to my new unminified copy.

## Unminified Application Code

Finally, I wanted to see my app's component and hook names in traces, so I could help orient myself when digging through a large trace. This is a `Next.js` app using Webpack, so I was able to make this change by modifying the Webpack config in `next.config.js`:

```js
webpackConfig.optimization.minimizer = [
  new TerserPlugin({
    parallel: true,
    terserOptions: {
      keep_fnames: true,
      keep_classnames: true,
    },
  }),
];
```

## The Fun Part: Analyzing a Trace


<!---
title: Hide Your Large JSON Files from The TypeScript Compiler
description: I spent some time at work looking into speeding up TypeScript type checking via tsc. One of the biggest wins I found was also one of the simplest to implement.
socialImage: https://github.com/DrewML/dotcomblog/assets/5233399/84efd441-b6af-4d2d-82ad-598959a8666a
slackLabel1: Reading Time
slackLabel1Value: 3 minutes
slackLabel2: Publish Date
slackLabel2Value: November 6, 2023
-->

# Hide Your Large JSON Files from The TypeScript Compiler

I spent some time at work looking into speeding up TypeScript type checking via tsc. One of the biggest wins I found was also one of the simplest to implement.

## resolveJsonModule

TypeScript supports a pretty cool feature, `resolveJsonModule`, which gives you 2 key things:

1. Will prevent the compiler from barking if you import a JSON file as if it was a JavaScript module
2. Will infer types based on the shape of the imported JSON


Unfortunately, the inference provided by this feature can become quite expensive if you have massive JSON files being imported anywhere.

## Lottie

My employer's application is using [Lottie](http://airbnb.io/lottie/) to support Adobe After Effects animations across multiple platforms. These animations are driven by JSON files that can be pretty large (hint hint).

We have 7 Lottie JSON files currently in our repo, with 3 of them being > 800kb on disk. Following the [performance tracing](https://github.com/microsoft/TypeScript-wiki/blob/main/Performance-Tracing.md) steps for the TypeScript compiler (`--generateTrace`), I was able to identify pretty quickly that each of these large JSON files contributed to slowness during compilation when imported into one of our components.

For one React component that imported 2 of these large JSON files, TypeScript spent 2.8s processing the JSON's types on average on my 2019 i9 MBP. 

## Hiding the JSON

Because Lottie JSON has the sole purpose to drive animations and not be used in our application code directly, there's no value in us getting types inferred. However, we have other use cases where we still need to rely on `resolveJsonModule` when pulling in some smaller config files.

I ended up hiding these files from TypeScript by making a few changes to our app:

1. Renamed Lottie JSON files from `*.json` to `*.lottiejson`
2. Configured webpack to use `json-loader` with the `.lottiejson` file extension (default config in webpack for `*.json` files)
3. Wrote a Jest transform that emulates the default JSON handling, i.e. `module.exports = ${sourceText};`, and added to `moduleFileExtensions`
4. Added a `.d.ts` file that does `declare module '*.lottiejson'`, with a default export typed as `object`

After making these changes, we observed an average speedup of 12s in our CI pipeline when running `tsc`.

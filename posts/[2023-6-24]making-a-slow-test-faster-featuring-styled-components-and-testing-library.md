<!---
title: Making a Slow Test Faster Featuring Styled Components and Testing Library
description: I have very little patience for slow unit test suites, and I really enjoy the challenge of making slow things fast. I think my co-workers know this, and some of them have found I can be easily nerd sniped into doing this kind of work (and I love it).
socialImage: https://user-images.githubusercontent.com/5233399/245847699-1f8096c5-18b4-4d38-aec6-1f4c3fdc1c2d.png
slackLabel1: Reading Time
slackLabel1Value: Depends!
slackLabel2: Publish Date
slackLabel2Value: June 24, 2023
-->

# Making a Slow Test Faster Featuring Styled Components and Testing Library

I have very little patience for slow unit test suites, and I really enjoy the challenge of making slow things fast. I think my co-workers know this, and some of them have found I can be easily [nerd sniped](https://en.wikipedia.org/wiki/Nerd_sniping) into doing this kind of work (and I love it).

In this post we're going to walk through the steps to diagnose _and_ remedy a real unit test I'm working on speeding up at work.

## Prerequisite Knowledge

Before jumping into a test, I want to set some context:

- This is from my employer's code base (not public), so code has been modified in examples to be more generic
- The application is built using React 17
- Unit tests are run in Jest, and most React tests are written using `react-testing-library` (and its constituent parts like `user-event`)
- App is pretty typical, using a lot of well-known tools from JavaScript land (`styled-components`, `apollo-client`, `react-router`, etc).
- App is a few years old, and has had close to 100 unique contributors

## Our Test Case

TODO

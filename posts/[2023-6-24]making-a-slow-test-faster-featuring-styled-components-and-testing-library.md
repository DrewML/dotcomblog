<!---
title: Making a Slow Test Faster Featuring Styled Components and Testing Library
description: I have very little patience for slow unit test suites, and I really enjoy the challenge of making slow things fast. I think my co-workers know this, and some of them have found I can be easily nerd sniped into doing this kind of work (and I love it).
socialImage: https://user-images.githubusercontent.com/5233399/245847699-1f8096c5-18b4-4d38-aec6-1f4c3fdc1c2d.png
slackLabel1: Reading Time
slackLabel1Value: Depends!
slackLabel2: Publish Date
slackLabel2Value: June 24, 2023
draft: true
-->

# Making a Slow Test Faster Featuring Styled Components and Testing Library

I have very little patience for slow unit test suites, and I really enjoy the challenge of making slow things fast. I think my co-workers know this, and some of them have found I can be easily [nerd sniped](https://en.wikipedia.org/wiki/Nerd_sniping) into doing this kind of work (and I love it).

In this post we're going to walk through the steps to diagnose _and_ remedy a real unit test I'm working on speeding up at work.

## Prerequisite Knowledge

Before jumping into a test, I want to set some context:

- This is from my employer's code base (not public), so code has been modified in examples to be more generic
- Ecommerce app for grocery chain, built using React 17
- Unit tests are run in Jest, and most React tests are written using `react-testing-library` (and its constituent parts like `user-event`)
- App is pretty typical, using a lot of well-known tools from JavaScript land (`styled-components`, `apollo-client`, `react-router`, etc).
- App is a few years old, and has had close to 100 unique contributors (so testing patterns proliferate once added)

## Our Test Case

Our shoppers have a page where they're shown products and categories they've previously saved, and they're able to quickly access them again. If the shopper has more than 15 saved items, we paginate the results. Our test case is verifying the happy path for pagination:

```js
it('paginates when > 15 products', async () => {
  render(<SavedPage products={products} categories={categories} />);

  // Switch to saved Products view
  await userEvent.click(screen.getByLabelText(/Select Products/));
  // Verify we're on the products view, and have > 15 products
  expect(await screen.findByRole('heading', { name: /19 products/ })).toBeInTheDocument();
  // Verify we're showing 15 products by checking for total # of add to cart buttons
  expect(screen.getAllByRole('button', { name: /Add to cart/ })).toHaveLength(15);
  // click to page 2
  await userEvent.click(screen.getByRole('button', { name: /Go to page 2/ }));
  // Verify we're showing the correct number of products on page 2
  expect(screen.getAllByRole('button', { name: /Add to cart/ })).toHaveLength(4);
});
```

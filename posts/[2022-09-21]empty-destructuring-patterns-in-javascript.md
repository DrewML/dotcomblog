<!---
title: Empty Destructuring Patterns in JavaScript
description: Digging into why JavaScript supports empty destructuring patterns
-->

# Empty Destructuring Patterns in JavaScript

@RyanCavanaugh posted a [fun quiz on Twitter](https://twitter.com/searyanc/status/1572730317433352192):

<p align="center">
  <a href="https://twitter.com/searyanc/status/1572730317433352192">
    <img
         src="https://user-images.githubusercontent.com/5233399/243551502-1112cc6b-938e-4d2b-8b28-363f84e4dd7c.png"
         alt="screenshot of the linked twitter quiz, showing an example of JavaScript destructuring with no variables in the pattern, and a survey with 3 options: Works as a no-op, fails to parse, or throws an error"
    />
  </a>
</p>

I logged a guess for `Fails to parse`, suspecting that JavaScript was going to surprise me as usual. _It did_ 😄

## Wut?

The correct answer to the quiz was `Throws (can't read properties from undefined)`, and I was _really_ curious why.

Turns out, empty destructuring assignments are valid. But not just valid: they're very explicitly supported, all the way back to the introduction of destructuring in ES2015.

<pre>
Source: https://262.ecma-international.org/6.0/

12.14.5.2 Runtime Semantics: DestructuringAssignmentEvaluation

with parameter value

ObjectAssignmentPattern : { }
    1. Let valid be RequireObjectCoercible(value).
    2. ReturnIfAbrupt(valid).
    3. Return NormalCompletion(empty).

ObjectAssignmentPattern :
    { AssignmentPropertyList }
    { AssignmentPropertyList , }
        1. Let valid be RequireObjectCoercible(value).
        2. ReturnIfAbrupt(valid).
        3. Return the result of performing DestructuringAssignmentEvaluation for AssignmentPropertyList using value as the argument.
</pre>

The first half of `DestructuringAssignmentEvaluation` is where this is specified. It only validates the right-hand side of the assignment can be coerced to an object, and nothing more.

All this leads us to one unfortunate truth: every line below is valid code that will not throw an error during parsing or execution 😔

```js
var {} = {}
let {} = {};
const {} = {};
({} = {})
```

## But Why?

With the help of Google and WayBack Machine, I've found an answer...sort of?

https://web.archive.org/web/20161222134616/http://wiki.ecmascript.org/doku.php?id=harmony:destructuring

> Empty patterns are allowed. Rationale: same basis case as for initialisers helps code generators avoid extra checks.

> (We don’t understand the statement about Empty patterns. Besides, they aren’t allowed by the above grammar. – MarkM & Tom)

> The grammar does produce empty patterns such as in var {} = obj; and [] = returnsArray(). Note the parentheses and Kleene * usage. The rationale is that empty initialisers are allowed, and indeed top-down parsers must parse an empty pattern on the left of assignment as a primary initialiser expression first, only then discover a = token and revise the AST appropriately to make the initialiser be a destructuring pattern.
> 
> The further rationale is that code generators should not need to check and avoid emitting a no-op empty destructuring assignment or binding, if it happens that there are no properties or elements to destructure for the particular case the code generator is handling (say, based on database records or JSON). This is a bit thin by itself, but with the first point (symmetry with initialisers) it adds up to a better case for empty patterns.

### My Interpretation

If I understand the texts above correctly, I think there were 2 motivations:

1. The `{}` syntax in JavaScript is sort of "overloaded" to be used for assigning properties (`var obj = { a: 1 }`), short-hand assigning properties (`var obj = { a }`), and selecting properties (`{ a } = obj`). When a parser is inspecting the code, it needs to treat the opening `{` as the start of a new object literal until it finds a `=` after `}`. I think the comments around symmetry are for developer intuition, but I'm guessing there was also a desire prevent the cover grammar[^1] from making things parsing even more complex.
2. Code generators (modern example: gRPC) will want to have generic code that can collect a list of fields being accessed, and spit the code out. Not allowing empty destructuring patterns would require codegen authors to add special handling to the case of 0 properties.

## Bonus

This is valid too!

```js
function emptyPattern({}) {}
```

[^1]: Dr. Axel Rauschmayer explains the concept of a "cover" grammar much better than I could in his article [ECMAScript 6: arrow functions and method definitions](https://2ality.com/2012/04/arrow-functions.html). Look at the `Parsing arrow functions` section for an overview.

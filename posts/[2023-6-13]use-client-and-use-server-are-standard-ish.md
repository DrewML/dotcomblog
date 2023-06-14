<!---
title: "use client" and "use server" are standard-ish
description: React's magical "use client" and "use server" strings are standard JavaScript, but they're also...kind of not.
socialImage: https://user-images.githubusercontent.com/5233399/244180224-4add9944-1627-4de5-b0b8-e1d5c0e4436a.png
slackLabel1: Reading Time
slackLabel1Value: 4 minutes
slackLabel2: Publish Date
slackLabel2Value: June 13, 2023
-->

# "use client" and "use server" are standard-ish

[React Server Components](https://react.dev/blog/2023/03/22/react-labs-what-we-have-been-working-on-march-2023#react-server-components) are here (sort of), and there's been some changes since the original announcement. One set of changes in particular stood out to me in the [Server Module Conventions v2 RFC](https://github.com/reactjs/rfcs/pull/227):

> - No more file extension conventions.
> - A "use client" directive at the top of a file defines that it's a boundary between server and client.

## Wait, What's a Directive?
Although not explicitly defined in the RFC, there's a hint at what a _directive_ is in one of the examples:

> When a Component with a "use client" directive _(similar to "use strict")_ is imported in a "React Server" environment its exports gets replaced with a special "Reference" object

The important bit here is the similarity to [Strict Mode's `"use strict"`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode), originally added in ES5 (2009). As always, we can look to the spec for an authoritative answer on language features. [11.2.1 Directive Prologues and the Use Strict Directive](https://tc39.es/ecma262/2023/#sec-directive-prologues-and-the-use-strict-directive):

> A Directive Prologue is the longest sequence of ExpressionStatements occurring as the initial StatementListItems or ModuleItems of a FunctionBody, a ScriptBody, or a ModuleBody and where each ExpressionStatement in the sequence consists entirely of a StringLiteral token followed by a semicolon. The semicolon may appear explicitly or may be inserted by automatic semicolon insertion (12.10). A Directive Prologue may be an empty sequence.
>
> A Use Strict Directive is an ExpressionStatement in a Directive Prologue whose StringLiteral is either of the exact code point sequences "use strict" or 'use strict'. A Use Strict Directive may not contain an EscapeSequence or LineContinuation.

Spec text is wordy as usual, but it boils down to the definition of two concepts:

- `Directive Prologue`: Gives special meaning to a string in the very top of a JS module or function
- `Use Strict Directive`: The 1 "official" `Directive Prologue`

I'm making the argument that `use client` is standard-_ish_ because of the note at the end of 11.2.1, making it clear that "implemention specific" Directive Prologues are allowed:

> NOTE The ExpressionStatements of a Directive Prologue are evaluated normally during evaluation of the containing production. Implementations may define implementation specific meanings for ExpressionStatements which are not a Use Strict Directive and which occur in a Directive Prologue.

## Directive Examples

```javascript
'use client'; // Directive, because it's at the top of a module
'use foobar'; // Directive, because it's at the top of a module and only has another directive before it

function someServerAction() {
  'use server'; // Directive, because it's at the top of a function
  'use bizzbuzz'; // Directive, because it's at the top of a function and only has another directive before it
  
  const someVal = null;
  'use bazz'; // Not a directive, because it has other code (someVal) before it in the function
}

'use client'; // Not a directive, because it has other code (someServerAction) before it in the module
```

## Not the First, Probably Not the Last

React isn't the first tool in the JavaScript ecosystem to introduce its own directive. There's a [great answer on Stack Overflow](https://stackoverflow.com/a/37535869) with a good collection of prior art:

- `use asm`: Mozilla's [`asm.js`](https://en.wikipedia.org/wiki/Asm.js) uses a [special directive to opt a function into static type checking](http://asmjs.org/spec/latest/#validation)
- `use sanity` and `use stricter`: Google proposed two new directions for JavaScript in 2015, "SaneScript" and "SoundScript." [SaneScript](https://github.com/tc39/notes/blob/main/meetings/2015-01/JSExperimentalDirections.pdf) used a `"use sanity"` directive, and [SoundScript](https://2ality.com/2015/02/soundscript.html) used a `"use stricter"` directive. The `SoundScript` proposal even supported passing arguments/options in a custom directive, `"use stricter+types"`.
- `use nobabel`: Electron's `electron-compile` [supported a directive at one point](https://github.com/atom/atom/issues/8416#issuecomment-131979794) to disable Babel on specific files.
- `use babel`: The Atom Editor [added built-in support for transpilation of any file using a directive](http://web.archive.org/web/20151226200823/http://blog.atom.io/2015/02/04/built-in-6to5.html)

## Closing Thoughts

I have reservations about Server Components, but I'm happy with this particular decision from the React team. Compared to other syntatic features like JSX, I appreciate that the use of standards-based syntax ensures that existing tools in the JavaScript ecosystem (ESLint/Prettier/TypeScript/etc) will continue to work.

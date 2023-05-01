<!---
title: No Build Blog with Cloudflare Workers, GitHub, and Markdown
description: Documenting my experience building a Markdown blog engine for myself that requires no builds for client-side code
socialImage: https://user-images.githubusercontent.com/5233399/235488415-310fbaad-a1e5-475e-b929-c4e87ef2811d.jpg
draft: true
-->

# No Build Blog with Cloudflare Workers, GitHub, and Markdown

I've been trying to start blogging consistently for years. Writing is _hard_, and finding the motivation to write is even more of a struggle. So, I did what most developers do when faced with this challenge.

## I Built a Blog Platform

I managed to convince myself for a moment in time that my lack of writing is due to me not using a Platform[^1] with features that would make me enjoy the writing process. I know I was lying to myself, but I tend to let my procrastination do its thing when it comes to side projects.

Based on my past experience using different Blog Platforms (closed-source SaaS and open-source), I knew there were certain features that were important to me.

### Requirements List

Before starting to build, I spent a few weeks (cough, months) jotting down notes on my phone when I'd have an idea of some feature I really wanted.

- **Minimal Front-End Tooling**: The site should be fairly simple, so no need to opt-in to the complexity of modern front-end
- **Instant Publishing**: I don't want to wait on CI to build and publish artifacts, and I don't want to run a build locally to publish a new post or edit
- **Minimal Mucking with Persistence**: Want to serve the site in a way that will be fast, which will require some form of persistence. _But_, I don't want to manage a database for an app of this size
- **Support for Comments**: Visitors should be able to leave feedback on posts (and ideally spam can be limited)
- **Support for Server-Side Code**: Fully static sites can be limiting. I want to have the ability to add logic on the server that runs at request time, not at build time
- **Support Posting and Editing from a Browser**: I don't want to pull up an IDE just to do some writing. It's distracting!
- **Markdown++**: Want to author posts in Markdown, but retain the ability to drop down to HTML when Markdown is too limiting
- **"Widget" Support**: Want the ability to author reusable functionality, like a video widget that lazy initializes YouTube embeds
- **Minimal Infrastructure Complexity to Manage**: Rules out AWS immediately 😅

Based on these requirements, I made some decisions on the tech stack

### Tech Stack Justification

- **Cloudflare Workers** for the server
  - "Serverless"
  - Cloudflare's Cache can act as a good persistence layer for any data fetched via HTTP
  - Fairly effortless to deploy updates
- **GitHub** for content management
  - Great built-in Markdown editor on mobile and desktop web
  - Good APIs for indexing and fetching data
  - Support for transforming Markdown to HTML


[^1]: I can call it an "Engine" or a "Platform" all I want, but that's an awfully fancy name for a few TypeScript files and a collection of Markdown documents.

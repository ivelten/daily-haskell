# Daily Haskell In Real Life

The source repository for the **[Daily Haskell In Real Life](https://ivelten.github.io/daily-haskell/)** blog — a deep-dive into practical Haskell for working engineers: real-world patterns, libraries, and techniques for daily professional use.

## What this repository is

This repository contains the Hugo site source and compiled output deployed to **GitHub Pages**. It is **not** where new content is authored or managed by hand.

All content lifecycle management — topic discovery, AI-assisted drafting, human review, and automated deployment — is handled by **[Jarvis](https://github.com/ivelten/jarvis)**, a Haskell-based orchestration engine that owns the full publishing pipeline for this blog.

## How content gets here

1. **Jarvis** discovers topics via the Gemini API, prioritised by real-world engagement metrics.
2. A draft is generated and posted to a private Discord forum channel for review.
3. The author reads, iterates with AI feedback, and approves (or rejects) the draft.
4. On approval, Jarvis commits the final Markdown post to `content/posts/` in this repository and triggers the **GitHub Actions deploy workflow**.
5. Hugo builds the site and GitHub Pages serves it.

## Tech stack

| Layer | Tool |
| --- | --- |
| Site generator | [Hugo](https://gohugo.io/) |
| Theme | [hugo-coder](https://github.com/luizdepra/hugo-coder) |
| Hosting | GitHub Pages |
| Orchestrator | [Jarvis](https://github.com/ivelten/jarvis) (Haskell) |
| AI backend | Google Gemini API |
| Review channel | Discord (forum channel) |

## Manual edits

While Jarvis is the primary manager, you can still edit site configuration, theme settings, or static assets directly in this repository. Content posts under `content/posts/` are generated and committed by Jarvis.

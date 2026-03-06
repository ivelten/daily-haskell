+++
title = "How I Built a Cyborg Blog in 4 Days with Haskell, AI, and a Discord Bot"
date = "2026-03-05"
tags = ["haskell", "ai", "github-copilot", "hugo", "discord", "gitops", "meta"]
+++

Four days ago, I had a blog with no theme, no posts, and no pipeline. Today, I have a fully operational content factory: a Haskell orchestration engine that discovers topics, drafts posts using the Gemini API, sends them to me for review on Discord, and deploys them automatically to GitHub Pages. This is the story of how I built it, what I used, and why I think the approach matters beyond this specific project.

## The Premise: A Cyborg, Not a Robot

I want to be precise about the goals from the start, because they shape every decision I made.

I did not want an AI blog. I did not want a system that publishes whatever a language model produces, unchecked, as if my name were not attached to it. I have views on quality. I have a voice. And frankly, I have enough experience to know when a draft is wrong, even if the grammar is impeccable.

What I wanted was a *cyborg* workflow — an architecture where I remain firmly in the loop, but the mechanical overhead of the publishing process is automated away. The AI does the research and the first draft. I do the judgment. The pipeline does the rest.

This distinction is not philosophical posturing. It is the design constraint that shaped the entire architecture.

## Day 1: Scaffolding the Haskell Project

I started with a blank Cabal project and a devcontainer. The toolchain choice — Haskell, GHC 9.6, PostgreSQL — was deliberate. I have spent the better part of the last two years convincing myself that Haskell is a good language for building reliable backend systems, and using it to build the thing that runs my blog felt like the right kind of proof.

And from this first commit, GitHub Copilot with Claude Sonnet 4.6 was my pair programmer for the entire Jarvis project — not just the blog. Every module, every data type, every SQL migration, every test. I designed each feature, I reviewed every generated step, and I decided what was good enough to commit. The AI wrote code at a speed I could not match alone. I provided the judgment that no model can substitute.

The project is structured as a single Cabal project with a library (`Orchestrator`) and an executable (`Main`). The library is organized into focused modules:

```text
src/Orchestrator/
├── AI/          -- Gemini API client
├── Database/    -- Persistent schema and connection pool
├── Discord/     -- discord-haskell bot
├── GitHub/      -- GitHub REST API client
├── Posts/       -- Hugo markdown renderer
├── Topics/      -- Content selector and ingestion logic
├── Pipeline.hs  -- Top-level orchestration
└── TextUtils.hs -- Slug generation, title splitting, text truncation
```

Each module has a single, clear responsibility. The boundary between them is a plain Haskell data type. The `Pipeline.hs` module wires them together. Nothing leaks.

## Day 2: The Concurrency Model

The most interesting design decision in the whole project is the concurrency model. Let me show you the core of `Main.hs`:

```haskell
main :: IO ()
main = do
  -- ... setup omitted ...

  _ <- forkIO $
    scheduledLoop "[Discovery]"
      (cfgDiscoveryIntervalSecs cfg)
      (runDiscovery pipeEnv)

  _ <- forkIO $
    scheduledLoop "[Drafts]"
      (cfgDraftIntervalSecs cfg)
      (runDraftGeneration pipeEnv)

  putStrLn "[Jarvis] All workers started. Discord bot running..."
  startBot dcCfg
```

Two workers are forked into the background. The main thread blocks on `startBot`, which runs the Discord event loop. This is idiomatic Haskell concurrency: lightweight threads, no shared mutable state between workers (each reads from the database independently), and a clean separation between the scheduled work and the reactive event handling.

The `scheduledLoop` function is worth highlighting on its own:

```haskell
scheduledLoop :: String -> Int -> IO () -> IO ()
scheduledLoop prefix intervalSecs action = do
  result <- try action :: IO (Either SomeException ())
  case result of
    Left ex  -> putStrLn $ prefix <> " Error: " <> displayException ex
    Right () -> pure ()
  putStrLn $ prefix <> " Sleeping for " <> show intervalSecs <> "s."
  sleepSecs intervalSecs
  scheduledLoop prefix intervalSecs action
```

Exceptions are caught, logged, and swallowed. The loop continues. This is exactly the right behaviour for a background worker: a failed discovery run should not crash the Discord bot that is currently handling a review. The system degrades gracefully.

One deliberate simplification worth naming: the `forkIO` calls have no mechanism to signal the workers to finish cleanly when the process is terminated. There is no `MVar` handshake, no `async`/`wait`, no graceful shutdown protocol. This is intentional for now — and the reasoning is straightforward. Every meaningful state change in the system is persisted to PostgreSQL *before* any external call is made. If the process is killed mid-flight, the database is the source of truth, and the next run will pick up from a consistent state. There is no in-memory work worth waiting for. Waiting for workers to drain before exiting would add complexity without adding safety.

## Day 3: The Review Flow

The review flow is the most operationally interesting part of the system. When a draft is ready, `processDraft` in `Pipeline.hs` does this:

```haskell
processDraft env rcKey rc = do
  draft <- generateDraft pipeAiCfg [rcToDiscovered rc]
  now   <- getCurrentTime
  markAsDrafted env rcKey now
  postDraftKey <- persistInitialDraft env rcKey rc draft now
  rr <- mkReviewRequest env rcKey postDraftKey now draft
  registerForReview pipeDcCfg rr
```

The step sequence matters: the database is updated *before* the Discord message is sent. If the Discord call fails, the item is already marked as `ContentDrafted`, so it will not be picked up again on the next cycle. Atomicity is enforced at the process boundary, not just at the database level.

The `mkReviewRequest` function is particularly elegant in how it handles the bilingual content problem. The Brazilian Portuguese body is tracked via an `IORef`, *not* exposed in the `ReviewRequest` record that the Discord bot sees. The English body is what the reviewer reads and revises. When approval fires, the pt-br body is retrieved from the `IORef` and committed alongside the English version:

```haskell
approve ptBrRef finalBodyEn tags = do
  finalBodyPtBr <- readIORef ptBrRef
  publishDraft env rcKey postDraftKey createdAt
    finalBodyEn finalBodyPtBr tags
```

This keeps the Discord interface clean — reviewers only ever interact with readable English — while ensuring both language versions are always in sync. The pt-br translation is revised in tandem whenever the reviewer requests a change.

## Day 4: The Blog Itself

On day four, I built the Hugo site — also with GitHub Copilot and Claude Sonnet 4.6. This is where the cyborg approach becomes reflexive: I used an AI assistant to build the blog that an AI assistant will populate. The same workflow applied: I described what I wanted, reviewed what was generated, adjusted direction when needed, and moved on. The entire conversation that produced this blog is itself a record of that process.

The theme is [hugo-coder](https://github.com/luizdepra/hugo-coder), added as a git submodule. The site supports English and Portuguese from the start. Each post is committed in two files with language suffixes — `my-post.en.md` and `my-post.pt-br.md` — and Hugo's multilingual mode handles the rest.

The deployment pipeline is a standard GitHub Actions workflow that runs `hugo --minify` and publishes to the `gh-pages` branch. Jarvis triggers it via the GitHub REST API after every successful commit.

## Why Haskell?

This is the part I want to dwell on, because it is the honest answer to the question I expect to be asked.

I chose Haskell because I believe it is exceptionally well-suited to AI-assisted development, and I wanted to test that belief on a real project. Here is the core argument:

Functional code expresses *what* a computation means, not *how* to execute it mechanically. Business logic becomes a composition of pure functions. Side effects are explicit, declared in the types. Data flows through pipelines rather than being mutated in place. When I ask GitHub Copilot to implement a function in Haskell, it works within a system that *forces* precision: the types must line up, the effects must be correct, the structure must be consistent. The language is a constraint, and constraints are good for AI collaboration.

The counterargument — that Haskell's learning curve makes it impractical — is real, but it is less relevant than it used to be. With AI tooling, the mechanical aspects of Haskell (imports, operator precedence, monad transformer stacking) are largely automated. What remains is the conceptual clarity that makes Haskell worthwhile in the first place. You still need to understand what you are building. The machine helps you write it down.

A necessary disclaimer: I am not claiming Haskell is the only language worth using, nor that it is universally the right choice. I do not believe in silver bullets. This is a deliberate personal decision and a concept test — I wanted to validate my hypothesis on a real project, with real constraints, under real time pressure. The conclusions I draw here are honest observations from that experiment, not a mandate.

## The Testing Discipline

One thing I want to be explicit about: every feature generated with AI assistance was unit tested before being considered done. Not as an afterthought — as the gate for moving on to the next step.

The test suite for Jarvis has two layers. The unit tests — covering pure logic like slug generation, text utilities, and pipeline orchestration steps — fake all external dependencies in-process: no Discord, no Gemini, no GitHub API. These run fast and run everywhere. The integration tests, however, do exercise the real database: they test the Persistent schema migrations and the database layer against an actual PostgreSQL instance. This is by design — database migrations are exactly the kind of thing you do not want to fake.

Both layers are wired into the devcontainer. The development environment is a [VS Code Dev Container](https://code.visualstudio.com/docs/devcontainers/containers) that spins up the application container alongside a PostgreSQL sidecar automatically using `docker-compose`. Once inside the container, GHC 9.6 and Cabal are on the `PATH`, the database is running, and `cabal test` just works. No manual setup, no environment variables to configure for local development. This was another area where the cyborg approach paid off — the entire devcontainer configuration was also built with Copilot, and it works reliably.

I will be honest: my commit messages during these four days are not a model of clarity. I was moving quickly, and the discipline I applied to the code itself did not always make it into the commit log. That is something I intend to improve as the project matures. But the substance was never in doubt — each step was reviewed, each new module was tested, and the pipeline was run end-to-end before I declared anything finished.

This is the discipline that makes AI-assisted development trustworthy rather than reckless. The speed is real. The accountability has to be real too.

## What I Learned

Four days. One Haskell application. One Hugo blog. One working MVP.

The most important lesson is that the cyborg model is not a compromise — it is a genuine improvement over both fully manual and fully automated alternatives. I get the efficiency of AI drafting without sacrificing editorial judgment. The system scales with my attention: if I am busy, drafts queue up; when I am ready, I review and approve from Discord in minutes.

The second lesson is about the collaboration model itself. GitHub Copilot with Claude Sonnet 4.6 was not a code generator I pointed at a blank file. It was a pair programmer I worked with iteratively — describing intent, reviewing output, pushing back on anything that did not meet the bar, and moving forward only when I was satisfied. The four days were intense, but they were not reckless.

I strongly believe this is the ideal workflow for right now: treat the AI agent as your pair in an Extreme Programming pair programming model. Not a tool you command. A partner you think alongside. In classic XP pairing, you have a driver and a navigator — one writes, one observes and challenges. With AI, the roles are natural: the AI drives with speed and encyclopedic skill at the keyboard; you navigate with domain expertise, architectural judgment, and accountability for the outcome. You hold the steering wheel. The AI runs the engine.

What makes this model work is the discipline it demands from *you*. You cannot switch off. You must understand every line before it is committed. You must know why a design decision was made, because you will be the one defending it, extending it, and debugging it six months from now. The AI gives you leverage. Expertise is what makes that leverage safe.

The third lesson is about Haskell specifically. The discipline that the language imposes — the explicitness, the types, the separation of pure and effectful code — translated directly into a system that is easier to reason about, easier to extend, and harder to break accidentally.

The four-day timeline would not have been possible without AI tooling. But the reliability of the result is not down to any single factor — it is the sum of the cyborg discipline, the architectural decisions made along the way, and the soundness that Haskell enforces by default.

## Next Steps

The MVP is ready. But it is not finished — there are two areas I want to refine.

The first is the **review process**. The current Discord bot handles the happy path well, but the edge cases of reviewer interaction — partially typed approval phrases, rapid emoji reactions, concurrent feedback in the thread, unexpected message formats — have not all been deliberately exercised. I want to harden the bot's event handling with more targeted tests and intentional chaos testing of the Discord interaction layer before this runs unattended.

The second is **infrastructure**. Right now, Jarvis runs locally. That means the discovery worker, the draft worker, and the Discord bot are only alive when my laptop is open. The obvious next step is to deploy it to a persistent machine — a small [Hetzner Cloud](https://www.hetzner.com/cloud/) instance is the plan. The setup is straightforward: a single VPS running Docker Compose with the Jarvis executable and a PostgreSQL container, managed with a simple `systemd` service or a Compose restart policy. Hetzner gives good value for the use case: low cost, reliable European infrastructure, and enough compute for a lightweight always-on orchestrator.

Once that is in place, the pipeline runs without me having to think about it. Jarvis discovers topics, queues drafts, and pings me on Discord when a review is ready. I approve or give feedback from my phone. The post goes live. That is the goal.

This [blog](https://github.com/ivelten/daily-haskell) exists to document that journey. [Jarvis](https://github.com/ivelten/jarvis) exists as the infrastructure that keeps it running. I am genuinely excited for what comes next.

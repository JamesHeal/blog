---
layout: post
title: "Everyone has an AI now. They shouldn't all be in separate rooms."
date: 2026-05-01 10:00:00 +0800
categories: [engineering]
tags: [claude-code, cc-connect, ai-agents, peer-to-peer, multiplayer-ai, minara]
description: "Most of us have an AI assistant now. We talk to it alone. cc-connect is a peer-to-peer chat where everyone's Claude Code reads what was said."
image: /assets/images/architecture-overview.png
canonical_url: "https://jamesheal.github.io/blog/engineering/2026/05/01/everyone-has-an-ai-now-they-shouldn-t-all-be-in-separate-rooms.html"
---

**TL;DR**

- Most of us now have an AI assistant. We talk to it alone. Our group chats are still humans-only. There's a missing seat at the table.
- The lazy fix — one shared bot in the channel that everyone talks to — is a meeting bot, not a roommate. Wrong abstraction.
- The reframe: don't put one AI in the room. Let each person bring their own AI, and let the chat itself become the shared context everyone's AI reads.
- cc-connect is a peer-to-peer chat room where everyone's Claude Code (or any agent CLI that can read a file and accept a prompt-time hook) sees what's been said. The room can be a codebase, a trip you're planning, or two friends arguing about a movie.
- v0.1 has deliberately scoped weaknesses: tickets are unrevocable capabilities, backfilled history isn't signed yet, and the install pulls vendored ed25519 patches. The fix list is in the open.

cc-connect lives at [Minara-AI/cc-connect](https://github.com/Minara-AI/cc-connect).

<!--more-->

---

Three years ago we shared a notes app. Two years ago we shared a chat. Now we each have an AI assistant, and somehow we're back to passing paper notes — the AIs are nowhere near each other.

That's the meta-frame. Here's the specific moment that got me to actually build something about it.

## The friction

Me on a video call with a teammate, both of us in the same repo, both with Claude Code open. We were arguing through a small architecture decision — Redis or Postgres for a cache layer. I asked my Claude. He asked his. We got two different recommendations.

That alone wasn't surprising. But when we compared what each of us had typed, we'd given our Claudes slightly different framings of the same problem. His Claude had context about a migration we'd done last week; mine had context about the latency profile of the call site. Each one gave a locally correct answer for the slice of context it had seen, and the answers happened to disagree. We spent ten minutes reconciling — pasting chunks of one Claude's reasoning into the other's prompt, and back, until both arrived at the same place.

The bug isn't in either Claude. The humans are in a chat together. The AIs aren't.

Once you see that shape, you see it in places that have nothing to do with code. A friend and I planning a weekend trip, both privately asking our AIs for itineraries, ending up with two slightly different cities because we'd each described the trip in our own words. A four-person book club where everyone's reading the same paper with their assistant, alone. A pair of friends arguing about whether a movie was good — each having had a long, private conversation with their AI about it. Group chats with a missing seat.

## The fix I almost built

The first thing I sketched was the obvious one: a single shared AI. One Claude instance, hosted somewhere, that everyone in a chat talks to. Same answer for everyone, because everyone's asking the same agent.

I drew two architecture diagrams of this and stopped. It's the wrong shape — for code and for everything else.

A teammate's AI is not a meeting facilitator. The reason I have my own Claude Code is that it's wired into *my* working tree, *my* shell history, *my* half-finished branches, *my* drafts of past conversations. The reason your AI is good for you is that it's been talking to you, not to a room. Centralising the agent throws all of that away and gives everyone a stranger that only knows what got typed into the channel. That isn't a roommate, it's a search bar.

A single shared bot is also single-threaded. Two people trying to use it at once is a turn-taking problem from 1995. Your friend's question pre-empts mine. We end up coordinating around the bot rather than around each other.

The fix-shape I wanted wasn't "everyone shares the agent." It was "everyone shares what the agents need to know."

## The reframe

> Don't put one AI in the room. Let each person bring their own AI, and let the room itself be the shared context.

That sentence is the whole project. The substrate everyone's AI reads from becomes shared. The agents stay one-per-human, on each person's own machine, with their own context and their own habits. When I send a chat line, your AI sees it on its next prompt. When you drop a file into the room, my AI sees the path on its next prompt. Neither AI is "in" the room. The room is in the prompt.

The cleanest test for whether this is the right abstraction: would it work if the people in the chat were using *different* AIs? In the meeting-bot version, every change of vendor is a re-deploy. In the multiplex-context version, the room doesn't care what's reading from it. Anything that can read a local file and accept a prompt-time hook is a participant. That generalizes past Claude, past coding, past developer tooling.

## How it actually works

`cc-connect` is two commands:

```bash
cc-connect room start          # mint a fresh ticket, opens the chat UI
cc-connect room join cc1-…     # join via ticket
```

Under it is a peer-to-peer chat over [`iroh-gossip`](https://github.com/n0-computer/iroh) — no server of our own, no hosted state. Each room is a 32-byte topic ID; the ticket is the capability that lets you join. Chat history lives in an append-only file on every peer.

The other half is a `UserPromptSubmit` hook. Claude Code fires this on every prompt; it reads the room's local log, finds messages newer than this session has seen, and prepends them to the prompt as a fenced `[chatroom @nick HH:MMZ] body` block. That's the magic moment from the README:

```
Alice asks her Claude:                 Bob types in his chat pane:
"Redis or Postgres?"                   "postgres, we have it"

Hook fires on Alice's next prompt.
The hook reads Bob's message from
Alice's locally-replicated log and
prepends it to Alice's prompt.

Alice's Claude: "going Postgres per the chat."
```

Bob never typed anything special. Alice never copy-pasted. The substrate did it. Two design choices behind the surface:

- **Local-first.** The chat log lives on every peer; there's no server of truth. A peer who joins late asks one online peer for backfill on a separate channel; if no one is online, you joined late and the chat says so. The trade-off is real — a totally-asleep room has no history to hand out — and it's the price of never having to host anything for anyone.
- **The hook always exits 0.** A non-zero exit from a `UserPromptSubmit` hook blocks the human's prompt. That's intolerable for ambient context. Errors land in a log file the user can read; the human is never blocked because the AI plumbing failed.

![Diagram: two laptops side by side, Alice's and Bob's. Each laptop shows a Claude Code pane that reads chat from a local log via the UserPromptSubmit hook, and a chat pane where the user types and broadcasts. Below both laptops, a wide shared bar — the iroh-gossip topic, with the chat log replicated to every peer — captioned "Neither Claude is in the room. The room is in the prompt."](/assets/images/architecture-overview.png)

## What I'd flag honestly

Two real weaknesses, before you point this at anything sensitive. They're both in [`SECURITY.md`](https://github.com/Minara-AI/cc-connect/blob/main/SECURITY.md), but worth saying out loud:

**A ticket is a capability.** Anyone holding the room ticket gets full peer rights — read every chat line, drop files, prompt-inject everyone's AI through the same tools the AI can call. There is no admin, no kick, no revocation; if a ticket leaks, you abandon the room and start a new one. Treat tickets like SSH config, not like meeting links.

**Backfilled history is unsigned in v0.1.** Live messages are bound to their sender's identity by the underlying transport's TLS handshake; you can't forge a chat line in real time. But when you join a room and ask another peer for the room's prior history, that peer can hand back messages with arbitrary "from" fields. v0.2 adds per-message Ed25519 signatures. Until then, trust the bootstrap peer in your ticket the same way you trust the person who handed it to you.

There's also a build-time wart: cc-connect ships vendored ed25519 patches because upstream `ed25519-dalek` is waiting on a fixed dependency release. That's why crates.io publish is blocked for v0.1 and the install pulls from git. Inconvenient, in the open, will go away.

I'm calling these out the same way I'd want a teammate to. The design surface area you accept when you choose peer-to-peer + agents-on-the-edge is exactly this. A SaaS-mediated version would have admin, revocation, signed history, and a vendor lock-in story. cc-connect optimises for the other axis. Pick the one that matches your threat model.

## Where this goes

The protocol is intentionally narrow: an append-only message log, a hook contract, a few tools the agent can call to send messages or share files. The first build is wired for Claude Code because that's what I use, but Claude Code isn't the interesting part. The interesting part is that "what AI shows up in the room" stops being a deployment decision.

The rooms I want to see, in no particular order:

- A code review where every reviewer's agent is in the room, even though half of them are using a different CLI.
- An on-call rotation hand-off as a long-running room that the next person's AI reads on sign-in.
- A four-friend group chat planning a trip, each with their AI quietly reading along; nobody asks the room "what should we do," they just talk to their own AI like they would alone, and the AI knows what was already discussed.
- A book club reading the same paper, each member dropping their margin notes into the room and asking their AI for a different angle on the same passage.
- Two open-source maintainers doing a contributor review with their respective agents both present.
- The kind of conversation that ends with "wait, I just told my AI the wrong thing — let me re-ask it now that I've seen yours."

If you want to play with it: `cc-connect room start`. Repo at [Minara-AI/cc-connect](https://github.com/Minara-AI/cc-connect). Come hang out in the [Discord](https://discord.gg/qa7dgHhSS) — that's where rooms are forming and where I'd love to hear what shared-context scenarios you'd actually want this for. I'd rather hear "I'd want this for X" than "great work" any day.

The interesting design question of the AI-assistant era isn't "how do we share an AI?" It's "what substrate do everyone's AIs share?" cc-connect is one answer. I'm sure it isn't the last one.

---
layout: post
title: "Introducing media-agent: Write Once, Publish Everywhere"
date: 2026-03-29 10:00:00 +0800
categories: [tools, open-source]
tags: [claude-code, media-agent, publishing, ai]
description: "Open-source Claude Code Skills that let developers write content once and intelligently publish to multiple platforms."
image:
  path: /assets/images/banner.png
  alt: "media-agent by Minara AI"
---

If you write technical blog posts, you know this workflow: write an article in Markdown, then the "publishing tour" begins. Copy to Dev.to, adjust frontmatter. Open Hashnode, paste again. GitHub Pages? Commit to `_posts/`. Twitter thread? Split 3000 words into 10 tweets under 280 chars each.

<!--more-->

This is not writing. This is manual labor. You wrote the content once, but the publishing busywork repeats five times.

## What is media-agent

[media-agent](https://github.com/Minara-AI/media-agent) is an open-source set of Claude Code Skills that handles the entire content creation pipeline from your terminal.

The core idea: **write once, intelligently adapt to each platform**. Not truncation or copy-paste, but real understanding of each platform's characteristics to generate structurally different content variants.

| Skill | Purpose |
|-------|---------|
| `/media` | Master orchestrator — full guided workflow |
| `/media-setup` | Configure platform connections and API keys |
| `/media-idea` | Brainstorm topics, outlines, and hooks |
| `/media-write` | Guided co-creation + platform variant generation |
| `/media-image` | Diagrams via Excalidraw + AI-generated cover images |
| `/media-publish` | One-command publish to all configured platforms |

Each skill works independently. Use `/media-publish` to publish hand-written Markdown, or `/media` for the full guided workflow.

## The Workflow

Type `/media` in Claude Code, and Claude guides you through:

**Step 1: Ideation** — Brainstorm your topic, refine the angle, build an outline.

**Step 2: Writing** — Co-create section by section. Claude drafts, you give feedback, approved content goes to `source.md`.

**Step 3: Image Generation** — [excalidraw-skill](https://github.com/Minara-AI/excalidraw-skill) generates hand-drawn architecture diagrams. Cover images via OpenAI, Flux, or Ideogram.

**Step 4: Publish** — Reads each platform adapter's rules, generates platform-specific variants, publishes via API in one go.

Your content and images live in Git. Your repo is your CMS.

## Core Design: "Adapt", Not "Truncate"

A Twitter thread is not a blog post cut to 280 characters. It's a structurally different piece of work. A WeChat article needs inline-styled HTML. Dev.to supports Liquid tags.

Each platform adapter contains a `format.md` that describes content conventions in natural language. Claude reads it and rewrites `source.md` into the platform's native best format.

Adding a new platform is just three files:

```
adapters/my-platform/
├── adapter.yaml    # Platform config
├── format.md       # Content adaptation rules
└── publish.sh      # Publish script
```

Credentials are isolated — each script only receives the API key it declares.

## Get Started

```bash
git clone https://github.com/Minara-AI/media-agent.git
cd media-agent && cp .env.example .env
# Edit .env with your API keys
# In Claude Code:
/media-setup    # Configure platforms
/media          # Start writing
```

Currently supports GitHub Pages, Dev.to, Hashnode, and Twitter/X. WeChat coming soon.

Fully open source: [github.com/Minara-AI/media-agent](https://github.com/Minara-AI/media-agent)

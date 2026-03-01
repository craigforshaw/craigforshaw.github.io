+++
title = 'Vibe Coding My Hugo Blog on GitHub Pages'
date = '2025-10-23T20:30:00+00:00'
draft = false
slug = 'vibe-coding-my-hugo-blog-on-github-pages'
tags = ['hugo', 'github-pages', 'vibe-coding', 'devops', 'blogging']
description = 'How I built and shipped my Hugo blog with an AI pair programmer—from setup to themes, posts, images, and deployment.'
+++

I've been posting on medium for a while now but I've always wanted a blog that was my own, simple to maintain, looked clean, and deployed automatically. Instead of overthinking every decision, I leaned into the current IT buzz word of the moment **vibe coding**: AI prompting with fast iterations, small fixes, and shipping in the flow.

This post is a quick recap of what I did, what went wrong, and what finally worked.

## Why Hugo + GitHub Pages

I picked Hugo for three reasons:

- It is fast and markdown-first.
- It works great with static hosting.
- It keeps content and code in one place.

For hosting, GitHub Pages was the natural fit since its free, I'm used to working with it and I wanted CI/CD from day one.

## What I Built

I set up the site in my `craigforshaw.github.io` repository with:

- Hugo Extended
- PaperMod theme
- GitHub Actions deployment workflow
- About page
- Post structure using both single files and page bundles

After the initial setup, I added real content and started customizing the home page and navigation.

## The Vibe Coding Workflow

The flow was basically:

1. Make a change.
2. Run/preview immediately.
3. Fix the next error fast.
4. Repeat until it feels right.

That sounds obvious, but the key was not getting stuck trying to perfect the architecture before publishing anything.

## Copilot model

I initially started out with the **GitHub model** but it didn't seem to vibe so well, made quite alot of errors or didn't produce what i was looking for. As soon as i switched to **Claude Sonnet** then I was able to make more progress as this model understood my prompts better and was able to implement fixes faster.

## Issues I Hit (and Fixed)

### 1) Hugo config parse errors

I hit TOML parsing issues from apostrophes/quotes in front matter and config values. The fix was to normalize quotes and keep front matter syntax strict.

### 2) Preview server confusion

At times the local preview looked down, but it was mostly terminal state and previous failed runs. Restarting Hugo cleanly and checking logs solved it.

### 3) Workflow reliability

The initial GitHub Actions workflow had extra moving parts that were not necessary for my setup. Simplifying the workflow reduced failure points and made deploys predictable.

### 4) Post image management

I switched to **page bundles** for content that has multiple screenshots. Keeping `index.md` and images together made writing and maintenance much easier.

## Content Migration from Medium

I migrated selected Medium posts into the Hugo blog. For posts with lots of images, the page bundle structure made it straightforward:

- One folder per post
- `index.md` for content
- Screenshots in the same folder
- Relative image links in markdown

This keeps each post self-contained and easy to move/edit later.

## Home Page Layout Decisions

I experimented with profile/home-info mode vs a posts-first home page. The big takeaway: choose the layout that matches your goal.

- If you want personal branding first: profile mode.
- If you want content discovery first: post list on home.

I leaned toward clarity and easy navigation over heavy customization.

## What I Like About the Current Setup

- Writing is fast: just markdown.
- Deploys are automatic on push.
- Theme is clean and readable.
- Content and infrastructure live together in one repo.

Most importantly, I can now focus on publishing rather than platform maintenance.

## Lessons Learned

- Ship the first version quickly.
- Keep the deployment path simple.
- Prefer conventions over custom complexity.
- Treat errors as part of the workflow, not blockers.

Vibe coding worked because it kept momentum high while still producing a solid, maintainable result.

## What’s Next

Next improvements I want to make:

- Better image optimization for large screenshots
- More consistent post templates
- Stronger internal linking between related posts
- Custom domain hardening and final polish

If you are setting up your own technical blog, my advice is simple: start with a minimal Hugo setup, automate deployment early, and publish your first real post as quickly as possible.

Done beats perfect.

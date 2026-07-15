# Cloudy with a Chance of Errors

Personal blog at **https://reb-b.github.io** — mixing cognitive neuroscience, cloud computing, AI, and personal experience.

Built with [Jekyll](https://jekyllrb.com/) + [Chirpy theme](https://github.com/cotes2020/jekyll-theme-chirpy). Deployed automatically via GitHub Actions on every push to `main`.

---

## Writing a post

Create a file in `_posts/` named `YYYY-MM-DD-your-title.md`:

```markdown
---
title: "Your Post Title"
date: 2026-07-14 10:00:00 +0200
categories: [neuroscience]
tags: [tag1, tag2]
---

Your content here in Markdown.
```

Push to `main` → live in ~2 minutes.

---

## Local preview

Requires Ruby 3.1+. If your system Ruby is older, install a newer one first:

```bash
brew install ruby
```

Then add it to your PATH (Homebrew prints the exact lines after install). Then:

```bash
gem install bundler
bundle install
bundle exec jekyll serve
```

Open **http://localhost:4000** — live-reloads as you edit.

---

## Customising the blog

All settings are in `_config.yml`. Fields left blank to fill in later:
- `twitter.username`
- `social.email`
- `social.fediverse_handle`
- `social.links` — LinkedIn, etc.
- `avatar` — path or URL to a profile photo

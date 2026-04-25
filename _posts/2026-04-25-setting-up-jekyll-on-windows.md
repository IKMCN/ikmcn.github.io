---
layout: post
title: "Setting up Jekyll on Windows for GitHub Pages"
description: "Installing Ruby and Jekyll on Windows, converting a static HTML site, and getting it live on GitHub Pages in an afternoon."
date: 2026-04-25
category: Tooling
---

I had a static HTML site on GitHub Pages — hand-written files, each page its own `.html`. It worked, but adding a new post meant copying an entire file, updating the HTML, then manually adding a card to the index. Not painful enough to fix immediately, but annoying enough to eventually fix properly.

Jekyll is what GitHub Pages is designed for. You write posts in Markdown, Jekyll converts them to HTML using your layouts, and GitHub builds it automatically on every push. This is how I got there on Windows.

## Installing Ruby

Jekyll is built on Ruby, so that comes first. On Windows the easiest route is RubyInstaller.

Go to [rubyinstaller.org/downloads](https://rubyinstaller.org/downloads) and download the **Ruby+Devkit** installer — it's the one marked with an arrow on the page. Run it, keep all the defaults.

At the end of the installation it will ask about MSYS2 components:

```
1 - MSYS2 base installation
2 - MSYS2 system update (optional)
3 - MSYS2 and MINGW development toolchain
Which components shall be installed? If unsure press ENTER [1,3]
```

Press **Enter** to accept the default. This installs the build tools Jekyll needs. It takes a few minutes.

Once it's done, close your terminal and open a fresh one. Verify Ruby installed correctly:

```bash
ruby -v
```

You should see something like `ruby 4.0.3 (2026-04-21)`. If the command isn't recognised, close and reopen the terminal — the PATH needs to refresh.

## Installing Jekyll

With Ruby in place, install Jekyll and Bundler via RubyGems:

```bash
gem install jekyll bundler
```

This pulls in Jekyll and its dependencies. Takes a couple of minutes.

## Setting up the repo

Clone your GitHub Pages repo and navigate into it:

```bash
git clone https://github.com/yourusername/yourusername.github.io
cd yourusername.github.io
```

Initialise a new Jekyll site in the current folder:

```bash
jekyll new . --force
```

The `--force` flag is needed if the folder isn't empty. Then install the dependencies:

```bash
bundle install
```

Start the local server:

```bash
bundle exec jekyll serve
```

Open `http://localhost:4000` in your browser. You should see the default Jekyll site — a basic Minima-themed page. That means it's working.

## Replacing the default theme with your own design

The default Jekyll theme (Minima) is fine but you probably want your own look. The structure you need is:

```
_layouts/
  default.html    ← shared nav, head, footer
  post.html       ← individual article template
_posts/
  YYYY-MM-DD-your-post-title.md
assets/
  css/
    main.css
index.html
_config.yml
Gemfile
```

The key insight is that `_layouts/default.html` wraps every page. Put your nav and footer there once and they appear everywhere. Your CSS goes in `assets/css/main.css` and gets linked in the default layout.

## Writing posts

Every post is a Markdown file in `_posts/` named with the date first:

```
_posts/2026-04-25-my-post-title.md
```

At the top of each file, front matter tells Jekyll how to handle it:

```yaml
---
layout: post
title: "My post title"
description: "A short summary shown in listing pages."
date: 2026-04-25
category: Architecture
---

Your content here in Markdown.
```

Jekyll picks up the date from the filename, but having it in the front matter too makes it explicit. The `description` field is useful for populating listing pages and cards on your homepage.

From here, adding a new post is just creating a new `.md` file. No HTML, no copying templates, no manually updating an index.

## Pushing to GitHub Pages

Once you're happy with the site locally, push it up:

```bash
git add .
git commit -m "migrate to Jekyll"
git push origin main
```

GitHub Pages detects the Jekyll structure automatically and builds the site. Give it a minute, then check `https://yourusername.github.io`. Your site should be live.

If you want to confirm GitHub Pages is building correctly, go to your repo on GitHub, click **Settings → Pages** and check the build status there.

## What you end up with

A site where adding a new post takes two minutes — create a `.md` file, write in Markdown, push. No HTML, no copying, no manual index updates. The layouts handle everything else.

It's the same principle as good software architecture: define the structure once, reuse it everywhere, change it in one place when you need to.

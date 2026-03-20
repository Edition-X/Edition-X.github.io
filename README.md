# Edition-X

A professional technical blog by Daniel Kelly, built with Jekyll and hosted on GitHub Pages.

## Local Development

1. Install Ruby 3.3.6 (via rbenv):
   ```bash
   rbenv install 3.3.6
   rbenv local 3.3.6
   ```

2. Install dependencies:
   ```bash
   gem install bundler
   bundle install
   ```

3. Serve locally:
   ```bash
   bundle exec jekyll serve --livereload
   ```

   The site will be available at `http://localhost:4000`

## Deployment

Push to the `main` branch. GitHub Pages automatically builds and deploys the site.

New articles should be written on a feature branch and merged to `main` via pull request.

## Structure

```
.
├── _config.yml           # Jekyll configuration
├── _layouts/
│   ├── default.html      # Base layout (SEO, nav, footer)
│   └── post.html         # Blog post layout
├── _posts/               # Blog posts (YYYY-MM-DD-title.md)
├── assets/
│   ├── css/style.css     # Main stylesheet
│   ├── images/           # Profile photo and images
│   ├── favicon.png       # Favicon (PNG)
│   └── favicon.ico       # Favicon (ICO)
├── index.html            # Homepage
├── about.html            # About page
├── blog.html             # Blog listing
├── 404.html              # Custom 404 page
├── robots.txt            # Search engine directives
└── Gemfile               # Ruby dependencies
```

## Adding Posts

Create a new branch and add a file in `_posts/` with the format `YYYY-MM-DD-title.md`:

```markdown
---
title: "Your Post Title"
date: YYYY-MM-DD
description: "A short summary for SEO and social sharing."
tags: [devops, aws]
---

Your post content here.
```

Open a PR to merge to `main` — the post goes live once merged.

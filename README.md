# Edition-X Blog

A minimal, professional technical blog built with Jekyll and GitHub Pages.

## Setup

### Local Development

1. Install Ruby and Jekyll:
   ```bash
   gem install jekyll bundler
   ```

2. Install dependencies:
   ```bash
   bundle install
   ```

3. Serve locally:
   ```bash
   bundle exec jekyll serve
   ```

   The site will be available at `http://localhost:4000`

### Deployment

Push to the `main` branch of the `Edition-X.github.io` repository. GitHub Pages will automatically build and deploy the site.

## Structure

```
.
├── _config.yml           # Jekyll configuration
├── _layouts/
│   ├── default.html      # Base layout for all pages
│   └── post.html         # Layout for blog posts
├── _posts/               # Blog posts (YYYY-MM-DD-title.md)
├── assets/
│   └── css/
│       └── style.css     # Main stylesheet
├── index.md              # Home page
├── about.md              # About page
├── blog.md               # Blog index
└── Gemfile               # Ruby dependencies
```

## Adding Posts

Create a new file in `_posts/` with the naming convention `YYYY-MM-DD-title.md`:

```markdown
---
layout: post
title: Your Post Title
---

Your post content here.
```

Posts will automatically appear on the blog index page.

## Customization

- Edit `_config.yml` to change site title, description, and author
- Modify `assets/css/style.css` for styling changes
- Update navigation links in `_layouts/default.html`

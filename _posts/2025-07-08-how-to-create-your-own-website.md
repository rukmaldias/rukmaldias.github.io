---
layout: post
title: "How to Create Your Own Website"
date: 2025-07-08
description: "A step-by-step guide to creating a professional personal website using Al-Folio, a Jekyll-based template designed for academics and professionals."
---

Creating a professional personal website doesn't have to be complicated. In this guide I'll walk you through setting up a site using **Al-Folio** — a beautiful, simple, clean, and responsive Jekyll theme tailored for professionals and academics, with complete documentation available on GitHub.

## Prerequisites

Before getting started, make sure you have the following installed:

- Git
- A GitHub account (for hosting)
- Docker and docker-compose
- Basic command-line familiarity

## 1. Initial Setup

Clone the Al-Folio repository and run it locally with Docker:

```bash
git clone https://github.com/alshedivat/al-folio.git
cd al-folio
docker-compose up
```

Once running, open your browser at `http://localhost:8080` to preview the site.

## 2. Core Configuration

Open `_config.yml` and update it with your personal details — your name, email, and contact information:

```yaml
first_name: Rukmal
last_name: Dias
email: rukmaldias@gmail.com
description: > 
  Software engineer and researcher.
```

## 3. Social Media Links

Configure your social profile links in `_data/socials.yml`:

```yaml
github_username: rukmaldias
linkedin_username: your-linkedin
twitter_username: your-twitter
```

## 4. Content Customisation

With the config in place, personalise the content:

- Add a **profile picture** by replacing the default image in `assets/img/`
- Update the **About** section in `_pages/about.md`
- Add your **publications**, **projects**, and any other sections relevant to your work

## 5. Additional Features

Al-Folio supports several optional features worth enabling:

- **Google Analytics** — add your tracking ID to `_config.yml`
- **Blog posts** — create markdown files in `_posts/` (just like this one)
- **News updates** — edit `_data/news.yml` for a timeline of updates

## Deployment

Once you're happy with the local preview, push to GitHub and let GitHub Actions handle the rest:

```bash
git add .
git commit -m "Initial site setup"
git push origin main
```

Your site will be live at `https://<your-github-username>.github.io` within a minute or two.

## Troubleshooting

Keep an eye on the Docker logs while developing locally — they'll surface any configuration errors before you push:

```bash
docker-compose logs -f
```

Validate your changes in the browser before every push, and you'll rarely run into surprises in production.

---

That's it. Al-Folio gives you a polished, professional site with minimal effort — and once it's deployed, adding new content is as simple as writing a markdown file.

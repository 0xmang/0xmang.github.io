---
title: "How I Set Up My Hugo Blog Site on Linux/macOS"
date: 2026-04-10
draft: false
tags:
  - Hugo
  - Static Site
  - Blogging
  - Linux
  - macOS
categories:
  - Web
  - DevOps
cover:
  image: "/images/hugo-logo-wide.svg"
  alt: "Hugo static site generator branding"
  caption: "Building a fast static site with Hugo."
---

# How I Set Up My Hugo Blog Site on Linux/macOS

This post documents how I set up a Hugo static website from scratch on Linux/macOS.

Hugo is a static site generator. Instead of using a database like WordPress, Hugo takes Markdown files, templates, and configuration files, then generates a complete static website made of HTML, CSS, JavaScript, images, and other assets.

The final generated site is placed in the `public/` directory, which can then be uploaded to a hosting provider such as GitHub Pages, Cloudflare Pages, Netlify, Render, an S3 bucket, Azure Storage static website hosting, or any standard web server.

---

## 1. Install prerequisites

Before installing Hugo, I made sure Git was installed.

### macOS

```bash
xcode-select --install
```

If Homebrew was not already installed, I installed it from:

```text
https://brew.sh/
```

Then I installed Git and Hugo:

```bash
brew install git hugo
```

Verify the installation:

```bash
git --version
hugo version
```

Output:

```text
git version 2.45.0
hugo v0.159.2+extended darwin/arm64 BuildDate=...
```

---

### Linux

On Ubuntu/Debian-based systems:

```bash
sudo apt update
sudo apt install -y git hugo
```

Verify the installation:

```bash
git --version
hugo version
```

Output:

```text
git version 2.43.0
hugo v0.152.2 linux/amd64 BuildDate=...
```

> Note: The version available through `apt` can sometimes be older than the latest Hugo release. If a theme requires a newer Hugo version, install Hugo from the official GitHub releases page instead.

---

## 2. Create a new Hugo site

I created a new Hugo project using the `hugo new site` command.

```bash
hugo new site my-hugo-site
cd my-hugo-site
```

Output:

```text
Congratulations! Your new Hugo site was created in /path/to/my-hugo-site.
```

At this point, the project structure looked like this:

```text
my-hugo-site/
├── archetypes/
├── assets/
├── content/
├── data/
├── hugo.toml
├── i18n/
├── layouts/
├── static/
└── themes/
```

Important folders:

| Folder/File | Purpose |
|---|---|
| `content/` | Markdown pages and blog posts |
| `themes/` | Hugo themes |
| `static/` | Static files copied directly to the final site |
| `layouts/` | Custom templates |
| `assets/` | CSS, JavaScript, images, and processed assets |
| `hugo.toml` | Main Hugo configuration file |
| `public/` | Generated website after running `hugo` |

---

## 3. Initialize Git

I initialized a Git repository so the site could be version-controlled.

```bash
git init
```

Output:

```text
Initialized empty Git repository in /path/to/my-hugo-site/.git/
```

Then I created a `.gitignore` file:

```bash
cat > .gitignore <<'EOF'
public/
resources/_gen/
.hugo_build.lock
.DS_Store
EOF
```

Why ignore `public/`?

The `public/` directory is generated output. I usually do not commit it unless my hosting setup specifically requires it.

---

## 4. Add a Hugo theme

For this example, I used the Ananke theme because it is commonly used in Hugo quick-start examples.

```bash
git submodule add https://github.com/theNewDynamic/gohugo-theme-ananke.git themes/ananke
```

Output:

```text
Cloning into '/path/to/my-hugo-site/themes/ananke'...
```

Then I updated `hugo.toml` to use the theme.

```bash
cat > hugo.toml <<'EOF'
baseURL = "https://example.com/"
languageCode = "en-us"
title = "My Hugo Blog"
theme = "ananke"

[pagination]
pagerSize = 10

[params]
description = "My personal blog built with Hugo."
author = "Your Name"
show_reading_time = true
EOF
```

Replace:

```text
https://example.com/
```

with the real domain of the blog.

For local testing, the exact `baseURL` is not critical, but it matters when deploying the site.

---

## 5. Create the first blog post

I created my first post using Hugo’s content generator:

```bash
hugo new posts/my-first-post.md
```

Output:

```text
Content "/path/to/my-hugo-site/content/posts/my-first-post.md" created
```

Then I opened the file:

```bash
nano content/posts/my-first-post.md
```

Example content:

```markdown
---
title: "My First Hugo Post"
date: 2026-05-22T10:00:00-04:00
draft: true
---

# My First Hugo Post

This is my first post on my Hugo blog.

I set up this site using Hugo, Git, and a theme. The site is written in Markdown and generated into static HTML.
```

The important setting here is:

```yaml
draft: true
```

Draft posts do not appear in the production build unless I explicitly include drafts.

---

## 6. Run the site locally

To preview the site locally, I ran:

```bash
hugo server -D
```

The `-D` flag includes draft posts.

Output:

```text
Watching for changes in /path/to/my-hugo-site/{archetypes,assets,content,data,i18n,layouts,static,themes}
Start building sites …
Web Server is available at http://localhost:1313/ (bind address 127.0.0.1)
Press Ctrl+C to stop
```

Then I opened the site in a browser:

```text
http://localhost:1313/
```

To stop the server:

```text
Ctrl + C
```

---

## 7. Edit the homepage and site settings

The main configuration is in:

```text
hugo.toml
```

Example:

```toml
baseURL = "https://example.com/"
languageCode = "en-us"
title = "My Hugo Blog"
theme = "ananke"

[pagination]
pagerSize = 10

[params]
description = "My personal blog built with Hugo."
author = "Your Name"
show_reading_time = true
```

Common settings to change:

| Setting | Meaning |
|---|---|
| `baseURL` | The final website URL |
| `title` | Site title |
| `theme` | Active Hugo theme |
| `languageCode` | Site language |
| `params.description` | Short site description |
| `params.author` | Author name |

---

## 8. Add an About page

I created a simple About page:

```bash
hugo new about.md
```

Then I edited it:

```bash
nano content/about.md
```

Example:

```markdown
---
title: "About"
date: 2026-05-22T10:30:00-04:00
draft: false
---

# About

Welcome to my blog.

I use this site to document projects, technical notes, tutorials, and things I learn while working with cloud, security, automation, and development tools.
```

---

## 9. Add more posts

To create additional posts:

```bash
hugo new posts/setting-up-hugo.md
hugo new posts/azure-governance-as-code.md
hugo new posts/linux-notes.md
```

Each post is a Markdown file under:

```text
content/posts/
```

Example post structure:

```markdown
---
title: "Setting Up Hugo"
date: 2026-05-22T11:00:00-04:00
draft: false
tags:
  - Hugo
  - Blogging
---

# Setting Up Hugo

This post explains how I set up my Hugo site step by step.
```

---

## 10. Add images and static files

For static files such as images, PDFs, or downloads, I used the `static/` folder.

Example:

```bash
mkdir -p static/images
cp ~/Downloads/site-logo.png static/images/
```

Then I referenced the image in Markdown:

```markdown
![Site Logo](/images/site-logo.png)
```

Anything placed in `static/` is copied directly to the final site root.

For example:

```text
static/images/site-logo.png
```

becomes:

```text
https://example.com/images/site-logo.png
```

---

## 11. Build the production site

When I was ready to generate the final static website, I ran:

```bash
hugo
```

Output:

```text
Start building sites …
hugo v0.159.2+extended darwin/arm64

                   | EN
-------------------+-----
  Pages            | 12
  Paginator pages  |  0
  Non-page files   |  0
  Static files     |  3
  Processed images |  0
  Aliases          |  1
  Cleaned          |  0

Total in 120 ms
```

This created the production website in:

```text
public/
```

The `public/` folder contains the generated HTML site:

```text
public/
├── index.html
├── posts/
├── about/
├── images/
├── sitemap.xml
└── index.xml
```

---

## 12. Build with minification

For a cleaner production build, I used:

```bash
hugo --gc --minify
```

What this does:

| Option | Purpose |
|---|---|
| `--gc` | Runs garbage collection for generated resources |
| `--minify` | Minifies HTML, CSS, JavaScript, JSON, SVG, and XML where applicable |

Output:

```text
Start building sites …
Total in 145 ms
```

---

## 13. Preview the production build locally

After building the site, I tested the generated `public/` folder locally.

From the project root:

```bash
cd public
python3 -m http.server 8080
```

Output:

```text
Serving HTTP on :: port 8080 (http://[::]:8080/) ...
```

Then I opened:

```text
http://localhost:8080/
```

To stop the Python web server:

```text
Ctrl + C
```

Then return to the project root:

```bash
cd ..
```

---

## 14. Commit the site to Git

After confirming the site worked locally, I committed the Hugo source files.

```bash
git status
git add .
git commit -m "Initial Hugo site setup"
```

Output:

```text
[main abc1234] Initial Hugo site setup
 12 files changed, 430 insertions(+)
 create mode 100 hugo.toml
 create mode 100 content/posts/my-first-post.md
 create mode 100 .gitignore
```

---

## 15. Push to GitHub

I created a new GitHub repository, then connected my local project to it.

```bash
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/YOUR-REPO.git
git push -u origin main
```

Output:

```text
Enumerating objects: 25, done.
Counting objects: 100% (25/25), done.
Writing objects: 100% (25/25), done.
Branch 'main' set up to track remote branch 'main' from 'origin'.
```

Replace:

```text
YOUR-USERNAME
YOUR-REPO
```

with the actual GitHub username and repository name.

---

## 16. Deployment options

There are multiple ways to deploy a Hugo site.

Common options:

| Platform | Build command | Publish directory |
|---|---|---|
| GitHub Pages | `hugo --gc --minify` | `public` |
| Cloudflare Pages | `hugo --gc --minify` | `public` |
| Netlify | `hugo --gc --minify` | `public` |
| Render | `hugo --gc --minify` | `public` |
| Static web server | Upload generated files | `public/` |

For most static hosting platforms, the important settings are:

```text
Build command: hugo --gc --minify
Publish directory: public
```

---

## 17. Optional: GitHub Actions deployment workflow

If deploying with GitHub Actions, I can create a workflow like this:

```bash
mkdir -p .github/workflows
nano .github/workflows/hugo.yml
```

Example workflow:

```yaml
name: Build Hugo Site

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository and submodules
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Set up Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: latest
          extended: true

      - name: Build
        run: hugo --gc --minify

      - name: Upload public directory as artifact
        uses: actions/upload-artifact@v4
        with:
          name: hugo-public-site
          path: public
```

This workflow builds the site and uploads the generated `public/` folder as a workflow artifact.

For actual deployment, I would add the deployment step for the hosting provider I choose.

---

## 18. Common commands I use

| Task | Command |
|---|---|
| Create a new site | `hugo new site my-hugo-site` |
| Start local server with drafts | `hugo server -D` |
| Create a new post | `hugo new posts/post-name.md` |
| Build production site | `hugo` |
| Build minified production site | `hugo --gc --minify` |
| Check Hugo version | `hugo version` |
| Initialize Git | `git init` |
| Commit changes | `git add . && git commit -m "message"` |

---

## 19. Troubleshooting

### Theme does not load

If the theme was added as a Git submodule, make sure submodules are initialized:

```bash
git submodule update --init --recursive
```

Then restart the Hugo server:

```bash
hugo server -D
```

---

### Draft posts do not show

Draft posts only show when using:

```bash
hugo server -D
```

To publish a post, change:

```yaml
draft: true
```

to:

```yaml
draft: false
```

---

### The deployed site has broken links

Check the `baseURL` in `hugo.toml`.

Example:

```toml
baseURL = "https://yourdomain.com/"
```

Then rebuild:

```bash
hugo --gc --minify
```

---

### Public folder is missing

The `public/` folder is created only after running:

```bash
hugo
```

or:

```bash
hugo --gc --minify
```

---

### Old content still appears after rebuild

Clean the generated files and rebuild:

```bash
rm -rf public resources/_gen
hugo --gc --minify
```

---

## 20. Final project structure

After setup, the site looked like this:

```text
my-hugo-site/
├── .git/
├── .github/
│   └── workflows/
│       └── hugo.yml
├── archetypes/
├── assets/
├── content/
│   ├── about.md
│   └── posts/
│       ├── my-first-post.md
│       └── setting-up-hugo.md
├── layouts/
├── public/
├── static/
│   └── images/
│       └── site-logo.png
├── themes/
│   └── ananke/
├── .gitignore
└── hugo.toml
```

---

## Conclusion

That is the full process I used to set up a Hugo blog site on Linux/macOS.

The basic workflow is:

```text
Install Hugo → Create site → Add theme → Write Markdown posts → Preview locally → Build static files → Deploy public/
```

Hugo is fast, simple, and easy to version-control because the entire site is just files. Once the project is set up, publishing a new blog post is usually as simple as creating a Markdown file, previewing it locally, committing it to Git, and deploying the generated site.

---

## References

- Hugo Quick Start: https://gohugo.io/getting-started/quick-start/
- Hugo Installation: https://gohugo.io/installation/
- Hugo Basic Usage: https://gohugo.io/getting-started/usage/
- Hugo Themes: https://themes.gohugo.io/

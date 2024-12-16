---
title: "How We Do Documentation Engineering"
meta_title: "How We Do Documentation Engineering"
description: ""
date: 2024-12-16T08:00:00Z
image: "/images/posts/2024/documentation-engineering/how-we-do-product-documentation-engineering.jpg"
categories: ["Product", "Documentation", "Engineering"]
author: "Medcl"
tags: ["Product", "Documentation", "Engineering"]
draft: false
---

At INFINI Labs, we see product documentation as an integral part of the product development process. Effective documentation ensures that users understand, adopt, and get the most out of our offerings.

Take a look at our documentation site: [https://docs.infinilabs.com/](https://docs.infinilabs.com/). It hosts detailed documentation for each of our products.

Managing comprehensive documentation might seem like an enormous task, especially since we don’t have a dedicated documentation team. However, with limited resources and many competing priorities, we’ve streamlined a practical and efficient approach to product documentation engineering.

So, how do we do it?

---

## The Tools We Use

Our documentation workflow relies on the following tools:

- **[GitHub Pages](https://pages.github.com/)**: For hosting our documentation website.
- **[GitHub Actions](https://github.com/features/actions)**: To automate builds and deployments.
- **[Hugo](https://gohugo.io/)**: A fast and flexible static site generator.
- **[Markdown](https://www.markdownguide.org/)**: To write clean and easily maintainable documentation.

We love GitHub's ecosystem for its reliability and developer-friendly features. Using GitHub Pages, we can host our documentation effortlessly. Once our documentation is compiled into static files using Hugo, it’s lightweight, fast, and easy to serve to users.

To enhance the user experience further, we integrate **offline search functionality** powered by our own [pizza-searchbox](https://github.com/infinilabs/pizza-searchbox/). This ensures users can quickly find what they need, even without an internet connection.

---

## How It All Works Together

### Organizing Components Across Repositories

We maintain separate repositories for each part of the documentation workflow. The final compiled version of all product documentation lives here:

- Compiled docs: [https://github.com/infinilabs/docs](https://github.com/infinilabs/docs)
- Hugo theme for docs: [https://github.com/infinilabs/docs-theme](https://github.com/infinilabs/docs-theme)
- Product with docs: [https://github.com/infinilabs/gateway](https://github.com/infinilabs/gateway)

By separating concerns, we make it easier to manage updates, streamline collaboration, and keep everything organized.


As you can see the layout of compiled folder is looks like this:

![Documentation Engineering](/images/posts/2024/documentation-engineering/compiled-documents-folder-layout.png)

Each products have folder for each different version, and the `main` is alwasy point to the latest version.

### And how did these static docs coming from?

Checkout this specify [product's](https://github.com/infinilabs/gateway/tree/main/docs) repo for example:

![Documentation Engineering](/images/posts/2024/documentation-engineering/product-docs-layout.jpg)

As you can see, in the product’s repository, there’s a folder named `docs`, which contains all the documentation specific to that product.

Alongside it, there’s a `config.yaml` defines the basic configuration for the product:

```yaml
# VERSIONS=latest,v1.0 hugo    --minify --baseURL="/product/v1.0/"  -d public/product/v1.0

title: INFINI Gateway
theme: book

# Book configuration
disablePathToLower: true
enableGitInfo: false

# Needed for mermaid/katex shortcodes
markup:
  goldmark:
    renderer:
      unsafe: true
  tableOfContents:
    startLevel: 1

# Multi-lingual mode config
# There are different options to translate files
# See https://gohugo.io/content-management/multilingual/#translation-by-filename
# And https://gohugo.io/content-management/multilingual/#translation-by-content-directory
defaultContentLanguage: en
languages:
  en:
    languageName: English
    contentDir: content.en
    weight: 3


menu:
  before: []
  after:
    - name: "Github"
      url: "https://github.com/infinilabs/gateway"
      weight: 10

...
EMITTED
...
```

Make sure you changed the right github's repo address and the product name.

And also there’s a `Makefile` that defines how we build the docs:

```shell
SHELL=/bin/bash

# Basic info
PRODUCT?= $(shell basename "$(shell cd .. && pwd)")
BRANCH?= main
VERSION?= $(shell [[ "$(BRANCH)" == "main" ]] && echo "main" || echo "$(BRANCH)")
CURRENT_VERSION?= $(VERSION)
VERSIONS?= "main"
OUTPUT?= "/tmp/docs"
THEME_FOLDER?= "themes/book"
THEME_REPO?= "https://github.com/infinilabs/docs-theme.git"
THEME_BRANCH?= "main"

.PHONY: docs-build

default: docs-build

docs-init:
	@if [ ! -d $(THEME_FOLDER) ]; then echo "theme does not exist";(git clone -b $(THEME_BRANCH) $(THEME_REPO) $(THEME_FOLDER) ) fi

docs-env:
	@echo "Debugging Variables:"
	@echo "PRODUCT: $(PRODUCT)"
	@echo "BRANCH: $(BRANCH)"
	@echo "VERSION: $(VERSION)"
	@echo "CURRENT_VERSION: $(CURRENT_VERSION)"
	@echo "VERSIONS: $(VERSIONS)"
	@echo "OUTPUT: $(OUTPUT)"

docs-config: docs-init
	cp config.yaml config.bak
	# Detect OS and apply the appropriate sed command
	@if [ "$$(uname)" = "Darwin" ]; then \
		echo "Running on macOS"; \
		sed -i '' "s/BRANCH/$(VERSION)/g" config.yaml; \
	else \
		echo "Running on Linux"; \
		sed -i 's/BRANCH/$(VERSION)/g' config.yaml; \
	fi

docs-build: docs-config
	hugo --minify --theme book --destination="$(OUTPUT)/$(PRODUCT)/$(VERSION)" \
        --baseURL="/$(PRODUCT)/$(VERSION)"
	@$(MAKE) docs-restore-generated-file

docs-place-redirect:
	echo "<!DOCTYPE html> <html> <head> <meta http-equiv=refresh content=0;url=main /> </head> <body> <p><a href=main />REDIRECT TO THE LATEST_VERSION</a>.</p> </body> </html>" > $(OUTPUT)/$(PRODUCT)/index.html

docs-restore-generated-file:
	mv config.bak config.yaml

```


Usually, there’s no need to make any changes—simply copy these files to your new product, and everything will work seamlessly.

---

## Automation Makes Everything Easier

And we use `.github/workflows/build-docs.yml` to define how to build the documentation once someone pushed the code, or someone released a new version, take a look at the github actions:

```yaml
name: Build and Deploy Docs

on:
  push:
    branches:
      - main
      - 'v*'
    tags:
      - 'v*'

jobs:
  build-deploy-docs:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Product Repo
        uses: actions/checkout@v2
        with:
          fetch-depth: 0

      - name: Set Variables Based on Ref
        id: vars
        run: |
          PRODUCT_NAME=$(basename $(pwd))  # Get the directory name as the product name
          echo "PRODUCT_NAME=$PRODUCT_NAME" >> $GITHUB_ENV
          CURRENT_REF=${GITHUB_REF##*/}
          IS_SEMVER=false
          SEMVER_REGEX="^v([0-9]+)\.([0-9]+)\.([0-9]+)$"

          if [[ "${GITHUB_REF_TYPE}" == "branch" ]]; then
            if [[ "$CURRENT_REF" == "main" ]]; then
              echo "VERSION=main" >> $GITHUB_ENV
              echo "BRANCH=main" >> $GITHUB_ENV
            elif [[ "$CURRENT_REF" =~ $SEMVER_REGEX ]]; then
              IS_SEMVER=true
              echo "VERSION=$CURRENT_REF" >> $GITHUB_ENV
              echo "BRANCH=$CURRENT_REF" >> $GITHUB_ENV
            else
              echo "Branch '$CURRENT_REF' is not a valid semantic version. Skipping build."
              exit 0
            fi
          elif [[ "${GITHUB_REF_TYPE}" == "tag" ]]; then
            if [[ "$CURRENT_REF" =~ $SEMVER_REGEX ]]; then
              IS_SEMVER=true
              echo "VERSION=$CURRENT_REF" >> $GITHUB_ENV
              echo "BRANCH=main" >> $GITHUB_ENV  # Set BRANCH to 'main' for tags
            else
              echo "Tag '$CURRENT_REF' is not a valid semantic version. Skipping build."
              exit 0
            fi
          fi

          # Gather branches and tags, filter for semantic versions, sort, remove duplicates
          VERSIONS=$(git for-each-ref refs/remotes/origin refs/tags --format="%(refname:short)" | \
            grep -E "^v[0-9]+\.[0-9]+\.[0-9]+$" | sort -Vr | uniq | tr '\n' ',' | sed 's/,$//')
          echo "VERSIONS=main,$VERSIONS" >> $GITHUB_ENV

      - name: Install Hugo
        run: |
          wget https://github.com/gohugoio/hugo/releases/download/v0.79.1/hugo_extended_0.79.1_Linux-64bit.tar.gz
          tar -xzvf hugo_extended_0.79.1_Linux-64bit.tar.gz
          sudo mv hugo /usr/local/bin/

      - name: Checkout Docs Repo
        uses: actions/checkout@v2
        with:
          repository: infinilabs/docs
          path: docs-output
          token: ${{ secrets.DOCS_DEPLOYMENT_TOKEN }}

      - name: Build Documentation
        run: |
          (cd docs && OUTPUT=$(pwd)/../docs-output make docs-build docs-place-redirect)

      - name: Commit and Push Changes to Docs Repo
        working-directory: docs-output
        run: |
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          
          if [[ -n $(git status --porcelain) ]]; then
            git add .
            git commit -m "Rebuild $PRODUCT_NAME docs for version $VERSION"
            git push origin main
          else
            echo "No changes to commit."
          fi

      - name: Rebuild Docs for Latest Version (main), if not already on main
        run: |
          # Only rebuild the main branch docs if the current ref is not "main"
          if [[ "$CURRENT_REF" != "main" ]]; then
            echo "Switching to main branch and rebuilding docs for 'latest'"

            # Checkout the main branch of the product repo to rebuild docs for "latest"
            git checkout main

            # Ensure the latest changes are pulled
            git pull origin main
            
            # Build Docs for Main Branch (latest)
            (cd docs && OUTPUT=$(pwd)/../docs-output VERSION="main" BRANCH="main" make docs-build docs-place-redirect)

            # Commit and Push Latest Docs to Main
            cd docs-output
            git config user.name "GitHub Actions"
            git config user.email "actions@github.com"
            
            if [[ -n $(git status --porcelain) ]]; then
              git add .
              git commit -m "Rebuild $PRODUCT_NAME docs for main branch with latest version"
              git push origin main
            else
              echo "No changes to commit for main."
            fi
          else
            echo "Current ref is 'main', skipping rebuild for 'latest'."
          fi
        working-directory: ./  # Working in the product repo
```

If you look closely at the GitHub Actions workflow, it simplifies the entire process by automating key tasks:
- Monitor Branches and Tags: Watches for branches or tags starting with v and validates them as semantic versioned branches or tags. Only valid versions trigger the documentation build process.
- Fetch Theme: Pulls the documentation theme from a separate repository to ensure a consistent look and feel across all products.
- Build and Deploy: Compiles the documentation into static files using Hugo, commits the changes with a clear and informative message, and pushes the updates to the docs repository.

Through GitHub Actions, the entire workflow becomes seamless:
- Compile: Converts Markdown files into a polished static site using Hugo.
- Deploy: Publishes the site effortlessly to GitHub Pages.

This automation reduces manual effort, ensures consistency across documentation, and allows us to focus on delivering quality content to users.

---

## Why This Approach Works for Us

1. **Efficiency**: With automation, we can focus on content rather than operations.
2. **Scalability**: As our product offerings grow, this workflow scales effortlessly.
3. **User Experience**: A fast, searchable, and offline-ready documentation site means better support for our users.

---

## Final Thoughts

Product documentation isn’t just a necessity—it’s a competitive advantage. By leveraging the right tools and automating where possible, we’ve built a process that delivers high-quality documentation without needing a dedicated team.

Want to see it in action? Visit our documentation site here: [https://docs.infinilabs.com/](https://docs.infinilabs.com/).

Let us know what you think! Your feedback helps us improve not only our products but also the way we document and share them with the world.
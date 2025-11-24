---
title: "Coco AI v0.9: Full support for automated AI review of GitLab merge requests (MR)"
meta_title: "Coco AI v0.9 Released"
description: "Coco AI v0.9 is now available!"
date: "2025-11-21T09:00:00+08:00"
categories: ["Release", "News"]
image: "/images/posts/2025/coco-v0.9/cover.jpg"
author: "Shiyang"
tags:
  ["Coco AI", "Release", "AI", "Search", "Assistant", "Gitlab", "Open Source"]
lang: "en"
category: "Product"
subcategory: "Released"
draft: false
---

# Coco AI

Coco AI is an [open-source](https://github.com/infinilabs/coco-app/), cross-platform search and assistant system designed
for both individuals and enterprises. Today, we are excited to announce its latest
release 0.9.0.

## Release Highlights

Coco AI v0.9 fully supports automatic AI review of GitLab merge requests (MR) and has been refactored into a plugin pipeline architecture. It adds more than 10 data source connectors such as Neo4j and MongoDB, opening a new paradigm of "AI + development" collaboration.

## Coco App v0.9.0

### Features

- feat: support switching groups via keyboard shortcuts #911
- feat: support opening logs from about page #915
- feat: support moving cursor with home and end keys #918
- feat: support pageup/pagedown to navigate search results #920
- feat: standardize multi-level menu label structure #925
- feat(View Extension): page field now accepts HTTP(s) links #925
- feat: return sub-exts when extension type exts themselves are matched #928
- feat: open quick ai with modifier key + enter #939
- feat: allow navigate back when cursor is at the beginning #940
- feat(extension compatibility): minimum_coco_version #946
- feat: add compact mode for window #947
- feat: advanced settings search debounce & local query source weight #950
- feat: add window opacity configuration option #963
- feat: add auto collapse delay for compact mode #981

### Bug fix

- fix: automatic update of service list #913
- fix: duplicate chat content #916
- fix: resolve pinned window shortcut not working #917
- fix: WM ext does not work when operating focused win from another display #919
- fix(Window Management): Next/Previous Desktop do not work #926
- fix: fix page rapidly flickering issue #935
- fix(view extension): broken search bar UI when opening extensions via hotkey #938
- fix: allow deletion after selecting all text #943
- fix: prevent shaking when switching between chat and search pages #955
- fix: prevent duplicate login success messages #977
- fix: fix quick ai not continuing conversation #979

### Improvements

- refactor: improve sorting logic of search results #910
- style: add dark drop shadow to images #912
- chore: add cross-domain configuration for web component #921
- refactor: retry if AXUIElementSetAttributeValue() does not work #924
- refactor(calculator): skip evaluation if expr is in form “num => num” #929
- chore: use a custom log directory #930
- chore: bump tauri_nspanel to v2.1 #933
- refactor: show_coco/hide_coco now use NSPanel’s function on macOS #933
- refactor: procedure that convert_pages() into a func #934
- refactor(post-search): collect at least 2 documents from each query source #948
- refactor: custom_version_comparator() now compares semantic versions #941
- chore: center the main window vertically #959
- refactor(view extension): load HTML/resources via local HTTP server #973

### Screenshots

![coco-app-1](/images/posts/2025/coco-v0.9/1.png)

![coco-app-2](/images/posts/2025/coco-v0.9/2.png)

## Coco Server v0.9.0

### Breaking changes

- refactor: make connectors pipeline-based (#545) #545
- refactor: re-implemented security features; rerun setup required

### Features

- feat: neo4j connector #539
- feat: add integrated store #551
- feat: rbac based security
- feat: user level data ownership & sharing
- feat: add permission check to management UI
- feat: add route permission verification
- feat: add entity card for users
- feat: add view to document management
- feat: add webhooks ui page (#558)
- feat: gitlab merge events webhook processor
- feat: add extension store integration
- feat: add support for editing connector processor configuration
- feat: add base path support for customizing endpoint
- feat: add pinyin support to name fields

### Bug fix

- fix: reset search keyword after extension type changed
- fix: adjust full-screen widget issues

### Improvements

- chore: change the home page to the search page after enabling search #541
- chore: update search api to support query dsl #550
- feat: mongodb connector #552
- chore: default sort by created
- chore: adjust locales
- chore: add confirmation password to user form
- chore: adjust connector type
- chore: adjust connector oauth redirect
- refactor: refactoring for integration
- refactor: remove token from integration
- chore: disabled email when editing user
- chore: adjust search settings
- refactor: add hover background effect for dark mode
- chore: fix document search
- chore: format date
- refactor: update initial value
- chore: fix missing datasource name
- chore: hide modal after installation finishes
- chore: refactoring via framework change
- chore: in order to deepthink we should fetch more docs #577

### Screenshots

![coco-server-3](/images/posts/2025/coco-v0.9/3.png)

![coco-server-4](/images/posts/2025/coco-v0.9/4.png)

![coco-server-5](/images/posts/2025/coco-v0.9/5.png)

## Getting Started

Coco AI is available on macOS, Windows, and Ubuntu. You can start using it today by downloading from the following links:

- **Download Coco AI**: <https://coco.rs>
- **Desktop App**: <https://github.com/infinilabs/coco-app>
- **Coco Server**: <https://github.com/infinilabs/coco-server>
- **Docs for Desktop App**: <https://docs.infinilabs.com/coco-app/main>
- **API for Coco Server**: <https://docs.infinilabs.com/coco-server/main>

## Looking Ahead

We’re committed to improving Coco AI, adding more features, and enhancing the integration with both personal and enterprise systems. Stay tuned for future updates and features as we continue to develop Coco AI into a powerful AI-driven knowledge management tool.

---

**Happy searching with Coco AI!**  
We look forward to seeing how Coco AI can help streamline your information access and improve productivity.

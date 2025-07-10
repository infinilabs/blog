---
title: "Coco AI v0.6.0 released, introduce the extension store"
date: "2025-06-29T18:00:00+08:00"
categories: ["Release", "News"]
image: "/images/posts/2025/coco-v0.6/banner.png"
author: "SteveLauC"
tags:
  [
    "Coco AI",
    "Release",
    "AI",
    "Search",
    "Productivity",
    "RAG",
    "Assistant",
    "Knowledge Management"
  ]
lang: "en"
category: "Product"
subcategory: "Released"
draft: false
---

# Coco AI

![boom](/images/posts/2025/coco-v0.6/coco_0_6_0_release.png)

Today, we are thrilled to announce the release of Coco AI 0.6.0, a fully 
[open-source][app-github], cross-platform intelligent search and assistant system. 
Check out our [website] to learn more!

## Release Highlights

In the last release, we introduced the [initial support for third-party extensions][pr572], 
taking a big step toward extensibility.  However, that feature was only available 
to developers — not end users.

In this release, we’re excited to introduce the Extension Store, where users can 
browse and install extensions with ease.

In this release, we introduce the extension store, where users can browse 
and install extensions.

Extension store can be opened in 2 ways:

1. Directly search for it:

   ![ext](/images/posts/2025/coco-v0.6/extension_store_search_result.png)

2. Click the "+" button in the Extension setting page

   ![ext](/images/posts/2025/coco-v0.6/extension_setting_panel.png)

Once the Extension Store is open, you can scroll through all available extensions 
or use search to find something specific:

![ext](/images/posts/2025/coco-v0.6/extension_store.png)

You can press "Command+Enter" to install the selected extension in the extension 
store, or you can take a closer look at it by pressing "Enter" to open the details
page:

![ext](/images/posts/2025/coco-v0.6/extension_store_detail_page.png)

Third party extensions are open-source as well, check [it][extension-source] out!

## Detailed release notes

### Coco App v0.6.0

#### Features

- feat: support `Tab` and `Enter` for delete dialog buttons #700
- feat: add check for updates #701
- feat: impl extension store #699
- feat: support back navigation via delete key #717

#### Bug fix

- fix: quick ai state synchronous #693
- fix: toggle extension should register/unregister hotkey #691
- fix: take coco server back on refresh #696
- fix: some input fields couldn’t accept spaces #709
- fix: context menu search not working #713
- fix: open extension store display #724

#### Improvements

- refactor: use author/ext_id as extension unique identifier #643
- refactor: refactoring search api #679
- chore: continue to chat page display #690
- chore: improve server list selection with enter key #692
- chore: add message for latest version check #703
- chore: log command execution results #718
- chore: adjust styles and add button reindex #719

### Coco Server v0.6.0

#### Bug fix

- fix: remove manually_renamed_title from assistant search

# Getting Started

Coco AI is available for macOS, Windows, and Ubuntu. You can start using it today 
by downloading [here](https://coco.rs/download)

Coco AI is available on macOS, Windows, and Ubuntu. Get started by downloading it [here][download].


[app-github]: https://github.com/infinilabs/coco-app
[website]: https://coco.rs/
[extension-source]: https://github.com/infinilabs/coco-extensions
[pr572]: https://github.com/infinilabs/coco-app/pull/572
[download]: https://coco.rs/download